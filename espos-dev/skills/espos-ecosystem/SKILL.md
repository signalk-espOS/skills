---
name: espos-ecosystem
description: Orientation for working on espOS or any of its child repos (registry, signalk-espos-manager, espos-p4-cockpit, espos-ble-gateway, the cockpit WASM, signalk-hmi-designer, signalk-espos-stream) — which repo owns what, the contracts between them, and the house rules that differ per repo. Load first in any espOS session.
---

# The espOS ecosystem

Snapshot: espOS **v0.15.0** on ESP-IDF **v6.0.3**, October 2026. Versions move; read
`version.txt` and `.idf-version` in the checkout instead of trusting the numbers here.

espOS is the ESP-IDF-native successor to SensESP: a small device runtime (WiFi /
Ethernet, provisioning, config store + web UI, SignalK discovery / token / delta stream /
PUT / REST, mDNS, health watchdog, logs, core dump, signed OTA) that a firmware brings up
with one `espos_start()` call. Firmware projects sit on top; a registry lists their
releases; a SignalK plugin keeps a boat's devices updated from it.

## Repos and what each owns

| Repo | Role | Language / gate |
|---|---|---|
| `signalk-espOS/espOS` | **Primary.** Runtime components `components/espos_*`, CMake prologue, sdkconfig layers, partitions, reusable release workflows, docs site | C11 (+C++ for n2k/voice/audio). `test/host/run_all.sh`, `scripts/build.sh` |
| `signalk-espOS/registry` | `projects/<id>.json` per firmware → CI-generated `index.json` | Node 24 scripts, no deps |
| `signalk-espOS/signalk-espos-manager` | SignalK plugin: mDNS discovery, registry client, firmware mirror, OTA jobs, fleet API key; plus the hosted browser flasher (GitHub Pages) | TS + Preact. `npm run format && npm run build && npm test` |
| `signalk-espOS/espos-release-test` | Fixture for espOS's reusable `release-firmware.yml`; also the registry-consumer template (no submodule) | C |
| `signalk-espOS/skills` | These skills | Markdown |
| `dirkwa/espos-p4-cockpit` | ESP32-P4 panel firmware: JSON Layout Player (LVGL), N2K gateway, voice | C/C++, has AGENTS.md |
| `dirkwa/espos-ble-gateway` | BLE → signalk-server BLE provider API. `main.c` is one `espos_start(NULL)`; logic lives in espOS `espos_ble` | C, no AGENTS.md |
| `dirkwa/espos-p4-cockpit-wasm` | Cockpit `widget_factory.cpp` + LVGL in WASM for the designer preview. Committed `public/jlp_wasm.{js,wasm}` | C++/emscripten |
| `dirkwa/signalk-hmi-designer` | SignalK webapp + plugin that designs/pushes JLP layouts | TS/React |
| `dirkwa/signalk-espos-stream` | SignalK plugin: Chromium-in-container → MJPEG to the cockpit `stream` widget | TS, container |

Read each repo's `AGENTS.md` first where one exists. Several are partly stale (old
`sensesp-*` / `signalk-esp32-stream` / PlatformIO `src/` names); trust the tree over the
prose and fix the prose when you touch it.

## Where a change belongs

* WiFi, Ethernet, provisioning, config, the device web UI, SignalK client, mDNS, health,
  logs, core dump, OTA, a P4 radio/hosted Kconfig key → **espOS**, with a host test.
  Board-only things (display, touch, audio driver, LVGL, JLP, N2K wiring) stay in the
  firmware. A workaround in a firmware must name the espOS issue/PR and be removed when
  the bump lands.
* **Land espOS first, then bump the submodule** in a separate `chore: bump espos to vX`
  PR. Never merge a consumer PR that depends on unmerged espOS work.
