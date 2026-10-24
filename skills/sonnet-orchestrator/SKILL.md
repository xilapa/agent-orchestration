---
name: sonnet-orchestrator
description: Sonnet-based agent orchestration with adaptive model selection based on task complexity
version: '1.2.0'
---

# Sonnet Orchestrator

Split every change into four phases. The main session (Sonnet) thinks; Sonnet and Haiku agents type. Pick the agent model per piece (see "Choosing the agent model").

## Limits
- Max depth 3: **Sonnet main session (1) → Sonnet implementer (2) → Haiku helper (3)**. A Haiku
  the main session launches directly is depth 2.
- Max 4 agents running at once per spawner (the main session, and each Sonnet implementer).
- These are instructions, not enforced settings, so every agent prompt must start with its
  depth: "You are depth 2 of 3: you may spawn Haiku agents" or "You are depth 3: never spawn
  agents" (a Haiku launched by the main session is also told "never spawn agents").

## 1. Investigate and decompose (Sonnet, main session)
- Read the code, logs, profiles, dumps and docs needed to understand the problem.
- Find the root cause or the exact design; decide the approach.
- Break the task into pieces, note dependencies, score each piece for complexity.
- Parallel pieces must touch different files. If two must touch the same file, run them one
  after the other, or give each its own worktree (`isolation: worktree`).
- Write a precise spec per piece: files, functions, exact behaviour, edge cases, how to build,
  how to verify, and the project's standing rules (commit author, what may be committed,
  folders that must not be touched, docs to update).

## 2. Implement (agents)
- Launch an Agent per piece with the model chosen below and its spec as the prompt.
- Tell each to: make only the specified changes, build, report the diff summary, build
  output and anything that did not match the spec. It must not commit unless the spec says
  so, and must ask (in its report) instead of guessing when the spec is unclear.
- Wait for them (`run_in_background: false` when the next step depends on them).
- **Sub-delegation (Sonnet implementers only):** a Sonnet implementer may split its piece into
  independent sub-pieces and spawn Haiku agents in parallel. Sub-pieces must touch different
  files, and each Haiku prompt must be a complete mini-spec. Haiku agents never commit and
  never spawn agents. Sonnet must wait for every Haiku report (they arrive as hand-back
  messages, not tool results), then integrate, build, and own its result. Its report must
  list which sub-pieces went to Haiku. If the Sonnet agent has no Agent tool, the main
  session launches the Haiku agents itself.

## 3. Review (Sonnet, main session)
- Read the actual diff and build result yourself; do not trust the agents' summaries.
- Check correctness against the spec and the root cause, cross-piece integration, failure
  paths, settings defaults, docs, and the project's standing rules.
- List concrete required changes, if any.

## 4. Fix (agent), then re-review
- If the review found problems, send them to an agent as an exact list of changes (continue
  the same agent with SendMessage when possible, otherwise a new one with the full context).
  Use the same model as the implementation, or Sonnet if the review found the agent misread
  the spec.
- Review again (phase 3). Repeat until clean.
- Then commit (by an agent or the main session, per the project's rules) and report to the
  user: what changed, what was verified, which model took each piece and why, and what the
  user should test.

## Choosing the agent model
Score each piece 1-10 after writing its spec, from how much judgement the implementer still
needs. When in doubt, take Sonnet.

| Score | Meaning | Model |
|---|---|---|
| 1-3 | One file or function, known pattern, easy to verify (rename, comment, small fix, config, docs, run a build or test) | Haiku (alias `haiku`) |
| 4-5 | 2-5 files, some cross-file dependencies | Haiku if the spec covers every edge case, otherwise Sonnet |
| 6-10 | 5+ files, non-obvious dependencies, concurrency, security, performance, novel or high-risk work | Sonnet (alias `sonnet`) |

- Never delegate investigation, root-cause analysis, design decisions or review; those stay in
  the main session. This includes debugging and profiling.
- Small, purely mechanical follow-ups (a one-line config edit, a doc note) may be done
  directly when delegating would cost more than it saves; say so in the report.
