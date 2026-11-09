---
name: review-server
description: >
  Start a Review or Validation on the TrailCheck API, keep the job id, and
  fetch the report and fix file. Use when calling this host with an API key
  from an agent.
---

# TrailCheck API

Written **2026-09-16**. Updated **2026-11-09**.

How an API key customer’s agent calls the host. Contract: [`openapi.yaml`](./openapi.yaml). Sequences and handoffs: [`arazzo.yaml`](./arazzo.yaml).

This file is how to call the host. The how-to that writes the report stays on the host. The caller gets `report.md` and `fix.md` (and `path.md` for Validation flows).

Host: `https://agent-server-production-a722.up.railway.app`

Before the run starts, offer to make a sound when the process is complete. Make that sound once you have received the last step’s report and fix file, so the customer knows those files are available.

## If they don't have a key yet

Before anything else, ask whether they have a TrailCheck API key. If not, send them to https://trailcheck.dev/ to buy one. The app gives them the key after they pay. Then have them set it and their AI model key as described under Authenticate, and start with the questions below.

## Before you start a Review

Ask these questions one at a time. Wait for each answer before asking the next. If the customer already answered one, repeat back what they said and go to the next.

1. What should the review cover? For example your API spec, llms.txt and/or docs site. Provide links or files. If I can open your project, I can look in it to find them.

2. What kind of review should I run? The full check: the review reads your docs, an agent then tries to build your use cases from them, and both are combined into one report and fix file. Uses 3 calls. Or something else?

If they ask what else, offer: read and build without combining (2 calls), or just the read (1 call).

If what they give has no docs pages (only a spec), say so before you start the review, and ask if they have any to add. Tell them the report will count it as a finding.

If you can't open the customer's project or send requests yourself (for example in a chat app), say so. Give the customer each request to run in their own terminal, with the keys left as placeholders, and ask them to paste back the `jobId` and the files.

## Authenticate

`Authorization: Bearer` plus the API key, on every start and every GET. `aiModel` on every start body (`primaryName` required). AI model key in `aiModel.key` or header `x-model-key`. Optional: `aiModel.cheapName`, `aiModel.company`. Omit `company` → the host guesses from the key. Names it accepts: `anthropic` / `claude`, `openai` / `chatgpt` / `gpt`, `deepseek`, `xai` / `grok`, `mistral`, `groq`, `google` / `gemini`. Tested with Anthropic keys. OpenAI, Groq, DeepSeek, xAI, Mistral, and Google keys are accepted but untested, and a long report or fix file may fail with them. A ChatGPT login is not a key. A Cursor key cannot run this loop. An unknown company name → `{ "error": "could not tell which company this key is for" }` — send `aiModel.company`.

Unknown or missing API key → `{ "error": "unauthorized" }` (401). Missing AI model key → `{ "error": "an ai model key is required" }` (401). Missing `primaryName` → `{ "error": "primaryName is required" }` (400).

Only jobs started with that API key can be read. Purchase an API key at https://trailcheck.dev/.

Don't ask for either key in the chat. Ask the customer to set both as environment variables in their own terminal (`TRAILCHECK_API_KEY` and `MODEL_KEY`), and read them from there. If a key is pasted into the chat anyway, warn the customer about the security issues, but allow them to use it if they choose to. When the run is done, tell the customer to replace it: a new TrailCheck key from the app, and a new AI model key from their AI company.

## What counts

Each accepted start counts as one call: review, validation, or combine, including a rerun. The count is decremented immediately when a call is accepted, but if the job ends with no report, you aren't charged a call for that run. A saved report counts as one call. Each call is one third of the review + cold build as sold in the app (1 review + cold build = 1 review call, 1 build validation call, and 1 combine call). However the customer can use the 15 available calls in any ratio. A top-up adds 3 calls and does not change how many have been used or the paid-through date. The server refuses the start when there are no calls left, or the paid-through date has passed. When the server accepts the start, `remainingCalls` in the 202 says how many calls are still available. When the server refuses the start, the status is 402, and the response has `remainingCalls` when the server knows that number. Calls to fetch a finished job and write a fix file do not count toward the call quota.

**Note:** `GET /jobs` lists the jobs this API key started, newest first. `[]` when there are none. Keep the `jobId` from the 202. The list is how to find that id again. Every start POST is a new job. If the 202 already arrived, GET the files (and `GET /jobs/<Job ID>/info` if desired).

## Sequences

Note: At the moment, the cold build is a paper build only: the agent works through the use cases from your resources. It doesn't run code or send authenticated requests to your API.

