# System Design — DAG Execution Model &amp; File-Based Mailbox

This document explains *how* SEC-SWARM runs: the deterministic DAG, the orchestrator's control loop, and the file-based mailbox that carries state between agents.

---

## 1. Mental model in one paragraph

SEC-SWARM is a **relay race**. The orchestrator (`index.js`) is the referee: it never runs "downstream" logic itself, it just decides *which runner goes next* and hands them a baton. Each runner (an agent) is a separate, sandboxed CLI process that reads the baton, does its job, and drops a handoff note before tapping out. The baton and the notes are **files on disk** in a directory called `.agent_state/`. The referee reads each note, checks it, and writes the next baton. The order of runners is fixed in code — the runners never decide the order themselves.

---

## 2. Prerequisite concepts (skip if familiar)

A few terms are used throughout. If any are new:

- **DAG (Directed Acyclic Graph):** a set of steps connected by one-way arrows, with no way to loop back forever. "Directed" = arrows have a direction (A → B, not B → A). "Acyclic" = you can't get into an infinite cycle. In SEC-SWARM the steps are the agents and the arrows are the fixed order the orchestrator runs them in. *(The one apparent loop — the verifier feeding back to the coder — is bounded to 5 attempts, so it still terminates. See [`error-handling.md`](./error-handling.md).)*
- **Orchestrator:** the plain program that decides what runs next. Here it's `index.js`, a small Node.js script with zero npm dependencies. Crucially, **the orchestrator is deterministic code, not an LLM.**
- **Agent:** one LLM-powered CLI process (a GitHub Copilot CLI invocation) that is given exactly one task at a time and a limited view of the filesystem.
- **Handoff:** a Markdown file an agent writes when it finishes, summarizing what it did and where it left its outputs. It's how the next agent (via the orchestrator) picks up context.
- **Baton / `current_task.md`:** the task file the orchestrator writes *for* an agent before running it. It contains the instructions plus the relevant upstream handoffs.
- **HITL (Human-in-the-Loop):** a deliberate pause where the program waits for a human to type approval before continuing.

---

## 3. Why deterministic control flow

The single most important design choice is that **the graph is owned by code, not by a model.**

Frameworks like LangChain agents or AutoGen let the LLM decide which tool or sub-agent to invoke next. That is powerful for open-ended exploration and terrible for a production pipeline that runs `terraform` and triggers deploys, because:

1. **Non-reproducibility.** The same request can take different paths on different runs. You cannot build reliable operations on top of that.
2. **Unbounded blast radius.** A single bad "decision" token can send the agent down the wrong branch (e.g., deploying before the security review).
3. **Opaque debugging.** When it goes wrong, there is no clean "step N received X and produced Y" record — you get a tangled trace.

SEC-SWARM inverts this: **the LLM is a worker inside a node**; it does the creative work (write PySpark, write Terraform, review for secrets) but it never decides the topology. The topology is this fixed list in the orchestrator:

```
Tree 1 (Pipeline):   0 planner → 1 script_coder → 2 data_cleaner → 3 glue_verifier
                     → 4 tf_coder → 5 secops_reviewer → 6 git_operator → 7 deploy_trigger
Tree 2 (Analytics):  8 metrics_generator → 9 powerbi_agent
```

Same input → same path, every time.

---

## 4. The file-based mailbox (`.agent_state/`)

### 4.1 The protocol

Every step follows the same four-beat cycle:

```
  ┌─────────────────────────────────────────────────────────────────┐
  │ 1. WRITE BATON   orchestrator writes .agent_state/current_task.md │
  │                  (instructions + relevant upstream handoffs)       │
  │                                                                    │
  │ 2. SPAWN AGENT   orchestrator runs the agent, sandboxed to a       │
  │                  specific set of --add-dir directories             │
  │                                                                    │
  │ 3. AGENT WORKS   agent reads current_task.md, does its job, writes │
  │                  .agent_state/<name>_handoff.md (+ any artifacts)   │
  │                                                                    │
  │ 4. READ HANDOFF  orchestrator reads + validates the handoff, then  │
  │                  composes the next baton                            │
  └─────────────────────────────────────────────────────────────────┘
```

Concretely, a slice of `.agent_state/` during a pipeline run looks like:

