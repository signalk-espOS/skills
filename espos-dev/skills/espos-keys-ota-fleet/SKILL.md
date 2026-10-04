---
name: espos-keys-ota-fleet
description: Signing keys, OTA and fleet updates for espOS devices — dev vs named vs CI signing keys and why a wrong key strands OTA, the device OTA flow and manifest, listing a firmware in the signalk-espOS registry, and how signalk-espos-manager discovers, authenticates (fleet API key) and updates a boat's devices.
---

# Keys, OTA and keeping a fleet current

Verified against espOS v0.15.0, registry `schema: 1`, signalk-espos-manager 0.1.0,
October 2026. Sources: `espOS/docs/ota.md`, `espOS/docs/security.md`,
`registry/README.md`, `registry/schema/project.schema.json`, manager `README.md` and
`src/`.

## Two different "keys" — do not conflate them

| | Firmware **signing key** | **Fleet key** |
|---|---|---|
| What | RSA-3072 (Secure Boot V2 scheme) private key, one per firmware project | One HTTP API Bearer key for every espOS device on a boat |
| Where | CI secret (`SIGNING_KEY_PEM`, `COCKPIT_SIGNING_KEY_PEM`, `GATEWAY_SIGNING_KEY_PEM`); locally `secure_boot_signing_key.pem`, git-ignored | Manager setting `auth.fleetKey`; per-device overrides in `<dataDir>/keys.json` (0600) |
| Checked by | `esp_ota_end` against the public key compiled into the **running** app | espOS httpd: `Authorization: Bearer <key>` or `espos_sid` cookie |
| Lose / change it | Every fielded device refuses OTA → USB reflash | Re-key over the API |

There is no fleet-wide *signing* key. Whoever builds a firmware owns its key; a third
party who wants their own build on their own boat generates their own key and accepts
that devices flashed with it only take OTAs signed by it.

## Signing keys in practice

Signed apps **without** hardware Secure Boot (`CONFIG_SECURE_SIGNED_APPS_NO_SECURE_BOOT`,
RSA scheme, `SECURE_SIGNED_ON_UPDATE_NO_SECURE_BOOT`, `SECURE_BOOT_BUILD_SIGNED_BINARIES`
in `espos.defaults`). Every build is signed, dev builds included. ESP32 (classic) needs
chip rev ≥ 3 (`CONFIG_ESP32_REV_MIN_3`). No eFuse anti-rollback and no hardware Secure
Boot in any profile; "rollback" means app rollback only.

* **Dev key (default).** Prologue with no `SIGNING_KEY`: a missing
  `<project>/secure_boot_signing_key.pem` is auto-generated with a warning. Every checkout
  and every example directory gets its **own** key, so OTA between them fails with
  "Secure boot signature verification failed" — expected, not a bug.
* **Named key.** `espos_project_prologue(… SIGNING_KEY /path/key.pem)` — never generated,
  fatal if missing, fatal if an existing `sdkconfig` names another key (delete it). The
  prologue fingerprints the key into `build/espos_signing_key.stamp` and forces a re-sign
  when it changes. Check an image:
  `espsecure verify-signature --version 2 --keyfile key.pem build/<app>.bin`.
