---
name: espos-firmware-project
description: Start, build, configure, test and maintain an ESP-IDF firmware on espOS — setting up a new firmware repo (CI, Docker Hub login, repo settings, Dependabot), the CMake prologue, sdkconfig layering, partition tables, the IDF version pin, the locked build wrapper, host tests, bumping the espos submodule and its dependencies.lock, and releasing through espOS's reusable workflows.
---

# A firmware on espOS

Verified against espOS v0.18.0 / ESP-IDF v6.0.3, October 2026. Primary sources in the
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
espos_project_ui_partition()  # after project(), always — see below
```

`espos_project_ui_partition()` is defined by `espos_httpd` (its `project_include.cmake`),
so registry builds have it too. The default bundle is embedded in the app image
(`CONFIG_ESPOS_HTTPD_UI_EMBED`) and also packed into `storage`; OTA never rewrites
`storage`, so a device first flashed before espOS #202 (the embedded UI) keeps its old
page until it takes a release that includes #202.

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

## Setting up a new firmware repo

Start from **dirkwa/espos-ble-gateway**, the most up to date consumer. Copy its
`.github/workflows/` (`ci.yml`, `pr-title.yml`, `release-please.yml`,
`release-firmware.yml`, `track-espos.yml`) plus `release-please-config.json` and
`.release-please-manifest.json`, then rename. Copy the current files rather than writing
them from scratch or from memory.

**CI shape** (`ci.yml`): one matrix leg per target, `submodules: recursive`,
`fetch-depth: 0` (the version comes from `git describe`), the IDF version read from
`.idf-version`, the build through `espressif/esp-idf-ci-action` pinned by SHA. Third-party
actions are pinned by full commit SHA with the tag in a comment.

**Docker Hub login, in every job that runs the espressif/idf container.** Docker Hub
rate-limits anonymous pulls per runner IP, and shared runners hit it ("toomanyrequests").
Put this step before each `esp-idf-ci-action` use:

```yaml
- name: log in to Docker Hub
  if: vars.DOCKERHUB_USERNAME != ''
  continue-on-error: true      # forks and a revoked token pull anonymously, not fail
  uses: docker/login-action@dbcb813823bdd20940b903addbd779551569679f # v4.6.0
  with:
    username: ${{ vars.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}
```

* `DOCKERHUB_USERNAME` is an Actions **variable** holding the **Docker ID** (the name at
  the top of hub.docker.com), not the account email.
* `DOCKERHUB_TOKEN` is an Actions **secret**. Never a variable: variables are plain text
  on the settings page and unmasked in logs. A token found stored as a variable is
  exposed; rotate it. A step's `if:` cannot test a secret, which is why it tests the
  variable.
* The token is a Docker Hub personal access token with **Public Repo Read-only** access
  (Account settings, Personal access tokens). One token can serve every repo. Docker Hub
  shows it once; generate a new one rather than hunting for an old one.
* A login in the caller does not reach a reusable workflow's jobs: espOS's
  `build-firmware.yml` (which the gateway's release calls) has no login step today, so
  those release builds still pull anonymously.
* v3 of the action prints a `punycode` DeprecationWarning; v4 does not. The warning
  is harmless; an empty password shows as `Password required` and only means the
  secret is missing.

**Repo settings before the first release:**
* Actions, General: "Allow GitHub Actions to create and approve pull requests" (else
  release-please fails with zero jobs, and track-espos cannot open its PR).
* The signing key secret (`<PROJECT>_SIGNING_KEY_PEM`), see `espos-keys-ota-fleet`.
* `.github/dependabot.yml` for `github-actions`, weekly, `commit-message.prefix: ci`,
  one group. Without it SHA pins go stale unnoticed.
  Dependabot PRs get no Actions secrets, so their Docker Hub login falls back to
  anonymous; that is what `continue-on-error` is for.

**Scripts:** build with `espos/scripts/build.sh` (or a `scripts/build.sh` shim to it, as
the cockpit has), never bare `idf.py build`. `espos/scripts/regen_lock.sh` and the other
`espos/scripts/check_*` scripts serve espOS's own tree, not a consumer's: a consumer
regenerates its lock with `idf.py reconfigure` (below).

**A target that needs a newer espOS than the pin** gets a configure-time guard after
`project()`, so CI and the release refuse the build rather than ship a broken image:

```cmake
idf_build_get_property(_target IDF_TARGET)
file(STRINGS "${CMAKE_CURRENT_LIST_DIR}/espos/version.txt" _espos_version LIMIT_COUNT 1)
if(_target STREQUAL "esp32c5" AND _espos_version VERSION_LESS "0.18.0")
    message(FATAL_ERROR "my-fw on esp32c5 needs espOS >= 0.18.0; espos is at ${_espos_version}")
endif()
```

That target's CI leg is red until the bump lands; that is the guard working.

## Bumping espOS in a firmware

```bash
git -C espos fetch --tags && git -C espos checkout vX.Y.Z   # --tags or describe drifts
idf.py reconfigure            # regenerates dependencies.lock where it is committed
git commit -am "chore: bump espos to vX.Y.Z"   # "chore!:" when espOS broke API
```

* Cockpit commits `dependencies.lock`; the gateway git-ignores it. Use `reconfigure`,
  **not** `update-dependencies`, so unrelated components do not ride along.
* Cockpit's and the gateway's `track-espos.yml` do this daily from espOS
  `releases/latest` (the gateway's PR is titled `fix: bump espOS to vX.Y.Z`); the
  cockpit's also dispatches a signed firmware build on the branch. Let that PR carry a
  bump rather than pushing the same submodule move into a feature PR. If the cockpit's
  lock solve fails it still opens a PR titled
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
