# Local Studio Audio API Skill

A Codex-compatible skill for operating Local Studio's ElevenLabs and MiniMax audio-workbench features through the website's HTTP API.

## Contents

- `SKILL.md` — invocation guidance and API workflow selection.
- `references/api-reference.md` — human-oriented authentication, endpoint, payload, job, SSE, and download guide.
- `references/openapi.json` — OpenAPI 3.1 machine-readable specification.

## Install

Clone this repository and copy the `local-studio-audio-api` directory into your Codex skills directory (normally `~/.codex/skills/`). The skill name is `local-studio-audio-api`; use `$local-studio-audio-api` to invoke it explicitly.

## Connection requirements

Set the base URL to the Local Studio origin. Local development defaults to `http://localhost:4321`; use HTTPS for remote services. Authenticate with the same secret used for the website login, sent as a Bearer token or `X-API-Key` header. The Local Studio server retains the ElevenLabs and MiniMax provider keys; this repository contains no credentials.

See `references/api-reference.md` before calling an endpoint, especially for multipart uploads, asynchronous jobs, SSE, and binary downloads.
