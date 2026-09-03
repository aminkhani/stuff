```table-of-contents
title: **Table of Contents**
style: nestedList # TOC style (nestedList|nestedOrderedList|inlineFirstLevel)
minLevel: 0 # Include headings from the specified level
maxLevel: 0 # Include headings up to the specified level
includeLinks: true # Make headings clickable
hideWhenEmpty: false # Hide TOC if no headings are found
debugInConsole: false # Print debug info in Obsidian console
```
---
## 🧩 What is Git?

**Git** is a **content-addressed object database** with a thin porcelain of commands bolted on top. Every file version, directory listing and commit is stored **once**, keyed by the hash of its own content. Branches are not copies of anything — a branch is a 41-byte text file holding one commit hash.

Think of a stack of numbered photographs 📸 of the whole project. Each photo (**commit**) says *"this is what everything looked like, and the photo before me was #abc123"*. A **branch** is a sticky note you move from photo to photo.
---
## 💡 Why do we need it?
- 🕰️ **Reversible history** — any past state is one command away, and `reflog` recovers even the states you thought you destroyed.
- 🌿 **Free branching** — a branch is one file containing a hash, so isolating a feature costs nothing.
- 🤝 **Concurrent work** with a real merge algorithm instead of `project_final_v3_ACTUAL.zip`.
- 🔍 **Forensics** — `blame`, `log -S`, `bisect` answer "who broke this, when, and in which commit".
- 🚦 **The substrate for CI/CD** — every pipeline, review gate and deploy keys off a commit SHA ([CI/CD](../CI-CD/README.md)).
---
## ⚙️ How it works

Four immutable object types, each named by the hash of its own content:

```text
blob     file CONTENT only — no name, no path, no mode
 ^
tree     directory listing: mode + type + hash + name; points at blobs and other trees
 ^
commit   one root tree + parent commit(s) + author/committer + message
 ^
ref      .git/refs/heads/main -> a commit hash.  The branch IS this file; HEAD points at the ref.
```

Identical content means the identical blob, stored once, however many commits or paths reference it. A commit does **not** store a diff — it stores a full **snapshot** (a tree); diffs are computed on demand. This is why branching, tagging and `checkout` are all cheap, and why history is a DAG rather than a list.

**The three areas** — every day-to-day command is just moving data between them:

| Area | Physical form | Moved by |
|---|---|---|
| **Working tree** | your actual files on disk | your editor, `git restore <path>` |
| **Index / staging** | `.git/index` — a binary list of blobs + modes | `git add`, `git restore --staged` |
| **HEAD** | a ref pointing at the current commit | `git commit`, `git switch`, `git reset` |

`git status` is a three-way diff across all three. `git diff` = working tree vs index; `git diff --staged` = index vs HEAD.
---
## 🌿 merge vs rebase vs squash
| Operation | History shape | New SHAs? | Safe on a pushed branch |
|---|---|---|---|
| **fast-forward merge** | ref slides forward, no merge commit | no | ✅ (only possible with no divergence) |
| **`merge --no-ff`** | both lines preserved + a merge commit with two parents | no | ✅ always |
| **`rebase`** | your commits replayed onto a new base — linear, **new SHAs** | yes | ❌ never on a shared branch |
| **squash merge** | one new commit containing the whole branch diff | yes (collapsed) | ✅ for the target; the source branch is now orphaned — delete it |

Verdict: **rebase your own local feature branch** onto `main` before opening the PR (linear history, bisect-friendly), then **merge or squash** to integrate. Rebasing or force-pushing a branch someone else has already pulled gives them duplicated commits and a manual mess — that is the one hard rule.
---
## 🔄 Undo: reset vs revert vs restore vs checkout
| Command | HEAD | Index | Working tree | Loses work? | OK on pushed commits |
|---|---|---|---|---|---|
| `git reset --soft <c>` | moves | untouched | untouched | no | ❌ rewrites history |
| `git reset --mixed <c>` *(default)* | moves | reset to `<c>` | untouched | staged state | ❌ |
| `git reset --hard <c>` | moves | reset | **overwritten** | **yes** | ❌ |
| `git revert <c>` | new commit on top | — | — | no | ✅ the safe undo |
| `git restore <path>` | — | — | file rewritten from index | uncommitted edits | ✅ |
| `git restore --staged <path>` | — | unstages | untouched | no | ✅ |
| `git switch <branch>` | moves to another ref | — | checked out | no (refuses when dirty) | ✅ |

