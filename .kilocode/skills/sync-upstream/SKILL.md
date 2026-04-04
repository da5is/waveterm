---
name: sync-upstream
description: Guide for syncing this fork with the official wavetermdev/waveterm upstream repository. Use when merging upstream changes, resolving fork conflicts, or bringing the fork up to date.
---

# Syncing Fork with Upstream (wavetermdev/waveterm)

This guide explains how to manually sync this fork with the official Wave Terminal repository. There is also an automated GitHub Actions workflow (`.github/workflows/sync-fork.yml`) that runs this daily, but manual syncs are needed when you want changes immediately or need to resolve conflicts interactively.

## Fork Overview

This is an unofficial fork (`da5is/waveterm`) of [wavetermdev/waveterm](https://github.com/wavetermdev/waveterm) that adds Windows ARM64 builds. The fork maintains a small set of CI/build commits on top of upstream's `main` branch.

### Fork-Specific Changes

These are the commits unique to this fork (all CI/build related):

- **Windows ARM64 build workflow** — `.github/workflows/build-win-arm64.yml`
- **Build-time fork patches** — artifact naming, auto-updater URL overrides
- **Removed workflows** — `deploy-docsite.yml` (no Pages auth), TestDriver workflows (no API key)
- **Upstream sync workflow** — `.github/workflows/sync-fork.yml`
- **README fork banner** — auto-updated by the sync workflow

### Known Conflict Points

When merging upstream, conflicts typically occur in:

| File | Reason | Resolution |
|------|--------|------------|
| `.github/workflows/deploy-docsite.yml` | Fork deletes it; upstream modifies it | Keep deletion (`git rm`) |
| `.github/workflows/testdriver-build.yml` | Fork deletes it; upstream modifies it | Keep deletion (`git rm`) |
| `.github/workflows/testdriver.yml` | Fork deletes it; upstream modifies it | Keep deletion (`git rm`) |
| `README.md` | Fork adds banner; upstream modifies content | Usually auto-merges; if not, keep both changes |

## Initial Setup (After a Fresh Clone)

After cloning this fork for the first time (e.g., on a new machine), you must add the upstream remote. This is a local git config and is **not** stored in the repository:

```powershell
cd C:\Users\da5is\src\forks\waveterm
git remote add upstream https://github.com/wavetermdev/waveterm.git
```

Verify with `git remote -v` — you should see both `origin` (da5is/waveterm) and `upstream` (wavetermdev/waveterm).

## Step-by-Step Guide

### Step 1: Ensure Upstream Remote Exists

```powershell
git remote -v
```

If `upstream` is not listed, run the setup step above.

### Step 2: Fetch Upstream

```powershell
git fetch upstream
```

### Step 3: Check What's Changed

Compare the fork and upstream to understand what will be merged:

```powershell
# Find the common ancestor
$mb = git merge-base main upstream/main
Write-Host "Merge base: $mb"

# See fork-only commits (your changes)
git --no-pager log --oneline "$mb..main"

# See upstream-only commits (what's new)
git --no-pager log --oneline "$mb..upstream/main"

# Check for overlapping files (potential conflicts)
$forkFiles = git diff --name-only "$mb..main"
$upstreamFiles = git diff --name-only "$mb..upstream/main"
$forkFiles | Where-Object { $upstreamFiles -contains $_ }
```

If `git merge-base --is-ancestor upstream/main HEAD` returns exit code 0, the fork is already up to date — no merge needed.

### Step 4: Merge Upstream

```powershell
git merge upstream/main --no-edit
```

If the merge completes cleanly, skip to Step 6.

### Step 5: Resolve Conflicts

For each conflicted file, check against the **Known Conflict Points** table above.

**For files the fork intentionally deleted:**

```powershell
git rm .github/workflows/deploy-docsite.yml
git rm .github/workflows/testdriver-build.yml
git rm .github/workflows/testdriver.yml
```

**For README.md conflicts** (if any):

The fork adds a banner between `<!-- FORK-BANNER-START -->` and `<!-- FORK-BANNER-END -->` markers. Keep the banner and accept upstream's content changes below it.

**For any other conflicts:**

These are unexpected — review manually. The fork should only have CI/build changes, so upstream content changes should generally be accepted.

After resolving all conflicts:

```powershell
git commit --no-edit
```

### Step 6: Pull from Origin (if needed)

If the automated sync workflow or other contributors have pushed to origin since your last pull:

```powershell
git pull origin main --no-edit
```

### Step 7: Push

```powershell
git push origin main
```

### Step 8: Verify

```powershell
# Confirm clean state
git --no-pager status

# Confirm upstream is now an ancestor of HEAD
git merge-base --is-ancestor upstream/main HEAD && Write-Host "Sync verified"
```

## Automated Sync Workflow

The GitHub Actions workflow at `.github/workflows/sync-fork.yml` automates this process:

- **Schedule:** Daily at 06:42 UTC
- **Trigger:** Also available via `workflow_dispatch` (manual run from GitHub UI)
- **What it does:**
  1. Fetches upstream/main
  2. Merges if there are new commits
  3. Patches README.md with an updated fork banner (version, upstream SHA, download links)
  4. Pushes to origin/main

If the automated workflow fails (e.g., due to a merge conflict), perform the manual steps above to resolve it, then the workflow will succeed on its next run.

## Troubleshooting

### Push Rejected

If `git push` is rejected because origin has newer commits (e.g., from the automated sync workflow):

```powershell
git pull origin main --no-edit
git push origin main
```

### Unexpected Conflicts in Source Files

If upstream changes conflict with fork changes in source files (not CI/workflows), it means the fork has diverged beyond CI/build changes. Review carefully:

1. Check if the fork change is still needed
2. If so, reapply it on top of the upstream version
3. If not, accept the upstream version

### Verifying Fork Patches Still Apply

After syncing, verify the build still works. The fork patches are applied at build time via the `build-win-arm64.yml` workflow, so check:

```powershell
# Ensure the ARM64 build workflow is intact
Test-Path .github/workflows/build-win-arm64.yml

# Ensure deleted workflows stay deleted
-not (Test-Path .github/workflows/deploy-docsite.yml)
-not (Test-Path .github/workflows/testdriver-build.yml)
-not (Test-Path .github/workflows/testdriver.yml)
```
