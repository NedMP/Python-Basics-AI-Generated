

# Git Automation with Python (GitPython) — updated Oct 6, 2025

Automate Git repositories using **GitPython**. Create and clone repos, stage and commit files, branch and merge, generate diffs, add tags, and interact with remotes from Python. Includes patterns for CI/CD, pre-commit hooks, and credentials. Pair with `033_KEYCHAIN_SECRETS.md` for secret storage and `026_TASK_SCHEDULING.md` to run jobs on a schedule.

---

**Assumptions**  
- macOS + zsh, Python 3.12–3.13 in a venv.  
- Git is installed and configured (`git --version`).  
- SSH keys or token-based auth available for private remotes.

**Install**
```sh
pip install GitPython python-dotenv
```

---

## Table of Contents
- [0) When to automate Git](#0-when-to-automate-git)
- [1) Setup: repo objects, safety, and paths](#1-setup-repo-objects-safety-and-paths)
- [2) Create or clone a repository](#2-create-or-clone-a-repository)
- [3) Stage, commit, and author metadata](#3-stage-commit-and-author-metadata)
- [4) Branching, merging, and rebasing](#4-branching-merging-and-rebasing)
- [5) Diffs, status, and file listings](#5-diffs-status-and-file-listings)
- [6) Tags and releases](#6-tags-and-releases)
- [7) Remotes: fetch, pull, push](#7-remotes-fetch-pull-push)
- [8) Credentials: SSH and tokens](#8-credentials-ssh-and-tokens)
- [9) CI/CD examples (GitHub Actions)](#9-cicd-examples-github-actions)
- [10) Pre-commit hooks and formatting](#10-pre-commit-hooks-and-formatting)
- [11) Larger repo considerations: submodules and LFS](#11-larger-repo-considerations-submodules-and-lfs)
- [12) Troubleshooting](#12-troubleshooting)
- [13) Recap](#13-recap)

---

## 0) When to automate Git
**What**: Use Python to script repo maintenance, batch changes across many repos, or instrument CI/CD.  
**Why**: Reduces manual steps and enforces consistent processes.
- Bulk edits across org repos.  
- Release tagging and changelog generation.  
- Auto‑committing generated assets or docs.  
- Validation bots (lint, format, policy checks).

---

## 1) Setup: repo objects, safety, and paths
**What**: Create a `Repo` handle and guard against destructive operations.  
**Why**: Many GitPython calls act on the working tree; verify the target path.
```python
from pathlib import Path
from git import Repo, InvalidGitRepositoryError

path = Path("/Users/ned/Code_Stuff/Markdown/Python_Notes")
try:
    repo = Repo(path)
except InvalidGitRepositoryError:
    raise SystemExit(f"Not a git repo: {path}")

print(repo.active_branch.name)
print(repo.git_dir)  # .git directory
```
Tips:
- Prefer absolute paths.  
- Wrap destructive operations in confirmations or dry‑run flags.

---

## 2) Create or clone a repository
**What**: Initialize a new repo or clone an existing one.
```python
from git import Repo
from pathlib import Path

# init
proj = Path("/tmp/demo-repo"); proj.mkdir(parents=True, exist_ok=True)
repo = Repo.init(proj)
(proj/"README.md").write_text("demo\n")
repo.index.add(["README.md"])  # stage
repo.index.commit("init")

# clone
repo2 = Repo.clone_from("git@github.com:org/project.git", "/tmp/project")
```
Configure identity if missing:
```python
with repo.config_writer() as cw:
    cw.set_value("user", "name", "Ned Perkins")
    cw.set_value("user", "email", "ned@example.com")
```

---

## 3) Stage, commit, and author metadata
**What**: Add files and create commits with proper authorship and messages.
```python
from git import Repo, Actor
repo = Repo("/tmp/demo-repo")

# write file
p = repo.working_tree_dir + "/notes.txt"
open(p, "w").write("hello\n")

# stage specific files or patterns
repo.index.add(["notes.txt"])          # or: repo.git.add(A=True) to add all

# commit with author/committer
author = Actor("Ned Perkins", "ned@example.com")
repo.index.commit("Add notes", author=author, committer=author)
```
Amend last commit:
```python
repo.git.commit("--amend", "--no-edit")
```

---

## 4) Branching, merging, and rebasing
**What**: Create branches, switch, merge, and rebase safely.
```python
# create and checkout branch
b = repo.create_head("feature/x")
b.checkout()

# merge feature into main
repo.git.checkout("main")
repo.git.merge("--no-ff", "feature/x")

# rebase feature on main
repo.git.checkout("feature/x")
repo.git.rebase("main")
```
Handle conflicts:
```python
if repo.index.unmerged_blobs():
    print("merge conflicts present; aborting for manual resolution")
```

---

## 5) Diffs, status, and file listings
**What**: Inspect changes programmatically.
```python
# working tree vs index
for d in repo.index.diff(None):
    print("WT→Index:", d.a_path, d.change_type)

# index vs HEAD
for d in repo.index.diff("HEAD"):
    print("Index→HEAD:", d.a_path, d.change_type)

# patch text
patch = repo.git.diff("HEAD")
print(patch[:500])

# status-like summary
print(repo.git.status("--porcelain"))
```

---

## 6) Tags and releases
**What**: Mark release points and push tags.
```python
# lightweight tag
repo.create_tag("v1.0.0")

# annotated tag
repo.create_tag("v1.0.0", message="First stable release")

# push tags
origin = repo.remote(name="origin")
origin.push(tags=True)
```

---

## 7) Remotes: fetch, pull, push
**What**: Interact with upstream repositories.
```python
origin = repo.create_remote("origin", "git@github.com:NedMP/Python-Basics-AI-Generated.git") if "origin" not in [r.name for r in repo.remotes] else repo.remote("origin")

origin.fetch(prune=True)

# pull current branch
repo.git.pull("--ff-only", "origin", repo.active_branch.name)

# push current branch
repo.git.push("origin", repo.active_branch.name)
```
Add and remove remotes:
```python
repo.create_remote("backup", "git@github.com:org/backup.git")
repo.delete_remote("backup")
```

---

## 8) Credentials: SSH and tokens
**What**: Authenticate without embedding secrets in code.  
**Why**: Safer and works with CI.
- **SSH**: Use keys in `~/.ssh` and the agent. URLs like `git@github.com:org/repo.git`.  
- **HTTPS + PAT**: Store token in macOS Keychain or env (see `033_KEYCHAIN_SECRETS.md`). Use a credential helper.  
- **In-process HTTPS**: Provide env at runtime:
```python
import os
os.environ["GIT_ASKPASS"] = "/usr/bin/true"      # disable prompts
os.environ["GIT_USERNAME"] = "x-access-token"
os.environ["GIT_PASSWORD"] = os.getenv("GITHUB_TOKEN")
```
Prefer SSH for developer machines; PATs for headless CI runners.

---

## 9) CI/CD examples (GitHub Actions)
**What**: Auto‑commit generated files and tag releases.
```yaml
# .github/workflows/auto-commit.yml
name: Auto Commit
on:
  schedule: [{cron: '0 9 * * 1-5'}]
  workflow_dispatch:
jobs:
  run:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: {python-version: '3.13'}
      - run: pip install GitPython
      - name: Update and commit
        run: |
          python - <<'PY'
          from git import Repo, Actor
          repo = Repo('.')
          # write a generated file
          open('generated.txt','w').write('updated\n')
          repo.index.add(['generated.txt'])
          author = Actor('CI Bot','ci@example.com')
          repo.index.commit('ci: update generated', author=author, committer=author)
          repo.git.push('origin','HEAD:main')
          PY
```
Release tag on push:
```yaml
# .github/workflows/release.yml
on:
  push:
    tags: ['v*']
```

---

## 10) Pre-commit hooks and formatting
**What**: Enforce formatting and linting before commits.  
**Why**: Keeps repos clean automatically.
```sh
pip install pre-commit black ruff
pre-commit install
```
`.pre-commit-config.yaml` example:
```yaml
repos:
  - repo: https://github.com/psf/black
    rev: 24.8.0
    hooks: [{id: black}]
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.6.9
    hooks: [{id: ruff}]
```
Now `git commit` runs format/lint and blocks on violations.

---

## 11) Larger repo considerations: submodules and LFS
**What**: Manage nested repos and large binaries.

### 11.1 Submodules
```python
repo.create_submodule(name="libfoo", path="vendor/libfoo", url="git@github.com:org/libfoo.git")
repo.submodule_update(init=True, recursive=True)
```
Pull with submodules:
```sh
git pull --recurse-submodules
```

### 11.2 Git LFS
Store large assets using Git LFS to avoid bloating the repo.
```sh
brew install git-lfs
git lfs install
printf "*.zip filter=lfs diff=lfs merge=lfs -text\n" >> .gitattributes
```

---

## 12) Troubleshooting
- **`fatal: not a git repository`**: path wrong; ensure `.git` exists and you run inside the repo.  
- **`non-fast-forward` on push**: pull with `--ff-only` or rebase; avoid force pushes unless necessary.  
- **Auth prompts in automation**: missing SSH agent or token; set env vars and use credential helpers.  
- **Merge conflicts**: detect with `repo.index.unmerged_blobs()` and halt scripts for manual resolution, or implement a strategy.  
- **Detached HEAD**: checkout a branch before committing: `repo.git.checkout('-B','tmp')`.  
- **Filesystem watchers**: on macOS, large batch ops may hit FSEvents limits; pause watchers during mass edits.

---

## 13) Recap
```plaintext
Open Repo → add/commit with author → branch/merge/rebase → diff and status → tag → fetch/pull/push with credentials → enforce hooks → integrate with CI/CD → handle submodules/LFS → troubleshoot
```

**Next**: Combine with `030_BACKUP_SCRIPTS.md` for repo backups, `031_NOTIFICATION_AUTOMATION.md` for alerts on failures, and `035_SCRIPT_PACKAGING.md` to ship your automation as a CLI.