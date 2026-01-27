---
layout: post
title: "Git Bisect: Finding Bugs with Binary Search"
published: true
date: 2026-01-27
categories: git debugging productivity
---

# Motivation

In this post, we'll explore Git bisect - a powerful debugging tool that uses binary search to efficiently find the commit that introduced a bug. When you know a bug exists in the current code but worked correctly in a previous version, `git bisect` can help you pinpoint the exact commit that caused the problem, even in repositories with thousands of commits.

## What is Git Bisect?

Git bisect is a binary search algorithm implementation that helps you find the commit that introduced a bug. Instead of manually checking each commit (which could take hours or days), bisect systematically narrows down the problematic commit by testing commits in the middle of the range.

### Key Benefits

- **Efficient**: Finds bugs in O(log n) time instead of O(n)
- **Automated**: Can be fully automated with test scripts
- **Precise**: Identifies the exact commit that introduced the bug
- **Time Saving**: Saves hours or days of manual investigation
- **Works on Large Histories**: Effective even with thousands of commits

## Basic Git Bisect Workflow

### 1. **Start Bisect Session**

```bash
# Start bisect session
git bisect start

# Mark current commit as bad (contains the bug)
git bisect bad

# Mark a known good commit (bug didn't exist)
git bisect good v1.0.0
# or
git bisect good abc1234
# or
git bisect good HEAD~50
```

### 2. **Test Each Commit**

Git will automatically checkout a commit in the middle of the range. You need to test if the bug exists:

```bash
# After git checks out a commit, test your code
./run-tests.sh
# or
npm test
# or manually verify the bug

# If bug exists, mark as bad
git bisect bad

# If bug doesn't exist, mark as good
git bisect good
```

### 3. **Repeat Until Found**

Git will continue narrowing down until it finds the first bad commit:

```bash
# Git will keep checking out commits until it finds the culprit
# Output will look like:
# Bisecting: 12 revisions left to test after this (roughly 4 steps)
# [abc1234] Commit message here
```

### 4. **Finish Bisect Session**

```bash
# Once the bad commit is found, reset to original state
git bisect reset

# This returns you to the branch you were on before starting bisect
```

## Practical Examples

### Example 1: Finding a Regression Bug

Let's say you have a bug in your application that wasn't present 2 weeks ago:

```bash
# Start bisect
git bisect start

# Current commit has the bug
git bisect bad

# Find a commit from 2 weeks ago that was good
git log --until="2 weeks ago" --oneline | head -1
# Output: def4567 Fix user authentication

# Mark that commit as good
git bisect good def4567

# Git checks out a commit in the middle
# Test the application
./test-suite.sh

# If bug exists
git bisect bad

# If bug doesn't exist
git bisect good

# Continue until git finds the culprit
# Output: abc1234 is the first bad commit
# abc1234
# Author: Developer Name
# Date: 2025-11-15
# 
#     Refactor authentication logic

# Reset
git bisect reset
```

### Example 2: Finding When a Feature Broke

You know a feature worked in version 1.5.0 but is broken now:

```bash
# Start bisect
git bisect start

# Current state is bad
git bisect bad

# Version 1.5.0 was good
git bisect good v1.5.0

# Git will checkout commits between v1.5.0 and HEAD
# Test the feature at each commit
./test-feature.sh

# Mark as good or bad based on test results
git bisect good  # or git bisect bad

# Continue until found
git bisect reset
```

### Example 3: Finding a Performance Regression

You notice the application is slower than it was last month:

```bash
# Start bisect
git bisect start
git bisect bad

# Find a commit from last month
git log --until="1 month ago" --oneline | head -1
git bisect good <commit-hash>

# At each commit, run performance tests
./benchmark.sh

# Check if performance is acceptable
if [ $? -eq 0 ]; then
    git bisect good
else
    git bisect bad
fi

# Continue until performance regression is found
git bisect reset
```

## Automated Bisect with Scripts

### Using a Test Script

You can automate the entire bisect process with a script:

```bash
# Create a test script
cat > test-bug.sh << 'EOF'
#!/bin/bash
# Run tests and exit with appropriate code
npm test
if [ $? -eq 0 ]; then
    exit 0  # Good commit
else
    exit 1  # Bad commit
fi
EOF

chmod +x test-bug.sh

# Run bisect with the script
git bisect start
git bisect bad
git bisect good v1.0.0
git bisect run ./test-bug.sh

# Git will automatically test each commit
# Output will show the first bad commit
git bisect reset
```

### Example: Python Test Script

```python
#!/usr/bin/env python3
# test_regression.py
import subprocess
import sys

def test_application():
    """Run tests and return True if all pass"""
    result = subprocess.run(['pytest', 'tests/'], capture_output=True)
    return result.returncode == 0

if __name__ == '__main__':
    if test_application():
        sys.exit(0)  # Good commit
    else:
        sys.exit(1)  # Bad commit
```

```bash
# Use the Python script
git bisect start
git bisect bad
git bisect good v1.0.0
git bisect run python3 test_regression.py
git bisect reset
```

### Example: Java/Maven Test Script

```bash
#!/bin/bash
# test-java.sh
mvn test -q
if [ $? -eq 0 ]; then
    exit 0
else
    exit 1
fi
```

```bash
git bisect start
git bisect bad
git bisect good release-2.0
git bisect run ./test-java.sh
git bisect reset
```

## Advanced Bisect Usage

### 1. **Skip Commits**

Sometimes a commit can't be tested (e.g., it doesn't compile):

```bash
# During bisect, if a commit can't be tested
git bisect skip

# Git will choose another commit to test
```

### 2. **Visualize Bisect Progress**

```bash
# Show current bisect status
git bisect log

# Show which commits are good/bad
git bisect visualize
# or
git bisect view
```

### 3. **Bisect with Specific Paths**

Focus bisect on changes in specific files or directories:

```bash
git bisect start
git bisect bad
git bisect good v1.0.0

# Only consider commits that changed specific files
git bisect run git diff HEAD~1 HEAD --quiet -- path/to/file.java
# or test only when specific files changed
git bisect run sh -c 'git diff HEAD~1 HEAD --name-only | grep -q "src/main" && ./test.sh || exit 125'
```

### 4. **Bisect Across Merges**

Bisect works across merge commits:

```bash
git bisect start
git bisect bad main
git bisect good develop

# Git will test commits including merge commits
```

### 5. **Save and Resume Bisect**

You can pause and resume a bisect session:

```bash
# Start bisect
git bisect start
git bisect bad
git bisect good v1.0.0

# Do some testing, then need to switch branches
git bisect log > bisect-log.txt
git bisect reset

# Later, resume
git bisect start
git bisect replay bisect-log.txt
```

## Real-World Scenarios

### Scenario 1: Production Bug Investigation

```bash
# Bug reported in production
# Last known good deployment: 2025-11-01
# Current deployment: 2025-11-15

git bisect start
git bisect bad HEAD
git bisect good $(git log --until="2025-11-01" --format="%H" -n 1)

# Test at each commit
curl http://localhost:8080/api/health
# Check if bug exists

git bisect good  # or bad

# Found: commit abc1234 introduced the bug
git show abc1234
# Review the changes

git bisect reset
```

### Scenario 2: Test Suite Failure

```bash
# Tests were passing, now they're failing
# Find when they started failing

git bisect start
git bisect bad
git bisect good $(git log --grep="CI.*pass" --format="%H" -n 1)

# Automate with test script
git bisect run ./run-tests.sh

# Output shows the commit that broke tests
git bisect reset
```

### Scenario 3: Finding Security Vulnerability Introduction

```bash
# Security audit found a vulnerability
# Need to find when it was introduced

git bisect start
git bisect bad
git bisect good v2.0.0  # Last security audit was clean

# Run security scanner at each commit
git bisect run security-scanner.sh

# Found the commit
git show <bad-commit>
# Review and create fix

git bisect reset
```

## Bisect Best Practices

### 1. **Use Descriptive Commit Messages**

Good commit messages make bisect results more useful:

```bash
# After finding the bad commit
git show <bad-commit>
# Clear commit message helps understand what changed
```

