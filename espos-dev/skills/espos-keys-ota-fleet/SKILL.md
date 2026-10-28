---
name: espos-keys-ota-fleet
description: Signing keys, OTA and fleet updates for espOS devices. Dev vs named vs CI signing keys and why a wrong key strands OTA, the device OTA flow and manifest, the P4's radio co-processor update, listing a firmware in the signalk-espOS registry, and how signalk-espos-manager flashes, discovers, authenticates (fleet API key) and updates a boat's devices.
---

# Keys, OTA and keeping a fleet current

Verified against espOS v0.24.0, registry `schema: 1`, signalk-espos-manager 0.8.0,
October 2026. Sources: `espOS/docs/ota.md`, `espOS/docs/security.md`,
`espOS/docs/hardware.md`, `registry/README.md`, `registry/schema/*.schema.json`,
manager `README.md` and `src/`.

## Two different "keys": do not conflate them

| | Firmware **signing key** | **Fleet key** |
|---|---|---|
| What | RSA-3072 (Secure Boot V2 scheme) private key, one per firmware project | One HTTP API Bearer key for every espOS device on a boat |
| Where | CI secret (`SIGNING_KEY_PEM`, or a project-named one such as `COCKPIT_SIGNING_KEY_PEM`, `N2K_GATEWAY_SIGNING_KEY_PEM`); locally `secure_boot_signing_key.pem`, git-ignored | Manager setting `auth.fleetKey`, managed on the webapp's Devices page; per-device keys in `<dataDir>/keys.json` (0600) |
| On the device | the public key compiled into the **running** app; `esp_ota_end` checks against it | `httpd.api_key`; espOS httpd takes `Authorization: Bearer <key>` or the `espos_sid` cookie |
| Lose / change it | Every fielded device refuses OTA, so each needs a USB reflash | Tell the manager and the devices the new one |

Whoever builds a firmware owns its signing key. One key may sign several projects:
measured from the release images in October 2026, the cockpit and the BLE gateway share
key `b3381b48b9cc9941`, while the third-party 8relay uses `c21a11df97bc7699`. A third
party who wants their own build on their own boat generates their own key and accepts
that devices flashed with it only take OTAs signed by it. The registry never signs or
checks firmware; it only records each project's fingerprint.

Key fingerprint, the registry's `signingKeyId`: the first 16 hex characters of the
Secure Boot V2 public-key digest. This is the same digest IDF's
`esp_secure_boot_get_signature_blocks_for_running_app()` computes, and the same one that
would be burned to eFuse. Read it from an image or a key with no device:

```bash
python -m espsecure signature-info-v2 build/<app>.bin    # "Public key digest for block 0: b3 38 1b …"
python -m espsecure digest-sbv2-public-key --keyfile key.pem -o d.bin && od -An -tx1 d.bin | tr -d " \n" | head -c16
```

## Signing keys in practice

Signed apps **without** hardware Secure Boot (`CONFIG_SECURE_SIGNED_APPS_NO_SECURE_BOOT`,
RSA scheme, `SECURE_SIGNED_ON_UPDATE_NO_SECURE_BOOT`, `SECURE_BOOT_BUILD_SIGNED_BINARIES`
in `espos.defaults`). Every build is signed, dev builds included. ESP32 (classic) needs
chip rev ≥ 3 (`CONFIG_ESP32_REV_MIN_3`). No eFuse anti-rollback and no hardware Secure
Boot in any profile; "rollback" means app rollback only.

* **Dev key (default).** Prologue with no `SIGNING_KEY`: a missing
  `<project>/secure_boot_signing_key.pem` is generated with a warning. Every checkout
  and every example directory gets its **own** key, so OTA between them fails with
  "Secure boot signature verification failed". That is expected, not a bug.
* **Named key.** `espos_project_prologue(… SIGNING_KEY /path/key.pem)`: never generated,
  fatal if missing, fatal if an existing `sdkconfig` names another key (delete it). The
  prologue fingerprints the key into `build/espos_signing_key.stamp` and forces a re-sign
  when it changes. Check an image:
  `espsecure verify-signature --version 2 --keyfile key.pem build/<app>.bin`.