* **Release key.** Generate once, keep it outside the repo, back it up:
  ```bash
  espsecure generate-signing-key --version 2 --scheme rsa3072 signing_key.pem
  gh secret set SIGNING_KEY_PEM < signing_key.pem
  ```
  Pass it explicitly (`secrets: signing_key: ${{ secrets.… }}`); a called workflow that
  does not receive it builds with a throwaway key — USB flash works, every later OTA is
  rejected. This has broken releases repeatedly (gateway #13–#18, cockpit #150/#152).
  Never expose the key to `pull_request` builds.
* **Deliberately unsigned (USB-only) release:** repo variable
  `ESPOS_ALLOW_UNSIGNED_RELEASE=true` (cockpit: `COCKPIT_ALLOW_UNSIGNED_RELEASE`); notes
  get "Unsigned build:"; registry entry `"signed": false`.
* **Rotation:** ship one update signed with the OLD key whose image embeds the NEW public
  key. Moving a device to a different project's key needs USB.
* Registry consumers get no auto-generated key ("a key invented by a build step is a key
  nobody kept"). `*.pem` is git-ignored and `scripts/check_no_secrets.sh` fails CI if one
  is tracked.

## Device OTA flow (`espos_ota`)

Pull-based. Sources: `POST /api/v1/ota {"url": …}`, or a manifest from config namespace
`ota`: `manifest_src` (`url` | `signalk`), `manifest_url`, `manifest_path`, `channel`
(stable), `auto_check` (on), `check_h` (24), `auto_install` (**off**), `allow_insecure`,
`confirm_tmo_s` (600). `manifest_src="signalk"` resolves `manifest_path` against the
connected SignalK server (default `/plugins/signalk-espos-updates/manifest.json`; the
manager rewrites it to its own mirror).

1. First check ~20 s after the network is up, then every `check_h`.
2. Manifest `schema:1` `{app, builds[{version,target,channel,url,size,sha256,notes,date}]}`
   ≤16 KiB; highest version for this app/target/channel wins, prereleases sort below.
3. Before writing: `project_name` must equal the running app; then SHA + signature.
4. New image boots `pending_verify`, confirms on network-up (or 60 s with no station
   configured); no network within `confirm_tmo_s` → marks invalid and rolls back; a panic
   rolls back via the bootloader.
5. Endpoints: `GET /api/v1/ota/status` (`state`, `running.key_fp`,
   `running.pending_verify`), `POST …/ota/check`, `…/confirm`, `…/rollback`, SSE `ota`.

`http://` URLs are fine (`CONFIG_ESP_HTTPS_OTA_ALLOW_HTTP`): integrity is the signature.

## Listing a firmware in the registry

`signalk-espOS/registry`: one `projects/<id>.json` per firmware; CI (`reindex.yml`,
nightly 04:17 UTC, on merge, on dispatch) resolves GitHub releases into the committed
`index.json` that the manager and the hosted flasher read with one unauthenticated fetch.

* PR adds **only** `projects/<id>.json`, titled `add: <id>`; `validate.yml` fails any PR
  that touches `index.json`.
* `app` = CMake `project()` name from `GET /api/v1/system/ping` — the field people get
  wrong. `boards[].reportedAs` = `hardware.board` from `/system/info`, matched exactly.
* `assets.ota` / `assets.merged`: anchored regexes with `(?<version>…)`; with two+ boards
  on one target also `(?<board>…)` and distinct `assetSegment`s.
* `webAssetsBranch: "release-assets"` turns on CORS-readable `*WebUrl`s (the browser
  flasher uses only `mergedWebUrl`).
* Each release's `espos` field comes from the submodule SHA at the tag matched **exactly**
  against espOS tags — an untagged espOS pin shows no espOS version.
* Local check: `export GH_TOKEN=$(gh auth token); node scripts/validate.mjs; node scripts/reindex.mjs` (don't commit `index.json`).

## The manager: new boat → kept current

1. **First flash:** hosted flasher `https://signalk-espos.github.io/signalk-espos-manager/flash/`
   (WebSerial needs https; the plugin's Flash page links there) → pick board → USB-flash
   `-merged.bin`. esptool-js with `flashSize: "keep"` so signed images aren't rewritten.
2. **Discovery:** mDNS `_espos._tcp`, keyed on TXT `id`; plus `staticHosts`, SignalK
   `espos.<device>.*` hints. Whether a key is needed: `GET /api/v1/system/ping` → `auth`.
3. **Auth:** fleet key, one attempt per device per poll — espOS locks out for 30 s after
   5 bad keys in 60 s and setting a new key does not clear the throttle.
4. **Configure OTA:** writes `ota.manifest_src="signalk"` + `ota.manifest_path` together.
5. **Mirror:** registry refetched every `refreshH` (12); images cached under
   `<dataDir>/fw/<app>/<version>/` with a per-app manifest served from the
   **unauthenticated** webapp mount `/signalk-espos-manager/fw/<app>/manifest.json`
   (devices fetch with a bare `esp_http_client`). Byte limits mirror the firmware
   (version 31, url 255, notes 127, manifest 16 KiB). URLs given to devices use the
   server's LAN address, never loopback.
6. **Update job:** `POST /api/v1/ota {url}` → rebooting → verifying → auto-confirm after
   `confirmGraceS` (120); rollbacks are never retried; `maxConcurrent` 1, queue pauses on
   failure. Match order: `app` → target → `reportedAs` → channel → `signingKeyId` vs
   `running.key_fp`; mismatch or unsigned → `requiresUsb`.

Manager rules: `start()` never throws, offline is a state not an error, parsers tolerate
firmware 0.7–0.15 side by side.

## Known gaps (October 2026) — check before relying on them

* Registry `"signed": false` is never turned into a per-build `unsigned`, and the manager
  only reads `build.unsigned`; no entry sets `signingKeyId`, so the key-mismatch check
  never fires; `reindex.mjs` emits no `otaSha256`/`mergedSha256` though the manager can
  verify them.
* Manager `auth.autoProvision` is declared but not implemented. README: "works, not yet
  proven on hardware".
* `espOS/docs/ota.md` still says `<APP>_ALLOW_UNSIGNED_RELEASE`; the workflows read
  `ESPOS_ALLOW_UNSIGNED_RELEASE`.
