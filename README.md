# video-cooking

A router skill that chains [`video-download`](https://github.com/ChHsiching/video-download-skill) → [`video-subtitle`](https://github.com/ChHsiching/video-subtitle-skill) into one command, with [`cook`](https://github.com/ChHsiching/video-cook) as the deterministic executor. Give it a URL, get back a publish-ready shipment — cooked bilingual-subtitled video, upload metadata, cloud-srt files, cover, README.

This is the third skill in a four-part set:

| Skill | Job |
|---|---|
| `video-download` | Fetch raw video + metadata + cover from a URL |
| `video-subtitle` | Transcribe, translate, burn bilingual subtitles, ship the release set |
| **`video-cooking`** | **Chain the two above into one `/video-cooking` command, with intent capture and final shipment verification** |
| `cook` (CLI) | Deterministic executor — assembles every yt-dlp / whisperX / ffmpeg command correctly, runs the mechanical verification gates |

`video-cooking` dispatches, captures publish intent, and verifies. The actual work lives in the two skills it calls, executed by cook.

## When to use which

- **Have a URL, want the full pipeline** → `/video-cooking`
- **Just want the raw file** → `video-download` directly
- **Already have a local file** → `video-subtitle` directly
- **Want to review the raw video before the slow subtitle step** → `video-download`, review, then `video-subtitle`

## How they chain

The two downstream skills share a directory convention, so `video-cooking` hands the path and filename stem from one to the other, then verifies the final shipment:

```
/video-cooking <URL>
   │
   ├─ Step 0: capture publish intent (platforms, language, placement, paths)
   │
   ├─ Step 1: video-download  ──►  <output-root>/raw/<name>.{raw.mp4, source.json, jpg}
   │         cook verify-shipment --stage raw  (router's independent gate)
   │
   └─ Step 2: video-subtitle (+ intent from Step 0)
              ──►  <output-root>/{transcript, subtitle, cloud-srt, cooked}/...
                   cook verify-shipment  (final gate — full shipment must be present)
```

`<output-root>` defaults to `<cwd>/<author>/<video-name>/`. Both skills use the same `<name>` stem for every file they produce, so the whole pipeline produces one tidy per-video folder with no renaming.

## What makes this router different from running the two skills by hand

1. **Publish intent capture (Step 0).** The router asks once at the start which platforms the user is targeting, then tells `video-subtitle` explicitly to produce the full shipment (cloud-srt, cover.jpg, per-platform upload.md). Without this handoff, `video-subtitle` might skip cloud-srt or forget the cover — both are easy to miss when running by hand.

2. **Independent verification gates.** Each step's Done criterion includes a `cook verify-shipment` check at the router level, not just trust in the downstream's self-report. The final gate checks every file in the shipment exists and durations cross-check. This is what catches the "agent feels done but cover.jpg is missing" failure mode.

3. **Time budget and parallelism.** The router documents the end-to-end wall-clock expectation (~90–145 min on CPU, dub runs overnight per the cue-count formula) and the parallelism opportunities (long stages run foreground by default under your task manager; agent can author upload.md and README during the waits).

4. **Defaults that don't over-ask.** Platforms default to "all", language defaults to bilingual, placement defaults to bottom-bar, model defaults to large-v3. Only the output path confirmation is mandatory (it sets the stem for everything). The router only interrupts when there's a real reason to deviate.

## Requirements

```bash
npx skills add ChHsiching/video-download-skill
npx skills add ChHsiching/video-subtitle-skill
npx skills add ChHsiching/video-cooking-skill
pip install video-cook[all]
```

The router itself has no dependencies beyond the two skills and cook — it only dispatches and verifies.

## Install

```bash
npx skills add ChHsiching/video-cooking-skill
```

`video-cooking` is **user-invoked** (`disable-model-invocation: true`). It never auto-fires — you type `/video-cooking` when you want it. This keeps the three skills' trigger surfaces clean: `video-download` and `video-subtitle` remain independently auto-triggered for their own jobs, and the router only fires when you explicitly ask for the chained flow.

## Usage

Inside your agent:

> /video-cooking https://www.youtube.com/watch?v=...

The router captures publish intent (Step 0), runs `video-download` with path overrides, verifies the raw shipment, runs `video-subtitle` with the intent handoff, verifies the full shipment. Expect to wait — transcription is the slow step (CPU + `large-v3` runs at roughly 0.5–0.7× realtime).

## License

MIT
