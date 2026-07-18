# Error Handling — The Self-Correcting Execution Loop

This is the heart of SEC-SWARM. Most "AI writes code" demos stop at generation. SEC-SWARM **runs the generated code against real AWS infrastructure, reads the real failure, and fixes itself** — up to a bounded number of attempts — before it will let anything move toward a deploy.

---

## 1. The loop in one picture

```
          ┌──────────────────────────────────────────────────────────────┐
          │                     Verify → Fix → Re-verify                    │
          │                      (max 5 attempts)                           │
          └──────────────────────────────────────────────────────────────┘

   ┌─────────────────┐   python_handoff   ┌────────────────┐   cleaned_handoff
   │ 1_script_coder  │ ─────────────────► │ 2_data_cleaner │ ─────────────────┐
   └─────────────────┘                    └────────────────┘                  │
          ▲                                                                    ▼
          │                                                        ┌────────────────────┐
          │   Task: "Fix the script.                               │  3_glue_verifier   │
          │   Here is the CloudWatch                               │  run job in AWS dev │
          │   error + run status."                                 └────────────────────┘
          │                                                             │           │
          │                                            status: failed   │           │  status: succeeded
          └─────────────────────────────────────────────────────────── ┘           ▼
                                                                            ┌────────────────┐
                                                                            │   4_tf_coder   │
                                                                            └────────────────┘
```

- **Success path:** verifier returns `status: succeeded` → the DAG advances to Terraform (step 4).
- **Failure path:** verifier returns `status: failed` → the orchestrator loops back to the script coder with the error attached, re-runs the cleaner, and re-verifies.
- **Give-up path:** after `MAX_VERIFY_ATTEMPTS = 5` failed attempts, the orchestrator halts, prints the final verifier handoff, and exits non-zero — leaving all artifacts on disk for inspection.

---

## 2. Why this exists

Generating plausible-looking PySpark is easy. Generating PySpark that **actually runs in AWS Glue against the real schema** is the hard part, and it's exactly where LLMs fail silently — a wrong column name, a type mismatch, a missing null-handling branch, a bad S3 path. None of that shows up until the job runs.

So `3_glue_verifier` is not a linter. It is an **execution-based test**: it stands up the job in the dev account and observes what really happens. The feedback it produces is not "the model thinks this looks wrong" — it is the actual CloudWatch stack trace.

This is the design principle: **the ground truth is the runtime, not the model's opinion.**

---

## 3. What `3_glue_verifier` actually does

Each attempt, the verifier is handed a baton instructing it to:

1. Authenticate to the AWS **dev** account (`721527036298`) via `aws_auth.sh`.
2. Upload the generated script(s) from `.agent_state/scripts/` to S3.
3. Start the Glue job run in dev.
4. Poll until the run reaches a terminal state.
5. **On `SUCCEEDED`:** download sample output, validate the columns and types, write a data profile to `glue_verifier_data_profile.md`, save a sample parquet to `glue_verifier_sample.parquet`.
6. **On `FAILED`:** pull the run status **and the CloudWatch logs**, and write the error details into the handoff.
7. Write `glue_verifier_handoff.md` with an explicit `status: succeeded` or `status: failed`.

The `status:` line is the machine-readable signal the orchestrator branches on — it does a case-insensitive check for `status: succeeded` in the handoff text.

---

## 4. How the fix is fed back

When the verifier fails and attempts remain, the orchestrator composes a **new baton for the script coder** that contains three things:

1. The **original specifications** (so the coder never loses the intent).
2. The **verifier's error report** (run status + CloudWatch logs).
3. An instruction to **diagnose the root cause, fix the script(s) in `.agent_state/scripts/`, and describe the change** under a `## Fix Applied` section.

The coder produces an updated `script_coder_handoff.md`; the cleaner re-applies its rules; the verifier runs again. Because everything travels through the mailbox, each attempt's error and each attempt's fix are preserved on disk — you can reconstruct the entire "debugging conversation" after the fact.

---

## 5. Why the loop is *bounded* (max 5)

An unbounded retry loop is how you turn a bug into a runaway bill. The cap of 5 attempts is a deliberate trade-off:

- Enough attempts to absorb the common, genuinely fixable failures (typo, type mismatch, minor logic error).
- Few enough that a fundamentally-wrong spec or a transient AWS outage cannot spin forever burning LLM tokens and Glue compute.

When the cap is hit, the orchestrator does **not** silently continue. It halts the pipeline, prints the last verifier handoff (the most recent real error), and exits with a non-zero code. The partial state stays in `.agent_state/`, so a human can inspect it and either fix manually and `--from`-resume, or adjust the spec and re-run.

This is also what keeps the "loop" compatible with calling the system a DAG: the cycle is bounded, so the whole graph still terminates.

---

## 6. The other failure surfaces

The verifier loop is the marquee example, but reliability is designed in at several other points:

| Failure surface | Handling |
| --- | --- |
| **Security rejection (step 5)** | `secops_reviewer` returns `status: rejected` → hard **HITL** stop; a human must fix and continue. Not an auto-retry, because security issues need human judgment. |
| **Agent non-zero exit** | Any agent exiting non-zero throws; the orchestrator stops and points you at `.agent_state/` for the partial artifacts. |
| **Per-agent timeout** | `AGENT_TIMEOUT_MS` (default 5 min) bounds each spawn; the orchestrator logs exactly which step timed out. |
| **Handoff format drift** | The orchestrator validates required fields (e.g., the `status:` line) before advancing, so a malformed handoff fails fast instead of corrupting downstream steps. |
| **External API flakiness (Harness/GitHub)** | Step 7 polls with a timeout; step 6 relies on authenticated `gh` CLI. Low severity, explicitly acknowledged as a network dependency. |

---

## 7. On-demand troubleshooting: the `glue-troubleshoot` skill

Separate from the in-DAG retry loop, there's a standalone `glue-troubleshoot` skill (invoked ad-hoc, not part of the graph) that triages *recently failed* Glue jobs — pulling CloudWatch logs and the PySpark source into context. It's the manual counterpart to the automated loop: same philosophy (get the real logs), used when a human is investigating rather than when the pipeline is self-correcting.

---

## 8. What a reviewer should take away

> I did not just use an LLM to write code. I used an LLM to **run tests, read the real error output, and patch the code** — inside a loop that is bounded so it can't run away, and observable so every attempt is on disk. That is the difference between a demo and a system that has to work in the real world.
