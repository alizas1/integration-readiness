---
name: review-server
description: >
  Start a Review or Validation on the TrailCheck API, keep the job id, and
  fetch the report and fix file. Use when calling this host with an API key
  from an agent.
---

# TrailCheck API

Written **2026-09-16**. Updated **2026-11-26**.

How an API key customer’s agent calls the host. Contract: [`openapi.yaml`](./openapi.yaml). Sequences and handoffs: [`arazzo.yaml`](./arazzo.yaml).

This file explains how to call the TrailCheck API. The review method runs on the server, so the caller needs nothing else to run a check. Each job returns its file: the report (`report.md`) from a Review, Validation or Combined job, the path file (`path.md`) from a Validation, and the fix file (`fix.md`) from a Fix job.

Host: `https://agent-server-production-a722.up.railway.app`

Before the run starts, if you can play a sound on the customer's computer (for example, from a terminal), offer to make a sound when the process is complete. Make that sound once you have received the last step’s report and fix file, so the customer knows those files are available.

## If they don't have a key yet

Before anything else, ask whether they have a TrailCheck API key. If not, send them to https://trailcheck.dev/ to buy one. The app gives them the key after they pay. Then have them set it and their AI model key as described under Authenticate, and start with the questions below.

## Before you start a Review

Ask these questions one at a time. Wait for each answer before asking the next. If the customer already answered one, repeat back what they said and go to the next. Choosing an API is not an answer to question 1. If they pick an API you offered, or one they named, list the links you'll send as `resources` and the hosts the review will read. Ask whether anything they care about is on another host, for example a sample on GitHub, and whether to add docs pages, llms.txt or anything else before you go on.

1. What should the review cover? For example your API spec, llms.txt and/or docs site, the getting started and authentication pages, the guide for each use case, and sample code. Provide links or files. If I can open your project, I can look in it to find them. The review reads only the sites in the links you send. Links to other sites are skipped, so send a link for each site you need.

2. What kind of review should I run?
   - The full check plus fix file: the review reads your docs, an agent then tries to build your use cases from them and returns one report with the findings from both, and a fix file is written from that report. Uses 3 calls. Best if you can apply fixes yourself.
   - The full check without a fix file: same as above but doesn't write the fix file. Uses 2 calls. Best if you can't apply fixes yourself.
   - Something else?

If they ask what else, offer: the review plus a fix file, without the build (2 calls), or just the review (1 call).

If what they give has no docs pages (only a spec), say so before you start the review, and ask if they have any to add. Tell them the report will count it as a finding.

If you can't open the customer's project or send requests yourself (for example in a chat app), say so. Give the customer each request to run in their own terminal, with the keys left as placeholders, and ask them to paste back the `jobId` and the files.

## Authenticate

`Authorization: Bearer` plus the API key, on every start and every GET. `aiModel` on every start body (`primaryName` required). AI model key in `aiModel.key` or header `x-model-key`. Optional: `aiModel.cheapName`, `aiModel.company`. Omit `company` → the host guesses from the key. Names it accepts: `anthropic` / `claude`, `openai` / `chatgpt` / `gpt`, `deepseek`, `xai` / `grok`, `mistral`, `groq`, `google` / `gemini`. Tested with Anthropic keys. OpenAI, Groq, DeepSeek, xAI, Mistral, and Google keys are accepted but untested, and a long report or fix file may fail with them. A ChatGPT login is not a key. A Cursor key cannot run this loop. An unknown company name → `{ "error": "could not tell which company this key is for" }` — send `aiModel.company`.

Unknown or missing API key → `{ "error": "unauthorized" }` (401). Missing AI model key → `{ "error": "an ai model key is required" }` (400). Missing `primaryName` → `{ "error": "primaryName is required" }` (400).

Only jobs started with that API key can be read. Purchase an API key at https://trailcheck.dev/.

Don't ask for either key in the chat. Ask the customer to set both as environment variables in their own terminal (`TRAILCHECK_API_KEY` and `MODEL_KEY`), and read them from there. If a key is pasted into the chat anyway, warn the customer about the security issues, but allow them to use it if they choose to. Only in that case, when the run is done, tell the customer to replace the pasted key: a new TrailCheck key from the app, or a new AI model key from their AI company.

## What counts

