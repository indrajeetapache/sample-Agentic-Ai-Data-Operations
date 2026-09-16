# Indrajit — Personal Analysis Pack

Analysis workspace for understanding **sample-Agentic-Ai-Data-Operations** (ADOP / Agentic Data Onboarding Platform).

**Date started:** 2026-09-15  
**Repo:** https://github.com/aws-samples/sample-Agentic-Ai-Data-Operations  
**Local path:** this clone

## Contents

| File | Purpose |
|------|---------|
| [00_EXECUTIVE_SUMMARY.md](./00_EXECUTIVE_SUMMARY.md) | One-page verdict + how it works + top pitfalls |
| [01_CLAIMS_VERIFICATION.md](./01_CLAIMS_VERIFICATION.md) | Verify marketing / deck claims against repo evidence |
| [02_E2E_FLOW.md](./02_E2E_FLOW.md) | End-to-end: build-time agents + run-time pipeline |
| [03_PITFALLS_AND_GAPS.md](./03_PITFALLS_AND_GAPS.md) | Risks, incomplete areas, doc contradictions |
| [04_ARCHITECTURE_DEEP_DIVE.md](./04_ARCHITECTURE_DEEP_DIVE.md) | Agents, zones, codegen, MCP, security model |
| [diagrams/](./diagrams/) | Mermaid sources — drop PNGs here when you export |

## How to add your diagram

1. Open `diagrams/dev-only-agent-promotion.mmd`
2. Render via https://mermaid.live or:

```bash
cd Indrajit/diagrams
npx -y @mermaid-js/mermaid-cli@latest -i dev-only-agent-promotion.mmd -o dev-only-agent-promotion.png
```

3. Commit the PNG + this folder when ready to push (do not push until you ask).

## Quick verdict

**The deck claim is mostly true:** agents generate artifacts in Dev only; CI/CD promotes code to higher envs; agents do not run in Prod.

**Caveat:** repo frontmatter still says `design-complete, implementation pending`. Sample workloads exist, but CI/CD auto-promotion and full DevOps are **not fully productized** yet — see pitfalls doc.
