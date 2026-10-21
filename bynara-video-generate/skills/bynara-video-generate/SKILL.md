---
name: bynara-video-generate
description: Generate short MP4 videos from a text prompt or animate reference images using the Bynara Videos API (t2v, i2v, r2v). Use when the user asks to create, generate, or make a video/clip/animation/motion, or to animate an image or combine images into a moving video.
---

# Bynara Video Generation

Generate short MP4 clips via the Bynara Videos API (`https://api-images.bynara.id/v1/videos`). The endpoint is asynchronous: the POST immediately returns HTTP 202 with a task `id` and `status: "pending"`; poll `GET /v1/videos/{id}` every ~5 seconds until `status` is `succeeded`, then download the MP4. Generation typically takes 1–5 minutes and the finished file expires after 3 hours — always download it to disk.

## Requirements

- API key must be available in the `__BYNARA_API_KEY` environment variable. If it is unset or empty, stop and tell the user to set it (Windows: `setx __BYNARA_API_KEY "sk-..."` then restart ZCode, or add it to the ZCode settings `env` block). Never print the key value in output or logs.
- Use `curl` (available in Git Bash / most shells).
- Billing is per second of video: 720p Rp100/detik ($0.01/detik), 1080p Rp250/detik. Pick sensible defaults, but do not re-ask when the user already specified duration/resolution.

## 1. Pick the mode

| mode | what it does | images |
|---|---|---|
| `t2v` | text-to-video — generated purely from the prompt | none |
| `i2v` | image-to-video — the image becomes the video's FIRST frame; the model animates from that exact composition | exactly 1 |
| `r2v` | reference-to-video — images act as subject/style references (character, object, clothing) merged into a NEW scene from the prompt, not bound to any source frame | 1–3 |

Pick from the request: a prompt only → `t2v`; "animate this image / make it move" → `i2v`; "put this character/object into a scene" with 1–3 images → `r2v`. Example r2v: image 1 a wizard, image 2 a fire dragon, image 3 a castle, prompt "The wizard battles the fire dragon in the castle courtyard".

## 2. Model

`agnes-video-v2.0` is currently the ONLY available video model — use it for every mode. If the user names another model, tell them `agnes-video-v2.0` is the only one available. Pricing: 720p Rp100/sec, 1080p Rp250/sec, max clip 18 sec.

## 3. Submit the generation task

### t2v — JSON body

```bash
curl -sS -m 60 -X POST https://api-images.bynara.id/v1/videos \
  -H "Authorization: Bearer $__BYNARA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "agnes-video-v2.0",
    "mode": "t2v",
    "prompt": "<the prompt>",
    "resolution": "720p",
    "ratio": "16:9",
    "duration": 5
  }' > /tmp/byvideo_task.json
```

### i2v / r2v — multipart form

```bash
curl -sS -m 120 -X POST https://api-images.bynara.id/v1/videos \
  -H "Authorization: Bearer $__BYNARA_API_KEY" \
  -F "image=@/absolute/path/to/source.png" \
  -F "model=agnes-video-v2.0" \
  -F "mode=i2v" \
  -F "prompt=<the motion instruction>" \
  -F "resolution=720p" \
  -F "duration=5" \
  > /tmp/byvideo_task.json
```

For `r2v` add `-F "image2=@..."` and `-F "image3=@..."` (up to 3 images total). Public image URLs may be sent as JSON instead: `"image_urls": ["https://...", ...]`.

### Parameters

- `prompt` — required, up to 3,500 characters. Optional `"negative_prompt"` (same limit), JSON body only.
- `resolution` — `720p` (default) or `1080p`.
- `ratio` — `16:9`, `9:16`, `1:1`, `4:3`, `3:4`, `4:5`, `5:4`, `9:21`, `21:9`. t2v only; i2v follows the first frame's aspect ratio. Default `16:9`.
- `duration` — seconds, 3–15. Default 5.
- `seed` — optional integer [0, 2147483647] for reproducibility.
- `watermark` — optional boolean, provider watermark toggle.
- `enable_audio` — optional, adds a soundtrack; CodeBuddy/Seedance models only and OFF by default (audio renders can be rejected upstream by the copyright check). Do not send it for `agnes-video-v2.0`.

A successful submit returns HTTP 202: `{"id": "...", "status": "pending", "created": ...}`.

## 4. Poll until finished

`GET /v1/videos/{id}` every ~5 seconds until `status` is `succeeded` (typically 1–5 minutes; give up after ~10 minutes and report the last status). Run the loop in ONE bash call with a generous command timeout (up to 10 minutes), e.g. with Node:

```bash
for i in $(seq 1 90); do
  sleep 5
  curl -sS -m 60 "https://api-images.bynara.id/v1/videos/<id>" \
    -H "Authorization: Bearer $__BYNARA_API_KEY" > /tmp/byvideo_status.json
  if node -e "const s=JSON.parse(require('fs').readFileSync('/tmp/byvideo_status.json','utf8')).status; process.exit(['succeeded','failed'].includes(s)?0:1)"; then break; fi
done
cat /tmp/byvideo_status.json
```

(Without Node, poll in a few separate short calls and read the JSON each time.)

## 5. Download the MP4

The status response's `url` field is a RELATIVE path (e.g. `/v1/videos/<id>/download`). Prefix the API host and send the auth header:

```bash
mkdir -p generated-videos
curl -sS -m 300 -L "https://api-images.bynara.id/v1/videos/<id>/download" \
  -H "Authorization: Bearer $__BYNARA_API_KEY" \
  -o "generated-videos/<descriptive-name>.mp4"
```

Name files descriptively from the prompt (slugified, e.g. `cardboard-city-night.mp4`). Verify the file exists and is a real video (hundreds of KB at minimum for 3s); if the download returns JSON/HTML instead of an MP4, retry once without the auth header, then report if it still fails.

## 6. Report and handle errors

- On success: report the saved MP4 path plus the mode/duration/resolution used.
- On error (HTTP 4xx/5xx with `{"error": {"type", "message"}}`): show `type` and `message` verbatim. `unauthorized` means `__BYNARA_API_KEY` is missing/invalid; `rate_limit` means the user should retry later. Suggest a fix, do not retry a 401 in a loop.
- If polling ends with `status: "failed"`, report the status JSON verbatim instead of retrying blindly.
- Remind the user the upstream copy expires after 3 hours — the downloaded file is their permanent copy.