Each accepted start counts as one call: review, validation, fix file, or combine, including a rerun and a start again from a failed job. The count is decremented immediately when a call is accepted, but if the job ends with no report/fix file, you aren't charged a call for that run. Each successful report or fix file generation costs one call. Each call is one third of the review + cold build + fix file as sold in the app (1 review + cold build + fix file = 1 review call, 1 build validation call, and 1 fix file call). However the customer can use the 15 available calls in any ratio. A top-up adds 3 calls and does not change how many have been used or the paid-through date. Two starts don't use a call: a Fix job that continues a `Partial` one, and the first `/combine` that names a Validation whose merge didn't finish (see `mergeNote`). The server refuses the start when there are no calls left, or the paid-through date has passed. When the server accepts the start, `remainingCalls` in the 202 says how many calls are still available. It's left out when the start didn't use a call. When the server refuses the start, the status is 402, and the response has `remainingCalls` when the server knows that number. Calls to fetch a finished job do not count toward the call quota.

Calls left: `GET /balance`, with the same `Authorization` header. Checking doesn't use a call. 200 `{ "remainingCalls": 13, "used": 2, "included": 15, "paidThrough": "2026-12-15" }`. Job info has `callUsed`: `false` when the job ended with no report/fix file and the call was given back, or when the start didn't use a call.

**Note:** Keep the `jobId` from the 202. Every start POST is a new job.

## Sequences

Note: At the moment, the cold build is a paper build only: the agent works through the use cases from your resources. It doesn't run code or send authenticated requests to your API.

Typical order: Review → Validation → fix file. A Validation that starts from a Review returns the combined report: the Review's findings and the Validation's, in one report. A Fix job started from the Validation then writes one fix file for both. That workflow in [`arazzo.yaml`](./arazzo.yaml) is `review-then-validation-then-fix`. Pieces: `review`, `validation`, `generate-fix`, `fetch-fix`, `combined`, `combine-jobs`, `run-again`, `check-and-fetch`. Combine is an optional call available for non-standard flows: use it to join jobs yourself, for example when use cases for the same resources were split into two Reviews, or into two Validations from one Review. The Combined call is intended for jobs about the same resources, but the host does not check this. Pass one or more Review jobs, plus any related Validation jobs. Unrelated jobs will combine to create a misleading report.

## What comes back

Any start (`POST /review`, `/validation`, `/fix` or `/combine`) can include `notifyEmail`: an email address to receive a notification when the job ends. The email is sent from info@trailcheck.dev with the job status, the score or the reason the job failed, and how to get the resulting files. Files are not attached. Non-email addresses trigger an error (400 `notifyEmail must be an email address`). The address appears in the job info.

The report (`GET /jobs/<Job ID>/report`): 200 the report. A Validation that started from a Review returns the combined report, unless its job info has `mergeNote`. 409 `{ "error": "job is in progress" }` while `"In progress"`: wait. 404 `{ "error": "not found" }` unknown `jobId`. 404 `{ "error": "this job failed, no report available" }`. 404 `{ "error": "Fix job ID passed for a report. The report is on job <Job ID>." }`.

When links were skipped because their sites weren't in `resources`, the report lists them under Skipped links in What We Didn't Review. Tell the customer which sites were skipped, and ask whether it's worth starting a new Review that includes them. It uses a call.

The fix file (`GET /jobs/<Job ID>/fix`): 200 the fix file. On a `Partial` job, it's the part that was written, and the top of the file says where it stops and how to generate the rest. 409 `{ "error": "job is in progress" }`: wait. 404 `{ "error": "not found" }` unknown `jobId`. 404 `{ "error": "this job failed, no fix file available" }`. 404 on a Review, Validation or Combined job: `{ "error": "Review job ID passed for a fix file. The fix file is on Fix job <Job ID>." }`, naming the latest Fix job made from it; `{ "error": "Review job ID passed for a fix file. No fix file was made from this job yet: call POST /fix." }` when none was made; `{ "error": "Review job ID passed for a fix file. This job failed, so no fix file can be made from it." }` when that job failed.

The path file on Validation (`GET /jobs/<Job ID>/path`): 200 the path file. 409 `{ "error": "job is in progress" }` while `"In progress"`: wait. 404 `{ "error": "only Validation jobs have a path file" }`. 404 `{ "error": "not found" }` unknown `jobId`.

