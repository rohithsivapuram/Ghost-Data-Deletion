# Ghost Data Deletion-Audit Agent — User Manual

A complete, from-scratch guide to understanding this project: what it does, how it
is structured, how data flows through it end to end, and exactly what lives in
each source file.

---

## 1. What This Project Does

This is the solution for the HackerRank Orchestrate challenge **"Ghost Data Still
Exists."** It is a deletion-audit agent.

The business problem: a company's deletion workflow can *report* that a person's
data was deleted ("COMPLETE"), while copies of that data still linger in search
indexes, support messages, analytics, exports, caches, backups, replicas, and
tombstones — or while an active sync/restore process can quietly recreate deleted
data. Reported statuses are **claims**, not proof.

For each of the 518 requests in `dataset/deletion_requests.csv`, the agent:

1. Decides whether the request actually **authorizes deletion** (from the free-text
   `request_text`, which may be in any language).
2. Resolves the subject's **current and historical identifiers** (email, email
   hash, phone, name, stable `user_id`).
3. Reconstructs the **true per-record state** of that person's data at the case's
   `audit_at` cutoff, from supplied multimodal evidence — never from claims.
4. Applies the supplied **deletion policy** (`deletion_rules.csv`) to each record.
5. Decides a single verdict: **`COMPLETE`**, **`INCOMPLETE`**, or
   **`NOT_APPLICABLE`**.
6. Emits an ordered **`steps_json`** plan (findings → remediation → verification).
7. Writes **`output.csv`** at the repository root with exactly 518 rows sorted by
   `case_id`.

### The key design decision: deterministic-first, LLM-optional

The evidence is highly structured and carries explicit state signals, so the
**default engine is a pure deterministic Python pipeline with zero model calls.**
It is reproducible, costs nothing, and still satisfies the usage report (by
reporting zero calls). An optional LLM layer exists only as a fallback for two
genuinely fuzzy problems — ambiguous multilingual intent and unreadable PDF/image
artifacts — and normalizes its output back into the same structures so downstream
logic stays deterministic.

---

## 2. High-Level Architecture

The pipeline is a linear sequence of pure-where-possible components. Each stage
takes typed data and produces typed data; shared state (loaded dataset, parsed
evidence) is passed explicitly so behavior is deterministic.

```
                        dataset/*.csv  +  dataset/evidence/*
                                     │
                                     ▼
                           ┌──────────────────┐
                           │   Input Loaders  │  loaders.py
                           └──────────────────┘
                                     │  Cases, IdentityStore, rules, manifest
                                     ▼
                           ┌──────────────────┐
                           │ Intent Classifier│  intent.py   (per case)
                           └──────────────────┘
                              │ not deletion → NOT_APPLICABLE (empty plan)
                              │ deletion
                                     ▼
                           ┌──────────────────┐
                           │ Identity Resolver│  identity.py
                           └──────────────────┘
                              │ no subject → NOT_APPLICABLE
                              │ subject resolved
                                     ▼
          parse ALL evidence ONCE   ┌──────────────────┐
          (dispatch.parse_all) ───► │ Evidence Retriever│ evidence.py
                                     └──────────────────┘
                                     │  file-gate ≤ audit_at, match subject,
                                     │  closest observation per resource
                                     ▼
                           ┌──────────────────┐
                           │ State Reconstructor│ state.py
                           └──────────────────┘
                                     │  one RecordState per resource @ audit_at
                                     ▼
                           ┌──────────────────┐
                           │   Policy Engine   │ policy.py
                           └──────────────────┘
                                     │  PolicyDecision per record
                                     ▼
                           ┌──────────────────┐
                           │  Decision Engine  │ decision.py
                           └──────────────────┘
                                     │  COMPLETE / INCOMPLETE / NOT_APPLICABLE
                                     ▼
                           ┌──────────────────┐
                           │  Step Generator   │ steps.py + ordering.py
                           └──────────────────┘
                                     │  ordered, numbered steps
                                     ▼
                           ┌──────────────────┐
                           │    Serializer     │ serialize.py
                           └──────────────────┘
                                     ▼
                              output.csv (repo root)
                                     │
                                     ▼
                           ┌──────────────────┐
                           │  Output Validator │ validate.py
                           └──────────────────┘

   Cross-cutting:  logging_util.py (session log)   usage.py (token/cost report)
```

