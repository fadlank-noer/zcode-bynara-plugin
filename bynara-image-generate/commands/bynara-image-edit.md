---
description: Edit an existing image file using the Bynara Images API
argument-hint: <image-path> <edit instruction>
---

Edit an existing image with the Bynara Images API by following the `bynara-image-generate` skill exactly (endpoint `POST https://api-images.bynara.id/v1/images/edits`, multipart form with the source image, auth via the `__BYNARA_API_KEY` environment variable, save the result to disk and report the file path).

The first argument is the path to the source image; the rest is the edit instruction:

$ARGUMENTS
