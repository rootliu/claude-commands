---
argument-hint: [sync|discover|full|repair|merge] [--topics=reasoning,agentic,world-model,...] [--year=2026]
description: Sync missing Zotero PDFs from arXiv, discover new papers by topic, repair DB integrity, merge duplicate items, and import with user confirmation
---

# Zotero Paper Sync & Discovery

Manage a Zotero library: sync missing PDFs, discover new papers from arXiv/blogs/KOL, import selected papers with user confirmation, and merge duplicate entries safely.

## Modes

- `sync` (default): Scan Zotero DB for missing PDFs, download from arXiv, link to Zotero
- `discover`: Search arXiv + blogs for new papers matching research interests, ask user one-by-one, import selected
- `full`: Run sync first, then discover
- `repair`: Scan and fix DB integrity issues (invalid keys, wrong field IDs, orphan records, dangling references)
- `merge`: Detect and merge duplicate items (same arXiv ID / DOI / title), preserving PDFs, annotations, and user comments

Usage: `/sync-zotero`, `/sync-zotero sync`, `/sync-zotero discover --topics=reasoning,agentic`, `/sync-zotero full`, `/sync-zotero repair`, `/sync-zotero merge`

---

## Environment Setup

### Paths (adapt per machine)

Detect the current machine's Zotero installation:

```python
import os, glob

# Common Zotero data locations
candidates = [
    os.path.expanduser("~/Zotero"),                          # Windows/Linux default
    os.path.expanduser("~/Library/Application Support/Zotero"), # macOS
    os.path.expandvars("%APPDATA%/Zotero/Zotero"),           # Windows alt
]
ZOTERO_DIR = next((d for d in candidates if os.path.isfile(os.path.join(d, "zotero.sqlite"))), None)
```

Required paths:
- `ZOTERO_DB`: `{ZOTERO_DIR}/zotero.sqlite`
- `STORAGE`: `{ZOTERO_DIR}/storage/`
- `ACADEMY_DIR`: `~/Documents/academy/` (create if missing)

### Platform Notes

- **Windows (Python 3.14)**: SSL broken in urllib; always use `curl.exe` with `--ssl-no-revoke` for HTTPS downloads
- **All platforms**: Always `sys.stdout.reconfigure(encoding='utf-8')` in Python scripts
- **Real-time output**: Use `print = functools.partial(print, flush=True)` for background/long-running scripts
- **Zotero locking**: Copy DB for reads when Zotero is running; close Zotero for writes
- **Close Zotero**: On Windows use `Stop-Process -Name zotero -Force`; then wait 3 seconds before writing
- Check if Zotero is running: `curl -s http://127.0.0.1:23119/connector/ping` or check process list

---

## Mode 1: Sync Missing PDFs

### Step 1 — Scan Zotero DB

Copy the DB if Zotero is running, then query for all PDF attachments:

```sql
SELECT ia.itemID, ia.parentItemID, ia.syncState, ia.path, i.key
FROM itemAttachments ia
JOIN items i ON ia.itemID = i.itemID
WHERE ia.contentType = 'application/pdf'
  AND ia.parentItemID IS NOT NULL
```

For each attachment, check if file exists at `{STORAGE}/{key}/{filename}`. Classify:
- **OK**: File exists and size > 0
- **Missing (syncState=1)**: Zotero knows file should exist but it hasn't been downloaded
- **Missing (no file)**: Record exists but physical file absent

### Step 2 — Extract arXiv IDs

**IMPORTANT**: Search ALL metadata fields for arXiv IDs, not just specific ones. Different Zotero clients and plugins store arXiv info in different fields. Query all fields for each parent item:

```sql
SELECT id.fieldID, idv.value
FROM itemData id
JOIN itemDataValues idv ON id.valueID = idv.valueID
WHERE id.itemID = ?
```

Then scan ALL values with regex: `(\d{4}\.\d{4,5})`

Validate extracted IDs: the year prefix (first 2 digits) must be reasonable (15-27). Skip IDs like `2023.10004` or `1070.2023` which are DOI fragments, not arXiv IDs.

Known fields where arXiv IDs appear:
- `fieldID=19` (extra): `arXiv:XXXX.XXXXX`
- `fieldID=10` (url): `arxiv.org/abs/XXXX.XXXXX`
- `fieldID=107` (archiveID): `arXiv:XXXX.XXXXX`
- `fieldID=13` (archiveLocation): `http://arxiv.org/abs/XXXX.XXXXX` (used by some Zotero plugins)
- `fieldID=8` (DOI): `10.48550/arXiv.XXXX.XXXXX`

