# Audio Source Ingestion (voice memos, recordings)

An audio file — a voice memo, an interview, a recorded thought — is a **first-class
source**. It gets the same treatment as a PDF or screenshot: the original is archived
immutably in `raw/transcripts/` and a text transcript is saved beside it. This doc
defines that flow.

## Key difference from images: there is NO agent transcription tool

Vision has a ready `vision_analyze` tool. **Audio does NOT.** Hermes ships a full STT
engine (`tools/transcription_tools.py`, default `local` = faster-whisper, free, no API
key), but it is wired only into the messaging gateway / voice input — **it is not exposed
as a tool the agent can call.** So during ingest you must drive the engine yourself via
`code_execution` (it's already installed in the Hermes venv). Do NOT look for a
`transcribe` tool — there isn't one.

## Engine
- `faster-whisper` + `ctranslate2` are installed in the Hermes venv (local, free).
- Model + language come from config `stt.local` (e.g. `model: large-v3`, `language: zh`).
  `base` is fast but weak — for Chinese use `large-v3` (best) or `medium`.
- Decoding uses PyAV, so `mp3`/`m4a`/`wav`/`ogg`/`flac` work with **no separate ffmpeg**.
- First run downloads the model from HuggingFace to `~/.cache/huggingface/hub` (one-time,
  ~3 GB for large-v3). Expect a stall on the first transcription.

## Steps

### 1. Archive the original
Save the source audio to `raw/transcripts/<topic>.<ext>` (descriptive, lowercase-hyphen).
**Rename a generic source name (e.g. `audio.wav`, `recording.m4a`) to the descriptive
`<topic>` you'll use for the transcript, so the audio and its `.md` transcript share the
SAME basename** — e.g. `id2reflectance-voice-memo.wav` + `id2reflectance-voice-memo.md`.
This keeps the "no sibling `.md` = uningested" discovery (Check 2 of `audit-checklist.md`)
reliable: a generic `audio.wav` sitting next to a differently-named transcript false-flags
as untranscribed.
Once archived under its final name, this is the immutable Layer-1 source — never modify its
**content** (renaming at archive time is fine; editing the audio is not).

### 2. Transcribe via `code_execution` (no agent tool — call the engine directly)
Prefer Hermes' own `transcribe_audio()` (it honors the configured provider/model/language):

```python
import os
from tools.transcription_tools import transcribe_audio
# WIKI_PATH = your wiki root (set in ~/.hermes/.env), e.g. D:\wiki on Windows
audio = os.path.join(os.environ["WIKI_PATH"], "raw", "transcripts", "<topic>.m4a")
r = transcribe_audio(audio)
assert r["success"], r.get("error")
text = r["transcript"]
```

**Fallback for files >25 MB or long recordings** (`transcribe_audio` enforces a 25 MB cap):
call faster-whisper directly, which has no size cap:

```python
from faster_whisper import WhisperModel
m = WhisperModel("large-v3", device="auto", compute_type="auto")  # auto→CPU int8 if VRAM is full
segments, info = m.transcribe(audio, language="zh", beam_size=5)  # `audio` from the snippet above
text = " ".join(s.text.strip() for s in segments)
# info.language, info.duration are available for the frontmatter
```

If VRAM is tight (e.g. an 8 GB card already loaded), `device="auto"` falls back to CPU
(slower but works). For very long audio, transcribe in chunks and concatenate.

### 3. Save the transcript beside the original
Write `raw/transcripts/<topic>.md`:

```yaml
---
source: raw/transcripts/<topic>.m4a           # the audio this was transcribed from
source_url: ''                                # if it came from somewhere online
ingested: YYYY-MM-DD
sha256: <hex digest of the transcript BODY below>
transcribed_by: whisper (<model>, faster-whisper, local)   # provenance
language: zh                                  # info.language
duration_sec: <info.duration>
confidence: medium                            # ASR misreads names/terms — see caveat
---

<raw ASR output, word-for-word — no summary, no headings, no added metadata>
```

- **Verbatim ONLY.** The body is the raw ASR output word-for-word — exactly what the
  model heard. Do NOT summarize, restructure, add headings/sections, fix terms, or append
  metadata (arXiv URLs, paper titles, author lists, section numbers) the speaker didn't
  say. Summary / polish / structuring / enrichment belongs in **Layer-2 pages, NEVER in
  this raw note** — keep the raw layer a faithful, diffable record so ASR misreads can be
  caught later. (`source_url`/`title` may go in frontmatter; the body stays verbatim.)
- **`sha256` must be REAL** — computed over the transcript body. Never fabricate.
- The `.md` is the working copy for weaving; the audio file is the evidence.
- `raw/assets/` is NOT involved — audio is not embedded for display.

### 4. Weave into Layer 2 (normal Ingest steps ③④⑤⑥)
Check existing pages, create/update entity & concept pages, cross-link (≥2 `[[wikilinks]]`),
update `index.md` + `log.md`.
- Cite the source in page frontmatter: `sources: [raw/transcripts/<topic>.md]`.
- Provenance markers on synthesized claims: `^[raw/transcripts/<topic>.md]` (per
  `references/citation-format.md` — the raw-note path is the traceable anchor).

## Discovery (folder-drop path)
An audio file is binary and can't carry frontmatter. The uningested signal is:

> **an audio file in `raw/transcripts/` with no sibling `.md` of the same basename.**

List `raw/transcripts/*.{mp3,m4a,wav,ogg,flac}`, check each for a matching `.md`; any
audio file without one was dropped but never ingested → run steps 2–4 on it.

## Accuracy caveat
ASR (especially the `base` model, and on Chinese) misreads names, technical terms,
numbers, and acronyms. Set `confidence: medium` on pages built from a single transcript,
**flag uncertain terms** rather than asserting them, and prefer `large-v3`/`medium` over
`base`. A raw memo is the user's own words — when a claim is critical, keep the verbatim
quote in the transcript so the user can spot a misheard term.

## Windows paths
Use native paths (`<WIKI_PATH>\raw\transcripts\…`, where `<WIKI_PATH>` is your wiki root) in all `write_file` / `read_file` /
`code_execution` calls — not bash `/d/…` paths.
