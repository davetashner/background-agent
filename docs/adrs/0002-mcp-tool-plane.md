# ADR-0002: MCP-fronted tool plane for multi-client planners

**Status:** Accepted
**Date:** 2026-04-22

## Context

The user wants chat on multiple platforms (Discord, Slack, GitHub
comments) to feed the same planner logic. Naive approaches either (a)
duplicate the planner LLM and its tools per platform, or (b) inline
credentials in each chat adapter.

## Decision

Chat adapters are thin translators. They forward a normalized
`ChatMessage` to a single planner service. The planner's tools
(`code.search`, `code.read`, `dispatch.request`, `audit.log`, etc.) are
exposed via a single **MCP tool server** that owns credentials.

```
[Discord adapter] ─┐
[Slack adapter]   ─┼─> [planner LLM] ─── MCP ──> [MCP tool server] ── creds
[GitHub adapter]  ─┘
```

The planner LLM has no direct network access to GitHub or AWS; every
privileged action goes through the MCP server, which is the single place
to enforce policy.

## Consequences

- New chat platforms cost one small adapter, no planner changes.
- Credentials live in one service. Rotation is localized.
- All tool calls flow through one place — easy to audit and rate-limit.
- A bug in the MCP server is a bigger blast radius than a bug in one
  adapter. We compensate with restricted-mode PodSecurity and tight
  NetworkPolicy.

## Alternatives considered

**Per-adapter LLM.** Rejected — 3x the LLM cost and 3x the prompt drift.

**LLM embedded in each adapter with shared tool libraries.** Rejected —
credentials end up in every adapter's container; bigger blast radius than
the MCP-server design.

**Direct LLM → GitHub API calls.** Rejected — impossible to enforce
policy once credentials are in the LLM's context.