Typical order: Review → Validation → Combined. That workflow in [`arazzo.yaml`](./arazzo.yaml) is `review-then-validation-then-combined`. Pieces: `review`, `validation`, `combined`, `run-again`, `check-and-fetch`. Combined is intended for a Review and a Validation of the same use cases. The host does not check. Unrelated jobs will combine to create a misleading report.

If the customer wants to save calls, they can skip the Combined step. Often the Validation report and fix file are enough. A finding that appeared only in the Review is occasionally missing from the Validation outputs.

## What comes back

The server POSTs to `callbackUrl` (when provided) when there is a report. One try. Body: `reportMarkdown` and `fixMarkdown` (Validation also `pathMarkdown`). GET info / report / fix still run if called. Use an `https` address. The server sends the report to any address you give, and a plain `http` address sends it unencrypted. The callback is not signed yet. To be sure a report came from TrailCheck, fetch it with `GET /jobs/<Job ID>/report`.

Any start can include `notifyEmail`: an email address to receive a notification when the job ends. The email is sent from info@trailcheck.dev with the job status, the score or the reason the job failed, and how to get the resulting files. Files are not attached. Non-email addresses trigger an error (400 `notifyEmail must be an email address`). The address appears in the job info. Generate Fix File doesn't send an email.

The report (`GET /jobs/<Job ID>/report`): 200 the report. 404 `{ "error": "no report yet" }` while `"In progress"`. 404 `{ "error": "not found" }` unknown `jobId`.

The fix file (`GET /jobs/<Job ID>/fix`): 200 the fix file. 404 `{ "error": "not found" }` — unknown `jobId`, or no fix file yet. Wait while `"In progress"`. Stop when `fixFileStatus` is there and is not `writing the fix file`: it says why there is no fix file and what to do, and `fixFileCode` gives the reason as a code. The report is saved and the job is `"Completed"`. GET info first when those 404s need telling apart.

The path file on Validation (`GET /jobs/<Job ID>/path`): 200 the path file. 404 `{ "error": "no path file yet" }` while it is not written. 404 `{ "error": "not found" }` unknown `jobId`.

Job info (`GET /jobs/<Job ID>/info`): 200 JSON. Files are not in this call. `"In progress"`: wait once a minute. `"Completed"`: `finishedAt`, `score`, `scoreMax` (the maximum possible score) and `summary` when parsed. If the use cases couldn't be read, the job is still `"Completed"`. It has `useCaseScoresNote` and no `useCaseRecords`. `"Failed"`: `failureDetails` (a string: what happened, then what to do), `failureCode` and `finishedAt`. 404 `{ "error": "not found" }`: this `jobId` is unknown. `failureCode` gives the reason as a code. `fixFileStatus` is there when the report exists and there is no complete fix file: it says why and what to do, and `fixFileCode` gives the reason as a code (`writing`, `incomplete`, `refused`, `not_handed_in`, or a failure code). An incomplete fix file says so at the top, and calling Generate Fix File again goes on from where it stopped. The job is still `"Completed"`: the report is saved and the call is used. To get one, call Generate Fix File: `POST /jobs/<Job ID>/fix` with the same authentication and `aiModel` as a start. 202 `{ "jobId" }`, the same ID. Not counted. The job's status does not change: GET info until `fixFileStatus` is no longer `writing the fix file`, then GET the fix file. Calling it again while one is being written → `{ "error": "the fix file is already being written" }` (400). Sequence: `generate-fix` in [`arazzo.yaml`](./arazzo.yaml).

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

Send a public URL when the page or file is already available publicly as HTML, JSON, YAML, Markdown, or plain text (This includes OpenAPI files and llms.txt).

Anything else must be turned into one of these:

- A PDF, a Word file, a Google Doc, or a Confluence or Notion export: Markdown. Send that Markdown as its own string in `resources`, or publish it and send the URL.
- A spec that exists only as an internal file: file text or a string.
- A screenshot or diagram: written out in words as Markdown. The image file itself is not read.

Each entry in `resources` is a string. An object is left out of the job.

A start needs either `resources` or `priorReport`. If both are missing, the host returns `{ "error": "at least one resource or prior report is required" }` (400).

Optional on a Review: `guidelines`, a markdown string of their API conventions. The review checks the API against that file and against the checklist. Omit it when they have none. On a run-again, omitting it keeps the guidelines from the prior Review when that job had them. A string on this start is the file for this job. Send it on `POST /review`. On Validation or Combined the host returns `{ "error": "send guidelines on the Review start" }` (400). A value that is not a string returns `{ "error": "guidelines must be markdown text" }` (400).
