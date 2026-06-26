# Git and GitHub Hands-On Session Guide

This guide is designed for trainees to learn Git and GitHub through hands-on exercises. We'll cover essential Git commands, workflows, and GitHub features. Make sure you have Git installed and a GitHub account set up before starting.

## Prerequisites

- Install Git: [Download Git](https://git-scm.com/downloads)
- Create a GitHub account: [Sign up on GitHub](https://github.com)
- Basic command line knowledge

## Table of Contents

1. [Introduction to Git](#introduction-to-git)
2. [Setting Up Git](#setting-up-git)
3. [Creating a Repository](#creating-a-repository)
4. [Basic Git Workflow](#basic-git-workflow)
5. [Git Flow](#git-flow)
6. [Branching](#branching)
7. [Merging](#merging)
8. [Rebasing](#rebasing)
9. [Cherry-Picking](#cherry-picking)
10. [Reverting Commits](#reverting-commits)
11. [Stashing](#stashing)
12. [Working with GitHub](#working-with-github)
13. [Pull Requests](#pull-requests)
14. [Additional Topics](#additional-topics)

## Introduction to Git

Git is a distributed version control system that tracks changes in source code during software development. It allows multiple developers to work on the same project simultaneously without conflicts.

Key concepts:
- Repository (repo): A directory that contains your project files and the entire history of changes
- Commit: A snapshot of your changes
- Branch: A separate line of development
- Remote: A version of your repository hosted on the internet (like GitHub)

## Setting Up Git

Configure your Git identity:

```bash
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```

## Creating a Repository

### Local Repository

```bash
# Initialize a new Git repository
git init

# Add files to staging area
git add .

# Make your first commit
git commit -m "Initial commit"
```

### Remote Repository (GitHub)

1. Create a new repository on GitHub
2. Connect your local repo to the remote:

```bash
git remote add origin https://github.com/yourusername/yourrepo.git
git push -u origin main
```

## Basic Git Workflow

1. Make changes to files
2. Stage changes: `git add <filename>` or `git add .`
3. Commit changes: `git commit -m "Commit message"`
4. Push to remote: `git push`

## Git Flow

Git Flow is a branching model for Git. It defines a strict branching model designed around the project release.

### Initialize Git Flow

```bash
# Install git-flow (if not already installed)
# On Windows: Download from https://github.com/nvie/gitflow/wiki/Windows

# Initialize git-flow in your repository
git flow init
```

This will create branches like:
- `main` (or `master`): Production code
- `develop`: Integration branch for features
- `feature/`: Feature branches
- `release/`: Release branches
- `hotfix/`: Hotfix branches

## Branching

Branches allow you to work on different features or fixes without affecting the main codebase.

### Creating and Switching Branches

```bash
# Create and switch to a new branch
git checkout -b feature/new-feature

# Switch to an existing branch
git checkout main

# List all branches
git branch

# Delete a branch
git branch -d feature/new-feature
```

### Branching Strategies

- **Feature Branching**: Create a branch for each new feature
- **Git Flow**: Structured branching model (covered above)
- **GitHub Flow**: Simple branching model using main branch and feature branches

## Merging

Merging combines changes from different branches.

### Merge Commands

```bash
# Merge a branch into current branch
git merge feature/new-feature

# Fast-forward merge (when possible)
git merge --ff-only feature/new-feature

# No-fast-forward merge (always creates merge commit)
git merge --no-ff feature/new-feature
```

### Merge Conflicts

When Git can't automatically merge changes, you'll get a merge conflict. Resolve it by:

1. Edit the conflicting files
2. Remove conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`)
3. Stage the resolved files: `git add <filename>`
4. Complete the merge: `git commit`

## Rebasing

Rebasing rewrites commit history by moving or combining commits.

### Basic Rebase

```bash
# Rebase current branch onto another branch
git rebase main

# Interactive rebase (to edit, squash, or reorder commits)
git rebase -i HEAD~3  # Last 3 commits
```

### Rebase vs Merge

- **Merge**: Preserves history, creates merge commit
- **Rebase**: Cleaner history, rewrites commits

## Cherry-Picking

Cherry-picking applies specific commits from one branch to another.

```bash
# Cherry-pick a specific commit
git cherry-pick <commit-hash>

# Cherry-pick multiple commits
git cherry-pick <commit-hash1> <commit-hash2>
```

## Reverting Commits

Revert undoes changes by creating a new commit that reverses the changes.

```bash
# Revert a specific commit
git revert <commit-hash>

# Revert multiple commits
git revert <commit-hash1>..<commit-hash2>
```

Note: `git revert` creates a new commit, while `git reset` removes commits from history.

## Stashing

Stashing temporarily saves changes that aren't ready to be committed.

```bash
# Stash changes
git stash

# List stashes
git stash list

# Apply latest stash
git stash apply

# Apply specific stash
git stash apply stash@{1}

# Create a stash with a message
git stash push -m "Work in progress"

# Drop a stash
git stash drop stash@{0}
```

## Working with GitHub

### Cloning a Repository

```bash
git clone https://github.com/username/repo.git
```

### Forking

Forking creates a copy of someone else's repository in your GitHub account.

### Syncing with Upstream

```bash
# Add upstream remote
git remote add upstream https://github.com/original-owner/repo.git

# Fetch upstream changes
git fetch upstream

# Merge upstream changes
git merge upstream/main
```

## Pull Requests

Pull Requests (PRs) are proposals to merge changes from one branch into another.

### Creating a Pull Request

1. Push your feature branch to GitHub
2. Go to your repository on GitHub
3. Click "Compare & pull request"
4. Add title and description
5. Click "Create pull request"

### Reviewing Pull Requests

- Review code changes
- Add comments
- Request changes or approve
- Merge the PR

### PR Best Practices

- Write clear descriptions
- Reference issues
- Keep PRs small and focused
- Use labels and assignees

## Additional Topics

### Git Ignore

Create a `.gitignore` file to exclude files from version control:

```
# Compiled files
*.class
*.exe

# IDE files
.vscode/
.idea/

# OS files
.DS_Store
Thumbs.db
```

### Git Log and History

```bash
# View commit history
git log

# View compact log
git log --oneline

# View graph of branches
git log --graph --oneline --all
```

### Git Diff

```bash
# See changes in working directory
git diff

# See changes in staging area
git diff --staged

# Compare branches
git diff main..feature-branch
```

### Git Reset

```bash
# Soft reset (keeps changes in working directory)
git reset --soft HEAD~1

# Mixed reset (default, unstages changes)
git reset HEAD~1

# Hard reset (removes changes completely)
git reset --hard HEAD~1
```

### Git Reflog

Recover lost commits:

```bash
git reflog
git reset --hard <commit-hash>
```

### Collaboration Best Practices

- Commit often with clear messages
- Use branches for features and fixes
- Pull regularly to stay up-to-date
- Use PRs for code review
- Write meaningful commit messages

### Common Git Commands Cheat Sheet

- `git status`: Check repository status
- `git add .`: Stage all changes
- `git commit -m "message"`: Commit staged changes# Git and GitHub Hands-On Session Guide

This guide is designed to learn Git and GitHub through hands-on exercises. We'll cover essential Git commands, workflows, and GitHub features. Make sure you have Git installed and a GitHub account set up before starting.

## Hands-On Exercises

1. **Exercise 1**: Initialize a repository and make your first commit
2. **Exercise 2**: Create branches and switch between them
3. **Exercise 3**: Make changes, commit, and merge branches
4. **Exercise 4**: Practice rebasing
5. **Exercise 5**: Cherry-pick commits
6. **Exercise 6**: Revert a commit
7. **Exercise 7**: Use stash to save work in progress
8. **Exercise 8**: Create a pull request on GitHub

## Resources

- [Git Documentation](https://git-scm.com/doc)
- [GitHub Guides](https://guides.github.com/)
- [Learn Git Branching](https://learngitbranching.js.org/)
- [Pro Git Book](https://git-scm.com/book/en/v2)

Happy coding! 🚀