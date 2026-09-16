# 05 — User Onboarding & How to Use Guide

**Audience:** You (data engineer / platform user) using this repo to onboard a dataset.  
**Status of this guide:** Written from repo prompts + docs (verified against `docs/getting-started.md`, `prompts/`, `CLAUDE.md`).  
**What was missing before:** `Indrajit/` had *how the system works* and *pitfalls*, but **not** a step-by-step “what do I type / answer / run” user guide. This file closes that gap.

---

## 1. What “onboarding” means here

You describe a dataset in natural language (in Claude Code). Agents ask clarifying questions, then generate a full pipeline under:

```text
workloads/<your_dataset_name>/
```

That folder becomes the source of truth: configs, Glue/PySpark scripts, quality rules, Airflow DAG, tests, logs.

Agents run **only in Development**. Higher environments run the **generated artifacts**, not the agents.

---

## 2. Prerequisites (do once)

| Need | Why |
|------|-----|
| This repo cloned | Agents read prompts, `shared/`, `workloads/` |
| AWS account + credentials (Dev) | Glue, S3, Athena, Lake Formation, MWAA |
| Claude Code (or compatible assistant) open in this repo | Orchestrates agents |
| MCP servers configured (`.mcp.json` local or Gateway) | AWS operations via MCP-first |
| Python + ability to run `pytest` locally | Test gates |

**First-time AWS setup (mandatory before first onboard):**

In Claude Code, paste something like:

```text
Setup AWS environment for the Agentic Data Onboarding platform.

Account details:
- AWS Region: us-east-1
- Project name: data-onboarding
- Environment: dev

What I need created:
- [x] IAM roles
- [x] S3 data lake bucket
- [x] KMS encryption keys
- [x] Glue databases
- [x] Lake Formation LF-Tags
- [x] Lake Formation TBAC grants

Existing resources: none
```

Official prompts live under `prompts/environment-setup-agent/`:

1. `01-setup-aws-infrastructure.md` (required)  
2. `02-deploy-agentcore-gateway.md` (optional — cloud MCP)  
3. `03-deploy-agentcore-runtime.md` (optional — hosted agent)

**Skip setup** if Phase 0 health check already finds IAM, S3, KMS, Glue DBs, LF-Tags, MWAA.

---

## 3. End-to-end user journey (happy path)

```text
0. Env ready (MCP + AWS)
1. Check if dataset already exists
2. Start onboard (prompt or /onboard-workflow)
3. Answer discovery questions (Phase 1) — DO NOT SKIP
4. Approve plan (Phase 2)
5. Confirm profiling / PII report (Phase 3)
6. Review generated artifacts + tests (Phase 4)
7. Approve deploy (Phase 5)
8. Optional: E2E test on AWS
9. Optional: DevOps / IaC / monitoring
10. Commit workload to Git → promote via your CI/CD
```

Time (docs): ~30 min sequential, ~15–20 min with `/onboard-workflow`.

---

## 4. Step-by-step: onboard a new dataset

### Step A — Route / check existing

```text
Check if data from [DESCRIPTION] has already been onboarded.

Source details:
- Location: [S3_PATH or DATABASE.TABLE]
- Format: [CSV/JSON/Parquet]
- Description: [brief]

Report: existing workload status or confirm new data.
```

| Result | What you do |
|--------|-------------|
| Found | Reuse / modify that `workloads/<name>/` |
| Partial | Complete remaining zones or restart |
| Not found | Continue to Step B |

### Step B — Start onboarding

**Mode 1 — Sequential (simpler, cheaper):** paste an onboard prompt (template below).

**Mode 2 — Parallel Dynamic Workflow (faster, more tokens):**

```text
/onboard-workflow HIPAA

Onboard claims data from s3://... into Silver with dedup on claim_id ...
Run daily at 03:00 UTC. Apply HIPAA controls ...
```

