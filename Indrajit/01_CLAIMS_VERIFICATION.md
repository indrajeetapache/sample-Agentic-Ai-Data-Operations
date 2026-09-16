# 01 — Claims Verification

Source claim (from presentation screenshot):

> New GitHub repo (MIT). Automates data engineering via Claude Code on AWS Bedrock.  
> **All agents run in Development only.** Scripts/configs are version-controlled and promoted to QA / Staging / Production via CI/CD. Agents do **not** run in production — only artifacts.  
> Data Onboarding Agent orchestrates **7** other agents to generate a complete, tested pipeline.  
> https://github.com/aws-samples/sample-Agentic-Ai-Data-Operations

## Verdict matrix

| # | Claim | Status | Evidence | Notes |
|---|--------|--------|----------|-------|
| 1 | Public AWS Samples GitHub repo | **TRUE** | README + remote naming | This clone matches that sample |
| 2 | MIT license | **MOSTLY TRUE** | `LICENSE`, README L1111 | Exact license is **MIT-0** (MIT No Attribution), Amazon copyright |
| 3 | Automates data engineering (ETL, DQ, DAG, semantic) | **TRUE** | README “Why Agentify…”, `workloads/*`, `shared/templates/` | Designed for Claude Code; adaptable to other assistants |
| 4 | Claude Code + AWS Bedrock | **PARTIALLY TRUE** | README Bedrock `tool_choice` / `AgentOutput`; primary UX = Claude Code | Not “agents only on Bedrock Runtime” as sole runtime; Bedrock is used for structured agent output tooling |
| 5 | Agents run in **Development only** | **TRUE** | README L160, L479–487 | Explicit, non-negotiable design principle |
| 6 | Artifacts version-controlled | **TRUE** | `workloads/{name}/` committed; codegen headers | Intended path: generate → git → promote |
| 7 | Promoted via CI/CD to QA → Staging → Prod | **INTENDED / PARTIAL** | README L489–496 describes flow | Full auto CI/CD promote is **planned** (DevOps Q2–Q3 2026). Repo has some GitHub Actions (security), not a complete multi-env promote pipeline in-sample |
| 8 | Agents do not run in Production | **TRUE** | Same Environment Model section | Prod runs Glue/MWAA on generated scripts only |
| 9 | Data Onboarding Agent orchestrates **7** agents | **TRUE** | README L179–185 | Listed below |
| 10 | “Complete, tested pipeline without manual coding” | **ASPIRATIONAL / CONDITIONAL** | Test gates in workflow docs | Still requires human answers (Phase 1 gate), approvals, AWS setup, and often manual IaC apply |

## The 7 agents (orchestrated)

From `README.md` (Data Onboarding Agent coordination):

1. **Router Agent** — search `workloads/` for existing / partial / new  
2. **Metadata Agent** — profile schema, types, quality issues, semantic context  
3. **Ontology Agent** — OWL / semantic layer induction from catalog metadata  
4. **Data Quality Agent** — column rules, thresholds, pass/fail gates  
5. **Data Transformation Agent** — Glue Bronze→Silver→Gold scripts  
6. **Orchestration / Scheduling DAG Agent** — Airflow (or Step Functions) DAG  
7. **DevOps Agent** — IaC (CFN/Terraform/CDK) to promote artifacts  

Plus shared services (not counted in the “7”): Glue Catalog, Semantic Layer, Decision Engine, Memory, Agentrace logs.

## Related agents outside the “7”

These also exist in the platform narrative:

| Agent | Role | Maturity (repo docs) |
|-------|------|----------------------|
| Environment Setup | One-time AWS + MCP | Prompt-driven setup |
| Ontology Staging | Emit `ontology.ttl` + `mappings.ttl` | Staging only (no AWS Semantic Layer publish yet) |
| Prompt Intelligence | Learn from `trace_events.jsonl` | Available as shared module |
| Data Analysis / dashboard | Consume Gold | Referenced; less central |

## Doc contradictions to be aware of

1. **`CLAUDE.md` frontmatter:** `status: design-complete, implementation pending` vs README marketing that reads production-ready.  
2. **DevOps maturity:**  
   - README Environment Model: “DevOps Agent planned for Q2 2026”  
   - README §7: `/devops-workflow` “Available Now”  
   - `prompts/devops-agent/README.md`: Partial — IaC generator yes; CI/CD/monitoring still planned  
3. **CI/CD claim in the deck** describes the **target operating model**, not a fully shipped multi-account promote pipeline in this sample.

## Bottom line for the deck

Safe to say:

> Agents generate pipelines in Dev only. Generated code is git-versioned and **meant** to be promoted through CI/CD. Production runs artifacts, never agents. One orchestrator + seven specialist agents.

Avoid saying (without caveats):

> “Fully automated CI/CD to Production out of the box” or “100% hands-off / zero human input.”
