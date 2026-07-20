# video-cooking

A router skill that chains [`video-download`](https://github.com/ChHsiching/video-download-skill) → [`video-subtitle`](https://github.com/ChHsiching/video-subtitle-skill) into one command. Give it a URL, get back a cooked bilingual-subtitled video — no running the two skills by hand, no moving files between them.

This is the third skill in a three-skill set:

| Skill | Job |
|---|---|
| `video-download` | Fetch raw video + metadata + cover from a URL |
| `video-subtitle` | Transcribe, translate, burn bilingual subtitles |
| **`video-cooking`** | **Chain the two above into one `/video-cooking` command** |

`video-cooking` does nothing on its own — it's a thin dispatcher. All real work lives in the two skills it calls.

## When to use which

- **Have a URL, want the full pipeline** → `/video-cooking`
- **Just want the raw file** → `video-download` directly
- **Already have a local file** → `video-subtitle` directly
- **Want to review the raw video before the slow subtitle step** → `video-download`, review, then `video-subtitle`

## How they chain

The two downstream skills share a directory convention, so `video-cooking` just hands the path and filename stem from one to the other:

```
/video-cooking <URL>
   │
   ├─ video-download  ──►  <output-root>/raw/<name>.{raw.mp4, source.json, jpg}
   │
   └─ video-subtitle  ──►  <output-root>/{transcript, subtitle, cooked}/...
                            (finds everything from <output-root> + <name>)
```

`<output-root>` defaults to `<cwd>/<author>/<video-name>/`. Both skills use the same `<name>` stem for every file they produce, so the whole pipeline produces one tidy per-video folder with no renaming.

## Requirements

Both downstream skills installed:

```bash
npx skills add ChHsiching/video-download-skill
npx skills add ChHsiching/video-subtitle-skill
npx skills add ChHsiching/video-cooking-skill
```

The router itself has no dependencies — it only dispatches.

## Install

```bash
npx skills add ChHsiching/video-cooking-skill
```

`video-cooking` is **user-invoked** (`disable-model-invocation: true`). It never auto-fires — you type `/video-cooking` when you want it. This keeps the three skills' trigger surfaces clean: `video-download` and `video-subtitle` remain independently auto-triggered for their own jobs, and the router only fires when you explicitly ask for the chained flow.

## Usage

Inside your agent:

> /video-cooking https://www.youtube.com/watch?v=...

The router runs `video-download` (you'll confirm `<author>`/`<video-name>`/`<name>` once at the start), then hands off to `video-subtitle`, which runs end to end. Expect to wait — transcription is the slow step (CPU + `large-v3` runs at roughly 0.5–0.7x realtime).

## License

MIT
