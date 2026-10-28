---
name: espos-own-firmware
description: The path for a developer outside the espOS project who builds their own firmware on espOS and ships it to boats: choosing submodule or registry, their own signing key, keeping up with espOS releases automatically (track-espos.yml, and why Dependabot cannot do it), building and releasing for several chips and boards, listing the firmware in the signalk-espOS registry, and what the README must tell the boat owner.
---

# Your own firmware on espOS, from repo to boat

Verified against espOS v0.24.0, registry `schema: 1`, and the firmware repos'
workflows as of late October 2026. This is the order of work for someone outside the
project; `espos-firmware-project` has the detail behind each step and
`espos-keys-ota-fleet` the signing and OTA side. Two reference firmwares run all of
this in production:

* [dirkwa/espos-ble-gateway](https://github.com/dirkwa/espos-ble-gateway): five
  chips, one image per chip, 8 and 16 MB layouts for the ESP32, S3 and C6 dev kits,
  espOS's shared `build-firmware.yml`. **Start here** for the project layout.
* [dirkwa/espos-p4-cockpit](https://github.com/dirkwa/espos-p4-cockpit): one chip,
  two display boards, one image per board, a fully inline release job.

For the workflow files themselves (`ci.yml`, `track-espos.yml`, `pr-title.yml`,
`label-by-title.yml`), copy the newest repo,
[dirkwa/espos-bathy-bridge](https://github.com/dirkwa/espos-bathy-bridge), or the
Zigbee coordinator once its first PR merges. The repos copy each other and the older
copies miss fixes found later:

| Fix | Where it is | Missing from |
|---|---|---|
| `track-espos.yml` falls back to the workflow token when the App is not installed on the repo (the key alone answers 404 and failed every run) | nmea-gateway #16, bathy-bridge #9 | ble-gateway, cockpit |
| `track-espos.yml` closes a prerelease bump of the tag it now tracks (`sort -V` ranks `v1.2.0-rc1` above `v1.2.0`) | bathy-bridge #1 | ble-gateway, nmea-gateway, cockpit |
| `ci.yml` fails on CMake and linker warnings, which `idf.py build` lets through | bathy-bridge #1 | ble-gateway, nmea-gateway, cockpit |
| `pr-title.yml` enforces the 50-character subject for people and exempts bots, whose titles cannot be shortened | bathy-bridge #5 | the others check only the leading capital |
| the mirror's README is written after the branch is restored, so it names the new tag | bathy-bridge #1 | ble-gateway, nmea-gateway |
| the `release-assets` rollback deletes the branch through the API, since the release job has no checkout and `git push` there fails | Zigbee coordinator #1 | ble-gateway, nmea-gateway, bathy-bridge |

None of them releases through espOS's `release-firmware.yml` yet (they predate it), and
their release workflows are tied to the project: a project-named signing secret
reached through `secrets: inherit`, a `notify-registry` job that needs the org's
release App. For your release, use `release-firmware.yml` as in section 4 rather than
copying theirs.

A third-party project already listed in the registry: `BoatHacks/signalk-espOS-8relay`.

## 1. Pick how you consume espOS

| | Submodule (`espos/`, pinned by SHA) | Component registry (`signalk-espos/espos_*`) |
|---|---|---|
| Start | copy the gateway's `CMakeLists.txt`, `main/`, `partitions*.csv`, `sdkconfig.defaults*`, `version.txt`, `.idf-version`, `.gitignore` (keeps `*.pem` and `sdkconfig.local` out of git), `scripts/` | `idf.py create-project-from-example "signalk-espos/espos_core:from_registry"` |
| Prologue (sdkconfig layering, signing key, `PROJECT_VER`) | yes | no: you carry `partitions.csv`, `sdkconfig.defaults`, `.idf-version`, `version.txt` yourself |
| Following espOS | `track-espos.yml` opens a PR per release (below) | bump the `^0.x` range by hand |
| Board variants via the shared release workflow | from v0.19.0 (espOS #221); not before (trap below) | yes |

Take the submodule unless you have a reason not to: every in-house firmware does,
and the automation below assumes it. Add it at the latest release **tag**:

```bash
git submodule add https://github.com/signalk-espOS/espOS espos
git -C espos fetch --tags && git -C espos checkout v0.24.0
cp espos/.idf-version .          # must equal espOS's pin: configure fails when they differ, and CI reads it
```

`project(<name>)` in your root `CMakeLists.txt` is your firmware's identity: the
registry's `app`, the OTA project check, what `GET /api/v1/system/ping` reports.
Pick it once; renaming it later strands every device in the field.

## 2. Your own signing key, before the first release

Devices only take an OTA signed with the key they were flashed with. Generate one,
keep it outside the repo, back it up, and store it as a secret:

```bash
espsecure generate-signing-key --version 2 --scheme rsa3072 signing_key.pem
gh secret set SIGNING_KEY_PEM < signing_key.pem
```

Lose it and every board you shipped needs a USB cable. A release built without it
installs and then refuses every update; espOS's release workflow refuses to build one
unless the repository variable `ESPOS_ALLOW_UNSIGNED_RELEASE` is `true`. Record the key's fingerprint as
`signingKeyId` in the registry entry (`espos-keys-ota-fleet` has the command).

Also turn on **Settings → Actions → General → "Allow GitHub Actions to create and
approve pull requests"** (release-please and the tracking workflow open PRs; without it
they fail with zero jobs), and optionally the Docker Hub variable/secret pair from
`espos-firmware-project`.

Four files the reference repos have and a new repo forgets:

* **`.github/workflows/pr-title.yml`**: the gateway's checks titles against
  Conventional Commits, because release-please writes the changelog from them. Its
  `subjectPattern: ^(?![A-Z]).+$` also rejects a subject that starts with a capital
  letter: `feat: NMEA 2000 support` fails, `feat: add NMEA 2000 support` passes.
  Keep the rule or delete the pattern, but know it is there before your first PR.
* **`AGENTS.md`** plus a one-line **`CLAUDE.md`** (`@AGENTS.md`): the repo's rules
  for coding agents: build only through the wrapper, never commit secrets or boat
  configuration, commit and PR conventions, and whether AI attribution is allowed.
  Without it an agent follows its own defaults, attribution trailers included. The
  cockpit's is the model.
* **`.coderabbit.yaml`**: without it CodeRabbit reviews with its defaults. Copy the
  cockpit's and change its `path_filters` to your generated files (`managed_components/`,
  `build*/`, `*.bin`); its `sdkconfig.p4_cockpit*` entries are leftovers from 1.x.
  Make CodeRabbit skip release-please's PRs, as espOS does: they only bump the version
  and changelog of changes already reviewed. The cockpit's `"chore(release):"`
  keyword does not match release-please's `chore: release x.y.z` titles, so use:
  ```yaml
  reviews:
    auto_review:
      ignore_title_keywords:
        - "chore: release"
  ```
* **`.github/workflows/label-by-title.yml`** and **`.github/release.yml`**: the release
  notes GitHub generates are sorted by label, and this sets the label from the PR
  title's type. Without it every entry lands under one heading
  (`espos-firmware-project`, "Releasing", has the rest of the release-please setup,
  including `"initial-version": "0.1.0"` for a first release below 1.0).

## 3. Stay on the latest espOS automatically

### Why not Dependabot for the submodule

Dependabot has a `gitsubmodule` ecosystem, and it is the wrong tool here: it moves the
submodule to the **tip of espOS `main`**, not to release tags. You would ship
unreleased espOS, the build would report `v0.24.0-12-gabc1234`, and the registry
(which matches the pin against espOS tags exactly) records no espOS version for your
release at all. Do not add a `gitsubmodule` entry.

### `track-espos.yml`: one PR per espOS release

Copy bathy-bridge's
[`.github/workflows/track-espos.yml`](https://github.com/dirkwa/espos-bathy-bridge/blob/main/.github/workflows/track-espos.yml)
unchanged except as noted (the gateway's copy lacks the App fallback). Daily, it:

1. reads `repos/signalk-espOS/espOS/releases/latest` (stable only; a prerelease can
   be taken by naming the tag in a manual run);
2. skips if the pin already is that tag, if a PR for it was ever opened (closed
   counts: a declined bump is not reopened daily), or if the tag does not contain
   the current pin (never moves backwards on its own);
3. checks the submodule out at the tag, copies `espos/.idf-version` when ESP-IDF
   moved, and lists lines in `CMakeLists.txt`/`README.md` that still name the old IDF;
4. opens `fix: bump espOS to vX.Y.Z` from branch `track-espos-vX.Y.Z`; `fix:` so
   release-please proposes a patch release of your firmware once it merges;
5. closes older tracking PRs it supersedes, deleting their branches unless someone
   pushed work onto them.

It stops at the PR: a bump changes the binary on a boat, so a person merges it after
CI builds every target. What to adapt:

* **`client-id:`** is the signalk-espOS org's release App. Without
  `RELEASE_APP_PRIVATE_KEY`, or with the key but no installation of that App on your
  repo, the PR is opened with the workflow token (a warning says which). That works, but GitHub does not start a PR's checks for an event
  the workflow token caused: approve them if GitHub offers **Approve workflows to
  run**, or close and reopen the PR yourself. For checks that start by themselves, create
  your own GitHub App (contents + pull requests: write), install it on the repo,
  put its Client ID there and its key in `RELEASE_APP_PRIVATE_KEY`.
* **Committed `dependencies.lock`?** Borrow the cockpit's "Regenerate
  dependencies.lock" step (`idf.py reconfigure`, then `git add` the lock with
  `espos` and `.idf-version`). Leave out the rest of its version: it also dispatches
  its own release workflow on the tracking branch, which needs `actions: write`, a
  `workflow_dispatch` trigger and its signing secret. Simpler still is the
  gateway's choice, a git-ignored lock: nothing to regenerate.
* `--base main`: change it if your default branch is `master`.

Read the espOS release notes on each tracking PR: a green build proves it compiles,
not that it behaves the same.

### Dependabot for everything else

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule: { interval: weekly }
    commit-message: { prefix: ci }
    groups:
      actions: { patterns: ["*"] }
```

This also moves a tag-pinned reusable workflow
(`uses: signalk-espOS/espOS/.github/workflows/release-firmware.yml@v0.24.0`) to newer
espOS tags, so the build recipe follows espOS like the submodule does. The two arrive
as separate PRs; merge them in the same go, or move the `uses:` pin by hand in the
tracking PR, so the recipe and the compiled espOS stay on one release. Pin third-party
actions by SHA; Dependabot rewrites SHA pins and keeps the tag in the comment.

ESP-IDF component manager dependencies are not a Dependabot ecosystem. The cockpit's
`dependency-drift.yml` re-resolves weekly with `idf.py update-dependencies` and opens
an issue when `dependencies.lock` would change. A **registry consumer** has no
automation for espOS itself: a `^0.24.0` range never reaches 0.25.0 while espOS is
below 1.0, so watch espOS releases (GitHub **Watch → Custom → Releases**) and bump the
range by hand.

## 4. Several chips and boards

Decide per chip first, because the registry and the device both care:

| Situation | Images | Registry entry |
|---|---|---|
| One board per chip (C3, C6, S3 devkits) | one per chip, segment = target | `assetSegment` = target on each board |
| Several boards, one chip, **one image runs on all** | one | one board entry with `runsOn: [ …catalogue ids… ]`, `assetSegment` = target |
| Several boards, one chip, **different images** | one per board | `assetSegment` = the board segment on each board |

Whenever a release carries more than one image, the `assets` patterns need a
`(?<board>…)` group capturing the segment; without it the reindexer cannot tell the
images apart and indexes none of them.

Prefer one image per chip: espOS matches an OTA on app, target and channel, never on
board, so per-board images on one chip rely on the registry and `reportedAs` to keep
the wrong image off a board. The gateway runs one P4 image on three boards.

### Per-chip differences in one tree

* `sdkconfig.defaults.<target>`: IDF appends it after `sdkconfig.defaults`
  automatically (the gateway's `sdkconfig.defaults.esp32c5`, `.esp32p4`).
* A partition table per chip only when a chip needs different slots. Choose it
  before the prologue, as the gateway does for the C5's bigger app slots. Different
  **flash sizes** need no table of your own: release with `flash_tables` (below and
  `espos-firmware-project`, "Partitions"):
  ```cmake
  set(_partitions "${CMAKE_CURRENT_LIST_DIR}/partitions.csv")
  if(IDF_TARGET STREQUAL "esp32c5")
      set(_partitions "${CMAKE_CURRENT_LIST_DIR}/partitions-esp32c5.csv")
  endif()
  espos_project_prologue(NAME "my-fw" PARTITIONS "${_partitions}")
  ```
  A device keeps its partition table across OTA; a changed table reaches only
  boards flashed over USB.
* A chip that needs a newer espOS: gate it with `FATAL_ERROR` on
  `espos/version.txt` (`espos-firmware-project`), and set the board's `minVersion`
  in the registry to the first release whose image works.
* Locally, one build directory per target:
  `espos/scripts/build.sh -B build-esp32c6 -DSDKCONFIG=build-esp32c6/sdkconfig -DIDF_TARGET=esp32c6 build`.

### CI and release

PR CI: bathy-bridge's `ci.yml`, a `fail-fast: false` matrix over targets, each leg
`espressif/esp-idf-ci-action` with `.idf-version`, `fetch-depth: 0` (or the version
baked in is wrong), `test -s build/storage.bin` so a missing web UI fails the PR, and a gate on the
build log for the warnings `idf.py build` does not fail on.
These builds use a throwaway key; never give `pull_request` jobs the signing key.

Release: espOS's `release-firmware.yml` builds a list, attaches `-merged.bin` and
`-ota.bin` per entry, and mirrors them to `release-assets` for the browser flasher.
The job after release-please (full workflow in espOS `docs/releasing.md`):

```yaml
firmware:
  needs: release-please
  if: needs.release-please.outputs.release_created == 'true'
  permissions:
    contents: write
  uses: signalk-espOS/espOS/.github/workflows/release-firmware.yml@v0.24.0
  with:
    name: my-fw
    builds: '[{"target":"esp32c6"},{"target":"esp32s3"},{"target":"esp32c5"}]'
    tag: ${{ needs.release-please.outputs.tag_name }}
  secrets:
    signing_key: ${{ secrets.SIGNING_KEY_PEM }}
    dockerhub_username: ${{ vars.DOCKERHUB_USERNAME }}
    dockerhub_token: ${{ secrets.DOCKERHUB_TOKEN }}
```

**One image per flash size.** Dev kits are sold with 4 to 32 MB, and a partition
table is fixed to one size. Add `"flash_tables"` to a build and the same build is laid
out once per table, with no extra compile:

```yaml
builds: >-
  [{"target": "esp32c6", "flash_tables": "4mb 8mb 16mb"},
   {"target": "esp32c5", "flash_tables": "16mb"}]
```

Each size adds `my-fw-<board|target>-vX.Y.Z-merged-<size>.bin`; the registry lists
them as the build's `mergedByFlash` without any change to your entry, and the flasher
writes the largest the chip holds. Leave out a size whose app slots your image would
fill past ~90 % (the build fails outright if it does not fit), and set the board's
`flashMinBytes` in the registry to the smallest size you ship. The `uses:` pin must be
v0.19.0 or newer (espOS #223): `v0.18.1` rejects `flash_tables` as an unknown key and
stops the release before any build runs. Bring your own CSVs only for an extra
partition, the same size in every table, as the Zigbee coordinator does for the
Zigbee stack's `zb_nvs`, `zb_storage` and `zb_fct`. A network backup and restore then
reads and writes those, never espOS's `nvs`.

Assets: `my-fw-<board|target>-vX.Y.Z-{merged,ota}.bin`. The Docker Hub secrets need
v0.19.0 or newer; a v0.18.1 pin rejects them as undeclared.

**Board variants.** `builds` takes `{"target","board","sdkconfig_defaults"}`, one
defaults file per variant:

```yaml
builds: >-
  [{"target": "esp32p4", "board": "7b", "sdkconfig_defaults": "sdkconfig.board.7b"},
   {"target": "esp32p4", "board": "x7", "sdkconfig_defaults": "sdkconfig.board.x7"}]
```

From v0.19.0 (espOS #221) this is safe on a submodule
project: the workflow passes the files as `ESPOS_EXTRA_SDKCONFIG_DEFAULTS`, which the
prologue applies after `espos.defaults` and your `sdkconfig.defaults` (a developer's
`sdkconfig.local` still wins), and a step after the build stops the release unless
every `CONFIG_` line of the variant's files is in the result. Both halves have to be
that new: the `uses:` pin, which carries the workflow, and the `espos` submodule,
which carries the prologue. An older submodule ignores the variable, every variant
comes out the same, and the check fails the release, which is the point. Locally:
`idf.py -B build-x7 -DESPOS_EXTRA_SDKCONFIG_DEFAULTS=sdkconfig.board.x7 build`, one
build directory per board (the variable stays in its CMake cache) and a fresh
`sdkconfig`.

**Pinned to v0.18.1 or older, it is a trap.** There `sdkconfig_defaults` is passed as
`-D SDKCONFIG_DEFAULTS="sdkconfig.defaults;<yours>"`, and the prologue treats an
explicit `SDKCONFIG_DEFAULTS` as "the caller chose" and **drops `espos.defaults`,
its `.<target>` file and the profile** (the signed-update settings and task stacks
among them), keeping only the partition fragment. The build succeeds, with an image
that is not signed the way your boats expect. Either move both pins past v0.18.1, or
do what the cockpit does in its own release job: write `CONFIG_<YOUR>_BOARD_<X>=y`
into `sdkconfig.local` per matrix leg, fail the leg unless `build/config/sdkconfig.h`
defines that symbol, and attach the images and push the `release-assets` mirror
yourself (`espos-firmware-releases`). A registry-consumer project (no prologue) was
never affected. The cockpit's names put the board after the version,
`p4_cockpit-v1.5.5-7b-merged.bin`, so its registry regex differs from the one in
section 5.

With per-board images, also set `espos_start_opts_t.board` to a fixed string per
variant and put it in the registry as `reportedAs`, matched exactly; that is what lets
the manager refuse an OTA meant for the other board.

## 5. List it in the registry

Fork `signalk-espOS/registry`, add `projects/<id>.json` (copy
`projects/8relay.json` for one board, `ble-gateway.json` for several chips), open a
PR titled `add: <id>`. The fields that decide whether boats ever see an update:

* `app`: your CMake `project()` name, checked against `GET /api/v1/system/ping` on a
  running device. Not the repo name.
* `assets.ota` / `assets.merged`: anchored regexes with `(?<version>…)`, plus
  `(?<board>…)` when the segment names the board. For `release-firmware.yml`'s names:
  `"^my-fw-(?<board>[a-z0-9]+)-v(?<version>[0-9]+\\.[0-9]+\\.[0-9]+)-ota\\.bin$"`.
* `boards[]`: a catalogue id from `boards/` where one fits; the entry then needs
  only what is particular to your image (`target`, `assetSegment`, `reportedAs`,
  `flashMinBytes`, `notes`, `support`, `minVersion`). `support: "tested"` only for
  a board on your bench, as tested boards are offered first. A board missing from the
  catalogue can be added as `boards/<id>.json` in the same PR (only boards someone
  has run espOS firmware on), or described inline with its own `name` and `notes`.
  Never touch `index.json`; CI rebuilds it.
* `webAssetsBranch: "release-assets"`: without it the hosted flasher cannot download
  your images.
* `signingKeyId`, `license`, `channels` (`["stable","beta"]` if you ship prereleases).
* A use case in `usecases/` lists project ids; ask in the PR to be added to the one
  your firmware does, which is how owners find it from the flasher's start screen.

Run `node scripts/validate.mjs` (Node 24, `GH_TOKEN` set) before opening the PR; CI
shows how many releases and builds the entry resolves to. Zero means the regex does
not match your asset names. After merge, new releases are picked up by the hourly
reindex: the instant `firmware-released` dispatch the official repos send needs write
access to the registry, which outside projects do not have. If a release is missing
hours later, ask a maintainer to run the Reindex workflow.

## 6. What the README tells the boat owner

The manager's store and the hosted flasher link a project to
`https://github.com/<repo>`, so your README's first screen is the user manual; the
registry's `summary`/`description` are only the teaser. Write it for someone with a
board and a Signal K server, not a toolchain, and put the build instructions last:

1. **What it does**, in a paragraph: what appears in Signal K, and what it needs on
   the server side (a plugin, a provider).
2. **Hardware**: a table of boards with an honest status (in use / tested on the
   bench / builds only) matching the registry's `support`, flash size, and anything
   that kills a board or makes the radio deaf, such as the gateway's warning that
   the C6-MINI-1U module hears nothing without an external antenna (the registry's
   P4 PoE board notes warn against powering it from USB and PoE together).
3. **Wiring**: pinout, power, CAN or sensor connections, with a photo or diagram.
   This is the part only you can write.
4. **Install**: the [hosted flasher](https://signalk-espos.github.io/signalk-espos-manager/flash/)
   in Chrome or Edge over USB, or the Flash page of the signalk-espos-manager plugin;
   for manual flashing, the `-merged.bin` at offset `0x0`.
5. **First setup**: the `espOS-xxxx` WiFi portal, approving the access request in
   Signal K under Security → Access Requests, and your settings namespace in the
   device's web UI.
6. **Updates**: over the air through signalk-espos-manager; a board that ran a
   different project's firmware (another signing key) needs one USB flash first; a
   partition-table change reaches only USB-flashed boards.
7. **When it misbehaves**: the status endpoint, `http://<device>/api/v1/logs`, and
   where to file issues.

The gateway's README is the model for 1, 2, 5 and the status section; it does not
yet have an install section, so take 4 from here.

## Related

`espos-firmware-project` (CMake prologue, sdkconfig layering, CI secrets, bumping by
hand), `espos-keys-ota-fleet` (signing keys, OTA, the manager), `espos-ecosystem`,
and `espos-firmware-releases` in the `espos-firmware` plugin (why the
`release-assets` mirror exists).
