
# PRD — Data Processing & Storage (Local MVP)

## Scope

* File types: **native PDFs** (PyMuPDF) and **DOCX** (`python-docx`).
* Scanned PDFs/images: **optional** via Azure Document Intelligence OCR; otherwise skipped in MVP.
* Stores: **SQLite (FTS5)** for metadata + chunks; **FAISS** files for vectors.
* Citations: **built programmatically at answer time** from DB fields (not stored as strings).

---

# 1) Data Flow (Stage-by-Stage, with I/O and Storage)

## Stage 1 — Ingest (files → DB rows + vector index)

### 1.1 Detect & Extract

* **Input**: Files in `/data/raw_docs/**/*.pdf|.docx`
* **Process**:

  * Native PDFs: PyMuPDF → text + `page_start/end`, headings, lists.
  * DOCX: `python-docx` → text, paragraphs, headings.
  * Scanned PDFs/images (if used): Azure Document Intelligence (Read/Layout) → text + page + (optional) bounding boxes.
* **Output (in-memory)**: per-document text blocks with page numbers and rough structure.
* **Stored**: not yet—passed to chunking.

### 1.2 Chunk & Normalize

* **Input**: extracted text blocks (with pages/headings)
* **Process**: split by headings/section numbers; detect “Step N” blocks; fallback to \~800–1200 tokens; compute `section_path`, `page_start/end`.
* **Output (persisted)**:

  * `documents` row (one per file/version)
  * `chunks` rows (one per chunk; text is indexed by **FTS5**)
* **Stored in**: `/data/knowledge.db` (SQLite)

### 1.3 Metadata & Versioning

* **Input**: filename/front-matter
* **Process**: derive `title`, `bu`, `asset_id?`, `equipment_id?`, `rev_label`, `valid_from/to`, `status`; compute `checksum`; detect supersedence.
* **Output (persisted)**:

  * Updated `documents` row (`status='active'|'superseded'`)
  * Optional `relations` row: `(new_doc) -[supersedes]-> (old_doc)`
* **Stored in**: SQLite

### 1.4 Entities (Light)

* **Input**: chunk text
* **Process**: regex for `ASSET-\d+`, `EQP-[A-Z0-9]+`, `SOP-\d+`, standards (e.g., `API 510`)
* **Output (persisted)**: `entities`, `chunk_entities` rows
* **Stored in**: SQLite

### 1.5 Embeddings & Vector Index

* **Input**: chunk text
* **Process**: Azure OpenAI embeddings → **L2 normalize**; assign `faiss_id`; add to FAISS (IndexFlatIP + IDMap2)
* **Output (persisted)**:

  * `chunk_vectors` rows (vector cache)
  * `vector_index_map` rows (`faiss_id ↔ chunk_id`)
  * `index.faiss` + `meta.json`
* **Stored in**:

  * SQLite (`chunk_vectors`, `vector_index_map`)
  * Filesystem: `/data/faiss/index.faiss`, `/data/faiss/meta.json`

**Citations**: *not prebuilt.* We reconstruct citation objects when answering from `chunks.page_start/end`, `chunks.section_path`, and `documents.title/rev_label/source_path`.

---

## Stage 2 — Query (user → planner → BM25 + vectors → fusion → citations)

### 2.1 Planner (Intent & Slots)

* **Input**: user text + (optional) prior conversation context
* **Process**: small LLM (or rules) → `{intent, slots}` where `slots` may include `topic`, `asset`, `equipment`, `bu`, `latest_only=true`
* **Output**: `QuerySpec` JSON
* **Stored**: transient (in-memory); can be logged if needed

### 2.2 BM25 Candidates (SQLite/FTS5)

* **Input**: `QuerySpec` → compile safe **FTS5 MATCH** string + filters
* **Process**: parameterized SQL (below) to fetch lexical top-K
* **Output**: list of candidates `(chunk_id, doc_id, kw_score, section_path, page_start/end)`
* **Fetched from**: `/data/knowledge.db` (`chunks` + `documents`)

### 2.3 Semantic Candidates (FAISS)

* **Input**: original user query text
* **Process**: embed → search FAISS top-K; map `faiss_id → chunk_id` via `vector_index_map`
* **Output**: list `(chunk_id, sim)`
* **Fetched from**: `/data/faiss/index.faiss` + SQLite map

### 2.4 Fusion & Filters