The orchestrator (`pipeline.py`) wires these together and is driven by the terminal
entry point (`main.py`).

---

## 3. The `audit_at` Time-Gate (the core correctness idea)

`audit_at` is the UTC cutoff for each case; the answer must reflect the data state
**at that instant** and ignore later evidence. Gating happens at **two independent
levels**, which is why the agent is trustworthy:

1. **File-level gate** (`pipeline._file_eligible_records` using the manifest):
   a record may only contribute if the evidence file it came from has
   `captured_at <= audit_at`.
2. **Observation-level gate** (`evidence.select_closest`): among file-eligible
   records, keep per resource the observation with the greatest `observed_at` that
   is still `<= audit_at`. The State Reconstructor applies the same check again as
   defense in depth.

Claims are never proof: `reported_status` and every `system_activity.csv`
`reported_result` are loaded but never passed into state reconstruction.

---

## 4. End-to-End Flow for One Case

This is exactly what `pipeline.process_case` does:

1. **Intent** — `classify_intent(request_text)`. If it is not a deletion request
   (access, correction, export, opt-out, "keep my account"), return
   `NOT_APPLICABLE` with an empty step list immediately.
2. **Identity** — `resolve_identity(case, store)`. Match the case email against
   current and historical emails/hashes to find the stable `user_id`, then compile
   all identifiers valid at `audit_at`. If nothing matches, the Decision Engine
   returns `NOT_APPLICABLE` (nothing to audit).
3. **File gate** — keep only records from evidence files captured `<= audit_at`.
4. **Subject match** — `match_records_to_subject` keeps records that share an
   exact normalized identifier with the subject. Near-but-not-equal is excluded.
5. **Closest observation** — `select_closest` picks the latest observation
   `<= audit_at` per `(system_id, resource_id)`.
6. **State** — `reconstruct_state` builds one `RecordState` per resource: present?
   exposed identifiers? treatment (REDACTED/ANONYMIZED/DELETED)? active data flow?
   restore risk? which evidence proves it?
7. **Policy** — `apply_policy(state, rules)` per record: required action and whether
   the record is compliant, plus the finding/remediation/verification operations.
8. **Decision** — `decide(...)` returns exactly one status.
9. **Steps** — `generate_steps(status, decisions)` builds the ordered plan.

The full run (`pipeline.run`) loads inputs once, **parses all evidence once**, then
runs `process_case` for all 518 cases. Each case is isolated in a try/except so a
single failure still yields a safe `NOT_APPLICABLE,[]` row — guaranteeing exactly
518 output rows.

---

## 5. The Decision Logic (how a verdict is chosen)

| Situation | Verdict |
|---|---|
| `request_text` does not authorize deletion | `NOT_APPLICABLE` (empty plan) |
| Authorizing, but email matches no subject | `NOT_APPLICABLE` (nothing auditable) |
| Authorizing, subject resolved, every record compliant, no active flow/restore | `COMPLETE` |
| Authorizing, subject resolved, any non-compliant record / exposed PII / active flow / unblocked restore | `INCOMPLETE` |
| Authorizing, subject resolved, but zero located records | `NOT_APPLICABLE` (no proof of absence) |

The "zero located records" behavior is isolated in
`decision.EMPTY_DECISIONS_STATUS` so it can be tuned against the sample set without
touching the branching logic.

---

## 6. The Step Plan (`steps_json`)

Every step object has exactly seven fields: `step_no`, `phase`, `system_id`,
`resource_id`, `operation`, `state`, `evidence_id`.

