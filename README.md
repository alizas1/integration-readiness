# TrailCheck

Last updated **2026-10-30**.

Public contract for the TrailCheck API. Purchase a plan at https://trailcheck.dev/ and get an API key, then call this API with that API key and your AI model key. A full review includes 3 calls. A review takes between 3-20 minutes. A validation build takes longer.

- [SKILL.md](./SKILL.md): how an agent calls.
- [openapi.yaml](./openapi.yaml): the contract.
- [Rendered reference](https://alizas1.github.io/integration-readiness/): that same contract in the browser. 
- [arazzo.yaml](./arazzo.yaml): step order and job id handoffs.
- [TUTORIAL.md](./TUTORIAL.md): step-by-step guide for humans running a full review and applying the fixes.

Process:
1. Review - reads your resources, discovers use cases and gaps.
2. Validation - spins up an agent that runs a cold build to complete the use cases, validates and discovers gaps.
   **Note:** At the moment, the cold build is a paper build only: the agent works through the use cases from your resources. It doesn't run code or send authenticated requests to your API.
3. Combine - combines the information gathered in the steps above into a report and fix file.

For agents: Read [SKILL.md](./SKILL.md) before calling. 
