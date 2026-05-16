---
name: QuiverAI
description: Use when creating, refining, or vectorizing SVG assets with the QuiverAI MCP server, including model selection, reference-image prompting, structured prompt writing, upload handling, generation, vectorization, polling, and reading completed SVG content.
metadata:
  author: quiverai
  version: "2026.05.16"
---

# QuiverAI SVG Generation

Use this skill when a user wants to create, refine, or vectorize SVG assets with the QuiverAI MCP server.

Before starting, confirm the QuiverAI MCP server is connected and its tools are available.

## Core Workflow

1. Call `list_models` before choosing a model unless the user explicitly named one.
2. If the user provides or asks to use a reference image, call `create_upload`, upload the image to the returned URL, then pass the upload ID as a reference.
3. Call `create_generation` for text-to-SVG work, or `create_vectorization` when the task is raster-to-SVG conversion.
4. Poll `get_batch` until the batch reaches `completed`, `failed`, or another terminal status.
5. For completed outputs, call `get_artwork` for metadata.
6. Call `get_artwork_content` only for selected assets or when SVG content is needed.

## Model Selection

- Prefer Arrow 1.1 for general-purpose generation. It is the default choice for stability, speed, and accuracy.
- Use Arrow 1.1 Max for detail-heavy outputs, refinement-focused generations, engineering sketches, fashion sketches, or cases where richer detail is worth slower generation.
- Be careful with Arrow 1.1 Max when the user wants a simple icon, clean mark, or minimal illustration; extra detail can create more cleanup work.
- Avoid older model versions unless the user specifically asks for them or `list_models` shows Arrow 1.1 is unavailable.

## Prompt Craft

Reference-driven prompting and structured prompting produce the most predictable QuiverAI results.

When writing a generation prompt, prefer this structure:

```json
{
  "subject": "",
  "intended_use": "",
  "style": "",
  "composition": "",
  "color_palette": "",
  "typography": "",
  "preserve_from_reference": "",
  "change_from_reference": "",
  "constraints": ""
}
```

Fill only fields that help the task. Do not add decorative requirements the user did not ask for.

## Reference Images

Use reference images when the user has a desired style, color palette, layout, typography direction, or previous generation to build on.

When a reference image is available, be strict and explicit:

- Preserve the exact style, color scheme, composition, typography, and structure when the user wants a close match.
- State what should change separately from what should be preserved.
- Use the reference for color combinations, illustration style, composition, and typography.
- If the user wants a variation, derive the prompt from the reference first, then modify only the requested dimensions.

Avoid vague reference prompts such as:

```text
Create a vector like the reference image.
```

Prefer explicit reference prompts such as:

```text
Create a flat vector illustration matching the reference image's geometric style, muted color palette, centered composition, and clean typography. Preserve the overall structure and visual hierarchy. Change only the subject to a delivery drone carrying a small package.
```

## Strong Use Cases

QuiverAI is especially strong at:

- Flat vector illustrations
- Color palettes and combinations
- Carrying forward a specific reference style
- Wordmarks and typography
- Raster-to-SVG vectorization when the source image is not heavily textured
- Engineering and fashion-style sketches with Arrow 1.1 Max

## Result Handling

- Show the user completed artwork choices before fetching every SVG payload.
- Prefer `list_gallery` without content for browsing; use `get_artwork_content` for selected assets.
- If a batch fails, report the returned failure summary directly and adjust the next prompt based on the failure mode.
- If outputs are close but over-detailed, simplify the prompt and prefer Arrow 1.1 over Arrow 1.1 Max.
- If outputs miss a reference, make preservation language stricter and separate preserved attributes from requested changes.