Steps per status:

- **`NOT_APPLICABLE`** → `[]` (empty array).
- **`COMPLETE`** → only `VERIFICATION` steps, each citing the `evidence_id` that
  proves the compliant state at `audit_at`.
- **`INCOMPLETE`** → `FINDING` steps (each citing participant-visible evidence),
  then the full `REMEDIATION` plan, then `VERIFICATION` steps. Remediation and
  post-remediation verification describe *future* work, so their `evidence_id` is
  an empty string `""`.

### Ordering and numbering (deterministic)

Steps are sorted by the composite key then numbered `1..N`:

```
(Phase, OPERATION_ORDER[operation], system_id, resource_id, operation, state, evidence_id)
```

- Phase order: `FINDING` < `REMEDIATION` < `VERIFICATION`.
- Within `REMEDIATION`, `DISABLE_DATA_FLOW` and `BLOCK_RESTORE` are ranked before
  record ops (`DELETE`/`REDACT`/`ANONYMIZE`) so recreation paths are stopped first.
- The trailing fields extend the key to a *total* order so numbering is fully
  independent of the input record order.

---

## 7. The Policy Table (category → treatment)

`policy.py` maps each record's `data_category` to a required action and the ops to
emit when it is non-compliant:

| data_category | required action | finding | remediation | verification |
|---|---|---|---|---|
| USER_PROFILE, SECONDARY_IDENTIFIER, SEARCH_INDEX_ENTRY, CACHE_ENTRY, USER_EXPORT, MARKETING_PROFILE, SUPPORT_ATTACHMENT | DELETE | REPORT_REMAINING_RECORD | DELETE_RECORD | VERIFY_RECORD_ABSENT |
| SUPPORT_MESSAGE, EMAIL_DELIVERY_LOG | REDACT | REPORT_EXPOSED_PII | REDACT_RECORD | VERIFY_RECORD_REDACTED |
| ANALYTICS_PROFILE, PRODUCT_TELEMETRY | ANONYMIZE | REPORT_EXPOSED_PII | ANONYMIZE_RECORD | VERIFY_RECORD_ANONYMIZED |
| SECURITY_AUDIT_LOG, LEGAL_HOLD_RECORD | RETAIN | (none) | (none) | VERIFY_RECORD_PRESENT |
| BACKUP_ARCHIVE | RETAIN + block restore | REPORT_RESTORE_RISK | BLOCK_RESTORE | VERIFY_RESTORE_BLOCKED |
| ACCOUNT_TOMBSTONE | REDACT (keep user_id) | REPORT_EXPOSED_PII | REDACT_RECORD | VERIFY_RECORD_REDACTED |
| DATA_FLOW (active) | DISABLE | REPORT_ACTIVE_DATA_FLOW | DISABLE_DATA_FLOW | VERIFY_DATA_FLOW_DISABLED |

Special cases:

- **Tombstones**: keep the stable `user_id`; compliant only when all direct
  identifiers (email/hash/phone/name) are gone. The bare `user_id` is deliberately
  excluded from "exposed identifiers" so a correct tombstone is not flagged.
- **RETAIN records** are never marked for deletion/redaction/anonymization.
- **Backups** are never deleted; only their restore path is blocked.

---

## 8. File-by-File Reference

### Entry point and orchestration

**`code/main.py`** — Terminal entry point. Parses args/env (`--root`, `--sample`),
logs the session start, loads requests, runs the full pipeline, writes `output.csv`
at the repo root, validates it, writes the usage report, prints a status summary,
and appends a per-turn log entry. Exit code `0` when valid, `1` on any violation.
Reads config/secrets only from environment variables (`ENGINE`, `LLM_PROVIDER`,
`LLM_MODEL`, `KIRO_TOOL`).

