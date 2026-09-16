# 07 — End-to-End Plan: Who Runs What, Who Pays, Where to Configure

**Honest status:** Earlier Indrajit docs covered *what ADOP is*, *Spark vs ADOP*, and *prompts*.  
They did **not** fully document **who processes**, **who pays**, and **where/how to configure** as one end-to-end plan. **This file is that plan.**

---

## 0. Two clocks (read this first)

| Clock | What runs | Who “processes” | Who pays |
|-------|-----------|-----------------|----------|
| **Build time** (create/change a pipeline) | **Claude** (via Claude Code / optionally Bedrock) + MCP talking to AWS | Your laptop/session + AWS APIs for profile/deploy | **(A) LLM / Claude bill** + small Dev AWS API cost |
| **Run time** (daily jobs) | **Your Spark / Glue / Airflow** — **not Claude** | Your existing framework / AWS compute | **(B) AWS (or your cluster) compute bill only** |

If you only execute existing jobs → you pay **(B)** only. Claude is idle.

---

## 1. Who processes? (plain English)

### Build time — AI generates the automation

```text
You paste a prompt in Claude Code
        ↓
Claude (LLM) reasons, asks questions, writes configs/scripts/DAG
        ↓
MCP tools call AWS (list S3, Glue crawl, Athena sample, deploy…)
        ↓
Files land in workloads/<name>/  (+ optional Dev deploy)
```

| Component | Role |
|-----------|------|
| **You** | Describe need, answer PK/PII/schedule, approve |
| **Claude Code** | Host that runs the Data Onboarding Agent + sub-agents |
| **Claude models** (Opus/Sonnet) | Generate plans, YAML, Spark/Glue-style code, tests |
| **MCP servers** (`.mcp.json`) | Bridge from Claude → AWS APIs |
| **AWS Dev** | Store samples, register tables, run crawlers during onboard |

**Claude does not process your CSV/EBCDIC rows into Iceberg at scale.**  
It **writes the jobs** that later process those rows.

### Run time — your framework processes data

```text
File lands on S3
        ↓
Your scheduler (Airflow / Control-M / MWAA) triggers Spark/Glue
        ↓
Data → Iceberg / Hive / Snowflake
        ↓
No Claude. No ADOP agent. No prompt.
```

| Component | Role |
|-----------|------|
| **Your Spark framework** | Reads files, transforms, writes Iceberg/Hive/SF |
| **Scheduler** | Executes on cron |
| **AWS S3 / Glue / compute** (or on-prem Spark) | Actual data processing |

---

## 2. Who pays the price?

There are **separate bills**. Do not mix them.

### Bill A — AI / Claude (only when someone is onboarding or changing a pipeline)

| How you run Claude | Who pays | Where it shows up |
|--------------------|----------|-------------------|
| **Claude Code with Anthropic subscription / API** | Your org / you (Anthropic) | Anthropic / Claude billing |
| **Claude via Amazon Bedrock** (if configured that way) | Your AWS account | AWS bill → Bedrock model usage |
| **Cursor / other IDE** using Claude | Whoever owns that product’s API key / plan | That vendor’s bill |

**Rough cost from this repo’s README (estimates only):**

| Item | Estimate |
|------|----------|
| Tokens per full onboard | ~135K |
| ~All Opus | ~$3–4 per workload |
| Opus orchestrator + Sonnet sub-agents | ~$1–2 typical ballpark (README ~$0.73–$3.65 depending on mix) |
| `/onboard-workflow` parallel | **~3–5×** more tokens → higher Bill A |
| Daily execute with no onboard | **$0 on Bill A** |

> These figures **exclude** Glue/S3/Athena. They are LLM-only estimates from the sample README.

### Bill B — Data platform (when jobs run + when agents touch AWS)

| Activity | Who pays | Examples |
|----------|----------|----------|
| Daily Spark/Glue on your data | **Your AWS / data platform budget** | Glue DPU, EMR, S3, Athena, Snowflake credits |
| Phase 0–5 MCP calls in Dev | Same AWS Dev account | S3 list, Glue crawler, Athena sample queries |
| MWAA / Airflow | AWS or your Airflow host | Environment hours |

### Simple rule for your team

| Action | Bill A (Claude) | Bill B (AWS / Spark) |
|--------|-----------------|----------------------|
| Run existing pipeline | No | Yes |
| Onboard new CSV/EBCDIC with ADOP | Yes | Yes (Dev APIs + later job runs) |
| Only draft configs, never deploy | Yes | Little / none |
| Ignore ADOP forever | No | Yes (as today) |

