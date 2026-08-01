---
name: Generate and download a 3D world
description: Submit a SpAItial world-generation job from text or an image, poll to completion, and download the Gaussian Splat.
api: openapi/spaitial-ai-developer-api-openapi.json
operations: [V1Models_listModels, V1Files_uploadFile, V1Worlds_createJob, V1Worlds_getJobStatus, V1Worlds_getJobResult, V1Worlds_getSplatDownload, V1Worlds_createExport, V1Worlds_getExport]
source: https://docs.spaitial.ai/api/llm-skills
generated: '2026-07-21'
method: generated
---

# Generate and download a 3D world

Use the SpAItial Developer API to turn a text prompt, an image, or a 360 panorama
into an explorable 3D Gaussian Splat world.

## Auth
- Base URL `https://api.spaitial.ai`. Send `Authorization: Bearer spt_live_<key>`.
- World creation needs scope `worlds:create`; reads/downloads need `worlds:read`.
- World generation spends plan credits (Echo 2 - Standard = 160, Echo 2 HQ = 800).

## Steps
1. (Optional) List models with `V1Models_listModels` (`GET /v1/models`) to confirm auth and pick a model.
2. For an image flow, upload the file first with `V1Files_uploadFile` (`POST /v1/files`) and keep the returned `file_id`. Skip for text prompts.
3. Submit the job with `V1Worlds_createJob` (`POST /v1/worlds`). Provide a discriminated `input` (`type: text|url|base64|file_id|panorama_id`). Send an `Idempotency-Key` header so a retry does not double-charge. You get `202` with `request_id` and `status: PENDING`.
4. Poll `V1Worlds_getJobStatus` (`GET /v1/worlds/requests/{requestId}/status`) every 5-10s (with backoff after the first minute), or set `webhook.url` on the create call to receive a `world.completed` callback instead. Terminal states: `COMPLETED`, `FAILED`, `CANCELLED`.
5. On `COMPLETED`, fetch `V1Worlds_getJobResult` (`GET /v1/worlds/requests/{requestId}`) for the `world` object (`splat_url`, `panorama_url`, `viewer_url`).
6. Download the splat with `V1Worlds_getSplatDownload` (`GET /v1/worlds/requests/{requestId}/splat`) — it 302-redirects to a ~5-minute signed URL, so follow redirects and re-hit when it expires.
7. (Optional) Start a mesh export with `V1Worlds_createExport` (`POST /v1/worlds/requests/{requestId}/exports/{type}` with `type: mesh` or `mesh-simplified`), then poll `V1Worlds_getExport` for `status: READY` and its `download_url`.

## Rules
- Branch on `error.code`, not the message. Retry `5xx` and `409 RESOURCE_NOT_READY` with backoff; fix `4xx` first.
- `402 INSUFFICIENT_CREDITS` means buy/upgrade plan credit (free daily credit cannot pay for API usage).
- Respect rate limits: general 60/min, world-create 10/min, downloads 120/min (`X-RateLimit-*` + `Retry-After`).
