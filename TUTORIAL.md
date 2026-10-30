# Make your API agent-ready with TrailCheck

Your API documentation and resources might be fine for a human developer and still stop an AI agent cold. Agents don't ask questions; they guess, and those guesses become broken integrations. TrailCheck shows you exactly where agents get stuck on your API and gives you the fixes to close each gap.

In this tutorial, you'll check your API with the TrailCheck API and run the recommended fixes.

You'll complete the following steps:

1. [Set up your keys](#step-1-set-up-your-keys)
2. [Start a Review](#step-2-start-a-review)
3. [Wait for the job](#step-3-wait-for-the-job)
4. [Start a Validation](#step-4-start-a-validation)
5. [Combine the Review and the Validation](#step-5-combine-the-review-and-the-validation)
6. [Apply the fix file](#step-6-apply-the-fix-file)

A full check uses 3 calls from your plan (steps 2-5). A review takes between 5-20 minutes. A validation build takes longer.

TrailCheck itself is a plain HTTPS and JSON API, so you can call it from any language or HTTP client.

The examples in this tutorial use `curl` to call the API, `jq` to read the JSON responses, and a bash shell to run them. 

You can run the check in one of two ways:

- **Run the calls yourself.** Follow steps 1-6 and run each example in your shell.
- **Have your coding agent run the calls.** Set up your keys in Step 1, then point your agent at [SKILL.md](./SKILL.md). It explains how to call TrailCheck, including the order of the calls and how to pass job IDs between them. For example, with Claude Code:

  ```bash
  claude "Read https://alizas1.github.io/integration-readiness/SKILL.md, then run a TrailCheck Review, Validation, and Combine on our API. Use the keys in TRAILCHECK_API_KEY and MODEL_KEY."
  ```

  The agent ends up with the report and fix file, so you can go straight to [Step 6](#step-6-apply-the-fix-file). Steps 2-5 show what the agent does for you.

## Step 1: Set up your keys

TrailCheck needs two keys on every start: a TrailCheck API key, and a key for the AI model that runs the check.

1. Purchase a plan at [trailcheck.dev](https://trailcheck.dev/) and get your TrailCheck API key. Save it in a secure location.

2. Get an API key from your AI model company. Anthropic, OpenAI, DeepSeek, xAI, Mistral, Groq, and Google keys work.

   **Note:** A ChatGPT login is not a key, and a Cursor key can't run a check.

3. Save both keys as environment variables:

   ```bash
   export TRAILCHECK_API_KEY="<your TrailCheck API key>"
   export MODEL_KEY="<your AI model key>"
   ```

4. Send a `GET` request to `/health` to check that the server is up. This call does not need a key.

   ```bash
   curl -s "https://agent-server-production-a722.up.railway.app/health"
   # {"ok":true}
   ```

## Step 2: Start a Review

In this step, you'll start a Review and get back a job ID. A Review reads your docs and spec like a new integrator would and finds the gaps for the use cases you name. You'll use the job ID in every later step.

1. Send a `POST` request to `/review`. The request body takes these fields:

   | Field | Required | What to send |
   | --- | --- | --- |
   | `resources` | Required for first time review | The URLs of your docs and spec, or their text as markdown strings. |
   | `aiModel.primaryName` | Yes | The name of any model your AI model key can call. |
   | `title` | No | A name for the job. It appears in the job info. |
   | `useCases` | No | Specific tasks an integrator should be able to finish, as `{ "specified": [...] }` if you have them. Omit it and the Review finds use cases on its own. |
   | `guidelines` | No | Your API style rules as one Markdown string. The Review checks your API against them. |
   | `callbackUrl` | No - but recommended | A URL that TrailCheck POSTs the report and fix file to when the job finishes. |
   | `priorReport` | No | `{ "jobId": "..." }` of an earlier Review, to run that same check again. |
   | `aiModel.cheapName` | No | A cheaper model for the lighter passes. Omit it and `primaryName` runs every pass. |
   | `aiModel.company` | No - but recommended| The AI model company, for example `anthropic`. Omit it and TrailCheck works it out from the key. |
   | `aiModel.key` | Yes - unless sent as a header | Your AI model key, if you don't send it in the `x-model-key` header. If you send both and they differ, the server returns `400`. |

   Send these headers:

   | Header | Required | What to send |
   | --- | --- | --- |
   | `Authorization` | Yes | `Bearer <your TrailCheck API key>` |
   | `x-model-key` | Yes - unless sent as `aiModel.key` | Your AI model key. |
   | `Content-Type` | Yes | `application/json` |

   ```bash
   REVIEW_JOB_ID=$(curl -s -X POST "https://agent-server-production-a722.up.railway.app/review" \
     -H "Authorization: Bearer $TRAILCHECK_API_KEY" \
     -H "x-model-key: $MODEL_KEY" \
     -H "Content-Type: application/json" \
     -d '{
       "title": "Petstore API",
       "resources": ["https://petstore3.swagger.io/api/v3/openapi.json"],
       "useCases": { "specified": ["add a pet", "find pets by status"] },
       "aiModel": { "primaryName": "claude-sonnet-4-5" }
     }' | jq -r '.jobId // .error')
   echo "Review job: $REVIEW_JOB_ID"
   # Code in next step goes here...
   ```

   The server returns `202` with `{ "jobId": "...", "remainingCalls": 14 }`. `remainingCalls` is how many calls your plan has left after this start.
   If it prints an error message instead of an ID, check the message. For example, `unauthorized` means the TrailCheck API key is wrong or missing. A `402` means your plan has no calls left, its paid-through date has passed, or the key has no plan attached.

**Important:** Keep the job ID. If you lose it, send a `GET` request to `/jobs` with your `Authorization` header. It lists every job your API key started, newest first.

**Note:** TrailCheck reads HTML, JSON, YAML, Markdown, and plain text. It doesn't read PDFs or images. Convert those to Markdown first, and send the Markdown as a string in `resources`.

## Step 3: Wait for the job

In this step, you'll check the Review's status and wait for it to finish.

1. Send a `GET` request to `/jobs/<jobId>/info` to check the job's status. The response has these fields:

   | Field | When it's there | What it means |
   | --- | --- | --- |
   | `status` | Always | `In progress`, `Completed`, or `Failed`. |
   | `id`, `type`, `title`, `startedAt` | Always | The job ID, `Review`, `Validation`, or `Combined`, the title you sent, and when it started. |
   | `finishedAt` | `Completed` or `Failed` | When the job ended. |
   | `score`, `summary` | `Completed` | The report's score and a short summary. |
   | `failureDetails` | `Failed` | Why the job failed. |
   | `fixFileStatus` | The report exists but the fix file doesn't | Why: `fix file was not saved…` or `fix file failed: …`. See the note in Step 5. |
   | `useCaseRecords` | After the report is saved | Each use case the report checked, with its score and finding numbers. |

   Send this header:

   | Header | Required | What to send |
   | --- | --- | --- |
   | `Authorization` | Yes | `Bearer <your TrailCheck API key>` |

   ```bash
   # ...Code from previous step
   curl -s "https://agent-server-production-a722.up.railway.app/jobs/$REVIEW_JOB_ID/info" -H "Authorization: Bearer $TRAILCHECK_API_KEY"
   # Code in next step goes here...
   ```

   When the job is done, the response looks like this:

   ```json
   {
     "id": "00000000-0000-0000-0000-000000000000",
     "status": "Completed",
     "type": "Review",
     "title": "Petstore API",
     "startedAt": "2026-09-15T12:00:00.000Z",
     "finishedAt": "2026-09-15T12:20:00.000Z",
     "score": 76,
     "summary": "what we found"
   }
   ```

2. Add a function that checks the status once a minute until the job finishes. It prints the final job info and fails if the job failed.

   ```bash
   # ...Code from previous step
   wait_for() {
     while true; do
       info=$(curl -s "https://agent-server-production-a722.up.railway.app/jobs/$1/info" -H "Authorization: Bearer $TRAILCHECK_API_KEY")
       status=$(echo "$info" | jq -r .status)
       [ "$status" != "In progress" ] && break
       sleep 60
     done
     echo "$info" | jq .
     [ "$status" = "Completed" ]
   }
   # Code in next step goes here...
   ```

3. Wait for the Review.

   ```bash
   # ...Code from previous step
   wait_for "$REVIEW_JOB_ID"
   # Code in next step goes here...
   ```

   **Optional:** The Review has its own report and fix file. You'll get them in Step 5, combined with the Validation's, so you don't need them now. If you want to see them, for example if a later step fails, send a `GET` request to `/jobs/<jobId>/report` or `/jobs/<jobId>/fix` with your `Authorization` header.

## Step 4: Start a Validation

In this step, you'll start a Validation. A Validation runs a cold build: an agent builds your use cases against the API and records where it gets stuck.

**Note:** At the moment, the cold build is a paper build only: the agent works through the use cases from your resources. It doesn't run code or send authenticated requests to your API.

1. Send a `POST` request to `/validation`. The request body takes these fields:

   | Field | Required | What to send |
   | --- | --- | --- |
   | `priorReport` | Yes | `{ "jobId": "..." }` with one job ID: the Review to validate, or an earlier Validation or Combined job to run again. The Validation reuses that job's resources and use cases. |
   | `aiModel.primaryName` | Yes | The name of any model your AI model key can call. |
   | `title` | No | A name for the job. It appears in the job info. |
   | `resources` | No | More URLs or Markdown strings. They're added to the prior job's resources. |
   | `useCases` | No | More use cases, as `{ "specified": [...] }`. They're added to the prior job's use cases. |
   | `callbackUrl` | No - but recommended | A URL that TrailCheck POSTs the report, fix file, and path file to when the job finishes. |
   | `aiModel.cheapName` | No | A cheaper model for the lighter passes. Omit it and `primaryName` runs every pass. |
   | `aiModel.company` | No - but recommended| The AI model company, for example `anthropic`. Omit it and TrailCheck works it out from the key. |
   | `aiModel.key` | Yes - unless sent as a header | Your AI model key, if you don't send it in the `x-model-key` header. If you send both and they differ, the server returns `400`. |

   Send these headers:

   | Header | Required | What to send |
   | --- | --- | --- |
   | `Authorization` | Yes | `Bearer <your TrailCheck API key>` |
   | `x-model-key` | Yes - unless sent as `aiModel.key` | Your AI model key. |
   | `Content-Type` | Yes | `application/json` |

   Don't send `guidelines`: the server returns `400`.

   ```bash
   # ...Code from previous step
   VALIDATION_JOB_ID=$(curl -s -X POST "https://agent-server-production-a722.up.railway.app/validation" \
     -H "Authorization: Bearer $TRAILCHECK_API_KEY" \
     -H "x-model-key: $MODEL_KEY" \
     -H "Content-Type: application/json" \
     -d '{
       "priorReport": { "jobId": "'"$REVIEW_JOB_ID"'" },
       "aiModel": { "primaryName": "claude-sonnet-4-5" }
     }' | jq -r '.jobId // .error')
   echo "Validation job: $VALIDATION_JOB_ID"
   # Code in next step goes here...
   ```

2. Wait for the Validation.

   ```bash
   # ...Code from previous step
   wait_for "$VALIDATION_JOB_ID"
   # Code in next step goes here...
   ```

   **Optional:** Like the Review, the Validation has its own report and fix file, plus a path file: the route the agent took through your API. To get them, send a `GET` request to `/jobs/<jobId>/report`, `/jobs/<jobId>/fix`, or `/jobs/<jobId>/path` with your `Authorization` header.

## Step 5: Combine the Review and the Validation

In this step, you'll combine both jobs into one report and one fix file.

1. Send a `POST` request to `/combine`. The request body takes these fields:

   | Field | Required | What to send |
   | --- | --- | --- |
   | `priorReport` | Yes | `{ "jobId": [...] }` with two job IDs: the Review and the Validation. |
   | `aiModel.primaryName` | Yes | The name of any model your AI model key can call. |
   | `title` | No | A name for the job. It appears in the job info. |
   | `callbackUrl` | No - but recommended | A URL that TrailCheck POSTs the report and fix file to when the job finishes. |
   | `aiModel.cheapName` | No | A cheaper model for the lighter passes. Omit it and `primaryName` runs every pass. |
   | `aiModel.company` | No - but recommended| The AI model company, for example `anthropic`. Omit it and TrailCheck works it out from the key. |
   | `aiModel.key` | Yes - unless sent as a header | Your AI model key, if you don't send it in the `x-model-key` header. If you send both and they differ, the server returns `400`. |

   Send these headers:

   | Header | Required | What to send |
   | --- | --- | --- |
   | `Authorization` | Yes | `Bearer <your TrailCheck API key>` |
   | `x-model-key` | Yes - unless sent as `aiModel.key` | Your AI model key. |
   | `Content-Type` | Yes | `application/json` |

   Don't send `guidelines`: the server returns `400`.

   ```bash
   # ...Code from previous step
   COMBINED_JOB_ID=$(curl -s -X POST "https://agent-server-production-a722.up.railway.app/combine" \
     -H "Authorization: Bearer $TRAILCHECK_API_KEY" \
     -H "x-model-key: $MODEL_KEY" \
     -H "Content-Type: application/json" \
     -d '{
       "priorReport": { "jobId": ["'"$REVIEW_JOB_ID"'", "'"$VALIDATION_JOB_ID"'"] },
       "aiModel": { "primaryName": "claude-sonnet-4-5" }
     }' | jq -r '.jobId // .error')
   echo "Combined job: $COMBINED_JOB_ID"
   # Code in next step goes here...
   ```

**Important:** Only combine a Review and a Validation of the same use cases. TrailCheck doesn't check this, and unrelated jobs give a misleading report.

2. Wait for the Combined job, then send `GET` requests to `/jobs/<jobId>/report` and `/jobs/<jobId>/fix` to save its files.

   Send this header:

   | Header | Required | What to send |
   | --- | --- | --- |
   | `Authorization` | Yes | `Bearer <your TrailCheck API key>` |

   ```bash
   # ...Code from previous step
   wait_for "$COMBINED_JOB_ID"
   curl -sf "https://agent-server-production-a722.up.railway.app/jobs/$COMBINED_JOB_ID/report" -H "Authorization: Bearer $TRAILCHECK_API_KEY" -o combined-report.md
   curl -sf "https://agent-server-production-a722.up.railway.app/jobs/$COMBINED_JOB_ID/fix" -H "Authorization: Bearer $TRAILCHECK_API_KEY" -o combined-fix.md
   ```

**Note:** If a job fails, the jobs before it still have their files. Send the same `GET` requests with the ID of the latest job that completed. For example, if the Combined job fails, use `$VALIDATION_JOB_ID`.

**Note:** If the job info has a `fixFileStatus`, the report exists but the fix file doesn't. The job is still Completed. To write the fix file, send `POST /jobs/<jobId>/fix` with the `Authorization`, `x-model-key`, and `Content-Type` headers and an `aiModel` body with `primaryName`. This doesn't use a call from your plan. It keeps the same job ID, so wait for it and get the fix file the same way:

```bash
curl -s -X POST "https://agent-server-production-a722.up.railway.app/jobs/$COMBINED_JOB_ID/fix" \
  -H "Authorization: Bearer $TRAILCHECK_API_KEY" \
  -H "x-model-key: $MODEL_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "aiModel": { "primaryName": "claude-sonnet-4-5" } }'
wait_for "$COMBINED_JOB_ID"
curl -sf "https://agent-server-production-a722.up.railway.app/jobs/$COMBINED_JOB_ID/fix" -H "Authorization: Bearer $TRAILCHECK_API_KEY" -o combined-fix.md
```

## Step 6: Apply the fix file

In this step, you'll hand the fix file to your coding agent. `combined-fix.md` is written for a coding agent to follow, and it holds the fixes for the gaps in the report.

1. Get the fix file to your coding agent:

   - **If an agent ran the calls for you:** It already has the fix file. Go to the next step and stay in the same session.
   - **If you ran the calls yourself:** Copy `combined-fix.md` into the repo where your docs and spec live.

2. Choose an optional run mode. The fix file supports these modes, and you choose one by adding its phrase to your prompt:

   | Run mode | Add to your prompt | What the agent does |
   | --- | --- | --- |
   | Default | Nothing | Checks each issue against your repo, then asks you before it changes anything. |
   | Handle it yourself | `handle it yourself` | Checks every issue, fixes the ones it confirms, and opens one PR without asking you anything. It doesn't change API or service behavior. For those issues, it adds notes to the PR instead. |
   | Docs only | `docs only` | Fixes only docs, sample, and spec issues, and opens one docs PR. First it asks you once for your docs guidelines and how to handle issues that need a developer: document around them, or collect them in a separate notes file. If you skip that question, it ignores those issues. |
   | Prove everything | `prove everything` | Goes through the issues one at a time in the chat, and waits for your yes before each change. It offers to open a PR once it has gone through every issue. |

   In every mode, the agent treats the fix file as leads, not facts. It checks each issue in your repo first and skips any that are already fixed.

3. Ask your coding agent to apply the fixes in the run mode you chose. If you're starting a new session, start it in the repo with your docs and spec. For example, with Claude Code:

   ```bash
   claude "Run combined-fix.md. Handle it yourself."
   ```

4. Review the agent's recommended changes, and make any required edits/changes. Then publish your updated docs and spec.

**Tip:** To confirm the fixes worked, run the check again. See [Next steps](#next-steps).

## Full code

Save this as `trailcheck.sh` and run it with `bash trailcheck.sh`.

```bash
#!/usr/bin/env bash
set -euo pipefail

curl -s "https://agent-server-production-a722.up.railway.app/health"

wait_for() {
  while true; do
    info=$(curl -s "https://agent-server-production-a722.up.railway.app/jobs/$1/info" -H "Authorization: Bearer $TRAILCHECK_API_KEY")
    status=$(echo "$info" | jq -r .status)
    [ "$status" != "In progress" ] && break
    sleep 60
  done
  echo "$info" | jq .
  [ "$status" = "Completed" ]
}

# Review
REVIEW_JOB_ID=$(curl -s -X POST "https://agent-server-production-a722.up.railway.app/review" \
  -H "Authorization: Bearer $TRAILCHECK_API_KEY" \
  -H "x-model-key: $MODEL_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Petstore API",
    "resources": ["https://petstore3.swagger.io/api/v3/openapi.json"],
    "useCases": { "specified": ["add a pet", "find pets by status"] },
    "aiModel": { "primaryName": "claude-sonnet-4-5" }
  }' | jq -r '.jobId // .error')
echo "Review job: $REVIEW_JOB_ID"

wait_for "$REVIEW_JOB_ID"

# Validation
VALIDATION_JOB_ID=$(curl -s -X POST "https://agent-server-production-a722.up.railway.app/validation" \
  -H "Authorization: Bearer $TRAILCHECK_API_KEY" \
  -H "x-model-key: $MODEL_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "priorReport": { "jobId": "'"$REVIEW_JOB_ID"'" },
    "aiModel": { "primaryName": "claude-sonnet-4-5" }
  }' | jq -r '.jobId // .error')
echo "Validation job: $VALIDATION_JOB_ID"

wait_for "$VALIDATION_JOB_ID"

# Combined
COMBINED_JOB_ID=$(curl -s -X POST "https://agent-server-production-a722.up.railway.app/combine" \
  -H "Authorization: Bearer $TRAILCHECK_API_KEY" \
  -H "x-model-key: $MODEL_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "priorReport": { "jobId": ["'"$REVIEW_JOB_ID"'", "'"$VALIDATION_JOB_ID"'"] },
    "aiModel": { "primaryName": "claude-sonnet-4-5" }
  }' | jq -r '.jobId // .error')
echo "Combined job: $COMBINED_JOB_ID"

wait_for "$COMBINED_JOB_ID"
curl -sf "https://agent-server-production-a722.up.railway.app/jobs/$COMBINED_JOB_ID/report" -H "Authorization: Bearer $TRAILCHECK_API_KEY" -o combined-report.md
curl -sf "https://agent-server-production-a722.up.railway.app/jobs/$COMBINED_JOB_ID/fix" -H "Authorization: Bearer $TRAILCHECK_API_KEY" -o combined-fix.md
```

## Next steps

- **Run it again after you fix things.** Start a new Review with `"priorReport": { "jobId": "<Review job ID>" }`. It keeps the earlier use cases and adds any new `resources` you send.
- **Check against your own conventions.** Send `guidelines`, a Markdown string of your API style rules, on the Review start.
- **Skip the waiting loop.** Add a `callbackUrl` to any start. TrailCheck POSTs the report and fix file to it once, so keep polling as a fallback.
- **Watch your call count.** Each start uses one call from your plan, and a job that ends with no report gives the call back. Fetching files and writing a fix file are free. Jobs stay on the host for 3 months.
