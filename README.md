# multimodal-wiki — install & setup

A compounding personal knowledge base (Karpathy's LLM Wiki pattern) that ingests **text,
web pages, PDFs, screenshots, and audio** into one interlinked markdown vault with
provenance. This is a self-contained fork of Hermes' built-in `llm-wiki`, extended with
vision (screenshots) and whisper (audio) ingestion plus a stronger raw→Layer-2 contract.

## Install
1. Copy the whole `multimodal-wiki/` folder into your Hermes skills tree under a category,
   e.g.:
   - Windows: `%LOCALAPPDATA%\hermes\skills\research\multimodal-wiki\`
   - macOS/Linux: `~/.hermes/skills/research/multimodal-wiki/` (or your install's skills dir)
2. Restart Hermes so it discovers the new skill.
3. (Optional, recommended) To avoid both this and the built-in `llm-wiki` activating on the
   same request, ignore/disable the built-in `llm-wiki` — `multimodal-wiki` is the superset.

## Prerequisites (the skill is instructions only — configure these yourself)

1. **Vault path (`WIKI_PATH`)** — tell the skill where your wiki lives by setting `WIKI_PATH`
   in your Hermes `.env`. Both the agent (image/text ingest, per `SKILL.md`) and the audio
   transcription code read this one value, so it is the single source of truth for the vault root.
   - **File location** — `<HERMES_HOME>/.env`:
     - **Windows:** `%LOCALAPPDATA%\hermes\.env` (e.g. `C:\Users\<you>\AppData\Local\hermes\.env`)
     - **macOS/Linux:** `~/.hermes/.env`
   - **Add one line (no quotes):**
     ```ini
     WIKI_PATH=D:\wiki          # Windows
     # WIKI_PATH=/home/you/wiki # macOS/Linux
     ```
   - **Do NOT wrap the path in double quotes.** The `.env` is parsed by python-dotenv, which
     interprets backslash escapes inside `"..."` — so `"D:\temp"` would turn `\t` into a TAB and
     break the path. Write it bare, or use forward slashes (`WIKI_PATH=D:/wiki`) to be safe.
   - **Restart Hermes after editing `.env`** so the value is loaded into the environment. (A
     `/reload` only re-applies Hermes' own known keys, not a custom one like `WIKI_PATH`.)

   First run: ask it to "create a wiki" and it builds `SCHEMA.md` / `index.md` / `log.md`
   + the `raw/` tree.

2. **Screenshots → need a vision model.** Set `auxiliary.vision` (in `config.yaml` or the
   model settings UI) to a **vision-capable model** (a VLM via your provider). Text-only
   main models (e.g. DeepSeek) reject images at the API level, so without this, screenshot
   ingestion fails. See `image-ingest.md`.

3. **Audio → need whisper.** `faster-whisper` auto-installs (lazy) in the Hermes venv. Set:
   ```yaml
   stt:
     provider: local
     local:
       model: large-v3      # best for Chinese; `medium` is lighter; `base` is weak
       language: zh         # or your language; '' = auto-detect
   ```
   First transcription downloads the model to `~/.cache/huggingface/hub` (~3 GB for
   large-v3). Pre-pull to avoid a mid-demo stall:
   ```
   <hermes-venv>\Scripts\python.exe -c "from faster_whisper import download_model; download_model('large-v3')"
   ```
   See `audio-ingest.md`. (mp3/m4a/wav/ogg work; no separate ffmpeg needed.)

4. **Obsidian (optional)** — point Obsidian at the same vault to see the graph. Set
   `OBSIDIAN_VAULT_PATH` = `WIKI_PATH`.

## Use
- Drop a source in chat (URL, pasted text, a screenshot, an audio file) → "ingest into my wiki".
- Or place files under `raw/<subdir>/` and say "ingest the new files in my wiki" (it
  discovers uningested files — those missing a frontmatter/sibling `.md`).
- Ask questions → it answers from the compiled wiki with note-level source paths.

## What's bundled
- `SKILL.md` — the full workflow (ingest / query / lint) with text, web, PDF, **image**,
  and **audio** source branches.
- `references/image-ingest.md` — screenshot → vision → `raw/screenshots/` (+ text note).
- `references/audio-ingest.md` — audio → faster-whisper → `raw/transcripts/` (verbatim).
- `references/citation-format.md` — note-level provenance (copy-pasteable raw paths).
- `references/audit-checklist.md` — concrete 巡检/audit checklist (run with the SKILL.md Lint steps).
- `scripts/audit.py` — runs all 6 audit checks in one pass (the Lint step runs this; don't hand-roll).
- `references/pdf-extraction.md`, `references/bulk-arxiv-refs.md` — PDF/arxiv helpers.
- `references/reingest-stubs.md`, `references/sha256-bulk-fix.md` — repair playbooks for stub
  sources and checksum drift.

## Credit
Based on Hermes Agent's `llm-wiki` skill and Andrej Karpathy's LLM Wiki pattern; extended
with multimodal ingestion and provenance hardening.
