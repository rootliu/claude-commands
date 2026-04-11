---
name: disk-cleanup
description: Scan macOS disk for caches, duplicate files, and unused apps, then interactively clean with user confirmation.
user_invocable: true
---

# macOS Disk Cleanup

Scan a macOS machine for reclaimable disk space: caches, duplicate files, video app offline data, unused applications, and stale downloads. Present findings categorized by risk level, then clean interactively with user confirmation on each item.

## Usage

`/disk-cleanup` — full scan and interactive cleanup

## Workflow

### Phase 1: Disk Overview

Get a high-level picture of disk usage:

```bash
# Overall disk status
df -h /

# Top-level user directory breakdown
du -sh ~/*/  2>/dev/null | sort -rh | head -20

# Library breakdown (usually the biggest consumer)
du -sh ~/Library/*/  2>/dev/null | sort -rh | head -20

# Containers (app sandboxes — often huge)
du -sh ~/Library/Containers/*/ 2>/dev/null | sort -rh | head -15

# Application Support
find ~/Library/Application\ Support -maxdepth 1 -type d -exec du -sh {} \; 2>/dev/null | sort -rh | head -15

# Caches
du -sh ~/Library/Caches/*/ 2>/dev/null | sort -rh | head -15

# /Applications sizes
du -sh /Applications/*/ 2>/dev/null | sort -rh | head -20
```

Present a summary table with total disk, used, available, and top consumers.

### Phase 2: Identify Cleanup Candidates

Scan these categories and report sizes:

#### Category A: Safe Caches (auto-rebuild, no data loss)

| Item | Typical Location | Clean Command |
|------|-----------------|---------------|
| npm cache | `~/.npm/_cacache` | `npm cache clean --force` |
| pip cache | (via pip) | `pip3 cache purge` |
| Homebrew cache | `~/Library/Caches/Homebrew/` | `brew cleanup --prune=all` |
| Playwright browsers | `~/Library/Caches/ms-playwright/` | `npx playwright uninstall` or `rm -rf` |
| Chrome update cache | `~/Library/Caches/com.google.antigravity.ShipIt/` | `rm -rf` |
| Chrome browser cache | `~/Library/Caches/Google/` | Delete files: `find ... -type f -delete` (Chrome may lock dirs) |
| Codex cache | `~/Library/Caches/com.openai.codex/` | `rm -rf` |
| draw.io updater | `~/Library/Caches/draw.io-updater/` | `rm -rf` |
| AWS cache | `~/Library/Caches/aws/` | `rm -rf` |
| Lark/Feishu cache | `~/Library/Caches/LarkInternational/` | `rm -rf` |
| Python apple cache | `~/Library/Caches/com.apple.python/` | `rm -rf` |

**npm cache gotcha**: If `npm cache clean --force` fails with "root-owned files", user must run:
```bash
sudo chown -R $(id -u):$(id -g) ~/.npm
npm cache clean --force
```
If sudo is unavailable, fall back to `rm -rf ~/.npm/_cacache`.

#### Category B: Video/Media App Caches

These apps store offline video, chat files, and browsing caches in Containers or Application Support:

| App | Container/Path | What's inside |
|-----|---------------|---------------|
| Tencent Video | `~/Library/Containers/com.tencent.tenvideo/` | Offline videos, playback cache |
| Youku | `~/Library/Containers/com.youku.mac/` | Offline videos, playback cache |
| iQiyi | `~/Library/Containers/com.iqiyi.player/` | Offline videos, playback cache |
| bilibili | `~/Library/Application Support/bilibili/Cache/` | Video cache, code cache |
| Feishu/Lark | `~/Library/Containers/com.bytedance.macos.feishu/` | Chat files, docs cache |
| WeChat | `~/Library/Containers/com.tencent.xinWeChat/` | Chat history, files — **ask before cleaning** |
| Slack | `~/Library/Containers/com.tinyspeck.slackmacgap/` | Workspace cache |

**Cleaning strategy for Container apps:**
```bash
# Delete the Caches subdirectory only (preserves login/settings):
rm -rf ~/Library/Containers/<bundle-id>/Data/Library/Caches

# For Application Support apps, delete Cache dirs:
rm -rf "~/Library/Application Support/<app>/Cache"
rm -rf "~/Library/Application Support/<app>/Code Cache"
rm -rf "~/Library/Application Support/<app>/GPUCache"
```

**Warning**: WeChat Container contains chat history. Always ask before touching.

#### Category C: Stale App Data & Updater Bloat

