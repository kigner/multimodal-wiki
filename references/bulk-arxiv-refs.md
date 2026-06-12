# Bulk arXiv Reference Download

When the user asks to download all papers that a wiki-ingested paper references,
or to populate `raw/papers/` with the bibliography of a key source, follow this workflow.

## Workflow

### 1. Locate arXiv IDs from the source paper's bibliography

The source paper's raw extract is in `raw/papers/<source>.md`. Read its references
section (usually the last ~300 lines). Each entry gives: author, title, venue, and
often an arXiv ID in the form `arXiv:YYMM.NNNNN`.

For any reference without an explicit arXiv ID, search via the arxiv skill:
```
search_query=all:"<paper title keywords>"+<first author last name>
```

### 2. Batch-download via web_extract

`web_extract(urls=[...])` handles arXiv PDF URLs natively — pass up to 5 per call:
```
https://arxiv.org/pdf/<id>
```

Returns summarized markdown. This is sufficient for reference indexing; the full
PDF is always linked via the source URL in frontmatter.

### 3. Save stubs to raw/papers/

Each downloaded paper gets a stub `.md` file with:
```yaml
---
source_url: https://arxiv.org/abs/<id>
ingested: YYYY-MM-DD
arxiv_id: <id>
ref_label: <Paper Name> [<ref number from source>]
---
```

File naming convention:
```
<RefLabel> - <ShortTitle> [<arxiv_id>].md
```
For example: `Stable Diffusion [62] - High-Resolution Image Synthesis with Latent Diffusion [2112.10752].md`

**Note on content:** `web_extract` on arXiv PDFs returns LLM-summarized markdown
(since papers are typically >5000 chars). This is acceptable for reference indexing.
The full text is always available at the `source_url`. These stubs are NOT the same
as full paper extracts from local PDFs (which use pymupdf via `references/pdf-extraction.md`).

### 4. Log the batch

Single log entry covering all downloaded papers, grouped by category:
```markdown
## [YYYY-MM-DD] ingest | <Source Paper> 引用论文 (arXiv 批量下载)
- 来源: <source paper> 论文参考文献列表
- 入库: raw/papers/ (N 篇 arXiv 论文)
- 论文清单:
  架构基座: ...
  方法前提: ...
  对比基线: ...
```

### 5. Not found handling

Some papers (conference-only, non-arXiv venues like SIGGRAPH) won't be on arXiv.
Note them explicitly in the log entry.

## When This is Triggered
- "把这篇论文引用的论文下载下来"
- "download all papers referenced by X"
- User points to a paper in the wiki and wants its bibliography ingested
