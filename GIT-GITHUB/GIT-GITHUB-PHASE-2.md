# Git & GitHub — Enterprise Engineering Masterclass
## Phase 2 — Intermediate: Hands-on Learning Journal

> **Status:** Lessons 11–13 completed through staged-only stash restoration and cleanup (5 November 2026).  
> **Phase 1:** Lessons 1–10 documented in [`README.md`](README.md).  
> **Lab repository:** <https://github.com/mindae/git-enterprise-lab>  
> **Local lab:** `~/IdeaProjects/git-enterprise-lab` on macOS / Zsh; IntelliJ IDEA for visual inspection.  
> **Evidence:** Actual commands, SHAs, and status outputs shared during our hands-on sessions. This is not a live audit of all remote GitHub state.  
> **Scope:** Learning repository containing text/configuration examples, not a working payments application. No deployable application was built or deployed.

## Contents

1. [Goals, workflow and current checkpoint](#1-goals-workflow-and-current-checkpoint)
2. [Maintenance release and production-hotfix simulation](#2-maintenance-release-and-production-hotfix-simulation)
3. [Cherry-pick and forward-porting](#3-cherry-pick-and-forward-porting)
4. [Merge versus rebase on the same feature](#4-merge-versus-rebase-on-the-same-feature)
5. [Rejected pushes and `--force-with-lease`](#5-rejected-pushes-and---force-with-lease)
6. [Merge conflicts: index stages and resolutions](#6-merge-conflicts-index-stages-and-resolutions)
7. [Rebase conflicts: reversed ours/theirs](#7-rebase-conflicts-reversed-ourstheirs)
8. [Ignore rules, IntelliJ files and repository root](#8-ignore-rules-intellij-files-and-repository-root)
9. [Stash lifecycle: tracked and untracked files](#9-stash-lifecycle-tracked-and-untracked-files)
10. [Advanced stash: `--keep-index`, `--staged`, `--index`](#10-advanced-stash---keep-index---staged---index)
11. [Revision command sheet](#11-revision-command-sheet)
12. [Safety rules, unresolved items and next lessons](#12-safety-rules-unresolved-items-and-next-lessons)

---

## 1. Goals, workflow and current checkpoint

### Learning approach

- **CLI first**, then connect the operation to the IntelliJ Git Log, diff and conflict views.
- Every practical block: **Do / Don't / Stop at**; execute bite-sized steps; inspect actual output before proceeding.
- Avoid dangerous operations on protected/shared refs; use disposable `lab/*` branches.
- Keep revision of `fetch`, `log A..B`, `log A...B`, `diff`, HEAD/index/working tree, ancestry, branches and upstream relations interspersed throughout lessons.
- Interactive quiz options should start unselected; assess answers only after user submission.

### Latest observed local state — 5 November 2026

```text
Current branch: lab/stash-practise
Working tree:  clean (git status -sb printed only ## lab/stash-practise)
Stash list:    stash@{0}: On lab/conflict-dev-b: Temporary IntelliJ configuration before conflict lab
IDE files:     .idea/ ignored locally by .git/info/exclude
```

`lab/stash-practise` is spelled **practise** in the user's actual branch (not `practice`). No exercise-specific local modifications remain. Older IntelliJ stash deliberately retained; do not blindly `pop`/`drop` it. The stash stack numbering can change as new stashes are created/deleted.

### Phase 2 topics completed so far

- Release maintenance branch rooted at `v1.0.0`; simulated hotfix release `v1.0.1` with GitHub CI/PR controls.
- Forward-port of a release-only hotfix to `main` by cherry-pick.
- Merge and rebase feature integration, graph/parent and tree comparison.
- Non-fast-forward push rejection, stale lease protection and competing developer preservation.
- Deliberate *merge* conflict and *rebase* conflict, including index stages 1/2/3.
- Ignoring local IntelliJ metadata and comprehensive stash workflows.

### Distinguish business process from Git mechanics

**Branch ≠ environment. Tag ≠ deployment. GitHub Release ≠ deployable artifact.** An authentic production deployment needs a tested built artifact (JAR/image digest), source SHA, release provenance, approval and rollback plan. Our lab simulated the source/release-control steps only.

---

## 2. Maintenance release and production-hotfix simulation

### The business problem

Release `v1.0.0` is a fixed production *simulation* baseline. New features may accumulate on `main`, but an urgent patch must not accidentally ship those unreleased changes.

```text
v1.0.0  a8174ae  (fixed tagged source baseline)
   |\
   | \  main advances with PAY-201 and later forward-port
   |
   +--- release/1.0 gets PAY-301 fix and CI maintenance update
                |
             v1.0.1
```

**Observed important refs and commits:**

| SHA/ref | Meaning |
|---|---|
| `a8174ae` / `v1.0.0` | Existing annotated initial release baseline |
| `5f17c38` | PAY-201 feature commit |
| `6b508e7` | PAY-201 GitHub PR #3 merge on `main` |
| `80f9878` | PAY-202 notification feature on `feature/pay-202-notifications`, **not merged to main** |
| `release/1.0` | Maintenance branch created from `v1.0.0` |
| `f12c3b8` | PAY-301 hotfix: reject unsupported currencies |
| `8043428` | PAY-301 GitHub PR #4 merge into `release/1.0` |
| `c27d829` | Update GitHub Actions PR targets to include `release/1.0` |
| `c1c0e88` | CI maintenance GitHub PR #5 merge into `release/1.0` |
| `v1.0.1` | Annotated patch release at `c1c0e88990d7cac5ff738d1a1440df01b59792b2` |
| `717a7d48` | Annotated **tag object** for `v1.0.1` (different from target commit) |

A `protect-release-1.0-ci` GitHub ruleset was configured active for the maintenance branch, requiring PR and the `validate` check, and protecting against force push/deletion. The `.github/workflows/pr-check.yml` PR trigger included both branches:

```yaml
on:
  pull_request:
    branches: [main, release/1.0]
```

**Why the workflow update mattered:** A required CI check must actually run for the relevant maintenance PR. A check designed only for `main` would not necessarily run for a PR targeting `release/1.0`.

The `v1.0.1` tag and GitHub Release were published for the simulated patch line. There was **no executable artifact and no production deployment**. Tag source and annotation are distinct objects:

```bash
git show --no-patch v1.0.1
git rev-parse 'v1.0.1^{}'       # Target commit, not tag object
git ls-remote --tags origin 'v1.0.1*'
```

**Operational rule:** Identify the release SHA/artifact in production first. Build the patch from that release lineage; do not take the latest `main` and assume unrelated features will be excluded.

---

## 3. Cherry-pick and forward-porting

The PAY-301 hotfix first landed on `release/1.0`. It also needed to appear on `main` so a later general release would not lose the fix.

We forward-ported the fix using:

```bash
git cherry-pick -x f12c3b8
```

- `cherry-pick` replays the changes introduced by a selected commit onto the current branch.
- `-x` appends a provenance line in the new commit message indicating the original cherry-picked SHA (when the cherry-pick records one).
- **Original hotfix SHA:** `f12c3b8` (maintenance branch).
- **New main-line SHA:** `42946a2` (the replayed hotfix).
- **GitHub PR #6 merge into `main`:** `989a9c4`.

Observed ancestry test:

```bash
git merge-base --is-ancestor f12c3b8 main
echo $?  # 1 — original hotfix commit is NOT a main ancestor

git merge-base --is-ancestor 42946a2 main
echo $?  # 0 — forward-ported commit IS a main ancestor
```

**Key point:** Equivalent content does not imply identical commit identity/ancestry. Cherry-pick creates a new commit, often with a different parent and SHA. Verify both **commit reachability** and **content/change inclusion** as appropriate.

### Backport and forward-port terminology

- **Backport:** bring a fix from a newer development line to an older supported line.
- **Forward-port:** bring a fix made on a maintenance/release line into the newer development line.
- **Don't cherry-pick a broad feature or entire merge history blindly** just to deliver one narrow hotfix.

---

## 4. Merge versus rebase on the same feature

We compared integration of PAY-202 against `main` at `989a9c4` using separate disposable branches. Original `feature/pay-202-notifications` remained at `80f9878`.

| Experiment | Result | Structure |
|---|---|---|
| Merge | `lab/pay-202-merge` → `c5ee284` | Merge commit with parents `80f9878` and `989a9c4` |
| Rebase | `lab/pay-202-rebase` → `a29a1f8` | PAY-202 replayed on `989a9c4`; new SHA |
| Final trees | `git diff --stat lab/pay-202-merge lab/pay-202-rebase` was empty | Same file snapshot, different history |

```text
MERGE:                              REBASE:
   80f9878 (PAY-202)                    a29a1f8 (PAY-202 replay)
      \                                 |
       c5ee284 (merge)                989a9c4 (main baseline)
      /
   989a9c4 (main baseline)
```

The graph above is schematic: in the merge branch, `c5ee284` has both feature and main as parents. Rebase rewrites/recreates the feature commit with a different parent, whereas merging ordinarily preserves the original feature commit and creates a merge commit when fast-forward is not possible.

Useful comparison commands:

```bash
git show --no-patch --format='%H%nParents: %P' c5ee284
git show --no-patch --format='%H%nParents: %P' a29a1f8
git diff --stat lab/pay-202-merge lab/pay-202-rebase
git log --graph --oneline --decorate --all
```

**Enterprise preference:** Shared branch history is generally safer to integrate without rewriting. Rebasing local unpublished feature work can simplify the graph; rebasing an already shared branch requires coordination and may require a force push.

---

## 5. Rejected pushes and `--force-with-lease`

### Why normal push was rejected

On `lab/force-lease-demo`, the remote initially pointed to `80f9878`. The local feature was rebased onto `main`, producing `3bcdb59`. This was no longer a fast-forward update of the remote branch.

Observed status conceptually included `[ahead 3, behind 1]`. A normal push was rejected as **non-fast-forward**. `ahead`/`behind` are differences in reachable **commits**, not lines/files.

### Why a lease is safer than blind force

We simulated a competing developer pushing `174502e` to the remote branch, adding `competing-update.txt`. Then our attempted `--force-with-lease` expected the earlier remote SHA `80f9878`, but the remote was now at `174502e`.

**Observed result:** push rejected with `(stale info)` — the lease prevented replacing another developer's newer commit.

A lease says, in effect: *overwrite only if the remote still points to the commit I believe it points to*. It is not permission to ignore another person's changes. In higher-risk cases, an explicit expected SHA is safer than assuming a recently updated remote-tracking ref represents the state you originally reviewed.

### Safe recovery used in our lab

```bash
git fetch origin
git log --oneline --left-right HEAD...origin/lab/force-lease-demo
git merge origin/lab/force-lease-demo
git push origin lab/force-lease-demo
```

We preserved both branches of work in merge commit `4a92a7d`:

```text
4a92a7d (merge)
├── 3bcdb59   rebased local work
└── 174502e   competing remote update
```

A normal push then succeeded; local and remote pointed to `4a92a7d`.

**Never reflexively run `git push --force` on shared/protected branches.** `--force-with-lease` is a useful guard, but the safest action in this scenario was to examine the unexpected remote commit and integrate it deliberately.

---

## 6. Merge conflicts: index stages and resolutions

We intentionally made two developers change the same line:

| Commit | Branch | File value |
|---|---|---|
| `6ec1b0c` | `lab/conflict-base` | `payment.retry.max-attempts=3` |
| `da162e5` | `lab/conflict-dev-a` | `payment.retry.max-attempts=5` |
| `199646c` | Developer B original | `payment.retry.max-attempts=7` |

From `lab/conflict-dev-b` (Developer B), we merged `lab/conflict-dev-a`:

```bash
git merge lab/conflict-dev-a
# CONFLICT (content): Merge conflict in retry-config.properties

git status -sb
# UU retry-config.properties
```

Conflict markers represented:

```text
<<<<<<< HEAD
payment.retry.max-attempts=7
=======
payment.retry.max-attempts=5
>>>>>>> lab/conflict-dev-a
```

`UU` means both sides modified the path and Git left it unresolved.

### Git's index stages in an unresolved conflict

```bash
git ls-files -u retry-config.properties
git show :1:retry-config.properties   # common base = 3
git show :2:retry-config.properties   # ours = 7
git show :3:retry-config.properties   # theirs = 5
```

During this **regular merge**, `ours` was the branch we were on (Developer B) and `theirs` was the incoming branch (Developer A).

We deliberately resolved the business decision to **5**:

```bash
printf 'payment.retry.max-attempts=5\n' > retry-config.properties
git diff --check
git add retry-config.properties
git diff --staged -- retry-config.properties
# -payment.retry.max-attempts=7
# +payment.retry.max-attempts=5

git merge --continue
```

**Observed merge commit:** `ddff429` (`lab/conflict-dev-b`) with final value `5`. It has **two parents**. After `git add`, unresolved stage 1/2/3 entries were replaced with a normal stage-zero index entry; `git ls-files -u` would no longer show the file.

The untracked `.idea/` directory was not staged or committed.

---

## 7. Rebase conflicts: reversed ours/theirs

We created disposable branch `lab/conflict-rebase` from the **original** Developer B commit `199646c`, not from the merged result:

```bash
git switch -c lab/conflict-rebase 199646c
git rebase lab/conflict-dev-a
```

Git paused while replaying Developer B's commit on top of Developer A:

```text
## HEAD (no branch)
UU retry-config.properties
```

`HEAD (no branch)` is normal in the paused replay state. The Git-aware prompt may display `rebase-i` even if the user did not explicitly invoke `git rebase -i`.

**Verified conflict stages:**

| Index stage | Rebase meaning | Value |
|---|---|---|
| `:1:` | Common ancestor | `3` |
| `:2:` / ours | Current new-base history — Developer A | `5` |
| `:3:` / theirs | Replayed Developer B commit | `7` |

Observed markers:

```text
<<<<<<< HEAD
payment.retry.max-attempts=5
=======
payment.retry.max-attempts=7
>>>>>>> 199646c (test: developer B sets retry attempts to seven)
```

**Critical rule:** During **rebase**, ours refers to the history being built on the new base and theirs to the commit being replayed. Thus `ours` and `theirs` can feel reversed relative to a normal merge. Check the content and index stages; don't blindly choose an IDE button or `--ours`/`--theirs`.

We resolved this simulation to **7** and continued the rebase. Verified result:

```text
* 6d43a4a (lab/conflict-rebase) Developer B: retry=7
* da162e5 (lab/conflict-dev-a) Developer A: retry=5
* 6ec1b0c (lab/conflict-base) Original: retry=3
```

Actual verification:

```bash
git show -s --format='%H%nParents: %P%nSubject: %s' HEAD
# New commit 6d43a4a9925068e0594c8d8da67ceb79509c423e
# Sole parent: da162e59df46c15b67b1bf1f3ff899f8359ad696

git merge-base --is-ancestor da162e5 HEAD
echo $?   # 0: yes

git diff lab/conflict-dev-b..lab/conflict-rebase -- retry-config.properties
# -payment.retry.max-attempts=5
# +payment.retry.max-attempts=7
```

The different outcomes (`5` for merge versus `7` for rebase) were **our deliberate choices**, not intrinsic requirements of either strategy.

### Continue, abort, skip

```bash
git rebase --continue  # after resolving and staging conflicts
git rebase --abort     # cancel and return to pre-rebase state
git rebase --skip      # omit currently replayed commit — inspect carefully!

git merge --continue   # complete an in-progress merge
git merge --abort      # cancel an in-progress merge
```

**Danger:** `--skip` can silently omit a wanted fix or feature when used as a reflex to a conflict.

---

## 8. Ignore rules, IntelliJ files and repository root

IntelliJ created `.idea/` under the lab repository. Initially it appeared as an untracked directory, and in an earlier attempt IntelliJ metadata interfered with the branch/merge exercise.

### Three concepts

| Category | Meaning |
|---|---|
| Tracked | File is recorded in Git's index/history; a new ignore pattern does **not** untrack it |
| Untracked | Git sees a new file not yet added |
| Ignored | Git suppresses it from ordinary untracked-file reporting based on rules |

We confirmed no `.idea/` files were tracked. We chose a **local-only exclusion** rather than committing a team-wide `.gitignore`:

```bash
printf '\n/.idea/\n' >> .git/info/exclude

git check-ignore -v .idea/workspace.xml
# .git/info/exclude:8:/.idea/ .idea/workspace.xml

git ls-files .idea/   # no tracked IntelliJ files
git status -sb        # clean
```

- Root `.gitignore` is an ordinary repository file; shared with teammates if committed.
- `.git/info/exclude` is local to this clone and not a normal project commit.
- User/global ignore rules can also be configured through `core.excludesFile`.
- A team may intentionally commit select IntelliJ code-style/run configuration files. Respect existing repository conventions instead of always ignoring `.idea/` wholesale.

### Why the `?? ./` output appeared

Some status commands were run while the shell's current directory was **inside `.idea/`**. Git's path display was relative to that directory. Verify the repository root with:

```bash
git rev-parse --show-toplevel
# /Users/daisy/IdeaProjects/git-enterprise-lab
```

`git rev-parse --top-level` was an invalid query choice for this purpose; it printed `--top-level` rather than the repository root.

---

## 9. Stash lifecycle: tracked and untracked files

Our disposable branch was named `lab/stash-practise` (the user chose British spelling). We edited `retry-config.properties` with a WIP comment and explored a complete stash lifecycle.

### Basic stash

```bash
git stash push -m 'WIP: retry timeout investigation'
git status -sb                       # clean
git stash list
git stash show -p 'stash@{0}'

git stash apply 'stash@{0}'         # restore, keep entry
git status -sb                       #  M retry-config.properties
git stash drop 'stash@{0}'          # delete saved entry; working edits survive
```

**`stash@{0}` is the newest stash *position*, not a stable ID.** After dropping an entry, the next one shifts to `stash@{0}`. Always check `git stash list` before deleting/applying.

### Stashing untracked files

We also created new untracked `retry-notes.txt` alongside the modified tracked file:

```bash
git stash push -u -m 'WIP: retry investigation with notes'
git stash show --stat 'stash@{0}'
git stash show --include-untracked -p 'stash@{0}'
```

- `-u` / `--include-untracked` includes untracked non-ignored files.
- Ordinary `git stash push` defaults to tracked-file changes; ignored files are still left alone.
- Default `stash show -p` / `--stat` may omit the untracked-file portion; use `--include-untracked` to inspect it.
- `git stash pop 'stash@{0}'` restored both files and dropped the stash on success; `retry-notes.txt` remained **untracked**, not automatically staged.

### Apply versus pop versus drop

| Operation | Restores edits | Removes stash on success |
|---|---|---|
| `stash apply` | Yes | No |
| `stash pop` | Yes | Yes |
| `stash drop` | No | Yes |

If stash application conflicts, Git normally keeps the stash. Never assume a failed `pop` cleaned everything up.

We intentionally preserved an older stash created during our IntelliJ conflict-lab setup: `Temporary IntelliJ configuration before conflict lab`.

---

## 10. Advanced stash: `--keep-index`, `--staged`, `--index`

We prepared three states concurrently:

```text
 M payment-validation.txt  # unstaged validation logging experiment
M  retry-config.properties # staged retry timeout WIP comment
?? retry-notes.txt          # untracked notes
```

**Status syntax reminder:** first column = index, second = working tree (`M ` staged, ` M` unstaged, `MM` both; `??` untracked).

### `--keep-index`: preserve staged work in working state

```bash
git stash push --keep-index -m 'WIP: validation logging experiment'
```

Observed state:

```text
M  retry-config.properties
?? retry-notes.txt
```

`payment-validation.txt` no longer showed as modified. **But** `git stash show -p 'stash@{0}'` showed *both* the staged retry change and the unstaged validation experiment. `--keep-index` says what to **leave in the current index/working tree**, not what must be excluded from the stash object.

### Restore just the needed file from this stash

Because the staged retry change was already present, blindly applying the full stash could overlap with index changes. We retrieved only the missing validation experiment:

```bash
git restore --source='stash@{0}' --worktree -- payment-validation.txt
```

- `--source` selects the saved tree snapshot.
- `--worktree` writes to the working directory **without changing the index**.
- `--` separates flags from file paths.
- **Caution:** file restoration replaces current working-tree content of that file. Verify it is safe before executing.

We then explicitly dropped the redundant new stash, leaving the older IntelliJ stash untouched.

### `--staged`: stash only staged changes

From the same three-state setup:

```bash
git stash push --staged -m 'WIP: staged retry configuration'
```

Observed status afterward:

```text
 M payment-validation.txt
?? retry-notes.txt
```

The new stash contained only the retry configuration change. We then used:

```bash
git stash apply --index 'stash@{0}'
```

This restored `retry-config.properties` to the **staged** state, leaving `payment-validation.txt` unstaged and `retry-notes.txt` untracked. Plain `apply` typically restores edits without recreating their original staging state; `--index` also attempts to restore the saved index snapshot. Verified `apply` left the stash entry intact, and we dropped only that redundant entry afterward.

### Final housekeeping, verified

We reviewed all three disposable modifications and cleaned up **specific files only**:

```bash
git restore --staged --worktree -- retry-config.properties
git restore --worktree -- payment-validation.txt
rm retry-notes.txt
git status -sb
# ## lab/stash-practise

git stash list
# stash@{0}: On lab/conflict-dev-b: Temporary IntelliJ configuration before conflict lab
```

**Do not casually use** `git reset --hard`, `git clean -fd` or `git stash clear` as a cleanup shortcut.

### Zsh tip learned during lessons

Zsh interactive sessions may treat pasted lines beginning with `#` as commands rather than comments, producing `zsh: command not found: #`. Enable:

```zsh
setopt interactivecomments
```

Optionally put it in `~/.zshrc`. This is a shell convenience, unrelated to Git state. The user also customized their prompt to display Git branch information without long hostname/directory text.

---

## 11. Revision command sheet

### Status, staging and recovery-safe inspection

| Command | Use |
|---|---|
| `git status -sb` | Branch and concise staged/unstaged/untracked status |
| `git diff` | Working tree versus index (unstaged edits) |
| `git diff --staged` | Index versus HEAD (what the next commit would include) |
| `git add path` | Stage just one path |
| `git restore --staged -- path` | Unstage while preserving file edits |
| `git restore --worktree -- path` | Discard unstaged tracked edits — destructive |
| `git rev-parse --show-toplevel` | Find repository root from nested directory |
| `git ls-files -- path` | Determine whether file is tracked |
| `git check-ignore -v -- path` | Show which ignore rule applies |
| `git ls-files -u -- path` | Show unresolved conflict stages |
| `git show :1:path`, `:2:path`, `:3:path` | Inspect base/ours/theirs during conflict |

### History, ancestry and integration

| Command | Use |
|---|---|
| `git fetch origin` | Update remote-tracking refs, without integrating into current branch |
| `git log --oneline A..B` | Commits reachable from B but not from A |
| `git log --oneline --left-right A...B` | Exclusive commits on both sides |
| `git diff A..B -- path` | Compare tip snapshot A to tip snapshot B |
| `git diff A...B -- path` | Compare merge-base snapshot to B |
| `git merge-base --is-ancestor A B; echo $?` | `0` if A is ancestor of B; otherwise `1` |
| `git show -s --format='%H%nParents: %P' SHA` | Identify full SHA and commit parent(s) |
| `git log --graph --oneline --decorate --all` | Inspect branch ancestry and merge topology |
| `git cherry-pick -x SHA` | Replay a selected change with provenance |
| `git merge BRANCH` | Integrate branch history; merge or fast-forward as appropriate |
| `git rebase UPSTREAM` | Replay local commits onto an upstream tip (rewrites commit identities) |

### Stash options at a glance

| Command | Effect |
|---|---|
| `git stash push -m 'description'` | Stash tracked staged + unstaged changes |
| `git stash push -u -m 'description'` | Also save untracked non-ignored files |
| `git stash push --keep-index` | Save changes while leaving staged changes in index/working tree |
| `git stash push --staged` | Save only index/staged changes |
| `git stash list` | See stash stack and descriptive messages |
| `git stash show --include-untracked -p 'stash@{0}'` | Inspect tracked and stashed untracked patch |
| `git stash apply 'stash@{0}'` | Restore without dropping entry |
| `git stash apply --index 'stash@{0}'` | Also attempt restoration of original index state |
| `git stash pop 'stash@{0}'` | Apply; drop on success |
| `git stash drop 'stash@{0}'` | Delete saved entry, not already-restored working edits |
| `git restore --source='stash@{0}' --worktree -- FILE` | Restore one file from stash snapshot |

**Subtleties:** `git log A...B` is a *symmetric difference of commit sets*. `git diff A...B` is generally *merge-base-to-B snapshot difference*; don't transfer the same mental model between them. After cherry-picking, same content can exist under different commit SHAs.

---

## 12. Safety rules, unresolved items and next lessons

### Practical enterprise habits

1. Start with `git status -sb`; know the current branch and whether changes are staged, unstaged or untracked.
2. Fetch and compare incoming/outgoing history before integrating shared work.
3. Commit/PR narrowly focused changes; inspect `git diff --staged` first.
4. Prefer protected branches and required CI; do not rewrite `main` or maintenance history.
5. Treat published tags as immutable, and tie real releases to artifact digests and source commits.
6. When a conflict occurs, inspect `git status`, markers and index stages; choose the correct business resolution rather than treating ours/theirs as a universal winner.
7. For valuable unfinished work, `apply` + review + `drop` can be safer than immediately `pop`.
8. Recheck stash identities before referencing `stash@{0}`; entries shift after push/drop.
9. Never run broad destructive cleanup on unfamiliar repos. Avoid `git add .` when the index might capture IDE/runtime files.
10. In IntelliJ multi-root workspaces, check which repository/root an action affects. CLI operations and IntelliJ UI both act on the same underlying Git state.

### Outstanding items

- **Older IntelliJ stash:** intentionally left saved, not reviewed/restored/deleted.
- **Future lesson:** controlled recovery from mistaken reset, lost commit or accidental history rewrite; introduce `git reflog`, plus careful `git reset`/`git revert` distinctions.
- **Later Phase 2/3 subjects:** release automation, production-grade artifact provenance, trunk-based vs GitFlow tradeoffs, interactive rebase, `git bisect`, worktrees and audit/recovery drills as needed.
- **No claim of running a real application deployment:** both releases were source/release simulations.

### Next lesson checkpoint: recovery with reflog

Begin from the **clean** `lab/stash-practise` branch, but use a **fresh disposable branch** for recovery tests. Preserve existing tags, protected `main`, release branch and the IntelliJ stash. Work in small checkpoints; do not run destructive commands before inspecting exact refs and outputs.

---

*This Phase 2 continuation records the exercises conducted in chat through 5 November 2026. It is intended to live alongside the Phase 1 `README.md` and to be updated as additional lessons are completed.*
