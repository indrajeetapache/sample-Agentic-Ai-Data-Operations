# 04 — Architecture Deep Dive

## System layers

```
┌─────────────────────────────────────────────────────────┐
│  Interaction: Claude Code (+ optional Bedrock tool use) │
├─────────────────────────────────────────────────────────┤
│  Prompts & skills: prompts/, SKILLS.md, .claude/        │
├─────────────────────────────────────────────────────────┤
│  Orchestration: Data Onboarding Agent (main chat)       │
│  Sub-agents: Metadata, Transform, Quality, DAG, …       │
├─────────────────────────────────────────────────────────┤
│  Contracts & codegen: contracts/v1 + shared/codegen     │
│  Templates: shared/templates/*.j2                       │
├─────────────────────────────────────────────────────────┤
│  Workload artifacts: workloads/{name}/                  │
├─────────────────────────────────────────────────────────┤
│  Tooling: MCP (13 servers) + Decision Engine routing    │
├─────────────────────────────────────────────────────────┤
│  AWS: S3/Iceberg, Glue, Athena, LF, KMS, MWAA           │
└─────────────────────────────────────────────────────────┘
```

## Agent authorization model

| Actor | Can talk to human | Can write workload files | Can call MCP / AWS |
|-------|-------------------|--------------------------|--------------------|
| Data Onboarding (main) | Yes | Yes (orchestrate) | **Yes** |
| Sub-agents | No (return structured output) | Yes | **No** |
| DevOps (IaC gen) | Via workflow | Yes (`iac/`) | Apply is human |

Enforced by invariants (`tool-registry/invariants.yaml`) e.g. `sub-agent-no-mcp`, `mcp-first`, `bronze-immutable`, `quality-gates`, `no-credentials-in-code`.

## Decision Engine (tool routing)

5 steps:

1. Context — if sub-agent → stop (files only)  
2. Server health — REQUIRED block / WARN fallback / OPTIONAL defer  
3. Intent match — `TOOL_ROUTING.md`  
4. Fallback — MCP → CLI with warning  
5. Invariants — always on  

Canonical list: `tool-registry/servers.yaml`.

## Deterministic codegen chain

```
YAML spec → load_spec (JSON Schema) → spec_hash (SHA-256)
         → Jinja2 StrictUndefined render
         → 5-line header (spec_hash, template_id, template_hash, schema_version, rendered_at)
         → atomic write + ADOP_RENDERER_TOKEN
         → drift_validator in CI / Step 4.5.2
```

Templates include: `bronze_ingestion.py.j2`, `silver_transform.py.j2`, `gold_aggregate.py.j2`, `quality_check.py.j2`, `airflow_dag.py.j2`, `iceberg_ddl.sql.j2`, …

## Data zones (operational meaning)

| Zone | What you store | What you allow |
|------|----------------|----------------|
| Bronze | Exact landing copy | Read + append partitions; **no mutate** |
| Silver | Conformed entities | Upserts/dedup; PII treated; Iceberg time-travel |
| Gold | Business grain / KPIs | Aggregates, star or flat; highest quality bar |

Gold schema choice (Phase 1): Star (BI), Flat Iceberg (analytics/ML), Iceberg + DynamoDB (API).

## Security controls (platform intent)

1. Secrets via Secrets Manager / Airflow Connections / env — never in code  
2. No account/VPC/bucket hardcoding in source  
3. AES-256 (KMS) at rest; TLS in transit  
4. Mandatory PII detection utility + LF-Tags + TBAC  
5. Quality gates block promotion  
6. Least-privilege IAM  
7. Audit: who/what/when/where + CloudTrail post-deploy check  

## Logging & learning loop

- `AgentTracer` → `workloads/{name}/logs/trace_events.jsonl`  
- Sub-agent `decisions[]` in structured `AgentOutput`  
- `StructuredLogger` in ETL scripts  
- Prompt Intelligence mines traces → recommend prompt patches  

## What ADOP owns vs what it hands off

| Owns | Hands off / future |
|------|--------------------|
| Pipeline generation + local ontology staging | AWS Semantic Layer NL→SQL / SHACL / VKG |
| Dev MCP deploy helpers | Full multi-env GitOps (maturing) |
| IaC **generation** | IaC **apply** (human) |
| Quality rule generation | Org-specific DQ ownership & sign-off |

## Key entry files for further reading

| File | Why |
|------|-----|
| `README.md` | Product narrative + environment model |
| `CLAUDE.md` | Hard gates + security rules for agents |
| `docs/workflow-diagrams.md` | Mermaid phase / DAG diagrams |
| `docs/determinism.md` | Spec → artifact reproducibility |
| `SKILLS.md` | Per-agent skill definitions |
| `MCP_GUARDRAILS.md` | Per-phase allowed tools |
| `prompts/data-onboarding-agent/` | Runnable prompt pack |
| `workloads/claims*` / `customer_master` | Concrete examples |

## Repo status snapshot (this clone, 2026-09-15)

- Strong: architecture docs, contracts, templates, sample workloads, regulation prompts, guardrails.  
- Partial: DevOps/CI-CD automation, semantic publish, some prompt paths still “coming soon.”  
- Treat as: **reference architecture + working samples + agent playbooks**, not a fully closed SaaS.