* **Release key.** Generate once, keep it outside the repo, back it up:
  ```bash
  espsecure generate-signing-key --version 2 --scheme rsa3072 signing_key.pem
  gh secret set SIGNING_KEY_PEM < signing_key.pem
  ```
  Pass it explicitly (`secrets: signing_key: ${{ secrets.… }}`). A called workflow that
  does not receive it builds with a throwaway key: USB flash works, and every later OTA
  is rejected. This has broken releases repeatedly (gateway #13–#18, cockpit
  #150/#152). Never expose the key to `pull_request` builds.
* **Deliberately unsigned (USB-only) release:** repo variable
  `ESPOS_ALLOW_UNSIGNED_RELEASE=true` (cockpit: `COCKPIT_ALLOW_UNSIGNED_RELEASE`); the
  notes get "Unsigned build:"; registry entry `"signed": false`. The manager and the
  flasher treat such a project as USB-only.
* **Rotation:** ship one update signed with the OLD key whose image embeds the NEW public
  key. Moving a device to a different project's key needs USB.
* Registry consumers get no generated key ("a key invented by a build step is a key
  nobody kept"). `*.pem` is git-ignored and `scripts/check_no_secrets.sh` fails CI if one
  is tracked.

## Device OTA flow (`espos_ota`)

Pull-based. Sources: `POST /api/v1/ota {"url": …}`, or a manifest from config namespace
`ota`: `manifest_src` (`url` | `signalk`), `manifest_url`, `manifest_path`, `channel`
(stable), `auto_check` (on), `check_h` (24), `auto_install` (**off**), `allow_insecure`,
`confirm_tmo_s` (600). `manifest_src="signalk"` resolves `manifest_path` against the
connected Signal K server (default `/plugins/signalk-espos-updates/manifest.json`; the
manager rewrites it to its own mirror).

1. First check ~20 s after the network is up, then every `check_h`.
2. Manifest `schema:1` `{app, builds[{version,target,channel,url,size,sha256,notes,date}]}`
   ≤16 KiB; the highest version for this app/target/channel wins, prereleases sort
   below. The board is **not** a match key, which is why the manager writes one
   manifest per board (below).
3. Before writing: `project_name` must equal the running app; then SHA + signature.
4. The new image boots `pending_verify` and confirms on network-up (or after 60 s with
   no station configured). No network within `confirm_tmo_s` marks it invalid and rolls
   back; a panic rolls back through the bootloader.
5. Endpoints: `GET /api/v1/ota/status` (`state`, `running.key_fp`,
   `running.pending_verify`), `POST …/ota/check`, `…/confirm`, `…/rollback`, SSE `ota`.

`http://` URLs are fine (`CONFIG_ESP_HTTPS_OTA_ALLOW_HTTP`): integrity is the signature.

What an OTA changes and what it keeps: it writes the idle app slot only. The partition
table, `nvs` (settings, WiFi, token, keys), `nvs_keys` and `storage` survive. The web UI
is built into the app image since espOS 0.17.3 (#202), so an OTA brings the page that
matches its firmware; `GET /api/v1/system/info` → `ui` says `embedded`, `storage` or
`placeholder`. A partition-table change reaches only boards flashed over USB.

## The P4's radio co-processor (C6)

An ESP32-P4 has no radio; an ESP32-C6 provides WiFi and BLE over SDIO (`esp_hosted`),
and it runs firmware of its own. `GET /api/v1/system/info` → `hardware.coprocessor`
reports its version, and the `coprocessorStale` health condition is raised when
esp_hosted's own version check says it is older than the host expects (`0.0.0` means an
image too old to announce itself).

`POST /api/v1/system/coprocessor/update` updates it over the link. The P4 downloads the
image into PSRAM, accepts only the one whose SHA-256 is `ESPOS_COPROC_FW_SHA256` in
`components/espos_wifi/src/coproc_image.h` (the C6 image carries no signature the P4
could check, so the trust is in the signed P4 firmware), and streams it into the C6's
idle slot. It is refused while the P4's own image is still pending verification and
while a firmware OTA runs, and the other way round, because each ends in a restart.
The manager offers it when `coprocessor.stale` is set (`/api/fleet/:id/coprocessor`)
and holds a firmware update until a radio update has finished. Details in
`espOS/docs/hardware.md`, "Over the link".

## Listing a firmware in the registry

`signalk-espOS/registry`: one `projects/<id>.json` per firmware, plus a board
catalogue (`boards/<id>.json`: vendor, model, photo, chip, `flashSizes`, `usbBridge`,
`factoryApps`) and use cases (`usecases/<id>.json`: an owner-worded job with ranked
boards). CI resolves GitHub releases into the committed `index.json` that the manager
and the flasher read with one unauthenticated fetch.

* A PR adds **only** `projects/<id>.json` (and a `boards/` file for a new board),
  titled `add: <id>`; `validate.yml` fails any PR that touches `index.json`.
* `app` = CMake `project()` name from `GET /api/v1/system/ping`. This is the field
  people get wrong. `boards[].reportedAs` = `hardware.board` from
  `/api/v1/system/info`, matched exactly.
* `assets.ota` / `assets.merged`: anchored regexes with `(?<version>…)`; with two or more
  boards on one target also `(?<board>…)` and distinct `assetSegment`s. Per-size images
  (`-merged-<size>.bin`, espOS `flash_tables`) are found by themselves and listed as
  the build's `mergedByFlash`; set `flashMinBytes` on a board to the smallest size
  shipped.
* `webAssetsBranch: "release-assets"` turns on CORS-readable `*WebUrl`s (the flasher uses
  only the merged ones).
* Each release's `espos` field comes from the submodule SHA at the tag, matched
  **exactly** against espOS tags. An untagged espOS pin shows no espOS version.
* Local check: `export GH_TOKEN=$(gh auth token); node scripts/validate.mjs; node scripts/reindex.mjs` (don't commit `index.json`).

When the index is rebuilt: `reindex.yml` runs hourly, on merge and on a
`firmware-released` repository dispatch. GitHub runs the hourly cron late or drops it,
so the official firmware repos end each release with a `notify-registry` job that sends
the dispatch (it needs write access to the registry, which outside projects lack).
`concurrency.queue: max` keeps every queued run waiting instead of cancelling the
release's run behind the hourly one. A release missing from the manager is almost
always a stale index: a manual `workflow_dispatch` of `reindex.yml` fixes it.

## The manager: new boat, kept current

1. **First flash:** the flasher the plugin serves (**New board**), or the hosted copy at
   `https://signalk-espos.github.io/signalk-espos-manager/flash/` when the server is on
   plain http (WebSerial needs a secure page). It asks what the board should do (the
   registry's use cases) or which board it is, reads the chip, flash size, USB bridge
   and any firmware already on it ("This looks like…"), and writes the largest
   `mergedByFlash` image that fits. It refuses an image whose bootloader header names
   more flash than the chip has: such an image flashes and then never boots.
   esptool-js runs with `flashSize: "keep"` so signed images aren't rewritten.
2. **Discovery:** mDNS `_espos._tcp`, keyed on TXT `id`; plus `staticHosts` and Signal K
   `espos.<device>.*` hints. Whether a key is needed: `GET /api/v1/system/ping` → `auth`.
3. **Fleet key:** the Devices page can generate one, take your own, replace it or remove
   it. With `auth.autoProvision` (off by default, since it changes the device and its
   own web UI then asks for the key too) the manager writes it as `httpd.api_key` to every online
   device whose ping reports no key, re-checking ping just before the write. Replacing
   it rewrites every online device that used the old key; an offline one keeps the old
   key, which the manager remembers for it, and gets the new one when it answers again.
   Removing it never opens a device: each keeps the key as its own. One attempt per
   device per poll, because espOS locks out for 30 s after 5 bad keys in 60 s and setting
   a new key does not clear the throttle.
4. **Configure OTA:** writes `ota.manifest_src="signalk"` + `ota.manifest_path` together
   ("Fix this" on a device that is not set up for updates).
5. **Mirror:** the registry is re-read every `refreshH` (default 1 h, one conditional
   request). Images are cached under `<dataDir>/fw/<app>/<version>/` and served from the
   **unauthenticated** webapp mount `/signalk-espos-manager/fw/<app>/`, because a device
   fetches with a bare `esp_http_client` and has no admin login. There is one
   `manifest-<board>.json` per declared board plus `manifest.json` for devices whose
   board does not resolve: espOS matches on target and channel only, and one manifest
   per app once let a cockpit 7B offer itself the X 7's image (manager #35). Byte limits
   mirror the firmware (version 31, url 255, notes 127, manifest 16 KiB). URLs given to
   devices use the server's LAN address, never loopback.
6. **Update job:** `POST /api/v1/ota {url}`, then rebooting, verifying, and a confirm
   after `confirmGraceS` (120) of good health; rollbacks are never retried;
   `maxConcurrent` 1, and the queue pauses on failure. Match order: `app` → target →
   `reportedAs` → channel → `signingKeyId` vs `running.key_fp`. A mismatch, or a
   `signed: false` project, means `requiresUsb`.

Manager rules: `start()` never throws, offline is a state not an error, and the parsers
tolerate firmware from espOS 0.7 on side by side (unknown `/system/info` fields are
dropped, not fatal).
