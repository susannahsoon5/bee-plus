# Goal Statement

Build a local-first, single-user tool that turns a Bee wearable transcription
and its accompanying phone photos into a searchable, illustrated record of a
conference talk — and, from that record, a shareable HTML summary that makes
sense to someone who wasn't in the room.

Concretely, "done" for v1 means a user can:

1. Point the tool at a Bee conversation (or a time range), and it pulls the
   transcript and the Google Photos taken during that window.
2. Get each photo automatically placed next to the moment in the transcript
   it was taken, correcting for clock drift between phone and Bee, with
   photos outside the matching tolerance flagged "unplaced" rather than
   silently dropped or mis-attached.
3. Get slide/whiteboard photos turned into searchable text and diagram
   descriptions via a vision model, while non-slide photos (people, rooms)
   are skipped.
4. Generate a concise HTML summary of the talk's main points with the most
   relevant photos embedded, editable and re-runnable on demand.
5. List, full-text-and-semantic search, and delete past talks from local
   storage — without ever deleting the source photos in Google Photos.
6. Trust that transcripts and photos stay on their machine except for the
   Bee API, Google Photos API, and opt-in-visible Claude API calls used for
   slide interpretation and summarization (Claude Sonnet 5 throughout).

This is the first enhancement toward the broader goal of making Bee
transcriptions in general easy to search, delete, and enhance — so the
implementation should keep talk capture, photo matching, extraction, and
summarization as clearly separated components (per SPEC.md's Ingestors /
Store / Matcher / Slide extractor / Summarizer / Interface split) rather than
a single monolithic pipeline, so future enhancements beyond conference talks
can reuse the same store and matching primitives.

Success is measured against SPEC.md's Goals and Requirements (FR1–FR11,
NFR1–NFR5); the Non-Goals (no live transcription, no multi-user/hosted
service, no photo sources beyond Google Photos, no automatic talk-boundary
detection) define what is explicitly out of scope for v1.
