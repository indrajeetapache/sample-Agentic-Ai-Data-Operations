# Executive Summary — How ADOP Works & What to Watch

## What it is

An **agentic data onboarding platform**: describe a dataset; specialized agents generate a Bronze→Silver→Gold AWS lakehouse pipeline (Glue/Iceberg/Athena/Lake Formation/MWAA) under `workloads/{name}/`.

## Is the presentation claim true?

**Yes, with caveats.**

- Agents **do** run only in Development.  
- Artifacts **are** meant to be git-versioned and promoted; Prod runs **code**, not agents.  
- One Data Onboarding Agent + **7** specialists is accurate.  
- License is **MIT-0**, not classic MIT.  
- Full auto CI/CD to Prod and complete DevOps are **directionally true but not fully shipped** in this sample.

## How it works (compressed)

1. **Build (Dev):** Health check → human discovery → dedup → profile → generate specs → render templates → test gates → MCP deploy.  
2. **Promote:** Git → CI/CD → QA → Staging → Prod (org-owned maturity varies).  
3. **Run (Prod):** Airflow triggers Glue: Bronze (immutable) → Silver (≥80% DQ) → Gold (≥95% DQ).

## Biggest pitfalls

1. Human gate skipped → wrong business/compliance rules.  
2. Assuming CI/CD/DevOps is turnkey.  
3. MCP/AWS setup incomplete → Phase 0 blocks.  
4. Hand-editing generated scripts → drift failures.  
5. “HIPAA workflow” ≠ audit-ready without LF-Tags, masking, and process.  
6. Docs disagree on DevOps readiness / `implementation pending` status.

## How do I use it as a user?

See **`05_USER_ONBOARDING_GUIDE.md`** — setup → prompt → discovery answers → deploy → promote.

Coming from **Spark → Hive/Snowflake** with files on S3 (CSV/EBCDIC) and Iceberg support?  
See **`06_SPARK_HIVE_SF_COMPARISON_AND_PROMPTS.md`**.

**Who runs Claude, who pays, where to configure, how to configure agents, full E2E plan?**  
See **`07_E2E_PLAN_COST_CONFIG.md`** — Section **3.B** is the open-source agent config file map.

## Folder map

Start here → `07_` (cost/config/E2E plan) → `06_` (your stack) → `05_` (usage) → `01_`–`04_` (analysis) → `diagrams/`.
