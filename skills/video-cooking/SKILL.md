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

**Check and upgrade cook before starting the pipeline.** This is the agent's job, not the user's — the user never has to think about cook's version.

**Always run the upgrade first** (idempotent — `pip install -U` is a no-op if already latest):

```bash
<shared-venv>/Scripts/python -m pip install -U video-cook      # Windows
<shared-venv>/bin/python -m pip install -U video-cook           # macOS/Linux
```

Then **probe for the subcommands this run actually needs** — a version number is the wrong check (it couples the skill to cook's release schedule and goes stale every release). The right check is "does the command parse":

```bash
<venv>/Scripts/cook dub synth --help        # fails on cook < 0.3.0
```

For each stage the run will invoke, run its `--help` and read the exit code:
- Stages 1–2 (download + subtitle): `cook transcribe --help`, `cook subtitles --help`
- Stage 3 (dub, only if the user asked for Chinese dub): `cook dub synth --help`, `cook dub retime --help`

If any probe fails (exit non-zero, "invalid choice", or "unknown subcommand"), the upgrade didn't take — re-run the `pip install -U`, and if it still fails, surface the actual error to the user (network, permissions, PyPI outage). Only proceed when every probe passes.

This decouples the skill from cook's version numbering: a new cook release adds a subcommand, the skill's probe starts passing, no skill edit needed.

**YouTube download runtime.** Before `cook download` on a YouTube source, verify the download runtime is in place — YouTube's signature challenge needs a JS runtime and the yt-dlp extractor extras, both separate from cook itself. If `cook download` fails with either `n challenge solving failed` (signature challenge could not be solved) or `Only images available` (no video stream selected — the extractor fell back to thumbnails), treat it as a missing-runtime condition, not a source problem. Recover yourself, before surfacing anything to the user: install the default-extras downloader build (`pip install -U "yt-dlp[default]"`, which pulls `yt-dlp-ejs` and the YouTube extractor's other deps), confirm a JS runtime is present at Node ≥ 22, then re-run the download with `--js-runtimes node` so yt-dlp uses Node for the challenge. Setting up the deterministic backend is the agent's job, same convention as the cook upgrade above — the user should never have to think about yt-dlp's extractor deps.

## The pipeline

### Step 0 — Capture publish intent

The user typed `/video-cooking` because they want to publish. Capture the intent once, at the start, so you can pass it downstream:

- **Which platforms?** Default: **all** (B站 + 小红书 + YouTube + archive). Only ask if you have reason to believe they want a subset (e.g. they said "just for B站").
- **Subtitle language output?** Default: **bilingual** (中英). Only ask if they want single-language.
- **Subtitle placement?** Default: **bottom-bar**. Most technical content (IDE/terminal/UI demos, diagrams, dense slides) has on-screen material the subtitles would otherwise cover; bottom-bar pads a black strip below the frame so nothing is obscured. Only switch to **overlay** when the lower frame is genuinely empty (centered talking head, slides with a wide bottom margin) — and even then, bottom-bar is a safe default. **Bar height is adjustable** — surface the `--bar-px` knob (see Defaults table) when confirming placement if the source has tall content in its lower third that the default bar would clip.
- **Chinese dub?** Default: **no** (the bilingual subtitled release is the primary product). Set to **yes** only if the user said "连中配一起做" / "with Chinese dub" / "也做中配版本" — this triggers the optional Step 3 (`video-dubbing`), which clones the original speaker's voice and produces a second release with Chinese voiceover. Default-off because Step 3 is slow on CPU (hours for a 30-min video) and not always wanted.
- **Output paths?** Default: derive from source metadata (`<cwd>/<author>/<video-name>/`, `<name>` = `<video-name>`). Confirm with the user before download starts — these set the filename stem for every downstream artifact. **Confirm once here; do not re-ask downstream** — both `video-download` and `video-subtitle` would otherwise ask again.

Record the answers. Pass them to Step 2.

### Step 1 — Invoke `video-download` (or stage the raw/ from a local file)

**If the user already has the video file**, skip `cook download` and stage the `raw/` directory yourself — the pipeline downstream reads `raw/<name>.raw.mp4` + `raw/<name>.source.json` + `raw/<name>.jpg`, and these normally come from download. Build them by hand:

