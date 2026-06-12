# Re-ingesting Stub Files

When a lint reveals placeholder stubs in `raw/papers/` (frontmatter only, no body text, or
`*Content extracted via web_extract — full text available at the source URL.*`), convert them
to real content with this workflow.

## 1. Identify stubs

Stubs are born from:
- **Bulk arXiv bibliography downloads** (`references/bulk-arxiv-refs.md`): `web_extract` on
  `/pdf/` URLs returns LLM-summarized content that's often truncated. The raw note gets
  frontmatter + `sha256` but an empty or placeholder body.
- **Failed extractions**: `/abs/` returns metadata only; `/html/` often 404s.

Lint detection: `search_files` for files with `sha256` but fewer than ~5 non-frontmatter
body lines, plus the `*Content extracted via*` placeholder string.

## 2. Extract arXiv IDs from filenames

Stubs follow the naming convention:
```
<Author> [<refnum>] - <ShortTitle> [<arxiv_id>].md
```
Parse the `[<arxiv_id>]` suffix (e.g., `[2112.10752]`) from the filename.

## 3. Fetch real content

**URL priority order:**

| Priority | URL | Behaviour | When to use |
|----------|-----|-----------|-------------|
| 1st | `https://arxiv.org/pdf/<id>` | LLM-summarized markdown (~3-5K chars) | **Default — works for most papers** |
| 2nd | CVF open-access page | Full abstract + metadata | CVPR/ICCV/ECCV papers that fail `/pdf/` |
| 3rd | `web_search` + `web_extract` on blog/article | Third-party summaries | Last resort when URLs fail |

**Batch with `web_extract(urls=[...])`** — pass up to 5 `/pdf/` URLs per call for efficiency.

Example: 21 stubs → 5 batches of 5+5+5+4+2.

### CVF fallback pattern

For CVPR papers: `https://openaccess.thecvf.com/content_CVPR_<YEAR>/html/<Author>_<Title>_CVPR_<YEAR>_paper.html`

For ICCV: same pattern with `ICCV`.

This returns the abstract and metadata, sufficient for a reference-level summary.

### web_search fallback

When both URL formats fail, search for the paper title + "summary" or "explained":
```
web_search("<paper title> <first author> summary key contributions")
```
Extract the top blog/article result and use it as the body.

## 4. Write body content

Each stub gets a condensed summary covering:
- **Title, Authors, Venue/Year**
- **Abstract** (1-2 sentences)
- **Key Contributions** (bullet list)
- **Method** (core technical approach, 3-5 bullets)
- **Significance/Impact** (why it matters in the wiki's domain context)

Keep each summary scannable (~15-25 lines). The full paper is always available at the
source URL — the raw note is a reference index, not a replacement for reading the paper.

## 5. Update sha256 and write

Use `execute_code` to batch-process:
```python
import hashlib, re, os

# For each stub file:
# 1. Read current content, extract frontmatter
# 2. Replace body with new content
# 3. Recompute sha256 over body only
# 4. Update sha256 in frontmatter: sha256: <new_hash>
# 5. Write file

sha = hashlib.sha256(new_body.encode('utf-8')).hexdigest()
new_content = re.sub(r'sha256:.*', f'sha256: {sha}', new_content)
```

**Do NOT recompute sha256 over frontmatter + body** — only the body. This matches the
original ingest convention and prevents false drift detection.

## 6. Log the re-ingest

Single log entry covering the batch, grouped by category:

```markdown
## [YYYY-MM-DD] ingest | 重新摄入 N 篇 arXiv 引用论文存根
- 来源: [date] <original batch> 的 N 个存根
- 方法: web_extract /pdf/ + CVF 补充
- 类别分组:
  基础架构 (M): paper1, paper2, ...
  方法前提 (M): paper3, ...
- 更新: raw/papers/ 中 N 个 .md 文件（正文替换 + sha256 重算）
- 验证: N/N 全部通过（body 行数 M-N，sha256 已重算）
```

## Pitfalls

- **`/abs/` is useless for content** — it only returns the arXiv landing page (navigation,
  submission history, bibtex). Never use it for re-ingest; go straight to `/pdf/`.
- **`/html/` often 404s** — arXiv HTML rendering is inconsistent. Don't spend time
  debugging 404s; fall through to web_search.
- **sha256 must be body-only** — recomputing over frontmatter + body creates a different
  hash than the original ingest, triggering false drift in future lints.
- **Batch writes via execute_code** — writing 21 files one-by-one via `write_file` is
  very slow. Use `execute_code` with file I/O for bulk updates.
- **Do NOT create new Layer-2 pages during re-ingest** — these papers already have
  entity/concept pages. Re-ingest only updates the raw source, not the wiki layer.