**Who “pays” organizationally:** usually **Data/Platform** owns Bill B; **whoever owns Claude Code / Bedrock access** owns Bill A (often same platform team, sometimes AI CoE). Decide that up front.

---

## 3. Where to configure & how

### A. Claude / LLM access (Bill A)

| What | Where | What to set |
|------|-------|-------------|
| Claude Code install | Developer laptop / Dev workstation | Install Claude Code CLI; log in to Anthropic (or org SSO if provided) |
| Model choice for workflows | `.claude/commands/onboard-workflow.md` / workflow settings | Opus vs Sonnet routing (compliance → stronger models) |
| API key (if not CLI subscription) | Environment / Claude Code settings — **never commit keys to git** | `ANTHROPIC_API_KEY` or vendor-specific secret store |
| Bedrock path (optional) | AWS account + Bedrock model access | Enable Claude models in Bedrock; IAM for invoke; used more for structured tool_choice patterns in this repo |

**This repo does not put your Anthropic key in `.mcp.json`.** MCP is for **AWS tools**, not for paying Claude.

### B. AWS access for agents (Bill B – Dev)

| What | Where | What to set |
|------|-------|-------------|
| AWS credentials | `~/.aws/credentials` or env `AWS_ACCESS_KEY_ID` / role | Dev account only for agents |
| AWS profile + region for MCP | **Repo root `.mcp.json`** | `AWS_PROFILE`, `AWS_REGION` on each server (default `default` / `us-east-1`) |
| Switch region | Edit `.mcp.json` or `sed` as in `docs/mcp-setup.md` | e.g. `ap-south-1` |
| MCP local vs Gateway | `.mcp.json` vs `.mcp.gateway.json` | Local = laptop stdio; Gateway = AgentCore in AWS |
| Verify MCP | Terminal: `claude mcp list` | All REQUIRED servers up |

### C. Platform infra (one-time)

| What | Where | How |
|------|-------|-----|
| IAM, S3 lake, KMS, Glue DBs, LF-Tags, MWAA | AWS account via Environment Setup prompts | `prompts/environment-setup-agent/` |
| Detailed checklist | `docs/aws-account-setup.md` | Follow once per account |
| Airflow variables | MWAA / Airflow UI | Refs only — no secrets in plain text |

### D. Per-dataset pipeline config (after generate)

| What | Where |
|------|-------|
| Source path, schema, PII | `workloads/<name>/config/source.yaml` |
| Dedup, transforms | `workloads/<name>/config/transformations.yaml` (or silver/gold yaml in newer samples) |
| Quality rules | `workloads/<name>/config/quality_rules.yaml` |
| Schedule | `workloads/<name>/config/schedule.yaml` |
| DAG | `workloads/<name>/dags/` |
| Your framework adapt | **Your** Spark repo — map generated ideas into your job templates |

Prefer changing YAML/specs and re-generating over hand-editing rendered scripts (drift rules).

### E. Secrets (never in prompts long-term)

| Secret | Where |
|--------|-------|
| DB passwords, API keys | AWS Secrets Manager / Airflow Connections |
| Claude API key | Local env / org secret manager — not in `workloads/` |
| Bucket names / account IDs | Prefer config/env — avoid hardcoding in committed scripts |

---

## 4. End-to-end plan (your scenario: framework exists, Iceberg OK)

Use this as the **project plan**, not only a tech demo.

### Phase P0 — Decide ownership & money (1 meeting)

- [ ] Who owns **Bill A** (Claude/Bedrock)?  
- [ ] Who owns **Bill B** (AWS/Spark)?  
- [ ] Agents allowed **only in Dev** account? (recommended — matches ADOP design)  
- [ ] Will ADOP **replace** codegen or only **draft** into your Spark framework? → for you: **draft + adapt**  

### Phase P1 — Access & configure (1–2 days)

- [ ] Install Claude Code; confirm login / billing works (small test prompt).  
- [ ] Configure AWS Dev profile; `aws sts get-caller-identity`.  
- [ ] Install `uv`; open repo; set `.mcp.json` region/profile.  
- [ ] `claude mcp list` — fix REQUIRED servers (`glue-athena`, `lakeformation`/`iam` as applicable).  
- [ ] Run Environment Setup prompt **once** if lake not ready.  

### Phase P2 — Pilot (1 new source, low risk)

