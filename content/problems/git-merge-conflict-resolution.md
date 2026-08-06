---
title: "Git Merge Conflict Resolution - The Right Way"
date: 2026-08-05T11:00:00Z
draft: false
description: "How to properly resolve git merge conflicts without losing work"
categories: ["Development"]
tags: ["git", "version-control", "collaboration"]
---

## Problem

Merge conflict when trying to merge feature branch into main:

```
Auto-merging src/config.py
CONFLICT (content): Merge conflict in src/config.py
Automatic merge failed; fix conflicts and then commit the result.
```

## Root Cause

Two branches modified the same lines in `src/config.py`. Git cannot automatically determine which change to keep.

## Attempted Solutions

- **Attempt 1** — Accepted "theirs" with `git merge -X theirs branch`. Lost local changes.
- **Attempt 2** — Manually edited file but forgot to mark as resolved. Commit failed.

## Final Solution

```bash
# See which files have conflicts
git status

# Open conflicted file, look for conflict markers:
# <<<<<<< HEAD
# your changes
# =======
# their changes
# >>>>>>> branch-name

# Edit file, keep correct code, remove markers

# Stage resolved file
git add src/config.py

# Complete the merge
git commit -m "Merge branch 'feature' - resolve config conflict"
```

## Why It Works

Git marks conflicts with clear markers. Manual resolution ensures correct code is kept. `git add` tells git the conflict is resolved. Final commit completes the merge.
