# pi-herdr-subagents

Synchronous subagent delegation for [pi](https://github.com/badlogic/pi-mono) in [herdr](https://herdr.dev). Each child runs in its own Herdr pane, but the parent tool call waits for the child result.

## How it works

```text
parent calls subagent → child runs in Herdr pane → tool result returns → parent continues
```

Several `subagent` calls in one assistant tool-call batch start concurrently. Pi waits for every tool result before requesting the next parent response.

```typescript
subagent({ name: "Scout: auth", agent: "scout", task: "Map the authentication module." });
subagent({ name: "Scout: data", agent: "scout", task: "Map database access patterns." });
// Both children run concurrently. Both results are returned before the parent continues.
```

A pending-work widget is shown above the editor while foreground calls run. It is local UI only; the extension does not send status or completion steer messages.

## Install

```bash
pi install npm:pi-herdr-subagents
```

Run Pi inside Herdr. Herdr is required (`HERDR_ENV=1` and the `herdr` CLI must be available).

```bash
herdr
pi
```

## Tools

| Tool | Description |
| --- | --- |
| `subagent` | Start an autonomous child in a Herdr pane and wait for its result. |
| `subagent_resume` | Resume a child session and wait for its result. |
| `subagent_interrupt` | Send Escape to a running Pi-backed child turn. |
| `subagents_list` | List named agent definitions. |

`/subagent <agent> <task>` asks the parent to make a synchronous `subagent` call. `/plan` and `/iterate` are not provided.

Child tool results include the child ID, name, task, agent, exit code, elapsed duration, session and launch-script paths, resolved runtime, and one of `completed`, `failed`, `cancelled`, or `needs_help`.

## Cancellation and help

Cancelling the parent tool call closes the child pane and clears its local tracking entry. A cancelled result includes its session path when available.

A child can call `caller_ping` to request parent input. That call returns a `needs_help` result directly to the parent. The parent can then call `subagent_resume` with the returned session path and follow-up message.

## Bundled agents

| Agent | Role |
| --- | --- |
| `scout` | Fast codebase reconnaissance. |
| `worker` | Implements a focused task. |
| `reviewer` | Reviews code for correctness and security. |
| `visual-tester` | Tests web UIs through Chrome CDP. |

All supported agents must be autonomous and declare `auto-exit: true`. A named agent without that setting is rejected before a pane is created. Bare spawns are autonomous by default. Pi-backed children always receive `PI_SUBAGENT_AUTO_EXIT=1`; `subagent_done` remains available for explicit completion.

## Agent definitions

Place agent markdown files in `.pi/agents/` or `~/.pi/agent/agents/`. Discovery precedence is project, global, then bundled.

```markdown
---
name: researcher
description: Finds facts for a focused task
tools: read, bash
spawning: false
auto-exit: true
---

You are a research specialist. Finish the assigned task and summarize the result.
```

Supported frontmatter includes `name`, `description`, `model`, `thinking`, `tools`, `skills`, `session-mode`, `spawning`, `deny-tools`, `auto-exit`, `cwd`, and `disable-model-invocation`. `interactive` is not supported.

`session-mode` may be `standalone`, `lineage-only`, or `fork`. A `fork: true` tool argument overrides the agent default. It changes context inheritance, not autonomous completion.

## Development

```bash
npm run lint
npm test
PI_TEST_MODEL="deepseek/deepseek-v4-flash" PI_TEST_TIMEOUT=180000 npm run test:integration
```

The integration suite launches real Pi sessions in Herdr and can take several minutes.
