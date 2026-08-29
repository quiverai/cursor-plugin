---
name: quiverai
description: Use when creating, refining, animating, or vectorizing SVG assets with the QuiverAI MCP server, including model selection, reference-image prompting, structured prompt writing, direct source ingestion, generation, animation, vectorization, polling, and reading completed SVG content.
---

# QuiverAI SVG Generation

Use this skill when a user wants to create, refine, or vectorize SVG assets with the QuiverAI MCP server.

Before starting, confirm the QuiverAI MCP server is connected and its tools are available.

## Agent behavior

- Do not call tools for questions about QuiverAI capabilities, identity, or general guidance.
- If required inputs are missing, ask a short clarifying question before calling a create tool.
- Keep responses concise and action-oriented.
- Do not use emoji.
- Do not discuss backend implementation details, deployment details, internal assistant model IDs, or third-party service details.
- You may mention user-facing SVG model IDs such as `arrow-*` models when relevant to generation or vectorization tasks. Do not describe them as your own underlying chat model; they are only SVG tool models.

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
- Never pass a `taskId` to `get_creation` or `get_creation_content` (the server returns an MCP tool error result with guidance).

**When the user asks for an SVG**

- They mean a **creation**, not a task. Resolve to `get_creation_content` with a **creation id**.
- If you only have a `taskId`, call `get_task` first, read `creations[].id`, then `get_creation_content` for the chosen creation.

## Core workflow (create)

1. Call `list_models` before choosing a model unless the user explicitly named one.
2. When the task continues from an existing QuiverAI creation (animating it, referencing it, or producing a follow-up), call `list_creations` first to find the right creation ID. Browse without content; do not pull SVG payloads at this stage.
3. If the user provides or asks to use a reference/source image, pass it directly to the create tool as `{ "url": "https://...", "filename": "reference.png" }`, `{ "base64": "...", "mediaType": "image/png", "filename": "reference.png" }`, or `{ "uploadId": "..." }`. `filename` is optional. Quiver fetches or decodes non-upload sources, validates them, stores them, and persists only upload IDs.
4. Pick the right create tool: `create_generation` for text-to-SVG, `create_vectorization` for raster-to-SVG, or `create_animation` to animate an existing SVG creation or SVG source.
5. `create_generation` accepts optional `n` from 1 to 16 (default 1); `create_animation` accepts optional `n` from 1 to 4 (default 1).
6. Poll **`get_task`** with the returned **`taskId`** until status is terminal (`completed`, `failed`, or `stopped`). If every creation is `completed` but the parent task still says `generating`, stop polling, treat the completed creations as ready, and record the status mismatch in your response.
7. Read output creation ids via **`creationIds`** or `get_task.creations[].id`. Treat `get_task` as status-only; do **not** show SVG snapshots or SVG strings from task polling as final output.
8. For each completed output you will present, call **`get_creation_content`** with `{ "id": "<creation id>", "includePng": true }`. By default, show only the returned PNG preview to the user. Show SVG text/code only when the user explicitly asks for the SVG source or file content.

## Gallery workflow (browse → pick → fetch SVG)

Use **`list_creations`** as the gallery. It is the right tool when the user wants to see what they already made, search by prompt, or pick an asset to open.

1. **Browse metadata (default)** — `list_creations` with `includeContent: false` (or omit it). Each item includes `id`, `taskId`, `prompt`, `method`, `modelId`, `status`, and timestamps. Use prompts to choose the right creation.
2. **Browse with inline SVG (optional)** — `list_creations` with `includeContent: true` only when you need SVG for many items at once; prefer the two-step flow below for large galleries.
3. **Open one creation** — After choosing an `id` from the gallery:
   - `get_creation` for metadata (references, rating, capability fields) when needed.
   - **`get_creation_content`** for the full SVG string when the user wants the file or code.
   - **`get_creation_content`** with `includePng: true` when presenting the creation visually. The tool returns both the SVG and a PNG preview plus a `renderInstruction` telling you to display the PNG to the user. Default visual presentation should show the PNG, not inline SVG.

Filter gallery calls with `method` (`generate`, `vectorize`, or `animate`), `status`, `limit` (1–100), and `cursor` when the user narrows the scope (for example only completed generations).

## Model Selection

- Inspect each `list_models` entry's `access` field before selecting a model. Use models with `access.state: "ok"` directly. If a model is `locked`, explain that it requires one of `requiredPlans` or on-demand credits before using it.
- Inspect operation-level `availability` before calling paid operations. For example, only call `create_animation` when the selected model's `availability.animate.access.state` is `"ok"`; if it is `"locked"`, report the plan/credit requirement instead of trying the tool.
- Prefer Arrow 1.1 for general-purpose generation. It is the default choice for stability, speed, and accuracy.
- Use Arrow 1.1 Max for detail-heavy outputs, refinement-focused generations, engineering sketches, fashion sketches, or cases where richer detail is worth slower generation.
- Be careful with Arrow 1.1 Max when the user wants a simple icon, clean mark, or minimal illustration; extra detail can create more cleanup work.
- Avoid older model versions unless the user specifically asks for them or `list_models` shows Arrow 1.1 is unavailable.

## Prompt Craft

Reference-driven prompting and structured prompting produce the most predictable QuiverAI results.

When writing or improving an SVG/vector creation prompt on the user's behalf, keep it simple, concrete, and suited to SVG/vector output. Avoid raster/photo-oriented prompt language such as photorealism, camera lenses, depth of field, lighting rigs, bokeh, render quality, or image effects unless the user explicitly asks for that style. For open-ended requests such as "surprise me", prefer one clear vector concept over a dense art-direction paragraph.

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
- The optional `n` controls the number of animations and may be 1 to 4 (default 1). Animation tasks can return multiple creations.
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

- After generation or vectorization completes, fetch the completed creation content with `get_creation_content({ "id": "<creation id>", "includePng": true })` before presenting results. Show the returned PNG preview as the default final output; do not show task-poll SVG as final.
- Default gallery browse: `list_creations` without content; fetch SVG with `get_creation_content` only for the chosen creation id.
- Do not auto-run more paid `create_generation` or `create_vectorization` calls just because an output looks bad. Ask the user before spending another generation or vectorization call unless the user already gave a fixed iteration limit.
- If the user gave a fixed iteration limit, do not exceed it. When the limit is reached, report the best result and any remaining issues instead of starting another paid call.
- If `get_task` shows all creations completed while the parent task remains `generating`, fetch the completed creations with `get_creation_content({ "includePng": true })` and mention the task/creation status mismatch.
- If a task fails, use `get_task` failure fields and adjust the next prompt from the failure mode.
- If outputs are close but over-detailed, simplify the prompt and prefer Arrow 1.1 over Arrow 1.1 Max.
- If outputs miss a reference, make preservation language stricter and separate preserved attributes from requested changes.
- After an SVG/vector creation has been generated, do not suggest vectorizing it; suggest editing, iterating, exporting, or using it instead.
