# DevOps Engineer's Notebook
## Days 7–9 · Git & GitHub Mastery
### Owner: Sahil | Roadmap: 3 months · 2 hrs/day | Week 2

---

## 📋 Contents

| Day | Topic | Key Skill |
|-----|-------|-----------|
| 7   | Git Basics + SSH Auth + First Push | Push local repo to GitHub via SSH |
| 8   | Branches + Merging | The branch lifecycle (create → work → merge → delete) |
| 9   | Pull Requests + Merge Conflicts | Real team workflow + conflict resolution |

---

# DAY 7 — Git Foundations + SSH to GitHub

## Mental Model: What Git Actually Solves

Without Git: `backup.sh` → `backup_v1.sh` → `backup_final.sh` → `backup_final_FINAL_v2.sh` (the nightmare)

🎯 **Git's job**: Track every version of every file in a folder, with a message explaining *why* each change was made, so you can travel back in time.

| Term | What it means |
|------|---------------|
| **Repository (repo)** | A folder Git is watching. Has hidden `.git/` inside. |
| **Commit** | A snapshot of all files at one point in time, with a message. |
| **Remote** | A copy of your repo on another machine (e.g. GitHub). |

---

## The 4 File States

```
Untracked  →  Modified  →  Staged  →  Committed
              git add ↑    git commit ↑
```

