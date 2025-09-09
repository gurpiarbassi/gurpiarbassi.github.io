---
layout: post
title: "Git Worktree: Work on Multiple Branches Simultaneously"
published: true
date: 2025-09-09
categories: git productivity
---

# Motivation

In this post, we'll explore Git worktree - a powerful feature that allows you to work on multiple branches of the same repository simultaneously without cloning it multiple times. This is especially useful when you need to switch between different features, hotfixes, or versions without losing your current work.

## What is Git Worktree?

Git worktree allows you to have multiple working directories associated with a single Git repository. Each worktree can be on a different branch, enabling you to work on multiple features or versions simultaneously without the overhead of multiple repository clones.

### Key Benefits

- **Multiple Branches**: Work on different branches simultaneously
- **Space Efficient**: No need to clone the entire repository multiple times
- **Shared History**: All worktrees share the same Git history and objects
- **Independent Working**: Each worktree has its own working directory
- **Fast Switching**: No need to stash/unstash when switching contexts

## Basic Git Worktree Commands

### 1. **Create a New Worktree**

```bash
# Create a worktree for a new branch
git worktree add ../feature-branch feature/new-feature

# Create a worktree for an existing branch
git worktree add ../hotfix-branch hotfix/critical-bug

# Create a worktree in a specific directory
git worktree add /path/to/directory branch-name
```

### 2. **List All Worktrees**

```bash
# List all worktrees
git worktree list

# List worktrees with more details
git worktree list --porcelain
```

### 3. **Remove a Worktree**

```bash
# Remove a worktree (must be clean)
git worktree remove ../feature-branch

# Force remove a worktree (even if dirty)
git worktree remove --force ../feature-branch
```

### 4. **Prune Worktrees**

```bash
# Remove worktrees that no longer exist
git worktree prune
```

## Common Use Cases

### 1. **Feature Development**

```bash
# Main development
cd /project/main
git checkout main

# Create feature branch worktree
git worktree add ../feature-auth feature/user-authentication
cd ../feature-auth

# Work on feature without affecting main
git checkout feature/user-authentication
# ... make changes, commit, push
```

### 2. **Hotfix Development**

```bash
# Working on main branch
cd /project/main

# Create hotfix worktree
git worktree add ../hotfix-1.2.3 hotfix/critical-security-fix
cd ../hotfix-1.2.3

# Fix critical issue
git checkout hotfix/critical-security-fix
# ... make fixes, test, commit
```

### 3. **Version Comparison**

```bash
# Current version
cd /project/main
git checkout main

# Compare with previous version
git worktree add ../v1.0.0 v1.0.0
cd ../v1.0.0

# Now you can compare files side by side
diff -r /project/main /project/v1.0.0
```

### 4. **Testing Different Configurations**

```bash
# Production config
cd /project/main
git checkout main

# Development config
git worktree add ../dev development
cd ../dev
git checkout development

# Staging config
git worktree add ../staging staging
cd ../staging
git checkout staging
```

## Advanced Worktree Usage

### 1. **Worktree with Specific Remote**

```bash
# Create worktree tracking specific remote branch
git worktree add -b feature/new-feature ../feature-branch origin/develop
```

### 2. **Worktree with Detached HEAD**

```bash
# Create worktree at specific commit
git worktree add ../debug-commit abc1234
cd ../debug-commit
# You're now at commit abc1234 in detached HEAD state
```

### 3. **Worktree with Lock**

```bash
# Lock a worktree (useful for CI/CD)
git worktree add --lock ../deploy-branch deploy
# Worktree is locked and won't be pruned
```

### 4. **Worktree with Custom Directory Structure**

```bash
# Create worktree in custom location
git worktree add /tmp/feature-work feature/new-feature

# Or relative to current directory
git worktree add ../worktrees/feature-auth feature/user-authentication
```

## Worktree Management

### 1. **Check Worktree Status**

