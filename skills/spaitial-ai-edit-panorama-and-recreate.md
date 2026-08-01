---
name: Edit a panorama and create a world from it
description: Iteratively edit a world's panorama with prompts and reference images, then generate a new SpAItial world from the final panorama.
api: openapi/spaitial-ai-developer-api-openapi.json
operations: [V1Panoramas_editPanorama, V1Panoramas_downloadPanorama, V1Panoramas_getPanorama, V1Worlds_createJob, V1Worlds_getJobStatus]
source: https://docs.spaitial.ai/api/llm-skills
generated: '2026-07-21'
method: generated
---

# Edit a panorama and create a world from it

Run the app-style "edit the panorama, inspect, iterate, then generate" loop.

## Auth
- Base URL `https://api.spaitial.ai`, `Authorization: Bearer spt_live_<key>`.
- Panorama edits use scope `worlds:create`; reading/downloading uses `worlds:read`.

## Steps
1. Edit the panorama with `V1Panoramas_editPanorama` (`POST /v1/panoramas/edit`). `source` is one of `{type: request_id}`, `{type: world_id}`, or `{type: panorama_id}` plus a `prompt` and up to 3 optional reference `images`. Send an `Idempotency-Key`. Edits are synchronous and return a `pano_...` with `status: READY`.
2. Inspect the result with `V1Panoramas_downloadPanorama` (`GET /v1/panoramas/{panoramaId}/download`) — 302 to a signed URL; follow redirects. Use `V1Panoramas_getPanorama` for metadata.
3. Iterate by passing the returned `panorama_id` back as `source` to `V1Panoramas_editPanorama` ("add a sound system by the window").
4. When satisfied, create the world with `V1Worlds_createJob` (`POST /v1/worlds`) using `input: { type: panorama_id, panorama_id: pano_... }`.
5. Poll `V1Worlds_getJobStatus` to `COMPLETED` (generation restarts at the pano2video stage and keeps lineage to the source world).

## Rules
- Edited panoramas live 24 hours (`410 PANORAMA_EXPIRED` after TTL); `502 EDIT_FAILED` is retryable with a fresh/idempotent request.
- There is intentionally no aspect-ratio field; the panorama format is preserved so the result stays valid for world generation.
- `404 PANORAMA_NOT_FOUND` means the `pano_...` does not exist or is not owned by the caller.
