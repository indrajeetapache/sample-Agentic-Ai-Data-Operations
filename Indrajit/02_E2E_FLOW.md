# 02 — End-to-End Flow

There are **two timelines**. Confusing them is the #1 source of misunderstanding.

```
BUILD TIME (agents, Dev only)          RUN TIME (no agents)
─────────────────────────────────      ─────────────────────────────
Discover → Profile → Generate          MWAA/Airflow triggers Glue
→ Test → Deploy artifacts → Git        Bronze → Silver → Gold
→ (CI/CD promote)                      Quality gates + alerts
```

---

## A. Build-time flow (how a pipeline is *created*)

### Entry

User pastes a natural-language onboarding request into Claude Code (or `/onboard-workflow [REGULATION]` for parallel mode).

### Router (inline)

Search `workloads/`:

| Result | Action |
|--------|--------|
| FOUND | Point to existing folder; ask modify / query / re-run |
| PARTIAL | Report gaps; complete vs restart |
| NOT FOUND | Start Data Onboarding Agent |

### Phase 0 — Health check (read-only)

1. Scan AWS: IAM roles, S3 lake, KMS, Glue DBs, LF-Tags, MWAA, etc.  
2. MCP health across ~13 servers (REQUIRED / WARN / OPTIONAL).  
3. Gate: missing critical infra → Environment Setup Agent; REQUIRED MCP down → fix/block.

### Phase 1 — Discovery (human-in-the-loop, mandatory)

Agent **must ask** (must not invent):

- Zones (Bronze / Silver / Gold)  
- Source location, format, credentials pattern  
- PK, dedup, nulls, transforms  
- PII columns + regulation  
- Quality thresholds (or “use defaults”)  
- Schedule (cron)  
- Ontology enrichment opt-in  

Auto-profile first; ask only what wasn’t discovered.  
**Hard gate in `CLAUDE.md`:** no pipeline code until human answers are complete.

### Phase 2 — Dedup & plan

- Scan `workloads/*/config/source.yaml` for exact duplicate (block) or overlap (warn).  
- Connectivity / reuse checks against `shared/`.  
- Present plan → human approve.

### Phase 3 — Profiling (Metadata sub-agent)

- Glue Crawler + ~5% Athena sample  
- Types, nulls, distincts, PII candidates  
- Write tests → test gate  
- Present metadata report → human confirm  

### Phase 4 — Build (sub-agents + test gates)

Typical sequence (parallelizable in `/onboard-workflow`):

| Step | Agent | Outputs |
|------|-------|---------|
| 4.x | Metadata (formalize) | Schema, classifications, catalog/lineage configs |
| 4.x | Transformation | `transformations.yaml`, Bronze↔Silver↔Gold scripts, SQL |
| 4.x | Quality | `quality_rules.yaml`, check scripts |
| 4.x | DAG | Airflow DAG + schedule wiring |

Each sub-agent:

1. Emits **spec** validated by `contracts/v1/*.schema.json`  
2. Calls **deterministic renderer** (`shared.codegen.renderer`) → Jinja templates  
3. Writes unit + integration tests  
4. Orchestrator runs tests; 1 fail → retry with errors; 2 fails → escalate to human  

**Critical constraint:** sub-agents have **no MCP / no AWS**. They only write files under `workloads/{name}/`.

### Phase 5 — Deploy (main conversation + MCP)

- Upload scripts, register Glue tables, LF-Tags/TBAC, KMS, MWAA DAG load  
- `post_deployment_verifier.py` (7 checks)  
- Ask E2E pipeline test offer (Step 5.10)  
- Ask DevOps Agent offer (Step 5.11)  

### Optional enrichment

- Ontology Staging → `ontology.ttl`, `mappings.ttl`, `ontology_manifest.json` (local handoff to AWS Semantic Layer — publish is future)  
- Prompt Intelligence → learn from failure traces  

---

## B. Run-time flow (how data actually moves)

Medallion zones:

| Zone | Mutability | Format | Quality gate |
|------|------------|--------|--------------|
| Bronze | Immutable | Source format | None |
| Silver | Updatable | Iceberg on S3 Tables | ≥ 0.80 |
| Gold | Updatable | Iceberg (star / flat / API-oriented) | ≥ 0.95 |

Airflow task shape (conceptual):

```
extract → bronze
       → bronze_to_silver
       → quality_check_silver  ──fail──► quarantine + SNS alert
       → silver_to_gold
       → quality_check_gold    ──fail──► quarantine + SNS alert
       → update catalog / lineage
       → notify success
```

### Sample workload: `claims`

- HIPAA healthcare claims CSV → Silver Iceberg → Gold analytical Iceberg  
- Dedup on `claim_id`; PHI hash/mask; daily schedule  
- Local runnable transforms under `workloads/claims/scripts/transform/`  

`claims_v2` / `customer_master` show newer config shape (`bronze.yaml`, `silver.yaml`, ontology TTL, etc.).

---

## C. Promotion flow (what the deck diagram shows)

```
Dev (agents generate) → Git commit → CI/CD → QA → Staging → Production
                                              ↑
                                    humans review PRs / approvals
```

In Production: **only** Glue jobs, Iceberg tables, MWAA DAGs, LF policies — **no Claude / no Bedrock agents required for nightly ETL**.

---

## D. Modes of execution

| Mode | How | Wall clock (docs) | Cost |
|------|-----|-------------------|------|
| Sequential | Paste onboarding prompt | ~30 min | 1× |
| Dynamic Workflow | `/onboard-workflow HIPAA` | ~15–20 min | 3–5× |

Model routing (docs): compliance-critical regs → stronger models for build agents; adversarial review at chokepoints.

---

## E. File layout after a successful onboard

```
workloads/{name}/
├── config/     source, transforms, quality, schedule, semantic/ontology
├── scripts/    extract / transform / quality / load
├── dags/       Airflow DAG
├── sql/        bronze / silver / gold
├── tests/      unit + integration
├── logs/       trace_events.jsonl, run_*
└── README.md
```

Shared platform code: `shared/` (codegen, templates, utils, policies, logging).  
Prompts: `prompts/` (environment, onboarding, devops, regulations).  
Tooling: `tool-registry/`, `.mcp.json`, `TOOL_ROUTING.md`, `MCP_GUARDRAILS.md`.

---

## F. One-sentence E2E summary

**Human rules + profiled source → validated specs → rendered templates → tested workload folder → MCP deploy in Dev → git → promote artifacts → scheduled Bronze→Silver→Gold with quality gates.**
