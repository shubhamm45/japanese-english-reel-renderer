# Japanese with English Reel Renderer

Design v2 uses a warm cream background, coral/mint/ink lesson panels, larger headings, learning cues, three-step navigation, and gentle zoom/fade motion.

This bundle renders a narrated, three-card, 1080 × 1920 Japanese lesson for English-speaking learners as an H.264/AAC MP4. The teacher's narration is in English; Japanese appears only as the phrases being taught (always paired with romaji on the cards). An n8n workflow dispatches the render via `workflow_dispatch` and polls the Pages manifest before publishing to Instagram, Facebook and YouTube.

## 1. Upload and enable GitHub Pages

1. Create a **public** GitHub repository with default branch `main`.
2. Copy this folder's contents into the repository root. Include the hidden `.github/workflows/render-reel.yml` file.
3. Open Settings → Pages → Build and deployment → Source → **Deploy from a branch**, then select the `gh-pages` branch (the workflow pushes rendered output there directly; the GitHub Actions source mode 404s with this setup).
4. Open Actions → Render Japanese for English Speakers → Run workflow. Enter a fresh request ID, such as `ja-demo-20261010-001`, and the JSON from `sample-lesson.json`.
5. Wait for the render job to succeed. Open `PAGES_BASE_URL/requests/ja-demo-20261010-001.json`. Open its `video_url` and verify that the reel plays with English speech and Japanese cards (not boxes).

The workflow stores previously rendered files in a generated `gh-pages` branch. Use this repository only for this renderer. It preserves all old videos so that a new render does not remove a video Instagram is still fetching. Periodically remove old files from `gh-pages` once publishing is confirmed.

## 2. Qwen output

Ask Qwen to return only this JSON object, without markdown fences:

```json
{
  "title": "Say good morning like a native",
  "phrase": "おはようございます",
  "phrase_romaji": "ohayou gozaimasu",
  "meaning_en": "Good morning (polite).",
  "usage_en": "Use it when greeting anyone before noon. It is polite, so it works for coworkers and shop staff.",
  "example": "おはようございます。今日もいい天気ですね。",
  "example_en": "Good morning. The weather is nice again today.",
  "say_it": "Today's Japanese phrase is おはようございます — ohayou gozaimasu — which means good morning.",
  "get_it": "Say it to anyone you meet before noon. It is polite, so it also works at the office.",
  "use_it": "For example: おはようございます。今日もいい天気ですね。 Now say it out loud with me!",
  "level": "Beginner", "category": "phrases", "topic": "greetings",
  "caption": "English caption, max 1800 chars"
}
```

The teacher speaks English and teaches Japanese words. `say_it`/`get_it`/`use_it` are the English narration scripts (max 200/120/260 chars) with Japanese words woven in; n8n turns them into voiceover audio via Qwen TTS (voice Cherry, auto language detection) and passes the URLs as `narration_audio` — the renderer fails loudly if any narration URL is missing or has no audio stream. Limits: title 100; phrase/example/example_en 180 each; phrase_romaji 120; meaning_en 180; usage_en 300. Japanese text needs a CJK font — the workflow installs `fonts-noto-cjk` and fails loudly if none is found (DejaVu cannot render Japanese). The renderer rejects oversized text and videos above 120 seconds or 90 MiB.

## 3. Dispatch the render (n8n)

HTTP request:

- Method: `POST`
- URL: `https://api.github.com/repos/OWNER/REPO/actions/workflows/render-reel.yml/dispatches`
- `Accept`: `application/vnd.github+json`
- `Authorization`: `Bearer YOUR_TOKEN`
- `X-GitHub-Api-Version`: `2022-11-28`
- `Content-Type`: `application/json`

Use a fine-grained GitHub token scoped to this repository with **Actions: read and write**. `make-dispatch-body.json` shows the shape of the body. `inputs.lesson_json` is a JSON **string**, so the caller must escape it correctly.

Create a fresh request ID once per lesson (prefix `ja-`) and never reuse one. A successful dispatch starts the workflow; its response body is not the video URL.

## 4. Poll the manifest

Read:

`PAGES_BASE_URL/requests/REQUEST_ID.json`

A successful result has this form:

```json
{
  "request_id": "ja-demo-20261010-001",
  "status": "ready",
  "video_url": "https://OWNER.github.io/REPO/reels/ja-demo-20261010-001/reel.mp4",
  "design_version": 2,
  "test_audio": false
}
```

Handle HTTP 404 as "still waiting". Require matching request ID, `status = ready`, `test_audio = false` and `design_version = 2` before publishing. Stop waiting after ~20 minutes, mark failed, and inspect the matching Actions run. A missing JSON is never permission to publish an earlier video.

For the Instagram Reel, map only the returned `video_url` and use `cover_url` as the thumbnail source. Confirm the URL is publicly fetchable before the first real publish.

## 5. Local checks

Requires Python 3.12, FFmpeg/ffprobe and a Japanese font (e.g. `fonts-noto-cjk`, or set `REEL_FONT_JA` to a suitable TTF/OTC).

```sh
pip install -r requirements.txt
python render_reel.py --lesson sample-lesson.json --request-id local-demo-001 --base-url https://example.com --public public --test-audio
```

For offline media verification only, add `--test-audio`. This generates tones instead of speech and marks the manifest `test_audio: true`; never publish such a result.

## Included files

- `.github/workflows/render-reel.yml`: dispatch, render, preserve and deploy
- `render_reel.py`: English narration, Japanese/English card rendering (per-script fonts), encoding and validation
- `requirements.txt`: Python dependencies
- `sample-lesson.json`: lesson schema/example
- `make-dispatch-body.json`: example GitHub dispatch HTTP body
- `README.md`: setup and integration

No GitHub credentials, tokens or Instagram secrets are included. Live verification requires your actual repository and n8n workflow.
