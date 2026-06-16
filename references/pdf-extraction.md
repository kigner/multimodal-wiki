# Local PDF Extraction for Wiki Ingestion

When ingesting local PDF files (not URLs), `web_extract` won't work because it
expects HTTP(S) URLs. Use this workflow instead.

## Step 1: Copy PDFs to raw/papers/

```bash
cp /c/Users/user/source-dir/*.pdf "$WIKI/raw/papers/"
```

## Step 2: Install pymupdf (if needed)

```bash
uv pip install pymupdf
# or: pip install pymupdf
```

## Step 3: Extract text via execute_code

```python
import fitz  # pymupdf
import os, hashlib, re
from datetime import datetime

papers_dir = os.path.join(os.environ["WIKI_PATH"], "raw", "papers")  # your wiki root (e.g. D:\wiki)

for fname in sorted(os.listdir(papers_dir)):
    if not fname.endswith('.pdf'):
        continue

    pdf_path = os.path.join(papers_dir, fname)
    doc = fitz.open(pdf_path)
    text_parts = [page.get_text() for page in doc]
    doc.close()

    full_text = '\n\n'.join(text_parts)
    sha = hashlib.sha256(full_text.encode('utf-8')).hexdigest()[:12]

    # Raw frontmatter
    frontmatter = f"""---
source_path: RespSearch/{fname}
ingested: {datetime.now().strftime('%Y-%m-%d')}
sha256: {sha}
---

"""
    md_name = fname.replace('.pdf', '.md')
    md_path = os.path.join(papers_dir, md_name)
    with open(md_path, 'w', encoding='utf-8') as f:
        f.write(frontmatter + full_text)

    title = re.sub(r'\s*\[[a-f0-9]+\]\s*', '', fname.replace('.pdf', ''))
    print(f"[OK] {title[:80]} ({len(full_text)} chars)")
```

## Step 4: Categorize papers (for bulk ingest)

Use another execute_code pass to classify by keyword matching on titles, then
group into categories to plan which entity/concept pages to create.

## Key points

- **Always save both** the original `.pdf` AND the extracted `.md` in `raw/papers/`
- **The `.md` is the working copy** — read this for content extraction, but never modify it
- **SHA256 in frontmatter** covers the extracted text body only (not the frontmatter itself)
- **Use native Windows paths** (`<WIKI_PATH>\...`, your wiki root) in write_file/read_file calls, not bash paths