**`code/ghost_audit/pipeline.py`** — The orchestrator. `select_engine()` chooses
the engine from `ENGINE` (deterministic default, with a safe fallback).
`process_case(...)` runs the nine stages for one case. `run(...)` loads inputs,
parses all evidence once, builds the file-level `captured_at` map, and processes
every case with per-case try/except isolation.

### Data definitions

**`code/ghost_audit/models.py`** — All frozen dataclasses (immutable, hashable):
`Case`, `ResolvedIdentity`, `EvidenceFile`, `ParsedRecord`, `RecordState`,
`PolicyRule`, `PolicyDecision`, `Step`, `CaseResult`, plus identity types
(`IdentitySnapshot`, `AuditLogEntry`, `IdentityStore`) and `ActivityRow`. Defines
`ResourceKey = tuple[str, str]`.

**`code/ghost_audit/normalize.py`** — Language-agnostic normalization: `strip_accents`
(NFKD + drop combining marks), `normalize_text` (casefold + strip accents),
`normalize_email`, and `normalize_name`. Used everywhere identifiers are compared.

### Loading

**`code/ghost_audit/loaders.py`** — One loader per CSV: `load_requests`,
`load_identities` (snapshot + audit log into an `IdentityStore`), `load_rules`
(keyed by `data_category`), `load_evidence_manifest` (resolves evidence paths under
`dataset/`), and `load_system_activity` (claims only). `parse_timestamp` turns the
dataset's `...Z` timestamps into timezone-aware UTC datetimes. All paths resolve
relative to the repo root for cross-platform operation.

### Per-case pipeline stages

**`code/ghost_audit/intent.py`** — Deletion-intent classifier. A pluggable `Engine`
protocol with the zero-cost `DeterministicEngine` default. Uses a multilingual
deletion lexicon (Latin + CJK), an exclusion lexicon (access/correction/export/
opt-out), and "keep the account" signals. Exclusion + keep-account decisively
overrides incidental deletion verbs like "removing." Returns `IntentResult`
(boolean + reason tag).

**`code/ghost_audit/identity.py`** — `resolve_identity(case, store)`. Matches the
case email against current and historical emails/hashes to find the stable
`user_id` (historical hits resolve the same `user_id`). Compiles the identifier set
gated by `event_time <= audit_at` (snapshot values gated by `updated_at`; audit-log
changes contribute both old and new values when the change is `<= audit_at`).
Returns `None` when no subject matches.

**`code/ghost_audit/evidence.py`** — `eligible_files` (file-level `audit_at` gate),
`match_records_to_subject` (exact normalized identifier equality only), and
`select_closest` (greatest `observed_at <= audit_at` per `(system_id, resource_id)`).
All pure functions.

**`code/ghost_audit/state.py`** — `reconstruct_state(...)`. Turns the closest record
per resource into one `RecordState` each: presence, exposed direct identifiers
(excluding bare `user_id`), observed treatment, active-data-flow flag, restore-risk
flag, and the proving evidence id. `SYSTEM_CATEGORY` maps a `system_id` to a policy
category when a record lacks its own. Re-applies the `audit_at` gate as defense in
depth and sorts output for stable processing.

**`code/ghost_audit/policy.py`** — `apply_policy(state, rules)`. Implements the
category→treatment table above and returns a `PolicyDecision` with
`compliant`, `finding_op`, `remediation_op`, and `verification_op`. Finding and
remediation ops are nulled when the record is already compliant; `verification_op`
is always present. Handles DATA_FLOW, tombstones, backups, and RETAIN specially.

**`code/ghost_audit/decision.py`** — `decide(identity, decisions, intent)`. Returns
exactly one of `COMPLETE` / `INCOMPLETE` / `NOT_APPLICABLE` using the matrix in
section 5. The empty-decisions behavior is the tunable constant
`EMPTY_DECISIONS_STATUS`.

**`code/ghost_audit/ordering.py`** — The canonical ordering: the `Phase` IntEnum and
the `OPERATION_ORDER` rank map (disable/block before dependent record ops).

