# Prompt Engineering Patterns

This document describes the **reusable patterns** I used to make ten different LLM agents behave predictably enough to chain together into a production pipeline.

> [!IMPORTANT]
> This is about *patterns*, not payloads. The actual `instructions.md` files for each agent contain internal, business-specific detail and are **not** included in this repository. Everything below is the transferable engineering technique.

---

## 1. The core problem prompt engineering had to solve

In a deterministic DAG, the orchestrator guarantees *what order* agents run in. It cannot, by itself, guarantee that an agent produces output the *next* agent can consume. That contract — the shape of each agent's output — has to be enforced through the instructions. So the goal of every prompt is not "be clever," it's **"be predictable and composable."**

Two failure modes drove every pattern here:

1. **Handoff drift** — an agent finishes its work but writes its summary in a shape the orchestrator can't parse, breaking the relay.
2. **Scope creep** — an agent decides to "help" by touching files or systems outside its job, which is both a correctness and a security problem.

---

## 2. Pattern: the structured handoff contract

Every agent ends by writing `<name>_handoff.md` in a **fixed structure**. The orchestrator validates required fields before advancing, so the contract is enforced, not merely requested.

The reusable shape:

```markdown
# <Agent> Handoff

status: <succeeded | failed | approved | rejected>   # machine-readable, checked by orchestrator

## Summary
<1–3 sentences a human can skim>

## Artifacts Produced
- .agent_state/<file>            # explicit, relative paths
- .agent_state/scripts/<file>

## Details / Findings
<the substance — profile, errors, review notes, PR URL, etc.>

## Next Step Notes
<anything the downstream agent must know>
```

**Why it works:** the first line is a single machine-readable token (`status:`) the orchestrator branches on; the rest is human-readable. One artifact serves both the program and the person debugging it.

---

## 3. Pattern: single responsibility per agent

Each agent instruction defines **one job and its boundaries**, in this order:

1. **Role** — "You are the Glue verifier. You run scripts in dev and report whether they work."
2. **Inputs** — exactly which files in `.agent_state/` to read.
3. **Procedure** — a numbered, ordered list of steps (see §4).
4. **Outputs** — exactly which files to write, in the handoff contract shape.
5. **Boundaries** — what NOT to do ("do not modify Terraform," "do not open a PR").

Keeping each agent narrow is what makes the whole system debuggable: when something is wrong, the blame is localized to one node with one job.

---

## 4. Pattern: numbered, imperative procedures

Wherever an agent must perform a real sequence of side-effecting actions (auth → upload → run → poll → validate), the instruction is a **numbered imperative checklist**, not prose. Example shape (from the verifier's baton):

```
1. Authenticate to AWS dev via tools/aws_auth.sh
2. Upload script(s) from .agent_state/scripts/ to S3
3. Start the Glue job run in dev
4. Poll until completion
5. If SUCCEEDED: download sample output, validate columns/types
6. If FAILED: pull run status + CloudWatch logs, write error details
7. Write handoff with status: succeeded | failed
```

**Why it works:** ordered steps reduce the LLM's freedom to reorder or skip side effects, and the explicit success/failure branches (steps 5 vs 6) make the two outcomes symmetric and predictable.

---

## 5. Pattern: the baton carries just-enough upstream context

The orchestrator composes each `current_task.md` from **only the handoffs that step actually needs**, not the entire run history:

- The Terraform coder gets the script handoff + the Glue data profile (for accurate column types) — not the raw meeting transcript.
- The fix task gets the original spec + the verifier's error — not the intermediate cleaner output.

**Why it works:** it keeps each prompt focused (less distraction, lower token cost) while guaranteeing the agent has everything required for its decision. Context is curated by deterministic code, not left to the model to go fetch.

---

## 6. Pattern: explicit output-location discipline

Every instruction names the **exact relative path** for each output (`.agent_state/scripts/...`, `.agent_state/<name>_handoff.md`). Combined with filesystem sandboxing (`--add-dir`), this makes "where did the output go?" a non-question — outputs land in known, inspectable locations every time. This is also what makes the mock artifacts in [`../mock_data/`](../mock_data/) representative of a real run.

---

## 7. Pattern: status-driven branching over free-text interpretation

The orchestrator never tries to "understand" an agent's prose to decide what happens next. It looks for a specific token:

- `status: succeeded` / `status: failed` → verifier loop branch.
- `status: approved` / `status: rejected` → security HITL branch.

**Why it works:** parsing a controlled vocabulary is deterministic; parsing free text is not. The LLM writes rich explanations for humans, but the *control decision* rides on one token it's instructed to always emit.

---

## 8. Pattern: reference docs over inlined knowledge

Stable, shared knowledge (e.g., the raw-only vs. raw+modeled decision flow, the ETL validation checklist) lives in read-only `_shared/` reference docs that multiple agents are pointed at via `--add-dir`, rather than being copy-pasted into each instruction.

**Why it works:** one source of truth, no drift between agents, and updating a policy means editing one file instead of five prompts.

---

## 9. Summary table

| Pattern | Problem it solves |
| --- | --- |
| Structured handoff contract | Handoff drift; makes the relay composable |
| Single responsibility per agent | Localizes blame; keeps nodes debuggable |
| Numbered imperative procedures | Prevents skipped/reordered side effects |
| Just-enough upstream context | Focus + cost, without starving the agent |
| Explicit output locations | Observability; predictable file layout |
| Status-driven branching | Deterministic control flow over prose |
| Shared reference docs | One source of truth, no cross-agent drift |
