---
title: Review PRs without stashing, with git worktree
description: Work on multiple branches at once without stashing or cloning.
tags: tooling
date: 2025-10-13
---

You're mid-feature, server running, uncommitted changes everywhere. A teammate opens a pull request you want to run locally. Or you just wrote tests for your feature and want to run them on `main`. Either way, switching context is a pain.

You probably stash your work, check out the other branch, run things, then check out your branch and unstash. Or worse, clone the repo again just to avoid committing half-finished work.

There is a better way: `git worktree`.

> [!NOTE]
> A working directory is the checked‑out files for a specific branch or commit. By default, a repo has one working directory. `git worktree` lets you attach extra working directories that share the same `.git` store, each checked out at a different branch or commit.

## Review a PR without stashing

Say you're in `myproject/` on your feature branch (dirty or not). Fetch and create a new worktree for the branch you want to review:

```shell
git fetch
# 'other-branch' being the PR branch name
git worktree add ../myproject-other-branch origin/other-branch 
```

Now `../myproject-other-branch` is a clean checkout of that PR. You can `cd` into it, run tests, open it in your editor, or boot the app. Your main workspace is untouched. 

When you're done, simply run:

```shell
git worktree remove ../myproject-other-branch
```

Gone. No stashes, no half-merged leftovers, and assuming you've opened the other branch in a new editor instance, you even have your open tabs unchanged.

If you forget and delete the folder manually, `git worktree prune` cleans up references.

## Keep `main` around

Same idea when you just want to peek at `main`:

```shell
git worktree add ../myproject-main main
```

Now you'll have your current, uncommitted, work available in `myproject`, and a clean checkout of `main` in `../myproject-main`.

Run tests, compare behavior, or copy configs without leaving your context.

## Commands you will use

Once you get the hang of it, there are only a few commands to remember:

- List active worktrees: `git worktree list`
- Remove a worktree: `git worktree remove <path>`
- Clean up after manual deletes: `git worktree prune`

## Things to keep in mind

A branch can be checked out in only one worktree at a time. If you need the same code in two places, check out a different branch or use a detached HEAD.

Each worktree keeps its own working files like `node_modules`, builds, and caches. The repository data like commits, branches, tags and stashes, is shared across them all.

In practice this gives you room to review a PR without touching your current work, jump between branches quickly, run tests across branches, and fix something on `main` without killing your flow. It is also a safe place to experiment in a clean folder when you need one.

## Final words

Once you start reviewing PRs this way, you're not going back. No more stash dance. No more "I'll just clone it again." `git worktree` is a simple way to multitask in Git without breaking your flow.
