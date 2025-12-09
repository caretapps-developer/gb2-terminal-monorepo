# Branching Guide for GB2 Terminal Monorepo

This guide explains how to work with branches across the monorepo and its submodules.

## Current Branch Status

All repositories are now on the **`pay-on-auto-pilot`** branch:

- ✅ **gb2-terminal-monorepo**: `pay-on-auto-pilot`
- ✅ **gb2-terminal-web**: `pay-on-auto-pilot`
- ✅ **gb2-terminal-expo**: `pay-on-auto-pilot`

## Creating a New Feature Branch

To create a new feature branch across all repositories:

```bash
# 1. Create branch in monorepo
git checkout -b "feature-name"

# 2. Create branch in web submodule
cd gb2-terminal-web
git checkout -b "feature-name"
cd ..

# 3. Create branch in expo submodule
cd gb2-terminal-expo
git checkout -b "feature-name"
cd ..
```

### Quick Script

```bash
#!/bin/bash
BRANCH_NAME="$1"

echo "Creating branch: $BRANCH_NAME"

# Monorepo
git checkout -b "$BRANCH_NAME"

# Web submodule
cd gb2-terminal-web
git checkout -b "$BRANCH_NAME"
cd ..

# Expo submodule
cd gb2-terminal-expo
git checkout -b "$BRANCH_NAME"
cd ..

echo "✅ Branch '$BRANCH_NAME' created in all repositories"
```

## Checking Current Branches

To see which branch each repository is on:

```bash
echo "=== Monorepo ===" && git branch
echo "=== gb2-terminal-web ===" && cd gb2-terminal-web && git branch && cd ..
echo "=== gb2-terminal-expo ===" && cd gb2-terminal-expo && git branch && cd ..
```

## Switching Branches

To switch to an existing branch across all repositories:

```bash
# 1. Switch in monorepo
git checkout branch-name

# 2. Switch in web submodule
cd gb2-terminal-web
git checkout branch-name
cd ..

# 3. Switch in expo submodule
cd gb2-terminal-expo
git checkout branch-name
cd ..
```

## Making Changes on a Feature Branch

### Workflow for Changes in Web Submodule

```bash
# 1. Navigate to web submodule
cd gb2-terminal-web

# 2. Make your changes
# ... edit files ...

# 3. Commit in the submodule
git add .
git commit -m "Your commit message"

# 4. Push the submodule branch (optional, but recommended)
git push origin pay-on-auto-pilot

# 5. Return to monorepo root
cd ..

# 6. Update the submodule reference in monorepo
git add gb2-terminal-web
git commit -m "Update gb2-terminal-web submodule reference"
```

### Workflow for Changes in Expo Submodule

```bash
# 1. Navigate to expo submodule
cd gb2-terminal-expo

# 2. Make your changes
# ... edit files ...

# 3. Commit in the submodule
git add .
git commit -m "Your commit message"

# 4. Push the submodule branch (optional, but recommended)
git push origin pay-on-auto-pilot

# 5. Return to monorepo root
cd ..

# 6. Update the submodule reference in monorepo
git add gb2-terminal-expo
git commit -m "Update gb2-terminal-expo submodule reference"
```

## Pushing Feature Branch

To push all branches to their respective remotes:

```bash
# Push monorepo branch
git push origin pay-on-auto-pilot

# Push web submodule branch
cd gb2-terminal-web
git push origin pay-on-auto-pilot
cd ..

# Push expo submodule branch
cd gb2-terminal-expo
git push origin pay-on-auto-pilot
cd ..
```

## Merging Feature Branch Back to Main

### Step 1: Merge Submodules First

```bash
# Merge web submodule
cd gb2-terminal-web
git checkout main
git merge pay-on-auto-pilot
git push origin main
cd ..

# Merge expo submodule
cd gb2-terminal-expo
git checkout main
git merge pay-on-auto-pilot
git push origin main
cd ..
```

### Step 2: Update Monorepo References

```bash
# Update submodule references to point to merged main branches
git checkout main
git submodule update --remote --merge

# Commit the updated references
git add gb2-terminal-web gb2-terminal-expo
git commit -m "Update submodules to merged main branches"
```

### Step 3: Merge Monorepo Branch

```bash
# Merge the monorepo feature branch
git merge pay-on-auto-pilot
git push origin main
```

## Syncing with Remote Changes

If someone else has pushed changes to the feature branch:

```bash
# Update monorepo
git pull origin pay-on-auto-pilot

# Update web submodule
cd gb2-terminal-web
git pull origin pay-on-auto-pilot
cd ..

# Update expo submodule
cd gb2-terminal-expo
git pull origin pay-on-auto-pilot
cd ..
```

## Deleting Feature Branches

After merging, clean up the feature branches:

```bash
# Delete local branches
git branch -d pay-on-auto-pilot
cd gb2-terminal-web && git branch -d pay-on-auto-pilot && cd ..
cd gb2-terminal-expo && git branch -d pay-on-auto-pilot && cd ..

# Delete remote branches
git push origin --delete pay-on-auto-pilot
cd gb2-terminal-web && git push origin --delete pay-on-auto-pilot && cd ..
cd gb2-terminal-expo && git push origin --delete pay-on-auto-pilot && cd ..
```

## Best Practices

1. **Always commit in submodules first**, then update the monorepo reference
2. **Keep branch names consistent** across all repositories
3. **Push submodule changes** before pushing monorepo changes
4. **Communicate with team** when working on the same feature branch
5. **Test integration** after pulling changes from both submodules
6. **Document breaking changes** in the PostMessage API if any

## Troubleshooting

### Submodule is in "detached HEAD" state

```bash
cd gb2-terminal-web  # or gb2-terminal-expo
git checkout pay-on-auto-pilot
cd ..
```

### Submodule has uncommitted changes

```bash
cd gb2-terminal-web  # or gb2-terminal-expo
git status
# Either commit or stash the changes
git stash  # or git add . && git commit -m "..."
cd ..
```

### Monorepo shows submodule as modified

This is normal when the submodule is on a different commit. To update:

```bash
git add gb2-terminal-web  # or gb2-terminal-expo
git commit -m "Update submodule reference"
```

## Quick Reference Commands

```bash
# Check all branches
git branch && cd gb2-terminal-web && git branch && cd .. && cd gb2-terminal-expo && git branch && cd ..

# Pull all
git pull && cd gb2-terminal-web && git pull && cd .. && cd gb2-terminal-expo && git pull && cd ..

# Status all
git status && cd gb2-terminal-web && git status && cd .. && cd gb2-terminal-expo && git status && cd ..
```

## Working on "Pay on Auto Pilot" Feature

The current feature branch `pay-on-auto-pilot` is set up for developing the auto-pilot payment functionality. 

When making changes:
1. Consider the PostMessage API contract (see POSTMESSAGE_API.md)
2. Update XState machines if payment flow changes
3. Test the integration between web and native layers
4. Document any new message types or state changes

Happy coding! 🚀

