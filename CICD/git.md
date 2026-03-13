
# Git Command Line Guide

## Table of Contents
- [Introduction](#introduction)
- [Basic Configuration](#basic-configuration)
- [Repository Basics](#repository-basics)
- [Branching](#branching)
- [Merging](#merging)
- [Rollbacks](#rollbacks)
- [Cherry Picking](#cherry-picking)

## Introduction

Git is a distributed version control system that tracks changes in source code during software development. This guide covers essential Git operations from the command line.

## Basic Configuration

**Summary:** Set up your Git identity and preferences before making commits.

```bash
# Set your name and email
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# View configuration
git config --list

# Set default branch name
git config --global init.defaultBranch main
```

## Repository Basics

**Summary:** Initialize repositories, stage changes, and commit your work.

```bash
# Initialize a new repository
git init

# Clone an existing repository
git clone <repository-url>

# Check repository status
git status

# Add files to staging area
git add <file>
git add .  # Add all changes

# Commit changes
git commit -m "Commit message"

# View commit history
git log
git log --oneline --graph --all
```

## Branching

**Summary:** Branches allow parallel development by creating isolated environments for features or fixes.

### Technical Details
- Branches are lightweight pointers to commits
- HEAD points to the current branch
- Creating a branch doesn't create a new copy of files, just a new pointer

```bash
# List branches
git branch
git branch -a  # Include remote branches

# Create a new branch
git branch <branch-name>

# Switch to a branch
git checkout <branch-name>
git switch <branch-name>  # Modern alternative

# Create and switch to a new branch
git checkout -b <branch-name>
git switch -c <branch-name>

# Delete a branch
git branch -d <branch-name>  # Safe delete
git branch -D <branch-name>  # Force delete

# Rename current branch
git branch -m <new-name>

# Push branch to remote
git push -u origin <branch-name>
```

## Merging

**Summary:** Combine changes from different branches into one branch.

### Technical Details
- **Fast-forward merge:** When target branch hasn't diverged, Git simply moves pointer forward
- **Three-way merge:** When branches have diverged, Git creates a new merge commit
- **Merge conflicts:** Occur when same lines are modified in both branches

```bash
# Merge a branch into current branch
git merge <branch-name>

# Merge with no fast-forward (creates merge commit)
git merge --no-ff <branch-name>

# Abort a merge in case of conflicts
git merge --abort

# View merge conflicts
git status
git diff

# After resolving conflicts manually
git add <resolved-files>
git commit
```

## Rollbacks

**Summary:** Undo changes, revert commits, or reset the repository to a previous state.

### Technical Details
- **Reset:** Moves HEAD and branch pointer, modifies staging/working directory
- **Revert:** Creates new commit that undoes changes (safe for shared history)
- **Checkout:** Updates working directory without moving branch pointer

```bash
# Undo changes in working directory
git checkout -- <file>
git restore <file>  # Modern alternative

# Unstage files
git reset HEAD <file>
git restore --staged <file>

# Reset types
git reset --soft <commit>   # Keep changes staged
git reset --mixed <commit>  # Keep changes unstaged (default)
git reset --hard <commit>   # Discard all changes

# Revert a commit (creates new commit)
git revert <commit-hash>

# Revert multiple commits
git revert <oldest-commit>..<newest-commit>

# View reflog (history of HEAD movements)
git reflog

# Recover from reset using reflog
git reset --hard <reflog-hash>
```

## Cherry Picking

**Summary:** Apply specific commits from one branch to another without merging entire branch.

### Technical Details
- Creates a new commit with same changes but different hash
- Useful for hotfixes or selective feature porting
- Can cause duplicate commits if not careful with subsequent merges

```bash
# Cherry pick a single commit
git cherry-pick <commit-hash>

# Cherry pick multiple commits
git cherry-pick <commit1> <commit2>

# Cherry pick a range of commits
git cherry-pick <start-commit>..<end-commit>

# Cherry pick without committing
git cherry-pick -n <commit-hash>
git cherry-pick --no-commit <commit-hash>

# Continue after resolving conflicts
git cherry-pick --continue

# Abort cherry pick
git cherry-pick --abort

# Skip current commit during cherry pick
git cherry-pick --skip
```

## Additional Commands

```bash
# Stash changes temporarily
git stash
git stash pop
git stash list

# View differences
git diff
git diff --staged

# Remote operations
git remote -v
git fetch
git pull
git push

# Tag commits
git tag <tag-name>
git tag -a <tag-name> -m "message"
```
