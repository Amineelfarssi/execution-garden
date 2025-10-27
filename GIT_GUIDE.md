# Git Usage Guide for Execution Garden Architecture

## Quick Reference

### Initial Setup (First Time Only)

```bash
# Configure your identity
git config --global user.name "Your Name"
git config --global user.email "your.email@bank.com"

# Clone the repository (if hosted remotely)
git clone <repository-url>
cd execution-garden-architecture

# Or if starting from this local repo
cd /home/claude/execution-garden-architecture
```

### Daily Workflow

#### 1. Check Status
```bash
# See what files have changed
git status

# See detailed changes
git diff
```

#### 2. Stage Changes
```bash
# Stage specific files
git add ARCHITECTURE.md
git add docs/network/transit-gateway.md

# Stage all changes
git add .

# Stage files interactively
git add -p
```

#### 3. Commit Changes
```bash
# Commit with message
git commit -m "Add Transit Gateway routing details"

# Commit with detailed message
git commit -m "Update Phase 2 implementation" -m "- Added LiteLLM deployment steps
- Updated cost estimates
- Fixed typo in guardrails section"

# Amend last commit (before pushing)
git commit --amend
```

#### 4. View History
```bash
# See commit log
git log

# Compact view
git log --oneline --graph --all

# See changes in last commit
git show

# See changes in specific commit
git show <commit-hash>
```

### Branching Strategy

#### Feature Branches
```bash
# Create and switch to new branch
git checkout -b feature/network-updates

# Make your changes...
git add .
git commit -m "Update network architecture"

# Switch back to main
git checkout main

# Merge feature branch
git merge feature/network-updates

# Delete feature branch
git branch -d feature/network-updates
```

#### Common Branch Types
- `feature/*` - New features or sections
- `fix/*` - Bug fixes, corrections
- `docs/*` - Documentation updates
- `refactor/*` - Restructuring without changing functionality
- `release/*` - Preparing new version release

### Versioning Workflow

#### Creating a New Version

1. **Update CHANGELOG.md**
```bash
# Move [Unreleased] items to new version section
# Add version number and date: [1.1.0] - 2025-11-15
git add CHANGELOG.md
git commit -m "Prepare v1.1.0 release"
```

2. **Tag the Release**
```bash
# Create annotated tag
git tag -a v1.1.0 -m "Release version 1.1.0: Add Phase 2 details"

# View tags
git tag -l

# Show tag details
git show v1.1.0
```

3. **Push to Remote (if using remote repository)**
```bash
# Push commits
git push origin main

# Push tags
git push origin v1.1.0

# Or push all tags
git push origin --tags
```

### Collaboration

#### Pull Latest Changes
```bash
# Fetch and merge from remote
git pull origin main

# Fetch without merging
git fetch origin

# See what would be pulled
git fetch origin --dry-run
```

#### Resolve Conflicts
```bash
# If conflicts occur during merge/pull
# 1. Open conflicted files
# 2. Edit to resolve conflicts (look for <<<<<<, ======, >>>>>>)
# 3. Stage resolved files
git add <resolved-file>

# 4. Complete the merge
git commit -m "Merge resolved conflicts"
```

### Advanced Operations

#### View Specific Version
```bash
# Checkout specific version (detached HEAD state)
git checkout v1.0.0

# Return to latest
git checkout main
```

#### Compare Versions
```bash
# Compare two versions
git diff v1.0.0 v1.1.0

# Compare specific file between versions
git diff v1.0.0 v1.1.0 -- ARCHITECTURE.md
```

#### Undo Changes
```bash
# Discard uncommitted changes to file
git checkout -- ARCHITECTURE.md

# Undo last commit (keep changes)
git reset --soft HEAD~1

# Undo last commit (discard changes) - DANGEROUS!
git reset --hard HEAD~1

# Revert specific commit (creates new commit)
git revert <commit-hash>
```

#### Stash Changes
```bash
# Save current changes temporarily
git stash

# List stashes
git stash list

# Apply most recent stash
git stash apply

# Apply and remove stash
git stash pop

# Discard stash
git stash drop
```

### Remote Repository Setup

#### GitHub/GitLab/Bitbucket
```bash
# Add remote repository
git remote add origin https://github.com/your-org/execution-garden-architecture.git

# Verify remote
git remote -v

# Push to remote for first time
git push -u origin main

# Push tags
git push origin --tags
```

#### AWS CodeCommit
```bash
# Add AWS CodeCommit remote
git remote add origin https://git-codecommit.us-east-1.amazonaws.com/v1/repos/execution-garden

# Configure AWS credentials helper
git config --global credential.helper '!aws codecommit credential-helper $@'
git config --global credential.UseHttpPath true

# Push to CodeCommit
git push -u origin main
```

### Best Practices

#### Commit Messages
- Use present tense: "Add feature" not "Added feature"
- Keep first line under 50 characters
- Add detailed description after blank line if needed
- Reference issues/tickets: "Fix #123: Correct IAM policy"

Good commit messages:
```
✅ Add Transit Gateway routing configuration
✅ Update Phase 2 cost estimates to $1,070/month
✅ Fix typo in guardrails section
✅ Refactor: Reorganize security documentation

❌ Fixed stuff
❌ Updates
❌ WIP
```

#### When to Commit
- After completing a logical unit of work
- Before switching context
- At the end of each work session
- When tests pass (if applicable)

#### When to Create Version Tag
- After significant milestone (phase completion)
- Before major architectural changes
- Quarterly reviews
- When approved by architecture board

### Useful Aliases

Add these to your Git config:
```bash
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.unstage 'reset HEAD --'
git config --global alias.last 'log -1 HEAD'
git config --global alias.lg "log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"
```

### Troubleshooting

#### Problem: Accidentally committed sensitive data
```bash
# Remove file from last commit
git rm --cached secrets.yml
git commit --amend --no-edit

# If already pushed - use git-filter-repo or contact admin
```

#### Problem: Need to update .gitignore after files committed
```bash
# Remove files from tracking
git rm -r --cached .
git add .
git commit -m "Apply .gitignore rules"
```

#### Problem: Diverged branches
```bash
# Option 1: Merge
git pull origin main

# Option 2: Rebase (cleaner history)
git pull --rebase origin main
```

### Getting Help

```bash
# General help
git help

# Help for specific command
git help commit
git help branch

# Quick reference
git <command> --help
```

---

## Project-Specific Conventions

### Branch Naming
- `main` - Production-ready architecture
- `develop` - Integration branch for features
- `feature/short-description` - Feature branches
- `hotfix/issue-description` - Urgent fixes

### Review Process
- Create feature branch
- Make changes and commit
- Push to remote
- Create pull/merge request
- Get 2 approvals (Architect + Security)
- Merge to main
- Tag version if milestone

### Version Strategy
- **MAJOR** (2.0.0): Breaking architecture changes
- **MINOR** (1.1.0): New phases, significant additions
- **PATCH** (1.0.1): Corrections, clarifications

---

**Last Updated**: October 27, 2025  
**Maintained By**: Architecture Team