🎯 **Mental model: shopping cart**
- Working directory = whole store (everything you've touched)
- Staging = what you've put in the cart
- Commit = checkout (you pay, get a receipt with a message)

You can edit 5 files but only `git add` 2 → only those 2 get committed.

---

## Core Commands (Day 7)

```bash
# Identity setup (one-time, per machine)
git config --global user.name "yourname"
git config --global user.email "you@example.com"
git config --global init.defaultbranch main
git config --global core.editor vim         # or nano

# Repo lifecycle
git init                                    # make folder a repo
git status                                  # what's the state?
git add <file>                              # stage changes
git add .                                   # stage everything
git commit -m "msg"                         # commit with message
git commit                                  # opens editor for longer msg
git log --oneline                           # see commit history
```

---

## SSH Keys for GitHub

🎯 **The Universal SSH Key Rule** (from Day 4 notes, applied for real):
> Whoever wants access GIVES their PUBLIC key.
> Whoever grants access INSTALLS that public key.

You = wants access to GitHub. You give your **public** key. GitHub stores it.
Your **private** key never leaves your laptop. Ever.

### Generate keys

```bash
ssh-keygen -t ed25519 -C "your@email.com"
# Press Enter 3 times to accept defaults + skip passphrase
```

Creates two files in `~/.ssh/`:
- `id_ed25519` → PRIVATE (never share, chmod 600)
- `id_ed25519.pub` → PUBLIC (safe to share, paste anywhere)

### Verify permissions (SSH enforces these strictly)

| File | Required perm | Why |
|------|---------------|-----|
| `~/.ssh/` | 700 | Only you can enter |
| `~/.ssh/id_ed25519` | 600 | Only you can read private key |
| `~/.ssh/id_ed25519.pub` | 644 | Anyone can read public key |

### Add public key to GitHub

```bash
cat ~/.ssh/id_ed25519.pub      # copy this output
```
Paste at: github.com → Settings → SSH and GPG keys → New SSH key

### Test the connection

```bash
ssh -T git@github.com           # note: -T is UPPERCASE
# Success message (looks like error but isn't):
# "Hi <username>! You've successfully authenticated, but
#  GitHub does not provide shell access."
```

🚨 **Linux flags are case-sensitive**: `-t` ≠ `-T`. Read errors carefully.

---

## Push Local Repo to GitHub

```bash
# 1. Create EMPTY repo on github.com (no README/gitignore/license)
# 2. Link local → remote:
git remote add origin git@github.com:USER/REPO.git
git remote -v                              # verify

# 3. First push (with -u to set upstream):
git push -u origin main

# 4. After first push, just:
git push                                   # done. -u set it forever.
git pull                                   # pull changes from GitHub
```

🚨 **Why "empty repo"**: If GitHub creates a README, GitHub has 1 commit your local doesn't. Push gets rejected with "divergent history" → 20 min debugging.

---

## .gitignore — What NOT to Commit

🚨 **Real cost**: Bots scan GitHub 24/7 for leaked `.env` files. Within minutes of a commit with AWS keys, your account is mining crypto for strangers.

```
# Secrets
.env

# Logs  (note the asterisk — pattern matters!)
*.log

# Dependencies
node_modules/
venv/
__pycache__/

# OS junk
.DS_Store
Thumbs.db
```

### Glob patterns (same as Day 6 find/grep)

| Pattern | Matches |
|---------|---------|
| `*` | Any string |
| `*.log` | `debug.log`, `nginx.log`, `app.log` |
| `.log` | ONLY a file literally named `.log` |
| `?` | Exactly one character |
| `node_modules/` | Folder + everything inside |
| `**/temp/` | Any `temp/` folder at any depth (gitignore-specific) |

🚨 **The gotcha I hit**: Wrote `.log` instead of `*.log`. Missing the wildcard means only the literal name matches. Same regex-dot lesson from Day 6.

---

## Commit Message Hygiene

🚨 **DevOps reality**: At a real job, commit messages aren't decoration:
- Senior reads commits before opening the diff
- Engineer debugging a 2AM outage runs `git log` to find when bugs introduced
- `git blame` shows "who changed this line and why?" 6 months later

A bad commit message = unsolvable mystery in 6 months.

### The 3 Rules

1. **Imperative mood, not past tense.** ✅ `Add gitignore` ❌ `added gitignore`
   Read it as "this commit will _____" — like a command.
2. **Under 50 characters for first line.** Forces brevity.
3. **Say WHAT and WHY, not HOW.** Diff already shows HOW.

### Conventional commit format

```
<type>: <short description>
```

| Type | Use when |
|------|----------|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `chore` | Maintenance (gitignore, deps, configs) |
| `refactor` | Code change, no behavior change |
| `test` | Adding tests |
| `ci` | CI/CD pipeline changes |

### Bad → Good

| Bad | Good |
|-----|------|
| `added gitignore` | `chore: add .gitignore for env, logs, node_modules` |
| `update README` | `docs: add Day 9 section to README` |
| `fix stuff` | `fix: handle empty input in backup.sh` |

---

# DAY 8 — Branches & Merging

## Why Branches Exist

🎯 **Scenario**: Manager says "add encryption to `backup.sh` but don't break Sunday's production backup."

Without branches: edit directly = production broken for days, OR copy file = `_v2` nightmare.

With branches:
```
main:      ●───●───●  ← stable, production backup.sh
                  \
feature:           ●───●───●  ← your encryption experiment
```

🎯 **One word to internalize**: **isolation**. Same idea in Docker (containers), K8s (namespaces), AWS (accounts). Different tools, same concept.

🚨 **DevOps reality at real jobs**:
- `main` = production (customers using it)
- Feature branches = work-in-progress
- You **never** commit directly to main. Always: branch → work → PR → merge.

---

## Branch Commands

```bash
# Create + switch
git branch <name>                  # create branch (don't switch)
git switch <name>                  # switch to existing branch
git switch -c <name>               # create AND switch (shortcut — use this)
git checkout <branch>              # OLD way (do too much, prefer `switch`)

# Status check (do this BEFORE every commit)
git status                         # shows current branch (line 1)
git branch                         # lists branches, * = current
git branch -a                      # includes remote-tracking

# Push a new branch (first time)
git push -u origin <branch>

# Useful visualization
git log --oneline --all --decorate # see all branches + commits
```

### Branch naming conventions

| Pattern | Use for |
|---------|---------|
| `feature/<name>` | New features (`feature/add-encryption`) |
| `fix/<name>` | Bug fixes (`fix/backup-empty-dir-crash`) |
| `hotfix/<name>` | Urgent prod fixes |
| `chore/<name>` | Maintenance |

🎯 Slash groups branches in GitHub UI. Lowercase-with-hyphens, no spaces.

---

## The Merge

🚨 **The #1 fresher mistake**: Merging in the wrong direction.

🎯 **Rule**: You merge INTO the branch you're standing on.

```bash
# To merge feature → main:
git switch main                    # ⚠️ FIRST go to destination
git merge feature/add-encryption   # pull feature INTO main
git push                           # push merged main
```

### Two types of automatic merges

| Type | When | Result |
|------|------|--------|
| **Fast-forward** | main hasn't changed since branch | Just moves main pointer forward. No new commit. |
| **Merge commit** | Both branches moved forward | Creates a new "merge commit" tying them together |

---

## Cleanup After Merge

🎯 **Why delete merged branches**: A repo with 200 stale branches = maintenance nightmare. Real teams enforce cleanup.

```bash
# Delete local branch
git branch -d <branch>             # SAFE — refuses if unmerged
git branch -D <branch>             # FORCE — destructive, use sparingly

# Delete remote branch
git push origin --delete <branch>

# Verify both gone
git branch -a
```

### The `-d` safety check

🚨 `-d` refuses to delete if the branch isn't merged into `origin/main` (GitHub's main). This is **GitHub-as-backup protection**.