Job info (`GET /jobs/<Job ID>/info`): 200 JSON. Files are not in this call. 404 `{ "error": "not found" }`: this `jobId` is unknown. `"In progress"`: wait once a minute. `"Completed"`: `finishedAt`, `score`, `scoreMax` (the maximum possible score) and `summary` when parsed. `"Partial"`: a Fix job whose fix file stops partway. `partialFixDetails` says where. A Fix job started from it generates the rest, without using a call. `"Failed"`: `failureDetails` (a string: what happened, then what to do), `failureCode` (the reason as a code) and `finishedAt`. On a Fix job, `failureCode` can also be `refused` (the fix file check refused a line) or `not_handed_in` (the AI didn't hand in a fix file). If the use cases couldn't be read, the job is still `"Completed"`. It has `useCaseScoresNote` and no `useCaseRecords`. `useCasesNote`: the use cases section had extra text. That text isn't in `useCaseRecords`. Read it in the report. Each entry in `useCaseRecords` has an `id`: `uc-1`, `uc-2`, and so on. A job started from an earlier job keeps that job's IDs for the same use cases. Send them in `useCases.ids` to choose which use cases a new job checks.

Job info also shows how a job connects to others, so you can find the related files:

- `priorReportJobIds`: the earlier reports this job was made from. On a Validation from a Review, the Review. On a Combined job, every job it joined. On a rerun, the job it reran. On a Fix job, the report its fix file was made from.
- `includesReviewFindings`: `true` when a Validation's report already includes the Review's findings. You don't need to call `/combine`.
- `mergeNote`: returned when the Validation's report doesn't include the Review's findings. To get them in one report, call `/combine` with the Validation and the Review. The first `/combine` after this is free. The note says: `Review and Validation steps both completed successfully, but the reports didn't merge as expected. To get the combined report, call POST /combine with both job IDs. It's free this time.`
- `fixJobIds`: the fix files made from this report, oldest first. Get the newest with `GET /jobs/<last ID>/fix`.
- `continuedFromJobId`: this job picks up from an earlier one. It finishes a partial fix file, or retries a job that failed.

`GET /jobs` lists the key's jobs, newest first: `id`, `type`, `title`, `status`, `startedAt`, `finishedAt`, `score`. Filters: `type`, `status`, `limit`. `type` can be `Review`, `Validation`, `Combined` or `Fix`, and `status` can be `In progress`, `Completed`, `Failed` or `Partial`. The rest of a job's details are at `GET /jobs/<Job ID>/info`. The report file is at `GET /jobs/<Job ID>/report`, the fix file is at `GET /jobs/<Job ID>/fix`, and the path file is at `GET /jobs/<Job ID>/path`.

`scoreStatus` is present in two cases. When the score in the report did not match a count based on the findings, it names both numbers, e.g. `The report says score: 70, but counting the findings adds up to 76.` `score` and `report.md` then use that second number (76 in that string). When the server couldn't read the report's findings, it's `The server couldn't read the report's findings, so the score wasn't confirmed.` The job is still `"Completed"`, and `score` is the number the report gives, when it gives one.

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

For a Validation job (`POST /validation`) that builds on a Review job, name that Review job:

```json
{
  "priorReport": { "jobId": "<Review job ID>" },
  "aiModel": { "primaryName": "<model name>", "key": "<AI model key>" }
}
```

It uses the Review job's resources and use cases, and returns the combined report: the Review's findings and the Validation's, in one report. With no `title`, it uses the Review's. If the customer already ran a Review job on the same API, always send `priorReport`. Without it, the Validation job might not use the same use cases as the Review job.

A Validation job can also start without a Review job: send `resources` the same way as for a Review job. It finds its own use cases.

To run the same Validation job again, put its job ID in `priorReport.jobId`. A Validation started from a Validation that has `mergeNote`, or from a Combined job, may leave out findings only the Review had. To include any Review findings into the new report, call `/combine` with the Review and the new Validation.

For a Combined job (`POST /combine`), list the jobs to join: at least one Review job, plus any Validation jobs. If a Validation's report already includes the Review's findings, they aren't counted twice.

```json
{
  "priorReport": { "jobIds": ["<Review job ID>", "<Validation job ID>"] },
  "aiModel": { "primaryName": "<model name>", "key": "<AI model key>" }
}
```

For a Fix job (`POST /fix`), name the job whose report the fix file should be based on: a Review, Validation or Combined job:

```json
{
  "priorReport": { "jobId": "<Review, Validation or Combined job ID>" },
  "aiModel": { "primaryName": "<model name>", "key": "<AI model key>" }
}
```

It takes only `priorReport`, `aiModel` and `notifyEmail`. Its `title` is the title of that job, and with no `notifyEmail` it uses that job's address. To finish a `Partial` fix file, name the Partial Fix job: the new one generates the rest, without using a call. Naming a Failed or Completed Fix job writes a new fix file from the same report, and uses a call.

Any Review or Validation start can choose which use cases to check, in `useCases`:

```json
"useCases": { "specified": ["Send a payment"], "max": 2, "ids": ["uc-1"] }
```

- `specified`: use cases by title. Up to 4.
- `max`: how many use cases to check in all, 1 to 4. Without it, defaults to 4.
- `ids`: use case IDs from the job in `priorReport.jobId`, from its `useCaseRecords`. Send `priorReport.jobId` with it.

The ones named in `ids` and `specified` are checked first, and the rest are chosen up to `max`. With `ids` and no `max`, only those are checked. A Validation from a Review that checks only some use cases still includes all the Review's findings, and lists the use cases it didn't check as not run.

HTML, JSON, YAML, Markdown, and plain text are read. PDF, image, audio, video, and `octet-stream` are not.

A Review reads only the hosts in `resources`, plus hosts that an llms.txt in `resources` links to. A link it finds to any other host, for example GitHub, is not read. Send a URL on each host you need: every docs host, the spec host, the sample repo. Redirects are followed, up to 5, as long as the host was included in resources or in an llms.txt in resources. If a page redirects to a site that isn't in resources, the page it redirects to is skipped, like any other link to that site.

A job reads up to 85% of the model's limit (Claude keys). When it gets there, it stops reading and writes the report with what it has. Pages not read are listed under What We Didn't Review. A job works in steps, and at every step everything read so far is sent again, billed to the AI model key each time. If the customer sends a large spec or many pages, tell them this before you start. With other companies' keys, a job that reads too much fails, and the call is given back.

Send a public URL when the page or file is already available publicly as HTML, JSON, YAML, Markdown, or plain text (This includes OpenAPI files and llms.txt).

Anything else must be turned into one of these:

- A PDF, a Word file, a Google Doc, or a Confluence or Notion export: Markdown. Send that Markdown as its own string in `resources`, or publish it and send the URL.
- A spec that exists only as an internal file: file text or a string.
- A screenshot or diagram: written out in words as Markdown. The image file itself is not read.

Each entry in `resources` is a string. An object is left out of the job. `resources` that isn't a string or a list → `{ "error": "resources must be a string or list of strings" }` (400). A field that isn't listed in [`openapi.yaml`](./openapi.yaml) → `{ "error": "unknown field: <name>" }` (400), before a call is counted. A `priorReport` that names a job still running → `{ "error": "job is in progress" }` (409). A failed job of another kind → for example `Review job <Job ID> failed, so it has no report to build on. To start it again, send POST /review with this job ID as priorReport, then start the validation from the new review.` (400). On `POST /fix`: no `priorReport.jobId` → `jobId missing - send priorReport.jobId of the report the fix file will be based on` (400); a failed Review, Validation or Combined job → `this job failed, so it has no report to write a fix file from` (400); a Fix job already running for that report → `the fix file is already being written` (409). On `useCases`, all 400: `useCases.max must be a whole number from 1 to 4`; `useCases.ids must be a list of use case IDs`; `when specifying useCases.ids, priorReport.jobId must also be passed`; `use case uc-9 couldn't be found in job <Job ID>`; more use cases named than `max` → `3 use cases specified in useCases.ids and useCases.specified but useCases.max is 2`.

A start needs either `resources` or `priorReport`. If both are missing, the host returns `{ "error": "at least one resource or prior report is required" }` (400). A start can name a failed job in `priorReport.jobId` to start it again. It copies the failed job's resources, use case titles, title, style guide, notify email, and the job it started from, so a Validation from a Review still starts from that Review. Anything sent on the new start replaces what's copied. The new start sets its own limit: send `useCases.max` to choose how many use cases it checks. Without it, it checks up to 4. It must be the same kind of start as the failed job, and it uses a call. On `/combine`, send the failed job's ID as the only entry in `priorReport.jobIds`. When a job fails, `failureDetails` says what went wrong and what to fix. Apply the recommended fixes, then start it again.

Optional on a Review: `guidelines`, their API style guide as a markdown string. The review checks the API against that file and against the checklist. Omit it when they have none. On a run-again, omitting it keeps the guidelines from the prior Review when that job had them. A string on this start is the file for this job. Send it on `POST /review`. On Validation or Combined the host returns `{ "error": "send guidelines on the Review start" }` (400). A value that is not a string returns `{ "error": "guidelines must be markdown text" }` (400).
