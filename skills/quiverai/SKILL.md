---
name: quiverai
description: Use when generating, vectorizing, animating, or editing SVG assets with the QuiverAI MCP server, including model selection, reference-image prompting, structured prompt writing, direct source ingestion, generation, vectorization, animation, editing, and reading completed SVG content.
---

# QuiverAI SVG artwork

Use this skill when a user wants to generate, vectorize, animate, or edit SVG assets with the QuiverAI MCP server.

Before starting, confirm the QuiverAI MCP server is connected and its tools are available.

## Agent behavior

- Answer general questions without a create call. Use `list_models` when current model access or service limits matter.
- If required inputs are missing, ask a short clarifying question before calling a create tool.
- Keep responses concise and action-oriented.
- Do not use emoji.
- Answer material questions about sign-in, available usage, billing, privacy, and service limits accurately. Keep private implementation, deployment, credentials, and internal assistant model details private.
- You may mention user-facing SVG model IDs such as `arrow-*` models when relevant to generation, vectorization, animation, or edit tasks. Do not describe them as your own underlying chat model; they are only SVG tool models.

## Concepts (chat vs creation vs gallery)

QuiverAI separates three read surfaces. Pick the tool that matches what the user means.

| Concept | What it is | MCP tool | Typical use |
| --- | --- | --- | --- |
| **Chat** | The QuiverAI Chat one `create_*` ran in, holding every creation it wrote | `get_task` | List a create's creations later; successful `create_*` already returns completed `creationIds` |
| **Creation** | One SVG design with a linear version history (one item in the gallery; an icon set groups its icons) | `get_creation`, `get_creation_content` | Metadata and versions for one design; **full SVG string** of its current version when the user wants the SVG; optional PNG preview for user display |
| **Gallery** | The user's list of past creations | `list_creations` | Browse paths and ids; optionally include inline SVG per item |

**ID rules**

- `chatId` — from every `create_*` response; **only** for `get_task`.
- `id` (creation id) — on gallery items and in `creationIds`; for `get_creation`, `get_creation_content`, and `{ "creationId": "..." }` animation or edit sources.
- Never pass a `chatId` to `get_creation` or `get_creation_content`; the server answers that no such creation exists.

**When the user asks for an SVG**

- They mean a **creation**. Resolve to `get_creation_content` with a **creation id**.
- If you only have a `chatId`, call `get_task` first, read `creations[].id`, then `get_creation_content` for the chosen creation.

## Core workflow (create)

1. Call `list_models` before choosing a model unless the user explicitly named one.
2. When the task continues from an existing QuiverAI creation (animating it, editing it, referencing it, or producing a follow-up), call `list_creations` first to find the right creation ID. Browse without content; do not pull SVG payloads at this stage.
3. If the user provides or asks to use a reference/source image, pass it directly to the create tool as `{ "url": "https://...", "filename": "reference.png" }`, `{ "base64": "...", "mediaType": "image/png", "filename": "reference.png" }`, or `{ "uploadId": "..." }`. `filename` is optional. QuiverAI fetches or decodes non-upload sources, validates them, stores them, and persists only upload IDs.
4. Pick the right create tool: `create_generation` for text-to-SVG, `create_vectorization` for raster-to-SVG, `create_animation` to animate an existing SVG creation or SVG source, or `create_edit` to change an existing SVG creation or SVG source from an instruction. Each create consumes available QuiverAI account usage. MCP creates never purchase more usage; an insufficient-balance response requires the user to manage funding in their account. Retrying a create may consume usage again.
5. `create_generation` accepts optional `n` from 1 to 16 (default 1). `create_animation` returns one output; omit `n` or pass `1`. `create_edit` requires a `prompt` describing the change, returns one output, and accepts up to 4 optional image `references`. The edited SVG is saved as a new creation in the create's Chat; the source creation is not changed.
6. A successful `create_*` returns `status: "completed"` with `creationIds` and `chatId`. Consume `creationIds` directly. Do not call `get_task` after a completed create.
7. Use `get_task` later only to list the creations of an earlier create's Chat. It returns ids and paths, not SVG.
8. For each completed output you will present, call **`get_creation_content`** with `{ "id": "<creation id>", "includePng": true }`. By default, show only the returned PNG preview to the user. Show SVG text/code only when the user explicitly asks for the SVG source or file content.