**`code/ghost_audit/steps.py`** — `generate_steps(status, decisions)`. Builds the
per-status plan, pairs each operation with its single allowed state via
`OPERATION_STATE` (so illegal pairs can't be emitted), then sorts by a total key
and numbers `1..N`.

### Parsers (multimodal evidence)

**`code/ghost_audit/parsers/__init__.py`** — Package doc; re-exports
`dispatch_parse` and `parse_all`.

**`code/ghost_audit/parsers/dispatch.py`** — Routes an `EvidenceFile` to the right
parser by extension/filename (`.txt`→handover, `.log`→logs, `.json`→json_records,
`.sql`→sql, `.pdf`/image→optional). `dispatch_parse` never raises; `parse_all`
parses every file in manifest order (deterministic), aggregating records and
collecting `{file_id: error}` failures so processing continues on bad files.

**`code/ghost_audit/parsers/handover.py`** — Parses plain-text handover lines
`ts | system | resource | CATEGORY | STATE | identifier` into `ParsedRecord`s.

**`code/ghost_audit/parsers/logs.py`** — Parses `key=value` `*_read_completed` log
lines; maps `http_status` 404/200, `enabled`, `archive_present`/`restore_allowed`,
`result=hit/miss`, and email presence to state signals.

**`code/ghost_audit/parsers/json_records.py`** — Parses snapshot bundles
(`snapshots[].records[]`), identity-service tombstones
(`deleted_customer_references`), and service-export `profile_lookups` status codes.

**`code/ghost_audit/parsers/sql.py`** — Loads `.sql` dumps via `sqlite3` into an
in-memory DB; reads `customer_data_primary` and replica tables, parsing
`record_json` with `origin_system`/`collection_name`/`updated_at` for replication
drift.

**`code/ghost_audit/parsers/pdf.py`** / **`image.py`** — Optional parsers.
Deterministic PDF text extraction with optional OCR/LLM fallback; image parsing via
OCR/LLM only. Degrade to "no facts extracted" on failure.

**`code/ghost_audit/parsers/_common.py`** — Shared parser helpers.

### Output, validation, and conformance

**`code/ghost_audit/serialize.py`** — `steps_to_json` (canonical, compact JSON via
`sort_keys=True, separators=(",",":")` → idempotent), `json_to_steps` (round-trip
preserving array order), `build_rows` (fills any missing case with a safe default
so all 518 rows exist), and `write_output` (CSV with the exact header, rows sorted
by `case_id`, `QUOTE_MINIMAL`).

**`code/ghost_audit/validate.py`** — `validate_output` returns a list of `Violation`
(never raises) checking header, row count, unique/known case ids, valid statuses,
consecutive `step_no`, step schema, empty array for `NOT_APPLICABLE`, COMPLETE =
verification-only with non-empty evidence, and empty `evidence_id` on INCOMPLETE
remediation/verification. `compare_to_sample` / `load_output_rows` support the
sample comparison (key/whitespace-insensitive, array-order-significant).

**`code/ghost_audit/sample_harness.py`** — Runs the pipeline over the 32 sample
cases and compares to `sample_output.csv`, producing a `SampleRunReport` and a
human-readable `format_report`. Has its own CLI `main`.

**`code/ghost_audit/determinism.py`** — Runs the full pipeline twice and asserts the
two output artifacts are identical in row order, statuses, and step arrays.
`check_determinism` / `compare_rows` plus a CLI `main`.

### Cross-cutting

**`code/ghost_audit/logging_util.py`** — Session logger per AGENTS.md §5.
`log_session_start` and `log_turn` (append-only, UTF-8, `\n`), mandatory non-empty
`tool=`/`model=` lines, and `redact_secrets` to mask api keys/tokens/passwords/
bearer tokens before writing.

**`code/ghost_audit/usage.py`** — `UsageReporter` records model calls and writes
`usage_report.md`. Deterministic runs report zero calls and `$0.00`; LLM runs report
per-model and overall token totals, averages per case, and estimated cost. Never
writes secrets.

### Optional LLM engine

**`code/ghost_audit/engines/llm.py`** — Implements the `Engine` interface for
ambiguous intent and unreadable PDF/image fallback, normalizing output back into the
standard structures and recording calls via `UsageReporter`. Only used when
`ENGINE=llm_assisted`.

### Evaluation

**`code/evaluation/main.py`** — Evaluation harness entry point.

**`code/evaluation/generate_evaluation_report.py`** — Computes case-level,
step-level, field-level, and sequence accuracy vs `sample_output.csv`, adds custom
cases, and writes `evaluation_report.md`.

**`code/evaluation/generate_usage_report.py`** — Produces `usage_report.md` for the
final run.

### Tests

`code/tests/` contains unit tests (one per module) and property-based tests
(Hypothesis) named `test_*_properties.py`. Property tests validate universal
correctness properties: evidence time-gating, closest-observation selection, RETAIN
never removed, single-status assignment, phase ordering, consecutive numbering,
order-independence, idempotent serialization, and round-trip. Current state: all
tests pass.

---

## 9. How to Run It

All commands are run from the repository root. On this Windows machine the Python
launcher is `py` (use `python3` on macOS/Linux).

**Full run over all 518 cases** (writes `output.csv` at the repo root):

```bash
py code/main.py
```

**Sample conformance run** (32 public cases vs `sample_output.csv`):

```bash
py code/main.py --sample
# or the dedicated harness:
py -m ghost_audit.sample_harness --root .
```

**Determinism check** (runs twice, confirms identical output):

```bash
py -m ghost_audit.determinism --root .
```

**Run the test suite:**

```bash
cd code
py -m pytest -q
```

### Configuration (environment variables only — never hardcode secrets)

| Variable | Default | Meaning |
|---|---|---|
| `ENGINE` | `deterministic` | `deterministic` or `llm_assisted` |
| `LLM_PROVIDER` | (unset) | Provider name for the optional LLM engine |
| `LLM_MODEL` | `deterministic` | Model id (also used in the log `model=` line) |
| `KIRO_TOOL` | `ghost-audit` | Value for the log `tool=` line |

No secret value is ever printed or written to `output.csv`, `log.txt`, or the
reports.

---

## 10. Inputs and Outputs

### Inputs (all under `dataset/`)

- `deletion_requests.csv` — 518 cases to answer (the only file requiring
  predictions).
- `sample_deletion_requests.csv` / `sample_output.csv` — 32 public cases + correct
  answers.
- `user_identities.csv` — current identity snapshot per user.
- `user_audit_log.csv` — historical identity changes.
- `system_activity.csv` — global event stream (claims only; may include decoys).
- `deletion_rules.csv` — required action per data category.
- `evidence_files.csv` — manifest mapping `file_id` → path, origin, `captured_at`.
- `evidence/` — the actual evidence (databases/documents/images/logs/records).

### Output

- `output.csv` at the repo root: header `case_id,deletion_status,steps_json`,
  exactly 518 rows sorted by `case_id`.
- `code/evaluation/evaluation_report.md` and `code/evaluation/usage_report.md`.
- `log.txt` at the repo root (session + per-turn transcript).

---

## 11. Why It Is Trustworthy (correctness guarantees)

- **Time-correct**: two-level `audit_at` gating means no future evidence can leak
  into a verdict.
- **Evidence-grounded**: claims (`reported_status`, activity results) are never used
  as proof.
- **Deterministic**: evidence parsed once in fixed order; pure sorting/numbering;
  canonical JSON. Running twice yields byte-identical output.
- **Contract-safe**: the serializer guarantees exactly 518 rows (failed cases
  default to `NOT_APPLICABLE,[]`); the validator independently re-checks the
  contract; only allowed operation/state pairs are ever emitted.
- **Self-checking**: the sample harness compares against known-correct answers, and
  the determinism check proves reproducibility.
