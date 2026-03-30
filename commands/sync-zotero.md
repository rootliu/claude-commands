---
argument-hint: [sync|discover|full] [--topics=reasoning,agentic,world-model,...] [--year=2026]
description: Sync missing Zotero PDFs from arXiv, discover new papers by topic, and import with user confirmation
---

# Zotero Paper Sync & Discovery

Manage a Zotero library: sync missing PDFs, discover new papers from arXiv/blogs/KOL, and import selected papers with user confirmation.

## Modes

- `sync` (default): Scan Zotero DB for missing PDFs, download from arXiv, link to Zotero
- `discover`: Search arXiv + blogs for new papers matching research interests, ask user one-by-one, import selected
- `full`: Run sync first, then discover

Usage: `/sync-zotero`, `/sync-zotero sync`, `/sync-zotero discover --topics=reasoning,agentic`, `/sync-zotero full`

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

From parent item's metadata fields (use the CORRECT field IDs):
- `fieldID=19` (extra): Look for `arXiv:XXXX.XXXXX`
- `fieldID=10` (url): Look for `arxiv.org/abs/XXXX.XXXXX`
- `fieldID=107` (archiveID): Look for `arXiv:XXXX.XXXXX`

Regex: `(\d{4}\.\d{4,5})`

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

### Step 8 — Post-Write Validation

**CRITICAL**: After writing to DB, always validate:

```python
# 1. Check no FK violations
c.execute('PRAGMA foreign_key_check')
violations = c.fetchall()
assert not violations, f"FK violations: {violations}"

# 2. Check all synced=0 items have valid keys
c.execute('SELECT key FROM items WHERE synced = 0')
VALID_CHARS = set('23456789ABCDEFGHIJKLMNPQRSTUVWXYZ')
for (key,) in c.fetchall():
    assert len(key) == 8 and all(ch in VALID_CHARS for ch in key), f"Bad key: {key}"

# 3. Check no orphan attachments
c.execute('''SELECT ia.itemID FROM itemAttachments ia
             WHERE ia.parentItemID IS NOT NULL
             AND ia.parentItemID NOT IN (SELECT itemID FROM items)''')
orphans = c.fetchall()
assert not orphans, f"Orphan attachments: {orphans}"

# 4. Check no dangling deletedItems
c.execute('''SELECT di.itemID FROM deletedItems di
             WHERE di.itemID NOT IN (SELECT itemID FROM items)''')
dangling = c.fetchall()
# Clean up any dangling entries
for (item_id,) in dangling:
    c.execute('DELETE FROM deletedItems WHERE itemID = ?', (item_id,))
```

### Step 9 — Clean Up and Report

Remove temp files (`fix_zotero.py`, `papers_to_add.json`, DB copies).
Report: X papers added, total library now Y papers.
Remind user to sync: "Open Zotero and press Ctrl+Shift+S to sync to cloud."

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
    """Delete an item and all its related records."""
    # Delete child attachments first
    c.execute('SELECT itemID FROM itemAttachments WHERE parentItemID = ?', (item_id,))
    for (child_id,) in c.fetchall():
        delete_item(c, child_id)  # recursive for attachments

    # Remove from all related tables
    for table in ['itemData', 'itemTags', 'itemCreators', 'itemRelations',
                  'collectionItems', 'itemAttachments', 'deletedItems',
                  'itemAnnotations', 'itemNotes']:
        try:
            c.execute(f'DELETE FROM {table} WHERE itemID = ?', (item_id,))
        except:
            pass  # table may not exist in all Zotero versions

    # Remove storage directory
    c.execute('SELECT key FROM items WHERE itemID = ?', (item_id,))
    row = c.fetchone()
    if row:
        storage_dir = os.path.join(STORAGE, row[0])
        if os.path.isdir(storage_dir):
            shutil.rmtree(storage_dir)

    # Remove from items table
    c.execute('DELETE FROM items WHERE itemID = ?', (item_id,))
```

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