Prefer IDs from fields that contain `arxiv` in the value (more reliable than bare DOI matches).

### Step 3 — Check Latest Version on arXiv

Fetch `https://arxiv.org/abs/{id}` and parse:
```python
versions = re.findall(r'\[v(\d+)\]', html)
latest_v = max(int(v) for v in versions) if versions else 1
```

### Step 4 — Download PDFs

Use `curl.exe` (or `curl` on Linux/macOS):
```bash
# Windows: use --ssl-no-revoke to avoid CRYPT_E_REVOCATION_OFFLINE
curl.exe --ssl-no-revoke -L -o "{ACADEMY_DIR}/{arxivID}v{N} {Title}.pdf" "https://arxiv.org/pdf/{arxivID}v{N}"
```

Filename convention: `{arxivID}v{version} {Title}.pdf`
- Replace `:` with ` -` and remove `?` `"` from title
- Truncate title at 120 chars if needed

### Step 5 — Link to Zotero

**If Zotero is closed** (preferred — check connector ping fails):

For **existing attachments** (syncState=1, record exists but no file):
1. Copy PDF to `{STORAGE}/{att_key}/{expected_filename}`
2. `UPDATE itemAttachments SET syncState = 0 WHERE itemID = ?`

For **items with no attachment record** (parent exists, no PDF child):
1. Generate unique 8-char key (**CRITICAL: use Zotero charset**, see Key Generation below)
2. Create attachment item (itemTypeID=3, version=0, synced=0)
3. Create itemAttachments record (linkMode=0, contentType='application/pdf', path='storage:{filename}', syncState=0)
4. Set title field (fieldID=1) on attachment
5. Copy PDF to `{STORAGE}/{new_key}/{filename}`

**If Zotero is running**: Ask user to close Zotero, or write a `fix_zotero.py` script for later execution.

### Step 6 — Report & Clean Up

Report: X PDFs synced, Y still missing (no arXiv ID), Z errors.
Remove temp DB copy.

---

## Mode 2: Discover New Papers

### Step 1 — Extract Research Profile from Zotero

Scan all items in Zotero DB and extract:
- All arXiv IDs (for deduplication)
- All titles (for topic analysis)
- Count by year prefix to understand collection scope

```sql
SELECT i.itemID, idv_t.value AS title, idv_e.value AS extra, idv_u.value AS url
FROM items i
LEFT JOIN itemData id_t ON i.itemID = id_t.itemID AND id_t.fieldID = 1
LEFT JOIN itemDataValues idv_t ON id_t.valueID = idv_t.valueID
LEFT JOIN itemData id_e ON i.itemID = id_e.itemID AND id_e.fieldID = 19
LEFT JOIN itemDataValues idv_e ON id_e.valueID = idv_e.valueID
LEFT JOIN itemData id_u ON i.itemID = id_u.itemID AND id_u.fieldID = 10
LEFT JOIN itemDataValues idv_u ON id_u.valueID = idv_u.valueID
WHERE i.itemTypeID NOT IN (3, 14) AND i.itemID NOT IN (SELECT itemID FROM deletedItems)
```

### Step 2 — Search arXiv

Use parallel Task subagents to search multiple topics simultaneously.

**arXiv search URL pattern**: `https://arxiv.org/search/?query={query}&searchtype=all&start=0`

Topic queries to construct (based on `--topics` or default from library analysis):
- `{topic}+survey+{year}`
- `{topic}+LLM+{year}`
- Related terms for each topic

Parse results:
```python
papers = re.findall(
    r'<p class="list-title[^"]*"><a href="https://arxiv.org/abs/([\d.]+)".*?</a>.*?<p class="title[^"]*">\s*(.+?)\s*</p>',
    html, re.DOTALL
)
```

Filter: only IDs starting with `{year_prefix}` (e.g., "26" for 2026), not in existing library.

### Step 3 — Search Blogs/News/KOL (via Task subagents)

Launch parallel Task agents to search:

**Recommended sources** (use WebFetch):
- `sebastianraschka.com` -- ML research blog
- `www.interconnects.ai` -- Nathan Lambert's AI blog
- `simonwillison.net` -- LLM practitioner blog
- `karpathy.github.io` -- Karpathy's blog
- `thezvi.substack.com` -- AI analysis
- `www.latent.space` -- AI engineering
- `the-decoder.com` -- AI news
- `www.marktechpost.com` -- AI research news
- `newsletter.maartengrootendorst.com` -- NLP newsletter
- `news.ycombinator.com` -- HN (search AI/ML papers)
- `www.reddit.com/r/MachineLearning` -- Top posts
- `paperswithcode.com/greatest` -- Trending papers

