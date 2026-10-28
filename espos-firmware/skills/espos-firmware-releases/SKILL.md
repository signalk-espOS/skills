---
name: espos-firmware-releases
description: Publish firmware a browser-based flasher can actually download. Why GitHub release assets are unreachable from a web page, how to mirror them to a branch safely with force-with-lease, the GitHub Actions concurrency behaviour that silently drops a release (and the queue key that stops it), and the version-sort and no-checkout traps in release scripts.
---

# Publishing firmware a web flasher can fetch

Verified against GitHub Actions, October 2026.

## Why release assets do not work in a browser

**GitHub serves release downloads with no `Access-Control-Allow-Origin` header**, on
the `302` and on the redirect target, so a web page cannot fetch them at all. A
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
rather than the sum of every release. It is still a branch everyone who clones the
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
  # and any rollback needs this SHA. Recording one that never arrived is harmless:
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

The retry must redo exactly what the first attempt did: copy the README, replace this
tag's directory, re-prune. Two copies of that logic drift the moment one is fixed, so
make it one function called from both places.

Two bugs that only appear on the *second* retry, both found by testing rather than
reading:

* `git reset --soft HEAD~1` fails on the first attempt, because that commit has no parent.
  Use `git checkout --orphan` instead; a wholesale-replacement push wants no history.
* `git checkout --orphan fixed-name` fails the **second** time with
  `a branch named 'fixed-name' already exists`. Make the name unique per attempt.

And one that only appears if the README lives inside the work tree: if you write it to
`$work/README.md` and then `cd "$work"`, a retry's `find . -exec rm -rf` deletes it and
the restore copies a file onto itself. Stage it outside the repo (`RUNNER_TEMP`).

## Rolling back without eating someone else's release

If publishing the release assets fails *after* the mirror pushed, the branch advertises
a tag whose release has none. Undoing that is right, but the undo must also be leased,
against the commit **this run** pushed. Otherwise a run that failed late will flatten a
release that published after it.

The trap is subtler than it looks: **re-take the snapshot every time you re-read the
branch.** A single snapshot taken before the first attempt goes stale the moment a
retry moves `base`, and then the rollback restores a state from before another run's
tag existed, deleting their directory while their release still points at it.
Reproduced: the other tag's count went to 0. Verify the snapshot's HEAD equals `base`
and discard it if not; restoring the wrong commit is worse than not restoring.

Record "did the branch exist before this run" *before* anything that can fail. If a
snapshot clone failure skips that write, a rollback that reads "no base" concludes the
branch never existed and **deletes it**, turning a transient clone error into total
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

This serialises runs, and on its own it is **not** a safety guarantee. By default
GitHub keeps at most **one pending job per group**: publish three releases close
together and the middle one is cancelled. That release attaches nothing, and it never
reaches its own rollback step because its job never started. Someone has to notice and
re-run it by hand. The same default let an hourly cron run cancel the registry's
reindex that a release had just asked for.

Since May 2026 `concurrency` takes a `queue` key: `queue: max` keeps up to 100 runs
waiting instead of cancelling all but one.

```yaml
concurrency:
  group: mirror-${{ github.repository }}
  queue: max
  cancel-in-progress: false
```

actionlint 1.7.12 predates the key and reports `unexpected key "queue" for
"concurrency" section`; GitHub accepts it. Silence it per file in
`.github/actionlint.yaml` (`paths: <file>: ignore: [...]`) until actionlint knows it.

Make the push safe by itself anyway (the lease above): a queue orders runs, it does not
stop a manual re-run of an old tag racing a new one. `cancel-in-progress` stays
`false`, because cancelling the *running* job mid-push is how the branch ends up
holding neither release.

## Small things that cost real time

* **`sort -V` puts a prerelease above its release.** It ranks `v1.2.0-rc1` after
  `v1.2.0`, the opposite of semver. A script that closes "older" bump PRs with
  `sort -V | tail -n1` keeps `v1.2.0-rc1`'s PR open once `v1.2.0` is out, and one that
  picks the newest tag picks the rc. Compare the part before the `-` first
  (`[ "${OLD%%-*}" = "$TAG" ]` means OLD is a prerelease of TAG), or use a real semver
  comparison.
* **A job without a checkout cannot `git push`.** A release job that only downloads
  artifacts has no repository, so a rollback's `git push --delete "$remote" branch`
  fails there and leaves a branch a failed first release created. Delete through the
  API instead: `gh api -X DELETE repos/$REPO/git/refs/heads/<branch>`.
* **Write generated files after the restore.** A README written into the work tree
  before `git checkout FETCH_HEAD -- .` restores the branch is overwritten by the
  previous run's copy, which names the previous tag.
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
  is misleading advice for the former. Capture with `e=$(cmd 2>&1 >/dev/null)`, which is easy to
  write backwards.
* **Test the real script, not a paraphrase.** Extract the step's `run:` block straight
  out of the YAML and drive it against a throwaway local `git init --bare`. Every bug
  listed above was found that way, and none of them by reading the code.

## Related

`espos-registry-components` covers the other half of publishing: component checks that
keep working when a consumer installs from the ESP Component Registry.
