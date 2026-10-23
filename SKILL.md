---
name: local-studio-audio-api
description: Call the Local Studio Audio API to inspect service status, manage pronunciation rules and source documents, synthesize audio with ElevenLabs or MiniMax, follow jobs, inspect history, and download audio or ZIPs. Use when a user wants to operate Local Studio through its HTTP API.
---

# Local Studio Audio API

Use this skill to perform Local Studio website operations through its HTTP API rather than browser clicks.

## Connection and authentication

1. Determine the service origin from the user's configured deployment or request. For a local instance, the default is `http://localhost:4321`; for a remote instance, use its HTTPS origin.
2. Obtain the API secret from the caller's configured environment or ask for it only if an authenticated action requires it. It is the same secret used for website login. Do not print, persist, or put it in a URL, shell history, logs, or generated artifacts.
3. Send `Authorization: Bearer $LOCAL_STUDIO_API_KEY` (or `X-API-Key`). The server-side ElevenLabs and MiniMax provider keys are not client credentials and must never be requested or forwarded.
4. Read [references/api-reference.md](references/api-reference.md) for endpoint-specific payloads and workflow details. Use [references/openapi.json](references/openapi.json) as the machine-readable contract. The live `GET /api/openapi.json` is authoritative if a deployment has changed.

## Operating workflows

- Check `/api/health` first when the user asks about availability or provider configuration. It reports booleans, not provider secrets.
- For pronunciation rules, read the current dictionary before changing it. A `PUT /api/dictionary` can update either/both language arrays; omitted language arrays are preserved. Verify the saved response.
- For source files, use multipart upload with the repeated `documents` field, then use returned extracted text/document IDs as inputs to batch synthesis.
- For one-off ElevenLabs output, call `/api/synthesize`; for multi-document work, call `/api/synthesize-batch`, retain `jobId`, and poll `/api/jobs/{id}` or consume `/api/jobs/{id}/events`.
- For MiniMax voice cloning, upload a sample to `/api/minimax/clone-upload`, pass its `file.file_id` to `/api/minimax/voice-clone`, and use the returned voice ID for synthesis. MiniMax single synthesis is asynchronous: poll `/api/minimax/tasks/{taskId}`; download only after success. Batch synthesis returns a Local Studio `jobId` and uses `/api/minimax/jobs/{jobId}` or its SSE route.
- Treat API download URLs as relative to the service origin. Audio and ZIP endpoints return binary data; save/forward the bytes without JSON parsing. Do not claim completion until the final job state or downloaded file is verified.

## HTTP client details

- Browser `EventSource` cannot attach the API-key header. For SSE, use `fetch()` with the auth header and read `response.body`; each update is a `data: <JSON>\n\n` record.
- For `multipart/form-data`, let the HTTP client set the multipart boundary. Do not manually set `Content-Type` for a `FormData` body.
- Check HTTP status before parsing a response. Error bodies are generally JSON; binary success bodies are not.
- Keep long-running job IDs and returned audio IDs exactly as given. Do not infer that an HTTP `202` means synthesis is complete.

## Reference files

- [API reference](references/api-reference.md): routes, request options, media types, async workflows, and errors.
- [OpenAPI 3.1 document](references/openapi.json): machine-readable endpoint and schema inventory.
