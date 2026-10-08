---
name: zerv-e2e
description: E2E test zerv shared workflows (shared-lock, shared-unlock, shared-pr-unlock, versioning) through the example-zerv-flow caller pipelines, using real PRs, labels, and GitHub runners. Use when asked to e2e test zerv, verify shared-lock behavior end to end, or validate a zerv branch/PR against a real caller. Defaults to testing zerv main; pass a branch, SHA, or zerv PR URL to test a specific zerv change.
---

# zerv-e2e

End-to-end test of zerv shared workflows by running the real example-zerv-flow CI/CD pipelines against a chosen zerv ref. No mocks: real PRs, real `deploy-*` labels, real lock branches, real runners.

## Inputs

Optional argument: the zerv ref to test.

- None → test zerv `main`
- Branch name (e.g. `revert/shared-lock-timeout-migration`) → test that branch
- zerv PR URL or number (e.g. `https://github.com/wislertt/zerv/pull/301`) → resolve head branch via `gh pr view <N> --repo wislertt/zerv --json headRefName,isCrossRepository`. If cross-repo (fork), use the head SHA as ref instead of a branch name.

Echo the resolved ref before starting and use it everywhere below as `$ZERV_REF`.

This skill always runs the full matrix (T1–T4, ~20–30 min wall with the `ci-minimal` fast path; ~35–45 min without). It is the deep check, run when you want complete verification; everyday regression coverage lives in zerv's own CI (the lock-poll and sandbox contract harnesses run on every PR that touches lock workflows).

## Prerequisites

- `gh` authenticated with push access to `wislertt/example-zerv-flow`
- Caller repo checkout: `/Users/wisl/Desktop/vault/personal-repo/example-zerv-flow` (work from its main checkout, not a worktree)
- No leftovers from a previous e2e run: check for open PRs titled `chore: e2e ...` and remote branches `e2e/*` plus stale lock branches (`d-branch-deploy-lock`). Clean them up first (see Cleanup) or reuse them.
- `semantic-pull-request.yml` enforces conventional commit PR titles — e2e PR titles must use a valid type prefix (`chore:`/`ci:`), never `e2e:`.

## How the caller consumes zerv

All zerv workflows are pinned by SHA in `.github/workflows/{ci,cd,deploy-env,pr-unlock}.yml`:

```
uses: wislertt/zerv/.github/workflows/shared-lock.yml@<40-char-sha> # vX.Y.Z
```

Repointing the ref is the only change needed to test any zerv version.

Caller behavior that matters for assertions:

- `ci.yml` (pull_request: opened/synchronize/reopened): deploy jobs gated by `deploy-<env>` labels; `lock_key_owner = github.ref` → `refs/pull/N/merge` (unique per PR); `unlock_after_deploy: false` → lock stays held after deploy.
- There is **no `labeled` trigger** — to fire deploy after adding a label, push an empty commit.
- Opening a PR triggers a first ci run with no labels (all deploy legs skipped) — harmless, expect it.
- Lock branch for env `d` is `d-branch-deploy-lock` in example-zerv-flow (github/lock: branch `<key>-branch-deploy-lock` holding lock.json with reason `owner:<id>`).
- `pr-unlock.yml` (pull_request: closed/unlabeled) releases locks.
- Measured run cost (2026-10-07): execution is **~4 min** per ci run (label checks ~5s, lock step reached ~23s in); the variable part is hosted-runner **queue wait** — usually seconds, but observed 21+ min once under runner-pool contention. Budget per run: queue + 4 min. The full rust (3 OS) + python (15 job) matrix runs too but is irrelevant to lock assertions — never gate a test on the whole run going green; assert on the lock/deploy job names.

## Setup (once per run)

From the example-zerv-flow repo:

```bash
git fetch origin main
git checkout -B e2e/zerv-$(echo "$ZERV_REF" | tr '/' '-') origin/main
# Repoint every zerv pin to $ZERV_REF
perl -pi -e 's{(wislertt/zerv/\.github/workflows/[^\@ ]+)\@[0-9a-f]{40}}{$1\@'"$ZERV_REF"'}g' .github/workflows/*.yml
grep -rn 'wislertt/zerv/.github/workflows' .github/workflows/   # verify: all refs show $ZERV_REF
git commit -am "chore: point zerv workflows at $ZERV_REF for e2e lock test"
git push -u origin HEAD
```

Open **PR 1** (`gh pr create --title "chore: e2e zerv lock test PR 1 ($ZERV_REF)" --body "..." --fill`-style; do NOT merge). Then create a second branch from the same commit for **PR 2**:

```bash
git checkout -b e2e/zerv-...-pr2   # from the same repointed commit
git push -u origin HEAD
gh pr create ... # PR 2
```

