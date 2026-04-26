---
title: "Bare repo worktrees: core.bare override required with extensions.worktreeConfig"
tags: ["git", "worktree", "ops"]
sources:
  - url: "https://raw.githubusercontent.com/git/git/v2.54.0/Documentation/config/extensions.adoc"
    title: ""
    accessed_at: "2026-04-23"
  - url: "https://raw.githubusercontent.com/git/git/v2.54.0/Documentation/git-worktree.adoc"
    title: ""
    accessed_at: "2026-04-23"
contributors: ["WSvX"]
created: 2026-04-23
updated: 2026-04-23
---

When a bare repo has extensions.worktreeConfig=true in .git/config, linked worktrees inherit core.bare=true from the shared config. This causes git status and other working-tree commands to fail with: fatal: this operation must be run in a work tree.

## Root Cause

Without extensions.worktreeConfig, git automatically exempts linked worktrees from core.bare in the shared config. With the extension enabled, that exemption is removed and core.bare=true leaks to all linked worktrees.

## Fix

Create a config.worktree file in each linked worktree admin directory: .git/worktrees/<id>/config.worktree containing [core] bare=false. git worktree add does NOT create this automatically.

## Why not remove bare=true from .git/config?

Removing core.bare=true breaks git rev-parse --is-bare-repository (returns false). The bare root needs it. The docs say to move it to config.worktree but bare repos have no main worktree to move it to.

## Source

git/git v2.54.0 Documentation/config/extensions.adoc: If core.bare is true, then it must be moved from GIT_COMMON_DIR/config to GIT_COMMON_DIR/config.worktree

## Applies To

All linked worktrees from this bare repo. Any new worktrees created via git worktree add must have config.worktree created manually afterward.
