# UNH Genesis: First-Time GitHub, Ubuntu Setup, Fork, Clone, Run and Contribute Guide

**Audience:** New Genesis collaborators with little or no Git/GitHub experience.
- **Platform:** Each person's **own Ubuntu or Ubuntu/WSL user account**. Use that account for every Git, Conda, test and source-edit command.
- **Canonical source:** https://github.com/utsav-sth/UNH-Genesis-Control
- **Canonical branch:** `main` (use the current approved release, not an arbitrary unvalidated feature branch).
- **Live hardware operations:** Follow [`GENESIS_PANOPTICON_OPERATOR_RUNBOOK.md`](GENESIS_PANOPTICON_OPERATOR_RUNBOOK.md) **only after explicit lab authorization and account-specific commissioning**.
**Last reviewed:** October 30, 2026.

> [!IMPORTANT]
> **Three different things:** (1) GitHub account = your login on github.com; (2) **fork** = your own repository hosted on GitHub; (3) **clone** = a working copy on your Ubuntu computer. Your Ubuntu username and GitHub username can be different. **Pushing to your fork does not change `utsav-sth/UNH-Genesis-Control`**. A change reaches the original project only if its maintainers intentionally accept it (for example, through a pull request).
>
> **A fork is not a hardware sandbox.** Two Ubuntu users running the same forked software against the same physical instruments can conflict even though their Git repositories are independent. Never run an additional physical IOC, RF output, robot, Magnet or device owner just because a clone is available. Development and testing can be done independently; physical control requires an explicit operator handoff and valid instrument permissions.

## What you will accomplish

1. Create and secure a free GitHub account.
2. Install Git and GitHub CLI (`gh`) in Ubuntu.
3. Configure your Git name/email and authenticate *your own* Ubuntu account.
4. Fork Genesis into **your GitHub account**, then clone your fork into **your Ubuntu home directory**.
5. Verify that Git's `origin` is *your fork* and `upstream` is the original project.
6. Install a separate Python/Conda environment, run safe checks and tests, and learn where the operator runbook begins.
7. Work on a personal feature branch, review, commit and push your changes to your fork only.
8. Update your fork from the canonical `main` and deal with common errors.

**Legend:** `✅ ROUTINE` = normal onboarding/Git operation; `🔎 CHECK` = read-only verification; `⚠️ LAB-ONLY` = approval and hardware-specific prerequisites; `🛠️ RECOVERY` = troubleshooting. Type each block into an Ubuntu terminal unless the instructions say **GitHub website**. Text in ALL CAPS such as `YOUR_NAME` is a placeholder, not a literal username.

---

## Part A: First-time account and Git setup (do once)

### 1. Create your GitHub account (GitHub website)

1. Visit **https://github.com/signup** and follow the instructions.
2. Select your own unique GitHub username (example: `new-collaborator`).
3. Verify your email address and configure two-factor authentication (2FA).
4. Sign in to GitHub. **Do not use another lab member's GitHub account or credentials.**
5. Keep your GitHub username handy; it may differ from your Ubuntu login name.

GitHub account instructions: https://docs.github.com/en/account-and-profile/how-tos/account-management/creating-an-account-on-github

### 2. Open Ubuntu as yourself