Each agent should:
1. Fetch the source
2. Extract any arXiv paper references (pattern: `\d{4}\.\d{4,5}`)
3. Return list of `arxiv_id | title | description`

### Step 4 — Verify Candidates on arXiv

Deduplicate all found papers. Remove those already in library.

Verify each candidate exists on arXiv:
```python
url = f'https://arxiv.org/abs/{arxiv_id}'
html = curl_fetch(url)
title = re.search(r'<meta name="citation_title" content="([^"]+)"', html)
```

Drop any that return 404, wrong topic (hallucinated by search agents), or are domain-specific applications not matching research interests.

### Step 5 — Curate and Filter

Prioritize:
1. **Surveys and technical reports** (broad impact)
2. **Papers from known authors/labs** (Google, Meta, OpenAI, etc.)
3. **Papers with high relevance** to user's existing research themes
4. **Foundational papers** over narrow application papers

Deprioritize/skip:
- Domain-specific applications (materials science, medical, agriculture etc. unless user has those papers)
- Workshop/competition papers (SemEval tasks, shared tasks)
- Narrow benchmarks without methodological contribution

### Step 6 — Ask User One-by-One

Use `AskUserQuestion` tool for each paper:

```
{arxiv_id} - {title}
{authors}

{one-line Chinese description of why it's relevant}
```

Options: "下载" / "跳过"

Collect all "下载" selections.

### Step 7 — Batch Download and Import

For all selected papers:
1. Fetch metadata from arXiv (title, version, date)
2. Download PDFs via curl to `ACADEMY_DIR`
3. Check if Zotero is running (ping connector)
4. If closed: write directly to DB
5. If running: ask user to close Zotero first

**Creating Zotero items** (when writing directly to DB):

```python
import datetime

now = datetime.datetime.now(datetime.timezone.utc).strftime('%Y-%m-%d %H:%M:%S')

# 1. Create parent preprint item (version=0, synced=0 for new local items)
c.execute('''INSERT INTO items (itemTypeID, dateAdded, dateModified, clientDateModified,
             libraryID, key, version, synced) VALUES (31, ?, ?, ?, 1, ?, 0, 0)''',
             (now, now, now, gen_key(c)))
parent_id = c.lastrowid

# 2. Add metadata fields -- USE CORRECT FIELD IDs!
fields = {
    1:   title,                             # title
    6:   date,                              # date (e.g. "2026-03-15")
    8:   f'10.48550/arXiv.{arxiv_id}',      # DOI
    10:  f'http://arxiv.org/abs/{arxiv_id}', # url
    16:  'arXiv.org',                        # libraryCatalog
    19:  f'arXiv:{arxiv_id} [cs]',           # extra
    69:  'arXiv',                            # repository
    107: f'arXiv:{arxiv_id}',                # archiveID
}
for field_id, value in fields.items():
    vid = get_or_create_value(c, value)
    c.execute('INSERT OR IGNORE INTO itemData (itemID, fieldID, valueID) VALUES (?, ?, ?)',
              (parent_id, field_id, vid))

# 3. Create attachment item (version=0, synced=0)
att_key = gen_key(c)
c.execute('''INSERT INTO items (itemTypeID, dateAdded, dateModified, clientDateModified,
             libraryID, key, version, synced) VALUES (3, ?, ?, ?, 1, ?, 0, 0)''',
             (now, now, now, att_key))
att_id = c.lastrowid

# 4. Create attachment record
c.execute('''INSERT INTO itemAttachments (itemID, parentItemID, linkMode, contentType, path, syncState)
             VALUES (?, ?, 0, 'application/pdf', ?, 0)''',
             (att_id, parent_id, f'storage:{pdf_filename}'))

# 5. Set title on attachment
vid = get_or_create_value(c, pdf_filename)
c.execute('INSERT INTO itemData (itemID, fieldID, valueID) VALUES (?, 1, ?)', (att_id, vid))

# 6. Copy PDF to storage
storage_dir = os.path.join(STORAGE, att_key)
os.makedirs(storage_dir, exist_ok=True)
shutil.copy2(src_pdf, os.path.join(storage_dir, pdf_filename))
```

### Step 8 — Validate and Commit

**MANDATORY**: Every DB write operation MUST end with `validate_and_commit()`. Never call `conn.commit()` directly.