| Item | Path | Description |
|------|------|-------------|
| ClassIn EeoUpdater | `~/Library/Application Support/ClassIn/EeoUpdater*` | Old installer packages, ClassIn itself unaffected |
| Quark cloud cache | `~/Library/Application Support/Quark/` | Local file cache for Quark cloud drive |
| aDrive cache | `~/Library/Application Support/aDrive/` | Aliyun Drive local cache |
| Baidu Netdisk cache | `~/Library/Application Support/com.baidu.BaiduNetdisk-mac/` | Local cache |
| Movies/EasyDataSource | `~/Movies/EasyDataSource/` | Mango TV (.mgcf encrypted offline video cache) |
| Movies/Videos | `~/Movies/Videos/` | Old video downloads |
| Trae IDE data | `~/Library/Application Support/Trae/` | If IDE not in use |

#### Category D: Unused Applications

Check `/Applications/` for large apps that may not be needed. Common candidates:

| App | Typical Size | Notes |
|-----|-------------|-------|
| iMovie.app | ~3.7GB | Apple built-in, free to reinstall from App Store |
| GarageBand.app | ~1.1GB | Apple built-in, free to reinstall |
| Python Editor.app | ~374MB | Redundant if VS Code / other IDE installed |
| Scratch 3.app | ~301MB | Scratch programming tool |

**Important**: Apps in `/Applications/` often require `sudo rm -rf` to delete. Cannot be done programmatically without user entering password. Generate the command and ask user to execute it.

#### Category E: Stale Home Directory Files

Check for orphan files in `~/`:
```bash
ls -lhS ~/*.py ~/*.txt ~/*.jpg ~/*.png ~/DS_Store 2>/dev/null
```

### Phase 3: Interactive Cleanup

**Ask user one-by-one for each item using `AskUserQuestion`.**

Order by category:
1. **Category A** (safe caches) — batch all into one confirmation since they're all safe
2. **Category B** (video caches) — one app at a time
3. **Category C** (stale data) — one item at a time
4. **Category D** (app uninstall) — one app at a time, note if `sudo` needed
5. **Category E** (stale files) — batch or one at a time

For each item, present:
- Name and size
- What it is
- Risk level (safe / needs re-login / permanent)

### Phase 4: Execute Deletions

For each confirmed item:

```bash
# Caches and app data — direct rm
rm -rf "<path>"

# /Applications apps — needs sudo, ask user to run:
# sudo rm -rf /Applications/<app>.app

# npm/pip/brew — use their built-in clean commands first
```

**Error handling:**
- "Permission denied" on `rm -rf` → likely needs `dangerouslyDisableSandbox: true` or `sudo`
- "Operation not permitted" on sandbox → retry with sandbox disabled
- npm "root-owned files" → needs `sudo chown` first
- Chrome cache "Directory not empty" → Chrome is running, use `find -type f -delete` instead
- `/Applications/*.app` permission denied → must use `sudo`, tell user to run the command manually

### Phase 5: Report

After all operations, show:

```bash
df -h /
```

Summary table:

```
| Category       | Item              | Size    | Status |
|----------------|-------------------|---------|--------|
| Cache          | npm               | 3.1GB   | Done   |
| Video cache    | Tencent Video     | 2.5GB   | Done   |
| App uninstall  | iMovie            | 3.7GB   | Done   |
| ...            | ...               | ...     | ...    |
|                | **Total freed**   | **XXG** |        |
```

Show before/after: `X.XGB → Y.YGB available (ZZ% → WW% used)`

## Key Lessons & Gotchas

1. **`~/Library/Containers/`** is usually the single biggest space consumer on macOS. Each sandboxed app gets its own container.
2. **npm cache** frequently has root-owned files from old npm bugs. Always try `npm cache clean --force` first, fall back to `sudo chown` + retry, then `rm -rf` as last resort.
3. **Chrome cache directories** can't be fully removed while Chrome is running. Use `find -type f -delete` to clear contents.
4. **Apple built-in apps** (iMovie, GarageBand, Keynote, Pages, Numbers) are free to reinstall from App Store but need `sudo` to delete.
5. **WeChat container** contains irreplaceable chat history. Never delete without explicit user confirmation and backup reminder.
6. **Video app caches** (Tencent Video, Youku, iQiyi) store encrypted offline content that is useless once cleared. Users will need to re-login.
7. **Homebrew `brew cleanup --prune=all`** also removes old Ruby versions bundled with Homebrew, which can free significant space.
8. **`.mgcf` files** in Movies/EasyDataSource are Mango TV encrypted video segments — cannot be played standalone, safe to delete.
9. **Sandbox restrictions** in Claude Code: `~/Documents`, `~/Library`, `/Applications` writes need `dangerouslyDisableSandbox: true`. If that fails with EACCES, the operation needs `sudo`.
10. **Always run `df -h /` before and after** to give the user a clear picture of impact.
