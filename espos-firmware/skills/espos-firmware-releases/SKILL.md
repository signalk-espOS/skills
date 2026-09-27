---
name: espos-firmware-releases
description: Publish firmware a browser-based flasher can actually download — why GitHub release assets are unreachable from a web page, how to mirror them to a branch safely with force-with-lease, and the GitHub Actions concurrency behaviour that silently drops a release.
---

# Publishing firmware a web flasher can fetch

Verified against GitHub Actions, September 2026.

## Why release assets do not work in a browser

**GitHub serves release downloads with no `Access-Control-Allow-Origin` header** — on
the `302` and on the redirect target — so a web page cannot fetch them at all. A
browser-based flasher (esptool-js, ESP Web Tools) pointed at a
`github.com/.../releases/download/...` URL fails with a CORS error, not a 404, and no
amount of client-side code fixes it.

`raw.githubusercontent.com` **does** send the header and honours `Range` requests. So
the working pattern is: keep the release as the archive, and mirror the same images to
an orphan branch laid out as `<tag>/<asset>`. Server-side consumers keep using the
release URL, which CORS does not affect.

Two consequences worth planning for:

* **Ordering.** If your catalogue/index derives a mirror URL by rewriting the release
  URL, it will publish links that 404 until the branch exists. Ship the workflow first,
  cut one release to populate the branch, *then* declare the mirror in the index.
* **Mirror everything the index will advertise.** If the index emits both a "merged"
  and an "ota" web URL by rewriting, mirroring only the one your flasher reads leaves
  the other 404ing. Cheaper to mirror both than to make the index verify existence.

## Size the branch deliberately

The branch is force-pushed as one commit with no history, so a clone is one snapshot
rather than the sum of every release — but it is still a branch everyone who clones the
repo may fetch. Firmware is large: two board variants × (15 MB merged + 4.5 MB OTA) is
~39 MB **per tag**. Keep the newest few and prune, and say in the branch's own README
what the retention actually is. ("Holds only the current one" while the code keeps
three is the kind of comment that outlives its truth.)

## `--force-with-lease`, not `--force`

A force-push to a shared branch loses whatever another run published in between. Use a
lease against the tip you actually read, and retry from the new tip:

```bash
git fetch -q --depth 1 "$remote" mirror-branch
git checkout -q FETCH_HEAD -- .
base=$(git rev-parse FETCH_HEAD)      # what the lease will assert

stage_and_push() {
  git add -A
  git commit -q -m "firmware for $TAG"
  # Record BEFORE pushing: a run cancelled mid-push may have landed the commit,
  # and any rollback needs this SHA. Recording one that never arrived is harmless —
  # the rollback's lease is then stale, so it is refused.
  git rev-parse HEAD > "$MARKER"
  if [ -n "${base:-}" ]; then
    git push -q --force-with-lease="mirror-branch:${base}" "$remote" HEAD:mirror-branch
  else
    # First publish: an empty lease means "expect the branch not to exist".
    git push -q --force-with-lease="mirror-branch:" "$remote" HEAD:mirror-branch
  fi
}
```

### Retry only when the branch actually moved

Every push failure is not a lease rejection. A network blip, a revoked token or a
protected branch fails the same command, and looping over one of those ends in a
message blaming concurrency for something else. Ask the remote:

| remote tip after the failure | meaning | action |
|---|---|---|
| unreadable | not a lease rejection | fail, do not retry |
| still `$base` | nothing moved | fail; look for auth/network/hook |
| equals your own commit | the push landed, error came after | treat as success |
| some other commit | genuine rejection | re-read, re-stage, retry |

Cap the retries (three is plenty) and fail rather than loop.

### Re-stage through one function

The retry must redo exactly what the first attempt did — copy the README, replace this
tag's directory, re-prune. Two copies of that logic drift the moment one is fixed, so
make it one function called from both places.

Two bugs that only appear on the *second* retry, both found by testing rather than
reading:

* `git reset --soft HEAD~1` fails on the first attempt — that commit has no parent.
  Use `git checkout --orphan` instead; a wholesale-replacement push wants no history.
* `git checkout --orphan fixed-name` fails the **second** time with
  `a branch named 'fixed-name' already exists`. Make the name unique per attempt.

And one that only appears if the README lives inside the work tree: if you write it to
`$work/README.md` and then `cd "$work"`, a retry's `find . -exec rm -rf` deletes it and
the restore copies a file onto itself. Stage it outside the repo (`RUNNER_TEMP`).

## Rolling back without eating someone else's release

If publishing the release assets fails *after* the mirror pushed, the branch advertises
a tag whose release has none. Undoing that is right — but the undo must also be leased,
against the commit **this run** pushed. Otherwise a run that failed late will flatten a
release that published after it.

The trap is subtler than it looks: **re-take the snapshot every time you re-read the
branch.** A single snapshot taken before the first attempt goes stale the moment a
retry moves `base`, and then the rollback restores a state from before another run's
tag existed — deleting their directory while their release still points at it.
Reproduced: the other tag's count went to 0. Verify the snapshot's HEAD equals `base`
and discard it if not; restoring the wrong commit is worse than not restoring.

Record "did the branch exist before this run" *before* anything that can fail. If a
snapshot clone failure skips that write, a rollback that reads "no base" concludes the
branch never existed and **deletes it** — turning a transient clone error into total
loss of the mirror.

Two more, from testing:

* The snapshot must be a **non-shallow** clone. Pushing the pre-push commit from the
  `--depth 1` work repo is refused with `shallow update not allowed`, because the
  branch has no history and that commit is not an ancestor of anything. `--single-branch`
  is fine and worth having; `--depth` is not.
* Print the success line **inside** the `if`. After a `||`, a refused lease logs the
  warning *and* "restored", and that log is what an operator reads to decide whether to
  repair the mirror by hand.

## GitHub concurrency will silently drop a release

```yaml
concurrency:
  group: mirror-${{ github.repository }}
  cancel-in-progress: false
```

This serialises runs, and it is **not** a safety guarantee. GitHub keeps at most **one
pending job per group**: publish three releases close together and the middle one is
cancelled. That release attaches nothing, and it never reaches its own rollback step
because its job never started — someone has to notice and re-run it by hand.

So treat the group as an optimisation and make the push safe by itself (the lease
above). `cancel-in-progress` stays `false`: cancelling the *running* job mid-push is
how the branch ends up holding neither release.

Also: `concurrency` accepts only `group` and `cancel-in-progress`. There is no `queue`
key — `actionlint` rejects it.

## Small things that cost real time

* **Guard the tag shape.** A tag becomes a directory name on the mirror branch. Reject
  `*/*`, `.`, `..` and leading `-` before touching the branch.
* **Always keep the tag being published, regardless of pruning.** Re-publishing an old
  tag while newer ones fill the keep-quota otherwise prunes the directory just written,
  and the step reports success with URLs that 404.
* **Sort by version, not lexically** (`sort -rV`), or `v0.10.0` prunes before `v0.6.0`.
* **Distinguish "branch absent" from "fetch failed."** Only `git ls-remote --exit-code`
  returning 2 means no such branch; 128 means the remote was unreachable. Treating them
  alike is how an auth error force-pushes an empty branch over every release's images.
* **Keep stderr when a check fails.** `gh release view "$TAG" >/dev/null 2>&1` makes an
  expired token indistinguishable from a missing release, and "create the release first"
  is misleading advice for the former. Capture with `e=$(cmd 2>&1 >/dev/null)` — easy to
  write backwards.
* **Test the real script, not a paraphrase.** Extract the step's `run:` block straight
  out of the YAML and drive it against a throwaway local `git init --bare`. Every bug
  listed above was found that way, and none of them by reading the code.

## Related

`espos-registry-components` covers the other half of publishing — component checks that
keep working when a consumer installs from the ESP Component Registry.