* **Input**: BM25 + FAISS candidates + `slots`
* **Process**: min-max normalize → `final = α*kw_norm + (1-α)*vec_norm` (start α=0.5); boost if asset/equipment matches; ensure `status='active'`; optional `seeAlso` expansion
* **Output**: **top-N chunks** with doc refs
* **Fetched from**: SQLite (`relations` if expanding)

### 2.5 Synthesis & Citations

* **Input**: top-N chunks (text + IDs) + user query
* **Process**: LLM writes concise answer using **only** supplied chunks; app builds citation objects:

  ```
  {doc_id, title, rev_label, section, pages[], chunk_id, source_path}
  ```
* **Output**: `answer_md`, `citations[]`
* **Stored**: transient; can be logged

---

# 2) Schema (SQLite + FTS5) — DDL & Creation Code

## 2.1 SQL DDL (copy-paste)

```sql
-- documents: one row per ingested file/version
CREATE TABLE IF NOT EXISTS documents (
  doc_id TEXT PRIMARY KEY,          -- uuid
  title TEXT,
  source_path TEXT,                 -- file:///… path
  bu TEXT,
  asset_id TEXT,
  equipment_id TEXT,
  rev_label TEXT,
  status TEXT,                      -- 'active'|'superseded'
  valid_from TEXT,
  valid_to TEXT,
  checksum TEXT,
  supersedes TEXT                   -- doc_id of previous version (nullable)
);

-- chunks: content-addressable passages, indexed with FTS5 (BM25)
CREATE VIRTUAL TABLE IF NOT EXISTS chunks USING fts5(
  chunk_id UNINDEXED,               -- uuid
  doc_id UNINDEXED,
  section_path,                     -- e.g., "2.3/Startup/Steps 1–5"
  page_start UNINDEXED,
  page_end UNINDEXED,
  text,                             -- full text of the chunk
  tokenize = 'porter'
);

-- vector cache: store normalized embeddings so we don’t recompute
CREATE TABLE IF NOT EXISTS chunk_vectors (
  chunk_id TEXT PRIMARY KEY,
  model TEXT NOT NULL,              -- e.g., 'text-embedding-3-small'
  dim INTEGER NOT NULL,
  vector BLOB NOT NULL,             -- float32 bytes (normalized)
  created_at TEXT
);

-- FAISS ID mapping (only if using IndexIDMap2 with external IDs)
CREATE TABLE IF NOT EXISTS vector_index_map (
  faiss_id INTEGER PRIMARY KEY,
  chunk_id TEXT UNIQUE
);

-- entities: optional catalog for assets/equipment/standards/terms
CREATE TABLE IF NOT EXISTS entities (
  entity_id TEXT PRIMARY KEY,
  type TEXT,                        -- 'asset'|'equipment'|'standard'|'term'
  canonical_value TEXT,
  alt_labels TEXT                   -- JSON array of synonyms
);

-- mentions of entities within chunks (spans optional)
CREATE TABLE IF NOT EXISTS chunk_entities (
  chunk_id TEXT,
  entity_id TEXT,
  span_start INTEGER,
  span_end INTEGER,
  confidence REAL
);

-- relations: cross-doc/chunk links with evidence for traceability
CREATE TABLE IF NOT EXISTS relations (
  src_type TEXT, src_id TEXT,
  rel_type TEXT,                    -- 'supersedes'|'seeAlso'|'appliesToAsset'|'appliesToEquipment'
  dst_type TEXT, dst_id TEXT,
  confidence REAL,
  evidence TEXT                     -- JSON: {"phrase":"…","page":7,"span":[123,147]}
);
CREATE INDEX IF NOT EXISTS rel_idx ON relations (src_id, rel_type, dst_id);

-- helpful view to select only latest (active) docs
CREATE VIEW IF NOT EXISTS active_documents AS
SELECT * FROM documents WHERE status = 'active';
```

## 2.2 Table Creation (Python) — automated init

