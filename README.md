# zcode-bynara-plugin

A collection of [ZCode](https://zcode.ai) plugins for the Bynara API (`api-images.bynara.id`): generate images and videos directly from ZCode.

## Marketplace contents

| Plugin | Version | What it does |
|---|---|---|
| `bynara-image-generate` | 0.1.2 | Generate images from a text prompt or edit existing images (`agnes-image-2.1-flash`, `agnes-image-2.0-flash`, `gpt-image-2`) |
| `bynara-video-generate` | 0.1.0 | Generate short MP4 clips — text-to-video (**t2v**), image-to-video (**i2v**, the image becomes the first frame), reference-to-video (**r2v**, 1–3 reference images merged into a new scene) — with `agnes-video-v2.0` |

## Installation

1. In ZCode, open **Plugin Marketplace → Add → Add Plugin Marketplace** and paste this folder's path.
2. Pick the **zcode-bynara-plugin** marketplace, then **Install** the plugins you want.
3. Set your Bynara API key, then restart ZCode:

   ```
   setx __BYNARA_API_KEY "sk-nry-..."
   ```

   Never commit an API key to this repository — keys are read from the `__BYNARA_API_KEY` environment variable.

## Usage

| Command | What it does |
|---|---|
| `/bynara-image <prompt>` | Generate an image from a text prompt |
| `/bynara-image-edit <image> <instruction>` | Edit an existing image |
| `/bynara-video <prompt>` | Generate a video from a text prompt (takes 1–5 minutes) |
| `/bynara-video-animate <image> [image2 image3] <prompt>` | Animate a single image (becomes the first frame) or merge 1–3 reference images into a new scene |

The `bynara-image-generate` and `bynara-video-generate` skills also trigger automatically from natural-language prompts (e.g. "draw me...", "turn this photo into a video...").

## Notes

- Every request is billed per use on your Bynara account.
- Videos: 3–15 seconds long (billed per second; 720p Rp100/sec, 1080p Rp250/sec). Results are downloaded to disk automatically because server-side files expire after 3 hours.
- Images: results arrive as base64 and are saved to `generated-images/`; videos go to `generated-videos/`.
