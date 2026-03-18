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

- **Windows (Python 3.14)**: SSL broken in urllib; always use `curl.exe` for HTTPS downloads
- **All platforms**: Always `sys.stdout.reconfigure(encoding='utf-8')` in Python scripts
- **Zotero locking**: Copy DB for reads when Zotero is running; close Zotero for writes
- Check if Zotero is running: `curl -s http://127.0.0.1:23119/connector/ping`

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

From parent item's metadata fields:
- `fieldID=16` (extra): Look for `arXiv:XXXX.XXXXX`
- `fieldID=13` (url): Look for `arxiv.org/abs/XXXX.XXXXX`

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
curl -L -o "{ACADEMY_DIR}/{arxivID}v{N} {Title}.pdf" "https://arxiv.org/pdf/{arxivID}v{N}"
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
1. Generate unique 8-char key (uppercase + digits, no collision)
2. Create attachment item (itemTypeID=3)
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
LEFT JOIN itemData id_e ON i.itemID = id_e.itemID AND id_e.fieldID = 16
LEFT JOIN itemDataValues idv_e ON id_e.valueID = idv_e.valueID
LEFT JOIN itemData id_u ON i.itemID = id_u.itemID AND id_u.fieldID = 13
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
- `sebastianraschka.com` — ML research blog
- `www.interconnects.ai` — Nathan Lambert's AI blog
- `simonwillison.net` — LLM practitioner blog
- `karpathy.github.io` — Karpathy's blog
- `thezvi.substack.com` — AI analysis
- `www.latent.space` — AI engineering
- `the-decoder.com` — AI news
- `www.marktechpost.com` — AI research news
- `newsletter.maartengrootendorst.com` — NLP newsletter
- `news.ycombinator.com` — HN (search AI/ML papers)
- `www.reddit.com/r/MachineLearning` — Top posts
- `paperswithcode.com/greatest` — Trending papers

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
5. If running: write `fix_zotero.py` script and ask user to close Zotero

**Creating Zotero items** (when writing directly to DB):

```python
# 1. Create parent preprint item
c.execute('''INSERT INTO items (itemTypeID, dateAdded, dateModified, clientDateModified,
             libraryID, key, version, synced) VALUES (31, ?, ?, ?, 1, ?, ?, 0)''',
             (now, now, now, parent_key, max_ver))
parent_id = c.lastrowid

# 2. Add metadata fields
fields = {
    1: title,           # title
    6: date,            # date (from arXiv citation_date)
    13: url,            # url (https://arxiv.org/abs/{id})
    16: f'arXiv:{id}',  # extra
    59: f'10.48550/arXiv.{id}'  # DOI
}
for field_id, value in fields.items():
    vid = get_or_create_value(c, value)
    c.execute('INSERT OR IGNORE INTO itemData (itemID, fieldID, valueID) VALUES (?, ?, ?)',
              (parent_id, field_id, vid))

# 3. Create attachment item
c.execute('''INSERT INTO items (itemTypeID, ...) VALUES (3, ...)''')
att_id = c.lastrowid

# 4. Create attachment record
c.execute('''INSERT INTO itemAttachments (itemID, parentItemID, linkMode, contentType, path, syncState)
             VALUES (?, ?, 0, 'application/pdf', ?, 0)''',
             (att_id, parent_id, f'storage:{pdf_filename}'))

# 5. Copy PDF to storage
shutil.copy2(src, os.path.join(STORAGE, att_key, pdf_filename))
```

### Step 8 — Clean Up and Report

Remove temp files (`fix_zotero.py`, `papers_to_add.json`, DB copies).
Report: X papers added, total library now Y papers.

---

## Zotero DB Reference

### Item Types
| itemTypeID | Type |
|---|---|
| 3 | attachment |
| 14 | attachment (alt) |
| 22 | journalArticle |
| 31 | preprint |
| 40 | webpage |

### Field IDs
| fieldID | Name | Usage |
|---|---|---|
| 1 | title | Paper title |
| 6 | date | Publication date |
| 13 | url | arXiv URL |
| 16 | extra | `arXiv:{id}` |
| 59 | DOI | `10.48550/arXiv.{id}` |

### Key Generation
```python
import string, random
def gen_key(existing_keys):
    chars = string.ascii_uppercase + string.digits
    while True:
        k = ''.join(random.choices(chars, k=8))
        if k not in existing_keys:
            existing_keys.add(k)
            return k
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

---

## Error Handling

- **DB locked**: Copy DB for reads; ask user to close Zotero for writes
- **SSL errors (Python urllib)**: Use `curl.exe` / `curl` for all HTTPS
- **arXiv rate limiting**: Add `time.sleep(1)` between requests; batch 5 then pause
- **Encoding errors (Windows)**: Always `sys.stdout.reconfigure(encoding='utf-8')`
- **Duplicate items**: Check existing arXiv IDs before creating; delete accidental duplicates from `items`, `itemData`, `itemAttachments`, `itemCreators`, `itemTags`, `itemRelations`, `collectionItems`
- **Hallucinated papers from search agents**: Always verify arXiv ID exists before presenting to user
