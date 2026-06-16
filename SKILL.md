---
name: multimodal-wiki
description: "把内容摄入/写入个人知识库(wiki)、巡检与查询它 — the skill for any knowledge base / wiki / 知识库 ingest, audit, or query. Multimodal compounding knowledge base (Karpathy's LLM Wiki pattern): ingest text, web pages, PDFs, screenshots (read via a vision model), and audio (transcribed via whisper) into one interlinked markdown wiki with note-level provenance. Triggers/触发词: 摄入、写入我的知识库/wiki、入库、巡检/审计(lint/audit)、查询知识库、raw 目录有新文件、直接发截图/录音入库. Self-contained fork of llm-wiki. Setup: README.md."
version: 1.0.0
author: AIDiscovery2045 (fork of Hermes Agent's llm-wiki)
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [wiki, knowledge-base, multimodal, vision, audio, research, notes, markdown, rag-alternative]
    category: research
    related_skills: [obsidian, arxiv]
---

# Multimodal Wiki

Build and maintain a persistent, compounding knowledge base as interlinked markdown files —
ingesting not just text and PDFs but also **web pages, screenshots (read via a vision
model), and audio (transcribed via whisper)**, each with note-level provenance.
Based on [Andrej Karpathy's LLM Wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f),
extended from Hermes Agent's `llm-wiki` skill. **Install & setup: see `README.md` (skill root).**

Unlike traditional RAG (which rediscovers knowledge from scratch per query), the wiki
compiles knowledge once and keeps it current. Cross-references are already there.
Contradictions have already been flagged. Synthesis reflects everything ingested.

**Division of labor:** The human curates sources and directs analysis. The agent
summarizes, cross-references, files, and maintains consistency.

## When This Skill Activates

**This is THE skill for any knowledge base / wiki / 知识库 ingest, audit, or query work.**
Use it when the user (English or 中文):
- Asks to ingest / add / process a source into their wiki — **「摄入」「写入我的知识库」「入库」「ingest into my wiki」**
- Drops files into `raw/` and asks to process them — **「摄入 raw 目录」「把 raw 新增内容写入 wiki」**
- Directly attaches a **screenshot / diagram / image** to ingest — read it via vision (`references/image-ingest.md`)
- Directly attaches an **audio file / voice memo** to ingest — transcribe via whisper (`references/audio-ingest.md`)
- Asks to **lint / audit / health-check / 巡检 / 审计** their wiki
- Asks a question and an existing wiki is present — **「查询我的知识库」**
- Asks to create, build, or start a wiki / knowledge base — **「建知识库」**
- References their wiki, knowledge base, 知识库, or "notes" in a research context

## Wiki Location

**Location:** Set via `WIKI_PATH` environment variable (e.g. in `~/.hermes/.env`).

If unset, defaults to `~/wiki`.

```bash
WIKI="${WIKI_PATH:-$HOME/wiki}"
```

The wiki is just a directory of markdown files — open it in Obsidian, VS Code, or
any editor. No database, no special tooling required.

## Architecture: Three Layers

```
wiki/
├── SCHEMA.md           # Conventions, structure rules, domain config
├── index.md            # Sectioned content catalog with one-line summaries
├── log.md              # Chronological action log (append-only, rotated yearly)
├── raw/                # Layer 1: Immutable source material
│   ├── articles/       # Web articles, clippings
│   ├── papers/         # PDFs, arxiv papers
│   ├── transcripts/    # Meeting notes, interviews
│   └── assets/         # Images, diagrams referenced by sources
├── entities/           # Layer 2: Entity pages (people, orgs, products, models)
├── concepts/           # Layer 2: Concept/topic pages
├── comparisons/        # Layer 2: Side-by-side analyses
└── queries/            # Layer 2: Filed query results worth keeping
```

**Layer 1 — Raw Sources:** Immutable. The agent reads but never modifies these.
**Layer 2 — The Wiki:** Agent-owned markdown files. Created, updated, and
cross-referenced by the agent.
**Layer 3 — The Schema:** `SCHEMA.md` defines structure, conventions, and tag taxonomy.

## Resuming an Existing Wiki (CRITICAL — do this every session)

When the user has an existing wiki, **always orient yourself before doing anything**:

① **Read `SCHEMA.md`** — understand the domain, conventions, and tag taxonomy.
② **Read `index.md`** — learn what pages exist and their summaries.
③ **Scan recent `log.md`** — read the last 20-30 entries to understand recent activity.

```bash
WIKI="${WIKI_PATH:-$HOME/wiki}"
# Orientation reads at session start
read_file "$WIKI/SCHEMA.md"
read_file "$WIKI/index.md"
read_file "$WIKI/log.md" offset=<last 30 lines>
```

Only after orientation should you ingest, query, or lint. This prevents:
- Creating duplicate pages for entities that already exist
- Missing cross-references to existing content
- Contradicting the schema's conventions
- Repeating work already logged

For large wikis (100+ pages), also run a quick `search_files` for the topic
at hand before creating anything new.

## Initializing a New Wiki

When the user asks to create or start a wiki:

1. Determine the wiki path (from `$WIKI_PATH` env var, or ask the user; default `~/wiki`)
2. Create the directory structure above
3. Ask the user what domain the wiki covers — be specific
4. Write `SCHEMA.md` customized to the domain (see template below)
5. Write initial `index.md` with sectioned header
6. Write initial `log.md` with creation entry
7. Confirm the wiki is ready and suggest first sources to ingest

### SCHEMA.md Template

Adapt to the user's domain. The schema constrains agent behavior and ensures consistency:

```markdown
# Wiki Schema

## Domain
[What this wiki covers — e.g., "AI/ML research", "personal health", "startup intelligence"]

## Conventions
- File names: lowercase, hyphens, no spaces (e.g., `transformer-architecture.md`)
- Every wiki page starts with YAML frontmatter (see below)
- Use `[[wikilinks]]` to link between pages (minimum 2 outbound links per page)
- When updating a page, always bump the `updated` date
- Every new page must be added to `index.md` under the correct section
- Every action must be appended to `log.md`
- **Provenance markers:** On pages that synthesize 3+ sources, append `^[raw/articles/source-file.md]`
  at the end of paragraphs whose claims come from a specific source. This lets a reader trace each
  claim back without re-reading the whole raw file. Optional on single-source pages where the
  `sources:` frontmatter is enough.

## Frontmatter
  ```yaml
  ---
  title: Page Title
  created: YYYY-MM-DD
  updated: YYYY-MM-DD
  type: entity | concept | comparison | query | summary
  tags: [from taxonomy below]
  sources: [raw/articles/source-name.md]
  # Optional quality signals:
  confidence: high | medium | low        # how well-supported the claims are
  contested: true                        # set when the page has unresolved contradictions
  contradictions: [other-page-slug]      # pages this one conflicts with
  ---
  ```

`confidence` and `contested` are optional but recommended for opinion-heavy or fast-moving
topics. Lint surfaces `contested: true` and `confidence: low` pages for review so weak claims
don't silently harden into accepted wiki fact.

### raw/ Frontmatter

Raw sources ALSO get a small frontmatter block so re-ingests can detect drift:

```yaml
---
source_url: https://example.com/article   # original URL, if applicable
ingested: YYYY-MM-DD
sha256: <hex digest of the raw content below the frontmatter>
---
```

The `sha256:` lets a future re-ingest of the same URL skip processing when content is unchanged,
and flag drift when it has changed. Compute over the body only (everything after the closing
`---`), not the frontmatter itself.

## Tag Taxonomy
[Define 10-20 top-level tags for the domain. Add new tags here BEFORE using them.]

Example for AI/ML:
- Models: model, architecture, benchmark, training
- People/Orgs: person, company, lab, open-source
- Techniques: optimization, fine-tuning, inference, alignment, data
- Meta: comparison, timeline, controversy, prediction

Rule: every tag on a page must appear in this taxonomy. If a new tag is needed,
add it here first, then use it. This prevents tag sprawl.

## Page Thresholds
- **Create a page** when an entity/concept appears in 2+ sources OR is central to one source
- **Add to existing page** when a source mentions something already covered
- **DON'T create a page** for passing mentions, minor details, or things outside the domain
- **Split a page** when it exceeds ~200 lines — break into sub-topics with cross-links
- **Archive a page** when its content is fully superseded — move to `_archive/`, remove from index

## Entity Pages
One page per notable entity. Include:
- Overview / what it is
- Key facts and dates
- Relationships to other entities ([[wikilinks]])
- Source references

## Concept Pages
One page per concept or topic. Include:
- Definition / explanation
- Current state of knowledge
- Open questions or debates
- Related concepts ([[wikilinks]])

## Comparison Pages
Side-by-side analyses. Include:
- What is being compared and why
- Dimensions of comparison (table format preferred)
- Verdict or synthesis
- Sources

## Update Policy
When new information conflicts with existing content:
1. Check the dates — newer sources generally supersede older ones
2. If genuinely contradictory, note both positions with dates and sources
3. Mark the contradiction in frontmatter: `contradictions: [page-name]`
4. Flag for user review in the lint report
```

### index.md Template

The index is sectioned by type. Each entry is one line: wikilink + summary.

```markdown
# Wiki Index

> Content catalog. Every wiki page listed under its type with a one-line summary.
> Read this first to find relevant pages for any query.
> Last updated: YYYY-MM-DD | Total pages: N

## Entities
<!-- Alphabetical within section -->

## Concepts

## Comparisons

## Queries
```

**Scaling rule:** When any section exceeds 50 entries, split it into sub-sections
by first letter or sub-domain. When the index exceeds 200 entries total, create
a `_meta/topic-map.md` that groups pages by theme for faster navigation.

### log.md Template

```markdown
# Wiki Log

> Chronological record of all wiki actions. Append-only.
> Format: `## [YYYY-MM-DD] action | subject`
> Actions: ingest, update, query, lint, create, archive, delete
> When this file exceeds 500 entries, rotate: rename to log-YYYY.md, start fresh.

## [YYYY-MM-DD] create | Wiki initialized
- Domain: [domain]
- Structure created with SCHEMA.md, index.md, log.md
```

## Core Operations

### 1. Ingest

When the user provides a source (URL, file, paste), integrate it into the wiki.

#### 0. Discover uningested documents (when no specific source is named)

When the user says "ingest new documents" / "摄入新文档" without naming specific files,
run a discovery pass BEFORE the numbered steps below. Three signals reveal uningested content:

1. **Files in `raw/` missing frontmatter** — `search_files(target='files')` across `raw/`,
   then `read_file` each file's first 5 lines. Files without `source_url:` + `sha256:`
   frontmatter were placed there but never ingested.

2. **Files in `raw/` not referenced in `log.md`** — cross-reference the `raw/` file listing
   against recent log entries. A file present on disk but absent from the log was added by
   the user between sessions.

3. **Files in known source directories** (e.g., `RespSearch/`, `Downloads/`) not yet copied
   to `raw/` — `search_files(target='files')` on the source directory, compare against
   `raw/papers/`. Especially check subdirectories (`downloads/`) that may have been skipped
   in earlier batch ingests.

After discovery, report what you found to the user and confirm scope before proceeding.
For each discovered file, follow steps ①-⑥ below.

① **Capture the raw source:**
   - URL → use `web_extract` to get markdown, save to `raw/articles/`
   - PDF (URL, e.g. arxiv) → use `web_extract` (handles PDF URLs), save to `raw/papers/`
     For bulk-downloading an entire paper's bibliography, see `references/bulk-arxiv-refs.md`.
     For converting placeholder stubs to real content, see `references/reingest-stubs.md`.
   - PDF (local file) → see `references/pdf-extraction.md` — use pymupdf via
     execute_code to extract text, save the original PDF + extracted `.md` side-by-side
     in `raw/papers/`
   - Pasted text → save to appropriate `raw/` subdirectory
   - Image (screenshot/diagram, pasted in chat or placed in `raw/screenshots/`) →
     see `references/image-ingest.md` — use `vision_analyze` to extract a text
     description, save the original `.png` + extracted `.md` side-by-side in
     `raw/screenshots/`. A screenshot is a first-class source, NOT a passive
     `raw/assets/` attachment.
   - Audio (voice memo/recording, placed in `raw/transcripts/`) →
     see `references/audio-ingest.md` — there is NO transcribe tool; drive
     faster-whisper (local, in the Hermes venv) via `code_execution`, save the
     original audio + transcript `.md` side-by-side in `raw/transcripts/`.
   - Name the file descriptively: `raw/articles/karpathy-llm-wiki-2026.md`
   - **Add raw frontmatter** (`source_url`, `ingested`, `sha256` of the body).
     On re-ingest of the same URL: recompute the sha256, compare to the stored value —
     skip if identical, flag drift and update if different. This is cheap enough to
     do on every re-ingest and catches silent source changes.

② **Discuss takeaways** with the user — what's interesting, what matters for
   the domain. (Skip this in automated/cron contexts — proceed directly.)

③ **Check what already exists** — search index.md and use `search_files` to find
   existing pages for mentioned entities/concepts. This is the difference between
   a growing wiki and a pile of duplicates.

④ **Write or update wiki pages:**
   - **New entities/concepts:** Create pages only if they meet the Page Thresholds
     in SCHEMA.md (2+ source mentions, or central to one source)
   - **Existing pages:** Add new information, update facts, bump `updated` date.
     When new info contradicts existing content, follow the Update Policy.
   - **Cross-reference:** Every new or updated page must link to at least 2 other
     pages via `[[wikilinks]]`. Check that existing pages link back.
   - **Tags:** Only use tags from the taxonomy in SCHEMA.md
   - **Provenance:** On pages synthesizing 3+ sources, append `^[raw/articles/source.md]`
     markers to paragraphs whose claims trace to a specific source.
   - **Confidence:** For opinion-heavy, fast-moving, or single-source claims, set
     `confidence: medium` or `low` in frontmatter. Don't mark `high` unless the
     claim is well-supported across multiple sources.

⑤ **Update navigation:**
   - Add new pages to `index.md` under the correct section, alphabetically
   - Update the "Total pages" count and "Last updated" date in index header
   - Append to `log.md`: `## [YYYY-MM-DD] ingest | Source Title`
   - List every file created or updated in the log entry

⑥ **Report what changed** — list every file created or updated to the user.

A single source can trigger updates across 5-15 wiki pages. This is normal
and desired — it's the compounding effect.

### 2. Query

When the user asks a question about the wiki's domain:

> **CLOSED-WORLD — answer ONLY from the wiki (no web).** A query is answered from the wiki's
> own pages + their `raw/` sources. **Do NOT `web_search`, `web_extract`, or browse** during a
> query — the wiki IS the source of truth, and every claim must cite a `raw/...` note (see
> `references/citation-format.md`). A web result has no raw-note path, can't be cited, breaks
> provenance, and turns the wiki back into plain RAG. If the wiki doesn't cover something, **say
> so** (or mark `[来源不明]`) and offer to **ingest** a source for it — web access belongs to
> Ingest, never Query.
>
> **Stubs count as "not covered".** A `raw/` note that has frontmatter but an empty or
> placeholder body — e.g. only a title + `*Content extracted via web_extract — full text
> available at the source URL.*` — is NOT real wiki content. Treat it as unknown: mark the claim
> `[桩未摄入 / stub — not ingested]`, name the paper that needs re-ingesting (give its arXiv
> id / URL), and **NEVER backfill the gap from your own model / parametric memory.** An
> unsourced fabrication that *looks* cited is the single worst failure mode here — an honest gap
> is always better. Completing the stub's body is an **Ingest** action, done separately, never
> mid-query.

① **Read `index.md`** to identify relevant pages.
② **For wikis with 100+ pages**, also `search_files` across all `.md` files
   for key terms — the index alone may miss relevant content.
③ **Read the relevant pages** using `read_file`.
④ **Synthesize an answer** from the compiled knowledge. Cite the wiki pages
   you drew from: "Based on [[page-a]] and [[page-b]]..."
   **Note-level provenance (required):** every factual claim must ALSO carry the
   raw source note it traces to. Hermes chat renders everything as plain text (no
   clickable links), so the anchor is a copy-pasteable absolute native path — NOT
   an `^[...]` footnote (page-internal) and NOT a `file:///` link (does not render).
   Read each relevant page's `sources:` / `^[raw/...]` first to get the real
   raw-note filenames, put a numbered marker `[1]` after each claim, and end the
   answer with a `## Sources` block — one line per marker, formatted as:
     `[1]` `[[entity-page]]` · `` `<WIKI_PATH>\raw\papers\<file>.md` `` · §x.y (optional short quote)
   EVERY entry must carry its own backticked absolute native path — do not write a
   bare "同上"/"same"/"ditto" that omits the path; repeat it. If a claim is
   supported by a Layer-2 page (query/comparison/concept), follow that page's
   `sources:` down to the underlying raw note and cite the raw-note path — never
   stop at the Layer-2 page. Raw filenames contain `[hash]` brackets, so keep the
   path in backticks, never in `[[ ]]` or `^[ ]`. A bare section quote with no
   raw-note path is not acceptable. Full spec + example: `references/citation-format.md`.
⑤ **File valuable answers back** — if the answer is a substantial comparison,
   deep dive, or novel synthesis, create a page in `queries/` or `comparisons/`.
   Don't file trivial lookups — only answers that would be painful to re-derive.
⑥ **Update log.md** with the query and whether it was filed.

### 3. Lint

When the user asks to lint, health-check, or audit the wiki:

> **巡检 / audit — RUN THE CANONICAL SCRIPT, do NOT hand-roll one.** All 6 checks are already
> implemented and debugged in **`scripts/audit.py`**. Run it via `execute_code`:
> `python scripts/audit.py "<WIKI_PATH>"` (or paste its contents and set `WIKI`). Re-authoring an
> ad-hoc audit script per session is the source of recurring bugs (`NameError`, GBK/emoji
> `UnicodeEncodeError`, empty-taxonomy false-positive storms) — don't. If the script is missing
> or a check needs extending, **edit `scripts/audit.py` and keep it** rather than writing a throwaway.
> The script's logic mirrors `references/audit-checklist.md` (read it for the rationale behind each
> check) and the generic steps below — those are the SPEC, the script is the implementation.

① **Orphan pages:** Find pages with no inbound `[[wikilinks]]` from other pages.
```python
# Use execute_code for this — programmatic scan across all wiki pages
import os, re
from collections import defaultdict
wiki = "<WIKI_PATH>"
# Scan all .md files in entities/, concepts/, comparisons/, queries/
# Extract all [[wikilinks]] — build inbound link map
# Pages with zero inbound links are orphans
```

② **Broken wikilinks:** Find `[[links]]` that point to pages that don't exist.

> ⚠️ **Wikilink parsing:** When extracting wikilinks for lint, the regex MUST strip
> `#` anchors and `|` aliases before checking existence. `[[stable-diffusion#架构图详解]]`
> is valid (it links to page `stable-diffusion` with anchor `架构图详解`); do NOT flag
> `stable-diffusion#架构图详解` as a broken link. Similarly, `[[page|alias]]` should
> resolve to `page`. The correct pattern: extract the raw target, split on `#` then `|`,
> keep only the first segment as the page name.

③ **Index completeness:** Every wiki page should appear in `index.md`. Compare
   the filesystem against index entries. **Also check the header count** — the
   `总页数: N` or `Total pages: N` in the index header line often drifts from the
   actual count of listed pages. Report both discrepancies separately (missing
   entries vs stale count).

④ **Frontmatter validation:** Every wiki page must have all required fields
   (title, created, updated, type, tags, sources). Tags must be in the taxonomy.

⑤ **Stale content:** Pages whose `updated` date is >90 days older than the most
   recent source that mentions the same entities.

⑥ **Contradictions:** Pages on the same topic with conflicting claims. Look for
   pages that share tags/entities but state different facts. Surface all pages
   with `contested: true` or `contradictions:` frontmatter for user review.

⑦ **Quality signals:** List pages with `confidence: low` and any page that cites
   only a single source but has no confidence field set — these are candidates
   for either finding corroboration or demoting to `confidence: medium`.

⑧ **Source drift:** For each file in `raw/` with a `sha256:` frontmatter, recompute
   the hash and flag mismatches. Mismatches indicate the raw file was edited
   (shouldn't happen — raw/ is immutable) or ingested from a URL that has since
   changed. Not a hard error, but worth reporting. When many files show drift
   simultaneously, use the bulk-fix script in `references/sha256-bulk-fix.md`
   to recompute all hashes in one pass.

⑨ **Page size:** Flag pages over 200 lines — candidates for splitting.

⑩ **Tag audit:** List all tags in use, flag any not in the SCHEMA.md taxonomy.

⑪ **Log rotation:** If log.md exceeds 500 entries, rotate it.

⑫ **Report findings** with specific file paths and suggested actions, grouped by
   severity (broken links > orphans > source drift > contested pages > stale content > style issues).

⑬ **Append to log.md:** `## [YYYY-MM-DD] lint | N issues found`

## Working with the Wiki

### Searching

```bash
# Find pages by content
search_files "transformer" path="$WIKI" file_glob="*.md"

# Find pages by filename
search_files "*.md" target="files" path="$WIKI"

# Find pages by tag
search_files "tags:.*alignment" path="$WIKI" file_glob="*.md"

# Recent activity
read_file "$WIKI/log.md" offset=<last 20 lines>
```

### Bulk Ingest

When ingesting multiple sources at once, batch the updates:
1. Read all sources first
2. Identify all entities and concepts across all sources
3. Check existing pages for all of them (one search pass, not N)
4. Create/update pages in one pass (avoids redundant updates)
5. Update index.md once at the end
6. Write a single log entry covering the batch

### Archiving

When content is fully superseded or the domain scope changes:
1. Create `_archive/` directory if it doesn't exist
2. Move the page to `_archive/` with its original path (e.g., `_archive/entities/old-page.md`)
3. Remove from `index.md`
4. Update any pages that linked to it — replace wikilink with plain text + "(archived)"
5. Log the archive action

### Obsidian Integration

The wiki directory works as an Obsidian vault out of the box:
- `[[wikilinks]]` render as clickable links
- Graph View visualizes the knowledge network
- YAML frontmatter powers Dataview queries
- The `raw/assets/` folder holds images referenced via `![[image.png]]`

For best results:
- Set Obsidian's attachment folder to `raw/assets/`
- Enable "Wikilinks" in Obsidian settings (usually on by default)
- Install Dataview plugin for queries like `TABLE tags FROM "entities" WHERE contains(tags, "company")`

If using the Obsidian skill alongside this one, set `OBSIDIAN_VAULT_PATH` to the
same directory as the wiki path.

### Obsidian Headless (servers and headless machines)

On machines without a display, use `obsidian-headless` instead of the desktop app.
It syncs vaults via Obsidian Sync without a GUI — perfect for agents running on
servers that write to the wiki while Obsidian desktop reads it on another device.

**Setup:**
```bash
# Requires Node.js 22+
npm install -g obsidian-headless

# Login (requires Obsidian account with Sync subscription)
ob login --email <email> --password '<password>'

# Create a remote vault for the wiki
ob sync-create-remote --name "LLM Wiki"

# Connect the wiki directory to the vault
cd ~/wiki
ob sync-setup --vault "<vault-id>"

# Initial sync
ob sync

# Continuous sync (foreground — use systemd for background)
ob sync --continuous
```

**Continuous background sync via systemd:**
```ini
# ~/.config/systemd/user/obsidian-wiki-sync.service
[Unit]
Description=Obsidian LLM Wiki Sync
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/path/to/ob sync --continuous
WorkingDirectory=/home/user/wiki
Restart=on-failure
RestartSec=10

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now obsidian-wiki-sync
# Enable linger so sync survives logout:
sudo loginctl enable-linger $USER
```

This lets the agent write to `~/wiki` on a server while you browse the same
vault in Obsidian on your laptop/phone — changes appear within seconds.

## Pitfalls

- **Never modify files in `raw/`** — sources are immutable. Corrections go in wiki pages.
- **Always orient first** — read SCHEMA + index + recent log before any operation in a new session.
  Skipping this causes duplicates and missed cross-references.
- **Always update index.md and log.md** — skipping this makes the wiki degrade. These are the
  navigational backbone.
- **Don't create pages for passing mentions** — follow the Page Thresholds in SCHEMA.md. A name
  appearing once in a footnote doesn't warrant an entity page.
- **Don't create pages without cross-references** — isolated pages are invisible. Every page must
  link to at least 2 other pages.
- **Frontmatter is required** — it enables search, filtering, and staleness detection.
- **Tags must come from the taxonomy** — freeform tags decay into noise. Add new tags to SCHEMA.md
  first, then use them.
- **Keep pages scannable** — a wiki page should be readable in 30 seconds. Split pages over
  200 lines. Move detailed analysis to dedicated deep-dive pages.
- **Ask before mass-updating** — if an ingest would touch 10+ existing pages, confirm
  the scope with the user first.
- **Rotate the log** — when log.md exceeds 500 entries, rename it `log-YYYY.md` and start fresh.
  The agent should check log size during lint.
- **Handle contradictions explicitly** — don't silently overwrite. Note both claims with dates,
  mark in frontmatter, flag for user review.
- **Windows path resolution** — on Windows, always use native paths (e.g., `<WIKI_PATH>` like `D:\wiki\...`)
  with `write_file`. Bash paths like `/d/wiki/...` resolve inconsistently between
  `terminal` (where `/d/` → `D:\`) and `write_file` (where it may become `D:\d\...`).
  Same applies to `read_file`, `search_files`, and `patch`. Use the `D:\` form for all
  wiki file operations.
- **Wikilink `#` anchors are NOT broken links** — `[[page#heading]]` is a valid Obsidian
  wikilink with an anchor. Lint must strip `#...` and `|alias` before checking if the
  target page exists. A naive regex that treats the full string as a page slug will
  produce false positives.
- **Sha256 drift is often systemic, not per-file** — when many raw files show sha256
  mismatches simultaneously, suspect a systemic cause (placeholder hashes from initial
  ingest, line-ending changes, or tool-level encoding differences). Fix in bulk with
  the script in `references/sha256-bulk-fix.md` rather than one-by-one.
- **arXiv `web_extract` truncation** — arXiv URL formats have very different behaviour:
  - **`/abs/<id>`** — returns ONLY a metadata landing page (navigation, submission history,
    bibtex). Useless for content extraction; do NOT use.
  - **`/html/<id>` or `/html/<id>v1`** — often 404s (especially on newer papers).
  - **`/pdf/<id>`** — works reliably, returns LLM-summarized markdown (~3-5K chars). This
    is the PRIMARY format for ingest.
  URL priority order: `/pdf/<id>` first. If it fails, try CVF open-access pages for CVPR/ICCV
  papers (`openaccess.thecvf.com/content_<VENUE>/html/...`). As last resort, use `web_search`
  for third-party summaries and write the result into the raw note's body. Always recompute
  `sha256` after updating body content.
  Stubs (`XXX [NN] - Title [arxivid].md`, no body text) are born from bulk bibliography
  downloads; Check 2 of the audit-checklist flags them. To convert stubs to real content,
  see `references/reingest-stubs.md`. **Web supplementation is an Ingest-only action — NEVER
  do it during a Query.** The Query path is closed-world: a stub encountered at query time
  is reported as a gap (see the Query section), not silently filled from the web.

## Related Tools

[llm-wiki-compiler](https://github.com/atomicmemory/llm-wiki-compiler) is a Node.js CLI that
compiles sources into a concept wiki with the same Karpathy inspiration. It's Obsidian-compatible,
so users who want a scheduled/CLI-driven compile pipeline can point it at the same vault this
skill maintains. Trade-offs: it owns page generation (replaces the agent's judgment on page
creation) and is tuned for small corpora. Use this skill when you want agent-in-the-loop curation;
use llmwiki when you want batch compile of a source directory.
