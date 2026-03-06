# Git - Version Control System

## 1. Introduction

Git is a distributed version control system designed to handle everything from small to very large projects with speed and efficiency. Created by Linus Torvalds in 2005 for the development of the Linux kernel, Git has become the most widely adopted version control system in the software development industry.

### Key Features:
- **Distributed**: Every developer has a complete copy of the repository
- **Fast**: Most operations are performed locally
- **Secure**: Uses SHA-1 hashing for data integrity
- **Branching**: Lightweight branching and merging capabilities
- **Staging Area**: Intermediate area for preparing commits

## 2. Installation Steps

### macOS
```bash
# Using Homebrew
brew install git

# Using MacPorts
sudo port install git

# Download from official website
# https://git-scm.com/download/mac
```

### Linux (Ubuntu/Debian)
```bash
sudo apt update
sudo apt install git
```

### Linux (CentOS/RHEL/Fedora)
```bash
# CentOS/RHEL
sudo yum install git

# Fedora
sudo dnf install git
```

### Windows
1. Download Git for Windows from https://git-scm.com/download/win
2. Run the installer and follow the setup wizard
3. Git Bash will be available for command-line operations

### Verify Installation
```bash
git --version
```

### Initial Configuration
```bash
# Set your username and email
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# Set default editor (optional)
git config --global core.editor "code --wait"  # VS Code
git config --global core.editor "vim"          # Vim
```

## 3. What Git is Used For?

Git serves as the foundation for modern software development workflows.

### Primary Use Cases:
- **Version Control**: Track changes to source code over time
- **Collaboration**: Enable multiple developers to work on the same project
- **Backup**: Provide a complete history of all changes
- **Branching**: Create separate lines of development for features, fixes, and releases
- **Code Review**: Facilitate peer review processes through pull requests
- **Deployment**: Integrate with CI/CD pipelines for automated deployments

### Core Concepts:
- **Repository**: A directory containing project files and Git metadata
- **Commit**: A snapshot of the repository at a specific point in time
- **Branch**: A separate line of development
- **Merge**: Combining changes from different branches
- **Remote**: A copy of the repository hosted on a server (GitHub, GitLab, etc.)

## 4. Deep Dive

### 4.1 Repository Management

#### Creating a New Repository
```bash
# Initialize a new Git repository
git init

# Initialize with a specific branch name
git init -b main
```

#### Cloning Repositories
```bash
# Clone from HTTPS
git clone https://github.com/username/repository.git

# Clone from SSH
git clone git@github.com:username/repository.git

# Clone specific branch
git clone -b branch-name https://github.com/username/repository.git

# Clone with limited history (shallow clone)
git clone --depth 1 https://github.com/username/repository.git
```

#### Repository Configuration
```bash
# View all configuration
git config --list

# View specific configuration
git config user.name

# Set repository-specific configuration
git config user.name "Your Name"
git config user.email "your.email@example.com"

# Set global configuration
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```

### 4.2 Basic Operations

#### Staging Files
```bash
# Add specific file
git add filename.txt

# Add all files in current directory
git add .

# Add all files with specific extension
git add *.js

# Add files interactively
git add -i

# Add only modified files (not new files)
git add -u
```

#### Committing Changes
```bash
# Commit with message
git commit -m "Add new feature"

# Commit all staged changes with message
git commit -am "Update documentation"

# Amend last commit
git commit --amend

# Create empty commit
git commit --allow-empty -m "Trigger deployment"
```

#### Viewing History
```bash
# View commit history
git log

# View compact history
git log --oneline

# View history with graph
git log --graph --oneline --all

# View specific file history
git log -- filename.txt

# View changes in last commit
git show
```

#### Repository Status
```bash
# Check repository status
git status

# Check status in short format
git status -s

# Check ignored files
git status --ignored
```

### 4.3 Branching and Merging

#### Creating and Managing Branches
```bash
# Create new branch
git branch feature-branch

# Create and switch to new branch
git checkout -b feature-branch

# Switch to existing branch
git checkout branch-name

# List all branches
git branch

# List all branches (local and remote)
git branch -a

# Delete branch
git branch -d branch-name

# Force delete branch
git branch -D branch-name
```

#### Merging Branches
```bash
# Merge branch into current branch
git merge feature-branch

# Merge with no fast-forward
git merge --no-ff feature-branch

# Abort merge
git merge --abort

# Continue merge after resolving conflicts
git merge --continue
```