```python
def validate_and_commit(conn):
    """Validate DB integrity and commit. Raises on failure, rolls back."""
    c = conn.cursor()
    VALID_CHARS = set('23456789ABCDEFGHIJKLMNPQRSTUVWXYZ')
    errors = []

    # 1. Check no FK violations
    c.execute('PRAGMA foreign_key_check')
    violations = c.fetchall()
    if violations:
        errors.append(f"FK violations: {violations}")

    # 2. Check all synced=0 items have valid keys
    c.execute('SELECT itemID, key FROM items WHERE synced = 0')
    for item_id, key in c.fetchall():
        if len(key) != 8 or not all(ch in VALID_CHARS for ch in key):
            errors.append(f"Invalid key '{key}' on itemID={item_id}")

    # 3. Check no orphan attachments
    c.execute('''SELECT ia.itemID FROM itemAttachments ia
                 WHERE ia.parentItemID IS NOT NULL
                 AND ia.parentItemID NOT IN (SELECT itemID FROM items)''')
    orphans = c.fetchall()
    if orphans:
        errors.append(f"Orphan attachments: {[o[0] for o in orphans]}")

    # 4. Clean dangling deletedItems (auto-fix, not an error)
    c.execute('''SELECT di.itemID FROM deletedItems di
                 WHERE di.itemID NOT IN (SELECT itemID FROM items)''')
    for (item_id,) in c.fetchall():
        c.execute('DELETE FROM deletedItems WHERE itemID = ?', (item_id,))

    # 5. Check wrong field IDs on unsynced items (legacy data from old skill)
    c.execute('''SELECT id.itemID, id.fieldID, idv.value
                 FROM itemData id
                 JOIN itemDataValues idv ON id.valueID = idv.valueID
                 WHERE id.itemID IN (SELECT itemID FROM items WHERE synced=0)''')
    for item_id, field_id, value in c.fetchall():
        if field_id == 13 and 'arxiv' in value.lower():
            errors.append(f"itemID={item_id}: fieldID=13 used for URL (should be 10)")
        elif field_id == 59 and '10.48550' in value:
            errors.append(f"itemID={item_id}: fieldID=59 used for DOI (should be 8)")

    if errors:
        conn.rollback()
        raise ValueError("DB validation failed:\n" + "\n".join(errors))

    conn.commit()
    print(f"validate_and_commit: OK")
```

Use it as the ONLY way to commit:
```python
# After all writes...
validate_and_commit(conn)
conn.close()
```

### Step 9 — Clean Up and Report

Remove temp files (`fix_zotero.py`, `papers_to_add.json`, DB copies).
Report: X papers added, total library now Y papers.
Remind user to sync: "Open Zotero and press Ctrl+Shift+S to sync to cloud."

---

## Mode 3: Repair

Scan and fix DB integrity issues. Requires Zotero to be closed. Always backs up DB before modifying.

Usage: `/sync-zotero repair`

### Step 1 — Pre-flight checks

1. Check Zotero is not running (`curl -s http://127.0.0.1:23119/connector/ping` must fail)
2. If running, ask user to close Zotero first
3. Backup DB: `shutil.copy2(ZOTERO_DB, ZOTERO_DB + ".bak.repair")`

### Step 2 — Fix invalid item keys

Scan all items for keys containing characters outside Zotero's valid charset (`23456789ABCDEFGHIJKLMNPQRSTUVWXYZ`). Characters `0`, `1`, and `O` are NOT valid.

For each invalid key:
1. Generate a new valid key using `gen_key(cursor)`
2. Rename the storage directory: `os.rename(STORAGE/old_key, STORAGE/new_key)`
3. Update the items table: `UPDATE items SET key=? WHERE itemID=?`

### Step 3 — Fix wrong field IDs (legacy data)

Scan `itemData` for items using wrong field IDs from older skill versions:

| Wrong fieldID | Wrong fieldName | Correct fieldID | Correct fieldName | Detection pattern |
|---|---|---|---|---|
| 13 | archiveLocation | 10 | url | value contains `arxiv` |
| 59 | reporterVolume | 8 | DOI | value contains `10.48550` |
| 16 | libraryCatalog | 19 | extra | value starts with `arXiv:` followed by digits |

For each wrong mapping:
1. Check if the correct fieldID already exists for that item
2. If not: `UPDATE itemData SET fieldID=? WHERE itemID=? AND fieldID=?`
3. If yes: `DELETE FROM itemData WHERE itemID=? AND fieldID=?` (remove duplicate)