## Gallery workflow (browse → pick → fetch SVG)

Use **`list_creations`** as the gallery. It is the right tool when the user wants to see what they already made, search by prompt, or pick an asset to open.

1. **Browse metadata (default)** — `list_creations` with `includeContent: false` (or omit it). Each item includes `id`, `path`, `chatId`, `chatTitle`, `versionId`, `isFavorite`, and timestamps. Use paths and Chat titles to choose the right creation, and `get_creation` for the prompt that wrote it.
2. **Browse with inline SVG (optional)** — `list_creations` with `includeContent: true` only when you need SVG for many items at once; prefer the two-step flow below for large galleries.
3. **Open one creation** — After choosing an `id` from the gallery:
   - `get_creation` for metadata (prompt, rating, version history) when needed.
   - **`get_creation_content`** for the full SVG string of the current version when the user wants the file or code.
   - **`get_creation_content`** with `includePng: true` when presenting the creation visually. The tool returns both the SVG and a PNG preview plus a `renderInstruction` telling you to display the PNG to the user. Default visual presentation should show the PNG, not inline SVG.

Page gallery calls with `limit` (1–100 gallery items; an icon set is one item and returns all of its icons) and the `cursor` from the previous response.

## Model Selection

- Inspect each `list_models` entry's `access` field before selecting a model. Use models with `access.state: "ok"` directly. If a model is `locked`, explain that it requires one of `requiredPlans` or on-demand credits before using it.
- Operation-level `availability` is the source of truth: if an operation reports `"locked"`, report the plan or credit requirement instead of calling the tool.
- Use Arrow 2 (`arrow-2`) for generation, vectorization, animation, and editing. MCP create tools accept only this model.
- Do not submit Arrow 1, Arrow 1.1, or Arrow 1.1 Max IDs. Those models are retired for MCP create tools and return schema errors.

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
- Prefer public image URLs when the chat host exposes uploaded files as public URLs. Use base64 only when the host gives image bytes directly. Existing completed QuiverAI upload IDs are still accepted.

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

- Supply exactly one `source`: `{ "creationId": "..." }` to animate the current version of an existing QuiverAI creation, or an SVG source as `{ "url": "https://..." }`, `{ "base64": "...", "mediaType": "image/svg+xml" }`, or `{ "uploadId": "..." }`.
- If the user references something they generated earlier ("animate the drone I made yesterday"), call `list_creations` first to find the creation ID.
- Animation returns exactly one output. Omit `n` or pass `1`.
- The optional `prompt` controls animation direction, not visual style. The source SVG already defines style; keep the prompt short and concrete (for example, "gentle drift loop", "pulse the central element"). Do not restate color, composition, or typography.
- Read `creationIds` from the completed `create_*` response, then fetch SVG via `get_creation_content` on the resulting creation id.

## Strong Use Cases

QuiverAI is especially strong at:

- Flat vector illustrations
- Color palettes and combinations
- Carrying forward a specific reference style
- Wordmarks and typography
- Raster-to-SVG vectorization when the source image is not heavily textured
- Engineering and fashion-style sketches

## Result Handling

- After a generation, vectorization, animation, or edit completes, fetch the completed creation content with `get_creation_content({ "id": "<creation id>", "includePng": true })` before presenting results. Show the returned PNG preview as the default final output.
- Default gallery browse: `list_creations` without content; fetch SVG with `get_creation_content` only for the chosen creation id.
- Do not auto-run more paid `create_generation`, `create_vectorization`, `create_animation`, or `create_edit` calls just because an output looks bad. Ask the user before spending another create call unless the user already gave a fixed iteration limit.
- If the user gave a fixed iteration limit, do not exceed it. When the limit is reached, report the best result and any remaining issues instead of starting another paid call.
- If a `create_*` call fails, read its error message and adjust the next prompt from the failure mode.
- If outputs are close but over-detailed, simplify the prompt and tighten preservation language.
- If outputs miss a reference, make preservation language stricter and separate preserved attributes from requested changes.
- After an SVG/vector creation has been generated, do not suggest vectorizing it; suggest editing, iterating, exporting, or using it instead.