```bash
# See which worktrees are dirty
git worktree list --porcelain | grep -A1 "branch"

# Check if worktree is clean
cd /path/to/worktree
git status
```

### 2. **Move Worktree**

```bash
# Move worktree to new location
git worktree move ../old-location ../new-location
```

### 3. **Repair Worktree**

```bash
# Repair worktree if .git file is corrupted
git worktree repair /path/to/worktree
```

## Best Practices

### 1. **Organize Worktree Directories**

```bash
# Create a worktrees directory
mkdir worktrees
git worktree add worktrees/feature-auth feature/user-authentication
git worktree add worktrees/hotfix-1.2.3 hotfix/critical-bug
git worktree add worktrees/version-1.0.0 v1.0.0
```

### 2. **Use Descriptive Names**

```bash
# Good: Descriptive worktree names
git worktree add ../feature-user-auth feature/user-authentication
git worktree add ../hotfix-security hotfix/security-patch

# Avoid: Generic names
git worktree add ../work1 feature/user-authentication
git worktree add ../work2 hotfix/security-patch
```

### 3. **Clean Up Regularly**

```bash
# Remove completed worktrees
git worktree remove ../feature-auth

# Prune orphaned worktrees
git worktree prune
```

### 4. **Use Worktree for CI/CD**

```bash
# Create worktree for deployment
git worktree add --lock ../deploy-branch deploy
cd ../deploy-branch
# Run deployment scripts
```

## Common Pitfalls

### 1. **Working in Wrong Directory**

```bash
# Make sure you're in the right worktree
pwd
git branch
```

### 2. **Forgetting to Remove Worktrees**

```bash
# Always clean up when done
git worktree remove ../feature-branch
```

### 3. **Mixing Worktree and Regular Git Commands**

```bash
# Don't use git checkout to switch branches in worktree
# Use git worktree add instead
git worktree add ../new-branch new-branch
```

### 4. **Not Checking Worktree Status**

```bash
# Always check if worktree is clean before removing
git status
git worktree remove ../feature-branch
```

## Worktree vs Other Approaches

### Worktree vs Multiple Clones

| Approach | Space Usage | Setup Time | Branch Switching | Isolation |
|----------|-------------|------------|------------------|-----------|
| Multiple Clones | High | Slow | Fast | Complete |
| Git Worktree | Low | Fast | Fast | Good |
| Git Stash | Low | Fast | Slow | Poor |

### Worktree vs Git Stash

```bash
# Using stash (slower, can lose work)
git stash
git checkout other-branch
# ... work on other branch
git checkout original-branch
git stash pop

# Using worktree (faster, safer)
git worktree add ../other-branch other-branch
cd ../other-branch
# ... work on other branch
# No need to switch back and forth
```

## Integration with IDEs

### VS Code

```bash
# Open specific worktree in VS Code
code /path/to/worktree

# Or open multiple worktrees
code /path/to/main-worktree
code /path/to/feature-worktree
```

### IntelliJ IDEA

```bash
# Open worktree as new project
idea /path/to/worktree
```

## Summary

Git worktree is a powerful feature that enables efficient multi-branch development. Key takeaways:

1. **Space Efficient** - No need to clone repositories multiple times
2. **Fast Context Switching** - Work on multiple branches simultaneously
3. **Shared History** - All worktrees share the same Git objects
4. **Independent Working** - Each worktree has its own working directory
5. **Easy Cleanup** - Simple commands to manage worktrees

### When to Use Git Worktree

- Working on multiple features simultaneously
- Comparing different versions or branches
- Testing different configurations
- Hotfix development while maintaining main work
- CI/CD deployment scenarios

### When to Avoid Git Worktree

- Simple single-branch development
- When you prefer to clone repositories
- When working with very large repositories
- When you need complete isolation between branches

Remember: Git worktree is a tool to improve productivity, not a replacement for good Git workflow practices. Use it when it makes sense for your development process!
