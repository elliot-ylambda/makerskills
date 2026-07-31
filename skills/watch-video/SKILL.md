---
name: watch-video
description: When you want to transcribe or analyze a local/attached video. Three depth modes are available — transcript, visual, and dense local-frame analysis. Remote video acquisition and direct provider uploads require a reviewed typed asset/video action and otherwise return execution_unavailable. Saves outputs under ~/Documents/videos/ and can capture a summary to second-brain.
metadata:
  version: 0.2.4
---

# /watch-video — Transcribe and analyze any video at the depth you choose

Replaces and broadens the prior `youtube-transcript` skill. YouTube is now one of many sources; depth is user-controlled.

## Hosted Agent execution status

Local file inspection, transcription, and frame extraction are supported.
Remote URL download, authenticated platform transcript retrieval, provider file
upload, and generated-media download are `execution_unavailable` until a
reviewed typed asset/video action is advertised. Never use a downloader, direct
HTTP client, provider key, or loopback service from shell. Ask the user to
attach/download the video or paste a transcript instead.

## Step 1 — Parse input

Accept:
- **YouTube**: full URL, `youtu.be/<id>`, `youtube.com/shorts/<id>`, raw 11-char ID
- **Loom**: `loom.com/share/<id>` or `loom.com/embed/<id>`
- **Vimeo**: `vimeo.com/<id>`
- **Riverside**: download URL or local file
- **Zoom**: local `.mp4` from a downloaded recording
- **X / IG / TikTok video**: URL for identification only; ask the user to
  attach/download the media unless a reviewed typed social asset action exists
- **Local file**: any path to an `.mp4` / `.mov` / `.webm` / `.mkv`

Detect source from URL pattern or file extension. If ambiguous, ask.

## Step 2 — Parse depth mode

| Invocation | Mode | What you get |
|---|---|---|
| `/watch-video <url>` | **transcript** (default) | Clean text, metadata, optional chapters |
| `/watch-video <url> transcript` | transcript | Same as default |
| `/watch-video <url> visual` | visual | Transcript + frames at intervals + Claude vision pass identifying key moments |
| `/watch-video <url> multimodal` | multimodal | Dense local-frame analysis through an advertised vision surface |

If the depth isn't specified and the video is >10 minutes, ask before defaulting (visual/multimodal cost real money on long videos).

## Step 3 — Pull metadata

For remote URLs, return `execution_unavailable` and request an attached/local
file or pasted transcript. For local files, use ffprobe:

```bash
ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 "<file>"
```

## Step 4 — Build workdir

```
~/Documents/videos/<source>-<slug>-<date>/
```

Where:
- `source`: `youtube` / `loom` / `vimeo` / `riverside` / `zoom` / `local`
- `slug`: kebab-case of title (first 4–6 words, max 50 chars)
- `date`: `YYYY-MM-DD`

## Step 5 — Get the transcript

**Backend selection** (local/attached files only):

1. **MLX-Whisper local** (default — fast on Mac M-series):
   ```bash
   # Both mlx-whisper and a complete local model directory must already exist.
   test -d "<local-model-dir>" || exit 2
   HF_HUB_OFFLINE=1 TRANSFORMERS_OFFLINE=1 python3 -c \
     "import json, mlx_whisper, sys; json.dump(mlx_whisper.transcribe(sys.argv[1], path_or_hf_repo=sys.argv[2]), sys.stdout)" \
     "<file>" "<local-model-dir>" > "<workdir>/transcript-raw.json"
   ```

   Do not pass a registry/repository model identifier. If the local model path
   is missing or incomplete, report the prerequisite and stop; model download
   is not a sandbox fallback.

2. **whisper.cpp** (fallback only when both its executable and model file are
   already local; never invoke its model-download helper)

**Clean the transcript** (only needed for YouTube auto-subs which have rolling captions; Whisper output is already clean):

```bash
# YouTube VTT cleanup — de-dup rolling captions, strip tags, paragraph-break on cue gaps >2s
awk '
  /^WEBVTT/ || /^Kind:/ || /^Language:/ || /^NOTE/ { next }
  /-->/ { in_cue = 1; last = ""; next }
  /^$/ { if (last) print last; in_cue = 0; last = ""; next }
  in_cue { gsub(/<[^>]+>/, "", $0); last = $0 }
  END { if (last) print last }
' "<workdir>/transcript.en.vtt" | awk '!seen[$0]++' > "<workdir>/transcript.txt"
```