- [ ] Pick one S3 CSV (or simple EBCDIC with known copybook).  
- [ ] Paste onboard prompt (see `06_...` Part 4/5); answer questions; approve.  
- [ ] Review `workloads/<pilot>/`.  
- [ ] **Adapt** generated transforms/DQ/schedule into **your** Spark + Iceberg framework.  
- [ ] Execute with **your** scheduler on a Dev path.  
- [ ] Measure: hours saved vs hand-write; Bill A $ for that onboard; Bill B for test run.  

### Phase P3 — Operating model

- [ ] New source → optional ADOP draft → human review → merge to your framework → execute.  
- [ ] No new source → **do not** run Claude; only execute.  
- [ ] Rule changes → either edit your configs or re-ask ADOP, then execute.  

### Phase P4 — Optional advanced (later)

- [ ] Compliance prompt packs if regulated.  
- [ ] Ontology only if semantic layer is a goal.  
- [ ] DevOps IaC/alarms if you want generated ops docs.  
- [ ] Prompt Intelligence if you keep ADOP traces.  

**Do not** block the pilot on Phase P4.

### Phase P5 — What success looks like

| Success | Not success |
|---------|-------------|
| New source drafted in hours, runs on your Iceberg path | Expecting Claude to replace nightly Spark |
| Clear Bill A vs Bill B ownership | Surprise Bedrock/Claude charges on execute days |
| Config known: `.mcp.json` + AWS Dev + Claude login | Keys committed to git |
| Team uses the 3 benefits only | Buying “AI runs Prod” story |

---

## 5. Configuration map (one page)

```text
┌─────────────────────────────────────────────────────────────┐
│ YOU / ORG                                                   │
│  • Claude Code login / API key     → Bill A                 │
│  • Decide Dev-only agents                                   │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│ THIS REPO                                                   │
│  • .mcp.json                 → AWS region, profile, MCP     │
│  • prompts/                  → what to paste                │
│  • .claude/commands/         → /onboard-workflow            │
│  • workloads/<name>/config/  → per-dataset rules            │
└──────────────────────────┬──────────────────────────────────┘
                           │ MCP (build time)
┌──────────────────────────▼──────────────────────────────────┐
│ AWS DEV                                                     │
│  • Credentials ~/.aws        → Bill B (Dev)                 │
│  • S3 / Glue / Athena / LF / KMS / MWAA                     │
└──────────────────────────┬──────────────────────────────────┘
                           │ after you adapt & promote
┌──────────────────────────▼──────────────────────────────────┐
│ YOUR FRAMEWORK (execute)                                    │
│  • Spark jobs + Iceberg (+ Hive/SF if needed)               │
│  • Your scheduler            → Bill B (Prod/compute)        │
│  • No Claude                                            │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. FAQ

**Q: Does Claude process my EBCDIC file?**  
A: No. Claude helps **create** the job. Spark processes the file.

**Q: Do we pay Claude every night?**  
A: No — only when someone runs the agent to create/change a pipeline.

**Q: Where do I put the Anthropic key?**  
A: Claude Code login or local/org secret env — **not** in `.mcp.json`, **not** in `workloads/`.

**Q: Where do I point to our S3 and region?**  
A: Tell the agent in the prompt (S3 path); set region/profile in `.mcp.json` for MCP; dataset details end up in `workloads/<name>/config/`.

**Q: We already have the framework — do we still configure all of this?**  
A: Only if you want ADOP **drafting**. For execute-only days: no Claude config needed. For a pilot: configure Claude + Dev AWS MCP once.

**Q: Was this documented before?**  
A: Partially scattered in README cost section + `docs/mcp-setup.md`. **Not** as one E2E “who pays / where configure / plan” until this file (`07_`).

---

## 7. Related Indrajit docs

| Doc | Gap it covers |
|-----|----------------|
| `05_USER_ONBOARDING_GUIDE.md` | How to prompt / steps |
| `06_SPARK_HIVE_SF_COMPARISON_AND_PROMPTS.md` | Your stack + 3 benefits + sample prompts |
| **`07_` (this file)** | **Who processes, who pays, where configure, E2E plan** |
| `02_E2E_FLOW.md` | Technical phase flow |
| `03_PITFALLS_AND_GAPS.md` | Risks |

### Official repo pointers

- Cost estimates: `README.md` → “Cost Estimation (LLM Token Usage)”  
- MCP configure: `docs/mcp-setup.md`  
- AWS setup: `docs/aws-account-setup.md`  
- Env agent: `prompts/environment-setup-agent/`  
