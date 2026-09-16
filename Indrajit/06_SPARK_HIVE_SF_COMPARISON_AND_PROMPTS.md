# 06 — Your Spark/Hive/SF Framework vs ADOP  
## Plain-language comparison + how to talk to this framework (with S3 → Iceberg examples)

**Audience:** Data engineers who already have a Spark ingestion framework that lands data in Hive and/or Snowflake, and want to understand ADOP (this repo) and how to ask it for work.

---

## Part 1 — Side-by-side comparison (layman)

### What both sides are trying to do

| Step | Your current framework (typical) | ADOP (this project) |
|------|----------------------------------|---------------------|
| 1. Files arrive | CSV / EBCDIC / etc. land on **S3** (or similar) | Same idea — files already on **S3** |
| 2. Ingest | Spark job reads files | Generated **Glue / Spark-style** job reads files into **Bronze** |
| 3. Clean | Your Spark transforms | Generated **Bronze → Silver** transforms (dedup, types, mask PII) |
| 4. Serve | Write **Hive** and/or **Snowflake** | Write **Iceberg** tables on S3 (Silver/Gold). Query via Athena / Spark |
| 5. Schedule | Your Airflow / Control-M / etc. | Generated **Airflow (MWAA)** DAG |
| 6. Quality | Your DQ checks (if any) | Generated quality rules + gates (Silver ≥80%, Gold ≥95%) |
| 7. Who writes the code? | **You / your team** | **AI agents generate** it after you answer questions |

### One-line difference

- **Your framework** = the **engine** that already runs Spark and loads Hive/SF.  
- **ADOP** = an **AI co-pilot that builds a new pipeline package** (configs + jobs + DAG + tests) for you — default target is **Iceberg on AWS**, not classic Hive/SF.

### Important truths

1. ADOP does **not** replace Spark. Spark/Glue still processes data.  
2. ADOP does **not** magically plug into your existing Hive metastore or Snowflake without adaptation.  
3. If **your framework already supports Iceberg**, ADOP is a strong fit for **generating** Iceberg pipelines; you can later align generated jobs with your framework’s patterns.  
4. In **production**, only the **generated jobs** run — not the AI chat.

```text
TODAY                          ADOP
─────                          ────
You write Spark         →      You describe need in English
Spark runs              →      Agents generate Spark/Glue + DAG
Write Hive / SF         →      Write Iceberg (default)
You operate jobs        →      Same: scheduler runs jobs (no AI)
```

---

## Part 2 — How you “submit an ask” to this framework

Think of Claude Code (opened in this repo) as a **ticket desk** for pipeline creation.

### How communication works

1. You open this repo in **Claude Code** (or a similar AI coding assistant that can use the repo prompts).  
2. You paste a **prompt** (your “ticket”).  
3. The agent may **check AWS + MCP health** first.  
4. It **asks you questions** (primary key, dedup, PII, schedule, quality).  
5. You answer in plain English (or structured bullets).  
6. You **approve** the plan / profiling / final artifacts.  
7. It writes everything under `workloads/<name>/`.  
8. Optionally it **deploys to Dev AWS**; you later promote via Git/CI/CD.

### Two ways to submit

| Style | What you type | When to use |
|-------|---------------|-------------|
| **Simple ask** | Just paste “Onboard …” | Learning, one dataset, lower cost |
| **Workflow ask** | `/onboard-workflow` then your story (optional: `HIPAA` / `GDPR` / etc.) | Faster parallel build, compliance-heavy |

### What good communication looks like

Tell it, in any order, the **who / what / where / when / rules**:

| Piece | Plain English | Example |
|-------|---------------|---------|
| **What** | Dataset name + business meaning | “Customer claims files” |
| **Where** | S3 path (files already there) | `s3://my-landing/claims/incoming/` |
| **Format** | File type | CSV or EBCDIC |
| **Target** | You want Iceberg | “Silver and Gold as Iceberg” |
| **Key** | What makes a row unique | `claim_id` |
| **Clean rules** | Dedup, nulls, masks | “Keep latest by file date; mask SSN” |
| **Quality** | How strict | “Use defaults” or “completeness 95%” |
| **When** | Schedule | “Daily 2 AM UTC” |
| **Compliance** | If any | HIPAA / GDPR / none |