Save final to `<workdir>/transcript.txt`.

## Step 6 — If `transcript` mode: stop here

Output:
- `transcript.txt`
- `metadata.json`
- One-line summary in chat: title, source, duration, word count
- Path to workdir
- (Optional) Step 9 — offer to capture to second-brain

## Step 7 — If `visual` mode: extract frames + vision pass

### Frame extraction (ffmpeg)

Cadence by source heuristic:

| Source type | Frame cadence |
|---|---|
| Screen-share / Loom / demo | 1 frame per **5s** (UI changes fast) |
| Talking head / podcast | 1 frame per **30s** (slow change) |
| Slide presentation | 1 frame per **10s** + force a frame on each detected scene change |
| Default if unsure | 1 frame per **15s** |

```bash
mkdir -p "<workdir>/frames"
ffmpeg -i "<workdir>/video.mp4" -vf "fps=1/15" "<workdir>/frames/frame-%04d.png" -y
```

For scene-change detection (slide decks especially):

```bash
ffmpeg -i "<workdir>/video.mp4" -vf "select='gt(scene,0.3)',showinfo" -vsync vfr "<workdir>/frames/scene-%04d.png" 2> "<workdir>/scene-detection.log"
```

### Vision pass

Pair each frame with the transcript chunk for the same timestamp window. Then batch-send to Claude vision for synthesis.

**Per-frame batch prompt** (up to ~10 frames per call):

> Here are N frames from a video at timestamps T1..TN. For each frame, describe what's on screen in 1–2 sentences. Flag: (a) UI changes from previous frame, (b) text visible on screen, (c) any moment that looks like a decision, action, or notable event. Also note the transcript text spoken during this window.

Save the output as `<workdir>/moments.md`:

```markdown
# Key moments — <title>

## 00:00:15 (frame-001.png)
**On screen**: Login form, email field focused
**Transcript**: "So you just open it up and..."
**Note**: Beginning of UI demo

## 00:00:45 (frame-002.png)
**On screen**: Dashboard with 4 cards
**Transcript**: "And here's where you see all your projects."
**Note**: Major view change — first time the dashboard appears
```

### Generate summary

After moments are identified, synthesize the whole video into `<workdir>/summary.md`:

```markdown
# Summary — <title>

**Source:** <source URL / file>
**Duration:** <hh:mm:ss>
**Watched at:** <date>
**Mode:** visual

## TL;DR
<2–4 sentences>

## Key moments
- 00:00:15 — <one-line>
- 00:00:45 — <one-line>

## Action items flagged
- <item> [timestamp]

## Decisions flagged
- <decision> [timestamp] — consider routing to /decide

## Quotes worth keeping
- "..." [timestamp]

## Open questions
- <question raised but not answered>
```

## Step 8 — If `multimodal` mode

### Backend selection

Direct provider video upload is `execution_unavailable`; a provider API key is
not a sandbox transport. Use **dense local frame analysis** only when the
current model/tool surface accepts the extracted images:

1. **Dense vision pass**:
   - Frame cadence: 1 frame per **3s** (much denser than visual mode)
   - Batch through Claude vision with the multimodal-analysis prompt
   - Slower and more expensive than Gemini for long videos — warn the user before running on >10min content

### Multimodal output

Same `summary.md` template as Step 7 + an extended section:

```markdown
## Multimodal observations
- **Body language / delivery**: <observations on talking-head video>
- **Pacing**: <fast/slow/uneven>
- **Visual style**: <brand audit, ad review, design observations>
- **Audio quality / atmosphere**: <music, silence, background>
```

Exact extra sections depend on the use case (brand audit, ad review, talk delivery review, client-call read). Use case is inferred from the source + the user's verbal framing when invoking.

## Step 9 — Optional: capture to second-brain

After any mode completes, offer:

> *"Want to capture this to second-brain? I'll write a `call-<slug>.md` (or `meeting-` / `note-` / `resource-`) to `${SECOND_BRAIN_VAULT:-$HOME/Documents/SecondBrain}/raw/` with the summary, source URL, and transcript link."*

