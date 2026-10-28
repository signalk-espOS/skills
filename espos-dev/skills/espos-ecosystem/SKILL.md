---
name: espos-ecosystem
description: Orientation for working on espOS or any of its child repos (registry, signalk-espos-manager, espos-p4-cockpit, espos-ble-gateway, espos-nmea-gateway, espos-bathy-bridge, espos-zigbee-coordinator, the cockpit WASM, signalk-hmi-designer, signalk-espos-stream). Which repo owns what, the contracts between them, and the house rules that differ per repo. Load first in any espOS session.
---

# The espOS ecosystem

Snapshot: espOS **v0.24.0** on ESP-IDF **v6.0.3**, late October 2026. Versions move;
read `version.txt` and `.idf-version` in the checkout instead of trusting the numbers
here.

espOS is a firmware base for ESP32 boards that talk to Signal K. Despite the name it
is not an operating system: it is a set of ESP-IDF components a firmware links in,
much like SensESP, but on plain ESP-IDF 6 instead of Arduino and PlatformIO. It does
the parts every such device needs (WiFi or Ethernet, the setup portal, a settings page,
finding the Signal K server and getting approved, sending deltas, PUT and REST, mDNS,
a health watchdog, logs, core dump, signed OTA), started with one `espos_start()`
call. Firmware projects sit on top. A registry lists their releases, and a Signal K
plugin keeps a boat's devices updated from it.

## Repos and what each owns

| Repo | Role | Language / gate |
|---|---|---|
| `signalk-espOS/espOS` | **Primary.** Components `components/espos_*`, CMake prologue, sdkconfig layers, partition tables, reusable release workflows, docs site | C11 (+C++ for n2k/voice/audio). `test/host/run_all.sh`, `scripts/build.sh` |
| `signalk-espOS/registry` | `projects/<id>.json` per firmware, the `boards/` and `usecases/` catalogue, and the CI-generated `index.json` | Node 24 scripts, no deps |
| `signalk-espOS/signalk-espos-manager` | Signal K plugin: mDNS discovery, registry client, firmware mirror, OTA jobs, radio co-processor updates, fleet API key, plus the browser flasher (bundled and on GitHub Pages) | TS + Preact. `npm run format && npm run build && npm test` |
| `signalk-espOS/espos-release-test` | Fixture for espOS's reusable `release-firmware.yml`; also the registry-consumer template (no submodule) | C |
| `signalk-espOS/skills` | These skills | Markdown |
| `dirkwa/espos-p4-cockpit` | ESP32-P4 panel firmware: JSON Layout Player (LVGL), N2K gateway, voice | C/C++, has AGENTS.md |
| `dirkwa/espos-ble-gateway` | BLE to signalk-server's BLE provider API. `main.c` is one `espos_start(NULL)`; the logic lives in espOS `espos_ble` | C, no AGENTS.md |
| `dirkwa/espos-nmea-gateway` | NMEA 2000 (CAN, candump over TCP) to Wi-Fi on espOS `espos_n2k`; NMEA 0183 over a UART planned. Decodes nothing | C, has AGENTS.md |
| `dirkwa/espos-bathy-bridge` | ESP32-P4 that shows its microSD to a chartplotter as a USB card (`espos_usb_storage`) and uploads the plotter's RealBathy records to crowd-depth under the device's own vessel context | C, has AGENTS.md |
| `dirkwa/espos-zigbee-coordinator` | ESP32-C5/C6 Zigbee coordinator serving the ZBOSS NCP to Zigbee2MQTT over TCP; NCP vendored from esp-coordinator. First PR open, October 2026 | C, has AGENTS.md |
| `dirkwa/signalk-zigbee` | Signal K plugin running Mosquitto + Zigbee2MQTT in a container and mapping devices to Signal K paths. Just started | TS |
| `dirkwa/espos-p4-cockpit-wasm` | Cockpit `widget_factory.cpp` + LVGL in WASM for the designer preview. Committed `public/jlp_wasm.{js,wasm}` | C++/emscripten |
| `dirkwa/signalk-hmi-designer` | Signal K webapp + plugin that designs and pushes JLP layouts | TS/React |
| `dirkwa/signalk-espos-stream` | Signal K plugin: Chromium in a container, MJPEG to the cockpit `stream` widget | TS, container |

Read each repo's `AGENTS.md` first where one exists. Some are partly stale (old
`sensesp-*`, `signalk-esp32-stream` or PlatformIO `src/` names). Trust the tree over
the prose, and fix the prose when you touch it.

The firmware repos copy each other's workflows (`ci.yml`, `release-firmware.yml`,
`track-espos.yml`, `pr-title.yml`). Fixes land in the newest copy first and reach the
older ones later, if at all. Copy from the newest repo (bathy-bridge, or the Zigbee
coordinator once its first PR merges) and compare with the gateway's; `espos-own-firmware` lists the known
differences.

## Where a change belongs