`git checkout` does the jobs of both `switch` and `restore`, which is why everyone was confused for a decade — prefer the newer pair (Git ≥ 2.23). **`git reflog` is the undo net:** every movement of `HEAD` is journaled for ~90 days (`gc.reflogExpire`), so even a bad `reset --hard` is recoverable. A commit is only truly gone once it is unreachable **and** `gc` has run.
---
## 🛠️ The rest of the toolbox
- **`cherry-pick <sha>`** — replay one commit elsewhere (new SHA). Right for a hotfix; a *habit* of it means your branching model is wrong.
- **`stash`** — `git stash push -u` (`-u` also stashes untracked files, the usual surprise), then `git stash pop`. A shelf, not storage.
- **`bisect`** — binary search over history for the commit that introduced a bug: `log₂ n` builds instead of `n`.
- **`worktree add ../hotfix main`** — a second checkout of the same repo in another directory sharing one `.git`. Beats cloning twice or stashing just to glance at another branch.
- **Tags** — annotated (`git tag -a v1.4.2 -m "..."`: a real object with author, date, message, signable) vs lightweight (a bare ref). Follow **SemVer** — MAJOR breaking / MINOR features / PATCH fixes — and build releases from tags, not branches.
- **`.gitignore` vs `.gitattributes`** — the first decides what Git *ignores*; the second decides how Git *treats tracked files* (line endings, diff/merge drivers, `filter=lfs`, `linguist-*`). Constantly confused, completely different jobs.
- **Hooks** live in `.git/hooks` and are **not cloned** — use the **`pre-commit`** framework (`.pre-commit-config.yaml` + `pre-commit install`) so lint, format and secret-scan run identically for everyone and again in CI.
- **Big repos** — `--depth=1` (shallow, for CI), `--filter=blob:none` (blobless/partial clone), `sparse-checkout` (a subset of paths). Combine the last two for monorepos.
- **submodule vs subtree vs monorepo** — a submodule pins another repo at a SHA (explicit, but easy to forget and everyone needs `--recurse-submodules`); a subtree vendors the code into your tree (nothing extra for clients, messier upstreaming); a monorepo removes the problem and adds build-tooling complexity.
---
## ⚖️ Branching strategies
| Strategy | Branches | Cadence | CI fit | Pick it when |
|---|---|---|---|---|
| **Trunk-based** | `main` + very short-lived branches, feature flags | continuous | ideal — everyone integrates daily | mature CI, strong test suite, SaaS |
| **GitHub Flow** | `main` + feature branch → PR → deploy on merge | continuous | very good | small/medium teams, one production environment |
| **GitLab Flow** | GitHub Flow + `staging`/`production` environment branches | promotion per environment | good | you need explicit staging→prod gates or regulated releases |
| **Git Flow** | `develop`, `feature/*`, `release/*`, `hotfix/*`, `main` | scheduled versioned releases | heavy — long-lived branches, painful merges | installable software with several supported versions in the field |

Overhead and merge pain both grow straight down the table. Default to **GitHub Flow**; move to trunk-based once your tests are good enough to earn it.
---
## 🧪 Example
Recovering a branch you just destroyed:
```bash
git reflog                     # 8f2a1c9 HEAD@{3}: commit: add invoice export
git branch rescue 8f2a1c9      # safest: point a NEW branch at the lost commit
git reset --hard 8f2a1c9       # or move this branch back - overwrites the working tree
```
Cleaning up a feature branch before review:
```bash
git rebase -i origin/main      # editor opens: pick / reword / squash / fixup / drop, oldest first
git push --force-with-lease    # NOT --force: refuses if someone else pushed in the meantime
```
Hunting a regression across 500 commits:
```bash
git bisect start
git bisect bad                 # current HEAD is broken
git bisect good v1.3.0         # this tag was fine
git bisect run pytest -x tests/test_invoices.py   # automated: ~9 steps for 500 commits
git bisect reset               # always finish with this
```
Fixing Windows line endings once, for everyone:
```ini
# .gitattributes - commit this; it overrides every developer's local core.autocrlf
* text=auto eol=lf
*.sh text eol=lf
*.ps1 text eol=crlf
*.png binary
```
---
## 🚨 When NOT to use it / limits
- ❌ **Large binaries** — every version of a 200 MB asset lives in every clone forever, and history cannot be slimmed without rewriting it. Use **Git LFS** (pointer files) or an artifact store; never commit build output, `node_modules/` or `*.sqlite3`.
- ❌ **As a deployment mechanism.** `git pull` on a production box means no build step, no immutable artifact, no atomic rollback — and a `.git` directory sitting on a web host. Build an artifact, deploy the artifact ([CI/CD](../CI-CD/README.md)).
- ❌ **Storing secrets or generated files** — history is forever and public repos are scraped within minutes of a push.
- ❌ **Force-pushing a shared branch** — you are rewriting other people's history. Protect `main` and require a PR instead.
- ❌ **Hundreds of GB or millions of files** without partial clone / sparse-checkout — plain Git will crawl.
---
## 🔐 Security notes & production hardening

