# Local Studio API

The API exposes the audio workbench operations used by the website. JSON requests and responses use UTF-8. Unless listed as public below, endpoints require authentication.

## Authentication

Use the same secret entered at the website login screen. The server stores only its configured password hash and validates the supplied value against it. Existing browser login cookies continue to work.

```sh
export LOCAL_STUDIO_API_KEY='your-login-secret'
curl -H "Authorization: Bearer $LOCAL_STUDIO_API_KEY" http://localhost:4321/api/dictionary
# Also accepted: -H "X-API-Key: $LOCAL_STUDIO_API_KEY"
```

Do not put the credential in a URL or query parameter. For Server-Sent Events, use `fetch()` with an authorization header and read its response stream; browser `EventSource` cannot set custom authorization headers. Serve remote deployments over HTTPS.

The API secret is exactly the website login secret; there is no separate client API-key provisioning flow. `GET /api/auth/me`, `POST /api/auth/login`, and `POST /api/auth/logout` are public routes. Login accepts `{ "password": "..." }` and returns an HttpOnly browser-session cookie. For scripts and non-browser clients, send the secret as a Bearer header or `X-API-Key` on each protected request. A valid API credential also makes `/api/auth/me` return `authenticated: true`.

Use a base URL such as `http://localhost:4321` only for local development; use the deployment's HTTPS origin for a remote server. The API base URL must not contain credentials.

Public endpoints: `GET /api/health`, `GET /api/openapi.json`, `GET /api/auth/me`, `POST /api/auth/login`, and `POST /api/auth/logout`. `GET /api/auth/me` also reports `authenticated: true` for a valid API credential. Login returns an HttpOnly session cookie for browser use; logout clears that cookie.

## Shared operations

### Health

`GET /api/health` returns `{ ok, elevenlabsConfigured, minimaxConfigured, ffmpeg }`.

### Pronunciation dictionary

- `GET /api/dictionary` returns the current dictionary object.
- `PUT /api/dictionary` saves it. Send either `{ "rules": [...] }` or `{ "rules_english": [...], "rules_chinese": [...] }`. Each rule has `string_to_replace`, `alias`, and optional `type` (`alias` or `phoneme`). At least one complete rule is required.

Language-specific partial updates preserve the omitted language's existing rules. The legacy `rules` array updates the English rules and retains the Chinese rules. The response contains the saved dictionary; use it to verify the change.

### Upload and extract documents

`POST /api/upload` uses `multipart/form-data`; add one or more files under the repeated field name `documents`. TXT, Markdown (`.md`), HTML (`.html`/`.htm`), PDF, DOCX, CSV, and JSON are supported. Limit: 200 files, 40 MiB per file. Response: `{ "documents": [{ "id", "filename", "characters", "text" }] }`. Preserve returned `text` when sending a document to synthesis; `documentId` is optional in batch input.

## ElevenLabs

### Voices

`GET /api/voices` returns the currently configured supported voice(s). It requires an ElevenLabs provider key on the server.

### Synthesize one text

`POST /api/synthesize`

```json
{
  "text": "Text to speak",
  "filename": "chapter.txt",
  "voice_id": "VZ0WbDi66FzW2TpV5M8Q",
  "model_id": "eleven_multilingual_v2",
  "voice_settings": {
    "stability": 0.5,
    "similarity_boost": 0.75,
    "style": 0,
    "use_speaker_boost": true,
    "speed": 1
  },
  "output_format": "mp3_44100_128",
  "text_normalization": "auto",
  "apply_dictionary": true,
  "normalize": true
}
```

`text` is required. `voice_id` must match the configured voice. Supported output formats: `mp3_44100_128`, `mp3_44100_192`, `mp3_22050_32`, `pcm_16000`. Text normalization: `auto`, `on`, `off`. Voice setting numbers are clamped to supported ranges; `eleven_v3` only receives stability, and `eleven_flash_v2_5` receives stability, similarity boost, and speed. `normalize` controls the server's FFmpeg loudness-normalization pass. Response includes the generated `id`, `filename`, `downloadUrl`, and effective settings.

