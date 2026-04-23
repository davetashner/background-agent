# Design: background-agent

## Goals

1. A human can describe a coding task in chat (Discord first; Slack and GitHub
   comments later) and collaboratively refine it with an LLM planner.
2. When the human approves a hydrated plan, a sandboxed agent produces a PR
   for human review.
3. The agent cannot open a PR without (a) passing an opaque `validate` tool
   and (b) proving CI is green on the pushed commit.
4. Every action touching code or credentials is attributable and auditable.
5. A failed run is observable: the planner (and the human) learn *why* the
   agent could not finish.

## Non-goals (prototype phase)

- Auto-merge. PRs are always human-reviewed.
- Agent-to-agent orchestration beyond planner → implementer.
- Production-grade multi-tenant isolation. The prototype assumes a single
  org with a small number of approved operators.
- Cost optimization. Correctness and safety first.

## Components

### Chat adapters

Thin processes that translate chat-platform events to/from a normalized
`ChatMessage` envelope and call the planner. They hold no LLM logic and no
repo credentials. One adapter per platform.

- Discord adapter (v1).
- Slack adapter (v2).
- GitHub comment adapter (v3).

### Planner service

The conversational LLM. One planner process can be shared across chat
platforms because the adapters normalize their inputs. The planner:

- Reads code via MCP tools.
- Refines the task through dialogue with the human.
- Produces a hydrated implementation prompt.
- **Cannot** commit, push, open PRs, or mutate the target repo in any way.
- Cannot dispatch to the implementer on its own; dispatch requires an
  explicit human approval message from the chat session.

### MCP tool server

A single MCP server fronts the tools the planner (and future clients) can
call:

- `code.search(repo, query)` — read-only code search across allowlisted
  repos.
- `code.read(repo, path, rev)` — read a file at a revision.
- `repo.list_allowlisted()` — enumerate repos the planner may discuss.
- `plan.submit(prompt_yaml)` — validate the shape of a hydrated prompt.
- `dispatch.request(prompt_yaml, human_approval)` — send the prompt to the
  dispatcher *only if* `human_approval` is a valid signed approval token
  from an authorized human.
- `audit.log(event)` — append to audit log.

The MCP server owns credentials. Chat adapters and the planner LLM never
see raw tokens.

### Dispatcher

Receives approved hydrated prompts. Responsibilities:

1. Persist the prompt and approval token to the audit log (before anything
   else).
2. Create an ephemeral Kubernetes namespace for the run.
3. Mint a GitHub App installation token scoped to the single target repo.
4. Launch one implementer pod with the prompt and credentials mounted.
5. Stream pod logs to the audit sink.
6. On pod exit, post the outcome back to the originating chat thread.

### Implementer pod

A Kubernetes pod running Claude Code with:

- The hydrated prompt as the user message.
- A system prompt that is logged verbatim before pod start.
- A short-lived GitHub App token for the single target repo.
- A `validate` CLI on `$PATH` that returns only `pass`/`fail` + abstracted
  fix hints.
- Claude Code hooks enforcing the PR gate and loop budgets.
- A tool allowlist file read at startup; deviations are blocked by hooks.
- No inbound ingress; no interactive session possible.

The pod runs exactly one Claude Code session and exits.

### Validate service

Opaque verifier. Takes a working tree and returns `{ok: bool, hints: [...]}`
where hints are abstracted so the implementer cannot tell which checker ran
(see ADR-0004).

### Audit log

Append-only sink. Records:

- Every chat message involving the planner (with platform IDs).
- Every MCP tool call with arguments and results.
- Every human approval token with signer identity.
- Every implementer pod's system prompt, user prompt, tool-call stream,
  validate result, CI result, and final outcome.

Store: S3 with Object Lock (Compliance mode) for immutability.

## Flow

```
+--------+   chat    +---------+   mcp    +----------+
| human  | <-------> | adapter | <------> | planner  |
+--------+           +---------+          +----------+
                                             |
                                             | mcp: tool calls
                                             v
                                        +----------+
                                        | mcp tool |
                                        |  server  |
                                        +----------+
                                             |
                      human approval         | dispatch.request
                      (signed in chat)       v
                                        +------------+
                                        | dispatcher |
                                        +------------+
                                             |
                                    spawn    | k8s api
                                             v
                                    +------------------+
                                    | implementer pod  |
                                    |  (Claude Code)   |
                                    +------------------+
                                       |         |
                                   git push    validate
                                       |         |
                                       v         v
                                  +-------+  +---------+
                                  |  CI   |  | validate|
                                  +-------+  +---------+
                                       \       /
                                        \     /
                                         hook gate
                                            |
                                            v
                                       +---------+
                                       |   PR    |
                                       +---------+
                                            |
                                            v
                                     human reviewer
```

## Identity model

- **Humans** authenticate to chat via the platform's native auth; the MCP
  server maintains an `authorized_operators` list keyed by platform ID.
- **Planner** runs as a K8s ServiceAccount with IRSA to an IAM role with
  read-only S3 (audit log read), no GitHub credentials.
- **MCP server** holds the GitHub App private key in AWS Secrets Manager.
- **Dispatcher** holds the K8s API token and the GitHub App key (for
  minting per-run installation tokens).
- **Implementer pod** gets only a short-lived installation token scoped to
  one repo with `contents:write` and `pull_requests:write`, valid for the
  pod's lifetime.

## Tool allowlist

A YAML file at `config/implementer-tools.yaml`:

```yaml
bash:
  - git
  - rg
  - cat
  - ls
  - find
  - node
  - npm
  - python
  - pytest
claude_code_tools:
  - Read
  - Edit
  - Write
  - Grep
  - Glob
  - Bash
custom:
  - validate
```

Modifications require PR approval from two CODEOWNERS. See ADR-0005.

## Open questions

- Sourcegraph vs. local-clone code search for the planner.
- Whether the MCP server and dispatcher collapse into one service for v1.
- Audit log retention period (default: 7 years; revisit).
