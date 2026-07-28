---
name: video-cooking
description: Router that chains video-download → video-subtitle → (optional) video-dubbing into one command.
disable-model-invocation: true
---

# video-cooking

A **router**: give it a video URL, it runs `video-download` to fetch the raw video, then runs `video-subtitle` on the result, and optionally runs `video-dubbing` to produce a Chinese-dubbed release. The downstream skills share a directory convention, so no file moving between them — the router hands the path and filename stem from one to the other, then verifies the final shipment.

This skill dispatches and verifies. All real work (downloading, transcribing, translating, burning, dubbing) lives in the skills it calls, executed by the [`cook`](https://github.com/ChHsiching/video-cook) CLI.

## When to reach for this router

Type `/video-cooking` when you have a **URL** and want the full raw-to-cooked pipeline.

Reach for the individual skills instead when:

- **Just want the file, no subtitles** → `video-download` directly. The router always continues to subtitles.
- **Already have a local file** → `video-subtitle` directly. The router's job is the download step you'd otherwise skip.
- **Want to review the raw video before committing to the (slow) subtitle pipeline** → `video-download` first, review, then `video-subtitle`. The router runs straight through with no review point.

## Prerequisites

Both downstream skills must be installed:

- `video-download` (`npx skills add ChHsiching/video-download-skill`)
- `video-subtitle` (`npx skills add ChHsiching/video-subtitle-skill`)

The optional Step 3 (Chinese dub) needs an additional skill:

- `video-dubbing` (`npx skills add ChHsiching/video-dubbing-skill`) — only required if the user wants the Chinese-dubbed release. Skip the install check if Step 0's "Chinese dub?" answer is no.

Plus the [`cook`](https://github.com/ChHsiching/video-cook) CLI (`pip install video-cook[all]`), which both downstream skills use as their deterministic executor. If any are missing, stop and tell the user which to install.

**Check and upgrade cook before starting the pipeline.** This is the agent's job, not the user's — the user never has to think about cook's version. Resolve the shared venv the same way `video-subtitle`'s Step 0 does (`VIDEO_TOOLS_VENV` env var → `~/.venvs/video-tools/` → system Python), then run `cook --version` from it. Compare against the minimum each stage needs:

- Stages 1–2 (download + subtitle): cook ≥ 0.1.0 — any released version works.
- Stage 3 (dub, optional): cook ≥ 0.2.0 — the `cook dub` subcommand was added in 0.2.0. Only enforce this when the user asked for the Chinese dub.

If the installed cook is older than the minimum (or missing entirely), **upgrade it yourself** by running the venv's pip directly — do not stop and ask the user to do it:

```bash
<shared-venv>/Scripts/python -m pip install -U video-cook      # Windows
<shared-venv>/bin/python -m pip install -U video-cook           # macOS/Linux
```

After the upgrade finishes, re-run `cook --version` to confirm it now reports ≥ the minimum. Only then proceed to Step 0. Running an older cook into a stage that needs a newer one fails with a confusing "unknown subcommand" error — upgrading up front is cheaper than debugging that mid-pipeline.

## The pipeline

### Step 0 — Capture publish intent

The user typed `/video-cooking` because they want to publish. Capture the intent once, at the start, so you can pass it downstream:

- **Which platforms?** Default: **all** (B站 + 小红书 + YouTube + archive). Only ask if you have reason to believe they want a subset (e.g. they said "just for B站").
- **Subtitle language output?** Default: **bilingual** (中英). Only ask if they want single-language.
- **Subtitle placement?** Default: **overlay**. Consider proactively suggesting **bottom-bar** when the video is clearly an IDE/terminal/UI demo (dense visual content that overlay subtitles would cover) — but don't ask unless it matters.
- **Chinese dub?** Default: **no** (the bilingual subtitled release is the primary product). Set to **yes** only if the user said "连中配一起做" / "with Chinese dub" / "也做中配版本" — this triggers the optional Step 3 (`video-dubbing`), which clones the original speaker's voice and produces a second release with Chinese voiceover. Default-off because Step 3 is slow on CPU (hours for a 30-min video) and not always wanted.
- **Output paths?** Default: derive from source metadata (`<cwd>/<author>/<video-name>/`, `<name>` = `<video-name>`). Confirm with the user before download starts — these set the filename stem for every downstream artifact. **Confirm once here; do not re-ask downstream** — both `video-download` and `video-subtitle` would otherwise ask again.

Record the answers. Pass them to Step 2.

### Step 1 — Invoke `video-download`

Hand it the URL plus any overrides from Step 0 (`--author`, `--name`). `video-download` (via `cook download`) reports `<output-root>` and `<name>` when done — the path to its `raw/` directory and the shared filename stem. **Capture both values**; they are the handoff to Step 2.

Done when `video-download` reports done **and** `cook verify-shipment <output-root> <name> --stage raw` exits 0. The stage check is the router's independent gate — don't just trust the downstream's "done", verify the raw/ shipment (mp4 + source.json + jpg) is actually present.

If `cook download` fails (auth wall it couldn't crack, network, etc.), stop and surface its error — Step 2 only starts on a confirmed Step 1 success.

### Step 2 — Invoke `video-subtitle`

Pass the `<output-root>` and `<name>` from Step 1, **plus the publish intent from Step 0**. Tell `video-subtitle` explicitly:

> "This run is for upload to `<platforms from Step 0>`. Produce the full shipment: cooked mp4, upload.md with per-platform titles/descriptions/chapters, cloud-srt/ for soft-sub platforms, cooked/cover.jpg. Don't skip cloud-srt or cover — the user is going to upload. The source context at `raw/<name>.source.json` (run `cook show-source` to surface it) has the author, links, and source description — use it for translation context and upload metadata, don't just rely on the transcript."

Without this, `video-subtitle` might treat cloud-srt/ as lazy, forget cover.jpg, or translate purely from the transcript and miss the author/links/description the source platform already provided. The intent handoff is what makes the router produce a publish-ready shipment every time.

`video-subtitle` (via cook) runs end to end: extract audio → transcribe → **audit ASR proper nouns** → translate (with source context) → subtitles → burn → upload.md (with source context) → cover → README.

Done when `video-subtitle` reports done **and** `cook verify-shipment <output-root> <name>` exits 0 (full shipment, all stages). This is the router's final gate — the run is not done until every file in the shipment exists and the duration cross-checks pass. If `cook verify-shipment` reports missing files, surface them and go back to the relevant step.

### Step 3 — Invoke `video-dubbing` (optional, produces the Chinese-dubbed release)

**Skip this step unless the Step 0 "Chinese dub?" answer was yes.** The bilingual subtitled release from Step 2 is the primary product; the Chinese dub is an additive bonus.

Pass `<output-root>` and `<name>`. Tell `video-dubbing`:

> "The bilingual cooked video is done. Produce the Chinese dub for upload to `<platforms from Step 0>` alongside the bilingual release."

`video-dubbing` reads `raw/<name>.raw.mp4` (original audio, for Demucs separation + voice cloning reference) and `transcript/<name>.en.full.srt` (the full-sentence English transcript — it translates this into a dub script itself, since dubbing needs complete sentences not subtitle fragments), and writes its outputs to a new `dubbed/` stage folder plus `cooked/<name>.dubbed.mp4`. It does not modify anything `video-subtitle` produced.

Done when `video-dubbing` reports done **and** `cooked/<name>.dubbed.mp4` exists and plays clean end-to-end. This is an additive stage — if it fails, the Step 2 shipment is still complete and publishable.

## Time budget

The pipeline is long. Set expectations with the user, and use the wait productively.

| Stage | Wall-clock | Notes |
|---|---|---|
| Download | ~5 min | Depends on source quality and network |
| Transcribe | ~50–90 min | CPU + `large-v3` at 0.5–0.7× realtime. GPU + float16 is ~5–10× faster. **The slow step.** |
| Translate | ~10–20 min | Agent work — depends on transcript length |
| Subtitle processing | ~30 sec | cook subtitles runs the full shorten/merge/ass pipeline |
| Burn | ~10–20 min | ffmpeg re-encode, 1080p, ~6× realtime on CPU |
| upload.md + README | ~10 min | Agent authoring |
| Dub (optional Step 3) | ~10 hrs on CPU | IndexTTS2 synthesis ~7h (single-thread constraint) + minterpolate re-timing ~3h. **Runs detached overnight.** GPU doesn't help (IndexTTS2 is CPU-bound by the single-thread constraint). |

**Parallelism:** transcription and burning both run detached (cook handles this). While they run, the agent can:
- During transcription: pre-read the partial transcript, draft upload.md titles/description
- During burning: write the README (you know the file layout by then)

Don't sit idle waiting for detached jobs — poll the log periodically and fill the wait with authoring work.

## Defaults (don't over-ask)

The pipeline has sensible defaults. Only interrupt the user when you have reason to believe they want to deviate:

| Decision | Default | When to ask |
|---|---|---|
| Platforms | all (B站 + 小红书 + YouTube + archive) | User said "just for X" |
| Subtitle language | bilingual (中英) | User asked for single-language |
| Subtitle placement | overlay | Video is clearly IDE/terminal/UI demo → suggest bottom-bar |
| Transcription model | large-v3 | Video >60 min → mention medium is 2–3× faster, slightly less accurate |
| Output paths | derived from source metadata | Always confirm before download (sets the stem for everything) |
| Quality | best available | User said "1080p is fine" / "skip 4K" → pass `--quality 1080` |
| Chinese dub | off | User said "连中配一起做" / "with Chinese dub" → run Step 3 |

The path confirmation is the only one that's not optional — it sets the `<name>` stem that every downstream file inherits. Everything else has a working default; let the user override only if they speak up.
