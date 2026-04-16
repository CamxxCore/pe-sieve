# Syncing with Upstream Repository

This repository is a fork of [hasherezade/pe-sieve](https://github.com/hasherezade/pe-sieve). This guide explains how to keep your fork synchronized with the original repository.

## Repository Setup

- **Origin** (Your Fork): `https://github.com/CamxxCore/pe-sieve.git`
- **Upstream** (Original): `https://github.com/hasherezade/pe-sieve.git`

## Syncing with Upstream

### 1. Fetch Latest Changes from Upstream

```powershell
# Fetch all branches and commits from upstream
git fetch upstream

# View what changed
git log --oneline master..upstream/master
```

### 2. Merge Upstream Changes into Your Fork

```powershell
# Make sure you're on your master branch
git checkout master

# Merge upstream changes
git merge upstream/master

# Or use rebase to maintain a linear history
git rebase upstream/master
```

### 3. Push Updated Master to Your Fork

```powershell
# Push the synchronized changes to your GitHub fork
git push origin master
```

## Complete Sync Workflow

```powershell
# One-command sync (merge strategy)
git checkout master && git fetch upstream && git merge upstream/master && git push origin master

# One-command sync (rebase strategy - cleaner history)
git checkout master && git fetch upstream && git rebase upstream/master && git push origin master
```

## Handling Conflicts

If you have local changes that conflict with upstream:

```powershell
# See what files have conflicts
git status

# Edit conflicted files, then:
git add <resolved-files>
git merge --continue  # if merging
# or
git rebase --continue  # if rebasing
```

## Creating Pull Requests to Upstream

To contribute your changes back to the original repository:

1. **Push your feature branch to your fork:**
   ```powershell
   git push origin your-feature-branch
   ```

2. **Create PR on GitHub:**
   - Go to https://github.com/hasherezade/pe-sieve
   - Click "New Pull Request"
   - Click "compare across forks"
   - Select: `hasherezade/pe-sieve:master` ← `CamxxCore/pe-sieve:your-feature-branch`
   - Describe your changes and submit

## Checking Synchronization Status

```powershell
# Compare your master with upstream
git fetch upstream
git log --oneline --graph --left-right --cherry-pick master...upstream/master

# Count commits ahead/behind
git rev-list --left-right --count master...upstream/master
```

## Advanced: Cherry-Pick Specific Commits

If you only want specific upstream commits:

```powershell
# Fetch upstream
git fetch upstream

# View commits
git log upstream/master

# Cherry-pick specific commit
git cherry-pick <commit-hash>

# Push to your fork
git push origin master
```

## Useful Aliases

Add these to your git config for easier upstream management:

```powershell
git config alias.sync-upstream '!git fetch upstream && git merge upstream/master && git push origin master'
git config alias.fetch-up 'fetch upstream'
git config alias.show-upstream 'log --oneline master..upstream/master'
```

Then use:
```powershell
git sync-upstream       # Complete sync in one command
git show-upstream       # View upstream changes
```

## Emergency: Reset to Upstream

If your fork is completely out of sync and you want to match upstream exactly:

```powershell
# ⚠️ WARNING: This discards ALL your local changes!
git fetch upstream
git checkout master
git reset --hard upstream/master
git push --force origin master
```

## Notes

- **Merge vs Rebase**: 
  - Use `merge` if you want to preserve the complete history
  - Use `rebase` if you want a cleaner, linear history
  
- **When to Sync**: 
  - Before starting new features (ensures you're working with latest code)
  - Regularly (weekly/monthly) to avoid large divergence
  - Before creating pull requests to upstream

- **Your Custom Changes**: 
  - Keep your customizations in separate feature branches
  - Don't modify master directly - makes syncing easier
  - Document your changes in commit messages

## Verification

To verify the upstream remote is configured correctly:

```powershell
git remote -v
# Should show both 'origin' and 'upstream'

git fetch upstream
# Should successfully fetch from hasherezade/pe-sieve
```