The fix: `git push` your local main first → THEN `-d` succeeds.

```bash
# Production-correct cleanup pattern:
git switch main
git push                           # ship merged main to GitHub
git branch -d feature/<name>       # safe to delete now
```

---

## ⚠️ Uncommitted Changes Follow You Across Branches

🚨 **The single most dangerous Git behavior for beginners.**

```bash
# I'm on feature branch, edit README.md, DON'T commit
git switch main
# Git output: "M  README.md" — that's WARNING, not error
git status                         # README.md shows modified ON MAIN
```

🎯 **Why**: Working directory is "live." Uncommitted changes aren't attached to any commit, so they float and travel with you across branches.

**Real production risk**: You're on `feature/payment-fix`, 30 lines of uncommitted debug code. Manager: "URGENT, deploy hotfix to main!" You `git switch main`, your debug code comes with you, you commit thinking it's part of the hotfix. You just shipped untested payment code to prod. 💥

🎯 **The discipline**: **Always commit (or stash) before switching branches.**

```bash
git commit -m "WIP: in-progress"   # commit even if half-done
# OR
git stash                          # park changes temporarily
```

---

# DAY 9 — Pull Requests & Merge Conflicts

## Pull Requests — The Real Team Workflow

🎯 **What is a PR?** A formal request to merge a branch into another (usually main), with a review step before the merge.

🚨 **DevOps reality**: At real jobs, you never local-merge to main. Production repos have **branch protection rules**:
- ❌ Can't push directly to main
- ✅ Must open PR
- ✅ Reviewer must approve
- ✅ CI/CD tests must pass

### PR workflow

```bash
# 1. Create branch, do work, commit
git switch -c feature/new-thing
# (edit files)
git add .
git commit -m "feat: add new thing"

# 2. Push branch
git push -u origin feature/new-thing
# GitHub gives URL to open PR

# 3. On GitHub:
#    - Verify base = main, compare = feature
#    - Fill title (use conventional commit format)
#    - Fill description: What / Why / How to verify
#    - Click Create pull request

# 4. (Self-)review on "Files changed" tab
# 5. Pick merge strategy → Confirm merge
# 6. Delete branch (GitHub button or via CLI)
```

### PR description format

```markdown
## What
Added a new X to do Y.

## Why
Reason this change was needed.

## How to verify
- Step 1
- Step 2
- Expected behavior
```

🚨 Even on solo learning repos, **practice this format**. The habit doesn't materialize at your first job — it has to be built now.

---

## The 3 Merge Strategies