### What bad communication looks like

- “Onboard the S3 data” (no path, no key, no schedule).  
- Pasting passwords / secret keys in chat.  
- Saying “don’t ask questions, just do everything” → agents may invent wrong business rules.

---

## Part 3 — Step-by-step process (always the same)

Use this every time — CSV, EBCDIC, or other files on S3.

### Step 0 — One-time prep (first time only)

1. Clone repo; open in Claude Code.  
2. Have Dev AWS credentials ready.  
3. Ask once:

```text
Setup AWS environment for the Agentic Data Onboarding platform.
Region: us-east-1
Project: data-onboarding
Environment: dev
Create: IAM, S3 lake, KMS, Glue DBs, Lake Formation tags, MWAA if needed.
```

4. Wait until environment + MCP look healthy (agent’s Phase 0).

**Skip Step 0** if your AWS Dev is already set up.

---

### Step 1 — Check if already onboarded

```text
Check if claims landing data from s3://my-landing-bucket/claims/incoming/
has already been onboarded. Format: CSV. Report found / partial / not found.
```

- **Found** → reuse that workload; ask to modify.  
- **Not found** → go to Step 2.

---

### Step 2 — Submit the main ask (your “ticket”)

Paste a full prompt (examples in Part 4 and 5).

---

### Step 3 — Answer the agent’s questions

It will ask things like:

- What is the primary key?  
- How do we handle duplicates?  
- Which columns are PII?  
- Quality thresholds or “use defaults”?  
- Cron schedule?  
- Do you want ontology / semantic enrichment? (yes/no)

**Answer honestly.** This is the same information you’d put in a design doc for your Spark framework.

---

### Step 4 — Approve the plan

Agent shows: source, zones, transforms, schedule.  
Say **Approved** or correct it.

---

### Step 5 — Confirm profiling

Agent samples data from S3 (types, nulls, possible PII).  
Confirm or correct (especially PII and primary key).

---

### Step 6 — Review what it generated

Folder appears:

```text
workloads/<dataset_name>/
  config/     ← source, transforms, quality, schedule
  scripts/    ← Spark/Glue-style jobs
  dags/       ← Airflow schedule
  tests/
  logs/
```

Because you asked for **Iceberg**, Silver/Gold should be Iceberg (platform default).

---

### Step 7 — Approve Dev deploy (optional but usual)

Agent registers tables, uploads jobs, etc. in **Dev**.  
Then it may offer:

- Run end-to-end test on AWS  
- Generate DevOps / IaC / monitoring (you apply IaC)

---

### Step 8 — Commit & promote (your process)

```text
Dev (generated code) → Git → CI/CD → QA → Staging → Prod
```

Prod runs **jobs only**, not the AI.

---

## Part 4 — Example A: CSV already on S3 → Iceberg

### Story (layman)

“We already drop daily CSV claim files on S3. We want them cleaned and stored as Iceberg tables so analysts can query them. Our framework supports Iceberg.”

### Prompt you can paste

