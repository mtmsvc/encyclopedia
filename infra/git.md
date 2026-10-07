# git
A version control system: it records the history of a project as a series of snapshots, so I can go back to any point, work on several things in parallel, and share work with others.
GitHub is a hosting service for git repositories (plus pull requests, CI, etc.). git works without GitHub; GitHub is just a remote.

## How it works
- **Commit** - a snapshot of the whole project at one moment, plus a message, author, and a pointer to the previous commit. Commits form a chain (a graph, when branches split and merge). Each commit has a unique ID (hash), e.g. `af683ab`.
- **Three areas** where a change can be:
  1. **Working directory** - the files as I see them on disk.
  2. **Staging area (index)** - the changes I selected for the next commit (`git add`).
  3. **Repository** - the committed history (`git commit`), stored in the `.git` folder.
- **Branch** - just a movable pointer to a commit. Creating a branch is cheap: it creates a pointer, not a copy of the files. A new commit moves the current branch's pointer forward.
- **HEAD** - a pointer to where I am right now (usually the current branch).
- **Remote** - a copy of the repository somewhere else (GitHub). `origin` is the default name for it. Local and remote are synced only explicitly: `push` sends my commits, `pull` fetches and merges theirs.

## Setup on a new machine
```bash
git config --global user.name "My Name"
git config --global user.email "me@example.com"
git config --global init.defaultBranch main
```
SSH key, so GitHub accepts pushes without a password:
```bash
ssh-keygen -t ed25519 -C "me@example.com"   # Enter for default path
cat ~/.ssh/id_ed25519.pub                    # copy -> GitHub: Settings -> SSH and GPG keys -> New SSH key
ssh -T git@github.com                        # test: "Hi <username>! You've successfully authenticated"
```
The private key (`id_ed25519`, no `.pub`) never leaves the machine.

## Starting a repository
**New project, then GitHub:**
1. On GitHub: New repository, **empty** (no README, .gitignore, license - they would conflict with local files).
2. Locally:
```bash
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin git@github.com:<username>/<repo>.git
git push -u origin main      # -u: remember origin/main, so later just `git push`
```

**Existing repository from GitHub:**
```bash
git clone git@github.com:<username>/<repo>.git
```

## Everyday commands
```bash
git status                  # what changed, what is staged
git diff                    # unstaged changes; --staged for staged ones
git add <file>              # stage a file; `git add .` stages everything
git commit -m "message"     # commit the staged changes
git push                    # send commits to the remote
git pull                    # get commits from the remote and merge them
git log --oneline -10       # last 10 commits, short
```

## Branches and pull requests
Work on a branch, merge into `main` through a pull request (PR), so CI checks the change before it lands. `main` always stays working.
```bash
git switch -c <branch>      # create a branch and switch to it
git push -u origin <branch> # first push of the branch
# -> on GitHub: open a pull request, wait for CI, merge
git switch main             # back to main
git pull                    # get the merged result
git branch -d <branch>      # delete the local branch
git push origin --delete <branch> # delete the remote branch 
```
`git switch <branch>` switches to an existing branch. `git branch` lists branches.

## .gitignore
A file listing what git must never track: `.venv`, caches, build output, `.env` with secrets.
It only affects untracked files. If something was already committed before being ignored, untrack it (files stay on disk):
```bash
git rm -r --cached .
git add .
git commit -m "Untrack ignored files"
```
A secret that was ever pushed to a public repo is leaked, even after deleting it - rotate it.

## Moving and renaming
`git mv <old> <new>` - move or rename a tracked file, so history follows it. For untracked files, plain `mv` + `git add`.

## Undoing things
```bash
git restore <file>                # discard unstaged changes in a file
git restore --staged <file>       # unstage, keep the changes
git commit --amend -m "new msg"   # fix the last commit (only if not pushed yet)
git revert <commit>               # new commit that undoes <commit> - safe on pushed history
git reset --soft HEAD~1           # undo the last commit, keep its changes staged (only if not pushed)
git stash                         # put uncommitted changes aside; `git stash pop` brings them back
```
Rule: never rewrite history that is already pushed (`--amend`, `reset`); use `revert` instead.
