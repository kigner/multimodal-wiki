# Citation Format for Query Answers (note-level provenance)

Default Query behavior (Core Operations → Query, step ④) only cites `[[page]]`
wikilinks: "Based on [[page-a]]...". That tells the reader *which wiki page* an
answer used, but not *which source paper* a specific claim came from — so answers
end up with floating quotes like `§3.3: "..."` and no path back to the original.

This doc upgrades Query answers to **note-level provenance**: every factual claim
carries the raw source note it traces to, as a copy-pasteable absolute path.

## Host reality (decides the format)

**Hermes chat renders everything as plain text — nothing is clickable.** Tested:
`[[wikilinks]]`, bare paths, and `file:///` links all show as inert text. So:

- The anchor is a **copy-pasteable absolute native path** the reader can paste into
  Explorer / an editor. This is the required, load-bearing part.
- **Do NOT emit `file:///` links** — they don't render and only add noise.
- **Do NOT use `^[raw/...]` footnotes** in answers — page-internal, also plain text.
- Click-to-open lives in **Obsidian** (open the same vault): the `[[entity-page]]`
  wikilink in each source line is the jump-in point for the graph.

## Rules

1. **Trace before you write.** For each relevant Layer-2 page, read its `sources:`
   frontmatter and inline `^[raw/...]` markers — these hold the real raw-note
   filenames. Map each claim to the specific raw note it comes from. Never invent a
   path or a section number.

2. **Every claim carries a source.** No statement of fact may appear without a
   source. A bare section quote with no path (e.g. `§3.3: "uses a CLIP text
   encoder"`) is **not acceptable** — that is the exact failure this doc prevents.

3. **Source line format** — one per numbered marker, in a `## Sources` block
   (localize the heading, e.g. `## 来源`):

   ```
   [1] [[entity-page]] · `D:\res_wiki\raw\papers\<file>.md` · §x.y (optional short quote)

   [2] [[entity-page]] · `D:\res_wiki\raw\papers\<other>.md` · §x.y
   ```

   - **One BLANK LINE between every entry (hard break — required).** Hermes strips/reflows
     markdown on the final reply (`display.final_response_markdown: strip`), so a *single*
     newline merges into the previous line and `[1][2][3]…` collapse onto ONE line. A blank
     line forces each `[n]` onto its own line. Never put entries on consecutive single-newline lines.
   - **`[[entity-page]]`** — the Layer-2 page, for Obsidian navigation.
   - **Backticked absolute native path** — *required, the anchor.* Native Windows
     path per `pdf-extraction.md`. Raw filenames contain `[hash]` brackets, so keep
     the path in backticks — never wrap it in `[[ ]]` or `^[ ]`.
   - **§x.y** — *only if it actually appears in the raw note.* Sections are
     best-effort: ingestion concatenates page text without reliable section markers,
     so never manufacture a section you cannot see.

4. **Every entry repeats its own path.** Do NOT write a bare "同上" / "same" /
   "ditto" that omits the path. If two claims share a source, repeat the full
   backticked path on each line — it's cheap and keeps every line self-resolving.

5. **Resolve Layer-2 to the raw note.** If a claim is supported by a Layer-2 page
   (a `queries/`, `comparisons/`, or `concepts/` page rather than a paper), follow
   that page's `sources:` down to the underlying raw note and cite the **raw-note
   path** — never stop the chain at the Layer-2 page. (Failure example: citing
   `[[arc2face-improvements]] §2.2` with no `raw/papers/...md` behind it.)

6. **Untraceable claims.** If a claim cannot be traced to a specific raw note, drop
   it or mark it `[来源不明]`. Do not fabricate a citation to fill the slot.

## Example

```
Arc2Face 仅用 ArcFace 身份向量驱动生成，不需要文本提示 [1]。它发现 FFHQ 等常用
数据集身份多样性不足，改用 WebFace42M 上采样构建训练集 [2]。

## 来源
[1] [[arc2face]] · `D:\res_wiki\raw\papers\Arc2Face A Foundation Model for ID-Consistent Human Faces [353538b9].md` · §Abstract

[2] [[arc2face]] · `D:\res_wiki\raw\papers\Arc2Face A Foundation Model for ID-Consistent Human Faces [353538b9].md` · §3.1 ("we meticulously upsample ... WebFace42M")
```

## Key points

- **The absolute native path is the contract.** Hermes shows it as text, but the
  reader can copy it to open the file. Provenance must survive with nothing
  clickable — so the path is mandatory and `file:///`/`^[]` forms are not used.
- **Note-level, not page-level:** the anchor is the source paper's `.md` in
  `raw/papers/`. From that raw note the reader reaches the original `.pdf` beside it.
- **Don't invent.** Copy paths verbatim from `sources:` / `^[...]`; copy sections
  only if present. An honest "[来源不明]" beats a fabricated citation.