**Note**: Only fix field 16 if value matches `arXiv:\d{4}\.\d+` pattern. The value `arXiv.org` in field 16 is CORRECT (it's libraryCatalog).

### Step 4 — Fix orphan attachments

```python
c.execute('''SELECT ia.itemID FROM itemAttachments ia
             WHERE ia.parentItemID IS NOT NULL
             AND ia.parentItemID NOT IN (SELECT itemID FROM items)''')
```

For each orphan: use `delete_item(c, orphan_id)` to cleanly remove.

### Step 5 — Fix dangling deletedItems

```python
c.execute('''SELECT di.itemID FROM deletedItems di
             WHERE di.itemID NOT IN (SELECT itemID FROM items)''')
for (item_id,) in c.fetchall():
    c.execute('DELETE FROM deletedItems WHERE itemID = ?', (item_id,))
```

### Step 6 — FK validation

```python
c.execute('PRAGMA foreign_key_check')
violations = c.fetchall()
```

Report any remaining violations for manual investigation.

### Step 7 — Commit and report

Use `validate_and_commit(conn)` to commit all fixes. Report:
- X invalid keys fixed
- X wrong field IDs corrected
- X orphan attachments removed
- X dangling deletedItems cleaned
- FK violations: pass/fail

Remind user: "Open Zotero and press Ctrl+Shift+S to sync."

---

## Mode 4: Merge Duplicates

Detect and merge duplicate items while preserving all user content (PDFs, highlights, comments, notes, tags, collection memberships). Requires Zotero to be closed.

Usage: `/sync-zotero merge`

### Design principles (lessons learned)

1. **Union-find across ALL signatures**, never exclusive priority. A duplicate pair may match on title but not on arXiv ID (e.g. one item has `arXiv:` metadata, the other has only the title). Build disjoint-set clusters by walking every signature type.
2. **Normalize aggressively for title matching**. Strip all non-alphanumeric + non-CJK characters, lowercase. Short normalized titles (< 8 chars) are unreliable — skip them.
3. **Validate arXiv IDs**. The regex `\d{4}\.\d{4,5}` matches DOI fragments like `2023.10004`. Require year prefix in `15..27`.
4. **Ignore synthesized DOIs**. `10.48550/arXiv.*` is derived from the arXiv ID, not a real DOI — do not cluster on it (use the arXiv ID signature instead).
5. **Primary selection is deterministic**: `(-has_pdf, -n_comments, -n_annotations, dateAdded, itemID)`. Keep the richest record; if tied, keep the oldest (earliest `dateAdded`), then lowest itemID.
6. **Preview before destructive ops** — especially if the match count is unexpectedly large. Use `AskUserQuestion` to confirm scope.
7. **Protect items with user comments**. If a duplicate has user-authored comments on annotations and the primary does not, this changes the primary selection (comments dominate PDF presence in terms of irreplaceable user work). See scoring tuple above.
8. **Post-merge set `synced=0, dateModified=now`** on surviving primaries so Zotero cloud sync picks up the change.

### Step 1 — Pre-flight checks

1. Confirm Zotero is not running (`curl -s http://127.0.0.1:23119/connector/ping` must fail)
2. If running: instruct user or run `Stop-Process -Name zotero -Force` (Windows), wait 3 s
3. Backup DB: `shutil.copy2(ZOTERO_DB, ZOTERO_DB + ".bak.merge")` — MANDATORY, never skip

### Step 2 — Collect item metadata

For every non-attachment, non-note item (itemTypeID NOT IN (3, 14)), not in `deletedItems`, gather:

- `itemID`, `key`, `itemTypeID`, `dateAdded`
- `title` (fieldID=1)
- `arxiv_id` — scan ALL fields (19, 10, 107, 13, 8, 2) with `\d{4}\.\d{4,5}` then filter by year prefix 15..27
- `doi` (fieldID=8) — drop if starts with `10.48550/arXiv.`
- `norm_title` — lowercase, stripped of non-alphanumeric/CJK chars
- `has_pdf` — existence check on disk, not just DB row (verify file exists and size > 0)
- `n_annotations`, `n_comments` — count highlights; count annotations where `comment IS NOT NULL AND comment != ''`

### Step 3 — Cluster with union-find

```python
from collections import defaultdict

parent = {iid: iid for iid in info}
def find(x):
    while parent[x] != x:
        parent[x] = parent[parent[x]]
        x = parent[x]
    return x
def union(a, b):
    ra, rb = find(a), find(b)
    if ra != rb: parent[ra] = rb

# Bucket by each signature independently
groups = defaultdict(list)
for iid, d in info.items():
    if d['arxiv']:
        groups[f'a:{d["arxiv"]}'].append(iid)
    if d['doi']:
        groups[f'd:{d["doi"]}'].append(iid)
    if d['norm'] and len(d['norm']) >= 8:
        groups[f't:{d["norm"]}'].append(iid)

# Union all items sharing any signature
for sig, ids in groups.items():
    if len(ids) > 1:
        for i in ids[1:]:
            union(ids[0], i)

# Materialize clusters
clusters = defaultdict(list)
for iid in info:
    clusters[find(iid)].append(iid)
clusters = [ids for ids in clusters.values() if len(ids) > 1]
```

### Step 4 — Select primary per cluster

```python
def score(d):
    # lower is better
    return (0 if d['has_pdf'] else 1,
            -d['n_comments'],
            -d['n_annotations'],
            d['dateAdded'] or '9999',
            d['itemID'])

for cluster in clusters:
    cluster.sort(key=lambda iid: score(info[iid]))
    primary = cluster[0]
    duplicates = cluster[1:]
```

### Step 5 — Preview and confirm

Write a JSON plan listing each cluster with primary + duplicates and reasons (has_pdf, has_comments, dateAdded). Print a summary and — if the count is large (> 20) or the user has not pre-confirmed — use `AskUserQuestion` with options:

- "Merge all N groups"
- "Show plan and let me review first"
- "Cancel"

### Step 6 — Execute merge

For each duplicate group, inside a single transaction:

```python
def merge_duplicate(c, primary_id, dup_id):
    # 1. Move annotations from duplicate's PDF to primary's PDF (if primary has one)
    c.execute('SELECT itemID FROM itemAttachments WHERE parentItemID = ?', (dup_id,))
    dup_atts = [r[0] for r in c.fetchall()]

    c.execute("""SELECT itemID FROM itemAttachments
                 WHERE parentItemID=? AND contentType='application/pdf' LIMIT 1""",
              (primary_id,))
    primary_pdf = c.fetchone()
    primary_pdf_id = primary_pdf[0] if primary_pdf else None

    for att_id in dup_atts:
        c.execute("SELECT COUNT(*) FROM itemAnnotations WHERE parentItemID = ?", (att_id,))
        n_ann = c.fetchone()[0]
        if n_ann > 0 and primary_pdf_id:
            # Move annotations to primary's PDF
            c.execute('UPDATE itemAnnotations SET parentItemID=? WHERE parentItemID=?',
                      (primary_pdf_id, att_id))
        elif not primary_pdf_id:
            # Primary has no PDF -- reparent the whole attachment
            c.execute('UPDATE itemAttachments SET parentItemID=? WHERE itemID=?',
                      (primary_id, att_id))
            continue  # do not delete, it now belongs to primary

    # 2. Move notes to primary
    c.execute('UPDATE itemNotes SET parentItemID=? WHERE parentItemID=?',
              (primary_id, dup_id))

    # 3. Delete the duplicate (and any remaining child attachments)
    delete_item(c, dup_id)

# Mark primaries as locally modified so Zotero syncs them
now = datetime.datetime.now(datetime.timezone.utc).strftime('%Y-%m-%d %H:%M:%S')
for primary_id in primary_ids:
    c.execute('UPDATE items SET dateModified=?, synced=0 WHERE itemID=?',
              (now, primary_id))
```

**Note on collections**: `UPDATE collectionItems SET itemID=primary WHERE itemID=dup` can fail on the unique constraint `(collectionID, itemID)` if both items are in the same collection. Prefer to simply delete the duplicate's `collectionItems` rows via `delete_item()` — the primary likely already has the same memberships, and if not the user can re-add.

**Note on tags**: same caveat; `delete_item()` cleans `itemTags` for the duplicate. If you want to preserve unique tags from the duplicate, run `INSERT OR IGNORE INTO itemTags (itemID, tagID, type) SELECT ?, tagID, type FROM itemTags WHERE itemID = ?` with `(primary_id, dup_id)` BEFORE calling `delete_item`.

### Step 7 — Validate and commit

Must use `validate_and_commit(conn)` (see Mode 2 Step 8). It runs `PRAGMA foreign_key_check` and rolls back on any violation. **Rollback is the correct behavior** — the backup is on disk and the script can be rerun after the logic bug is fixed.

### Step 8 — Report

- `N` clusters detected, `M` items merged (M = sum of duplicates, not clusters)
- Library size before/after
- Annotations reparented, notes reparented, storage folders removed
- Remind user: "Open Zotero and press Ctrl+Shift+S to sync deletions to cloud."

### Common pitfalls

| Symptom | Root cause | Fix |
|---|---|---|
| 0 duplicate clusters found when obvious dupes exist in UI | Exclusive signature priority (arxiv beats title) | Union-find across ALL signatures; never `elif` between signature types |
| `FK violation on fulltextItems` during merge | `delete_item` missed `fulltextItems` / `fulltextItemWords` / `itemTopLevel` / `highlights` / `annotations` tables | Use the expanded `delete_item` (see Zotero DB Reference below) |
| `UNIQUE constraint failed: collectionItems` | Primary already in the same collection as duplicate | Do not UPDATE; let `delete_item` drop the duplicate's `collectionItems` row |
| Unpacking error on `norm_title` | Function returns empty string in one branch, tuple in another | Make return type consistent (`return ('', '')` for empty case) |
| Today's newly-imported paper appears "deleted" | User imported twice at different times; merge correctly removed one copy | Check `zotero.sqlite.bak.merge` vs live DB to prove no unintended delete |
| Merge silently removes items user still wanted | Scoring did not weight `n_comments` high enough | Ensure `(-has_pdf, -n_comments, ...)` places comments before recency |

---

## Zotero DB Reference

### Item Types
| itemTypeID | Type |
|---|---|
| 3 | attachment |
| 14 | note |
| 22 | journalArticle |
| 31 | preprint |
| 40 | webpage |

### Field IDs for Preprints (itemTypeID=31)

**CRITICAL**: Use the correct field IDs. These are verified against the Zotero schema:

| fieldID | fieldName | Usage | Example |
|---|---|---|---|
| 1 | title | Paper title | "Attention Is All You Need" |
| 2 | abstractNote | Abstract | (full abstract text) |
| 6 | date | Publication date | "2026-03-15" |
| 8 | **DOI** | Digital Object ID | "10.48550/arXiv.2603.12345" |
| 10 | **url** | Paper URL | "http://arxiv.org/abs/2603.12345" |
| 11 | accessDate | When item was added | "2026-03-30 12:00:00" |
| 14 | shortTitle | Short title | "Attention" |
| 15 | language | Paper language | "en" |
| 16 | libraryCatalog | Source catalog | "arXiv.org" |
| 19 | **extra** | Extra metadata | "arXiv:2603.12345 [cs]" |
| 69 | **repository** | Repository name | "arXiv" |
| 107 | **archiveID** | Archive identifier | "arXiv:2603.12345" |

**WRONG field IDs to AVOID** (these are valid fields but NOT the standard ones):
| fieldID | fieldName | DO NOT use for |
|---|---|---|
| 13 | archiveLocation | ~~URL~~ (use fieldID=10 url instead) |
| 59 | reporterVolume | ~~DOI~~ (use fieldID=8 DOI instead) |

### Key Generation

**CRITICAL**: Zotero keys use a RESTRICTED character set that excludes `0` (zero), `1` (one), and `O` (letter O) to avoid visual ambiguity. Using the wrong charset causes `"'{key}' is not a valid item key"` errors during sync.

```python
import random

# Zotero's ACTUAL key charset -- NO '0', '1', or 'O'
ZOTERO_KEY_CHARS = '23456789ABCDEFGHIJKLMNPQRSTUVWXYZ'

def gen_key(cursor):
    """Generate a valid 8-char Zotero item key."""
    while True:
        key = ''.join(random.choices(ZOTERO_KEY_CHARS, k=8))
        cursor.execute('SELECT COUNT(*) FROM items WHERE key = ?', (key,))
        if cursor.fetchone()[0] == 0:
            return key
```

### Value Lookup/Create
```python
def get_or_create_value(c, value):
    c.execute('SELECT valueID FROM itemDataValues WHERE value=?', (value,))
    row = c.fetchone()
    if row: return row[0]
    c.execute('INSERT INTO itemDataValues (value) VALUES (?)', (value,))
    return c.lastrowid
```

### Sync Model

Items in Zotero have two key sync fields:
- `synced`: 0 = has local changes pending upload, 1 = in sync with server
- `version`: 0 = never synced to server, >0 = last known server version

For **new local items**: set `version=0, synced=0`. Zotero will upload them on next sync.
For **items synced from server**: they have `version>0, synced=1`. Do not modify these unless intentional.

### Attachment Model

```
itemAttachments columns:
  itemID, parentItemID, linkMode, contentType, charsetID, path, syncState,
  storageModTime, storageHash, lastProcessedModificationTime
```

- `linkMode=0`: imported file (stored in Zotero storage)
- `syncState=0`: file is present locally
- `syncState=1`: file needs to be downloaded
- `path`: format is `storage:{filename}` for imported files
- `storageModTime`, `storageHash`: can be NULL; Zotero fills these during sync

### Deleting Items

When deleting items from the DB directly, clean up ALL related tables:

```python
def delete_item(c, item_id):
    """Delete an item and all its related records.

    CRITICAL: Must recurse into THREE kinds of child relationships before deleting:
    - itemAttachments.parentItemID  (child PDFs, snapshots)
    - itemNotes.parentItemID        (child notes — easy to forget)
    - itemAnnotations.parentItemID  (highlights on a PDF child)

    Forgetting itemNotes.parentItemID causes FK violations like
    `('itemNotes', <note_id>, 'items', 0)` when you later try to empty trash or
    run merge. Observed 2026-05-11 on a 600+ item library — 7 orphan notes
    rolled back a 49-item trash-empty operation until this recursion was added.
    """
    # 1. Recurse into child ATTACHMENTS
    c.execute('SELECT itemID FROM itemAttachments WHERE parentItemID = ?', (item_id,))
    for (child_id,) in c.fetchall():
        delete_item(c, child_id)

    # 2. Recurse into child NOTES (via itemNotes.parentItemID, not itemAttachments)
    c.execute('SELECT itemID FROM itemNotes WHERE parentItemID = ?', (item_id,))
    for (note_id,) in c.fetchall():
        delete_item(c, note_id)

    # 3. Recurse into child ANNOTATIONS
    c.execute('SELECT itemID FROM itemAnnotations WHERE parentItemID = ?', (item_id,))
    for (ann_id,) in c.fetchall():
        delete_item(c, ann_id)

    # 4. Get key for storage cleanup BEFORE deleting the items row
    c.execute('SELECT key FROM items WHERE itemID = ?', (item_id,))
    row = c.fetchone()
    key = row[0] if row else None

    # 5. Drop every row referencing this itemID. The fulltext* / highlights /
    #    itemTopLevel tables are REQUIRED on Zotero 7+ to avoid FK violations
    #    during merge or empty-trash. Missing tables are try/except'd.
    tables = [
        'itemData', 'itemTags', 'itemCreators', 'itemRelations',
        'collectionItems', 'itemAttachments', 'deletedItems',
        'itemAnnotations', 'itemNotes',
        # Extended coverage for Zotero 7+:
        'fulltextItems', 'fulltextItemWords', 'itemTopLevel',
        'highlights', 'groupItems', 'feedItems',
        'syncedSettings', 'publicationsItems',
    ]
    for t in tables:
        try:
            c.execute(f'DELETE FROM {t} WHERE itemID = ?', (item_id,))
        except sqlite3.OperationalError:
            pass  # table absent in this schema version

    # 6. Remove storage directory (linked-file entries outside STORAGE stay intact)
    if key:
        storage_dir = os.path.join(STORAGE, key)
        if os.path.isdir(storage_dir):
            try:
                shutil.rmtree(storage_dir)
            except Exception:
                pass

    # 7. Finally drop the items row itself
    c.execute('DELETE FROM items WHERE itemID = ?', (item_id,))
```

**Common FK-rollback scenarios** (all fixed by the expanded recursion + table list above):

| FK violation signature | Root cause |
|---|---|
| `('itemNotes', <note_id>, 'items', 0)` during empty-trash or merge | `delete_item` didn't recurse via `itemNotes.parentItemID` — child notes of the deleted parent became orphans |
| `('fulltextItems', <att_id>, 'items', 0)` | PDF was indexed; deleting the attachment row left an orphan fulltext entry |
| `('itemTopLevel', <iid>, 'items', 0)` | `itemTopLevel` view/materialized table references removed item |
| `('highlights', <ann_id>, 'items', 0)` on Zotero 7+ | newer schema has a separate highlights table |

The `empty-trash` workflow (and `merge` Step 6) should run `PRAGMA foreign_key_check` **before** `commit()`, and rollback on any violation — the DB backup at Step 1 makes the rerun cheap.

---

## Error Handling

- **DB locked**: Copy DB for reads; ask user to close Zotero for writes. Use retry loop (30 attempts, 2s apart) with `BEGIN EXCLUSIVE` test
- **SSL errors (Python urllib)**: Use `curl.exe` / `curl` for all HTTPS; on Windows add `--ssl-no-revoke`
- **arXiv rate limiting**: Add `time.sleep(1)` between requests; batch 5 then pause
- **Encoding errors (Windows)**: Always `sys.stdout.reconfigure(encoding='utf-8')`
- **Output buffering**: Use `print = functools.partial(print, flush=True)` for real-time output
- **Duplicate items**: Check existing arXiv IDs before creating; delete accidental duplicates using `delete_item()` above
- **Hallucinated papers from search agents**: Always verify arXiv ID exists before presenting to user
- **Invalid keys after creation**: Always run post-write validation (Step 8). If keys contain `0`, `1`, or `O`, regenerate them
- **FK violations**: Run `PRAGMA foreign_key_check` after writes. Fix orphan attachments and dangling deletedItems entries
