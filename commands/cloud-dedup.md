---
name: cloud-dedup
description: Scan multiple cloud storage accounts (Aliyun Pan, Baidu Netdisk) for duplicate and junk files, then batch delete with user confirmation.
user_invocable: true
---

# Cloud Storage Dedup & Cleanup

Scan multiple Aliyun Cloud Drive and Baidu Netdisk accounts for duplicate files, junk files, and unwanted formats, then batch delete with user confirmation.

## Prerequisites

- `aliyunpan` CLI installed and accounts logged in (`aliyunpan loglist` to verify)
- `BaiduPCS-Go` CLI installed and accounts logged in (`BaiduPCS-Go loglist` to verify)
- Python 3.9+

**Config fix**: If aliyunpan shows "config file permission denied", run:
```bash
chmod 600 /usr/local/Cellar/aliyunpan/0.3.8/bin/aliyunpan_config.json
```

## Workflow

### Phase 1: Scan all accounts

For each cloud account, export a full file tree to `$TMPDIR/cloud-dedup/`:

**Aliyun accounts:**
```bash
aliyunpan su <UID>         # switch account
aliyunpan tree / > $TMPDIR/cloud-dedup/tree_ali_<nickname>.txt
```

**Baidu accounts:**
```bash
BaiduPCS-Go su <UID>       # switch account
BaiduPCS-Go tree / > $TMPDIR/cloud-dedup/tree_baidu_<nickname>.txt
```

Store account metadata (UID, nickname, CLI tool) in `$TMPDIR/cloud-dedup/accounts.json`.

### Phase 2: Parse trees and detect duplicates

Write a Python script (`$TMPDIR/cloud-dedup/find_duplicates.py`) that:

1. **Parses tree output** into `(full_path, filename)` tuples per account. Tree format:
   ```
   ├── dirname/
   │   ├── file.txt
   │   └── subdir/
   │       └── nested.txt
   └── another.txt
   ```
   Use regex `r'[├└]── (.+)'` to extract names. Track directory depth via character position (`pos // 4 + 1`). Maintain a `path_stack` for full path reconstruction.

2. **Group files by filename** across all accounts. Skip common false positives:
   - Generic names: `cover.jpg`, `folder.jpg`, `Thumbs.db`, `.DS_Store`, `desktop.ini`
   - Numbered tracks: files matching `^\d{1,3}\.(flac|mp3|wav|ape|m4a)$`
   - Mac metadata: files starting with `._`

3. **Classify duplicates**:
   - **Within-account**: same filename appears in multiple directories of one account
   - **Cross-account**: same filename appears in 2+ accounts
   - **By category**: books (azw3/epub/mobi/pdf), music (flac/mp3/ape/dsf/wav), photos (jpg/jpeg/png/heic), videos (mp4/mkv/avi/flv), other

4. **Present findings** to user as a summary table with counts per category and account.

### Phase 3: Determine keep/delete strategy

For each duplicate group, apply these rules (confirm with user):

- **Books**: Keep the copy in the account with the most format variants (azw3 > epub > mobi > pdf). If tied, keep in the account with the larger book collection.
- **Music**: Keep the higher quality format. For exact duplicates, keep in the primary music account.
- **Photos**: Keep in the primary photos account. Flag personal photos (camera roll) for manual review.
- **Cross-account**: If one account has a superset, delete from the account with fewer files.
- **Always ask** before deleting personal content (photos, videos named with dates/camera names).

### Phase 4: Junk file cleanup

Scan for and offer to delete:

| Type | Pattern | Description |
|------|---------|-------------|
| Mac metadata | `._*` | macOS resource fork files, completely useless on cloud |
| Thumbnails | `Thumbs.db` | Windows thumbnail cache |
| Desktop config | `desktop.ini` | Windows folder config |
| DS_Store | `.DS_Store` | macOS folder metadata |
| Unwanted formats | `*.dsf` (or user-specified) | Formats user wants removed |

### Phase 5: Execute deletions

Generate JSON delete lists at `$TMPDIR/cloud-dedup/delete_<account>_<category>.json`.

**Aliyun Pan deletion:**
```python
# Wildcard per directory (fastest for bulk ._* cleanup):
aliyunpan rm "<directory>/._*"

# Batch individual files (up to 20 per command):
aliyunpan rm "<path1>" "<path2>" ... "<path20>"

# IMPORTANT: aliyunpan does NOT support -f flag
# Deleted files go to cloud recycle bin (recoverable)
```

**Baidu Netdisk deletion:**
```python
# Individual or small batches:
BaiduPCS-Go rm "<path1>" "<path2>" ... "<path10>"

# Switch accounts:
BaiduPCS-Go su <UID>
```

**Unicode handling** (critical for Japanese/Chinese filenames):
```python
import unicodedata
# If delete fails, retry with NFD normalization:
nfd_path = unicodedata.normalize('NFD', path)
if nfd_path != path:
    # retry with nfd_path
```

**Error handling:**
- Batch fails -> fall back to individual file deletion
- Individual fails -> try NFD normalization
- Still fails -> log and skip (file may already be deleted or path changed)
- Network EOF -> retry after 2-second wait

### Phase 6: Report

After deletion, print summary table:

```
| Account | Category | Attempted | Succeeded | Failed |
|---------|----------|-----------|-----------|--------|
| ...     | ...      | ...       | ...       | ...    |
```

## Key Lessons & Gotchas

1. **aliyunpan `tree /`** is far faster than recursive `ll` for large accounts (90K+ files).
2. **aliyunpan has no `-f` flag** for `rm`. Don't use it.
3. **aliyunpan `su`** requires UID or nickname, not numeric index.
4. **BaiduPCS-Go `su`** accepts UID directly: `BaiduPCS-Go su <uid>`.
5. **BaiduPCS-Go search** may not find hidden files (Thumbs.db) that appear in tree output.
6. **Token expiry**: aliyunpan tokens can expire mid-session. Check with `aliyunpan who` before large operations. If expired, user must `aliyunpan login` again.
7. **Tree snapshots are point-in-time**: Files may have been deleted since scan. Treat "file not found" on delete as success.
8. **Special characters in filenames**: Unicode quotes (`\u201c` `\u201d`), fullwidth characters, and NFC/NFD normalization differences can cause delete failures.
9. **Wildcard `._*`** in aliyunpan only matches current directory, not recursive.
10. **Long batch commands** with Chinese filenames can fail silently. Keep batch size <= 20 for aliyunpan, <= 10 for BaiduPCS-Go.
