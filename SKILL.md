---
name: higgsfield
description: Generate marketing images, UGC videos, and DTC ads using the Higgsfield MCP. Use this skill whenever the user asks to "create an image", "make a video", "make a UGC", "make an ad", "advertise this product", "take inspiration from this URL", build a poster, design an avatar or character, train a Soul / digital twin, run a virality check, or do any visual content generation for marketing. Also use it when the user references a Higgsfield model by name (soul_2, nano_banana_2, seedance, kling, marketing_studio_image, marketing_studio_video) or talks about Marketing Studio, DTC Ads, hooks, settings, brand kits, or product/webproduct ads. Trigger even when the request is casual ("make me a video for X", "design something for our launch"). When in doubt, use it — model selection and the upload/Marketing-Studio flow are easy to get wrong without this skill.
---

# Higgsfield (101Creed marketing tool)

You are running visual content generation for the 101Creed marketing team via Higgsfield's MCP tools. Without this skill the easy mistakes are: wrong model, skipped `models_explore`, training a Soul Character for a one-off request, fumbling the upload flow, and confusing `product` vs `webproduct` in Marketing Studio. The job of this skill is to keep you on the rails.

## The three things Higgsfield does

Map every request to one of these three modes before picking a tool:

1. **Free-form generation** — make any image or video from a prompt, optionally with reference media. Use `generate_image` / `generate_video` directly.
2. **DTC Ads via Marketing Studio** — turn a product/website URL (or product image) into a polished ad. Always goes through `show_marketing_studio` first, then `generate_video` with `model: marketing_studio_video` (or `generate_image` with `model: marketing_studio_image`).
3. **Soul Characters** — train a reusable identity from 5–20 photos, then generate with that identity via `model: soul_2` + `soul_id`. Only train when the user explicitly asks for a reusable character.

If a request is ambiguous between two modes, ask the user one short clarifying question rather than guessing — the modes use different tools and have different cost profiles.

## Model selection cheatsheet

These are the defaults a marketing operator should use. Only deviate if the user names a specific model or has a clear constraint (4K text, multi-shot, motion transfer, etc.).

**Images:**
- Commercial product / ad-style image → `marketing_studio_image`
- Text-only character/avatar (no reference photos) → `soul_cast`
- Reusable character already trained → `soul_2` + `soul_id`
- One-off character / portrait / fashion / UGC / editorial → `soul_2` (with reference) or `nano_banana_2`
- Top quality, 4K, or contains text/diagrams → `nano_banana_2`

**Videos:**
- Default UGC video → `seedance_2_0` (strong identity, accepts audio via `medias`)
- Multi-shot, motion transfer, or in-video audio you want explicitly → `kling3_0`
- Anything driven by a URL or product image → `marketing_studio_video` (after `show_marketing_studio`)

**Always call `models_explore` first when:**
- The user names a model you don't recognize
- You need non-default `aspect_ratio`, `duration`, or other constraints
- You're about to pass medias and aren't sure of the role name (e.g. `start_image` vs `image`)

`models_explore({ action: 'recommend', query: '<goal + input context>', input: 'image' or 'text' })` returns the right `model_id`, the allowed `aspect_ratios`, declared `parameters`, and `medias[].roles`. Trust those declared roles — the server may auto-coerce but it's noisy.

## Common marketing workflows

### "Create image of [thing]"

1. Decide the mode from the description: commercial/product → `marketing_studio_image`; character/portrait → `soul_2` or `nano_banana_2`; text-heavy or 4K → `nano_banana_2`.
2. Call `generate_image({ params: { model, prompt, count: 1, aspect_ratio? } })`.
3. If the user attached a reference image, pass it via `medias: [{ role: 'image', value: '<media_id|job_id|https-url>' }]`.

Don't include `aspect_ratio` unless the user specified one — let the model default. Don't pass `count > 1` unless they asked for variants; each result costs credits.

### "Create a UGC video of [thing]"

1. Default model: `seedance_2_0`. It accepts in-video audio by passing an audio file in `medias` — **do not** pass `generate_audio: true`, that parameter doesn't exist on seedance.
2. If the user attached a reference image, that's an image-to-video request. Pass `medias: [{ role: 'start_image', value: '<...>' }]`. Confirm the role name via `models_explore` if you haven't recently.
3. Default `duration` — leave it off and let the model decide unless the user said how long. Seedance and Kling have different allowed durations; the server clamps to the nearest legal value but tells you in `adjustments`.

### "Take inspiration from this URL: [url]"

This is always the Marketing Studio flow. Do not try to scrape the URL yourself.

1. **Decide product vs webproduct:**
   - `product` — URL points to a single sellable item (Amazon SKU page, BMW model page, brand product page, a single SKU on a brand site, a direct product image). The ad features that item.
   - `webproduct` — URL points to a site/app as a whole (App Store, Google Play, SaaS landing page, company homepage with no single SKU). The ad promotes the site.
   - **In doubt**: omit `type` and let the server infer. App Store / Google Play → `webproduct`. Otherwise default to `product`.

2. Call `show_marketing_studio({ action: 'fetch', url: '<url>' /*, type: 'product' or 'webproduct' if known */ })`. This kicks off an async fetch and immediately renders the widget. The user sees a progress pill; **do not poll** — the widget handles status.

3. The response includes `next_step`. When ready, call `generate_video({ params: { model: 'marketing_studio_video', ...next_step } })`.

