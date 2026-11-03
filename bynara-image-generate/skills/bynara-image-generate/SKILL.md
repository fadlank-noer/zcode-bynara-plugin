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
  -d '{"model": "gpt-image-2", "size": "1024x1024", "prompt": "<the prompt>", "response_format": "b64_json"}' \
  > /tmp/gen_result.json
```

## Models and sizes — read models.json first

The authoritative model list — ids, supported sizes, per-image prices, and the current default — lives in **`models.json`** in this skill's directory (same folder as this SKILL.md). **Read it before generating** and drive every request from it: use `default_model` unless the user's wording matches another model's `routing_hints`, and take sizes and prices only from that file. When several models match and the user named one explicitly, use the named one; otherwise use `default_model`.

Rules that apply regardless of what the JSON contains:

- Do NOT send shorthand tiers like `1K`, `2K`, or `4K` as `"size"` — the API rejects them with `"Invalid image size."`. Always use explicit pixel dimensions from the JSON's `sizes` arrays. Default to `1024x1024` (square); use landscape/portrait ratios when the user's prompt implies one.
- If the user names a model that is not in the JSON (e.g. a removed `agnes-image-*` model), do not send it — tell the user which models the JSON lists instead.
- If `models.json` is missing or unreadable, fall back to `gpt-image-2` at `1024x1024` and tell the user the model list could not be loaded, so it may be out of date.

The default request body (check `default_model` in `models.json` for the current default):

```json
{"model": "gpt-image-2", "size": "1024x1024", "prompt": "<the prompt>", "response_format": "b64_json"}
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

For multiple input images use repeated `-F "image=@..."` fields. Add `-F "model=..."` (a model id from `models.json`) when the user specifies a model or quality tier. If the endpoint rejects multipart with a content-type error, retry with JSON: `-H "Content-Type: application/json"` and `-d '{"model": "...", "image": "<base64 of file>", "prompt": "..."}'` (encode with `base64 -w0 < file`).

## 3. Handle the response and save the image(s)

Always save results to disk so the user can view them.

1. **Save the raw curl output first** to a temp file (e.g. `/tmp/gen_result.json` or `/tmp/gen_result_N.json`). The base64 string can be very long and piping it directly may truncate or corrupt it. On Windows, curl may also report exit code 56 ("schannel: server closed abruptly") while still writing a usable body — judge by the file contents, not the exit code.
2. **Validate the JSON before decoding** (`JSON.parse` or `python -c "import json;json.load(open('gen_result.json'))"`). **Known API issue (observed 2026-11-03):** the server currently aborts large `b64_json` responses mid-body (~64 KB delivered of ~2 MB), leaving truncated JSON. If parsing fails, delete the file and retry the same request (up to 3 attempts, `sleep 15` between). If every attempt truncates, stop and tell the user the API is truncating responses right now (a server-side issue) instead of retrying further or falling back to the `url` field — the `url` field is a relative path that returns 404 when downloaded.
3. **Decode the base64** into the final image file. Use whichever tool is available in the environment:
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
4. Create an output directory in the current workspace, e.g. `generated-images/`.
5. Name files descriptively from the prompt (slugified, e.g. `red-apple-on-snow.png`); add `-2`, `-3` suffixes when `n > 1`.
6. Clean up temp files after decoding.

**Do not attempt to download via the `url` field** — it is a relative path that returns 404 when accessed directly (all known download-path variants return 404). The base64 route, with the retry procedure above, is the only workable method.

## 4. Report and handle errors

- On success: tell the user how many images were created and give each saved file path. Do not dump raw base64 into the conversation.
- On error (HTTP 4xx/5xx with `{"error": {"type", "message"}}`): show the API's `type` and `message` verbatim. `unauthorized` means `__BYNARA_API_KEY` is missing/invalid; `rate_limit` means the user should retry later. Suggest a fix, do not retry a 401 in a loop.
- If the response contains a URL but download fails, report the URL directly so the user can open it.