```text
.agent_state/
├── current_task.md                     # the baton for whoever runs next
├── formal_specs.md                     # planner output (the approved spec)
├── jira_ticket.md                      # planner output
├── planner_handoff.md                  # step 0 handoff
├── script_coder_handoff.md             # step 1 handoff
├── scripts/                            # generated Glue PySpark lives here
├── data_cleaner_handoff.md             # step 2 handoff
├── glue_verifier_handoff.md            # step 3 handoff (status: succeeded|failed)
├── glue_verifier_data_profile.md       # step 3 output on success (column types)
├── glue_verifier_sample.parquet        # step 3 sample output on success
├── tf_coder_handoff.md                 # step 4 handoff
├── secops_reviewer_handoff.md          # step 5 handoff (status: approved|rejected)
├── git_operator_handoff.md             # step 6 handoff (contains PR URL)
└── deploy_trigger_handoff.md           # step 7 handoff (deploy status)
```

### 4.2 Why files instead of memory?

Passing state in-process would be faster, but disk buys four properties that matter far more than a few milliseconds:

| Property | What it gives you |
| --- | --- |
| **Observability** | Every input and output is a human-readable file. You can `cat`, `diff`, and `grep` the entire history of a run without a special tool. |
| **Debuggability** | When a step fails, the *exact* prompt it received (`current_task.md`) and whatever it managed to produce are still on disk. You reproduce the failure by re-reading files, not by guessing. |
| **Resumability** | Because state is durable, a new invocation can reload prior handoffs and continue (see §6). |
| **Process isolation** | Each agent is a separate sandboxed process that only communicates through files. No shared memory means no hidden coupling and a much smaller trust surface. |

This is the concrete meaning of "debugging model behavior": the agents' entire I/O is a set of plain-text files you can read.

---

## 5. The two execution trees

### Tree 1 — Data Pipeline (steps 0–7)

Strictly sequential. Meeting transcript → planner → code → verified-in-dev → Terraform → security review → PR → Harness deploy to SQA. Contains the self-correction loop across steps 1–3.

### Tree 2 — Analytics (steps 8–9)

Sequential. Deployed tables → Snowflake SQL metrics → PowerBI reports. It can run two ways:

- **Chained:** after Tree 1's step 7 emits an *"SQA deploy success"* event, Tree 2 runs against the freshly deployed tables.
- **Standalone:** `--tree=analytics` runs it on already-deployed tables without touching the pipeline.

Tree selection itself is deterministic — the orchestrator infers a recommendation from the input (a transcript with pipeline signals → `pipeline`; a metrics-only request → `analytics`; both → `all`) and confirms before executing.

---

## 6. Resumability (`--from N`)

`--from N` tells the orchestrator to **skip** steps `0..N-1` and load their handoffs from `.agent_state/` instead of re-running them.

```bash
# Full run from a transcript
node index.js --input meeting_notes.md

# The security review failed; a human fixed the artifact.
# Resume from step 5 without regenerating steps 0–4.
node index.js --from 5
```

The orchestrator enforces an invariant before resuming: **every required prior handoff must exist on disk.** If, say, `tf_coder_handoff.md` is missing when you ask to resume from step 5, it aborts with a clear message rather than continuing with a hole in its context. This is why resumability and the file mailbox are the same design decision — durable files are what make "continue where we left off" *safe*, not just possible.

---

## 7. HITL gates

The orchestrator uses Node's `readline` to block on human input at exactly two gates (plus a confirmation before a resumed run):

1. **After planning** — the human reviews `formal_specs.md` and approves before any code is written.
2. **After security review** — if `secops_reviewer_handoff.md` contains `status: rejected`, the run stops and waits for a human to fix and continue.

At each gate the user types `y` to proceed or `q` to abort. Aborting is clean: the partial state remains on disk for a later `--from` resume.

---

## 8. Configuration surface

| Variable | Default | Purpose |
| --- | --- | --- |
| `AGENT_TIMEOUT_MS` | `300000` (5 min) | Per-agent execution timeout; the orchestrator logs which step timed out |
| `HARNESS_API_KEY` / `HARNESS_ACCOUNT` / `HARNESS_PROJECT` | — | Harness CI/CD auth for step 7 |
| `PBI_WORKSPACE` / `PBI_TENANT_ID` | — | PowerBI deployment target for step 9 |

---

## 9. Design trade-offs, honestly stated

- **Sequential, not parallel.** Within a tree, steps run one at a time. Some non-dependent steps *could* be parallelized, but sequential execution keeps the mailbox and the debugging story simple. Parallelism is explicitly future work.
- **Disk I/O overhead.** Writing/reading Markdown between every step costs milliseconds. Accepted deliberately in exchange for observability and resumability.
- **Handoff format drift.** An agent might not follow the handoff SOP exactly. Mitigated by having the orchestrator validate required fields (e.g., a `status:` line) before advancing.
- **CLI headless behavior.** The pipeline depends on the agent CLI exiting cleanly in a non-interactive context; this was a Phase-0 validation risk with a documented fallback (stdin pipe / alternate CLI).
