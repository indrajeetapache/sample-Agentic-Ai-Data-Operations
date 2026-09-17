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

### B. How to configure **agents** in this open-source repo (file-by-file)

There is **no separate “agent UI” or `agents.yaml` with a start button**.  
Agents are configured by editing **markdown skills/commands**, **Claude Code settings/hooks**, **MCP JSON**, and **policy/registry YAML**. Claude Code reads those files when you open this repo.

#### Big picture — what controls what

```text
You open this repo in Claude Code
        │
        ├─ CLAUDE.md + .claude/rules/*     → hard rules (human gate, security)
        ├─ SKILLS.md                       → what each agent is allowed to do / how it behaves
        ├─ prompts/**                      → copy-paste playbooks (env setup, onboard, regulation)
        ├─ .claude/commands/*.md           → slash commands (/onboard-workflow, /devops-workflow)
        │                                    ★ THIS is where model = haiku/sonnet/opus is set
        ├─ .claude/settings.json + hooks/  → gates (discovery, codegen, logging)
        ├─ .mcp.json                       → which AWS tools agents can call
        ├─ tool-registry/* + TOOL_ROUTING.md → which tool for which intent
        ├─ shared/policies/**/*.cedar      → authorization / guardrails (Cedar)
        └─ workloads/<name>/config/*       → per-dataset output (after generation)
```

#### 1) Configure which agents exist and how they behave

| Goal | File(s) | How |
|------|---------|-----|
| Master agent playbooks (Router, Onboarding, Metadata, Transform, Quality, DAG, …) | **`SKILLS.md`** | Edit the skill section for that agent (phases, spawn prompts, constraints). This is the main “agent brain” text. |
| Project-wide non-negotiables | **`CLAUDE.md`** | Human-in-the-loop gate, security, zones, codegen rules. Change only if you know impact. |
| Zone discovery questions | **`.claude/rules/00-zone-questions.md`** | What the agent must ask for Bronze/Silver/Gold. |
| Architecture / Python / SQL / quality / logging conventions | **`.claude/rules/02-*.md` … `09-*.md`** | Path-scoped rules Claude Code loads automatically. |
| Runnable prompt packs (setup, onboard, govern, regulation) | **`prompts/environment-setup-agent/`**, **`prompts/data-onboarding-agent/`**, **`prompts/devops-agent/`**, **`prompts/data-onboarding-agent/regulation/`** | Edit or add markdown prompts users paste; regulation files change compliance behavior. |

**Practical tip:** To change “what Metadata Agent does,” edit its section in `SKILLS.md` and keep matching spawn text in `.claude/commands/onboard-workflow.md` / onboarding prompts in sync.

#### 2) Configure models (haiku / sonnet / opus) and parallel workflow

| Goal | File | How |
|------|------|-----|
| `/onboard-workflow` model routing | **`.claude/commands/onboard-workflow.md`** | Search for `model: 'haiku'|'sonnet'|'opus'` and `REGULATIONS_REQUIRING_OPUS` / `getBuildModel()`. Example: HIPAA/SOX/PCI → Opus for build; else Sonnet. Adversarial/verify stay Opus. |
| `/devops-workflow` models | **`.claude/commands/devops-workflow.md`** | Same pattern — health=haiku, generate=sonnet, security review=opus. |
| Force cheaper models for all builds | Edit those command files | Change `opus` → `sonnet` (or `haiku` for checks only). **Tradeoff:** lower Bill A, weaker compliance review. |
| Slash command tool permissions | Frontmatter of same `.md` files | `allowed-tools: ...` at top of command file. |

Example (concept) inside `onboard-workflow.md`:

```javascript
// Build model by regulation — EDIT THIS LIST to change who gets Opus
const REGULATIONS_REQUIRING_OPUS = ['HIPAA', 'SOX', 'PCI']

function getBuildModel(regulation) {
  // return 'sonnet'  // ← uncomment-style change: force cheaper builds
  return regs.some(r => REGULATIONS_REQUIRING_OPUS.includes(r)) ? 'opus' : 'sonnet'
}

await agent(`...prompt...`, { model: 'haiku', label: 'health:...' })
await agent(`...prompt...`, { model: BUILD_MODEL, label: 'transform:...' })
```

Sequential mode (paste “Onboard …” without `/onboard-workflow`) uses the **main chat model** you selected in Claude Code — not the workflow script’s per-agent map.

#### 3) Configure Claude Code hooks / safety gates

| Goal | File | How |
|------|------|-----|
| Enable/disable hooks | **`.claude/settings.json`** | Lists PreToolUse / PostToolUse hooks. |
| Block writes until discovery answered | **`.claude/hooks/check-discovery-gate.sh`** | Enforces human gate. |
| Force template codegen (no free-form script edits) | **`.claude/hooks/enforce_template_codegen.py`** | Blocks Write/Edit under scripts/dags/sql without renderer token. |
| Require logging patterns | **`.claude/hooks/check-logging.sh`** | Logging discipline. |
| Log Q&A | **`.claude/hooks/log_conversation.py`** | Trace capture on AskUserQuestion. |