The supported voice is the configured Yefeng voice (`VZ0WbDi66FzW2TpV5M8Q`); `voice_id` may be omitted to use it. `/api/voices` lists that supported voice and requires the server-side ElevenLabs provider key to be configured.

### Batch synthesis

`POST /api/synthesize-batch` returns `202 { "jobId", "total" }`.

```json
{
  "items": [{ "documentId": "doc-1", "filename": "chapter.txt", "text": "Text to speak" }],
  "voice_id": "VZ0WbDi66FzW2TpV5M8Q",
  "model_id": "eleven_multilingual_v2",
  "voice_settings": { "stability": 0.5, "similarity_boost": 0.75, "style": 0, "use_speaker_boost": true, "speed": 1 },
  "output_format": "mp3_44100_128",
  "text_normalization": "auto",
  "apply_dictionary": true,
  "normalize": true
}
```

Poll `GET /api/jobs/{jobId}` for progress, or consume `GET /api/jobs/{jobId}/events` as an SSE stream. Job statuses are `queued`, `running`, `completed`, `completed_with_errors`, or `failed`; each document has `queued`, `processing`, `done`, or `error` status.

### History and audio downloads

- `GET /api/history` lists saved jobs; `GET /api/history/{jobId}` returns a record including input text.
- `GET /api/download/{audioId}?extension=mp3&filename=chapter.mp3` downloads a generated audio file. `extension` can be `mp3`, `pcm`, `flac`, `wav`, `pcmu_raw`, `pcmu_wav`, or `opus`; `filename` is only a download name.
- `POST /api/download-batch` returns a ZIP. Body: `{ "files": [{ "id": "audio-uuid", "filename": "chapter.mp3" }] }`. `{ "ids": ["audio-uuid"] }` is also accepted.

## MiniMax

MiniMax operations require a configured MiniMax provider key on the server. The web/API credential remains the Local Studio login secret; the provider key is never sent to clients.

### Upload a voice sample and clone a voice

- `POST /api/minimax/clone-upload` is multipart with audio field `file` (maximum 20 MiB). It returns the MiniMax file-upload result, including `file.file_id`.
- `POST /api/minimax/voice-clone` body:

```json
{
  "file_id": 1234,
  "voice_id": "my_custom_voice",
  "text": "Optional preview text",
  "model": "speech-2.8-hd",
  "need_noise_reduction": false,
  "need_volume_normalization": false
}
```

`file_id` and a valid `voice_id` are required. `text` and `model` optionally request a preview.

### Single synthesis

`POST /api/minimax/synthesize` returns the upstream asynchronous synthesis submission result (typically including a task ID). Example:

```json
{
  "text": "Text to speak",
  "voice_id": "my_custom_voice",
  "model": "speech-2.8-hd",
  "voice_setting": { "speed": 1, "vol": 1, "pitch": 0, "emotion": "calm" },
  "audio_setting": { "format": "mp3", "sample_rate": 32000, "bitrate": 128000, "channel": 2 },
  "language_boost": "auto",
  "text_normalization": false,
  "latex_read": false,
  "subtitle_enable": false
}
```

For compatibility, voice and audio settings can also be sent as top-level `speed`, `vol`, `pitch`, `emotion`, `format`, `sample_rate`, `bitrate`, and `channel` fields.

Poll `GET /api/minimax/tasks/{taskId}` for a single synthesis task. When the status is `Success`, the response includes an authenticated `audio_download_url` containing the successful task's `file_id`; fetch it to download the generated file without exposing the MiniMax provider key. Direct downloads require `GET /api/minimax/tasks/{taskId}/audio?file_id=...` and reject a file ID that does not match the task result.

The audio route accepts optional `format` for the download filename/content type. `file_id` must be a positive integer from the successful task response. The API verifies task success and exact task/file ID pairing; a pending task or mismatched file ID returns `409`. Send the same Local Studio login secret when fetching the relative `audio_download_url`.