Type prefix by source:

| Source | Prefix |
|---|---|
| Loom / Zoom / Riverside / Otter / call recording | `call-` |
| Meeting (own notes, not a transcript) | `meeting-` |
| Talk / keynote / conference | `note-` |
| Ad / landing-page video / marketing reference / competitor video | `resource-` |

File body: 1-line source, the summary, link to full workdir.

## Step 10 — Report

In chat:

- One-line headline: `<source> · <title> · <duration> · <mode> · <word count> words`
- Workdir path
- For `visual` / `multimodal`: brief list of top 3 key moments
- For all modes: any action items / decisions flagged for triage
- If captured to second-brain: that path too

## Sources reference

| Source | Download | Built-in transcript | Notes |
|---|---|---|---|
| YouTube | User-attached local file | User may also paste a transcript | Remote acquisition is unavailable |
| Loom | User-attached local file | User may also paste a transcript | Remote acquisition is unavailable |
| Vimeo | User-attached local file | User may also paste a transcript | Remote acquisition is unavailable |
| Riverside | User-attached local export | User may also attach its transcript | Remote acquisition is unavailable |
| Zoom | Local `.mp4` (downloaded recordings) | Sometimes (Zoom audio transcript file) | Client calls |
| X / IG / TikTok | `social-fetch` for typed metadata; user attaches media | No | Short-form |
| Local file | n/a | n/a | Drop a path |

## Composes with

- `social-fetch` — for X/IG/TikTok URL metadata (engagement, author, replies) before video processing
- `second-brain` — capture summary as `raw/call-<slug>.md`, `meeting-`, `note-`, or `resource-` per source type
- `decide` — when a video contains a flagged decision, route to `/decide` for structured capture
- `pm` — action items flagged in summary can be triaged to project boards
- `slide-deck` — talk recordings → outline extraction → deck draft (loop)
- `jab-hook` — quotes + clip-worthy moments from podcast/talk videos feed BIP/promo posts
- **`skillify from-video`** — primary use case for `visual` mode on process recordings. the user records themselves doing a workflow (Loom/screen-share), this skill extracts transcript + key visual moments, then `skillify` synthesizes the workflow into a SKILL.md. "Record once, AI converts to skill."

## Error handling

| Failure | Response |
|---|---|
| Video unavailable / private / region-locked | Report and stop |
| No subtitles + Whisper/model not installed locally | Report the missing preinstalled local dependency and stop transcription |
| ffmpeg missing (for visual/multimodal) | Report the missing preinstalled local dependency and stop frame extraction |
| Vision pass returns empty / unclear | Lower the frame count, retry, or fall back to transcript-only with a note |
| No advertised vision surface | Return `execution_unavailable` and offer transcript-only mode |

## Notes on quality

- **User picks depth, not the skill.** Transcript / visual / multimodal are 3 different cost + latency profiles. Long videos (>10 min) always confirm before spending on visual/multimodal.
- **Platform transcript first, Whisper second.** YouTube auto-subs, Loom transcripts, Riverside built-in transcripts — all free + instant when they exist. Fall back to MLX-Whisper local only when nothing platform-provided works.
- **MLX-Whisper local is the fast path on Mac.** M-series machines transcribe faster than real-time. Cloud Whisper is a distant second choice — costs money, network dependency, worse latency on typical durations.
- **Frame cadence by source type.** Screen-share / demos need 1 frame per 5s (UI changes fast); talking-head podcasts need 1 per 30s (slow change). Default 15s if unsure. Wrong cadence = missed key moments OR wasted vision-pass cost.
- **720p is plenty.** Ask the user for a 720p local attachment when possible;
  larger source files waste local processing time and storage.
- **Scene-change detection catches slide transitions.** When the video is a slide presentation, add `ffmpeg -vf "select='gt(scene,0.3)'"` to force a frame on each detected slide change — more reliable than pure time-based sampling.
- **Multimodal cost warning is non-optional.** Gemini multimodal on a 60-min video is meaningfully expensive. Warn before running; offer transcript-only as fallback if the user isn't sure.
- **Summary format includes routing hints.** `## Decisions flagged` + `## Action items flagged` sections signal `/decide` and `/pm` follow-ups. Downstream composability lives in the summary structure.