### 2. **Automate When Possible**

```bash
# Always use scripts for repeatable tests
git bisect run ./test-script.sh
```

### 3. **Test in Isolated Environment**

```bash
# Use clean environment for testing
git bisect run sh -c 'make clean && make test'
```

### 4. **Document Your Findings**

```bash
# After finding the bad commit
git bisect log > bisect-results.txt
git show <bad-commit> >> bisect-results.txt
```

### 5. **Use Tags for Known Good States**

```bash
# Tag releases to easily reference good commits
git tag v1.0.0
git tag v1.1.0

# Later, use tags in bisect
git bisect good v1.0.0
```

## Common Pitfalls and Solutions

### Pitfall 1: Flaky Tests

**Problem**: Tests are non-deterministic, causing false positives/negatives.

**Solution**: Run tests multiple times or use more reliable tests:

```bash
# Run tests multiple times
git bisect run sh -c './test.sh && ./test.sh && ./test.sh'
```

### Pitfall 2: Build Failures

**Problem**: Some commits don't compile.

**Solution**: Skip commits that don't build:

```bash
git bisect run sh -c 'make build || exit 125; ./test.sh'
# Exit code 125 tells git bisect to skip this commit
```

### Pitfall 3: Multiple Bugs

**Problem**: Multiple bugs were introduced at different times.

**Solution**: Fix bugs one at a time, or use more specific tests:

```bash
# Test for specific bug only
git bisect run ./test-specific-bug.sh
```

### Pitfall 4: Forgetting to Reset

**Problem**: Left in bisect state, causing confusion.

**Solution**: Always reset after bisect:

```bash
# Always run this after bisect
git bisect reset
```

## Bisect vs Other Debugging Methods

### Bisect vs Manual Checking

```bash
# Manual checking (slow, O(n))
git log --oneline | while read commit; do
    git checkout $commit
    ./test.sh
done

# Bisect (fast, O(log n))
git bisect start
git bisect bad
git bisect good <old-commit>
git bisect run ./test.sh
```

### Bisect vs Git Blame

```bash
# Git blame shows who last changed a line
git blame file.java

# Bisect finds when a behavior changed
git bisect start
git bisect bad
git bisect good <old-commit>
git bisect run ./test-behavior.sh
```

## Integration with CI/CD

### Automated Bisect in CI

```yaml
# .github/workflows/bisect.yml
name: Git Bisect
on:
  workflow_dispatch:
    inputs:
      good_commit:
        description: 'Known good commit'
        required: true

jobs:
  bisect:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run bisect
        run: |
          git bisect start
          git bisect bad HEAD
          git bisect good ${{ github.event.inputs.good_commit }}
          git bisect run ./test-suite.sh
          git bisect log > bisect-results.txt
          git bisect reset
      - name: Upload results
        uses: actions/upload-artifact@v3
        with:
          name: bisect-results
          path: bisect-results.txt
```

## Summary

Git bisect is an essential tool for efficient bug hunting. Key takeaways:

1. **Binary Search Efficiency** - Finds bugs in logarithmic time
2. **Automation Support** - Can be fully automated with scripts
3. **Precise Results** - Identifies exact commit introducing the bug
4. **Works on Large Histories** - Effective even with thousands of commits
5. **Flexible** - Supports skipping commits, specific paths, and more

### When to Use Git Bisect

- Finding when a bug was introduced
- Investigating regressions
- Performance issue debugging
- Security vulnerability tracking
- Test failure investigation
- Feature breakage analysis

### When to Avoid Git Bisect

- Bugs that are non-deterministic (flaky tests)
- When you already know the approximate time/commit
- When tests take too long to run
- When the codebase has too many breaking changes between commits

### Quick Reference

```bash
# Start bisect
git bisect start
git bisect bad [commit]
git bisect good [commit]

# During bisect
git bisect good
git bisect bad
git bisect skip

# Automated
git bisect run <script>

# Finish
git bisect reset
git bisect log
git bisect visualize
```

Remember: Git bisect is most effective when you have reliable, automated tests. The more automated your testing, the more powerful bisect becomes!

