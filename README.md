# Integration Readiness

Written **2026-09-24**. Updated **2026-10-02**.

Public contract for the Review Server API.

For now, reach out to alizasolomondx@gmail.com to get an API key.

Each accepted start counts as one call: review, validation, combined, or run-again. The count is decremented immediately when a call is accepted, but if the job ends with no report, the count returns to the original number. A saved report counts as one call. Each call is one third of the review + cold build as sold in the app (1 review + cold build = 1 review call, 1 build validation call, and 1 combine call). However the customer can use the 15 available calls in any ratio. A top-up adds 3 calls and does not change how many have been used or the paid-through date. The server refuses the start when there are no calls left, or the paid-through date has passed. When the server accepts the start, the response says how many calls are still available. When the server refuses the start, the status is 402, and the response says how many calls are still available when it knows that number. Calls to fetch a finished job and write a fix file do not count toward the call quota.

- [Rendered reference](https://alizas1.github.io/integration-readiness/)
- [openapi.yaml](./openapi.yaml) — the contract
- [arazzo.yaml](./arazzo.yaml) — step order and job id handoffs
- [SKILL.md](./SKILL.md) — how an agent calls the host
