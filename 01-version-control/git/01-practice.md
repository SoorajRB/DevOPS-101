# Git Practice Guide

## Overview
This practice guide covers essential Git concepts and provides hands-on exercises to build practical Git skills for DevOps workflows.

## Prerequisites
- Git installed on your system
- Basic command line familiarity
- A GitHub/GitLab account (optional but recommended)

## Exercise 1: Git Repository Setup

### Objective
Learn to initialize a Git repository and make your first commit.

### Steps
1. **Create a new directory and initialize Git:**
   ```bash
   mkdir my-devops-project
   cd my-devops-project
   git init
   ```

2. **Configure Git (if not already done):**
   ```bash
   git config --global user.name "Your Name"
   git config --global user.email "your.email@example.com"
   ```

3. **Create your first file:**
   ```bash
   echo "# My DevOps Project" > README.md
   ```

4. **Add and commit your first file:**
   ```bash
   git add README.md
   git commit -m "Initial commit: Add README"
   ```

### Expected Outcome
- A Git repository with one commit
- Understanding of `git init`, `git add`, and `git commit`

## Exercise 2: Basic Git Workflow

### Objective
Practice the basic Git workflow: modify, stage, and commit.

### Steps
1. **Create a new file:**
   ```bash
   echo "This is a sample configuration file" > config.txt
   ```

2. **Check repository status:**
   ```bash
   git status
   ```

3. **Stage and commit the new file:**
   ```bash
   git add config.txt
   git commit -m "Add configuration file"
   ```

4. **Modify an existing file:**
   ```bash
   echo "Updated project description" >> README.md
   ```

5. **Stage and commit the modification:**
   ```bash
   git add README.md
   git commit -m "Update README with project description"
   ```

### Expected Outcome
- Understanding of `git status`
- Multiple commits in repository
- Familiarity with staging and committing workflow

## Exercise 3: Branching and Merging

### Objective
Learn to create branches, make changes, and merge them back.

### Steps
1. **Create and switch to a new branch:**
   ```bash
   git checkout -b feature/new-feature
   ```

2. **Make changes on the new branch:**
   ```bash
   echo "New feature implementation" > feature.txt
   git add feature.txt
   git commit -m "Add new feature implementation"
   ```

3. **Switch back to main branch:**
   ```bash
   git checkout main
   ```

4. **Merge the feature branch:**
   ```bash
   git merge feature/new-feature
   ```

5. **Delete the feature branch:**
   ```bash
   git branch -d feature/new-feature
   ```

### Expected Outcome
- Understanding of branching workflow
- Successful merge of feature branch
- Clean repository state

## Exercise 4: Working with Remote Repositories

### Objective
Learn to work with remote repositories (GitHub/GitLab).

