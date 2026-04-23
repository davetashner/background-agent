# CLAUDE.md — background-agent

This file tells Claude Code (and any other agent) how to work in this repo.
User-global rules in `~/.claude/CLAUDE.md` also apply; the rules here refine
them for this project.

## What this repo is

A prototype of a security-first background coding agent system. See
`README.md` for the component breakdown. The key invariants below are
**load-bearing for the security posture** — treat them as non-negotiable
unless the user explicitly overrides.

## Core invariants (do not relax without the user's OK)

1. **Planner is read-only.** The planner service and its MCP tools must never
   gain the ability to commit, push, or open PRs. If a task seems to require
   that, push back.
2. **Implementer accepts no human input.** The implementer agent's only
   input is the hydrated prompt from the dispatcher. Never add a UI, webhook,
   or chat channel that lets a human talk to it mid-run.
3. **`validate` is opaque.** The implementer must not be able to introspect
   what `validate` checks. Don't add logging, error messages, or tool output
   that leaks validator internals to the implementer's context.
4. **PR gate is hook-enforced, not prompt-enforced.** The "must pass validate
   and CI before opening a PR" rule lives in Claude Code hooks, not in the
   system prompt. Prompts can be jailbroken; hooks can't.
5. **Tool allowlist requires two human approvals.** The allowlist file is
   owned by CODEOWNERS requiring 2 reviews. Do not bypass this by inlining
   commands elsewhere.
6. **Audit log is append-only.** Every system prompt from planner to
   implementer is logged before dispatch. Never add a code path that skips
   the log.
7. **Binary outcomes.** An implementer run ends with either a PR opened or a
   structured failure report — nothing in between. No partial commits, no
   "draft for later" states.
8. **Loop budgets are mandatory.** Every implementer run has enforced
   wall-clock, token, and validate-call budgets. Do not add "just this once"
   escape hatches.

## Working agreements

- **Design before code.** This project is in design phase. Prefer updating
  the design doc and asking the user before scaffolding new services.
- **Worktrees for new work** (see global CLAUDE.md).
- **Beads for task tracking** (see global CLAUDE.md). If there is no
  `.beads/` directory yet, ask before initializing one.
- **No secrets in repo.** Ever. Use K8s secrets or a secret manager.
- **No new dependencies with known critical CVEs.** Check before adding.

## Things to ask the user before assuming

- Target deployment (real cloud K8s vs. local `kind` vs. Docker Compose
  prototype with aspirational K8s manifests).
- Which chat adapter(s) to build first.
- Whether the `validate` tool should be a stub or a real verifier for v1.
- Language/runtime for each service (not yet decided).
- Identity model: what GitHub App / bot account opens PRs; who is allowed
  to request plans.

## Style

- Terse code, no over-abstraction, no speculative flexibility.
- Comments only when the *why* is non-obvious.
- Threat-model comments are welcome when they explain a security choice
  that would otherwise look like over-engineering.


<!-- BEGIN BEADS INTEGRATION v:1 profile:minimal hash:ca08a54f -->
## Beads Issue Tracker

This project uses **bd (beads)** for issue tracking. Run `bd prime` to see full workflow context and commands.

### Quick Reference

```bash
bd ready              # Find available work
bd show <id>          # View issue details
bd update <id> --claim  # Claim work
bd close <id>         # Complete work
```

### Rules

- Use `bd` for ALL task tracking — do NOT use TodoWrite, TaskCreate, or markdown TODO lists
- Run `bd prime` for detailed command reference and session close protocol
- Use `bd remember` for persistent knowledge — do NOT use MEMORY.md files

## Session Completion

**When ending a work session**, you MUST complete ALL steps below. Work is NOT complete until `git push` succeeds.

**MANDATORY WORKFLOW:**

1. **File issues for remaining work** - Create issues for anything that needs follow-up
2. **Run quality gates** (if code changed) - Tests, linters, builds
3. **Update issue status** - Close finished work, update in-progress items
4. **PUSH TO REMOTE** - This is MANDATORY:
   ```bash
   git pull --rebase
   bd dolt push
   git push
   git status  # MUST show "up to date with origin"
   ```
5. **Clean up** - Clear stashes, prune remote branches
6. **Verify** - All changes committed AND pushed
7. **Hand off** - Provide context for next session

**CRITICAL RULES:**
- Work is NOT complete until `git push` succeeds
- NEVER stop before pushing - that leaves work stranded locally
- NEVER say "ready to push when you are" - YOU must push
- If push fails, resolve and retry until it succeeds
<!-- END BEADS INTEGRATION -->