1. **Pick `<name>`** — a slugified stem (e.g. `AI Skills for Real Engineering Teams.mp4` → `ai-skills-for-real-engineering-teams`). This stem propagates to every downstream file; choose it once and use it everywhere. Spaces in filenames break cook's path handling.
2. **Create the directory layout**: `<output-root>/raw/`, `transcript/`, `subtitle/`, `cloud-srt/`, `cooked/`, `scripts/`.
3. **Copy the video** to `raw/<name>.raw.mp4`.
4. **Write `raw/<name>.source.json`** — fetch the source page (the URL the user gave) and extract at minimum: `title`, `uploader`, `channel`, `uploader_url`, `webpage_url`, `duration`, `description`, `tags`. The description is the richest source — it often contains the topic outline, chapter titles, and mentioned tools/people that downstream translation and upload.md both need. Don't skip this; `cook show-source` reads it for translation context and upload metadata.
5. **Extract `raw/<name>.jpg`** — grab the source page's video poster (preferred — it's the author's chosen thumbnail; check the `<video>` element's `poster` attribute, or the page's og:image). The poster is usually at `image.mux.com/.../thumbnail.jpg?time=N` (Mux-hosted) or a CDN URL. If no poster is found, fall back to `ffmpeg -ss 3 -i raw/<name>.raw.mp4 -frames:v 1` (skip t=0 — first frames are often mid-blink or not yet settled). For JS-rendered pages where `curl` returns empty, use a real-browser fetcher to read the rendered DOM.
6. **Verify**: `cook verify-shipment <output-root> <name> --stage raw` must exit 0 before proceeding. If it reports missing files, the staging is incomplete.

When staging is done, skip to Step 2 with `<output-root>` and `<name>` in hand.

**If the user gave a URL (no local file)**, invoke `video-download`:

Hand it the URL plus any overrides from Step 0 (`--author`, `--name`). `video-download` (via `cook download`) reports `<output-root>` and `<name>` when done — the path to its `raw/` directory and the shared filename stem. **Capture both values**; they are the handoff to Step 2.

Done when `video-download` reports done **and** `cook verify-shipment <output-root> <name> --stage raw` exits 0. The stage check is the router's independent gate — don't just trust the downstream's "done", verify the raw/ shipment (mp4 + source.json + jpg) is actually present.

