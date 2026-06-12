# Wiki Audit Checklist (巡检 / audit)

Use this when the user asks to **巡检 / audit / health-check** their wiki. Run all checks and
produce a structured report graded 🔴 (broken) / 🟡 (needs attention) / ✅ (ok), grouped by
severity. This is the concrete, vault-tuned companion to the generic Lint steps in `SKILL.md`
— run both.

## Before you start
- Resolve the vault path (`WIKI_PATH`, memory, or ask).
- **Orient first**: read `SCHEMA.md` (conventions, tag taxonomy, log/index format), then
  `index.md`, then recent `log.md`. The checks below validate the vault AGAINST its own SCHEMA
  — don't assume a fixed format, read what this vault declares.

## Check 1 — Index consistency
- List all `.md` under `entities/ concepts/ comparisons/ queries/` (`search_files target=files`).
- Parse `index.md`: count listed pages and the declared total (e.g. `总页数: N` / `Total pages: N`).
- Cross-check both directions: a page on disk missing from the index, and an index entry with no file.
- Common root cause of mismatch: an early batch log entry undercounted the pages actually created.

## Check 2 — Uningested raw sources
- List everything under `raw/` recursively (`articles/ papers/ transcripts/ screenshots/ assets/`).
- Parse `log.md` for ingest references (e.g. `入库: raw/...` / `ingest | ...`).
- Flag raw files not referenced in the log. Pay special attention to **non-`.md` sources with no
  sibling extract** (binary sources can't carry frontmatter, so "no sibling `.md`" = uningested):
  - a `.pdf` in `raw/papers/` with no matching `.md`
  - a `.png` in `raw/screenshots/` with no matching `.md` (image not read into a note yet)
  - an audio file in `raw/transcripts/` with no matching `.md` (not transcribed yet)
  - a `.txt` URL-stub never fetched/expanded
  - a **text extract (`.md`) with frontmatter but an empty / placeholder body** — e.g. only a
    title + `*Content extracted via web_extract — full text available at the source URL.*`.
    It carries a *real* `sha256` (computed over the placeholder) so it silently passes Check 3,
    yet it is NOT actually ingested. Detect by `grep`-ing `raw/` for that placeholder line and
    for notes whose body is under ~5 non-frontmatter lines. The classic offenders are the
    **bulk-bibliography stubs** (`XXX [NN] - Title [arxivid].md`, no sibling `.pdf`, born from
    arXiv `web_extract` truncation). Flag each 🟡 **"stub — re-ingest needed"** with its arXiv
    id so an Ingest pass can fill the body (web supplementation is Ingest-only; Query is
    closed-world and reports stubs as gaps).
- **Audio naming mismatch**: when an audio file (e.g. `audio.wav`) has a transcript `.md` with a
  *different* basename (e.g. `id2reflectance-voice-memo.md`), the script-only `replace('.wav','.md')`
  check will false-positive — manually verify that a sibling transcript exists under any name.
  Still flag the mismatch as 🟡 hygiene (rename audio to match the transcript for grep-ability).

## Check 3 — Frontmatter quality
- Every Layer-2 page: YAML frontmatter present with all required fields per SCHEMA (typically
  `title, created, updated, type, tags, sources`, plus `confidence`).
- `type` ∈ {entity, concept, comparison, query, summary}; `tags` non-empty and all in the SCHEMA taxonomy.
- **⚠️ YAML list pitfall:** When programmatically parsing frontmatter with a naive `split(":", 1)` parser,
  tags formatted as `[gan, diffusion, seminal]` will retain the `[` / `]` brackets in the value.
  Always strip `[` `]` before splitting tag lists on `,` and checking against taxonomy, otherwise
  every page will false-positive as "non-taxonomy tags".
- Every `raw/` source note: `source`/`source_url`, `ingested`, and a **real** `sha256` — flag a
  literal `<placeholder>` or an obviously fake sequential hash (e.g. `a1b2c3d4…`).
- **Binary files are expected:** PDFs, PNGs, JPGs, and WAVs under `raw/` cannot carry YAML
  frontmatter. Do NOT flag them as "raw no frontmatter" — only flag text files (`.md`, `.txt`)
  that lack frontmatter. The frontmatter for binary sources lives in their sibling `.md` extract.

## Check 4 — Wikilink integrity
- Extract every `[[wikilink]]`; verify each target page exists on disk.
- Verify each page has ≥ 2 outbound wikilinks (orphans / under-linked pages → 🟡).

## Check 5 — File hygiene & convention split
- Duplicate images (same content, different location). Compute sha256 of same-named files in
  different directories to confirm identity before deleting.
- Filename typos — check for common misspellings: `pipline`→`pipeline`, `architecure`→`architecture`,
  `facelivt`→`face-livt` (but verify against SCHEMA convention first — lowercase compound names
  like `facelivt` may be correct). Also `search_files` for the typo pattern across all `.md`
  pages to find and update stale references after renaming.
- **Convention split** (multimodal-wiki):
  - source **screenshots/diagrams** → `raw/screenshots/` (+ a sibling `.md` extract)
  - **display/figure images** embedded in Layer-2 pages → `raw/assets/`
  - **audio** → `raw/transcripts/` (+ a sibling `.md` transcript)
  - Flag images in the wrong place — e.g. a source screenshot dumped straight into `assets/`,
    or an image embedded into a Layer-2 page with no raw note behind it.
- `.txt`/`.pdf` raw files that should have a `.md` extract beside them.

## Check 6 — Log & index format
- Each `log.md` entry follows the SCHEMA's declared format.
- Flag batch entries like `raw/papers/ (N 篇)` that omit individual file paths — they break
  grep-based cross-referencing (Check 2).

## Report
Group findings by severity (🔴 broken links / missing files > 🟡 orphans, source drift, hygiene
> ✅ clean), with specific file paths and a suggested fix per item. End with a one-line summary
and append a `## [YYYY-MM-DD] lint | N issues found` entry to `log.md`.

## Windows / tooling notes
- On Windows + MSYS bash, prefer `terminal` (find/grep) over `execute_code` for file
  cross-referencing — the sandbox's `search_files` can hit JSON parse errors on some paths.
- `log.md` entries can be multi-line; `read_file` the log structure first before grepping.
- **⚠️ Regex pitfall — paths with spaces:** When programmatically extracting file paths from
  `log.md` with a regex like `raw/\S+\.\w+`, `\S+` stops at the first space and silently
  truncates paths like `raw/papers/ArcFace [14] - ... .md` to `raw/papers/ArcFace`. Use
  `[^\n]+?` (match non-newline, non-greedy) instead: pattern
  `raw/(?:articles|papers|transcripts|screenshots|assets)/[^\n]+?\.(md|pdf|png|jpg|jpeg|wav|txt)`.
  This also catches paired entries like `file.pdf / .md` where the `.md` side is right of the
  first extension match. Also check indent-prefixed paths (`- raw/...`) with a second pass of
  `^-\s+(raw/...)`.
- **Programmatic audit:** a comprehensive Python audit script covering all 6 checks in one
  pass is available at `scripts/audit.py`. Run it from `execute_code` with the wiki path set,
  or as a standalone script.
