# agent-orchestration

Two [Claude Code](https://claude.com/claude-code) skills that split a coding task between a
main session and cheaper sub-agents, so the expensive model thinks and the cheaper ones type.

| Skill | Main session | Sub-agents | Use when |
|---|---|---|---|
| [`opus-orchestrator`](skills/opus-orchestrator/SKILL.md) | Opus investigates, writes the spec, reviews the diff | Sonnet or Haiku implements and applies review fixes | Hard problems: root-cause analysis, reverse engineering, anything where the plan matters more than the typing |
| [`sonnet-orchestrator`](skills/sonnet-orchestrator/SKILL.md) | Sonnet splits the task, routes pieces, integrates | Sonnet for complex pieces, Haiku for simple ones | Larger, well-understood tasks that split into many independent pieces |

## Install

Skills are folders with a `SKILL.md`. Copy the ones you want into your personal skills folder:

```bash
# macOS / Linux
cp -r skills/opus-orchestrator skills/sonnet-orchestrator ~/.claude/skills/
```

```powershell
# Windows (PowerShell)
Copy-Item -Recurse skills\opus-orchestrator, skills\sonnet-orchestrator "$HOME\.claude\skills\"
```

Or put them in a project's `.claude/skills/` folder to share them with that project only.
Restart Claude Code (or start a new session) so it picks them up.

## Use

Invoke a skill by name and describe the task:

```
/opus-orchestrator Fix the crash in the save-game loader; it only happens on the second load
/sonnet-orchestrator Rename the config keys across the repo and update the docs and tests
```

Include the goal, the files or area involved, constraints (folders not to touch, what may be
committed), and how to build or verify. The agents only know what the spec tells them.

### How `opus-orchestrator` works

1. **Investigate** (main session): read the code, find the root cause, write a spec complete
   enough that the implementer makes no design decisions.
2. **Implement** (Sonnet or Haiku): make only the specified changes, build, report the diff.
3. **Review** (main session): read the real diff and build output, list required changes.
4. **Fix**: send the list back to an agent, re-review until clean, then commit.

### How `sonnet-orchestrator` works

1. **Decompose** the task and score each piece 1-10 for complexity.
2. **Delegate**: 1-3 goes to Haiku, 6-10 to Sonnet, 4-5 to Haiku only if the spec covers every
   edge case. When in doubt, Sonnet.
3. **Sub-delegate**: a Sonnet sub-agent may split its piece and spawn Haiku agents.
4. **Integrate**: read the real diff, check it against the requirements, send fixes back.

## Limits

- **Depth 3 at most**: main session, then a Sonnet sub-agent, then a Haiku helper. Haiku
  helpers never spawn agents.
- **At most 4 agents running at once** per spawner.
- **Parallel agents must touch different files**, or each gets its own worktree.
- Haiku and Sonnet agents never commit unless the spec says so.

These limits are written into the prompts, not enforced by Claude Code. Every agent prompt starts
with its depth ("You are depth 3: never spawn agents"). In a test, a depth-3 Haiku told to
spawn a helper refused. A prompt that omits the depth line gets no such protection.

## Notes

- Model names (`opus`, `sonnet`, `haiku`) are the aliases the Agent tool accepts. Adjust the
  skills if you want different models or limits.
- Sub-delegation needs the Agent tool to be available to the Sonnet sub-agent. If it is not,
  the main session launches the Haiku agents itself.
- Check `git diff` yourself after a run; the skills tell the main session to do the same.
