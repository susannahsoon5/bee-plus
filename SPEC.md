# Spec

## Overview

A personal tool for capturing conference talks and turning them into shareable,
illustrated summaries. During a talk, the [Bee](https://www.bee.computer/) wearable
transcribes the speaker while the user photographs slides (and whiteboards,
demos) on their phone. Afterwards, this tool:

1. Pulls the transcription from the Bee API.
2. Pulls photos taken during the talk from Google Photos.
3. Matches each photo to the moment in the transcript it was taken, using
   capture timestamps, and embeds it inline.
4. Reads the content of each slide photo (OCR + visual interpretation) so slide
   text becomes part of the searchable, summarizable record.
5. Generates a summary of the talk's main points, illustrated with the most
   relevant photos, that is meaningful to someone who wasn't in the room.

The broader goal is to organise Bee transcriptions so they are easy to search,
delete, and enhance; conference-talk capture with photo attachment is the first
enhancement.

> **Assumptions made in this draft** (see Open Questions to confirm/override):
> single user, personal tool; local-first (runs on the user's machine, data
> stored locally); photos come from Google Photos; talk boundaries are chosen by
> the user after the fact from a time range; slide interpretation and
> summarization use the Claude API. Anything you disagree with, flag and I'll
> revise.

## Goals

- **Capture a talk end-to-end.** Given a time range (or a Bee conversation),
  assemble the transcript, the photos taken during it, and the extracted slide
  content into a single reviewable record.
- **Attach photos to the right moment.** Place each photo at the transcript
  point closest to its capture time, so a slide sits next to what the speaker
  was saying when it was shown.
- **Extract meaning from slides.** Turn each slide photo into text (titles,
  bullet points) plus a short description of any diagram/chart, so slide content
  is searchable and feeds the summary.
- **Produce a shareable summary.** Generate a concise write-up of the talk's
  main points with a handful of embedded, relevant photos, understandable by
  someone who wasn't there.
- **Organise transcriptions.** Store talks so they can be listed, searched
  (across transcript + slide text + summary), and deleted.
- **Tolerate clock drift.** Match photos to transcript moments robustly even
  when the phone clock and Bee clock differ by a minute or two.

## Non-Goals

- Real-time / live transcription or in-session photo attachment — this is a
  post-talk workflow.
- Being a general note-taking or PKM app; scope is Bee transcriptions + attached
  photos.
- Multi-user accounts, sharing infrastructure, or a hosted service (v1 is
  single-user and local; a summary can be *exported* to share, but the tool
  isn't a sharing platform).
- Editing or re-recording audio; Bee is the source of transcription.
- Photo sources other than Google Photos in v1.
- Automatic, unattended talk-boundary detection (v1 uses a user-provided time
  range; auto-segmentation is a possible later enhancement).

## Requirements

### Functional

- **FR1 — Bee ingestion.** Authenticate to the Bee API and fetch conversations /
  transcriptions with their timestamps. Store transcript segments with
  per-segment start/end times where available.
- **FR2 — Google Photos ingestion.** Authenticate (OAuth) to Google Photos and
  list photos within a time range, with each photo's capture timestamp and a way
  to fetch its bytes.
- **FR3 — Talk definition.** Let the user define a "talk" by a time range (start,
  end) and optional title. All transcript segments and photos in that window
  belong to the talk.
- **FR4 — Photo↔transcript matching.** For each photo in the talk, find the
  transcript segment whose time is closest to the photo's capture time (within a
  configurable tolerance window) and attach the photo there. Photos with no
  segment within tolerance are attached to the talk but flagged "unplaced".
- **FR5 — Clock-offset correction.** Support a per-talk time offset (phone clock
  vs Bee clock) that shifts all photo timestamps before matching. Offset can be
  set manually; auto-estimation is a stretch goal (see Open Questions).
- **FR6 — Slide content extraction.** For each attached photo, run it through a
  vision model to produce (a) transcribed slide text and (b) a short description
  of any non-text visual (chart/diagram/screenshot). Store this with the photo.
- **FR7 — Talk summary.** Generate a summary of the talk's main points from the
  transcript + extracted slide text, and select the N most relevant photos to
  embed. Output as a self-contained document (Markdown/HTML) that renders inline
  images.
- **FR8 — Storage & organisation.** Persist talks, transcripts, photos (or refs
  + local cache), slide extractions, and summaries. List and open past talks.
- **FR9 — Search.** Full-text search across transcript text, extracted slide
  text, and summaries; return matching talks and jump to the segment.
- **FR10 — Delete.** Delete a talk and its associated data (with confirmation).
  Deleting a talk must not delete the original photos in Google Photos.
- **FR11 — Review UI.** View a talk as transcript with inline photos; view/edit
  the summary; adjust the clock offset and re-match; add/remove a photo from a
  segment; mark the summary's chosen photos.

### Non-functional

- **NFR1 — Local-first & private.** Data stored on the user's machine. Photos and
  transcripts leave the machine only for (a) the APIs they come from and (b) the
  Claude API calls for slide interpretation and summarization. Make the Claude
  calls opt-in-visible so the user knows what's sent.
- **NFR2 — Idempotent ingestion.** Re-running ingestion for a talk doesn't
  duplicate segments/photos; matching is recomputable without data loss.
- **NFR3 — Cost-aware LLM use.** Slide extraction is one vision call per photo;
  cache results so re-summarizing doesn't re-pay for OCR. Summary is one call per
  talk (re-runnable on demand).
- **NFR4 — Resilience.** Partial failures (one photo fails OCR, Photos API
  rate-limits) don't fail the whole talk; failed items are retryable.
- **NFR5 — Secrets.** API keys / OAuth tokens stored outside the repo (env or a
  local credentials file), never committed.

## Design

### High-level flow

```
Bee API ────────► transcripts ──┐
                                 ├─► define talk (time range) ─► match photos ─┐
Google Photos ──► photos ────────┘        │                                     │
                                          ▼                                     ▼
                                  clock-offset correction              per-photo vision
                                                                       (slide text + desc)
                                                                                │
                                                                                ▼
                                                                    summary generation ──► shareable doc
                                        all persisted in local store; searchable
```

### Components

- **Ingestors** — `bee` and `google_photos` clients. Each fetches raw records for
  a time range and normalizes to internal models. OAuth/token handling lives
  here.
- **Store** — local SQLite database (single file, easy to back up) plus a local
  cache directory for downloaded photo bytes and generated docs. Tables:
  `talks`, `segments`, `photos`, `slide_extractions`, `summaries`. FTS index over
  transcript + slide text + summary for FR9.
- **Matcher** — given a talk's segments and photos (+ offset), computes
  photo→segment attachments by nearest timestamp within tolerance. Pure and
  recomputable (FR4/FR5/NFR2).
- **Slide extractor** — sends each photo to the Claude vision API and stores
  structured output `{ slide_text, visual_description, is_slide }`. Cached by
  photo id + image hash (NFR3).
- **Summarizer** — sends transcript + slide extractions to Claude, gets back
  `{ title, main_points[], chosen_photo_ids[] }` via structured outputs, renders
  a Markdown/HTML doc with the chosen photos embedded.
- **Interface** — v1: a CLI for ingest/define/match/extract/summarize/search/
  delete, plus a lightweight local web view for reviewing a talk with inline
  photos and editing the summary (FR11). (Interface split is an Open Question.)

### LLM usage (Claude API)

Both LLM tasks use the Anthropic Python SDK against the Claude API.

- **Model.** Default to **Claude Opus 4.8** (`claude-opus-4-8`) for both slide
  interpretation and summarization. For a high-volume/cost-sensitive personal
  run, **Claude Sonnet 5** (`claude-sonnet-5`) is a good cheaper alternative —
  make the model configurable. (Both support vision and structured outputs.)
- **Slide extraction (vision).** One request per photo: an `image` content block
  (base64 or Files API) + a prompt asking for slide text and a short visual
  description. Use **structured outputs** (`output_config.format` with a JSON
  schema) so the result parses reliably into
  `{ is_slide, slide_text, visual_description }`.
- **Summarization.** One request per talk: transcript text + per-photo slide
  extractions in the prompt; structured output
  `{ title, main_points: string[], chosen_photo_ids: string[] }`. Use adaptive
  thinking (`thinking: {type: "adaptive"}`) and stream if output is large. Render
  the chosen photos inline in the exported doc.
- **Auth.** `ANTHROPIC_API_KEY` from env (or `ant auth login` profile); never
  committed.

### Data model (sketch)

- `talk(id, title, start_ts, end_ts, clock_offset_secs, bee_conversation_id, created_at)`
- `segment(id, talk_id, start_ts, end_ts, speaker, text)`
- `photo(id, talk_id, google_media_id, capture_ts, local_path, attached_segment_id, placement_status)`
- `slide_extraction(photo_id, is_slide, slide_text, visual_description, model, created_at)`
- `summary(talk_id, title, main_points_json, chosen_photo_ids_json, doc_path, model, created_at)`

## Open Questions

1. **Interface split.** CLI-only, CLI + local web viewer, or a local web app for
   everything? (Draft assumes CLI + lightweight local viewer.)
2. **Talk boundaries.** Confirm v1 uses a user-provided time range (vs inferring
   from silence gaps, or mapping to Bee's own conversation boundaries if Bee
   already segments the day).
   Answer: Bee's conversation boundaries and silence gaps
4. **Clock offset.** Manual offset only in v1, or attempt auto-estimation (e.g.
   ask the user to photograph the first slide as the speaker starts, or align on
   a known event)? How much drift do you actually see between phone and Bee?
   Answer: Auto-estimation allowing a drift 30 seconds
6. **Matching tolerance.** What default window (±30s? ±2min?) and what should
   happen to photos outside it — dropped, flagged "unplaced", or attached to the
   nearest segment regardless?
   Answer: flagged as unplaced so I can decide where they are relevant.
8. **Placement granularity.** Embed at the exact transcript line, or just bucket
   photos into the talk for the summary to draw from? (Draft does exact-line with
   an unplaced fallback.)
   Answer: Draft does exact-line with option to move to where it should be or removed to unplaced fallback.
  
9. **Slide extraction depth.** OCR-style text only, or also interpret
   charts/diagrams in prose? (Draft does both.) Should non-slide photos (people,
   rooms) be extracted or skipped?
   Answer: Both text and charts for applicable photos. Other photos such as people and rooms can be skipped.
10. **Summary format & export.** Markdown, HTML, or both? Where do exported
   summaries go, and is "share" just handing over the file, or something richer
   later?
    Answer: HTML
12. **Bee API specifics.** Does the Bee API expose per-utterance timestamps (or
   only whole-conversation times)? Does it already segment the day into
   conversations we can reuse as talk candidates? (Affects FR1/FR3.)
Answer: Give me a toggle to turn it off or on.
14. **Google Photos scope.** Read-only access is enough? Any need to filter to a
   specific album, or is a time-range query over the whole library fine?
Answer: read-only access and time-range over whole library
16. **Model choice & cost.** Opus 4.8 everywhere, or Sonnet 5 for slide OCR and
    Opus 4.8 only for the final summary? Any monthly cost ceiling to design
    around?
    Answer: Sonnet 5 for everything
18. **Search.** Is local full-text search (SQLite FTS) sufficient, or do you want
    semantic/embedding search across talks?
    Answer: Add semantic/embedding search
    