* A consumer pins nothing espOS owns (`esp_hosted`, `esp_wifi_remote`, mDNS). Two exact
  pins on the same component are a hard solve failure (cockpit #158/#159).

## Cross-repo contracts — change in lockstep

| Contract | Sides | Spec |
|---|---|---|
| Device REST `/api/v1` (`system/ping`, `system/info`, `config`, `ota/*`) | espOS ↔ manager, designer (brightness via `PUT /api/v1/config {cockpit:{brightness}}`) | `espOS/docs/rest-api.md` (+ `ui/mock/server.mjs`) |
| mDNS `_espos._tcp` TXT `id,v,app,espos,target,api,auth` | espOS ↔ manager | `espos_net`; the `auth` TXT is hard-coded 0 — use `GET /api/v1/system/ping` |
| OTA manifest `schema:1` `{app, builds[{version,target,channel,url,size,sha256,notes,date}]}`, ≤16 KiB | espos_ota ↔ manager mirror | `espOS/docs/ota.md`; manager `src/mirror/version.ts` is a byte-exact port of `espos_ota_version_cmp` (not semver) |
| Registry entry + `index.json` | registry ↔ manager ↔ flasher | `registry/schema/project.schema.json` ("mirrors RegistryProject in the manager"); schema changes additive only, name the manager PR |
| Release asset names | firmware workflows ↔ registry `assets.*` regexes | regexes need `(?<version>)`, `(?<board>)` when several boards share a target |
| JLP: mDNS `_signalk-player._tcp`, `:8081 /hello /layout /screenshot /healthz` | cockpit ↔ designer ↔ wasm | `espos-p4-cockpit/JLP-PROTOCOL.md`; designer `webapp/src/schema.ts`; wire is additive or `schema` bumps |
| Stream: TCP 5004 `[u32 BE len][JPEG]` + 1-byte ACK; UDP 5005 touch LE u16 x,u16 y,u8 type | stream plugin ↔ cockpit `stream_client.cpp` | `signalk-espos-stream/AGENTS.md` |
| WASM build reads cockpit sources + `managed_components/` | cockpit → wasm → designer (`scripts/copy-wasm.sh`) | wasm shims break when `widget_factory.cpp` includes change; rebuild is a manual `rebuild-wasm` dispatch |

`app` in the registry is the firmware's CMake `project()` name as reported by
`GET /api/v1/system/ping` — not the repo name, not `app_name`. `boards[].reportedAs` must
equal `espos_start_opts_t.board` (`GET /api/v1/system/info` → `hardware.board`) exactly; it
gates OTA.

## House rules (they differ — check the repo you are in)

* Conventional Commits everywhere, lower-case subject, ≤50 chars in the firmware repos.
  PR titles become release-please changelog lines — **retitle a bot PR before
  squash-merging** (cockpit's "dependencies.lock not regenerated" title shipped into the
  changelog even though the lock was fixed).
* **espOS**: every commit signed off (`git commit -s`, DCO); every file has the two SPDX
  lines (`2026 Dirk Wahrheit`, `Apache-2.0`) and `reuse lint` must pass; never edit
  `CHANGELOG.md`; breaking public-header changes bump `ESPOS_ABI_VERSION` and use `!`;
  ask before adding any dependency.
* **cockpit, gateway, wasm, designer, stream, registry, manager**: no AI attribution
  anywhere — no `Co-Authored-By`, no "Generated with" — this overrides default tool
  attribution. Never auto-commit or push unless asked.
* **Licences differ**: espOS, registry, gateway are Apache-2.0; cockpit, wasm, designer
  (≥0.2.0), stream are source-available "no redistribution". Never propose relicensing.
  Runtime/bundled deps of the source-available repos must stay permissive.
* Never commit SSIDs, passwords, IPs, API keys or `*.pem`; espOS CI
  (`scripts/check_no_secrets.sh`) fails on tracked `*.pem`, `secrets/`, `*.nvs.*`.
* No version bumps or release work unless the user says release; release-please owns
  `version.txt`.

## Cloud-session gotchas

* Firmware checkouts arrive with the `espos/` submodule **uninitialised** (empty dir).
  `git submodule update --init --recursive` before building or grepping `espos/`; a sibling
  `../espOS` may exist but is not necessarily the pinned commit.
* No ESP-IDF toolchain is preinstalled; say so rather than claiming a build ran.
  Host-side checks that need no IDF: registry `node scripts/validate.mjs`, the Node
  repos' `npm test`, espOS `python3 -m unittest discover -s tools -p 'test_*.py'`.

## Related

`espos-firmware-project` (building a firmware on espOS), `espos-keys-ota-fleet`
(signing, OTA, registry listing, the manager), and the generic ESP-IDF skills in
`espos-firmware`.
