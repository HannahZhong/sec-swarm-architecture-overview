# Tooling &amp; Security

SEC-SWARM treats every agent as **untrusted by default**. An LLM process that can write PySpark, run Terraform, open PRs, and trigger deploys is a large amount of power to hand to something that can hallucinate. The security posture is built on one idea: **least privilege for both files and tools.**

---

## 1. Prerequisite concepts (skip if familiar)

- **Sandboxing:** restricting what a process is allowed to see and do, so that even if it misbehaves, the damage is bounded. Here, sandboxing is primarily about *which directories* an agent can read/write.
- **`--add-dir`:** a CLI flag that grants an agent visibility into a specific directory. If a directory isn't added, the agent cannot see it. This is an **allowlist**: nothing is visible unless explicitly granted.
- **MCP (Model Context Protocol):** a standard that lets AI tools auto-discover and connect to external "servers" that expose capabilities (filesystem access, web, databases, etc.). Convenient, but it *widens* the set of things an agent can reach — often more than you intend.
- **Least privilege:** the security principle that any component should have the minimum access required to do its job, and nothing more.

---

## 2. Filesystem sandboxing via `--add-dir`

Each agent is spawned with an **explicit, minimal** set of directories. It literally cannot see the rest of the filesystem. The grants are tailored per step:

| Agent | Directories granted (`--add-dir`) | Rationale |
| --- | --- | --- |
| `0_planner` | `.agent_state` | Only needs the mailbox to read the task and write specs |
| `1_script_coder` | `.agent_state`, `aws-secdatalake/src` | Needs existing script patterns to reference |
| `2_data_cleaner` | `.agent_state`, `aws-secdatalake/src` | Same reference scope for cleaning rules |
| `3_glue_verifier` | `.agent_state`, `aws-secdatalake/src` | Runs scripts; reads/writes only the mailbox |
| `4_tf_coder` | `.agent_state`, `aws-secdatalake/snowflake`, `aws-secdatalake/glue` | Needs Snowflake + Glue TF references, nothing else |
| `5_secops_reviewer` | `.agent_state` | Reviews artifacts already in the mailbox |
| `6_git_operator` | `.agent_state` | Commits artifacts from the mailbox |
| `7_deploy_trigger` | `.agent_state` | Triggers deploy from the git handoff |

**Consequences of this design:**

- An agent **cannot** wander into an unrelated repo, read a `.env` outside its grant, or write files where they don't belong.
- The `_shared/` knowledge docs are exposed **read-only** and only to the agents that need them (`pipeline-architecture.md` → script + TF coders; `pipeline-validation.md` → verifier + SecOps).
- "Where could this output possibly have gone?" has a short, enumerable answer for every step — which is what makes the file mailbox trustworthy as an audit trail.

---

## 3. Built-in MCP servers are explicitly disabled

Agents are launched with **`--disable-builtin-mcps`**. This is a deliberate reduction of the tool surface.

**The trade-off:**

- *With* built-in MCP: the agent auto-discovers a broad set of capabilities. Convenient, but you no longer have a tight, auditable list of what the agent can do — and a hallucinated tool call has more places to go.
- *Without* built-in MCP (SEC-SWARM's choice): the agent starts with **no ambient capabilities**. Every tool it can use is a **specific, vetted shell script** placed in that agent's `tools/` directory and referenced from its instructions.

So instead of "the agent can reach whatever MCP exposes," it's "the agent can run exactly these scripts, and I wrote every one of them":

| Agent | Vetted tool(s) | What it's allowed to do |
| --- | --- | --- |
| `0_planner` | `jira-api.js` | Create/search Jira tickets |
| `1/2/3` coders | `test_python.sh` | Lint + syntax-check Python |
| `3_glue_verifier` | `aws_auth.sh`, `glue_run.sh` | Auth to AWS dev, run/monitor a Glue job |
| `4_tf_coder` | `validate_tf.sh` | `terraform fmt` + `validate` |
| `5_secops_reviewer` | `scan_secrets.sh` | Snyk scan for secrets/vulns |
| `6_git_operator` | `create_pr.sh` | Branch, commit, open PR |
| `7_deploy_trigger` | `harness_api.sh` | Trigger Harness CI/CD |
| `8/9` analytics | `run_sql.sh`, `deploy_report.sh` | Snowflake SQL, PowerBI deploy |

This is least privilege applied to *tools*: a small, explicit, reviewable allowlist instead of a broad, auto-discovered one.

---

## 4. A dedicated security gate in the DAG

Security isn't only a sandboxing property of *how* agents run — it's also a **node in the graph**. `5_secops_reviewer` sits between artifact generation and any PR/deploy, and reviews every artifact for:

- **Hardcoded secrets** (target: catch **100%**) — via Snyk (`scan_secrets.sh`).
- **IAM over-privilege** (target: catch **>90%**) — least-privilege violations in the generated Terraform.
- **SQL injection** and **data-exfiltration risk**.

Its verdict is a machine-readable `status: approved | rejected`. A `rejected` verdict is a **hard HITL stop** — the pipeline will not proceed to a PR until a human resolves the finding and explicitly continues. Security failures are handled by human judgment, never by an automatic retry.

---

## 5. Secrets and configuration

Sensitive configuration is supplied via environment variables, never committed:

- `HARNESS_API_KEY`, `HARNESS_ACCOUNT`, `HARNESS_PROJECT` — Harness CI/CD auth.
- `PBI_WORKSPACE`, `PBI_TENANT_ID` — PowerBI deployment target.

The AWS dev account used for Glue verification (`721527036298`) is reached through `aws_auth.sh` rather than static credentials in code — which is exactly the kind of thing the SecOps gate is built to catch if it ever regressed.

---

## 6. Threat-model summary

| Threat | Mitigation |
| --- | --- |
| Agent reads/writes outside its job | `--add-dir` allowlist; nothing visible unless granted |
| Agent reaches unintended capabilities | `--disable-builtin-mcps`; only vetted `tools/` scripts |
| Hardcoded secret reaches a deploy | Dedicated SecOps node + Snyk scan; hard HITL stop on reject |
| Over-privileged IAM in generated TF | SecOps IAM review (>90% catch target) |
| Bad code reaches production | Executed-in-dev verification (steps 1–3) *before* PR |
| Runaway retries / cost | Bounded verifier loop (max 5) + per-agent timeout |
| Unattended mistake at a critical moment | HITL gates at spec approval, data-cleaning review, and security rejection |

---

## 7. What a reviewer should take away

> Every agent runs with the minimum filesystem it needs and a hand-picked set of vetted tools — I deliberately turned *off* the broad built-in MCP surface in favor of an explicit allowlist. On top of that, security is its own node in the pipeline with a hard human gate. This is the "handled production reliability before" story: safe-by-construction, not safe-by-hope.