Replace `HIPAA` with `GDPR` / `CCPA` / `SOX` / `PCI` / or omit if none.

### Step C — Answer discovery questions (critical)

The agent will auto-profile what it can, then ask. Prepare answers for:

#### Always (all zones)

| Topic | You must decide |
|-------|-----------------|
| PII / PHI / PCI columns | Confirm — do not let names alone decide |
| Compliance | GDPR / CCPA / HIPAA / SOX / PCI / none |
| Quality thresholds | Exact numbers **or** say “use defaults” |
| Schedule | Cron + failure handling (retries, alerts) |
| Ontology enrichment | Opt-in or opt-out (agent must ask) |

#### Bronze

- Source path / credentials pattern (Secrets Manager or Airflow Connection — not plaintext)  
- Batch vs streaming; retention for raw  

#### Silver

- Primary key  
- Dedup strategy (e.g. keep latest by timestamp)  
- Null handling (drop PK nulls only? quarantine?)  
- Transforms: masking, casts, **derived columns / business logic** (always asked)  
- Incremental vs full refresh  

#### Gold

- Business outcome / consumers  
- KPIs and aggregation grain  
- Schema: star vs flat denormalized vs API-oriented  
- BI tool / freshness SLA  

**Rule:** If you have not answered these, the agent should **not** generate pipeline code (`CLAUDE.md` human gate).

### Step D — Approve plan & metadata

- Phase 2: approve “no duplicate / proceed” plan.  
- Phase 3: confirm profiling (PK, PII flags, nulls). Correct mistakes here — cheaper than fixing Prod.

### Step E — Review artifacts

Expect under `workloads/<name>/`:

| Path | Contents |
|------|----------|
| `config/` | `source.yaml`, transforms, quality, schedule (+ semantic/ontology if opted in) |
| `scripts/` | extract / transform / quality |
| `dags/` | Airflow DAG |
| `sql/` | zone SQL |
| `tests/` | unit + integration |
| `logs/` | traces |

Run tests locally when possible:

```bash
cd workloads/<name>
pytest tests/ -v
```

### Step F — Deploy & verify

Approve Phase 5 deploy. Agent should:

- Upload / register via MCP  
- Apply LF-Tags where required  
- Run post-deployment verification (7 checks)  

Then you will be offered:

1. **E2E pipeline test** on AWS (Bronze→Silver→Gold→Athena)  
2. **DevOps Agent** (IaC, monitoring, runbook) — review IaC; **you** apply  

### Step G — Promote

```text
Dev generate → git commit → your CI/CD → QA → Staging → Prod
```

Production runs Glue + MWAA only — **no agents**.

---

## 5. Copy-paste onboard prompt template

```text
Onboard new dataset: [DATASET_NAME]

Source:
- Type: [S3/Database/Kafka/Kinesis/JDBC]
- Location: [FULL_PATH or topic/stream]
- Format: [CSV/JSON/Parquet]
- Frequency: [Daily/Hourly/Streaming]
- Credentials: [Secrets Manager ARN or Airflow connection id]

Schema (or say "discover from source"):
- column1: type, description, role (measure/dimension/identifier)
- ...

Bronze:
- Keep raw format: YES
- Retention: [DAYS]

Silver:
- PK: [COLUMN]
- Dedup: [strategy + order_by]
- Null handling: [policy]
- PII masking: [COLUMNS + method]
- Extra transforms: [derived columns / calculations / none]
- Format: Apache Iceberg

Gold:
- Use case: [Reporting/Analytics/ML/API]
- Format: [Star Schema / Flat / Iceberg+DynamoDB]
- KPIs / grain: [...]
- Quality threshold: 95%

Quality:
- Completeness: [...]
- Uniqueness: [...]
- Other rules: [...]
- Or: use defaults

Compliance: [GDPR|CCPA|HIPAA|SOX|PCI|none]
Ontology enrichment: [yes — use cases / consumers | no]

Schedule:
- Cron: [expression]
- Retries: [n]
- On failure: [alert channel]

Build complete pipeline with tests.
```

