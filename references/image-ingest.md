# Image Source Ingestion (screenshots, diagrams, figures)

A standalone image — a screenshot pasted in chat, a diagram saved to `raw/screenshots/`,
or a figure pulled from an article — is a **first-class source**, NOT a passive
`raw/assets/` attachment. It gets the same treatment as a PDF: the original is archived
immutably and a text extract is saved beside it. This doc defines that flow.

`web_extract` and pymupdf don't read images. Use the **`vision_analyze`** tool — it routes
the image to the model configured at `auxiliary.vision`. If that slot is unset or points at
a text-only model, vision fails: the main text model (e.g. `deepseek-v4-pro`) rejects images
at the API level (`unknown variant image_url, expected text`). A vision-capable model must
be configured first (see the host's model config).

## Directory roles (keep these SEPARATE)

```
raw/screenshots/
    <topic>.png   # ORIGINAL image — immutable Layer-1 source / evidence. Never embedded directly, never edited.
    <topic>.md    # vision text extract — saved beside the original (mirrors raw/papers/ .pdf + .md)
raw/assets/
    <topic>.png   # DISPLAY copy — created ONLY when a Layer-2 page embeds the image via ![[...]]
```

- `raw/screenshots/` is the **source archive**: original PNG + its text extract. Immutable.
- `raw/assets/` holds only images actually **displayed** inside wiki pages. Copy the original
  here when (and only when) a Layer-2 page needs to show it.
- **Don't dump the source screenshot straight into `assets/`** — that conflates evidence with
  display. The original lives in `raw/screenshots/`.

## Steps

### 1. Archive the original
Save the source image to `raw/screenshots/<topic>.png` (descriptive, lowercase-hyphen name).
If it was pasted in chat, the file sits under Hermes' `composer-images/` — copy it to
`raw/screenshots/` with a real name. Never modify it afterward.

### 2. Read it with the vision model
Run `vision_analyze` on the saved PNG with a thorough-description prompt, e.g.:

> "Describe everything in this image in exhaustive detail: all text, labels, numbers,
> modules/boxes, arrows and their direction, colors, and overall layout. Transcribe any
> diagram structurally (list each block and what connects to what)."

Capture the full returned description **verbatim** — that text is the extract.

### 3. Save the text extract beside the original
Write `raw/screenshots/<topic>.md`:

```yaml
---
source: raw/screenshots/<topic>.png          # the original this was extracted from
source_url: ''                               # if the image came from a web page, put it here
ingested: YYYY-MM-DD
sha256: <hex digest of the description BODY below>
extracted_by: vision (<vision model id>)     # which model read it — provenance
confidence: medium                           # vision can misread diagrams — see caveat
---

![[<topic>.png]]

<full vision description, verbatim>
```

- **`sha256` must be REAL** — compute it over the description body (everything after the
  closing `---`). Do NOT fabricate a placeholder. (Observed failure: a fake `a1b2c3…` hash
  silently breaks drift detection.)
- The `.md` is the **working copy** for downstream weaving; the `.png` is the evidence.

### 4. Weave into Layer 2 (normal Ingest steps ③④⑤⑥)
Proceed with the standard flow: check existing pages, create/update entity & concept pages,
cross-link (≥2 `[[wikilinks]]`), update `index.md` + `log.md`.

- Cite the image in page frontmatter: `sources: [raw/screenshots/<topic>.md]`.
- Provenance markers on synthesized claims: `^[raw/screenshots/<topic>.md]` (per
  `references/citation-format.md` — the raw-note path is the traceable anchor).
- To **display** the diagram in a Layer-2 page, copy the PNG to `raw/assets/<topic>.png` and
  embed `![[<topic>.png]]`. The source copy in `raw/screenshots/` stays untouched.

## Discovery (folder-drop path)
The Ingest discovery pass keys on missing frontmatter — but a `.png` is binary and can't
carry frontmatter. For images the uningested signal is different:

> **a `.png` in `raw/screenshots/` with no sibling `.md` of the same basename.**

List `raw/screenshots/*.png`, check each for a matching `.md`; any PNG without one was
dropped but never ingested → run steps 2–4 on it.

## Accuracy caveat
Vision models can confidently invent diagram details (exact layer counts, specific numbers,
label text). Set `confidence: medium` on pages built from a single image, and **flag precise
figures for human verification** rather than asserting them as fact. An honest "approx."
beats a fabricated specific.

## Inline images inside an article
When `web_extract` returns an article with figure URLs (`![cap](https://…png)`), those are
left un-read by default. For the **key** diagrams worth keeping: download the image to
`raw/screenshots/`, run steps 2–3, and fold the description into the article's wiki pages.
Don't leave load-bearing figures as remote hotlinks that rot.

## Windows paths
Use native paths (`<WIKI_PATH>\raw\screenshots\…`, where `<WIKI_PATH>` is your wiki root) in all `write_file` / `read_file` /
`vision_analyze` calls — not bash `/d/…` paths.
