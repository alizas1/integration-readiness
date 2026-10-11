# TrailCheck

Written **2026-09-24**. Updated **2026-10-11**.

Public contract for the TrailCheck API. Someone pays at https://trailcheck.dev/, then an agent calls this API with that API key and their AI model key.

Read [SKILL.md](./SKILL.md) before calling. This page is not the call instructions. The address, the headers, and the errors are there.

- [SKILL.md](./SKILL.md) — how an agent calls
- [openapi.yaml](./openapi.yaml) — the contract
- [arazzo.yaml](./arazzo.yaml) — step order and job id handoffs
- [Rendered reference](https://alizas1.github.io/integration-readiness/) — that same contract in the browser. It does not show this page.

A first run starts a review. Then it starts the cold build, the Validation call, from that review's job id. Save the job id. There is no list of past runs. Send the public docs, spec, and samples. [SKILL.md](./SKILL.md) says how to send a file that is not already a public page.