* WiFi, Ethernet, provisioning, config, the device web UI, the Signal K client, mDNS,
  health, logs, core dump, OTA, a P4 radio or esp_hosted Kconfig key: **espOS**, with a
  host test. Board-only things (display, touch, audio driver, LVGL, JLP, N2K wiring,
  the Zigbee NCP) stay in the firmware. A workaround in a firmware names the espOS
  issue or PR and goes when the bump lands.
* **Land espOS first, then bump the submodule** in a separate PR. Never merge a
  consumer PR that depends on unmerged espOS work.
* A consumer pins nothing espOS owns (`esp_hosted`, `esp_wifi_remote`, mDNS). Two exact
  pins on the same component are a hard solve failure (cockpit #158/#159).

## Cross-repo contracts: change in lockstep

| Contract | Sides | Spec |
|---|---|---|
| Device REST `/api/v1` (`system/ping`, `system/info`, `system/coprocessor/update`, `config`, `ota/*`) | espOS ↔ manager, designer (brightness via `PUT /api/v1/config {cockpit:{brightness}}`) | `espOS/docs/rest-api.md` (+ `ui/mock/server.ts`) |
| mDNS `_espos._tcp` TXT `id,v,app,espos,target,api,auth` | espOS ↔ manager | `espos_net`; the `auth` TXT is hard-coded 0, so use `GET /api/v1/system/ping` |
| OTA manifest `schema:1` `{app, builds[{version,target,channel,url,size,sha256,notes,date}]}`, ≤16 KiB | espos_ota ↔ manager mirror | `espOS/docs/ota.md`; manager `src/mirror/version.ts` is a byte-exact port of `espos_ota_version_cmp` (not semver) |
| Registry entry, board catalogue and `index.json` | registry ↔ manager ↔ flasher | `registry/schema/*.schema.json` (mirror `RegistryProject` in the manager); additive changes only, name the manager PR |
| Release asset names | firmware workflows ↔ registry `assets.*` regexes | regexes need `(?<version>)`, plus `(?<board>)` when several boards share a target; `-merged-<size>.bin` images become `mergedByFlash` with no regex change |
| JLP: mDNS `_signalk-player._tcp`, `:8081 /hello /layout /screenshot /healthz` | cockpit ↔ designer ↔ wasm | `espos-p4-cockpit/JLP-PROTOCOL.md`; designer `webapp/src/schema.ts`; the wire is additive or `schema` bumps |
| Stream: TCP 5004 `[u32 BE len][JPEG]` + 1-byte ACK; UDP 5005 touch LE u16 x, u16 y, u8 type | stream plugin ↔ cockpit `stream_client.cpp` | `signalk-espos-stream/AGENTS.md` |
| ZBOSS NCP over TCP 6638, one host at a time; mDNS `_espos-zigbee._tcp`, TXT `radio_type=zboss` (Zigbee2MQTT `port: mdns://espos-zigbee`) | Zigbee coordinator ↔ Zigbee2MQTT `zboss` adapter ↔ signalk-zigbee | coordinator `AGENTS.md`; Zigbee2MQTT needs `ghcr.io/tostmann/zigbee2mqtt-esp32` for the longer TCP timeouts and backups |
| WASM build reads cockpit sources + `managed_components/` | cockpit → wasm → designer (`scripts/copy-wasm.sh`) | wasm shims break when `widget_factory.cpp` includes change; the rebuild is a manual `rebuild-wasm` dispatch |

`app` in the registry is the firmware's CMake `project()` name as reported by
`GET /api/v1/system/ping`. It is not the repo name and not `app_name`.
`boards[].reportedAs` must equal `espos_start_opts_t.board` (`GET /api/v1/system/info`
→ `hardware.board`) exactly; it gates OTA.

## House rules (they differ, so check the repo you are in)

* Conventional Commits everywhere, lower-case subject, ≤50 chars in the firmware repos.
  PR titles become the release notes, so **retitle a bot PR before squash-merging**
  (the cockpit's "dependencies.lock not regenerated" title shipped into its changelog
  even though the lock was fixed).
* Release notes are the ones GitHub generates: each PR's title, author and link, sorted
  by a label that `label-by-title.yml` sets from the title's type (`.github/release.yml`).
  Only `feat`, `fix`, `perf`, `revert`, a breaking change or a `Release-As:` footer
  refreshes the release PR; espOS and bathy-bridge add `refactor`, stream and designer
  `build(deps)`. The gate is in the repo's release workflow and its AGENTS.md.
* **Squash-merge every PR**, the release PR included. release-please reads a merge
  commit and the branch commits under it as separate changes, so a merge commit lists
  the change twice and without its PR number (NMEA gateway 0.2.0).
* **espOS**: every commit signed off (`git commit -s`, DCO); every file has the two SPDX
  lines (`2026 Dirk Wahrheit`, `Apache-2.0`) and `reuse lint` must pass; never edit
  `CHANGELOG.md`; breaking public-header changes bump `ESPOS_ABI_VERSION` and use `!`;
  ask before adding any dependency.
* **espOS release PRs**: the `dependencies.lock` check is red on release-please's first
  commit. That is expected. Merge only once the `release-lock` job has pushed the
  regenerated lock and CI on that commit is green. Merging early broke `main`'s lock
  check after 0.16.0, 0.17.0 and 0.22.0. If the bot's commit starts no CI, close and
  reopen the PR.
* **cockpit, gateways, bathy-bridge, Zigbee coordinator, wasm, designer, stream,
  registry, manager**: no AI attribution anywhere (no `Co-Authored-By`, no "Generated
  with"). This overrides default tool attribution. Never auto-commit or push unless
  asked.
* **Licences differ**: espOS, registry, manager, the gateways, bathy-bridge, the Zigbee
  coordinator, designer and stream are Apache-2.0 (designer 0.2.0 to 0.3.0-beta.2
  shipped source-available; never retag or rewrite them). Cockpit and wasm are
  source-available "no redistribution". Never propose relicensing. Runtime and bundled
  deps must stay permissive everywhere.
* Never commit SSIDs, passwords, IPs, API keys or `*.pem`; espOS CI
  (`scripts/check_no_secrets.sh`) fails on tracked `*.pem`, `secrets/`, `*.nvs.*`.
* No version bumps or release work unless the user says release; release-please owns
  `version.txt`.

## Writing docs

The docs are read by boat owners and by people who have never used ESP-IDF, and a
Signal K developer called the earlier ones machine-written. espOS's `CONTRIBUTING.md`
("Style") sets the voice for every repo:

* Lead with what the reader does or gets, then the reason if they need it. Design
  reasoning goes in `docs/decisions.md`.
* Short sentences, one idea per paragraph. A sentence that needs a dash to hold a
  second thought becomes two sentences.
* A number, a file name or an example rather than "robust", "seamless" or
  "deliberately".
* **Signal K** with a space (code identifiers like `signalk-server` excepted), espOS
  with a small e, signalk-espOS for the whole project. espOS is a firmware base, not
  an operating system.
* Read what you wrote before opening the PR, and cut what a reader would skip.

## Signal K plugin repos (manager, designer, stream, signalk-zigbee)

* Signal K does not apply the config schema's `default`s at runtime: `start()` gets `{}`
  until the user saves the plugin config, so merge defaults (or parse the config) in
  `start()` yourself.
* The App Store installs plugins with `--ignore-scripts`; a dependency that needs an
  install script does not work there.
* The App Store score comes from the Signal K plugin registry's nightly scan. It runs the
  tests from the published npm package first (`npm test`, 60 s cap). Only if they do not
  pass does it clone the default branch HEAD (not the release tag) and test there: the
  declared plugin-ci `test-command` gets 240 s, a plain `npm test` 60 s. A suite that does
  not finish in time counts as failing. It deducts 10 points when it finds no plugin-ci
  run for the npm `gitHead` commit (no gitHead, no run, or runs that don't use the
  reusable plugin-ci workflow); a finished run counts even if matrix jobs failed, which
  the registry reports separately. It also deducts 5 each for no `signalk.screenshots`
  and no CHANGELOG or GitHub Release for the version. To re-score sooner, open a
  "Request a plugin re-score" issue on `SignalK/signalk-plugin-registry`.
* An `E404` from `npm publish` with OIDC means the npm trusted publisher is missing or
  names another workflow. Fix it on npmjs.com, then re-run the failed job; don't re-tag.

## Cloud-session gotchas

* Firmware checkouts arrive with the `espos/` submodule **uninitialised** (empty dir).
  Run `git submodule update --init --recursive` before building or grepping `espos/`. A
  sibling `../espOS` may exist but is not necessarily the pinned commit.
* The Claude GitHub App is not installed on the `SignalK` org, so a cloud session cannot
  open PRs, comment or call the API there. Read an upstream PR with
  `git fetch <remote> refs/pull/<n>/head`. Issue and PR comments render client-side, so
  `curl https://r.jina.ai/https://github.com/SignalK/<repo>/issues/<n>` reads them as
  text.
* No ESP-IDF toolchain is preinstalled. It installs from a git clone of
  `espressif/esp-idf` at the tag in `.idf-version` plus `./install.sh <target>`, and the
  ESP Component Registry was reachable in October 2026. Never claim a build ran when it
  did not. Host-side checks that need no IDF: registry `node scripts/validate.mjs`, the
  Node repos' `npm test`, espOS `python3 -m unittest discover -s tools -p 'test_*.py'`.

## Related

`espos-firmware-project` (building a firmware on espOS), `espos-own-firmware` (the path
for an outside developer), `espos-keys-ota-fleet` (signing, OTA, the registry, the
manager), and the generic ESP-IDF skills in `espos-firmware`.