Both PRs get label `deploy-d`. Also label **both PRs `ci-minimal`** immediately after opening them (`gh pr edit <PR> --add-label ci-minimal`) — ci.yml reads that label live at job time and skips the ~5 min test-matrix/pre-commit legs, which the lock tests never assert on. Total footprint: exactly 2 PRs.

## Test matrix

Run in order. Assert with `gh run view <run-id> --repo wislertt/example-zerv-flow --json jobs`. Jobs are nested under the caller matrix — full names look like `deploy-all-env / deploy (d) / lock-d / lock-d`, `deploy-all-env / deploy (d) / deploy-d`, `deploy-all-env / deploy (d) / unlock-d`; the `n` and `p` legs appear as skipped. Match on the leaf name suffix and verify actual names from the run before asserting. The lock branch holds `lock.json` at the repo root (not under `.github/workflows/`).

| #   | Test                                      | Action                                                                                                                                                                                             | Expected                                                                                                                                 |
| --- | ----------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| T1  | First acquire                             | Label PR 1 `deploy-d`, push empty commit (`git commit --allow-empty -m "chore: trigger ci for e2e lock test" && git push`), wait for run                                                           | `lock-d` success, `deploy-d` success. Lock branch `d-branch-deploy-lock` exists, lock.json reason contains `owner:refs/pull/<PR1>/merge` |
| T2  | Cross-PR contention                       | Label PR 2 `deploy-d`, push empty commit, wait for run                                                                                                                                             | PR 2 `lock-d` **failure** (fail-fast, timeout 0), `deploy-d` skipped. PR 1 lock untouched                                                |
| T3  | Own-lock reentrancy (the #297 regression) | Re-run PR 1's labeled ci run (`gh run rerun <run-id>`), wait                                                                                                                                       | `lock-d` **success** again — same owner re-acquires its own held lock with timeout 0. This is the test that fails on v0.8.36             |
| T4  | Release + reacquire                       | Remove `deploy-d` from PR 1 → `pr-unlock` fires. Verify `d-branch-deploy-lock` is gone (`gh api repos/wislertt/example-zerv-flow/git/ref/heads/d-branch-deploy-lock` → 404). Then re-run PR 2's ci | `unlock-d` success on PR 1; PR 2 `lock-d` now **success**, `deploy-d` success                                                            |

Waiting for runs (execution ~4 min each + queue variance; see measured costs above): poll

```bash
gh run list --repo wislertt/example-zerv-flow --branch <e2e-branch> --limit 1
gh run view <run-id> --repo wislertt/example-zerv-flow   # when completed
```

or `gh run watch <run-id> --exit-status`. Do not start T2/T3 until the previous run's state is settled (lock held as expected).

Failure diagnosis: if a step's result contradicts the table, pull the lock job log before concluding:

```bash
gh run view <run-id> --repo wislertt/example-zerv-flow --log-failed
```

## Pass criteria

All four rows above match expected results, and after cleanup no `e2e/*` branches, `e2e:*` PRs, or `d-branch-deploy-lock` remain. Report a table: test → expected → actual → PASS/FAIL, plus the zerv ref tested and run URLs.

## Cleanup

```bash
gh pr close <PR1> <PR2> --repo wislertt/example-zerv-flow   # closing also fires pr-unlock
git push origin --delete e2e/<branch1> e2e/<branch2>
gh api -X DELETE repos/wislertt/example-zerv-flow/git/refs/heads/d-branch-deploy-lock 2>/dev/null || true
git checkout main && git branch -D e2e/...
```

## Gotchas

- Never merge the e2e PRs — they exist only to trigger pull_request CI.
- Do not label both PRs immediately after opening them: the _opened_ run evaluates its label check mid-run and can acquire the lock first (seen 2026-10-07: PR 2's opened run stole the lock; PR 1's T1 then failed with contention). Add labels after PR 1's opened run has passed the label check, or only on synchronize.
- All commit messages and PR titles must be conventional (`chore: ...`, `ci: ...`) — `semantic-pull-request.yml` rejects non-conventional PR titles.
- Reusable workflows cannot be pinned to `refs/pull/N/merge`; use the head branch name (same-repo PRs) or head SHA (forks).
- If zerv ref is unreleased, the trailing `# vX.Y.Z` pin comments go stale — fine for e2e; do not "fix" them.
- v0.8.36 regression signature to watch for: T3 `lock-d` fails with "still locked by refs/pull/<PR1>/merge" even though the owner matches. If testing `main` after the revert PR merges, T3 must pass.
- Renovate PRs also run ci on their branches but use their own refs — they cannot steal the `d` lock unless labeled; ignore them.
