---
name: espos-firmware-project
description: Build, configure, test and maintain an ESP-IDF firmware on espOS — the CMake prologue, sdkconfig layering, partition tables, the IDF version pin, the locked build wrapper, host tests, bumping the espos submodule and its dependencies.lock, and releasing through espOS's reusable workflows.
---

# A firmware on espOS

Verified against espOS v0.15.0 / ESP-IDF v6.0.3, October 2026. Primary sources in the
espOS tree: `docs/development.md`, `docs/releasing.md`, `cmake/espos_project.cmake`,
`cmake/espos_version.cmake`, `scripts/build.sh`.

## Two ways to consume espOS

**Submodule (the normal path, all in-house firmware).** `espos/` is a submodule pinned by
SHA. Root `CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.22)
include("${CMAKE_CURRENT_LIST_DIR}/espos/cmake/espos_project.cmake")  # includes IDF's project.cmake — don't include both
espos_project_prologue(NAME "my-firmware"
                       PARTITIONS "${ESPOS_PARTITIONS_DIR}/16mb.csv"
                       COMPONENTS espos_ble espos_eth)              # optional components
project(my_firmware)          # this name is the registry "app" and the OTA project check
espos_project_ui_partition()  # after project(), always — packs the web UI into storage.bin
```

Prologue arguments: `NAME`, `PARTITIONS` (default 4 MB table), `COMPONENTS`,
`SIGNING_KEY`, `PROFILE` (`release`|`debug`, `-DESPOS_PROFILE=` wins),
`IDF_VERSION_FILE`. It sets `PROJECT_VER` from `git describe --tags --dirty` (fallback
`version.txt`), turns on `MINIMAL_BUILD`, manages the signing key and assembles sdkconfig
defaults.

**Optional components are excluded unless named in `COMPONENTS`:** `espos_ble espos_eth
espos_n2k espos_power espos_prov espos_voice espos_audio`. `espos_start()` only starts
what is in the build, so dropping one is silent — no BLE gateway, no Ethernet on a PoE P4.
Also list them in `main/CMakeLists.txt` `PRIV_REQUIRES`.

**Registry (third parties, `espos-release-test`).**
`idf.py create-project-from-example "signalk-espos/espos_core:from_registry"` or
`idf.py add-dependency "signalk-espos/espos_sk^0.15"`. No prologue (`cmake/` ships in no
archive), so the project carries its own `partitions.csv`, `sdkconfig.defaults`,
`.idf-version`, `version.txt` and a hand-made signing key;
`espos_core/project_include.cmake` fails configure and prints the missing lines.

## sdkconfig layering (later wins)

1. `espos/sdkconfig.d/espos.defaults` (+ `.esp32`, `.esp32p4` — P4 file hard-wires the
   Waveshare SDIO pins for the C6 co-processor; other P4 boards must override)
2. `espos/sdkconfig.d/<profile>.defaults` — **`release` burns flash-encryption eFuses,
   one-way.** `debug` adds heap poisoning, stack checks, debug logging.
3. project `sdkconfig.defaults` (+ `.<target>`) — only what differs from espOS
4. `sdkconfig.local` — git-ignored bench overrides (board choice, local tweaks)
5. generated partition + flash-size fragment, then the signing fragment

**Defaults reach a fresh `sdkconfig` only** — delete `build*/sdkconfig` after changing any
of them. An explicit `-DSDKCONFIG_DEFAULTS=` replaces layers 1–4 (only the partition
fragment survives), which silently drops `espos.defaults`; prologue projects pick variants
through `sdkconfig.local` instead.

## Partitions

Bundled tables `${ESPOS_PARTITIONS_DIR}/{4,8,16}mb.csv`: nvs, otadata, phy, nvs_keys,
coredump, `ota_0`/`ota_1`, LittleFS `storage` (web UI). A project table sets **no flash
size**, so a 16 MB custom table also needs `CONFIG_ESPTOOLPY_FLASHSIZE_16MB=y`, or the
4 MB fallback applies. A device keeps its partition table across OTA: changing offsets
in a shipped table strands OTA — that is why the gateway and cockpit keep their own.

## IDF version: a strict pin, not "latest"