```text
/onboard-workflow

Onboard dataset: claims_csv_iceberg

Business context:
We already land daily CSV files on S3. We want a Bronze → Silver → Gold pipeline.
Silver and Gold must use Apache Iceberg (our platform supports Iceberg).

Source (files already in S3):
- Type: S3 files (batch)
- Location: s3://my-landing-bucket/claims/incoming/
- Format: CSV
- Header: yes
- Delimiter: comma
- Frequency: daily drops under date prefixes like ingestion_date=YYYY-MM-DD/
- Credentials: use existing AWS role / Airflow connection for this bucket
  (do NOT hardcode secrets)

Bronze:
- Keep files as-is (immutable raw)
- Retention: 90 days (or ask me if unsure)

Silver (Iceberg):
- Table: iceberg, partitioned sensibly (e.g. by ingestion or service date)
- Primary key: claim_id
- Dedup: if duplicate claim_id, keep the latest by submission_date
- Nulls: drop rows only if claim_id is null; keep other nulls
- Type casts: money fields to decimal; dates to date
- PII: if present, mask or hash as you recommend — list columns and ASK me to confirm
- Extra transforms: trim strings; add ingestion_timestamp if missing

Gold (Iceberg, flat analytical table):
- Use case: analysts / BI ad-hoc queries (not star schema)
- Derived fields ideas: days_to_submission, claim_amount_tier
- Quality gate: 95%

Quality:
- Use platform defaults unless profiling shows issues — then propose rules and ask me

Compliance:
- None for this example (or say HIPAA if healthcare)

Ontology enrichment:
- No for now

Schedule:
- Daily at 02:00 UTC
- Retries: 3
- On failure: alert (SNS/email placeholder — ask me for channel)

Please:
1) Check if already onboarded
2) Profile a sample from S3
3) Ask me only what you cannot discover
4) Generate full workload with tests targeting Iceberg for Silver and Gold
5) Show me the plan before generating code
```

### How the conversation might go (plain English)

1. **You:** paste prompt above.  
2. **Agent:** “Found CSV, 31 columns, claim_id looks unique. Possible PII: email, phone. Is claim_id the PK? Confirm PII list. Dedup keep latest — OK?”  
3. **You:** “Yes PK is claim_id. Mask email and phone. Dedup OK. Use default quality. Schedule 02:00 UTC OK.”  
4. **Agent:** plan summary → you say **Approved**.  
5. **Agent:** builds `workloads/claims_csv_iceberg/`.  
6. **You:** review, approve deploy to Dev, run E2E if offered.

---

## Part 5 — Example B: EBCDIC file already on S3 → Iceberg

### Story (layman)

“Mainframe-style EBCDIC files land on S3. We need them decoded and stored as Iceberg. Our Spark framework can read EBCDIC if we give the right codec/copybook — tell the agent that.”

### Important note (honest)

ADOP examples lean toward CSV/JSON/Parquet. **EBCDIC works if you spell out** how to read it (code page, record layout / copybook, fixed width vs VB). The agent needs those details the same way a new teammate would.

### Prompt you can paste

```text
/onboard-workflow

Onboard dataset: mainframe_claims_ebcdic

Business context:
EBCDIC claim extract files are already landing on S3 from the mainframe.
We want Bronze (raw preserve) → Silver/Gold as Apache Iceberg.
Our Spark/Glue framework supports Iceberg writes. We need the pipeline generated.

Source (files already in S3):
- Type: S3 files (batch)
- Location: s3://my-landing-bucket/mainframe/claims/ebcdic/
- Format: EBCDIC
- Encoding / code page: CP037 (IBM037)   ← change if yours is different
- Record style: fixed-length
- Record length: 200 bytes               ← put your real length
- Copybook / layout: I will paste field positions below (or point to a file in repo)
- File naming: CLAIMS_YYYYMMDD.DAT
- Frequency: daily

Field layout (example — replace with real copybook):
- claim_id: bytes 1-12, PIC X(12)
- member_id: bytes 13-24, PIC X(12)
- service_date: bytes 25-32, PIC X(8) YYYYMMDD
- billed_amount: bytes 33-42, PIC S9(7)V99 (packed or zoned — specify which)
- ... add all fields ...

Bronze:
- Store original EBCDIC file bytes as-is (immutable)
- Also OK to land a decoded Parquet/CSV staging copy if needed — ask me which we prefer
- Retention: 180 days

Silver (Iceberg):
- Decode EBCDIC → typed columns using the layout above
- Primary key: claim_id
- Dedup: keep latest file date if claim_id repeats
- Reject / quarantine rows that fail decode or have null claim_id
- Cast dates and amounts correctly
- PII: confirm with me after you list candidates

Gold (Iceberg):
- Flat analytical Iceberg table for finance ops
- Grain: one row per claim_id
- Add simple derived: billed_amount_usd (if needed), service_month

Quality:
- Propose rules from profiling; I will confirm
- Critical: claim_id not null, service_date valid, amount parse success rate

Compliance:
- None (or name regulation if any)

Ontology:
- No

Schedule:
- Daily 03:30 UTC after mainframe drop (SLA: files arrive by 02:00 UTC)
- If file missing: fail the DAG and alert

Please ask clarifying questions about copybook / packed decimals before generating code.
Do not guess EBCDIC field positions.
Generate Iceberg Silver + Gold when I approve.
```

