---
title: "Caching Xcode builds on GitHub Actions"
date: 2026-02-17
draft: false
tags: [ios, xcode, ci, dev]
---

You add a few SPM dependencies, push to GitHub, and watch your CI time double. The obvious fix — cache `.build/` with `actions/cache` — doesn't help. Xcode recompiles everything anyway.

This article explains why, and presents two solutions with different trade-offs so you can pick the one that fits your project.

## The problem

Adding SPM dependencies is painless locally. Xcode builds them once, and subsequent builds only recompile what changed. But on CI, every push triggers a full rebuild of every dependency — even if you haven't touched `Package.resolved`.

The naive response is to cache `.build/` (or `DerivedData/`):

```yaml
- name: Cache build data
  uses: actions/cache@v5
  with:
    path: swift/.build
    key: spm-${{ runner.os }}-${{ hashFiles('swift/Package.resolved') }}
    restore-keys: spm-${{ runner.os }}-
```

The cache restores successfully — the files are bit-for-bit identical — yet xcodebuild recompiles every module from scratch. The build system acts as if everything changed.

Even the [official `actions/cache` examples for Swift](https://github.com/actions/cache/blob/main/examples.md#swift---spm) recommend caching `.build` without any further steps, and the same problem appears.

## How xcodebuild detects changes

Some build systems use content hashing to detect changes. Gradle and Bazel both work this way — robust, but with the overhead of reading and hashing every input file. xcodebuild takes a cheaper approach inherited from `make`: llbuild (the engine behind both xcodebuild and SwiftPM) calls `stat()` on each file and compares two filesystem metadata fields:

- **mtime** — last modification timestamp
- **inode** — the filesystem identifier for the file's data on disk

If either differs from what was recorded last time, the file counts as changed and everything that depends on it gets rebuilt. This is fast and works fine on a single machine where files change in place: editing updates the mtime, and the inode stays the same because the filesystem reuses the same data block. The trade-off is that it's sensitive to the filesystem environment — which is exactly what breaks on CI.

## Why CI breaks this

Two things happen on a CI runner that violate llbuild's assumptions.

### `actions/checkout` resets all mtimes

When `actions/checkout` clones the repo, git doesn't preserve modification times. Every file gets the timestamp of the checkout — typically "now". Even if you haven't touched `Sources/MyFile.swift` in months, the build system sees today's timestamp and concludes the file is new.

### `actions/cache` doesn't preserve inodes

`actions/cache` uses tar under the hood: it tars the specified paths on save, and extracts them on restore. Tar preserves contents, mtimes, and permissions — but extraction creates new files with new inodes. So the restored `.o` file is identical in content and mtime, but has a different inode. To llbuild, that's enough to count as changed.

The result: even with a perfect cache hit, either the source files have different mtimes or the cached build products have different inodes (or both), and xcodebuild rebuilds everything.

## Solution 1: cache only source package checkouts

The simplest approach is to cache only `SourcePackages` — the cloned dependency source code — and skip build products entirely:

```yaml
- name: Cache SPM checkouts
  uses: actions/cache@v5
  with:
    path: swift/.build/SourcePackages
    key: spm-${{ runner.os }}-${{ hashFiles('swift/Package.resolved') }}
    restore-keys: spm-${{ runner.os }}-

- name: Run tests
  run: |
    set -o pipefail && xcodebuild test \
      -scheme MyApp \
      -destination 'platform=macOS,arch=arm64' \
      -derivedDataPath .build/ | xcbeautify
```

Since you're not caching any build products, there are no stale `.o` files with wrong inodes. The build system starts fresh every time, which is what it expects. The cache only saves the time spent cloning packages from their git remotes.

This is the simplest possible setup — no mtime tricks, no extra build settings, no risk of stale build artifacts. The downside is that all dependency modules and your own code recompile every run, so the time savings are limited to skipping `git clone`. Best for small projects with few or lightweight dependencies, or projects where build time isn't a real bottleneck.

## Solution 2: full derived data cache with mtime restoration

For true incremental builds on CI, you need to fix both problems: restore correct mtimes on source files, and tell xcodebuild to ignore inode changes on build products.

```yaml
- name: Checkout
  uses: actions/checkout@v4
  with:
    fetch-depth: 0  # Full history needed for mtime restoration

- name: Restore file mtimes
  uses: chetan/git-restore-mtime-action@v2

- name: Cache SPM
  uses: actions/cache@v5
  with:
    path: |
      swift/.build/Build
      swift/.build/SourcePackages
    key: xcode-${{ runner.os }}-${{ hashFiles('swift/Package.resolved') }}-${{ github.sha }}
    restore-keys: |
      xcode-${{ runner.os }}-${{ hashFiles('swift/Package.resolved') }}-

- name: Run tests
  run: |
    set -o pipefail && xcodebuild test \
      -scheme MyApp \
      -destination 'platform=macOS,arch=arm64' \
      -derivedDataPath .build/ \
      IgnoreFileSystemDeviceInodeChanges=YES | xcbeautify
```

Each piece does a specific job.

### git-restore-mtime

[`chetan/git-restore-mtime-action`](https://github.com/chetan/git-restore-mtime-action) walks the git log and sets each file's mtime to the timestamp of the last commit that modified it. After this step, `Sources/MyFile.swift` has the same mtime it had on the machine that committed it — not the checkout time.

This requires `fetch-depth: 0` (a full clone) so the action has access to the complete git history. The checkout itself is slower, but the time saved on incremental builds more than makes up for it on projects with heavy dependencies.

### IgnoreFileSystemDeviceInodeChanges

This is an xcodebuild build setting (not a command-line flag — note the `=YES` syntax without a dash) that tells the build system to skip inode comparison when deciding whether a file has changed. With this setting, xcodebuild only checks mtimes.

Combined with mtime restoration, this means: if a cached build product was created from a source file, and that source file still has the same mtime it had when the product was built, xcodebuild correctly concludes nothing changed and skips the recompilation.

You can also set it globally via macOS defaults:

```bash
defaults write com.apple.dt.XCBuild IgnoreFileSystemDeviceInodeChanges -bool YES
```

This applies to all xcodebuild invocations on the machine, so you don't have to pass it on every command. On a CI runner that's fine — the machine is ephemeral. On a local machine, you probably don't want it, since inode tracking is correct and useful there.

### Cache key strategy

The key uses `Package.resolved` hash + `github.sha`, with a single restore-key fallback:

```yaml
key: xcode-${{ runner.os }}-${{ hashFiles('swift/Package.resolved') }}-${{ github.sha }}
restore-keys: |
  xcode-${{ runner.os }}-${{ hashFiles('swift/Package.resolved') }}-
```

- **Exact match** (`Package.resolved` hash + commit SHA): hits when the same commit was built before — e.g. a re-run of a failed job. Everything is up to date.
- **Partial match** (`Package.resolved` hash only): hits when dependencies haven't changed but your code has. Build products for dependencies are reused; only your code recompiles.

You might see examples with a third tier — an OS-only fallback like `xcode-${{ runner.os }}-` — that provides a warm cache even when dependencies change. But removed dependencies then leave ghost artifacts (stale build products and source checkouts) that waste cache space. Dropping the OS-only fallback means a `Package.resolved` change triggers a clean build with no ghosts, at the cost of one full rebuild when you add or remove a dependency. For most projects that's the better trade-off — dependency changes are rare.

#### Why github.sha is necessary (and its cost)

`actions/cache` is **write-once**: if a key already exists, the cache is restored but never overwritten. Without `github.sha`, the key would match on `Package.resolved` alone — the cache would be written on the first run and frozen forever. With each subsequent commit, more source files have newer mtimes than what the cached build products expect, and xcodebuild rebuilds them. After a few commits the cache is effectively useless: you're paying the cost of restoring hundreds of megabytes of stale artifacts only to rebuild everything anyway. At that point you're worse off than Solution 1.

`github.sha` fixes this by making the key unique per commit. Every successful build saves fresh artifacts, so the next run restores build products that are only one commit behind.

The trade-off is cache storage. Every commit creates a new cache entry, and GitHub enforces a 10 GB per-repository limit. Once you hit it, the oldest entries are evicted — potentially the one a long-lived PR branch needed. Entries not accessed for 7 days are also evicted automatically.

Note that with `github.sha`, you never get an exact cache hit on a *new* commit. Every run falls back to `restore-keys`, restores the closest partial match, rebuilds the delta, and saves a new entry. The exact match only helps when re-running the same commit.

This setup gives you true incremental builds and dramatically faster CI on projects with heavy dependencies. The cost is the extra moving parts (mtime action + build setting + cache paths), the slower `fetch-depth: 0` checkout, and the ghost artifacts that accumulate when you delete a source file (they don't break anything, just waste space until purged). Best for projects with multiple or heavy SPM dependencies where build time is a real concern.

## Purging the cache

Ghost artifacts from deleted source files, plus the write-once-per-commit pattern, mean the cache only grows. Entries not accessed for 7 days are evicted automatically, which naturally limits accumulation during active development. But if you need to reset sooner — the cache is bloated, or you suspect corrupted artifacts — you can purge it with the [`gh` CLI](https://cli.github.com/manual/gh_cache_delete):

```bash
gh cache delete --all -R owner/repo
```

For a hands-off approach, add a workflow that purges on a schedule and on demand:

```yaml
name: Purge CI Cache

on:
  workflow_dispatch:
  schedule:
    - cron: '0 0 * * 1'  # Mondays at midnight UTC

permissions:
  actions: write

jobs:
  purge:
    runs-on: ubuntu-latest
    steps:
      - name: Purge build caches
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: gh cache delete --all -R ${{ github.repository }}
```

The `actions: write` permission is required for cache deletion — without it the `GITHUB_TOKEN` won't have access. The schedule keeps ghost artifacts from accumulating, while `workflow_dispatch` lets you purge manually when needed.

## SwiftPM vs xcodebuild

If you use `swift build` / `swift test` instead of xcodebuild, the inode problem doesn't apply. SwiftPM uses a "device-agnostic" mode for file status tracking — it ignores device and inode numbers and relies on file size and mtime.

Both tools still use mtime, so `git-restore-mtime` is beneficial for either one. But only xcodebuild needs `IgnoreFileSystemDeviceInodeChanges=YES`. This is why a documentation workflow using `swift package generate-documentation` can get by with a simpler setup — just mtime restoration and caching `.build/`, no special build settings.

## Prebuilt binary dependencies

A third approach worth mentioning: distributing dependencies as prebuilt XCFrameworks instead of source packages. If a dependency ships a binary target, or you build one yourself, there's nothing to compile. But this comes with its own trade-offs (architecture management, trust, debugging difficulty) and is a separate topic.

## Notes and tips

**First run is always a cache miss.** No way around it. The first build on a new runner, new branch, or after a `Package.resolved` change is a full build. The savings come from the second run onward.

**Failed jobs don't save cache.** By default, `actions/cache` only saves on successful job completion. If your build fails, the cache isn't updated. This is usually what you want — you don't want to cache broken build artifacts. But it means a failing test doesn't contribute to warming the cache for the next run.

**Selective path caching.** You don't have to cache all of `DerivedData`. For xcodebuild, build products live under `.build/Build` and source packages under `.build/SourcePackages`. Caching just these two paths avoids storing logs, test results, and other ephemeral data — and helps stay within GitHub's 10 GB per-repository cache limit.

**`fetch-depth: 0` cost.** A full clone can add 5–30 seconds depending on repository size. For repos with very large histories, weigh the trade-off against the mtime restoration savings. For most projects, it's worth it.

**Cache key ordering matters.** `restore-keys` are tried in order. Put the most specific prefix first for the best partial match.

## See also

- [`chetan/git-restore-mtime-action`](https://github.com/chetan/git-restore-mtime-action) — restores file mtimes from git history
- [`irgaly/xcode-cache`](https://github.com/irgaly/xcode-cache) — a GitHub Action that wraps mtime restoration, `IgnoreFileSystemDeviceInodeChanges`, and cache management into a single step
- [`actions/cache`](https://github.com/actions/cache) — GitHub's official caching action
- [Managing caches](https://docs.github.com/en/actions/how-tos/manage-workflow-runs/manage-caches) — GitHub docs on cache management, including force-deleting entries