| Strategy | History looks like | When to use |
|----------|-------------------|-------------|
| **Merge commit** | Branchy Y-shape, shows parallel work | When branch commits are individually meaningful |
| **Squash and merge** | Linear — 1 PR = 1 commit on main | ✅ **Modern team default** (Flipkart, Razorpay, Atlassian) |
| **Rebase and merge** | Linear, all commits preserved | Open source style (Linux kernel, K8s) |

🎯 **Use squash-merge by default** for portfolio repos. Reasons:
1. Clean readable main history
2. Sloppy WIP commits get cleaned up
3. Reverting a feature = revert one commit
4. GitHub auto-appends `(#1)` PR number → traceability

---

## Staying in Sync: fetch vs pull

| Command | What it does |
|---------|--------------|
| `git fetch` | Downloads info from GitHub. **Doesn't change files.** Updates `origin/*` cache. |
| `git fetch --prune` | Same + removes cached refs of deleted remote branches |
| `git pull` | `fetch` + merge into current branch (two operations in one) |

🎯 **Mental model**:
- `fetch` = "tell me what's on GitHub, don't touch my files"
- `pull` = "fetch AND update my current branch"

🚨 Seniors often `git fetch` first, inspect, *then* decide whether to pull. Why? `pull` does a merge → can fail with conflicts. `fetch` is always safe.

### Why `origin/main` and `main` differ

Your local `origin/main` is a **cache** — a snapshot of GitHub's main *at your last sync*. If someone pushed to GitHub 5 min ago, your local cache doesn't know until you fetch.

---

## Merge Conflicts

🎯 **One-sentence definition**: A merge conflict happens when Git can't decide which version of a line to keep because two branches both changed the same line in the same file.

### When conflicts happen — and when they DON'T

| Scenario | Conflict? |
|----------|-----------|
| Branch A adds line 5, branch B adds line 20 to same file | ❌ Different lines |
| Branch A modifies line 5, branch B adds new file | ❌ Different files |
| Branch A modifies line 5, branch B modifies line 5 differently | ✅ **YES** |
| Branch A deletes file, branch B modifies same file | ✅ Special "delete vs modify" |

---

## Conflict Markers

When Git can't auto-merge, it writes BOTH versions into the file with markers:

```
<<<<<<< HEAD
# Production DevOps Repo - Main Branch
=======
# DevOps Learning Repo - Feature Branch Version
>>>>>>> feature/modify-readme
Day 8 of learning devops
```

| Marker | Meaning |
|--------|---------|
| `<<<<<<< HEAD` | Below this line: version from HEAD (current branch) |
| `=======` | Divider |
| `>>>>>>> branch-name` | Above this line: version from the other branch |

🎯 **Git's message**: *"I found a disagreement. Both versions are in your file. You delete what you don't want and remove the markers."*

---

## Resolving a Conflict — Step by Step

```bash
# 1. Attempt the merge — it fails with CONFLICT
git switch main
git merge feature/modify-readme
# CONFLICT (content): Merge conflict in README.md

# 2. Check status — see "both modified" (special phrase)
git status
# Unmerged paths:
#   both modified:   README.md

# 3. Open the file, see the markers, EDIT:
vim README.md
#    - Decide which content to keep (or write new)
#    - DELETE all 3 marker lines (<<<<<<<, =======, >>>>>>>)
#    - Save

# 4. VERIFY no markers remain
grep -E "^(<<<|===|>>>)" README.md     # empty output = clean
grep -rE "^(<<<|===|>>>)" .             # recursive check across repo

# 5. Tell Git the conflict is resolved
git add README.md
git status                              # "All conflicts fixed but still merging"

# 6. Conclude the merge
git commit                              # auto-generated msg in editor
# (or `git commit --no-edit` to skip the editor)
```

---

## Conflict Resolution Options

| Option | What to do | Result |
|--------|-----------|--------|
| **A. Keep main's version** | Delete feature's lines + markers | main's content wins |
| **B. Keep feature's version** | Delete main's lines + markers | feature's content wins |
| **C. Keep both** | Delete only markers, keep both content | Both lines stacked |
| **D. Write new** | Delete everything between markers, write fresh | Your synthesized version |