> [!IMPORTANT]
> **A secret committed once is compromised, even if the next commit removes it.** Order matters: **(1) rotate the credential** — it already exists in every clone, fork, CI cache and forge backup; **(2)** then scrub history with `git filter-repo --invert-paths --path .env` (or BFG); **(3)** force-push and have every collaborator re-clone. Step 2 rewrites every downstream SHA, so it needs coordination — and it never un-leaks the value. Rotation is the fix; scrubbing is cleanup.

- 🔎 **Scan before the commit exists** — `gitleaks` or `detect-secrets` as a `pre-commit` hook *plus* a CI job. Local hooks are bypassable with `--no-verify`, so the server-side check is the real control.
- 🚫 **An exposed `.git/` directory leaks your whole source tree** — a classic pentest finding: fetch `.git/` recursively, run `git checkout .`, and you have every file *and every old commit*, credentials included. Block it at the web server ([Nginx](../../SoftwareDesign/WebServer/Nginx.md)):
```nginx
location ~ /\.(git|env|hg|svn) { deny all; return 404; }
```
- ✍️ **Sign commits and tags.** `author` and `committer` are **free text you set yourself** — `git config user.email ceo@corp.com` is the entire "attack". A signature binds the commit to a key you control:
```bash
git config --global gpg.format ssh                              # SSH signing needs Git >= 2.34
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true
git tag -s v1.4.2 -m "release 1.4.2"    # verify in CI with: git verify-tag v1.4.2
```
- 🛡️ **Protect branches** on the forge: no force-push and no deletion on `main`, required reviews, required status checks, linear history if you rebase, and **CODEOWNERS** so security-relevant paths cannot merge without the right reviewer.
- 🔑 **Least privilege for machines** — per-repo **read-only deploy keys** instead of a human's SSH key, short-lived OIDC-issued CI tokens instead of long-lived PATs, and real secrets in a store rather than repo variables ([Vault](../Vault/README.md)).
- 📦 **Commit your lockfiles** (`uv.lock`, `poetry.lock`, `package-lock.json`) and pin dependencies — reproducible builds are a supply-chain control, not just a convenience ([SCA](../../Hardening/SCA.md)).

> [!NOTE]
> **Windows:** prefer a committed `.gitattributes` over each machine's `core.autocrlf=true` — the former is reviewable and applies in CI too. NTFS is **case-insensitive**, so `Models.py` and `models.py` collide and a rename made on Linux can arrive as a phantom conflict (`git config core.ignorecase false` makes Git stop pretending). Deep `node_modules`/venv paths fail checkout until `git config --system core.longpaths true`, which also needs the Windows `LongPathsEnabled` policy.
---
## 🐍 Django / Backend tie-in
- `.gitignore` essentials: `__pycache__/`, `*.pyc`, `.venv/`, `db.sqlite3`, `/media/`, `/staticfiles/`, `.env`, `local_settings.py`. Commit the settings *split*, keep the values in env.
- **Migrations are code** — commit them, review them, and never edit one that is already on `main`. Two branches each adding `0042_*` is the classic conflict: rebase onto `main`, then `python manage.py makemigrations --merge`.
- `git bisect run pytest` pairs perfectly with the Django test runner for regression hunts — which is another reason to keep the suite fast and deterministic ([FlakyTest](../../Python/FlakyTest.md)).
- Tag releases and let CI build the [Docker](../Docker/README.md) image from the tag, so a deployed image digest maps back to one exact commit.
---
## 🧠 Summary
| Concept | Takeaway |
|---|---|
| Object model | blob → tree → commit → ref, all content-addressed; a branch is just a movable pointer |
| Three areas | Working tree ↔ index ↔ HEAD — every command only moves data between them |
| Integrate | Rebase your *own* branch, merge or squash to share; never rebase shared history |
| Undo | `revert` is safe on pushed commits, `reset --hard` is not; `reflog` is the net |
| Strategy | GitHub Flow by default, trunk-based when your CI earns it, Git Flow only for versioned products |
| Security | Rotate before you scrub, sign commits, protect branches, block `.git/` on the web server |