To “loosen” gates for a private fork: edit or remove hook entries in `.claude/settings.json` (not recommended for Prod-like discipline).

#### 4) Configure AWS tools the agents can call (MCP)

| Goal | File | How |
|------|------|-----|
| Wire MCP servers Claude Code loads | **`.mcp.json`** (repo root) | Set `AWS_REGION`, `AWS_PROFILE` per server; add/remove servers. |
| Cloud MCP instead of laptop | Copy/use **`.mcp.gateway.json`** → as `.mcp.json` | After AgentCore Gateway deploy. |
| Canonical server list + REQUIRED/WARN/OPTIONAL | **`tool-registry/servers.yaml`** | Keep in sync with `.mcp.json`; validate with `python scripts/validate_tool_registry.py`. |
| Hard rules (MCP-first, no secrets, bronze immutable…) | **`tool-registry/invariants.yaml`** | Change severity/rules carefully. |
| Intent → which tool | **`TOOL_ROUTING.md`** | Add/change routing phrases and `not_when` conditions. |
| Per-phase allowed tools | **`MCP_GUARDRAILS.md`** | What may be called in Phase 0–5. |
| Custom MCP server code | **`mcp-servers/*/server.py`** | Extend Glue/Athena/LF/PII tools. |

Minimal local config example (edit every server’s `env` block the same way):

```json
"env": {
  "AWS_REGION": "ap-south-1",
  "AWS_PROFILE": "adop-dev"
}
```

Then:

```bash
aws sts get-caller-identity --profile adop-dev
claude mcp list
```

#### 5) Configure agent authorization / guardrails (Cedar policies)

| Goal | File | How |
|------|------|-----|
| Who may do what (onboarding vs sub-agents) | **`shared/policies/agent_authorization/*.cedar`** | e.g. `onboarding_agent.cedar` = main conversation full access; sub-agents more limited. |
| Quality / security / immutability guards | **`shared/policies/guardrails/*.cedar`** | e.g. quality gate threshold, PII masking, bronze immutability. |
| Schema for policies | **`shared/policies/schema.cedarschema`** | When adding principals/actions. |

Sub-agents are designed for **file generation only** (no MCP) — that split is policy + skill text, not a GUI toggle.

#### 6) Configure codegen templates (what agents emit)

| Goal | File | How |
|------|------|-----|
| Spec shapes | **`contracts/v1/*.schema.json`** | Extend fields → bump carefully. |
| Jinja templates for Glue/DAG/SQL/quality | **`shared/templates/*.j2`** | Change generated Spark/Glue/Airflow shape to closer match your framework. |
| Renderer | **`shared/codegen/`** | Spec → template → artifact. |
| Account topology defaults | **`shared/templates/account_topology.yaml`** | Single vs multi-account hints. |

**For your Spark→Iceberg shop:** the highest-leverage OSS customization is often **templates + contracts**, so drafts look like your framework — not rewriting every agent skill.

#### 7) Configure a single workload (after/during onboard)

| Goal | File |
|------|------|
| Source / schema / PII | `workloads/<name>/config/source.yaml` |
| Transforms | `.../transformations.yaml` or silver/gold yaml |
| Quality | `.../quality_rules.yaml` |
| Schedule | `.../schedule.yaml` |
| Semantic / ontology (optional) | `.../semantic.yaml`, `ontology.ttl`, `mappings.ttl` |

#### 8) Step-by-step: “I forked this OSS — how do I configure agents?”

1. **Clone / open repo in Claude Code** (agents are prompt-driven here).  
2. **Bill A:** log in to Claude / set API access (not in git).  
3. **Bill B Dev:** `~/.aws` profile; edit **`.mcp.json`** region + profile.  
4. **Validate tools:** `claude mcp list` + optional `python scripts/validate_tool_registry.py`.  
5. **One-time AWS lake:** run prompts under `prompts/environment-setup-agent/`.  
6. **Tune agent behavior (optional):**  
   - Cheaper models → edit **`.claude/commands/onboard-workflow.md`**  
   - Different questions → **`.claude/rules/00-zone-questions.md`** / **`SKILLS.md`**  
   - Closer to your Spark jobs → **`shared/templates/*.j2`** + **`contracts/v1/`**  
   - Compliance defaults → **`prompts/data-onboarding-agent/regulation/`**  
7. **Run:** paste onboard prompt or `/onboard-workflow HIPAA`.  
8. **Per dataset:** review/edit `workloads/<name>/config/`, then adapt into your framework and execute.

#### 9) What you typically should *not* configure for day-1

| Avoid at first | Why |
|----------------|-----|
| Deleting human-gate hooks | Agents will invent PK/PII/schedule |
| Putting Anthropic keys in `.mcp.json` | Wrong file; security risk |
| Editing generated `scripts/` by hand | Drift validator / hooks fight you — change templates/specs |
| Pointing MCP at **Prod** AWS | Design is Dev-only agents |