4. If the user wants a specific **hook** or **setting**, those only apply to presets `UGC`, `Tutorial`, `Unboxing`, `Product Review`, and `UGC Virtual Try On`. List options first with `show_marketing_studio({ action: 'list', type: 'hook' })` (or `'setting'`), present them to the user, then pass the chosen `hook_id` / `setting_id` to `generate_video`.

5. For an **uploaded product image** (not a URL) → product video: call `show_marketing_studio({ action: 'create', type: 'product', medias: [{ value: '<media_id|job_id|url>' }] })`. The server auto-titles and returns `next_step`; pass it to `generate_video` with `model: marketing_studio_video`.

### "Make a video from this image"

`generate_video({ params: { model: 'seedance_2_0', prompt: '<motion description>', medias: [{ role: 'start_image', value: '<...>' }] } })`. If the user wants the camera to land on a specific frame, also pass `end_image` if the model declares that role.

### "Predict the virality of this video"

`virality_predictor({ action: 'create', params: { model: 'virality_predictor', medias: [{ role: 'video', id: '<confirmed-media-uuid-or-completed-job-uuid>' }] } })`. The dashboard opens in the widget. To re-open later: `action: 'preview'` with the returned `job_id`.

### "Train a Soul Character"

Only do this when the user is explicit ("train a Soul", "make me a reusable identity", "digital twin", or supplies 5–20 photos of one subject). Otherwise it's a one-off generation — use `generate_image` with `soul_2` (with reference) or `nano_banana_2` and *do not* train.

For ambiguous requests like "make a character for my game", **offer the choice**:

> "I can either generate a one-off character image now (~30s) or train a reusable Soul Character you can summon across future generations (~10 min, needs 5–20 reference photos). Which would you like?"

Training call: `show_characters({ action: 'train', name: '<name>', medias: [{ value: '<media_id|job_id|url>' }, ...] })`. Returns immediately; the widget polls. After training is `ready`, generate with `model: 'soul_2'` and the returned `soul_id` (top-level field, not inside `medias`).

## Media handling — the upload flow

The single most common foot-gun: trying to reference a local file path. **You can't.** Every reference image/video must be one of:

- A `media_id` UUID returned by `media_confirm` (after upload)
- A `job_id` UUID from a previous completed Higgsfield generation
- A public `https://` URL (must be reachable; for avatar/Soul medias only `cloudfront.net` and `cdn.higgsfield.ai` hosts work)

**Local file → reference flow:**

1. `media_upload({ filename: '<name>', content_type: 'image/png' })` — or pass `files: [{filename, content_type}, ...]` for batches. Returns presigned `upload_url`(s) and a `media_id` for each.
2. PUT the bytes to each `upload_url` (the tool returns curl commands you can run).
3. `media_confirm({ type: 'image' | 'video' | 'audio', media_id: '<...>' })` — or `media_ids: [...]` for batches.
4. Now use that `media_id` as `medias[].value` in a generation call.

If the user pasted a public URL, skip steps 1–3 and pass the URL directly. If they reference a previous generation, pass its `job_id`.

## Workspace and credits

- If the user mentions a team or client workspace, call `list_workspaces` and confirm `is_selected` matches before generating. Switch with `select_workspace({ workspace_id })`. Don't ask if they haven't mentioned it.
- Before a batch (e.g., generating 4 variants × 3 prompts), call `balance` and surface the credit count so the user can opt out before spending.
- "How much do I have left?" → `balance` plus the last 5–10 `transactions`.

## After generation

Higgsfield jobs return immediately with a `job_id` and the widget renders progress. **Do not poll with `show_generations`** as a completion check — it's for browsing history, not as a tight loop. Trust the widget.

- Re-display a specific past result → `job_display({ ids: ['<job_id>'] })`
- Browse history → `show_generations({ type: 'image' or 'video' })`
- Reuse a past generation as reference in a new call → pass its `job_id` as `medias[].value`

## Gotchas — read this list before generating

1. **Don't guess models.** If it's not on the cheatsheet, call `models_explore`.
2. **Don't pass `generate_audio: true` to seedance.** Audio comes via a media in `medias`.
3. **Don't poll.** Widget handles status; move on.
4. **Don't train a Soul for a one-off request.** Offer the choice when ambiguous.
5. **Don't confuse product/webproduct.** `product` = "advertise this item". `webproduct` = "advertise this site/app".
6. **Don't pass local file paths anywhere.** Upload → confirm → use the UUID.
7. **Don't `generate_video({ model: 'marketing_studio_video' })` without first calling `show_marketing_studio`.** The fetch/create step is mandatory — it sets up the product/webproduct entity the model needs.
8. **Don't pass `count > 1` for expensive generations** unless the user explicitly asked for variants. Each result is its own charge.
9. **Don't include `aspect_ratio`, `duration`, etc. unless the user specified them.** Let model defaults win; the server returns `adjustments` if it clamped anything.
10. **Don't paraphrase the user's prompt creatively** for `marketing_studio_*` — Marketing Studio expects clean, intentional ad direction, not flowery descriptions. Pass through the user's brief.

## Talking to the user

When you're about to spend credits (any `generate_*` or `virality_predictor.create` call), say briefly what you're doing in one line — e.g. "Generating 1 UGC video with seedance_2_0 from the screenshot." If the user attached a reference, confirm you're using it. After the call, point at the widget and stop — don't narrate progress.

When the user is exploring ("what can Higgsfield do?", "what models are there?"), call `models_explore({ action: 'list' })` and summarise the relevant categories instead of listing every model.

When the user asks about an existing avatar, product, webproduct, hook, setting, or brand kit, route to the right `show_marketing_studio({ action: 'list', type: '<...>' })` call.
