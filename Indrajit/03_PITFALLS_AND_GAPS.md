# 03 — Pitfalls, Gaps & Risks

Practical risks when adopting or demoing this platform. Ordered by impact.

---

## 1. Design vs implementation gap

`CLAUDE.md` still says:

```text
status: design-complete, implementation pending
```

**Pitfall:** Treating the README marketing as “turnkey production platform” underestimates remaining glue: AWS account prep, MCP wiring, human gates, IaC apply, CI/CD promote.

**Mitigation:** Demo with sample workloads (`claims`, `claims_v2`, `customer_master`); treat greenfield onboard as a guided project, not a one-click product.

---

## 2. Human-in-the-loop is non-optional

Agents **must not** invent:

- Dedup strategy from column names  
- Null handling from profiling alone  
- Quality thresholds  
- Schedules from “source frequency”  
- PII columns from names alone  
- Transformation / ontology preferences  

**Pitfall:** Users who say “don’t ask questions, use defaults” get pipelines that look complete but encode wrong business rules → bad Gold KPIs and compliance gaps.

**Mitigation:** Budget time for Phase 1 answers; treat discovery checklist as a contract.

---

## 3. CI/CD promotion claim vs reality

Deck says artifacts promote via CI/CD to QA/Staging/Prod.

**Reality in-repo:**

- Operating model is documented and sound.  
- DevOps IaC **generator** exists; **apply** is deliberately human.  
- Auto CI/CD promote + self-healing called out as **planned** (Q2–Q3 2026 in places).  
- GitHub Actions present lean toward security scanning, not full multi-env deploy.

**Pitfall:** Leadership assumes “merge = Prod” already works end-to-end.

**Mitigation:** Say “promotion **path** is git + CI/CD; sample automates generation more than multi-env release.”

---

## 4. DevOps agent maturity inconsistency

Docs disagree:

| Source | Message |
|--------|---------|
| README Environment Model | DevOps planned Q2 2026 |
| README §7 `/devops-workflow` | Available now |
| `prompts/devops-agent/README.md` | Partial — IaC yes; CI/CD/monitoring planned |
| `prompts/README.md` | Design phase Q2–Q3 2026 |

**Pitfall:** Expecting full monitoring + auto-deploy from DevOps Agent today.

---

## 5. MCP / environment dependency

Phase 0 blocks if REQUIRED MCP servers are down: `glue-athena`, `lakeformation`, `iam`.

**Pitfalls:**

- Local mode (13 stdio servers on laptop) is heavy; some servers noted as slow startup.  
- Gateway mode needs Agentcore Gateway deployed first.  
- Sub-agents cannot call MCP — if someone tries “have Transformation Agent create the Glue table,” that violates architecture.

**Mitigation:** Run Environment Setup first; keep MCP health table green before demos.

---

## 6. Deterministic codegen friction

All `workloads/*/scripts|dags|sql` **must** go through `shared.codegen.renderer`. Hooks block free-form Write/Edit.

**Pitfalls:**

- Hand-editing generated Glue scripts causes **drift validator** failures.  
- Missing template slot → `MissingSlotError`; fixing by hacking templates without schema bump is forbidden.  
- Extending behavior requires schema + template version discipline.

**Mitigation:** Change specs/YAML, re-render; don’t patch generated files for “quick fixes.”

---

## 7. Sample workload incompleteness

| Workload | Observation |
|----------|-------------|
| `claims` | Strong transform + config + DAG; fewer formal tests than docs advertise (“50+ tests”) |
| `claims_v2` | Richer config (bronze/silver/gold/ontology); more complete script set |
| `customer_master` | Includes unit tests + ontology TTL |

**Pitfall:** Assuming every workload folder is a complete reference implementation of every phase artifact.

---

## 8. Compliance is prompt-driven, not magically enforced

Regulation folders (`gdpr`, `hipaa`, `pci`, …) guide generation. Runtime enforcement still depends on:

- Correct PII column confirmation  
- LF-Tags + TBAC grants  
- Masking/hashing in Silver transforms  
- Retention / erasure hooks actually wired  

**Pitfall:** “We ran `/onboard-workflow HIPAA`” ≠ “auditors signed off.”

---

## 9. Bronze immutability & quality gate bypass

Security rules: never mutate Bronze; never skip quality gates.

**Pitfall:** Ops pressure to “just load Gold” after a failed Silver gate creates shadow pipelines outside ADOP invariants.

---

## 10. Secrets & infra leakage

Rules ban hardcoded secrets, account IDs, bucket names in source.

**Pitfall:** Demos often paste real `s3://…` paths and ARNs into prompts/configs → accidental commit.

**Mitigation:** Secrets Manager / Airflow Connections / env placeholders only.

---

## 11. Cost & token burn

Dynamic Workflow is faster but **3–5× token cost**. Opus for orchestration + compliance builds is expensive.

**Pitfall:** Parallel onboard of many datasets without cost guardrails.

---

## 12. Semantic layer boundary

ADOP emits OWL/R2RML **staged locally**. It does **not** own NL→SQL, SHACL, VKG publish (AWS Semantic Layer future).

**Pitfall:** Promising “chat with your lake” as if ADOP alone delivers it.

---

## 13. Multi-account complexity

Default is single-account. Multi-account is opt-in with `sts:AssumeRole` + catalog vs compute split.

**Pitfall:** Enterprise orgs assume multi-account TBAC is automatic.

---

## 14. Agent non-determinism vs deterministic artifacts

Even with template codegen, **spec content** still depends on LLM judgment (column roles, rule wording) unless humans constrain Phase 1 tightly.

**Pitfall:** Two runs of the same prompt → different quality rules if discovery answers differ or defaults fill gaps.

**Mitigation:** Save approved YAMLs; re-render from frozen specs for reproducibility.

---

## 15. Logging / tracing discipline

Every workload needs `logs/` + `AgentTracer` + `StructuredLogger`. Easy to skip in rushed demos → Prompt Intelligence and audit trail degrade.

---

## Practical “gotchas” checklist before a live demo

- [ ] AWS account + Environment Setup done  
- [ ] REQUIRED MCP servers healthy  
- [ ] Sample data path reachable (or use `demo/sample_data`)  
- [ ] Phase 1 answers prepared (PK, PII, schedule, thresholds)  
- [ ] Expect human approvals at plan + metadata + final artifacts  
- [ ] Don’t promise auto-Prod deploy unless your org wired CI/CD  
- [ ] Prefer `claims` / `customer_master` for walkthrough of existing artifacts  

---

## Strengths (keep in the analysis)

- Clear Dev-only agent boundary (security-friendly)  
- Medallion + Iceberg + LF-Tags is a solid lakehouse pattern  
- Test gates after each sub-agent reduce silent bad codegen  
- Spec → template → hash/drift is serious engineering for reproducibility  
- Regulation prompts encode controls as first-class inputs  
- Explicit refusal to guess business rules (painful but correct)

---

## Weaknesses summary

| Theme | Weakness |
|-------|----------|
| Maturity | Design-complete; uneven implementation depth |
| Ops | CI/CD & full DevOps still maturing |
| UX | Heavy human gate + MCP/AWS prerequisites |
| Docs | Some contradictions (DevOps dates / availability) |
| Scope | Semantic publish & NL→SQL outside ADOP |
| Cost | Parallel workflows are token-heavy |
