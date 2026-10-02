# Integration Readiness

Written **2026-09-24**. Updated **2026-10-02**.

Public contract for the Review Server API.

For now, reach out to alizasolomondx@gmail.com to get an API key.

Each accepted start counts as one call: review, validation, combined, or run-again. If that job ends with no report, the call is put back. A saved report keeps the call. Each call is one third of the review + cold build as sold in the app (1 review + cold build = 1 review call, 1 build validation call, and 1 combine call). However the customer can use the 15 available calls in any ratio. Included is 15. Included returns to 15 and used returns to 0 only when they pay for another month. A top-up adds 3 to included and does not change used or the paid-through date. The server refuses the start when used has reached included, or the paid-through date has passed. An accepted start returns `left` (included minus used). A refusal is 402 and includes `left` when the server knows the number. Fetching a finished job is not counted. Writing the fix file is not counted.

- [Rendered reference](https://alizas1.github.io/integration-readiness/)
- [openapi.yaml](./openapi.yaml) — the contract
- [arazzo.yaml](./arazzo.yaml) — step order and job id handoffs
- [SKILL.md](./SKILL.md) — how an agent calls the host