### Steps
1. **Create a repository on GitHub/GitLab**
   - Go to GitHub.com or GitLab.com
   - Create a new repository (don't initialize with README)

2. **Add remote origin:**
   ```bash
   git remote add origin https://github.com/yourusername/your-repo-name.git
   ```

3. **Push your local repository:**
   ```bash
   git push -u origin main
   ```

4. **Make a change and push:**
   ```bash
   echo "Remote repository test" > remote-test.txt
   git add remote-test.txt
   git commit -m "Add remote test file"
   git push
   ```

### Expected Outcome
- Local repository connected to remote
- Changes pushed to remote repository
- Understanding of remote workflow

## Exercise 5: Conflict Resolution

### Objective
Learn to handle merge conflicts.

### Steps
1. **Create a branch and make changes:**
   ```bash
   git checkout -b conflict-branch
   echo "Content from conflict branch" > conflict-file.txt
   git add conflict-file.txt
   git commit -m "Add content from conflict branch"
   ```

2. **Switch to main and make conflicting changes:**
   ```bash
   git checkout main
   echo "Content from main branch" > conflict-file.txt
   git add conflict-file.txt
   git commit -m "Add content from main branch"
   ```

3. **Attempt to merge (this will create a conflict):**
   ```bash
   git merge conflict-branch
   ```

4. **Resolve the conflict:**
   - Open `conflict-file.txt` in your editor
   - You'll see conflict markers: `<<<<<<<`, `=======`, `>>>>>>>`
   - Edit the file to resolve the conflict
   - Remove the conflict markers

5. **Complete the merge:**
   ```bash
   git add conflict-file.txt
   git commit -m "Resolve merge conflict"
   ```

### Expected Outcome
- Understanding of merge conflicts
- Ability to resolve conflicts manually
- Successful conflict resolution

## Exercise 6: Git Log and History

### Objective
Learn to explore Git history and understand commits.

### Steps
1. **View commit history:**
   ```bash
   git log
   ```

2. **View compact history:**
   ```bash
   git log --oneline
   ```

3. **View history with graph:**
   ```bash
   git log --graph --oneline --all
   ```

4. **View changes in a specific commit:**
   ```bash
   git show <commit-hash>
   ```

5. **View file history:**
   ```bash
   git log --follow README.md
   ```

### Expected Outcome
- Understanding of Git history commands
- Ability to explore commit history
- Knowledge of different log formats

## Exercise 7: Stashing and Temporary Changes

### Objective
Learn to temporarily save changes without committing.

### Steps
1. **Make changes to a file:**
   ```bash
   echo "Temporary changes" >> README.md
   ```

2. **Stash the changes:**
   ```bash
   git stash
   ```

3. **Check that changes are stashed:**
   ```bash
   git status
   ```

4. **Apply the stashed changes:**
   ```bash
   git stash pop
   ```

5. **View stash list:**
   ```bash
   git stash list
   ```

### Expected Outcome
- Understanding of stashing workflow
- Ability to temporarily save work
- Knowledge of stash management

## Exercise 8: Git Tags and Releases

### Objective
Learn to create tags for releases and important milestones.

### Steps
1. **Create a lightweight tag:**
   ```bash
   git tag v1.0.0
   ```

2. **Create an annotated tag:**
   ```bash
   git tag -a v1.0.1 -m "Release version 1.0.1"
   ```

3. **List all tags:**
   ```bash
   git tag
   ```

4. **View tag information:**
   ```bash
   git show v1.0.1
   ```

5. **Push tags to remote:**
   ```bash
   git push origin --tags
   ```

### Expected Outcome
- Understanding of Git tagging
- Ability to create different types of tags
- Knowledge of tag management

## Exercise 9: Git Configuration and Aliases

### Objective
Learn to customize Git configuration and create useful aliases.

### Steps
1. **View current Git configuration:**
   ```bash
   git config --list
   ```

2. **Set up useful aliases:**
   ```bash
   git config --global alias.st status
   git config --global alias.co checkout
   git config --global alias.br branch
   git config --global alias.ci commit
   git config --global alias.unstage 'reset HEAD --'
   ```

3. **Create a custom alias for log:**
   ```bash
   git config --global alias.lg "log --oneline --graph --decorate"
   ```

4. **Test your aliases:**
   ```bash
   git st
   git lg
   ```

### Expected Outcome
- Customized Git configuration
- Useful aliases for common commands
- Improved Git workflow efficiency

## Exercise 10: Git Hooks (Advanced)

### Objective
Learn about Git hooks for automation.

### Steps
1. **Explore hooks directory:**
   ```bash
   ls .git/hooks/
   ```

2. **Create a pre-commit hook:**
   ```bash
   cat > .git/hooks/pre-commit << 'EOF'
   #!/bin/bash
   echo "Running pre-commit checks..."
   # Add your custom checks here
   echo "Pre-commit checks completed."
   EOF
   ```

3. **Make the hook executable:**
   ```bash
   chmod +x .git/hooks/pre-commit
   ```

4. **Test the hook:**
   ```bash
   echo "Test commit" > test-file.txt
   git add test-file.txt
   git commit -m "Test pre-commit hook"
   ```

### Expected Outcome
- Understanding of Git hooks
- Basic hook implementation
- Knowledge of automation possibilities

## Best Practices

### Commit Messages
- Use clear, descriptive commit messages
- Start with a verb in present tense
- Keep the first line under 50 characters
- Use body for detailed explanations

### Branching Strategy
- Use feature branches for new development
- Keep main/master branch stable
- Delete merged branches
- Use descriptive branch names

### Regular Workflow
- Pull latest changes before starting work
- Commit frequently with meaningful messages
- Test before pushing to remote
- Review changes before merging

## Common Git Commands Reference

| Command | Description |
|---------|-------------|
| `git init` | Initialize a new repository |
| `git clone <url>` | Clone a remote repository |
| `git add <file>` | Stage changes |
| `git commit -m "message"` | Commit staged changes |
| `git status` | Check repository status |
| `git log` | View commit history |
| `git branch` | List branches |
| `git checkout <branch>` | Switch to branch |
| `git merge <branch>` | Merge branch into current |
| `git pull` | Fetch and merge from remote |
| `git push` | Push commits to remote |
| `git stash` | Temporarily save changes |
| `git tag <name>` | Create a tag |

## Troubleshooting

### Common Issues and Solutions

1. **Accidentally committed to wrong branch:**
   ```bash
   git reset --soft HEAD~1
   git checkout correct-branch
   git cherry-pick <commit-hash>
   ```

2. **Undo last commit:**
   ```bash
   git reset --soft HEAD~1
   ```

3. **Discard local changes:**
   ```bash
   git checkout -- <file>
   ```

4. **Fix wrong commit message:**
   ```bash
   git commit --amend -m "Correct message"
   ```

## Next Steps

After completing these exercises, consider exploring:
- Git workflows (GitFlow, GitHub Flow)
- Advanced Git features (rebase, cherry-pick)
- Git hosting platforms (GitHub, GitLab, Bitbucket)
- CI/CD integration with Git
- Git security best practices

## Resources

- [Git Documentation](https://git-scm.com/doc)
- [Pro Git Book](https://git-scm.com/book/en/v2)