#### 10) Quick reference — “I want X → edit Y”

| I want to… | Edit this |
|------------|-----------|
| Change AWS region/profile for agents | `.mcp.json` |
| Use cheaper models in parallel onboard | `.claude/commands/onboard-workflow.md` |
| Change discovery questions | `.claude/rules/00-zone-questions.md` |
| Change agent responsibilities / spawn prompts | `SKILLS.md` |
| Change HIPAA/GDPR default controls | `prompts/data-onboarding-agent/regulation/*.md` |
| Make generated Spark look like our framework | `shared/templates/*.j2` + `contracts/v1/` |
| Add/remove AWS tools | `.mcp.json` + `tool-registry/servers.yaml` |
| Change tool choice rules | `TOOL_ROUTING.md`, `MCP_GUARDRAILS.md` |
| Loosen/tighten quality gate policy | `shared/policies/guardrails/dq_*.cedar` |
| Change slash-command workflow | `.claude/commands/onboard-workflow.md` or `devops-workflow.md` |
| Turn hooks on/off | `.claude/settings.json` |

---

### C. AWS access for agents (Bill B – Dev)

| What | Where | What to set |
|------|-------|-------------|
| AWS credentials | `~/.aws/credentials` or env `AWS_ACCESS_KEY_ID` / role | Dev account only for agents |
| AWS profile + region for MCP | **Repo root `.mcp.json`** | `AWS_PROFILE`, `AWS_REGION` on each server (default `default` / `us-east-1`) |
| Switch region | Edit `.mcp.json` or `sed` as in `docs/mcp-setup.md` | e.g. `ap-south-1` |
| MCP local vs Gateway | `.mcp.json` vs `.mcp.gateway.json` | Local = laptop stdio; Gateway = AgentCore in AWS |
| Verify MCP | Terminal: `claude mcp list` | All REQUIRED servers up |

### D. Platform infra (one-time)

| What | Where | How |
|------|-------|-----|
| IAM, S3 lake, KMS, Glue DBs, LF-Tags, MWAA | AWS account via Environment Setup prompts | `prompts/environment-setup-agent/` |
| Detailed checklist | `docs/aws-account-setup.md` | Follow once per account |
| Airflow variables | MWAA / Airflow UI | Refs only — no secrets in plain text |

### E. Per-dataset pipeline config (after generate)

| What | Where |
|------|-------|
| Source path, schema, PII | `workloads/<name>/config/source.yaml` |
| Dedup, transforms | `workloads/<name>/config/transformations.yaml` (or silver/gold yaml in newer samples) |
| Quality rules | `workloads/<name>/config/quality_rules.yaml` |
| Schedule | `workloads/<name>/config/schedule.yaml` |
| DAG | `workloads/<name>/dags/` |
| Your framework adapt | **Your** Spark repo — map generated ideas into your job templates |

Prefer changing YAML/specs and re-generating over hand-editing rendered scripts (drift rules).

### F. Secrets (never in prompts long-term)

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
│ THIS REPO — AGENT CONFIG (no GUI; edit these files)         │
│  • SKILLS.md / CLAUDE.md / .claude/rules/  → agent behavior │
│  • .claude/commands/onboard-workflow.md    → models/flow    │
│  • .claude/settings.json + hooks/          → safety gates   │
│  • .mcp.json + tool-registry/              → AWS tools      │
│  • shared/templates/ + contracts/v1/       → generated code │
│  • shared/policies/**/*.cedar              → auth/guards    │
│  • prompts/**                              → paste playbooks│
│  • workloads/<name>/config/                → per dataset    │
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
│  • No Claude                                                │
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

**Q: Where do I configure the agents themselves in this open-source project?**  
A: There is no agent GUI. Edit **`SKILLS.md`** (behavior), **`.claude/commands/onboard-workflow.md`** (models/parallel flow), **`.mcp.json`** (AWS tools), **`.claude/settings.json`** (hooks), and optionally **`shared/templates/`** / **`contracts/v1/`**. Full map: Section **3.B** above.

**Q: Was this documented before?**  
A: Partially scattered in README cost section + `docs/mcp-setup.md`. **Not** as one E2E “who pays / where configure / how to configure agents” until this file (`07_`).

---

## 7. Related Indrajit docs

| Doc | Gap it covers |
|-----|----------------|
| `05_USER_ONBOARDING_GUIDE.md` | How to prompt / steps |
| `06_SPARK_HIVE_SF_COMPARISON_AND_PROMPTS.md` | Your stack + 3 benefits + sample prompts |
| **`07_` (this file)** | **Who processes, who pays, where configure, how to configure agents (files), E2E plan** |
| `02_E2E_FLOW.md` | Technical phase flow |
| `03_PITFALLS_AND_GAPS.md` | Risks |

### Official repo pointers

- Cost estimates: `README.md` → “Cost Estimation (LLM Token Usage)”  
- MCP configure: `docs/mcp-setup.md`  
- AWS setup: `docs/aws-account-setup.md`  
- Env agent: `prompts/environment-setup-agent/`  