`espos/.idf-version` (now `v6.0.3`) is the pin; `cmake/espos_version.cmake` allows
`[ESPOS_IDF_MIN, ESPOS_IDF_MAX_EXCL)` = `[6.0.0, 6.1.0)` with a warning off the exact pin
and a fatal error outside (escape: `-DESPOS_ALLOW_IDF_MISMATCH=1`). A consumer's own
`.idf-version` must equal espOS's. Moving to a newer IDF is its own espOS PR (pin + range
+ `from_registry/.idf-version` + CI); consumers follow by bumping espOS. Nothing watches
for new IDF releases automatically — `dependency-drift.yml` and
`tools/espos_registry_updates.py` cover registry components only.

## Building

```bash
. ~/esp-idf-v6.0.3/export.sh                 # the version in .idf-version
git submodule update --init --recursive      # cloud checkouts arrive with espos/ empty
espos/scripts/build.sh -B build-esp32p4 -DSDKCONFIG=build-esp32p4/sdkconfig \
    -DIDF_TARGET=esp32p4 build               # one build dir per target
espos/scripts/build.sh -B build-esp32p4 -p /dev/ttyACM0 flash monitor
```

Always the wrapper, never bare `idf.py build`: one per-user lock for every espOS project
on the machine, `nice`/`ionice`, half the cores (`BUILD_JOBS`), one job under 3 GB free,
`BUILD_NOWAIT=1` to fail instead of queueing. Warnings are errors. CI needs
`fetch-depth: 0` or `git describe` — and the reported version — breaks.

## Testing

* espOS host tests (linux target, `libbsd-dev`): `test/host/run_all.sh [project…]`;
  `espos_httpd_test/run_test.py` drives the real REST server. Fuzz: `test/fuzz/run.sh`.
* Python tools: `python3 -m unittest discover -s tools -p 'test_*.py'`.
* Format: `scripts/check_format.sh --fix` (clang-format; C 4-space, C++ Google 2-space).
* Firmware repos have no host tests: verify on the device via
  `curl http://<dev>/api/v1/logs?limit=200`, `/api/v1/system/info` (`last_reset`),
  `/api/v1/ota/status`.
* Web UI: `cd espOS/ui && npm ci && npm run dev` (mock server); `npm run build` rewrites
  the committed `components/espos_httpd/ui-dist/`, and CI fails on a stale bundle.
  `docs/rest-api.md` and `ui/mock/server.mjs` change together.

## Bumping espOS in a firmware

```bash
git -C espos fetch --tags && git -C espos checkout vX.Y.Z   # --tags or describe drifts
idf.py reconfigure            # regenerates dependencies.lock where it is committed
git commit -am "chore: bump espos to vX.Y.Z"   # "chore!:" when espOS broke API
```

* Cockpit commits `dependencies.lock`; the gateway git-ignores it. Use `reconfigure`,
  **not** `update-dependencies`, so unrelated components do not ride along.
* Cockpit's `track-espos.yml` does this daily from espOS `releases/latest` and dispatches a
  signed firmware build on the branch. If the lock solve fails it still opens a PR titled
  "…dependencies.lock not regenerated": merge main into the branch, `idf.py reconfigure`,
  commit the lock, **retitle**, then merge.
* Copy `.idf-version` from espOS when it changes.

## Releasing

release-please opens `chore: release x.y.z`; merging it tags and releases, then the
firmware workflow builds, signs and attaches `-merged.bin` (USB, full flash) and
`-ota.bin` per target/board and mirrors them to the orphan `release-assets` branch (CORS,
see `espos-firmware-releases`). Prefer espOS's reusable workflow for new projects:

```yaml
uses: signalk-espOS/espOS/.github/workflows/release-firmware.yml@vX.Y.Z  # pin a tag
with: { name: my-fw, builds: '[{"target":"esp32c6"},{"target":"esp32"}]', tag: "${{ … }}" }
secrets: { signing_key: "${{ secrets.SIGNING_KEY_PEM }}" }  # inherit does not cross orgs
permissions: { contents: write }
```

Assets: `<name>-<board|target>-<tag>-{merged,ota}.bin`. Today the gateway calls
`build-firmware.yml@main` with its own mirror job and the cockpit builds fully inline;
migrating either changes asset names, so update its registry regex in the same breath.
Release repos need "Allow GitHub Actions to create pull requests" on, or release-please
fails with zero jobs. Tags are plain `vX.Y.Z`; betas use the GitHub prerelease flag.

espOS itself releases the same way, then `publish.yml` uploads every component to the ESP
Component Registry via OIDC (`gh workflow run publish.yml -f dry_run=true` to rehearse).
Registry versions are immutable: yank, never delete.

## Related

`espos-ecosystem`, `espos-keys-ota-fleet`, `espos-memory-budget`,
`espos-registry-components`, `espos-firmware-releases`.