```python
# init_db.py
import sqlite3, pathlib

DDL = r"""
PRAGMA foreign_keys=ON;

CREATE TABLE IF NOT EXISTS documents (
  doc_id TEXT PRIMARY KEY,
  title TEXT,
  source_path TEXT,
  bu TEXT,
  asset_id TEXT,
  equipment_id TEXT,
  rev_label TEXT,
  status TEXT,
  valid_from TEXT,
  valid_to TEXT,
  checksum TEXT,
  supersedes TEXT
);

CREATE VIRTUAL TABLE IF NOT EXISTS chunks USING fts5(
  chunk_id UNINDEXED,
  doc_id UNINDEXED,
  section_path,
  page_start UNINDEXED,
  page_end UNINDEXED,
  text,
  tokenize='porter'
);

CREATE TABLE IF NOT EXISTS chunk_vectors (
  chunk_id TEXT PRIMARY KEY,
  model TEXT NOT NULL,
  dim INTEGER NOT NULL,
  vector BLOB NOT NULL,
  created_at TEXT
);

CREATE TABLE IF NOT EXISTS vector_index_map (
  faiss_id INTEGER PRIMARY KEY,
  chunk_id TEXT UNIQUE
);

CREATE TABLE IF NOT EXISTS entities (
  entity_id TEXT PRIMARY KEY,
  type TEXT,
  canonical_value TEXT,
  alt_labels TEXT
);

CREATE TABLE IF NOT EXISTS chunk_entities (
  chunk_id TEXT,
  entity_id TEXT,
  span_start INTEGER,
  span_end INTEGER,
  confidence REAL
);

CREATE TABLE IF NOT EXISTS relations (
  src_type TEXT, src_id TEXT,
  rel_type TEXT,
  dst_type TEXT, dst_id TEXT,
  confidence REAL,
  evidence TEXT
);
CREATE INDEX IF NOT EXISTS rel_idx ON relations (src_id, rel_type, dst_id);

CREATE VIEW IF NOT EXISTS active_documents AS
SELECT * FROM documents WHERE status='active';
"""

def init_db(db_path: str = "/data/knowledge.db"):
    pathlib.Path(db_path).parent.mkdir(parents=True, exist_ok=True)
    con = sqlite3.connect(db_path)
    try:
        con.executescript(DDL)
        con.commit()
    finally:
        con.close()

if __name__ == "__main__":
    init_db()
    print("SQLite initialized.")
```

Run once at startup (your `ingest.py` should call `init_db()` if needed). No manual steps.

---

# 3) “How we answer” — exact query workflow (with premade SQL)

1. **Planner builds `QuerySpec`** (intent + slots)
   Example:

   ```json
   {"intent":"guideline_answer",
    "slots":{"topic":"compressor startup","asset":"ASSET-104","bu":"EAST","latest_only":true}}
   ```

2. **Compile safe FTS5 MATCH** from `slots.topic` (+ synonyms) and any hard IDs (regex).
   Example: `(compressor AND startup) OR "SOP-42"`

3. **BM25 (FTS5) candidates** — parameterized SQL:

   ```sql
   SELECT
     c.chunk_id, c.doc_id, c.section_path, c.page_start, c.page_end,
     bm25(chunks) AS kw_score
   FROM chunks c
   JOIN documents d ON d.doc_id = c.doc_id
   WHERE c MATCH :fts_query
     AND d.status = 'active'
     AND (:bu IS NULL OR d.bu = :bu)
     AND (:asset IS NULL OR d.asset_id = :asset)
     AND (:equipment IS NULL OR d.equipment_id = :equipment)
   ORDER BY kw_score
   LIMIT 200;
   ```

4. **Semantic candidates** — embed original user query; `faiss.search(k=200)`; map `faiss_id → chunk_id` via `vector_index_map`.

5. **Fusion & filters** — min-max normalize (`kw_norm = 1 - norm(kw_score)`, `vec_norm = norm(sim)`), `final = 0.5*kw + 0.5*vec`, boost if asset/equipment match; ensure active docs only; optional `seeAlso` expansion via `relations`.

6. **Citations** — programmatically build:

   ```json
   { "doc_id":"…","title":"SOP-42 Rev C","rev_label":"Rev C",
     "section":"2.3/Startup/Steps 1–5","pages":[7,8],
     "chunk_id":"…","source_path":"file:///…/SOP-42-RevC.pdf#page=7" }
   ```

   These are *derived* from `documents` + `chunks` for the chosen top chunks.

7. **Synthesis** — LLM composes the answer using **only** the top chunks; your UI renders `answer_md` + citations.

---

  * SQLite: `/data/knowledge.db`
  * FAISS: `/data/faiss/index.faiss` (+ `/data/faiss/meta.json`)
* Ingestion tasks map to Stage 1.1–1.5; retrieval tasks map to Stage 2.1–2.5

