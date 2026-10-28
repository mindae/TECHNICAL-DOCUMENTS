# Git & GitHub — Enterprise Engineering Masterclass

> **Learning notes and hands-on lab journal**  
> **Progress:** Phase 1 (Essentials), through Lesson 6B: feature branches, remote comparisons, and PR preparation.  
> **Last updated:** 28 October 2026  
> **Separate from:** `GITHUB-COPILOT/README.md`

## Why this handbook exists

This is a practical, three-phase learning programme that connects **Git commands**, **GitHub collaboration**, **releases**, **CI/CD**, **production hotfixes**, and **enterprise safety practices**. We learn with both a terminal and IntelliJ IDEA, while experimenting in a disposable repository rather than the working microservices repositories.

**Teaching convention:** Explain a concept → perform a hands-on scenario → inspect actual output → answer doubts → move to the next topic **only when ready**. Knowledge-check questions must not have a preselected answer.

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
- **Practice feature branches:** `feature/payment-validation` and `feature/pay-104-validation` (lowercase `pay`, as actually used).
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

**Last confirmed local state (28 October):**

```text
* feature/pay-104-validation 32a4cf4 [origin/feature/pay-104-validation]
  feature/payment-validation 9dbff7f
  main                       23a7f89 [origin/main: behind 1]
```

At that checkpoint, `origin/main` was at `24fa1c0`, and the PAY-104 branch (local and remote) was at `32a4cf4`. The feature and the remote main had each gained one exclusive commit. **PR creation/open status was not confirmed in the shared terminal output.** Do not assume the PR is merged.

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

**Checkpoint:** Phase 1, Lesson 6B; we have **not yet completed the PR merge**.

What to do **before moving to a new lesson**:

1. Verify the three-dot diff:
   ```bash
   git fetch origin
   git diff --stat origin/main...feature/pay-104-validation
   ```
   Expected based on prior history: only `payment-validation.txt | 1 +`.
2. Inspect the PAY-104 PR on GitHub. Confirm **whether a PR exists**, its **open status**, its proposed changes and any **merge-conflict/check** notices.
3. Review and discuss any questions about `git log` versus `git diff`, remote tracking, or PR state.
4. Once questions are resolved, practise PR review/merge (merge, squash, rebase options), fetch the updated `main`, and inspect the graph.
5. Subsequent Phase 1 topics: branch protection, CI/status checks, tags, GitHub Releases, connection to deployable artifacts.

**Lesson discipline:** Stop after each lab. Ask for doubts and await explicit confirmation before introducing a new lesson.

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

### Updating this README later

Append subsequent lessons with the **actual commits, branches, merge strategies, conflicts, tags, artifact versions, and observed terminal outputs**. Keep the theory near the lab that demonstrated it. Record unverified steps as *pending* rather than claiming success.

---

*This is a personal training reference. The repository and commit data above reflect our exercises as shared in chat, not a live verification of GitHub's current state.*
