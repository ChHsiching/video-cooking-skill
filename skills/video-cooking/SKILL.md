---
name: video-cooking
description: Router that chains video-download → video-subtitle → video-dubbing into one command.
disable-model-invocation: true
---

# video-cooking

A **router**: give it a video URL, it runs `video-download` to fetch the raw video, then runs `video-subtitle` on the result, then runs `video-dubbing` to produce a Chinese-dubbed release. The downstream skills share a directory convention, so no file moving between them — the router hands the path and filename stem from one to the other, then verifies the final shipment.

This skill dispatches and verifies. All real work (downloading, transcribing, translating, burning, dubbing) lives in the skills it calls, executed by the [`cook`](https://github.com/ChHsiching/video-cook) CLI.

## When to reach for this router

Type `/video-cooking` when you have a **URL** and want the full raw-to-cooked pipeline.

Reach for the individual skills instead when:

- **Just want the file, no subtitles** → `video-download` directly. The router always continues to subtitles.
- **Already have a local file** → `video-subtitle` directly. The router's job is the download step you'd otherwise skip.
- **Want to review the raw video before committing to the (slow) subtitle pipeline** → `video-download` first, review, then `video-subtitle`. The router runs straight through with no review point.

## Prerequisites

All three downstream skills must be installed:

- `video-download` (`npx skills add ChHsiching/video-download-skill`)
- `video-subtitle` (`npx skills add ChHsiching/video-subtitle-skill`)
- `video-dubbing` (`npx skills add ChHsiching/video-dubbing-skill`) — needed for Step 3 (Chinese dub), which is on by default.

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
- Stage 3 (dub): `cook dub synth --help`, `cook dub retime --help` — unless the dub was declined in Step 0

If any probe fails (exit non-zero, "invalid choice", or "unknown subcommand"), the upgrade didn't take — re-run the `pip install -U`, and if it still fails, surface the actual error to the user (network, permissions, PyPI outage). Only proceed when every probe passes.

This decouples the skill from cook's version numbering: a new cook release adds a subcommand, the skill's probe starts passing, no skill edit needed.

**System tools (not pip-installable).** cook shells out to two system binaries it cannot install itself: **ffmpeg** (extract / burn / download's stream-merge + ffprobe verify) and **Node ≥ 22** (YouTube's signature challenge — cook hard-codes the Node runtime for yt-dlp, so Node must be on PATH before any YouTube download). Run `cook doctor` first: it reports both and is the single check for the whole environment. If either is missing, install it (Windows: `winget install ffmpeg` + a Node LTS installer, or static builds on PATH; macOS: `brew install ffmpeg node`; Linux: distro packages) before starting — `cook download` / `cook extract` / `cook burn` will otherwise die with a clean "ffmpeg not found" message rather than proceed.

**YouTube signature-challenge recovery.** If `cook download` fails with `n challenge solving failed` or `Only images available` (the extractor fell back to thumbnails), that is a missing-runtime condition, not a source problem. Recover yourself before surfacing it: `cook doctor` (confirm Node is present), then `pip install -U "yt-dlp[default]"` in the cook venv (pulls `yt-dlp-ejs` and the YouTube extractor's deps), then re-run `cook download` — cook already wires yt-dlp to use Node for the challenge. Setting up the deterministic backend is the agent's job, same convention as the cook upgrade above.

## The pipeline

### Step 0 — Capture publish intent

The user typed `/video-cooking` because they want to publish. Capture the intent once, at the start, so you can pass it downstream:

- **Which platforms?** Default: **all** (B站 + 小红书 + YouTube + archive). Only ask if you have reason to believe they want a subset (e.g. they said "just for B站").
- **Subtitle language output?** Default: **bilingual** (中英). Only ask if they want single-language.
- **Subtitle placement?** Default: **bottom-bar**. Most technical content (IDE/terminal/UI demos, diagrams, dense slides) has on-screen material the subtitles would otherwise cover; bottom-bar pads a black strip below the frame so nothing is obscured. Only switch to **overlay** when the lower frame is genuinely empty (centered talking head, slides with a wide bottom margin) — and even then, bottom-bar is a safe default. **Bar height is adjustable** — surface the `--bar-px` knob (see Defaults table) when confirming placement if the source has tall content in its lower third that the default bar would clip.
- **Chinese dub?** **Always ask.** Default: **yes** — the Chinese-dubbed release is a standard part of the shipment (alongside the bilingual subtitled release). Surface the cost in the question: Step 3 takes ~10 hrs on CPU (runs overnight), so the user should choose knowingly. Record the answer — every downstream reference to "the dub decision" points here.
- **Output paths?** Default: derive from source metadata (`<cwd>/<author>/<video-name>/`, `<name>` = `<video-name>`). Confirm with the user before download starts — these set the filename stem for every downstream artifact. **Confirm once here; do not re-ask downstream** — both `video-download` and `video-subtitle` would otherwise ask again.

Record the answers. Pass them to Step 2.

### Step 1 — Get the raw video

**If the user already has the video file**, stage the `raw/` directory yourself — download normally produces it, but the pipeline downstream reads `raw/<name>.raw.mp4` + `raw/<name>.source.json` + `raw/<name>.jpg`, so these must exist before Step 2.

1. **Pick `<name>`** — a slugified stem (e.g. `AI Skills for Real Engineering Teams.mp4` → `ai-skills-for-real-engineering-teams`). This stem propagates to every downstream file; choose it once and use it everywhere. Spaces in filenames break cook's path handling.
2. **Copy the video** to `raw/<name>.raw.mp4` (create `raw/` — the other stage dirs are auto-created by their respective cook commands).
3. **Write `raw/<name>.source.json`** — fetch the source page and capture the fields `cook show-source` reads (title, uploader, webpage_url, description, etc.). The description is the richest source — it often contains the topic outline, chapter titles, and mentioned tools/people that downstream translation and upload.md both need.
4. **Extract `raw/<name>.jpg`** — grab the source page's video poster (preferred — it's the author's chosen thumbnail; check the `<video>` element's `poster` attribute, or og:image). The poster is usually at `image.mux.com/.../thumbnail.jpg?time=N` (Mux-hosted) or a CDN URL. If no poster is found, fall back to `ffmpeg -ss 3 -i raw/<name>.raw.mp4 -frames:v 1` (skip t=0 — first frames are often mid-blink). For JS-rendered pages where `curl` returns empty, use a real-browser fetcher to read the rendered DOM.
5. **Verify**: `cook verify-shipment <output-root> <name> --stage raw` must exit 0 before proceeding.

Then skip to Step 2 with `<output-root>` and `<name>` in hand.

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

### Step 3 — Invoke `video-dubbing` (produces the Chinese-dubbed release)

**Runs unless the dub was declined in Step 0.** The Chinese-dubbed release is part of the default shipment.

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

**Dub pipeline stage order** (run in this sequence). Five are `cook dub` stages run under the IndexTTS2 venv; two are agent-owned steps (`extract_reference` runs a skill script directly, `translate` is pure authoring). For the exact commands and the quality gate on the dub translation, follow `video-dubbing`'s SKILL.md Step 1–3 — it owns those details:

1. **separate** (`cook dub separate`) — Demucs splits `raw/<name>.raw.mp4`'s audio into vocals and accompaniment.
2. **extract_reference** (agent-owned) — runs the dubbing skill's `extract_reference.py` against the separated vocals to pull a voice-cloning reference clip. Not a `cook dub` command.
3. **translate** (agent-owned) — produce the dub translation file (`<name>.translations_dub.txt`), one Chinese line per full-sentence English cue from `transcript/<name>.en.full.srt`. This is your work, not cook's. Produce the file before invoking synth. Then generate `<name>.zh.dub.srt` via the dubbing skill's `make_zh_dub_srt.py`.
4. **synth** (`cook dub synth`) — IndexTTS2 synthesizes the Chinese audio cue by cue against the cloned voice.
5. **timeline** (`cook dub timeline`) — builds a string-of-pearls timeline placing each synthesized cue back-to-back.
6. **retime** (`cook dub retime`) — re-times the video to the new audio timeline. **This intentionally changes the dubbed video's length** — Chinese cues rarely match English timing — so a duration mismatch between `raw/<name>.raw.mp4` and `cooked/<name>.dubbed.mp4` is expected and is **not** a verification failure. Do not treat the gap as a defect.
7. **burn** (`cook dub burn`) — burns bilingual subtitles into the re-timed video and copies the upload subtitles `cloud-srt/zh.dub.srt` + `cloud-srt/en.dub.srt`.

**Dub subtitles are bilingual, in the same bar layout as the Step 2 release.** The burn in stage 7 runs the same `shorten` → `merge-short` → `biliteral` → `ass` pipeline on the dub's re-timed clock: Chinese fragments below, full-sentence English above, same 220px bottom bar. The union's repetition is role-swapped — English repeats across consecutive Chinese-fragment cues (in Step 2 it's the Chinese that repeats), so **English lines staying on screen across several cues are the design, not a defect**; treat any Gate C flag on them as a false positive.

**Gate C — full-cue review of the burned dubbed video.** After the dub burn completes, spawn a **fresh subagent** to run a full-cue review against the burned bilingual subtitles on `cooked/<name>.dubbed.mp4`. See [Full-cue review gates](#full-cue-review-gates). Step 3 is not done until Gate C clears.

Done when `video-dubbing` reports done **and** `cooked/<name>.dubbed.mp4` exists and plays clean end-to-end **and Gate C clears**. On failure, try to recover — the dub is part of the default shipment. The Step 2 bilingual release ships regardless.

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
  - **True duplicate cues** (Gates B and C) — the bilingual SRT is built by timestamp-union: when one language's cue span is longer than the other's and crosses the other's breakpoint, the longer span's text repeats across the cues it spans so each language stays fully readable. That **structural repetition is by design, not a defect** — read the `[biliteral] timestamp-union (...)` log line to confirm the run took the union path before flagging anything. The actual defect to flag is a cue whose text is **verbatim identical** to the previous cue in *both* languages (same ZH **and** same EN), which slips past the union's built-in dedup. Fix those at the source — the bilingual SRT and both ASS files, then re-burn (Gate B); the dub sources, then re-run `cook dub burn` (Gate C).
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
| Dub (Step 3) | ~10 hrs on CPU | IndexTTS2 synthesis ~7h (single-thread constraint) + minterpolate re-timing ~3h. **Runs overnight.** GPU doesn't help (IndexTTS2 is CPU-bound by the single-thread constraint). |

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
| Subtitle placement | bottom-bar (`--bar-px` on `cook subtitles` / `cook burn`; default and knobs in `--help`) | Lower frame is genuinely empty (centered talking head, wide-margin slides) → switch to overlay. Source has tall lower-third content the default bar would clip → raise `--bar-px` |
| Transcription model | large-v3 | Video >60 min → mention medium is 2–3× faster, slightly less accurate |
| Output paths | derived from source metadata | Always confirm before download (sets the stem for everything) |
| Quality | best available | User said "1080p is fine" / "skip 4K" → pass `--quality 1080` |
| Chinese dub | on (always ask) | See Step 0 — surface the ~10hr cost, let the user decline. |

The path confirmation is the only one that's not optional — it sets the `<name>` stem that every downstream file inherits. Everything else has a working default; let the user override only if they speak up.
