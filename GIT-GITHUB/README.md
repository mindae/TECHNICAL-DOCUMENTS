# Git & GitHub — Enterprise Engineering Masterclass

> **Learning notes and hands-on lab journal**  
> **Progress:** **Phase 1 (Essentials), Lessons 1–10 completed.** Phase 2 (Intermediate) is next.  
> **Last updated:** 30 October 2026  
> **Separate from:** `GITHUB-COPILOT/README.md`

## Why this handbook exists

This is a practical, three-phase learning programme that connects **Git commands**, **GitHub collaboration**, **releases**, **CI/CD**, **production hotfixes**, and **enterprise safety practices**. We learn with both a terminal and IntelliJ IDEA, while experimenting in a disposable repository rather than the working microservices repositories.

**Teaching convention:** State **Do / Don’t / Stop at** before commands → explain concept → hands-on exercise → inspect actual output → resolve doubts → move on **only when ready**. Knowledge-check questions must start with no preselected option and reveal answers only after submission.

## Contents

1. [Roadmap](#1-three-phase-roadmap)
2. [Lab and current repository state](#2-our-hands-on-lab)
3. [Lesson 1: working tree, index, and commits](#3-lesson-1-working-tree-staging-and-commits)
4. [Lesson 2: restore, unstaging, patch staging](#4-lesson-2-unstaging-and-selective-staging)
5. [Lesson 3: branches, HEAD, and history](#5-lesson-3-branches-head-and-commit-graphs)
6. [Lesson 4: remotes, fetch, pull, upstream](#6-lesson-4-remotes-fetch-pull-and-comparison)
7. [Lesson 5: divergent histories and merging](#7-lesson-5-divergence-rejected-pushes-and-merge)
8. [Lesson 6: feature branch and GitHub PR](#8-lesson-6-feature-branches-and-pull-requests)
9. [Two-dot and three-dot memory guide](#9-dot-syntax-memory-guide)
10. [Release/hotfix model we will practise](#10-how-git-connects-to-releases-and-hotfixes)
11. [Enterprise safety rules and IntelliJ mapping](#11-enterprise-safety-and-intellij-cheat-sheet)
12. [Next hands-on checkpoint](#12-current-checkpoint-and-next-steps)
13. [Quick command reference](#13-command-reference)
14. [Lesson 7: PR merge and fast-forward](#14-lesson-7-pr-merge-and-fast-forward-sync)
15. [Lesson 8: GitHub Actions, failure recovery, and rulesets](#15-lesson-8-github-actions-ci-and-rulesets)
16. [Lesson 9: Local and remote branch cleanup](#16-lesson-9-local-and-remote-branch-cleanup)
17. [Concept clarification: fetch, reachability, merge](#17-key-concepts-clarified)
18. [Lesson 10: annotated tags and GitHub Releases](#18-lesson-10-annotated-tags-and-github-releases)
19. [Phase 1 completion and Phase 2 starting point](#19-phase-1-completion-and-phase-2-starting-point)

---

## 1. Three-phase roadmap

### Phase 1 — Essentials

- Working directory, staging (index), commits, `HEAD`, branch pointers.
- `origin`, local/remote-tracking/upstream branches.
- **Fetch and inspect before integrating**; incoming/outgoing commits.
- Push rejection, merge and conflicts; feature branches and GitHub pull requests.
- Branch protection, reviews, checks, and basics of tags/releases/deployment artifacts.

### Phase 2 — Intermediate

- Merge vs rebase vs squash, cherry-pick, selective integration.
- Trunk-based development and GitFlow trade-offs; feature flags.
- Version tags, maintenance/release branches, build promotion, GitHub Releases.
- Production hotfixes, backports and forward-ports; avoiding unintended release of unreleased features.
- Full release-and-hotfix simulation.

### Phase 3 — Advanced

- `reflog`, `reset`, `revert`, `bisect`, interactive rebase, conflict recovery.
- Git internals, worktrees and complex history problems.
- Protected branch policies, release automation, auditability and rollbacks.
- Multi-service releases and incident-response exercises.

**Focus:** Know what happens to *commits, branches, files, artifacts and production* at each stage, not just the command syntax.

## 2. Our hands-on lab

- **Mac folder:** `~/IdeaProjects/git-enterprise-lab`
- **Remote:** <https://github.com/mindae/git-enterprise-lab>
- **Local default branch:** `main`
- **Remote name:** `origin`
- **Practice branches:** `feature/payment-validation`, `feature/pay-104-validation` (local and remote branch later deleted), and `feature/pay-106-ci-check`.
- We edited files on **GitHub's website** to simulate a second developer while using **Terminal/IntelliJ on Mac** as the first developer.
- We corrected a remote from `http://github.com/...` to `https://github.com/...` using `git remote set-url`.

### Actual commits from our lab

| SHA | Message | Why it matters |
|---|---|---|
| `e9a3309` | Initial release documentation | First local commit |
| `1361866` | docs: add planned features | Staging exercise |
| `9dbff7f` | feat(payment): add validation | Earlier feature branch, ancestor of subsequent main history |
| `8fc23ba` | feat: add remote payment feature note | GitHub-created commit later integrated locally |
| `2b054bb` | feat: add local payment improvement | Local-only commit during divergence exercise |
| `4ba01aa` | feat: add remote transaction feature PAY-103 | Remote-only commit during divergence exercise |
| `23a7f89` | Merge remote-tracking branch 'origin/main' | Two-parent merge; later pushed to GitHub `main` |
| `32a4cf4` | feat(payment): validate transaction currency | PAY-104 feature commit |
| `24fa1c0` | feat(payment): add PAY-105 transaction limits | GitHub `main` advanced independently of PAY-104 |
| `037ce0d` | Merge pull request #1 from mindae/feature/pay-104-validation | PR #1 merge commit; two parents |
| `07f1727` | ci: add payment documentation validation | Added `.github/workflows/pr-check.yml` |
| `580e92e` | test(ci): simulate failing PR validation | Intentional failing GitHub Actions run |
| `07d6a40` | fix(ci): restore passing PR validation | Restored passing workflow, still on PR #2 feature branch |
| `a8174ae` | Merge pull request #2 from mindae/feature/pay-106-ci-check | PR #2 merge commit; latest confirmed `main` |

**Last confirmed local state (30 October):**

- `main`, `origin/main` and `origin/HEAD` point to `a8174ae` (PR #2 merged).
- PAY-104 local branch was deleted with `git branch -d`; remote branch was subsequently deleted with `git push origin --delete feature/pay-104-validation`. The commit `32a4cf4` **remains reachable from `main`**.
- `feature/pay-106-ci-check` remains locally and on GitHub at `07d6a40`.
- `feature/payment-validation` remains locally at `9dbff7f`.
- The public lab repository has an active `protect-main-ci` ruleset targeting `main`, requiring PRs and the GitHub Actions `validate` status check, and disallowing force pushes and branch deletion.
- **Release milestone:** annotated tag `v1.0.0` exists locally and on GitHub, identifies `a8174ae`; GitHub Release **v1.0.0 - Initial Release** has been published. No executable artifact built or deployed.

## 3. Lesson 1: Working tree, staging, and commits

Git has three relevant areas:

1. **Working tree:** files currently edited on disk.
2. **Index / staging area:** snapshot selected for the next commit.
3. **Local repository (`HEAD`):** committed history.

```bash
mkdir -p ~/IdeaProjects/git-enterprise-lab
cd ~/IdeaProjects/git-enterprise-lab
git init -b main
printf 'Release version 1.0\n' > release.txt
git status
git add release.txt
git commit -m 'Initial release documentation'
git log --oneline
```

### Why `git add` matters

We staged one line, then added another:

```bash
echo 'Feature A planned' >> release.txt
git add release.txt
echo 'Feature B planned' >> release.txt

git status
git diff
git diff --staged
```

| Area | Content |
|---|---|
| `HEAD` | Release version 1.0 |
| Index | Release version 1.0 + Feature A |
| Working tree | Release version 1.0 + Feature A + Feature B |

- `git diff` shows **unstaged** change (Feature B).
- `git diff --staged` shows **staged** change (Feature A) relative to `HEAD`.
- A commit at that moment would include A but not B.

## 4. Lesson 2: Unstaging and selective staging

```bash
# Unstage a tracked file; keep ALL edits on disk.
git restore --staged release.txt

# Review current unstaged edits.
git diff

# Interactively stage some hunks; -p = patch.
git add -p release.txt

# Stage everything in this particular file.
git add release.txt
git diff --staged
git commit -m 'docs: add planned features'
```

`git add -p` options: `y` stage, `n` skip, `s` split a hunk (if feasible), `e` edit patch, `q` quit. Two adjacent lines may be a single unsplittable hunk.

**Danger distinction:**

| Command | Meaning |
|---|---|
| `git restore --staged file` | Unstage but **retain working edits** |
| `git restore file` | Discard **unstaged tracked-file edits** (destructive) |
| `git reset --hard HEAD` | Discard tracked staged/unstaged changes across repository (destructive) |

**Enterprise habit:** `git status` → `git diff` → stage only relevant files → `git diff --staged` → `git commit`. Avoid accidentally committing debug configuration or runtime database files.

## 5. Lesson 3: Branches, HEAD, and commit graphs

A branch is a **movable name/reference to a commit**, not an independent physical copy of the whole codebase. `HEAD` ordinarily points through the branch you checked out.

```bash
git branch --show-current
git switch -c feature/payment-validation
printf 'Payment validation implemented\n' > payment.txt
git add payment.txt
git commit -m 'feat(payment): add validation'
git log --oneline --decorate -4

git switch main      # Working files reflect main's committed snapshot.
git switch feature/payment-validation

git log --oneline --graph --decorate --all
```

Important references:

- `main`: local branch.
- `origin/main`: local **remote-tracking** reference, last updated by fetch/push; not the live server itself.
- `origin/HEAD`: local symbolic indication of the remote's default branch (when configured).
- `HEAD`: current checkout.
- `v1.0.0`: sample release tag.

**Production lesson:** A branch is not a deployment. Production may be running an image built from commit B while `main` has advanced to E. We must identify the deployed **artifact/image digest and source SHA** before hotfixing.

## 6. Lesson 4: Remotes, fetch, pull, and comparison

We created an empty GitHub repo, then linked the existing local repo:

```bash
git remote add origin https://github.com/mindae/git-enterprise-lab.git
git remote -v
git push -u origin main
```

- `origin` is a conventional remote nickname, not a synonym for GitHub.
- `-u` / `--set-upstream` associates the local branch with its default remote-tracking branch for common push/pull workflows.
- `git branch -vv` shows branch tracking relationships and ahead/behind counts for configured upstreams.

### Simulating a teammate on GitHub

We created `remote-change.txt` and committed `feat: add remote payment feature note` directly on GitHub. Before fetching, local `origin/main` could still be stale.

```bash
git fetch origin     # Updates remote-tracking refs and downloads objects.
git status -sb       # -s = short; -b = include branch info.
```

**Fetch does not integrate incoming commits into local `main` or change its working files.**

```bash
# Main has these, feature lacks them:
git log --oneline feature/pay-104-validation..origin/main

# Feature has these, main lacks them:
git log --oneline origin/main..feature/pay-104-validation

# Inspect both sides together:
git log --oneline --left-right origin/main...feature/pay-104-validation
```

`git pull` conceptually performs **fetch + integration**, which may merge, rebase, or fast-forward depending on flags/configuration. Prefer an explicit fetch-and-inspect when you need control.

```bash
git pull --ff-only   # Refuse pull when reconciliation needs merge/rebase.
```

`git fetch --all` fetches all configured remotes; it does **not** update all local branches. `git pull --all` does **not** pull every local branch independently.

## 7. Lesson 5: Divergence, rejected pushes, and merge

We made **different** commits on both sides without syncing:

- Local `main`: `2b054bb` — `local-change.txt`.
- GitHub `main`: `4ba01aa` — `remote-feature-103.txt`.
- Shared parent: `8fc23ba`.

After `git fetch origin`:

```text
## main...origin/main [ahead 1, behind 1]
```

**Meaning:** One local-only and one remote-only commit. On `main`, `git log main..origin/main` showed the remote PAY-103 commit; reversing it showed the local improvement.

We examined the outcome of a non-fast-forward push, then merged:

```bash
git merge origin/main
git status -sb
# Expected here: [ahead 2] — local improvement + merge commit.
git log --oneline --graph --decorate --all -8
git push origin main
```

Actual resulting graph:

```text
*   23a7f89 (main, origin/main) Merge remote-tracking branch 'origin/main'
|\
| * 4ba01aa feat: add remote transaction feature PAY-103
* | 2b054bb feat: add local payment improvement
|/
* 8fc23ba feat: add remote payment feature note
```

The merge commit `23a7f89` has **two parents**; inspect with:

```bash
git show --no-patch --pretty=raw 23a7f89
```

After push, `main` and `origin/main` pointed to the same merge commit; `git status -sb` showed no ahead/behind markers.

**Safety:** Never reflexively use `git push --force` on shared or protected branches. A normal push rejects a non-fast-forward update because it would cease to include remote history. Rebase is another valid reconciliation mechanism, but rewrites local commit SHAs; practise separately.

### Remote URL correction

We initially had an HTTP remote and GitHub responded with HTTPS redirect warnings:

```bash
git remote -v
git remote set-url origin https://github.com/mindae/git-enterprise-lab.git
git remote -v
```

This changes Git configuration, not commit history.

## 8. Lesson 6: Feature branches and pull requests

Our actual feature branch is **`feature/pay-104-validation`**:

```bash
git switch main
git pull --ff-only
git switch -c feature/pay-104-validation
printf 'PAY-104: Validate payment currency\n' > payment-validation.txt
git add payment-validation.txt
git commit -m 'feat(payment): validate transaction currency'
git push -u origin feature/pay-104-validation
```

Actual feature commit: `32a4cf4`, initially one commit ahead of the main tip `23a7f89`. Push created `origin/feature/pay-104-validation` and configured upstream.

### PR creation

GitHub provided this creation link:

<https://github.com/mindae/git-enterprise-lab/pull/new/feature/pay-104-validation>

- **Base:** `main`
- **Compare/head:** `feature/pay-104-validation`
- **Title:** `feat(payment): validate transaction currency`

A pull request is a **GitHub collaboration and review workflow**, not an underlying Git commit type. Pushing a feature branch **does not merge** it into `main`.

### When `main` advanced while feature stayed open

We simulated a second change on GitHub `main`:

- `24fa1c0` — `feat(payment): add PAY-105 transaction limits` (`payment-limits.txt`).
- `32a4cf4` — PAY-104 feature (`payment-validation.txt`).
- Shared ancestor: `23a7f89`.

The user's verified outputs:

```text
git branch -vv
* feature/pay-104-validation 32a4cf4 [origin/feature/pay-104-validation]
  feature/payment-validation 9dbff7f
  main                       23a7f89 [origin/main: behind 1]

git log feature/pay-104-validation..origin/main --oneline
24fa1c0 feat(payment): add PAY-105 transaction limits

git log origin/main..feature/pay-104-validation --oneline
32a4cf4 feat(payment): validate transaction currency
```

**Key learning:** The feature branch can be fully synchronized with its own upstream (`origin/feature/...`) while still lacking new commits that exist on `origin/main`. `git branch -vv` compares with each branch's configured upstream, not automatically with `main`.

PR status (open/no conflicts) and final merge were **not yet verified**. Continue from the checkpoint at the end of this document.

## 9. Dot syntax memory guide

**For `git log`, memorize one rule:**

> **`git log A..B` = commits in B that are not in A.**
>
> **LEFT = EXCLUDE. RIGHT = SHOW.**  Think **B − A**.

Apply it naturally:

```bash
# “What has main gained that my feature DOESN'T have?”
git log --oneline feature/pay-104-validation..origin/main

# “What does my feature have that main DOESN'T?”
git log --oneline origin/main..feature/pay-104-validation

# “Show both sides' unique commits at once.”
git log --oneline --left-right origin/main...feature/pay-104-validation
# < = exclusive to LEFT (main); > = exclusive to RIGHT (feature).
```

### `log` versus `diff`: two-dot and three-dot are NOT identical concepts

| Command | What it answers |
|---|---|
| `git log A..B` | Commits reachable from B but not A |
| `git log A...B` | Commits on either side, but not shared history (symmetric difference) |
| `git diff A..B` | Differences between the complete snapshots at tips A and B |
| `git diff A...B` | Differences between the merge base of A and B and **B** |

Our real demonstration:

```bash
git diff --stat origin/main..feature/pay-104-validation
```

```text
payment-limits.txt     | 2 --
payment-validation.txt | 1 +
2 files changed, 1 insertion(+), 2 deletions(-)
```

The two-dot **tip-to-tip comparison** makes PAY-105 look “removed” because the feature-tip snapshot does not contain it. **No PAY-105 commit was deleted from GitHub `main`.**

For a PR-focused comparison:

```bash
git diff --stat origin/main...feature/pay-104-validation
# Expected: payment-validation.txt | 1 +
```

The three-dot diff compares the shared ancestor (`23a7f89`) with the feature tip and isolates the changes PAY-104 introduces.

For clarity, one can express it explicitly:

```bash
git diff "$(git merge-base origin/main feature/pay-104-validation)" feature/pay-104-validation
```

**Caveat:** Commit comparisons measure history/reachability. Cherry-picked changes can have different SHAs even when their content is equivalent.

## 10. How Git connects to releases and hotfixes

Central production scenario to practise in Phase 2:

```text
main:        A --- B(v1.0.0 PROD) --- C --- D --- E
                    \
maintenance:         H(v1.0.1 patch)
```

Production runs **B**, but new work C/D/E is already on `main` and **must not be deployed** as part of the urgent fix.

A safe high-level model:

1. Identify exactly which immutable artifact/image digest is deployed and which **Git SHA** produced it.
2. Tag and retain release provenance (example: `v1.0.0` → B).
3. Implement/verify fix on trunk first where practical, then **backport** it to a maintenance branch rooted at B (commonly with `git cherry-pick`). Urgent fixes can begin on maintenance and be **forward-ported** promptly to trunk.
4. Run tests and release controls on the patch line; build and deploy only the patch artifact as `v1.0.1`.
5. Ensure the fix is also incorporated into `main`/future releases, with issue/PR tracking and regression tests.

**Do not** branch from latest E and assume only the hotfix will deploy; E's ancestor chain already includes the unreleased features. **Do not** leave a production-only patch that can be lost in the next major/minor release.

**Branch ≠ environment; tag ≠ deployment; Git commit ≠ deployable artifact.** CI builds the artifact, CD promotes/deploys it, and release records associate production with a specific image digest and source commit.

## 11. Enterprise safety and IntelliJ cheat sheet

### Safe workflow

```bash
git status -sb
git fetch origin
git branch -vv
git log --oneline --left-right origin/main...feature/pay-104-validation
git diff --stat origin/main...feature/pay-104-validation
```

Before committing, inspect *staged* changes:

```bash
git diff --staged
```

Before integrating, verify the current branch, upstream, test status and relevant PR policy. Prefer PRs and required checks over direct commits to shared `main`.

### IDE mapping

| CLI action | IntelliJ idea |
|---|---|
| `git status` / `git diff` | Commit/Changes and Diff views |
| `git add` / `git commit` | Commit window, check chosen changes (or staging UI, depending on settings) |
| `git branch -vv` / `git switch` | Branch widget, checkout/new branch |
| `git fetch origin` | Git → Fetch |
| `git log --graph --all` | Git Log graph |
| Compare against `origin/main` | Branch widget → Compare with Current, or Git Log / Diff |
| `git push` | Git → Push; inspect outgoing commits |
| PR creation/review | GitHub Pull Requests support or GitHub browser |

In a multi-repository IntelliJ workspace, always check **which Git root** an action will affect.

### Important commands to avoid using reflexively

- `git reset --hard`: destroys tracked uncommitted changes.
- `git push --force`: can rewrite shared remote history.
- `git restore file`: destroys unstaged edits to that tracked file.
- `git add .`: convenient, but can stage unrelated/debug/runtime files; verify with `git diff --staged`.
- `git pull` without knowing the integration strategy: may merge, rebase or fail on divergence.

## 12. Current checkpoint and next steps

**Completed:** **Phase 1 Lessons 1–10**, including two merged PRs, CI failure/repair, protected `main`, fast-forward synchronization, branch cleanup, annotated tag `v1.0.0` and published GitHub Release.

**Current repository:** Last confirmed `main` = `origin/main` = `a8174ae`, and published annotated tag `v1.0.0` points to the same commit. Release published, **no production deployment**.

**Next planned lesson (not yet practised):** **Phase 2** — trunk-based development, release/maintenance branches, version progression, backport/forward-port, cherry-pick, and simulated `v1.0.1` production hotfix while unreleased features remain on `main`.

**Procedure:** Start each lab with **Do / Don't / Stop at**; run only the stated steps; inspect output; ask about doubts before introducing the next topic.

## 13. Command reference

| Command | Meaning / reminder |
|---|---|
| `git status -sb` | Short status with current/upstream branch information |
| `git branch -vv` | Local branches + tracking upstream and ahead/behind |
| `git remote -v` | Show fetch and push remote URLs |
| `git remote set-url origin URL` | Replace URL of named remote |
| `git switch main` | Change checked-out branch |
| `git switch -c feature/name` | Create and switch to a feature branch |
| `git add file` | Stage a file's current contents |
| `git add -p file` | Interactively stage hunks |
| `git diff` | Working tree vs index |
| `git diff --staged` | Index vs HEAD |
| `git restore --staged file` | Unstage, preserve edits |
| `git commit -m 'message'` | Save staged snapshot locally |
| `git fetch origin` | Update remote-tracking refs; do not integrate |
| `git pull --ff-only` | Integrate only if fast-forward possible |
| `git push -u origin branch` | Push and set upstream tracking |
| `git log --oneline --graph --decorate --all` | Compact ancestry graph |
| `git log A..B` | B-only commits (left excluded, right shown) |
| `git log --left-right A...B` | Unique commits on each side, marked `<`/`>` |
| `git diff A..B` | Tip-to-tip snapshot comparison |
| `git diff A...B` | Merge-base-to-B comparison, useful for PR changes |
| `git merge origin/main` | Integrate fetched remote main into current branch |
| `git show --no-patch --pretty=raw SHA` | Inspect commit metadata/parents |
| `git merge-base --is-ancestor A B` | Check if A is reachable from B by following parent links; exit status 0=yes |
| `git config --get-all remote.origin.fetch` | Inspect which remote refs a fetch tracks |
| `git show origin/main:path/to/file` | Inspect fetched file content without checkout |
| `git branch --merged main` | Find local branches whose tips are reachable from main |
| `git branch --no-merged main` | Find branches not fully merged in main |
| `git branch -d feature/name` | Safely remove a local branch reference if Git considers it merged |
| `git push origin --delete feature/name` | Request deletion of branch on GitHub |
| `git tag -a v1.0.0 SHA -m "Release message"` | Create annotated tag at a specified commit |
| `git tag -n` | List tags and annotation messages |
| `git show --no-patch v1.0.0` | View annotated tag and target commit metadata |
| `git rev-parse 'v1.0.0^{}'` | Peel annotated tag to its target commit |
| `git push origin v1.0.0` | Publish one tag to remote |
| `git ls-remote --tags origin 'v1.0.0*'` | Inspect remote tag object and peeled target |

### Updating this README later

Append subsequent lessons with the **actual commits, branches, merge strategies, conflicts, tags, artifact versions, and observed terminal outputs**. Keep the theory near the lab that demonstrated it. Record unverified steps as *pending* rather than claiming success.

---

*This is a personal training reference. The repository and commit data above reflect our exercises as shared in chat, not a live verification of GitHub's current state.*


## 14. Lesson 7: PR merge and fast-forward sync

### Hands-on PR #1: PAY-104 currency validation

- Feature: `feature/pay-104-validation`, commit `32a4cf4`, adding `payment-validation.txt`.
- Remote `main` independently gained PAY-105 commit `24fa1c0`, adding `payment-limits.txt`.
- GitHub PR #1 merged using **Create a merge commit**: `037ce0d`.
- After `git fetch origin`, local `main` was at `23a7f89`, while `origin/main` was at `037ce0d`. Local Git reported **behind 3** because the history added PAY-105, PAY-104, and the PR merge commit. A PR is not necessarily one commit.

```bash
git fetch origin
git log --oneline --graph --decorate --all -10
git switch main
git status -sb            # [behind 3] in our actual exercise
git pull --ff-only        # Updating 23a7f89..037ce0d, Fast-forward
git status -sb            # synchronized
```

**Fast-forward rule:** The current branch tip must be reachable by following parent links backwards from the proposed target commit. It is *not enough* merely to have a common ancestor. A fast-forward moves a branch pointer, creating no commit, **even if the history it moves across already contains merge commits**.

```bash
git merge-base --is-ancestor 23a7f89 037ce0d
echo $?      # 0: yes
git merge-base --is-ancestor 037ce0d 23a7f89
echo $?      # 1: no
```

**Merge vs pull:** `git merge origin/main` integrates the **already-fetched** remote-tracking history. With a behind-only branch, it normally fast-forwards; on divergent branches, it generally creates a merge commit, potentially after conflict resolution. `git pull --ff-only` fetches first, then **refuses** to integrate diverged history through a merge commit or rebase. We used `git merge origin/main` for the divergent PAY-103 experiment (`23a7f89`) but `git pull --ff-only` after PR merges when local `main` was simply behind.

## 15. Lesson 8: GitHub Actions CI and rulesets

### Hands-on PR #2: CI validation

Created `feature/pay-106-ci-check` with `.github/workflows/pr-check.yml`:

```yaml
name: PR Validation
on:
  pull_request:
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
      - name: Validate payment documentation
        run: |
          test -s release.txt
          test -s payment.txt
          test -s payment-validation.txt
          echo "All payment documentation checks passed"
```

- `07f1727` introduced the workflow; GitHub Actions job **`validate` passed** (about 3 seconds in screenshot).
- `580e92e` deliberately added `test -s nonexistent-file.txt`; CI **failed** with `Process completed with exit code 1`. The `test` builtin reports a false condition through its exit code without necessarily printing a descriptive error.
- `07d6a40` removed the failing line; pushed to the existing feature branch; the latest PR check **passed**.
- GitHub PR #2 merged with a merge commit `a8174ae`; all three feature commits plus the merge commit became reachable from `main`. Local `main` was **behind 4** until `git pull --ff-only` advanced from `037ce0d` to `a8174ae`.

**More diagnostic CI failure message:**

```bash
if ! test -s required-file.txt; then
  echo "::error::required-file.txt is missing or empty"
  exit 1
fi
```

### GitHub branch ruleset

Created GitHub **Settings → Rules → Rulesets** → `protect-main-ci` with:

| Control | Lab setting |
|---|---|
| Enforcement | Active |
| Target | Default branch `main` |
| Require PR before merging | Enabled |
| Required status checks | Enabled; explicitly selected `validate` from GitHub Actions |
| Block force pushes | Enabled |
| Restrict branch deletions | Enabled |
| Require up-to-date branch before merging | Not enabled |
| Bypass list | Empty |

**Critical lesson:** Merely running GitHub Actions is different from **requiring** it as a status check. The `validate` job had to be explicitly selected inside the ruleset's additional settings. We verified the final ruleset and a successful merge, **but did not test an attempted PR merge while that rule was active and the check was failing**. Do not claim that experiment was performed.

**CI, PR approval, merge, release, and deployment are separate stages.** The source-branch merge does not by itself prove a production artifact was deployed.

## 16. Lesson 9: Local and remote branch cleanup

At `main = a8174ae`, Git showed all three local feature branches as merged:

```bash
git branch -vv
git branch --merged main
git branch --no-merged main   # no output in our exercise
```

We deleted **only** the old PAY-104 local branch:

```bash
git branch -d feature/pay-104-validation
# Deleted branch feature/pay-104-validation (was 32a4cf4).
```

The graph still showed `32a4cf4 (origin/feature/pay-104-validation)` because the *remote-tracking* branch reference remained. We then ran:

```bash
git push origin --delete feature/pay-104-validation
# [deleted] feature/pay-104-validation
git log --oneline --graph --decorate -10
git show --stat 32a4cf4
```

The `origin/feature/pay-104-validation` label disappeared but commit `32a4cf4` **remained visible and inspectable**, as it is an ancestor of PR #1's merge commit, which remains reachable from `main`.

**Three different things:**

1. Local branch `feature/pay-104-validation` — deleted with `git branch -d`.
2. GitHub remote branch and local `origin/feature/pay-104-validation` tracking ref — removed by successful remote deletion.
3. Commit `32a4cf4` — retained in `main` ancestry; deleting branch labels does not erase that history.

Do not automatically remove release/maintenance branches that may still be needed for hotfixes. `git branch -D` force-deletes the local ref, bypassing the merge-safety check; investigate before using it. Squash-merged PRs may have changes integrated but original feature commit SHAs not reachable from `main`, causing `-d` to refuse.

## 17. Key concepts clarified

### What `git fetch origin` actually downloads

`fetch` downloads required Git objects (**commits**, **trees**, **blobs/file content**) into the repository's object store and updates remote-tracking references; it does **not** check files out into the working tree or advance the checked-out local branch. You can examine a fetched file without switching:

```bash
git show origin/main:remote-feature-103.txt
```

In an ordinary clone, `git fetch origin` normally fetches all branches covered by the configured refspec, **regardless of the currently checked-out branch**. Check with:

```bash
git config --get-all remote.origin.fetch
# Typical: +refs/heads/*:refs/remotes/origin/*
git branch -r
```

`git fetch origin main` explicitly fetches `main` and records it in `FETCH_HEAD`; in a typical clone it also updates `origin/main`. Shallow/partial clones may differ in which Git objects are initially obtained.

### Commit reachability

A commit is **reachable** from a starting commit if Git can arrive at it by following zero or more **parent** links (backward, not forward). A merge commit has two parents and reaches the history from both sides. `git log A..B` shows commits reachable from B but not from A. Reachability explains fast-forward eligibility and why merged commits remain after feature branches are deleted.

### Difference between `git log` and `git diff` dots

```bash
# Commit sets: right minus left
git log --oneline feature/pay-104-validation..origin/main

# Symmetric difference: unique commits on either side
git log --oneline --left-right origin/main...feature/pay-104-validation

# Snapshot of B versus snapshot of A
git diff A..B

# Changes introduced on B since the common ancestor with A
git diff A...B
```

In the earlier PAY-104/PAY-105 divergence, `git diff origin/main..feature/pay-104-validation` showed a PAY-105 file removal *and* a PAY-104 file addition, because it compared branch-tip snapshots. The three-dot PR-style diff showed **only** the PAY-104 addition.

---

*Training journal based on observed outputs in the conversation. Phase 1 tag and GitHub Release completed; production deployments, cherry-pick hotfixes, and rollback simulations remain future exercises.*


## 18. Lesson 10: Annotated tags and GitHub Releases

**Completed on 30 October 2026.** This section records what was actually done, distinguishing source tagging and release publication from building and deploying an application.

### The four distinct entities

| Entity | Example in our lab | What it means |
|---|---|---|
| Git commit | `a8174ae` | Immutable source snapshot (object ID) |
| Git tag | `v1.0.0` | Human-readable release reference pointing, via an annotated tag object, to that commit |
| GitHub Release | `v1.0.0 - Initial Release` | GitHub publication with notes and source archives |
| Deployed artifact | **None** | A JAR/container/image digest would need a separate build and deployment process |

**Release URL:** <https://github.com/mindae/git-enterprise-lab/releases/tag/v1.0.0>  
**Tagged source SHA:** `a8174aedd9c48475cd079bde0a11c4e4f1adc59d`  
**Annotated tag object SHA:** `e49f90198a89ea455dd3a420ab6698cfcf2734b5`

### Part A — Create an annotated tag locally

We first verified that `main` and `origin/main` were both at merge commit `a8174ae` (PR #2) and created the tag explicitly on that commit:

```bash
cd ~/IdeaProjects/git-enterprise-lab
git switch main
git status -sb
git log -1 --oneline --decorate

git tag -a v1.0.0 a8174ae -m "Release v1.0.0 - initial Git enterprise lab"

git tag -n
git show --no-patch v1.0.0
git rev-parse 'v1.0.0^{}'
git log --oneline --decorate -3
```

Observed:

```text
v1.0.0          Release v1.0.0 - initial Git enterprise lab
a8174aedd9c48475cd079bde0a11c4e4f1adc59d
a8174ae (HEAD -> main, tag: v1.0.0, origin/main, origin/HEAD) Merge pull request #2 ...
```

**Why two dates?** The commit was created earlier on 30 October; the annotated tag was created later. An annotated tag is a separate Git object containing tagger, message and timestamp. Creating the tag did not change the commit, working tree or branch tip.

### Part B — Publish the Git tag to GitHub

```bash
git push origin v1.0.0
# * [new tag] v1.0.0 -> v1.0.0

git ls-remote --tags origin 'v1.0.0*'
```

Observed:

```text
e49f90198a89ea455dd3a420ab6698cfcf2734b5 refs/tags/v1.0.0
a8174aedd9c48475cd079bde0a11c4e4f1adc59d refs/tags/v1.0.0^{}
```

There are **not two tags**. The first SHA refers to the annotated **tag object**, while `^{}` denotes the **peeled target commit**. `git rev-parse 'v1.0.0^{}'` gives the actual tagged commit.

**Tag-push distinction:** `git push origin main` typically pushes the branch, not a new local tag; explicitly push the intended tag instead of automatically publishing all tags with `git push --tags`.

**GitHub UI distinction:** GitHub's **Commits** list did not show `v1.0.0` adjacent to `a8174ae` as `git log --decorate` did. The **Tags** page did show `v1.0.0` pointing to `a8174ae`. This is a difference in presentation, not an error. `git branch -vv` does not list tags.

### Part C — Publish the GitHub Release

In GitHub, we opened **Releases → Draft a new release** and chose the **existing** `v1.0.0` tag (did not create a new tag).

- **Release title:** `v1.0.0 - Initial Release`
- **Release label:** Normal/latest release, **not** pre-release
- **Release notes:** PAY-103 transaction note, PAY-104 currency validation note, PAY-105 transaction limits note, PAY-106 GitHub Actions PR-validation workflow; included validation, PR #1/#2, and `protect-main-ci` ruleset
- **Source commit:** `a8174ae`
- **Assets:** GitHub's auto-generated **Source code (zip)** and **Source code (tar.gz)**; no JAR, Docker image or uploaded binaries
- **Deployment note:** Explicitly stated that this is a **simulated learning release**, with no real application built or deployed

The user published the release and the GitHub screenshot showed `v1.0.0 - Initial Release` with a **Latest** label, the correct `a8174ae` SHA, published release notes and two source archives.

### Critical distinctions and enterprise practice

- A **branch** advances on new commits made while checked out; a release **tag** should remain anchored to the selected revision. Treat published release tags as immutable, even though Git technically permits retagging.
- A GitHub Release **is not** proof of deployment. Source ZIP/TAR.GZ files are not compiled deployable artifacts.
- A production deployment should identify the **exact built artifact** (e.g., image digest), its build/test provenance and the **source SHA/tag**. Deploying a tag alone is not a complete release process.
- The `v1.0.0` tag becomes our **fixed baseline** for Phase 2: develop newer features on `main`, then patch only the `v1.0.0` release line as `v1.0.1` without accidentally including unfinished features.

---

## 19. Phase 1 completion and Phase 2 starting point

**Phase 1 — Essentials: COMPLETE (Lessons 1–10).**

What was actually exercised: initial Git repository and staging; selective commits and branch creation; GitHub remote and upstream; fetch and incoming/outgoing comparisons; diverged local/remote changes; rejected push and merge; PR #1; branch ancestry and `pull --ff-only`; PR #2; GitHub Actions success → intentional failure → repair; `protect-main-ci` ruleset requiring PR and `validate` status check; merged-branch cleanup; annotated tag and published GitHub Release `v1.0.0`.

**Important evidence limit:** We did not attempt to merge a PR *while a failing status check was required*, so enforcement in that particular failure state was **not directly demonstrated**. The final PR was merged after the passing check and ruleset configuration.

**Starting state for Phase 2:**

```text
v1.0.0 (published release) → a8174ae
main / origin/main        → a8174ae  (last observed)
Production deployment    → none; simulated baseline only
```

**Next exercises (NOT yet completed):** choose trunk-based vs GitFlow workflow; make unreleased commits on `main`; retain a `release/1.0` maintenance line from tagged commit `a8174ae`; apply urgent patch without including new features; use selective `cherry-pick`, backport and forward-port; publish simulated `v1.0.1`; explain CI/artifact provenance and rollbacks.

**Learning process:** Always announce **Do / Don't / Stop at** *before* commands. After exercises, inspect outputs, clear doubts, then advance only when the learner explicitly confirms readiness. No MCQ option preselected.
