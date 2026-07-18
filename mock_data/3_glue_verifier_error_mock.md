# Glue Verifier Handoff

status: failed

<!--
  MOCK ARTIFACT — sanitized example of .agent_state/glue_verifier_handoff.md
  This is what step 3 writes on a FAILED run. The orchestrator reads `status: failed`,
  and (since attempts remain) feeds this whole report back to 1_script_coder as the
  "here is what broke, fix it" task. See docs/error-handling.md.
-->

## Attempt
2 / 5

## Verification Target
- AWS account: dev (721527036298)
- Glue job: `acme-audit-raw-to-modeled-dev`
- Job run ID: `jr_9f3c1a...e77` (sanitized)
- Script under test: `.agent_state/scripts/acme_audit_modeled.py`

## Outcome
Glue job run terminated with state **FAILED** after 78s.

## Root-Cause Signal (from CloudWatch)
```
Traceback (most recent call last):
  File "/tmp/acme_audit_modeled.py", line 61, in <module>
    modeled = flattened.withColumn(
              ...
pyspark.sql.utils.AnalysisException: cannot resolve 'actor.email' given input columns:
[event_id, event_time, actor_json, resource_json, action, outcome];
did you mean 'actor_json'?
```

## Diagnosis (verifier's read)
The script assumes `actor` was already exploded into a struct with an `.email`
field, but the raw vendor JSON delivers `actor` as a **stringified JSON blob**
(`actor_json`). The flatten step in the spec (item 2) was not applied before the
tokenization step, so the column `actor.email` does not exist at that point.

## What Was Correct
- AWS auth + S3 upload succeeded.
- Job started and read all 12 input files (dedup on `event_id` worked: 4,102 → 3,988 rows).
- Failure is isolated to the flatten/tokenize ordering, not infrastructure.

## Required Fix (fed back to 1_script_coder)
1. Parse `actor_json` (and `resource_json`) with `from_json` into structs **before**
   referencing nested fields.
2. Only then apply the `actor.email` → tokenized-hash transform (spec item 5).
3. Re-run: this should let the modeled schema resolve.

## Artifacts Produced
- .agent_state/glue_verifier_handoff.md   (this file)
- (no data_profile / sample parquet — only written on success)

---
> 🔁 Because `status: failed` and attempt 2 < 5, the orchestrator loops back to
> `1_script_coder` with this report attached, re-runs `2_data_cleaner`, and
> re-verifies. If attempt 5 also fails, the pipeline halts and this file is left
> on disk for a human. See [`docs/error-handling.md`](../docs/error-handling.md).