### Batch synthesis, job status, and downloads

`POST /api/minimax/synthesize-batch` returns `202 { "jobId", "total" }`. It accepts `items` (`documentId`, `filename`, `text`), `voice_id`, `model`, `voice_setting`, `audio_setting`, `language_boost`, `text_normalization`, `latex_read`, `subtitle_enable`, and `apply_dictionary`. `voice_setting` supports speed, volume (`vol`), pitch, and emotion; audio settings support format, sample rate, bitrate, and channel count.

- `GET /api/minimax/jobs/{jobId}` returns current or persisted job progress.
- `GET /api/minimax/jobs/{jobId}/events` streams progress as SSE.
- `POST /api/minimax/download-batch` returns a ZIP. Body: `{ "files": [{ "id": "audio-uuid", "filename": "chapter.mp3" }] }`.

Each completed MiniMax batch document has an audio result with `id`, `filename`, and `downloadUrl`. Fetch that relative URL through the authenticated common `GET /api/download/{id}` route to retrieve one file; use `/api/minimax/download-batch` when a ZIP is preferred.

## Errors and client behavior

Unauthorized requests return `401` JSON. Invalid payloads return `400`; a MiniMax task that is unfinished or has a mismatched file ID returns `409`; missing provider configuration returns `503`; upstream provider failures return `502`; missing jobs/files return `404`. Unknown `/api` paths return JSON `404` instead of the website HTML. Batch synthesis is asynchronous: retain the returned job ID, then poll the status route or consume its event stream. API responses do not expose provider credentials.

## Example: authenticated SSE with fetch

```js
const response = await fetch(`${baseUrl}/api/jobs/${jobId}/events`, {
  headers: { Authorization: `Bearer ${apiKey}` }
});
if (!response.ok) throw new Error(`SSE request failed: ${response.status}`);
const decoder = new TextDecoder();
let buffer = "";
for await (const chunk of response.body) {
  buffer += decoder.decode(chunk, { stream: true });
  let boundary;
  while ((boundary = buffer.indexOf("\n\n")) >= 0) {
    const event = buffer.slice(0, boundary).replace(/\r/g, "");
    buffer = buffer.slice(boundary + 2);
    const data = event.split("\n").filter((line) => line.startsWith("data:")).map((line) => line.slice(5).trimStart()).join("\n");
    if (data) console.log(JSON.parse(data)); // Job snapshot; stop when its status is terminal.
  }
}
```

## Shell examples

The examples assume `BASE_URL` and `LOCAL_STUDIO_API_KEY` are configured in the calling environment. Avoid substituting the secret directly into a command string that may be saved to shell history.

```sh
curl --fail-with-body "$BASE_URL/api/health"
curl --fail-with-body -H "Authorization: Bearer $LOCAL_STUDIO_API_KEY" "$BASE_URL/api/dictionary"

# One ElevenLabs synthesis; the JSON response contains a relative downloadUrl.
curl --fail-with-body -H "Authorization: Bearer $LOCAL_STUDIO_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"text":"A short sample.","voice_settings":{"speed":1},"apply_dictionary":true}' \
  "$BASE_URL/api/synthesize"

# Save a binary audio response (not JSON).
curl --fail-with-body -H "Authorization: Bearer $LOCAL_STUDIO_API_KEY" \
  "$BASE_URL/api/download/AUDIO_UUID?extension=mp3" -o speech.mp3

# Upload a source document; repeat -F documents=@... for additional files.
curl --fail-with-body -H "Authorization: Bearer $LOCAL_STUDIO_API_KEY" \
  -F 'documents=@chapter.docx' "$BASE_URL/api/upload"
```

For MiniMax, use the same Local Studio API credential and the `/api/minimax/...` routes. The provider secret remains configured on the service. `POST /api/minimax/synthesize` returns an upstream task ID, not necessarily audio; poll the task endpoint until `Success`, then fetch its `audio_download_url`. For either provider's batch route, retain `jobId`, inspect status/progress, then download completed audio IDs or use the provider-specific ZIP route.