More examples: repo `README.md` (batch S3, Kafka, Kinesis) and `docs/getting-started.md`.

---

## 6. How to use *without* onboarding something new

### Explore sample workloads

| Folder | Use for |
|--------|---------|
| `workloads/claims/` | HIPAA claims example (transforms + DAG + configs) |
| `workloads/claims_v2/` | Newer config shape + ontology TTL |
| `workloads/customer_master/` | Example with unit tests + ontology |

Example local run (claims):

```bash
python3 workloads/claims/scripts/transform/bronze_to_silver_claims.py \
  --local --bronze_path demo/sample_data/claims.csv \
  --silver_path /tmp/data-lake/silver/claims/claims.parquet
```

### Modify an existing workload

1. Route check → FOUND  
2. Say what to change (new Gold KPI, stricter quality, schedule)  
3. Prefer changing **config YAML / specs** and re-render — do **not** hand-edit generated scripts (drift validator).  

### Query / operate after deploy

- Athena on Silver/Gold Iceberg tables  
- MWAA UI for DAG runs  
- CloudWatch / SNS if DevOps artifacts were applied  

---

## 7. Two ways to run the agent

| | Sequential | `/onboard-workflow` |
|--|------------|---------------------|
| How | Paste prompt | Prefix with `/onboard-workflow [REG]` |
| Speed | Slower | Faster (parallel Phase 4) |
| Cost | Lower | ~3–5× tokens |
| Best for | Learning, small jobs | Compliance-heavy / multi-agent builds |

Skill definition: `.claude/commands/onboard-workflow.md`.

---

## 8. What you should prepare before a live onboard (checklist)

- [ ] Dev AWS account + region decided  
- [ ] Environment setup done (or Phase 0 green)  
- [ ] Source accessible (S3 path / topic / JDBC)  
- [ ] Credential pattern ready (Secrets Manager / Airflow conn) — no secrets in chat long-term  
- [ ] PK + dedup policy known  
- [ ] PII columns list + regulation known  
- [ ] Quality thresholds or “use defaults”  
- [ ] Cron schedule  
- [ ] Gold use case + schema choice  
- [ ] Ontology yes/no  

---

## 9. After onboarding — day-2 operations

| Task | How |
|------|-----|
| Re-run pipeline | Trigger DAG in MWAA / Airflow |
| Fix quality failures | Quarantine + fix rules/source; do not bypass gates |
| Change transforms | Update `config/` → regenerate via agent/renderer |
| Add monitoring | `/devops-workflow <workload> terraform` (review then apply) |
| Learn from failures | Prompt Intelligence on `logs/trace_events.jsonl` |

---

## 10. Official repo docs (deeper)

| Doc | Use when |
|-----|----------|
| `docs/getting-started.md` | Prompt cookbook |
| `prompts/data-onboarding-agent/README.md` | Phase map for onboarding agent |
| `prompts/data-onboarding-agent/regulation/` | Compliance-specific controls |
| `docs/workflow-diagrams.md` | Visual phase/DAG flows |
| `docs/mcp-setup.md` | MCP local vs gateway |
| `docs/aws-account-setup.md` | AWS prerequisites |
| `CLAUDE.md` | Non-negotiable human gate + security |

---

## 11. Answer to “are the details captured?”

| Topic | Captured in Indrajit before this file? | Now |
|-------|----------------------------------------|-----|
| Claims / marketing truth | Yes — `01_` | Yes |
| E2E system flow | Yes — `02_` | Yes |
| Pitfalls | Yes — `03_` | Yes |
| Architecture | Yes — `04_` | Yes |
| **User how-to: setup → prompt → answers → deploy → promote** | **No (only fragments)** | **Yes — this file** |

If you want this pushed to GitHub, say so and we can commit/push `05_USER_ONBOARDING_GUIDE.md`.