If `cook download` fails (auth wall it couldn't crack, network, etc.), stop and surface its error — Step 2 only starts on a confirmed Step 1 success.

### Step 2 — Invoke `video-subtitle`

Pass the `<output-root>` and `<name>` from Step 1, **plus the publish intent from Step 0**. Tell `video-subtitle` explicitly:

> "This run is for upload to `<platforms from Step 0>`. Produce the full shipment: cooked mp4, upload.md with per-platform titles/descriptions/chapters, cloud-srt/ for soft-sub platforms, cooked/cover.jpg. Don't skip cloud-srt or cover — the user is going to upload. The source context at `raw/<name>.source.json` (run `cook show-source` to surface it) has the author, links, and source description — use it for translation context and upload metadata, don't just rely on the transcript."

Without this, `video-subtitle` might treat cloud-srt/ as lazy, forget cover.jpg, or translate purely from the transcript and miss the author/links/description the source platform already provided. The intent handoff is what makes the router produce a publish-ready shipment every time.

`video-subtitle` (via cook) runs end to end: extract audio → transcribe → **audit ASR proper nouns** → translate (with source context) → subtitles → burn → upload.md (with source context) → cover → README.

**Gate A — full-cue review of the cleaned English transcript.** Once the ASR audit has corrected the English transcript and before translate-from-clean-source begins, spawn a **fresh subagent** (not the router agent) to run a full-cue review against the audited English SRT. See [Full-cue review gates](#full-cue-review-gates) for the reviewer contract. Translate does not start until Gate A clears — translation quality is bounded by source quality, so the cleaned English must be confirmed correct end to end first.

Done when `video-subtitle` reports done **and** `cook verify-shipment <output-root> <name>` exits 0 (full shipment, all stages) **and Gate B clears**. `cook verify-shipment` runs first — it catches missing files and wrong durations. Gate B (below) runs second — it catches wrong content. The run is not done until both pass. If `cook verify-shipment` reports missing files, surface them and go back to the relevant step.

**Gate B — full-cue review of the burned bilingual video.** After the bilingual cooked video is burned and `cook verify-shipment` exits 0, spawn a **fresh subagent** to run a full-cue review against the burned bilingual subtitles (every cue, both languages). See [Full-cue review gates](#full-cue-review-gates). Step 2 is not done until Gate B clears.

### Step 3 — Invoke `video-dubbing` (optional, produces the Chinese-dubbed release)

**Skip this step unless the Step 0 "Chinese dub?" answer was yes.** The bilingual subtitled release from Step 2 is the primary product; the Chinese dub is an additive bonus.

Pass `<output-root>` and `<name>`. Tell `video-dubbing`:

> "The bilingual cooked video is done. Produce the Chinese dub for upload to `<platforms from Step 0>` alongside the bilingual release."

`video-dubbing` reads `raw/<name>.raw.mp4` (original audio, for Demucs separation + voice cloning reference) and `transcript/<name>.en.full.srt` (the full-sentence English transcript — produce it in Step 2 via `scripts/make_full_srt.py`; dubbing needs complete sentences not subtitle fragments), and writes its outputs to a new `dubbed/` stage folder plus `cooked/<name>.dubbed.mp4`. It does not modify anything `video-subtitle` produced.

**The `--python` flag is mandatory when IndexTTS2 lives in a separate venv** (the common case — its heavy deps like torch are isolated from cook's own Python). cook runs each dub stage as a subprocess under that interpreter, so `from indextts import ...` resolves. Resolve the venv once (default `~/Git/index-tts/.venv`) and pass it to every dub command:

```
<venv>/Scripts/cook dub separate <root> <name> --python <indextts-venv>/Scripts/python.exe
<venv>/Scripts/cook dub synth    <root> <name> --python <indextts-venv>/Scripts/python.exe
...
# or all four stages at once:
<venv>/Scripts/cook dub full <root> <name> --python <indextts-venv>/Scripts/python.exe
```

**Dub pipeline stage order** (run in this sequence; six are deterministic-tool stages under the IndexTTS2 venv, one is agent-owned):

1. **separate** — Demucs splits `raw/<name>.raw.mp4`'s audio into vocals and accompaniment. (tool)
2. **extract_reference** — pulls a voice-cloning reference clip from the separated vocals. (tool)
3. **translate** — produce the dub translation file (`<name>.translations_dub.txt`), one Chinese line per full-sentence English cue from `transcript/<name>.en.full.srt`. **Agent-owned** — this is your work, not cook's. Produce the file before invoking synth.
4. **synth** — IndexTTS2 synthesizes the Chinese audio cue by cue against the cloned voice. (tool)
5. **timeline** — builds a string-of-pearls timeline placing each synthesized cue back-to-back. (tool)
6. **retime** — re-times the video to the new audio timeline. (tool) **This intentionally changes the dubbed video's length** — Chinese cues rarely match English timing — so a duration mismatch between `raw/<name>.raw.mp4` and `cooked/<name>.dubbed.mp4` is expected and is **not** a verification failure. Do not treat the gap as a defect.
7. **burn** — burns the Chinese subtitles into the re-timed video. (tool)

**Dub subtitle style is independent of the bilingual release.** The burn in stage 7 uses its own subtitle style (shorter bar, smaller font) tuned for the Chinese-only dub — it does **not** inherit the bilingual styling from Step 2. Treat any style difference between `cooked/<name>.mp4` and `cooked/<name>.dubbed.mp4` as expected: do not "fix" a benign difference, and do not copy the bilingual style settings into the dub burn.

**Gate C — full-cue review of the burned dubbed video.** After the dub burn completes, spawn a **fresh subagent** to run a full-cue review against the burned Chinese subtitles on `cooked/<name>.dubbed.mp4`. See [Full-cue review gates](#full-cue-review-gates). Step 3 is not done until Gate C clears.

Done when `video-dubbing` reports done **and** `cooked/<name>.dubbed.mp4` exists and plays clean end-to-end **and Gate C clears**. This is an additive stage — if it fails, the Step 2 shipment is still complete and publishable.

## Full-cue review gates

Three points in the pipeline seal human-readable content — the ASR-audited English transcript (Gate A), the burned bilingual video (Gate B), and the burned dubbed video (Gate C). Each is gated by a **fresh-subagent full-cue review** that runs in addition to the existence-and-duration check (`cook verify-shipment` or the file-exists check). The existence check runs first and catches missing files / wrong durations; the review gate runs second and catches wrong content. The stage is not done until both pass.

**Reviewer contract (same for all three gates):**

- **Fresh subagent, not the router agent.** The router agent is anchored on the work it just produced. Spawn a new subagent for the review so the read is independent.
- **Read every cue end to end.** Read the whole SRT/ASS, in order, in context. The review principle is "read every proper noun in context and confirm it via web search" — **not** "search for a memorized list of known error signatures." Pattern-matching known errors misses novel ones; reading every line in context catches them.
- **What to flag:**
  - **Split words across cues** — a word broken at a cue boundary that should be one token.
  - **Adjacent duplicate lines** — the same cue repeated back-to-back.
  - **ASR errors in proper nouns** — names, places, brands, libraries, commands the transcription got wrong. For every proper noun you cannot confirm from context, web-search it and confirm before passing.
  - **Missing translation lines** (Gates B and C only) — cues with English but no Chinese (Gate B) or no Chinese audio / subtitle (Gate C).
  - **Biliteral merge bleed** (Gate B only) — a cue whose text repeats the previous cue's content. The `cook subtitles` biliteral merge uses timestamp-union to reconcile mismatched EN/ZH cue counts; when one language's cue spans two of the other's, the spanning text can carry into the next cue. This is a merge artifact, not a translation error — fix it in the bilingual SRT and both ASS files, then re-burn.
- **Fail loop.** On any defect found, the router fixes every listed defect, then **re-runs the same gate** (fresh subagent, full re-read) — not a spot-check of just the fixed lines. The stage is not done until a full review pass finds zero defects.

These are gates (completion criteria), not suggestions. The run does not advance past Gate A, and Step 2 / Step 3 do not declare done, until the corresponding gate has cleared.

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
| Dub (optional Step 3) | ~10 hrs on CPU | IndexTTS2 synthesis ~7h (single-thread constraint) + minterpolate re-timing ~3h. **Runs overnight.** GPU doesn't help (IndexTTS2 is CPU-bound by the single-thread constraint). |

**Long-task execution:** cook runs long tasks (transcribe, burn, dub synth/retime) in the **foreground by default** — the command blocks until done and returns the exit code. When an outer task manager supervises the process (e.g. zcode's background tasks, or an agent shell), let it own the lifecycle: it tracks the process, notifies on completion, and can stop it. Run these long tasks through that manager rather than passing `--detach`. Reserve `--detach` for when you run cook directly from a terminal and want to reclaim it.

While long tasks run, the agent can:
- During transcription: pre-read the partial transcript, draft upload.md titles/description
- During burning: write the README (you know the file layout by then)

Don't sit idle waiting — fill the wait with authoring work.

## Defaults (don't over-ask)

The pipeline has sensible defaults. Only interrupt the user when you have reason to believe they want to deviate:

| Decision | Default | When to ask |
|---|---|---|
| Platforms | all (B站 + 小红书 + YouTube + archive) | User said "just for X" |
| Subtitle language | bilingual (中英) | User asked for single-language |
| Subtitle placement | bottom-bar (`--bar-px` default 220 on `cook subtitles` / `cook burn`) | Lower frame is genuinely empty (centered talking head, wide-margin slides) → switch to overlay. Source has tall lower-third content the default 220 bar would clip → raise `--bar-px` |
| Transcription model | large-v3 | Video >60 min → mention medium is 2–3× faster, slightly less accurate |
| Output paths | derived from source metadata | Always confirm before download (sets the stem for everything) |
| Quality | best available | User said "1080p is fine" / "skip 4K" → pass `--quality 1080` |
| Chinese dub | off | User said "连中配一起做" / "with Chinese dub" → run Step 3 |

The path confirmation is the only one that's not optional — it sets the `<name>` stem that every downstream file inherits. Everything else has a working default; let the user override only if they speak up.
