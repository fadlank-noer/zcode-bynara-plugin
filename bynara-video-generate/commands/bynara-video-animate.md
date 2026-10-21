---
description: Animate an image (i2v) or compose 1-3 reference images into a video (r2v) using the Bynara Videos API
argument-hint: <image-path> [image2 image3] <prompt>
---

Animate image(s) into a video with the Bynara Videos API by following the `bynara-video-generate` skill exactly: use mode `i2v` for exactly one image (it becomes the video's first frame) or `r2v` for 1–3 reference images (merged into a new scene). Send the image(s) as multipart fields `image`, `image2`, `image3` to `POST https://api-images.bynara.id/v1/videos`, poll until `succeeded`, auth via the `__BYNARA_API_KEY` environment variable, download the finished MP4 to disk and report the file path.

The first argument(s) are the image path(s); the rest is the motion prompt:

$ARGUMENTS