### How you should communicate EBCDIC specifics

Always say clearly:

1. **Code page** (e.g. CP037, CP1047).  
2. **Fixed vs variable** records.  
3. **Copybook / field offsets** (or path to copybook in repo).  
4. **Packed decimal / zoned / binary** for numeric fields.  
5. That **Bronze keeps original bytes** if compliance needs an audit copy.

If you don’t know something, say:  
“I don’t know the packed format — help me find it from a sample hex dump” — better than guessing.

---

## Part 6 — How this maps if you keep Hive / Snowflake too

Your framework supports Iceberg **and** you still use Hive/SF:

| Goal | What to tell ADOP |
|------|-------------------|
| Iceberg only (align with ADOP default) | “Silver and Gold must be Iceberg on S3” (examples above) |
| Iceberg + later Snowflake | “Generate Iceberg Gold first; add a final load-to-Snowflake step using our connector pattern X” |
| Must match your Spark job template | “Follow our framework conventions: … (job naming, database names, warehouse)” — paste your standards |
| Classic Hive external tables only | ADOP default won’t match 1:1 — ask it to generate transforms, then you adapt the writer to Hive |

**Practical hybrid pattern:**

1. Use ADOP to generate **clean + quality + schedule + Iceberg**.  
2. If Snowflake still needed, add one more task: `Gold Iceberg → Snowflake table` (your existing loader).  
3. Keep promoting code through **your** CI/CD.

---

## Part 7 — Mini phrasebook (how to talk to the agent)

| You want… | Say… |
|-----------|------|
| Start a new pipeline | “Onboard dataset: …” |
| Parallel faster build | `/onboard-workflow` then your story |
| Compliance | `/onboard-workflow HIPAA` (or GDPR/PCI/…) |
| Files already on S3 | “Source files are already in s3://… Do not generate a producer; only ingest.” |
| Iceberg | “Silver and Gold format: Apache Iceberg (required).” |
| Don’t invent rules | “Ask me before assuming PK, dedup, PII, or schedule.” |
| Use defaults for DQ | “Quality thresholds: use platform defaults.” |
| No ontology | “Ontology enrichment: no.” |
| Approve | “Approved — proceed.” |
| Stop / fix | “Stop generation. Change X to Y, then continue.” |
| Only Dev | “Deploy to Dev only. Do not touch Prod.” |

---

## Part 8 — Checklist before you paste a prompt

- [ ] S3 path is correct (Dev/test path if possible)  
- [ ] File format stated (CSV / EBCDIC / …)  
- [ ] For EBCDIC: code page + layout ready  
- [ ] You want Iceberg stated explicitly  
- [ ] Primary key known (or “help me choose after profiling”)  
- [ ] Dedup rule known  
- [ ] PII list or “propose and I’ll confirm”  
- [ ] Schedule known  
- [ ] No secrets in the prompt  

---

## Part 9 — Summary

| Question | Answer |
|----------|--------|
| What does ADOP do for us? | Builds the pipeline package (and optional Dev deploy) from a conversation |
| How do we submit work? | Paste a clear prompt in Claude Code in this repo |
| CSV on S3 → Iceberg? | Yes — see Part 4 |
| EBCDIC on S3 → Iceberg? | Yes if you provide layout/code page — see Part 5 |
| Same as our Spark→Hive/SF framework? | Same *idea*; different default *landing* (Iceberg). Hybrid possible |
| Do we still need humans? | Yes — for business rules, approvals, and Prod promotion |

---

## Related Indrajit docs

- `05_USER_ONBOARDING_GUIDE.md` — fuller platform how-to  
- `02_E2E_FLOW.md` — build-time vs run-time  
- `03_PITFALLS_AND_GAPS.md` — what can go wrong  
