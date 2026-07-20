---
name: video-cooking
description: Router that chains video-download → video-subtitle into one command.
disable-model-invocation: true
---

# video-cooking

A **router**: give it a video URL, it runs `video-download` to fetch the raw video, then runs `video-subtitle` on the result. The two downstream skills already share a directory convention, so no file moving between them — the router only hands the path and filename stem from one to the other.

This skill dispatches only. All real work (downloading, transcribing, translating, burning) lives in the skills it calls.

## When to reach for this router

Type `/video-cooking` when you have a **URL** and want the full raw-to-cooked pipeline.

Reach for the individual skills instead when:

- **Just want the file, no subtitles** → `video-download` directly. The router always continues to subtitles.
- **Already have a local file** → `video-subtitle` directly. The router's job is the download step you'd otherwise skip.
- **Want to review the raw video before committing to the (slow) subtitle pipeline** → `video-download` first, review, then `video-subtitle`. The router runs straight through with no review point.

## Prerequisites

Both downstream skills must be installed:

- `video-download`
- `video-subtitle`

If either is missing, stop and tell the user which one to install (`npx skills add ChHsiching/video-download-skill` / `npx skills add ChHsiching/video-subtitle-skill`).

## The pipeline

### Step 1 — Invoke `video-download`

Hand it the URL. It reports `<output-root>` and `<name>` when done — the path to its `raw/` directory and the shared filename stem of everything it produced there. **Capture both values**; they are the handoff to Step 2.

Done when `video-download` reports done. If it fails (auth wall it couldn't crack, network, etc.), stop and surface its error — Step 2 only starts on a confirmed Step 1 success.

### Step 2 — Invoke `video-subtitle`

Pass the `<output-root>` and `<name>` from Step 1. `video-subtitle` finds the raw mp4, the source metadata, and the cover under those values and runs end to end.

Done when `video-subtitle` reports done.
