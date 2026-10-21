---
description: Generate a video from a text prompt using the Bynara Videos API
argument-hint: <prompt>
---

Generate a video with the Bynara Videos API by following the `bynara-video-generate` skill exactly (mode `t2v`, endpoint `POST https://api-images.bynara.id/v1/videos`, poll until `succeeded`, auth via the `__BYNARA_API_KEY` environment variable, download the finished MP4 to disk and report the file path).

The user's video prompt:

$ARGUMENTS
