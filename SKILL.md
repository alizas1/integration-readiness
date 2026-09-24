---
name: review-server
description: >
  Start a Review or Validation on the Review Server API, keep the job id, and
  fetch the report and fix file. Use when calling this host with an API key
  from an agent.
---

# Review Server API

Written **2026-09-16**. Updated **2026-09-17**.

How an API key customer’s agent calls the host. Contract: [`openapi.yaml`](./openapi.yaml). Sequences and handoffs: [`arazzo.yaml`](./arazzo.yaml).

This file is how to call the host. The how-to that writes the report stays on the host. The caller gets `report.md` and `fix.md` (and `path.md` for Validation flows).

Host: `https://agent-server-production-a722.up.railway.app`

## Authenticate

`Authorization: Bearer` plus the API key, on every start and every GET. `aiModel` on every start body (`primaryName` required). AI model key in `aiModel.key` or header `x-model-key`. Optional: `aiModel.cheapName`, `aiModel.company`. Omit `company` → the host guesses from the key. Names it accepts: `anthropic` / `claude`, `openai` / `chatgpt` / `gpt`, `deepseek`, `xai` / `grok`, `mistral`, `groq`, `google` / `gemini`. A ChatGPT login is not a key. A Cursor key cannot run this loop. An unknown company name → `{ "error": "could not tell which company this key is for" }` — send `aiModel.company`.

Unknown or missing API key → `{ "error": "unauthorized" }` (401). Missing AI model key → `{ "error": "an ai model key is required" }` (401). Missing `primaryName` → `{ "error": "primaryName is required" }` (400).

Only jobs started with that API key can be read. For now, reach out to alizasolomondx@gmail.com to get an API key.

**Note:** There is no list call — make sure to keep the `jobId` from the 202. Without it the job cannot be looked up. Every start POST is a new job. If the 202 already arrived, GET the files (and `GET /jobs/<Job ID>/info` if desired).

## Sequences

Typical order: Review → Validation → Combined. That workflow in [`arazzo.yaml`](./arazzo.yaml) is `review-then-validation-then-combined`. Pieces: `review`, `validation`, `combined`, `run-again`, `check-and-fetch`. Combined is intended for a Review and a Validation of the same use cases. The host does not check. Unrelated jobs will combine to create a misleading report.

## What comes back

The server POSTs to `callbackUrl` (when provided) when there is a report. One try. Body: `reportMarkdown` and `fixMarkdown` (Validation also `pathMarkdown`). GET info / report / fix still run if called.

The report (`GET /jobs/<Job ID>/report`): 200 the report. 404 `{ "error": "no report yet" }` while `"In progress"`. 404 `{ "error": "not found" }` unknown `jobId`.

The fix file (`GET /jobs/<Job ID>/fix`): 200 the fix file. 404 `{ "error": "not found" }` — unknown `jobId`, or no fix file yet. Wait while `"In progress"`. Stop when `fixFileStatus` is `"fix file was not saved"`. GET info first when those 404s need telling apart.

The path file on Validation (`GET /jobs/<Job ID>/path`): 200 the path file. 404 `{ "error": "no path file yet" }` while it is not written. 404 `{ "error": "not found" }` unknown `jobId`.

Job info (`GET /jobs/<Job ID>/info`): 200 JSON. Files are not in this call. `"In progress"`: wait. `"Completed"`: `finishedAt`, `score` and `summary` when parsed. `"Failed"`: `failureDetails` (a string) and `finishedAt`. 404 `{ "error": "not found" }`: this `jobId` is unknown. `fixFileStatus` is `"fix file was not saved"` when the report exists but `fix.md` was never saved.

`scoreStatus` is present only when the score in the report did not match a count based on the findings. It returns a string that names both numbers, e.g. `The report says score: 70, but counting the findings adds up to 76.` `score` and `report.md` then uses that second number (76 in that string).

## What you send

For a Review with `resources`:

```json
{
  "resources": ["<docs or spec URL>"],
  "aiModel": {
    "primaryName": "<model name>",
    "key": "<AI model key>"
  }
}
```

HTML, JSON, YAML, Markdown, and plain text are read. PDF, image, audio, video, and `octet-stream` are not.

A start needs either `resources` or `priorReport`. If both are missing, the host returns `{ "error": "at least one resource or prior report is required" }` (400).
