# Git Reference — Commands I've Used

A running log of Git commands used throughout the bootcamp, with notes on what each one does.

---

## Setting Up a Repo

```bash
git init
```
Initialises a brand-new Git repository in the current folder. Creates the hidden `.git` folder that tracks everything.

```bash
git remote add origin https://github.com/YOUR-USERNAME/YOUR-REPO.git
```
Links your local repo to a remote repo on GitHub. `origin` is just the conventional name for your main remote.

```bash
git branch -M main
```
Renames the default branch to `main` (GitHub's current standard).

```bash
git push -u origin main
```
Pushes your local `main` branch to GitHub for the first time. The `-u` flag sets up tracking so future pushes just need `git push`.

---

## Saving & Pushing Changes

```bash
git status
```
Shows what files have changed, what's staged, and what's untracked. Run this before anything else to see where you are.

```bash
git add .
```
Stages all changed and new files in the current folder ready for a commit.

```bash
git add filename.md
```
Stages a specific file only — more precise than `git add .`

```bash
git commit -m "Your message here"
```
Saves a snapshot of everything staged. The message should describe what changed and why.

```bash
git push
```
Sends your committed changes up to GitHub.

---

## Checking History

```bash
git log
```
Shows the full commit history — who committed, when, and the message.

```bash
git log --oneline
```
Same as above but condensed to one line per commit. Easier to scan.

---

## Everyday Workflow

The typical sequence when you've made changes and want to push them:

```bash
git status          # see what changed
git add .           # stage everything (or add specific files)
git commit -m "msg" # save the snapshot
git push            # send to GitHub
```

---

*Updated as new commands are used throughout the bootcamp.*
