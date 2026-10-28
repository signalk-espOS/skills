# espOS skills

Claude Code skills distilled from building [espOS](https://github.com/signalk-espOS/espOS),
a firmware base for ESP32 boards that talk to Signal K. They encode what actually went wrong and what the
measurement showed, not a paraphrase of the ESP-IDF docs.

Install:

```
/plugin marketplace add signalk-espOS/skills
/plugin install espos-firmware@signalk-espos
/plugin install espos-dev@signalk-espos
```

`espos-firmware` is not espOS-specific: its three skills are about ESP-IDF, the ESP
Component Registry and GitHub Actions, and apply to any ESP32 firmware project.
`espos-dev` is the opposite: how espOS and its child repos fit together, for anyone
working on them.

## espos-firmware

Three sibling skills, installed together.

### espos-memory-budget

Why a board with 8 MB of PSRAM still exhausts internal RAM, why a classic ESP32
reports internal RAM that `malloc()` cannot get, which pool TLS and the radio really
draw from, and how to size task stacks from a long, busy high-water measurement
instead of a guess. Includes the four PSRAM traps (starting with
`CONFIG_SPIRAM_MALLOC_ALWAYSINTERNAL` defaulting to 16384, which leaves PSRAM
entirely unused), which task stacks can move to PSRAM and which cannot, why IRAM is
heap on a unified-SRAM part like the C5, the per-stage boot budget that turns "it
runs out of memory" into a number you can act on, and keeping an image inside an app
slot that is fixed at the first USB flash.

### espos-registry-components

Why a component's optional dependencies and feature checks silently stop working when
a consumer installs it from the ESP Component Registry rather than a checkout: target
names are namespaced (`myns__mycomp`), so `if(mycomp IN_LIST BUILD_COMPONENTS)` is
false and the feature compiles out with no error. Includes the same defect in IDF's
own `idf_component_optional_requires()`, a namespace-tolerant replacement, and why
your CI probably cannot catch this.

### espos-firmware-releases

Publishing firmware a browser-based flasher can download. GitHub serves release assets
with no CORS header, so a web page cannot fetch them at all; the fix is a mirror
branch, and doing that safely on a shared branch means `--force-with-lease` with a
retry. Includes the GitHub Actions concurrency behaviour that silently drops a release
(and `queue: max`, which stops it), the rollback traps that can delete another
release's firmware, and `sort -V` ranking a prerelease above its release.

## espos-dev

Four sibling skills for working on espOS itself and the projects built on it.

### espos-ecosystem

Which repo owns what (espOS, registry, signalk-espos-manager, the cockpit and its WASM
preview, the BLE and NMEA gateways, the bathy bridge, the Zigbee coordinator, the HMI
designer, the stream plugin), the contracts between them that must change in lockstep,
where a fix belongs, the house rules that differ per repo, and the voice the docs are
written in. Load it first.

### espos-firmware-project

Starting and building a firmware on espOS: the one-time repository setup (secrets,
the Docker Hub login CI needs and the three quiet ways it fails, Dependabot for action
pins, the release App), the CMake prologue and optional components, sdkconfig
layering, partition tables per flash size, the IDF version pin, the locked build
wrapper and a gate for the warnings `idf.py build` lets through, host tests, live and
dated Signal K publishing, bumping the `espos` submodule without breaking
`dependencies.lock`, gating a target on a minimum espOS release, and releasing through
espOS's reusable workflows with GitHub-generated notes.

### espos-own-firmware

The path for a developer outside the project who builds their own firmware on espOS
and ships it to boats: submodule or component registry, their own signing key,
following espOS releases with a tracking workflow (and why Dependabot's submodule
support cannot), which repo's workflow copies carry which fixes, building and
releasing for several chips, boards and flash sizes (including the trap that drops
espOS's defaults from a board variant), listing the firmware in the registry, and what
the README has to tell a boat owner.

### espos-keys-ota-fleet

Signing keys (development, named, CI secret, deliberately unsigned) and why the wrong one
strands OTA; the fleet key, which is an API key and not a signing key, and how the
manager generates, replaces and removes it; the device OTA flow and manifest; the P4's
radio co-processor update; listing a firmware in the registry and getting it indexed
quickly; and how signalk-espos-manager takes a boat from first USB flash to staying
current.

## Versions

Skills state the version they were verified against (ESP-IDF v6.0.3, espOS v0.24.0,
October 2026) wherever a fact can rot: Kconfig defaults, allocation sites, CLI
behaviour. Re-check those before trusting a number on a newer IDF.

## Licence

MIT, see [LICENSE](LICENSE). Contributions welcome; the same bar applies as to the
skills themselves: distilled from real use, and verified against the current version
before being written down.
