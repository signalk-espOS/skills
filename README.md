# espOS skills

Claude Code skills distilled from building [espOS](https://github.com/signalk-espOS/espOS),
a SignalK runtime for ESP32 devices. They encode what actually went wrong and what the
measurement showed — not a paraphrase of the ESP-IDF docs.

Install:

```
/plugin marketplace add signalk-espOS/skills
/plugin install espos-firmware@signalk-espos
```

Nothing here is espOS-specific in the sense of requiring it. All three skills are
about ESP-IDF, the ESP Component Registry and GitHub Actions, and apply to any ESP32
firmware project.

## espos-firmware

Three sibling skills, installed together.

### espos-memory-budget

Why a board with 8 MB of PSRAM still exhausts internal RAM, which pool TLS and the
radio really draw from, and how to size task stacks from a high-water measurement
instead of a guess. Includes the four PSRAM traps (starting with
`CONFIG_SPIRAM_MALLOC_ALWAYSINTERNAL` defaulting to 16384, which leaves PSRAM
entirely unused), and the per-stage boot budget that turns "it runs out of memory"
into a number you can act on.

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
retry. Includes the GitHub Actions concurrency behaviour that silently drops a release,
and the rollback traps that can delete another release's firmware.

## Versions

Skills state the version they were verified against (ESP-IDF v6.0.3, September 2026)
wherever a fact can rot — Kconfig defaults, allocation sites, CLI behaviour. Re-check
those before trusting a number on a newer IDF.

## Licence

MIT — see [LICENSE](LICENSE). Contributions welcome; the same bar applies as to the
skills themselves: distilled from real use, and verified against the current version
before being written down.