Use your own Ubuntu/WSL login (not `utsav`, `sliferlab`, or another person's account). If you normally SSH into a shared machine, connect with the SSH account assigned to you. The example below is **not** a connection command and does not need root access.

**🔎 CHECK:**

```bash
whoami
printf 'HOME=%s\n' "$HOME"
pwd
ls -ld "$HOME"
```

The home directory should belong to your Ubuntu user, usually `/home/<your-ubuntu-user>`. All new files below go into `$HOME`, not `/home/utsav/...`. If you don't have an Ubuntu account or `sudo` permission, ask the machine administrator; do not modify other users' homes.

### 3. Install Git and helpful Ubuntu tools

**✅ ROUTINE — the only place where `sudo` is normally used for system packages:**

```bash
sudo apt update
sudo apt install -y git gh screen curl ca-certificates
```

If `gh` is unavailable from your Ubuntu package sources, use the official install instructions at https://github.com/cli/cli/blob/trunk/docs/install_linux.md or ask the administrator. Do **not** add unreviewed package repositories just to make these commands pass.

**🔎 CHECK:**

```bash
git --version
gh --version
screen --version | head -1
```

**Never use `sudo git clone`, `sudo git commit`, `sudo git push`, or `sudo pip install` for this project.** Doing so can give files the wrong owner or share credentials across accounts.

### 4. Configure your personal Git identity

Your commit author identity should be yours. Use the email registered with GitHub or GitHub's optional `...@users.noreply.github.com` address if you prefer email privacy. The commit name is not required to match your Ubuntu username.

**✅ ROUTINE — replace the two example values:**

```bash
git config --global user.name "YOUR FULL NAME"
git config --global user.email "YOUR_GITHUB_EMAIL@example.com"
git config --global init.defaultBranch main
```

**🔎 CHECK:**

```bash
git config --global --get user.name
git config --global --get user.email
```

Do not copy a collaborator's name/email into your Git configuration.

### 5. Sign your Ubuntu account in to GitHub (recommended: GitHub CLI over HTTPS)

**✅ ROUTINE:**

```bash
gh auth login --hostname github.com --git-protocol https
```

At the prompts, choose **Login with a web browser**, and say **Yes** if asked to authenticate Git using your GitHub credentials. On a remote Ubuntu SSH terminal, use the one-time code and GitHub URL displayed in the terminal **in your local browser**; the browser does not need to run inside Ubuntu. If needed, run `gh auth setup-git` after login to enable Git credential handling.

**✅ ROUTINE / 🔎 CHECK:**

```bash
gh auth setup-git
gh auth status
gh api user --jq .login
```

The final command must print **your GitHub username**. Do not share authentication codes, passwords, credentials or tokens with other users. GitHub does not accept your normal website password for HTTPS Git pushes. With `gh`, a separate personal-access-token setup is not needed for this procedure.

**Alternative for advanced users:** SSH keys can be used instead of HTTPS; see https://docs.github.com/en/authentication/connecting-to-github-with-ssh . Keep **private** keys private; only the `.pub` file belongs in GitHub's SSH key settings. Do not mix SSH- and HTTPS-specific setup steps without deciding on one approach.

---

## Part B: Fork and clone your personal Genesis repository (do once)

### 6. Fork the original Genesis repository (GitHub website)

1. While signed in, visit **https://github.com/utsav-sth/UNH-Genesis-Control**.
2. Select **Fork** near the top-right of the repository page.
3. For **Owner**, choose **your own GitHub account**.
4. Keep the repository name **`UNH-Genesis-Control`** for these instructions.
5. Select **Copy the `main`/default branch only** for a simple new-user fork. Other branches are not needed for independent development; this does not delete them from the original project.
6. Select **Create fork**.
7. Confirm the page URL is `https://github.com/YOUR_GITHUB_USERNAME/UNH-Genesis-Control`, not `https://github.com/utsav-sth/UNH-Genesis-Control`.

GitHub's official forking walkthrough: https://docs.github.com/en/pull-requests/how-tos/work-with-forks/fork-a-repo

> The public project's fork will normally be public as well. **Do not commit secrets, private laboratory data, login details, instrument passwords or SSH keys.** If you need a private development space, ask the project owner about the approved collaboration model before copying controlled material.

### 7. Clone **your fork**, not the original, into Ubuntu

Back in **your Ubuntu terminal**, make sure `gh auth status` shows your own account. The next block discovers your GitHub username automatically so that you do not need to type a placeholder.

**✅ ROUTINE:**

```bash
cd "$HOME"
GH_USER="$(gh api user --jq .login)"
echo "Your GitHub username: $GH_USER"

# Stop if you already have a folder with this name. Do not overwrite it.
test ! -e "$HOME/UNH-Genesis-Control" || {
  echo '[STOP] ~/UNH-Genesis-Control already exists. Inspect it before proceeding.'
  exit 1
}

git clone "https://github.com/${GH_USER}/UNH-Genesis-Control.git" "$HOME/UNH-Genesis-Control"
cd "$HOME/UNH-Genesis-Control"
```

**🔎 CHECK:**

```bash
pwd
git branch --show-current
git log -1 --oneline
git status --short
git remote -v
```

Expected: `pwd` points inside **your own** `$HOME/UNH-Genesis-Control`; branch is `main`; `git status --short` prints nothing; `origin` points to **your fork**.

**Common mistake:** Cloning the original URL directly lets you read the code but usually does not give you permission to push changes to your own GitHub repository. If `origin` is not your personal fork, stop and review the remote settings before committing or pushing.

### 8. Identify the two GitHub remotes: `origin` and `upstream`

- `origin` = **your GitHub fork**. This is the only destination used for your pushes.
- `upstream` = **the original Genesis repository**. Use it only to read new commits and optionally synchronize your fork. Nothing in this guide pushes to `upstream`.

**✅ ROUTINE:**

```bash
cd "$HOME/UNH-Genesis-Control"
git remote add upstream https://github.com/utsav-sth/UNH-Genesis-Control.git

# Set conservative default push behavior for THIS checkout only.
git config --local remote.pushDefault origin
git config --local push.default simple
```

**🔎 CHECK — verify BOTH URLs:**

```bash
git remote -v
git config --local --get remote.pushDefault
```

You should see output with this structure:

```text
origin    https://github.com/YOUR_GITHUB_USERNAME/UNH-Genesis-Control.git (fetch)
origin    https://github.com/YOUR_GITHUB_USERNAME/UNH-Genesis-Control.git (push)
upstream  https://github.com/utsav-sth/UNH-Genesis-Control.git (fetch)
upstream  https://github.com/utsav-sth/UNH-Genesis-Control.git (push)
```

Git may display a push URL for `upstream` simply because the remote has a fetch URL; that is **not** evidence you have permission. **Never issue `git push upstream ...`.** If your `origin` incorrectly points to the original, stop; do not proceed to the commit/push section until corrected.

---

## Part C: Install Genesis in your OWN Ubuntu account

### 9. Check the required project and external software

A Git clone provides the **source code**, not the installed lab runtime. The current repo contains `environment.yml`, `requirements.txt`, `requirements-dev.txt`, the `scripts/` launchers and the `docs/` manuals. Additional **Phoebus**, **PyNMR**, lab network, serial/USB access and hardware credentials are not automatically supplied by Git.

**🔎 CHECK:**

```bash
cd "$HOME/UNH-Genesis-Control"
ls -l environment.yml requirements.txt requirements-dev.txt scripts/genesisctl.sh
command -v conda || echo '[INFO] Conda is not installed or not initialized'
command -v phoebus || echo '[INFO] Phoebus is not in PATH'
test -x /usr/local/bin/phoebus && echo '[INFO] System Phoebus launcher present'
```

If Conda is already installed **in your user account**, use that installation and skip Step 10. Do not assume `/home/utsav/miniconda3` or anyone else's Conda setup is usable.

### 10. Install Conda if needed (Miniforge, per-user; no sudo)

Miniforge is a Conda installer distributed by conda-forge. The example below targets **Linux x86-64** (`uname -m` should print `x86_64`), as on the existing Genesis WSL integration host. For ARM64 or other architectures, choose the matching installer from https://github.com/conda-forge/miniforge .

**🔎 CHECK:**

```bash
uname -m
```

**✅ ROUTINE — only if you do not already have Conda:**

```bash
cd "$HOME"
curl -fL -o Miniforge3-Linux-x86_64.sh \
  https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Linux-x86_64.sh

# Download the SHA256 file from the matching official release.
curl -fL -o Miniforge3-Linux-x86_64.sh.sha256 \
  https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Linux-x86_64.sh.sha256

# The checksum asset may refer to a versioned filename; compare hashes directly.
expected="$(awk '{print $1}' Miniforge3-Linux-x86_64.sh.sha256)"
actual="$(sha256sum Miniforge3-Linux-x86_64.sh | awk '{print $1}')"
if [ "$actual" = "$expected" ] && [ -n "$expected" ]; then
  echo '[PASS] Miniforge checksum matches the official release.'
  bash Miniforge3-Linux-x86_64.sh
else
  echo '[STOP] Miniforge checksum mismatch; installer NOT executed.'
fi
```

Follow its prompts, choose a **per-user location** (normally `$HOME/miniforge3`), and allow shell initialization if offered. Start a **new Ubuntu terminal**, or initialize in the current one with:

```bash
source "$HOME/miniforge3/etc/profile.d/conda.sh"
conda --version
```

If you are unsure about a download/checksum or lack machine permissions, ask the lab administrator rather than copying installation files from someone else's account.

### 11. Create the required Genesis Python environment

The repository's `environment.yml` currently specifies the `genesis-p1` environment, Python 3.13 and pip; the Python application dependencies are separately listed in `requirements.txt`.

**✅ ROUTINE:**

```bash
cd "$HOME/UNH-Genesis-Control"
conda env create -f environment.yml
conda activate genesis-p1
python -m pip install -r requirements.txt
```

If `conda env create` reports that `genesis-p1` **already exists**, do not delete it. Check the environment and ask whether this account already has a Genesis installation. If you plan to modify and test code, also install the development requirements:

```bash
python -m pip install -r requirements-dev.txt
```

**🔎 CHECK:**

```bash
which python
python --version
printf 'Conda environment: %s\n' "$CONDA_DEFAULT_ENV"
./scripts/genesisctl.sh help
./scripts/genesisctl.sh doctor
```

Expected active environment: **`genesis-p1`**. `doctor` may identify additional optional or lab-specific tools; not every workstation will be ready for physical operation after a Python package install.

### 12. Establish your independent Genesis working directory

**✅ ROUTINE — run at the start of every new Genesis terminal:**

```bash
cd "$HOME/UNH-Genesis-Control"
conda activate genesis-p1
export GENESIS_WT="$(git rev-parse --show-toplevel)"
printf 'Ubuntu user=%s\nGenesis directory=%s\nGit branch=%s\n' \
  "$(whoami)" "$GENESIS_WT" "$(git branch --show-current)"
```

A beginner **does not need Git worktrees**. Your fork's normal clone at `$HOME/UNH-Genesis-Control` is itself a valid working directory. The advanced lab runbook uses `/home/utsav/UNH-Genesis-Control/worktrees/main` as its **historical operator default**. **Do not copy or use that path in your own account.** Keep `$GENESIS_WT` set to your own checkout.

Before live operation, manually review site-specific paths in the runbook, for example `/home/utsav/jlab_pynmr`, `/home/utsav/UNH-Genesis-Control/local_data/`, `/usr/local/bin/phoebus`, Pi SSH configurations, logs, EPICS ports and fixed hardware IPs. Those paths are **examples of the commissioned lab environment** and do not automatically become valid just by cloning. Do not broadly search-and-replace names in physical configs without review.

### 13. Safely verify source and tests (no lab instrument control)

**✅ ROUTINE / 🔎 CHECK:**

```bash
cd "$HOME/UNH-Genesis-Control"
conda activate genesis-p1
./scripts/genesisctl.sh where
./scripts/genesisctl.sh status
python -m pytest -q
```

`status` is a read-only process summary. The test suite is the project's software regression check, **not proof that physical hardware is safe or that every device has been accepted**. If a test fails, record the failure and compare it with the current release notes rather than disabling interlocks or changing tests just to make them pass. Tests may require additional optional dependencies or may include known baseline exclusions; consult the repository's current test documentation.

**⚠️ LAB-ONLY — before starting any live system:**

- Obtain permission from the responsible experiment/DAQ operator and confirm **this specific Ubuntu user and working copy** are authorized to own hardware.
- Confirm only **one active owner per physical device** across all Ubuntu accounts, worktrees and Raspberry Pis. Same hardware addresses, EPICS ports, locks and power supplies are shared, even when source trees are different.
- Verify that the approved Genesis/PyNMR environments, Phoebus, X11/GUI forwarding if remote, EPICS CA/PVA network, drivers, USB/serial access, instruments and required SSH access are installed and tested **for your account**. PyNMR is separate from this Genesis fork and requires its own approved setup.
- Use the full current operator startup/status/shutdown and physical safety checklists in [`docs/operations/GENESIS_PANOPTICON_OPERATOR_RUNBOOK.md`](GENESIS_PANOPTICON_OPERATOR_RUNBOOK.md). Its source-specific absolute paths and historical assumptions require account-specific adaptation.
- **Do not** run `./scripts/genesisctl.sh mw-start`, `cryo-start`, `ssrf-start`, or an instrument IOC in a shared lab just to see whether it works. Physical operation is not a Git onboarding step.

Once the above **lab-only** requirements are satisfied, the routine command entry point (run from `$GENESIS_WT` with `genesis-p1` active) is:

```bash
./scripts/genesisctl.sh help
./scripts/genesisctl.sh status
./scripts/genesisctl.sh cryo-status
./scripts/genesisctl.sh mw-status
./scripts/genesisctl.sh ssrf-physical-status
./scripts/genesisctl.sh master-archive-status
```

These are **status** examples, not permission to take hardware ownership. For actual start/stop actions, use the documented 🟩 **OPERATOR** commands **in the required sequence** and the 🟧 **SPECIALIST** Magnet/physical-control restrictions. Trend History is an ordinary operator-managed start/stop subsystem, per the current Panopticon runbook.

---

## Part D: Edit, test, commit and push changes to **your fork only**

### 14. Understand `main` versus your feature branch

- `main` is the branch you copied from Genesis. Keep it synchronized with the approved project baseline; do not use it for untested live hardware changes.
- `feature/...` is a separate branch in **your fork** where you can edit, test and save a change without rewriting your fork's `main`.
- `git commit` saves a version **locally in Ubuntu**. `git push` uploads that commit **to your GitHub fork**. Neither command changes the original Genesis repository when `origin` points to your fork.

**✅ ROUTINE — create a new branch BEFORE editing:**

```bash
cd "$HOME/UNH-Genesis-Control"
git switch main
git status --short
# Continue only when the working tree is clean and no hardware run is active.
git switch -c feature/my-first-change
```

Replace `my-first-change` with a short meaningful name, such as `feature/flowmeter-docs` or `fix/microwave-ui-label`. A branch name should not contain spaces.

**🔎 CHECK:**

```bash
git branch --show-current
git remote -v
```

You should now be on `feature/my-first-change` and have **your fork** as `origin`.

### 15. Make a small change and inspect it

Open a file in your preferred editor (`nano`, VS Code Remote SSH, etc.). Start with documentation if you are learning Git. For example:

```bash
nano README.md
```

With `nano`, edit, then press **Ctrl+O**, **Enter** to save; **Ctrl+X** to exit. Do not edit tracked device-control or physical safety settings during an active run.

**🔎 CHECK:**

```bash
git status --short
git diff -- README.md
git diff --check
```

`M README.md` means the file has changed locally. `git diff` shows edits that are **not yet staged**. `git diff --check` catches trailing-whitespace problems. If you changed Python code, add/update relevant tests and run them **offline** before proposing any hardware validation:

```bash
conda activate genesis-p1
python -m pytest -q
```

### 16. Stage and commit only the intended files

**✅ ROUTINE:**

```bash
cd "$HOME/UNH-Genesis-Control"
# Example for a README change. Replace the filename for your actual change.
git add README.md

git diff --cached --check
git diff --cached --stat
git diff --cached -- README.md

git commit -m "Document beginner setup clarification"
```

**🔎 CHECK:**

```bash
git status --short
git log -2 --oneline
```

Stage named files instead of using `git add .` blindly. Never stage `local_data/`, archive CSV/JSONL files, passwords, `.env` files, SSH credentials, tokens, personal files or machine-specific runtime logs. Review every staged path with `git diff --cached --name-only` before committing.

If Git reports **nothing to commit**, inspect `git status` and the file you edited. Do not invent a dummy commit.

### 17. Push your feature branch to your own fork

**🔎 CHECK — do this before every new branch's first push:**

```bash
cd "$HOME/UNH-Genesis-Control"
git branch --show-current
git remote get-url origin
```

The remote URL **must** identify `YOUR_GITHUB_USERNAME/UNH-Genesis-Control`, not `utsav-sth/UNH-Genesis-Control`.

**✅ ROUTINE — explicit safe destination:**

```bash
git push -u origin feature/my-first-change
```

Go to `https://github.com/YOUR_GITHUB_USERNAME/UNH-Genesis-Control` in your browser and change the branch selector to `feature/my-first-change`. You should see your commit there.

**To continue editing the same branch later:**

```bash
cd "$HOME/UNH-Genesis-Control"
git switch feature/my-first-change
# Edit and test the necessary files, then stage named paths.
git add README.md
git commit -m "Clarify setup guidance"
git push origin feature/my-first-change
```

**No change to the original project occurs.** Do not open a pull request against `utsav-sth:main` unless you intentionally want to propose your change for review. Opening a pull request does not automatically merge a change, but do not submit private or unsafe work for merging.

### 18. Update your fork when the original project changes

**Easiest beginner option (GitHub website):** open **your fork's** `main` page, click **Sync fork**, review the new commits, then **Update branch**. This updates **your fork**, not the original. Do this only when no active hardware run is using that checkout.

After a successful website sync, update **your local `main`**:

```bash
cd "$HOME/UNH-Genesis-Control"
git status --short
# Continue only if clean and no hardware run is active.
git switch main
git pull --ff-only origin main
```

**Command-line alternative** for a clean, unmodified local `main`:

```bash
cd "$HOME/UNH-Genesis-Control"
git switch main
git status --short
git fetch upstream
git merge --ff-only upstream/main
# Only after the above succeeds and origin is verified as YOUR fork:
git push origin main
```

`--ff-only` refuses to overwrite divergent history. If it fails, **stop and ask for help**; do not run `git reset --hard`, `git push --force`, or a blind merge. If you have a feature branch that needs the latest `main`, update it later when no physical run is active:

```bash
git switch feature/my-first-change
git merge main
```

If Git reports a merge conflict, stop and review the affected files with your project mentor. Do not resolve conflicts in hardware-control, interlock or IOC configuration files without domain review.

### 19. Optional: Propose a change to the original project

If you want the Genesis maintainer to consider your tested feature, open **your fork** on GitHub and select **Contribute → Open pull request**, with **base repository** `utsav-sth/UNH-Genesis-Control` and **base branch** `main`. Describe exactly what changed, offline tests run, hardware testing **not** done, and any safety considerations. **Do not merge to the original yourself.** The original maintainers review and decide whether to merge it.

A pull request is optional. **You can work indefinitely on your own fork without proposing any changes upstream.**

---

## Part E: Everyday command reference and common problems

### 20. Git basics cheat sheet

| Command | What it does | Safe for new users? |
|---|---|---|
| `git status` | Show changed/staged files and current branch | Yes; read-only |
| `git branch --show-current` | Show branch name | Yes; read-only |
| `git remote -v` | Show fork/original remote URLs | Yes; read-only |
| `git log -5 --oneline` | Show recent commits | Yes; read-only |
| `git diff` | Inspect unstaged edits | Yes; read-only |
| `git diff --cached` | Inspect staged edits | Yes; read-only |
| `git switch -c feature/name` | Create a personal branch | Yes, with a clean checkout |
| `git add FILE` | Stage one chosen file | Yes; inspect before commit |
| `git commit -m "..."` | Save staged changes locally | Yes, on your feature branch |
| `git push -u origin feature/name` | Publish branch to **your fork** | Yes, after remote verification |
| `git pull --ff-only origin main` | Update local main from your fork | Yes, if clean and no hardware active |
| `git fetch upstream` | Download original project history without editing files | Yes; read-only to remote |
| `git push upstream ...` | Attempts to push to original project | **NO** |
| `git reset --hard`, `git clean -fd`, `git push --force` | Can destroy local work or rewrite history | **NO** without explicit expert review |

### 21. Troubleshooting

**"git: command not found"**: Repeat Step 3 or ask for system package access. Do not reinstall Git into another person's home.

**"gh: command not found" / authentication failed**: Install GitHub CLI, rerun `gh auth login` as **your Ubuntu user** and check `gh auth status`. Do not paste access tokens into chat, documents, shell history or Git URLs.

**"Repository not found" when cloning**: Check that you actually created a fork, that the fork URL is correct, and `gh api user --jq .login` returns the intended GitHub account. For private/organization repositories, access restrictions may also apply.

**Git asks for GitHub password on `git push`**: Do not type your normal GitHub website password. Run `gh auth login --hostname github.com --git-protocol https` and `gh auth setup-git`; confirm your fork uses an HTTPS remote.

**"Permission denied" / "403" when pushing**: First inspect `git remote -v`. If `origin` points to `utsav-sth`, you cloned the wrong repository or set the wrong remote. Do not request write access to the original just to finish onboarding. Confirm your fork and remote with the project mentor before fixing it.

**"Identity unknown" when committing**: Repeat Step 4 with your own name and email.

**"nothing to commit"**: Check whether you changed/saved a file and staged it; inspect `git status`.

**"branch already exists"**: `git switch feature/my-first-change` to reopen it, or choose a new branch name; don't delete existing work.

**"Your local changes would be overwritten" / merge conflicts**: STOP. Check `git status` and save/review your own edits before attempting to sync. Never blindly use `reset --hard` or `clean` on a lab checkout.

**`conda: command not found`**: Reopen the terminal after installing Miniforge or initialize Conda from your own installation. Don't source another user's Conda initialization.

**EPICS port already used, ssRF lock busy, access denied or device disconnected**: This is a **hardware/software ownership** issue, not a Git issue. Do **not** kill somebody else's IOC or remove locks. Stop and contact the current Genesis operator. Another user's fork is not another physical device.

**Phoebus/PyNMR cannot start**: Git and pip do not install those external programs automatically. Follow their separate approved installation and commissioning procedures. Do not use the repo's fixed lab IP settings outside the authorized lab environment.

### 22. A ten-second pre-push check (repeat every time)

**🔎 CHECK:**

```bash
cd "$HOME/UNH-Genesis-Control"
printf '\nUSER: '; whoami
printf 'BRANCH: '; git branch --show-current
printf 'PUSH DESTINATION: '; git remote get-url origin
printf '\nCHANGED FILES:\n'; git status --short
printf '\nSTAGED FILES:\n'; git diff --cached --name-only
printf '\nWHITESPACE CHECK:\n'; git diff --cached --check
```

Proceed with `git push origin YOUR_FEATURE_BRANCH` **only if** the destination is your fork, you are on the intended feature branch, and no secrets or unintended files were staged.

---

## Where to read next

- **Live operator procedures, clearly classified 🟩 OPERATOR / 🟦 EXPERT / 🟧 SPECIALIST:** [`docs/operations/GENESIS_PANOPTICON_OPERATOR_RUNBOOK.md`](GENESIS_PANOPTICON_OPERATOR_RUNBOOK.md)
- **Project architecture and hardware safety:** [`README.md`](../../README.md)
- **Subsystem documentation:** `docs/MICROWAVE_PHYSICAL_OPERATION.md`, `docs/SSRF_FINAL_PHYSICAL_OPERATION.md` and the current Cryo/Magnet documents.
- **GitHub beginner guide:** https://docs.github.com/en/get-started/learning-to-code/getting-started-with-git
- **Forking:** https://docs.github.com/en/pull-requests/how-tos/work-with-forks/fork-a-repo
- **Synchronizing a fork:** https://docs.github.com/en/pull-requests/how-tos/work-with-forks/syncing-a-fork
- **GitHub CLI login:** https://docs.github.com/en/github-cli/github-cli/quickstart
- **Miniforge installer and checksums:** https://github.com/conda-forge/miniforge

**Principle:** Clone and modify your own fork; run tests in your own Ubuntu account; push to `origin` only; use physical instruments **only** after authorization and verified exclusive ownership. Git independence does not imply independent physical equipment.