#### Resolving Merge Conflicts
```bash
# View conflicted files
git status

# Edit conflicted files manually
# Look for <<<<<<< HEAD, =======, and >>>>>>> markers

# After resolving conflicts
git add resolved-file.txt
git commit
```

### 4.4 Remote Operations

#### Working with Remotes
```bash
# Add remote repository
git remote add origin https://github.com/username/repository.git

# List remotes
git remote -v

# Remove remote
git remote remove origin

# Rename remote
git remote rename old-name new-name
```

#### Pulling Changes
```bash
# Pull changes from remote
git pull origin main

# Fetch changes without merging
git fetch origin

# Pull with rebase
git pull --rebase origin main
```

#### Pushing Changes
```bash
# Push to remote
git push origin main

# Push new branch
git push -u origin feature-branch

# Force push (use with caution)
git push --force

# Push tags
git push --tags
```



### 4.5 Advanced Features

#### Stashing
```bash
# Stash current changes
git stash

# Stash with message
git stash push -m "Work in progress"

# List stashes
git stash list

# Apply most recent stash
git stash pop

# Apply specific stash
git stash apply stash@{1}

# Drop stash
git stash drop stash@{0}

# Clear all stashes
git stash clear
```

#### Rebasing
```bash
# Rebase current branch onto main
git rebase main

# Interactive rebase
git rebase -i HEAD~3

# Abort rebase
git rebase --abort

# Continue rebase
git rebase --continue
```

#### Cherry-picking
```bash
# Cherry-pick specific commit
git cherry-pick commit-hash

# Cherry-pick without auto-commit
git cherry-pick -n commit-hash

# Cherry-pick range of commits
git cherry-pick start-commit..end-commit
```

### 4.6 Collaboration Workflows

#### Code Review Process
```bash
# Request review
git request-pull -p origin/main https://github.com/username/repo

# Review changes
git diff main..feature-branch

# Apply suggested changes
git commit --amend
git push --force-with-lease
```

#### Team Collaboration
```bash
# Set upstream for tracking
git branch --set-upstream-to=origin/main main

# Sync with upstream
git fetch upstream
git merge upstream/main

# Create patch
git format-patch -1 HEAD

# Apply patch
git apply patch-file.patch
```

### 4.7 Git Hooks and Automation

#### Pre-commit Hooks
```bash
# Create pre-commit hook
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/sh
# Run tests before commit
npm test
EOF

chmod +x .git/hooks/pre-commit
```

#### Post-commit Hooks
```bash
# Create post-commit hook
cat > .git/hooks/post-commit << 'EOF'
#!/bin/sh
# Send notification after commit
echo "Commit completed: $(git log -1 --oneline)"
EOF

chmod +x .git/hooks/post-commit
```

### 4.8 Git Submodules and Subtree

#### Submodules
```bash
# Add submodule
git submodule add https://github.com/username/library.git

# Initialize submodules
git submodule init
git submodule update

# Update all submodules
git submodule update --remote
```

#### Subtree
```bash
# Add subtree
git subtree add --prefix=vendor/library https://github.com/username/library.git main

# Push subtree changes
git subtree push --prefix=vendor/library https://github.com/username/library.git main
```

### 4.9 Git Best Practices

#### Commit Messages
- Use imperative mood ("Add feature" not "Added feature")
- Keep first line under 50 characters
- Separate subject from body with blank line
- Use body to explain what and why, not how

#### Branch Naming
- `feature/feature-name` for new features
- `bugfix/bug-description` for bug fixes
- `hotfix/urgent-fix` for critical fixes
- `release/version-number` for releases

#### Workflow Best Practices
- Commit frequently with meaningful messages
- Use branches for features and fixes
- Keep main branch stable
- Review code before merging
- Use descriptive commit messages
- Avoid committing large files or sensitive data

### 4.10 Troubleshooting Common Issues

#### Undoing Changes
```bash
# Undo last commit (keep changes)
git reset --soft HEAD~1

# Undo last commit (discard changes)
git reset --hard HEAD~1

# Undo uncommitted changes
git checkout -- filename.txt

# Revert specific commit
git revert commit-hash
```

#### Recovering Lost Commits
```bash
# View reflog
git reflog

# Recover lost commit
git checkout -b recovery-branch commit-hash
```

#### Cleaning Repository
```bash
# Remove untracked files
git clean -f

# Remove untracked files and directories
git clean -fd

# Preview what would be cleaned
git clean -n
```

## Next Steps

- [Practice](./01-practice.md)