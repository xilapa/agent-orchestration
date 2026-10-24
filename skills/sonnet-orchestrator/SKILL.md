---
name: sonnet-orchestrator
description: Sonnet-based agent orchestration with adaptive model selection based on task complexity
version: '1.1.0'
---

# Sonnet Orchestrator

The main session (Sonnet 5.5) splits a task, routes each piece to a Sonnet 5.5 or Haiku 5.5
agent by complexity, then reviews and integrates the result.

## Rules

- Models: Sonnet = `claude-sonnet-5-5`, Haiku = `claude-haiku-5-5` (Agent tool aliases `sonnet`, `haiku`).
- Max depth 3: **Sonnet Orchestrator (depth 1) → Sonnet Subagent (2) → Haiku Subagent (3)**.
  A Haiku the orchestrator launches directly is depth 2.
- Max 4 agents running at once per spawner: the orchestrator runs at most 4, and each Sonnet
  subagent runs at most 4 Haiku agents.
- These limits are instructions, not enforced settings. Nothing else enforces them.

## Complexity rubric (1-10)

| Score | Meaning | Route |
|---|---|---|
| 1-3 | One file or function, known pattern, easy to verify (rename, comment, small fix, config, docs) | Haiku |
| 4-5 | 2-5 files, some cross-file dependencies | Haiku if the spec covers every edge case, otherwise Sonnet |
| 6-10 | 5+ files, non-obvious dependencies, architecture, security, performance, novel or high-risk work | Sonnet |

When in doubt, take Sonnet. Debugging and design decisions stay with Sonnet.

## Workflow

### 1. Decompose
- Break the task into subtasks, note dependencies, score each one.
- Parallel subtasks must touch different files. If two must touch the same file, run them one
  after the other, or give each its own worktree (`isolation: worktree`).

### 2. Delegate
- Give each agent a complete spec: goal, files, exact behaviour, acceptance criteria, how to
  build and verify, and the project's standing rules (what may be committed, folders not to touch).
- Every agent prompt must start with its depth: "You are depth 2 of 3: you may spawn Haiku
  agents" or "You are depth 3: never spawn agents".
- Tell every agent: make only the specified changes, build, report a diff summary and anything
  that did not match the spec, never commit unless told, and ask instead of guessing when the
  spec is unclear.

### 3. Sub-delegation (Sonnet subagents only)
- A Sonnet subagent may split its task into independent pieces and spawn Haiku agents in
  parallel. Pieces touch different files and each Haiku gets a complete mini-spec.
- Haiku agents never spawn agents and never commit. Sonnet subagents never spawn other Sonnet agents.
- The Sonnet subagent waits for every Haiku report (they arrive as hand-back messages, not
  tool results), then integrates, builds, and owns its result. Its report lists which pieces
  went to Haiku.
- If a Sonnet subagent has no Agent tool, the orchestrator launches the Haiku agents itself.

### 4. Integrate
- Read the real diff and build output yourself; do not rely on agent summaries.
- Check it against the original requirements and the project's standing rules.
- Send concrete fixes back to the same agent (SendMessage), or a new one with full context,
  and review again until clean.
- Report: what changed, what was verified, which model took each piece and why, and anything
  the user should test.
