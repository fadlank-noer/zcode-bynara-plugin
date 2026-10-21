---
name: bynara-image-generate
description: Generate new images from a text prompt or edit an existing image file using the Bynara Images API. Use when the user asks to create, generate, draw, or make an image/picture/illustration, or to edit, modify, retouch, or extend an existing image.
---

# Bynara Image Generation

Generate images from a prompt or edit an existing image via the Bynara Images API. Both endpoints return OpenAI-style JSON: `{"data": [{"url": ...}]}` or `{"data": [{"b64_json": ...}]}`.

## Requirements

- API key must be available in the `__BYNARA_API_KEY` environment variable. If it is unset or empty, stop and tell the user to set it (Windows: `setx __BYNARA_API_KEY "sk-..."` then restart ZCode, or add it to the ZCode settings `env` block). Never print the key value in output or logs.
- Use `curl` (available in Git Bash / most shells). Always pass a generous timeout (`-m 300`) — image generation can be slow.

## 1. Generate an image from a prompt

Always request `response_format: "b64_json"` so you receive the image as base64 — the URL field in the response is a **relative path only** (e.g. `/v1/images/<id>/download`) and will return 404 when downloaded directly. Decoding from base64 is reliable and avoids that pitfall.

```bash
curl -sS -m 300 -X POST https://api-images.bynara.id/v1/images/generations \
  -H "Authorization: Bearer $__BYNARA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "agnes-image-2.1-flash", "size": "1024x1024", "prompt": "<the prompt>", "response_format": "b64_json"}' \
  > /tmp/gen_result.json
```

## Models and sizes

Only these models exist on the Bynara Images API. Default to `agnes-image-2.1-flash` unless the user asks for something else.

| model | supported sizes | notes |
|---|---|---|
| `agnes-image-2.0-flash` | `1024x1024`, `1536x1024`, `1024x1536`, `2048x2048`, `3072x2048`, `2048x3072`, `4096x4096`, `6144x4096`, `4096x6144` | cheapest flash tier |
| `agnes-image-2.1-flash` | `1024x1024`, `1536x1024`, `1024x1536`, `2048x2048`, `3072x2048`, `2048x3072`, `4096x4096`, `6144x4096`, `4096x6144` | newest flash — default |
| `gpt-image-2` | `1024x1024`, `1536x1024`, `1024x1536`, `2048x2048`, `3072x2048`, `2048x3072`, `4096x4096`, `6144x4096`, `4096x6144` | flat pricing at every size — use when the user wants the highest quality |

**Important:** Do NOT use shorthand sizes like `1K`, `2K`, or `4K` — the API rejects them with `"Invalid image size."`. Always use explicit pixel dimensions (see table above). Default to `1024x1024` (square). Use landscape/portrait ratios when the user's prompt implies one.

Pick the model from the user's wording: "cepat/murah" → a `flash` model; "kualitas tertinggi/terbaik" → `gpt-image-2`; "4K/resolusi tinggi" → any model with a `4096x…` or `6144x…` size. If the user names a model not in this table, tell them which models are available instead of sending it.

Include `"model"` in the request body whenever known; the default request becomes:

```json
{"model": "agnes-image-2.1-flash", "size": "1024x1024", "prompt": "<the prompt>", "response_format": "b64_json"}
```

Other optional JSON fields (include only when the user asks): `"n": 2`. Do not include `"response_format"` explicitly — `b64_json` is the default strategy because URL-based downloads fail on this API.

## 2. Edit an existing image

Send `multipart/form-data` with the source image and an edit instruction:

```bash
curl -sS -m 300 -X POST https://api-images.bynara.id/v1/images/edits \
  -H "Authorization: Bearer $__BYNARA_API_KEY" \
  -F "image=@/absolute/path/to/image.png" \
  -F "prompt=<the edit instruction>"
```

For multiple input images use repeated `-F "image=@..."` fields. Add `-F "model=..."` (same model table as above) when the user specifies a model or quality tier. If the endpoint rejects multipart with a content-type error, retry with JSON: `-H "Content-Type: application/json"` and `-d '{"model": "...", "image": "<base64 of file>", "prompt": "..."}'` (encode with `base64 -w0 < file`).

## 3. Handle the response and save the image(s)

Always save results to disk so the user can view them.

1. **Save the raw curl output first** to a temp file (e.g. `/tmp/gen_result.json` or `/tmp/gen_result_N.json`). The base64 string can be very long and piping it directly may truncate or corrupt it.
2. **Decode the base64** into the final image file. Use whichever tool is available in the environment:
   - **Node.js** (reliable cross-platform):
     ```bash
     node -e "
     const fs = require('fs');
     const data = JSON.parse(fs.readFileSync('gen_result.json'));
     const buf = Buffer.from(data.data[0].b64_json, 'base64');
     fs.writeFileSync('generated-images/<descriptive-name>.png', buf);
     console.log('Saved', buf.length, 'bytes');
     "
     ```
   - **Python** (if available):
     ```python
     import json, base64
     with open('gen_result.json') as f:
         data = json.load(f)
     buf = base64.b64decode(data['data'][0]['b64_json'])
     with open('generated-images/<name>.png', 'wb') as out:
         out.write(buf)
     print(f'Saved {len(buf)} bytes')
     ```
3. Create an output directory in the current workspace, e.g. `generated-images/`.
4. Name files descriptively from the prompt (slugified, e.g. `red-apple-on-snow.png`); add `-2`, `-3` suffixes when `n > 1`.
5. Clean up temp files after decoding.

**Do not attempt to download via the `url` field** — it is a relative path that returns 404 when accessed directly. The base64 route is the only reliable method.

## 4. Report and handle errors

- On success: tell the user how many images were created and give each saved file path. Do not dump raw base64 into the conversation.
- On error (HTTP 4xx/5xx with `{"error": {"type", "message"}}`): show the API's `type` and `message` verbatim. `unauthorized` means `__BYNARA_API_KEY` is missing/invalid; `rate_limit` means the user should retry later. Suggest a fix, do not retry a 401 in a loop.
- If the response contains a URL but download fails, report the URL directly so the user can open it.
