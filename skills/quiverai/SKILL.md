---
name: quiverai
description: Use when creating, refining, animating, or vectorizing SVG assets with the QuiverAI MCP server, including model selection, reference-image prompting, structured prompt writing, direct source ingestion, generation, animation, vectorization, polling, and reading completed SVG content.
---

# QuiverAI SVG Generation

Use this skill when a user wants to create, refine, or vectorize SVG assets with the QuiverAI MCP server.

Before starting, confirm the QuiverAI MCP server is connected and its tools are available.

## Concepts (task vs creation vs gallery)

QuiverAI separates three read surfaces. Pick the tool that matches what the user means.

| Concept      | What it is                                                                                         | MCP tool                               | Typical use                                                                                                    |
| ------------ | -------------------------------------------------------------------------------------------------- | -------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| **Task**     | One whole request (generation, vectorization, or animation) that may produce one or more creations | `get_task`                             | Poll status after `create_*`; inspect all outputs and failure summary for that request                         |
| **Creation** | One generated SVG asset (one row in the gallery)                                                   | `get_creation`, `get_creation_content` | Metadata for one asset; **full SVG string** when the user wants the SVG; optional PNG preview for user display |
| **Gallery**  | The user's list of past creations                                                                  | `list_creations`                       | Browse prompts and ids; optionally include inline SVG per item                                                 |

**ID rules**

- `taskId` — from every `create_*` response; **only** for `get_task` (polling / whole-request status).
- `id` (creation id) — on gallery items and in `creationIds`; for `get_creation`, `get_creation_content`, and `{ "creationId": "..." }` animation sources.
- Never pass a `taskId` to `get_creation` or `get_creation_content` (the server returns a 400 with guidance).

**When the user asks for an SVG**

- They mean a **creation**, not a task. Resolve to `get_creation_content` with a **creation id**.
- If you only have a `taskId`, call `get_task` first, read `creations[].id`, then `get_creation_content` for the chosen creation.

## Core workflow (create)

1. Call `list_models` before choosing a model unless the user explicitly named one.
2. When the task continues from an existing QuiverAI creation (animating it, referencing it, or producing a follow-up), call `list_creations` first to find the right creation ID. Browse without content; do not pull SVG payloads at this stage.
3. If the user provides or asks to use a reference/source image, pass it directly to the create tool as `{ "url": "https://..." }`, `{ "base64": "...", "mediaType": "image/png" }`, or `{ "uploadId": "..." }`. Quiver fetches or decodes non-upload sources, validates them, stores them, and persists only upload IDs.
4. Pick the right create tool: `create_generation` for text-to-SVG, `create_vectorization` for raster-to-SVG, or `create_animation` to animate an existing SVG creation or SVG source.
5. Poll **`get_task`** with the returned **`taskId`** until status is terminal (`completed`, `failed`, or `stopped`).
6. Read outputs via **`creationIds`** or `get_task.creations[].id`.
7. Call `get_creation` for metadata when needed; call **`get_creation_content`** when the user wants the SVG. Use `includePng: true` when the user should see the creation visually; render the returned PNG preview to the user.

## Gallery workflow (browse → pick → fetch SVG)

Use **`list_creations`** as the gallery. It is the right tool when the user wants to see what they already made, search by prompt, or pick an asset to open.

1. **Browse metadata (default)** — `list_creations` with `includeContent: false` (or omit it). Each item includes `id`, `taskId`, `prompt`, `method`, `modelId`, `status`, and timestamps. Use prompts to choose the right creation.
2. **Browse with inline SVG (optional)** — `list_creations` with `includeContent: true` only when you need SVG for many items at once; prefer the two-step flow below for large galleries.
3. **Open one creation** — After choosing an `id` from the gallery:
   - `get_creation` for metadata (references, rating, capability fields) when needed.
   - **`get_creation_content`** for the full SVG string when the user wants the file or code.
   - **`get_creation_content`** with `includePng: true` when the user wants to see the creation visually. The tool returns both the SVG and a PNG preview plus a `renderInstruction` telling you to display the PNG to the user.

Filter gallery calls with `method`, `status`, `limit`, and `cursor` when the user narrows the scope (for example only completed generations).

## Model Selection

- Inspect each `list_models` entry's `access` field before selecting a model. Use models with `access.state: "ok"` directly. If a model is `locked`, explain that it requires one of `requiredPlans` or on-demand credits before using it.
- Inspect operation-level `availability` before calling paid operations. For example, only call `create_animation` when the selected model's `availability.animate.access.state` is `"ok"`; if it is `"locked"`, report the plan/credit requirement instead of trying the tool.
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

When a reference image is available, pass it in `create_generation.references`. Be strict and explicit:

- Preserve the exact style, color scheme, composition, typography, and structure when the user wants a close match.
- State what should change separately from what should be preserved.
- Use the reference for color combinations, illustration style, composition, and typography.
- If the user wants a variation, derive the prompt from the reference first, then modify only the requested dimensions.
- Prefer public image URLs when the chat host exposes uploaded files as public URLs. Use base64 only when the host gives image bytes directly. Existing completed Quiver upload IDs are still accepted.

Avoid vague reference prompts such as:

```text
Create a vector like the reference image.
```

Prefer explicit reference prompts such as:

```text
Create a flat vector illustration matching the reference image's geometric style, muted color palette, centered composition, and clean typography. Preserve the overall structure and visual hierarchy. Change only the subject to a delivery drone carrying a small package.
```

## Animation

Use `create_animation` when the user wants to animate an SVG that already exists.

- Supply exactly one `source`: `{ "creationId": "..." }` for an existing QuiverAI creation, or an SVG source as `{ "url": "https://..." }`, `{ "base64": "...", "mediaType": "image/svg+xml" }`, or `{ "uploadId": "..." }`.
- If the user references something they generated earlier ("animate the drone I made yesterday"), call `list_creations` first to find the creation ID.
- Animation tasks always return a single creation.
- The optional `prompt` controls animation direction, not visual style. The source SVG already defines style; keep the prompt short and concrete (for example, "gentle drift loop", "pulse the central element"). Do not restate color, composition, or typography.
- Poll the returned `taskId` with `get_task`, then fetch SVG via `get_creation_content` on the resulting creation id.

## Strong Use Cases

QuiverAI is especially strong at:

- Flat vector illustrations
- Color palettes and combinations
- Carrying forward a specific reference style
- Wordmarks and typography
- Raster-to-SVG vectorization when the source image is not heavily textured
- Engineering and fashion-style sketches with Arrow 1.1 Max

## Result Handling

- Show the user completed creation choices (prompt + id) before fetching every SVG payload.
- Default gallery browse: `list_creations` without content; fetch SVG with `get_creation_content` only for the chosen creation id.
- If a task fails, use `get_task` failure fields and adjust the next prompt from the failure mode.
- If outputs are close but over-detailed, simplify the prompt and prefer Arrow 1.1 over Arrow 1.1 Max.
- If outputs miss a reference, make preservation language stricter and separate preserved attributes from requested changes.