🚨 **Real DevOps reality**: Option D is most common at real jobs. When two devs disagreed, often neither version is right — you write a third version that captures both intents.

---

## Merge Commits Have TWO Parents

After resolving and committing a merge, `git show HEAD` reveals:

```
commit cf58878...
Merge: bdfa320 4ef923c       ← TWO parent commits!
```

🎯 **Only merge commits have two parents.** Every other commit has one. This Y-shape is structurally unique in your repo's history.

```
                      cf58878 (main)  ← merge commit
                        ●
                       / \
              bdfa320 ●   ● 4ef923c
               "main"     "feature"
                     \   /
                      ● 63f5d00 (common ancestor)
```

---

## ⚠️ Production Disasters to Avoid

🚨 **Never commit conflict markers to production code.**
`<<<<<<<` is not valid syntax in any programming language. App crashes immediately. Your team mocks you. Don't be that engineer — ALWAYS grep before claiming "done."

🚨 **Never `kill -9` a Git operation mid-merge.**
Leaves your repo in a "stuck merging" state. To escape:
```bash
git merge --abort                  # cancel the in-progress merge
```

---

# Top Gotchas (Days 7–9)

| Gotcha | Lesson |
|--------|--------|
| Typed `yes` to non-yes-no prompt | Text in `(parens)` is the DEFAULT — press Enter to accept |
| `.log` vs `*.log` in gitignore | Always wildcard with `*`. Same as Day 6 regex/find. |
| Lowercase `-t` vs uppercase `-T` | Linux flags are case-sensitive. Read errors. |
| Uncommitted changes follow you | **Commit/stash before switching branches** |
| Merged in wrong direction | You merge INTO the branch you're standing on |
| Left conflict markers in file | grep before claiming resolved |
| `git branch -d` refuses to delete | Push merged main first, THEN delete |
| Forgot to verify with `git status` | Trust but verify (Day 6 principle still applies) |

---

# Mental Models That Compound

1. **Principle of Least Privilege** (Day 1) → SSH keys: only your private key on your laptop, public goes everywhere safely.

2. **Trust but verify** (Day 6) → `git status` before commit. `grep` for markers after resolve. `git log --all --decorate` to confirm state.

3. **Unix Philosophy** (Day 6) → `git` is dozens of small tools (`add`, `commit`, `push`, `log`, `diff`). Each does one thing. Pipe them together for workflows.

4. **Isolation** (Day 8) → Branches isolate work. Same pattern in Docker, K8s namespaces, AWS accounts.

5. **Read what the tool tells you** (Day 9) → Git's error messages are *instructions*. "use 'git commit' to conclude merge" is the literal next step.

---

# Top Commands Reference Card (Days 7–9)

| Category | Commands |
|----------|----------|
| **Setup** | `git config --global user.name/email`, `git config --global core.editor` |
| **Repo** | `git init`, `git status`, `git log --oneline --all --decorate` |
| **Stage/commit** | `git add <file>`, `git add .`, `git restore <file>`, `git commit -m "msg"` |
| **Branches** | `git branch`, `git branch -a`, `git switch -c <name>`, `git switch <name>`, `git branch -d <name>` |
| **Remote** | `git remote add origin <url>`, `git remote -v`, `git push -u origin <branch>`, `git push`, `git push origin --delete <branch>` |
| **Sync** | `git fetch`, `git fetch --prune`, `git pull` |
| **Merge** | `git merge <branch>`, `git merge --abort`, `git commit` (after resolving) |
| **SSH** | `ssh-keygen -t ed25519 -C "email"`, `ssh -T git@github.com`, `cat ~/.ssh/id_ed25519.pub` |
| **Inspect** | `git log`, `git diff`, `git show HEAD --stat`, `git show HEAD` |

---

*Days 7–9 · Week 2 of 12 · DevOps Engineer Roadmap · Sahil's Notebook*

