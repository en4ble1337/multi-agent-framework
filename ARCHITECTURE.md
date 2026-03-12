# Multi-Agent Workspace Architecture

> A neutral reference architecture for running autonomous agents with private workspaces, a shared coordination surface, and human supervision through chat.

## Goals

- Keep each agent independently replaceable.
- Make shared state inspectable without custom tooling.
- Keep ownership boundaries obvious.
- Let humans direct work and review outcomes in plain sight.
- Avoid introducing infrastructure that the workload does not need yet.

## Core Principles

1. Independence over central orchestration.
Each agent should continue working even if another agent fails.

2. Private by default.
Secrets, memory, browser state, and local project context stay with the owning agent.

3. Shared by convention.
Status, notes, research, and cross-agent artifacts live on a common surface.

4. Chat as the control plane.
Humans give direction and review outcomes through visible conversations.

5. Simple primitives first.
Files, folders, and explicit ownership rules are usually enough for small and medium deployments.

## Reference Topology

```text
                    Human operator
                          |
                review, directives, escalation
                          |
                      Chat surface
               shared rooms + direct messages
                          |
      +-------------------+-------------------+
      |                   |                   |
      v                   v                   v
   Agent A             Agent B             Agent C
 private area         private area         private area
      \                   |                   /
       \                  |                  /
        +-----------------+-----------------+
                          |
                     Shared surface
          announcements/ knowledge/ agents/ projects/
                          |
              explicit locking for shared writes
```

## Core Components

### Agent unit

An agent unit is one independently operated runtime with:

- a private workspace
- local memory and identity files
- private secrets and tool configuration
- optional browser or external service sessions
- one or more assigned responsibilities

The isolation boundary can be a virtual machine, a container, a user account, or another environment with clear ownership.

### Private workspace

The private workspace belongs to one agent only. Other agents should not write there, and ideally should not have read access either.

Typical private contents:

- `AGENTS.md` for local operating rules
- `IDENTITY.md` or equivalent metadata
- `MEMORY.md` or other persistent notes
- `TOOLS.md` or local secrets
- `projects/` for local work

### Shared surface

The shared surface is where public coordination happens. It can be a network filesystem, a synced folder, a mounted volume, or another storage layer with predictable visibility.

Typical shared contents:

- announcements
- standups or checkpoints
- shared research
- agent roster and status
- cross-agent project folders

### Chat surface

The chat surface is where humans and agents interact in real time or near real time. It should support:

- a direct path from the operator to one agent
- a broadcast path from the operator to many agents
- a shared room for group coordination
- optional logs or audit channels

## Suggested Filesystem Layout

One private workspace per agent:

```text
/private/agent-a/
  AGENTS.md
  IDENTITY.md
  MEMORY.md
  TOOLS.md
  HEARTBEAT.md
  projects/
```

One shared coordination surface:

```text
/shared/
  announcements/
    current.md
  conference/
    standup/
      YYYY-MM-DD.md
    decisions/
  knowledge/
    playbooks/
    research/
    lessons-learned.md
  agents/
    roster.md
    agent-a/
      PROFILE.md
      status.md
      outbox/
    agent-b/
      PROFILE.md
      status.md
      outbox/
  projects/
    project-name/
      README.md
      tasks.md
      shared-data/
```

## Ownership Rules

| Path or file type | Owner model | Notes |
|---|---|---|
| Private workspace files | Single agent | Keep secrets and local memory here. |
| `/shared/agents/<name>/status.md` | Single agent | Public status, written by the named agent. |
| `/shared/agents/<name>/outbox/` | Single agent writes, others read | Useful for asynchronous requests. |
| `/shared/conference/standup/*.md` | Multi-writer | Use append rules plus locking. |
| `/shared/knowledge/*` | Multi-writer | Shared research and reusable notes. |
| `/shared/projects/*` | Depends on project | Define ownership before work starts. |

Rule of thumb:

- If exactly one agent writes a file, keep ownership explicit and simple.
- If more than one agent may write, use a documented locking rule.

## Common Shared File Formats

### Status file

```markdown
# agent-a status
Updated: YYYY-MM-DD HH:MM UTC
State: active
Current task: brief description
Blockers: none
Next action: brief description
```

### Daily checkpoint

```markdown
## agent-a
Time: HH:MM UTC
Yesterday: short summary
Today: short summary
Blockers: none
```

### Agent-to-agent message

```markdown
# message for agent-b
From: agent-a
Date: YYYY-MM-DD HH:MM UTC
Priority: normal
Read: false

Request or context goes here.
```

## Chat Model

The exact product does not matter. The operating pattern does.

Recommended room types:

- a direct room for each agent and the operator
- a shared coordination room for all agents
- an announcements room used for broadcast instructions
- an optional logs room per agent

Recommended permission model:

- each agent can write in its own direct room
- all agents can read announcements
- all agents can read and write in the shared coordination room
- agents do not write in another agent's private room

## Communication Patterns

| Pattern | Medium | Use case |
|---|---|---|
| Operator to one agent | Direct room or direct message | Specific instructions, review, clarifications |
| Operator to all agents | Broadcast room plus shared file | Policy changes, global announcements |
| Agent to operator | Direct room or alert channel | Status, blockers, escalation |
| Agent to agent async | Shared outbox file | Non-urgent requests, handoffs, references |
| Agent to group | Shared coordination room | Time-sensitive group coordination |

## Agent Lifecycle

### Add a new agent

1. Create an isolated environment for the new agent.
2. Create its private workspace from a clean template.
3. Attach the shared coordination surface.
4. Fill in identity, scope, and local instructions.
5. Register the agent in `/shared/agents/roster.md`.
6. Create `PROFILE.md`, `status.md`, and `outbox/` under its shared directory.
7. Grant access to the relevant chat rooms.
8. Verify that it can read announcements, update status, and send a test message.

### Decommission an agent

1. Pause new work assignments.
2. Archive or remove the agent's chat rooms.
3. Remove the active roster entry.
4. Archive the agent's shared artifacts if needed.
5. Shut down or revoke the isolated runtime.

## Scaling Guidance

| Scale | Guidance |
|---|---|
| 1 to 3 agents | Keep the model simple and manual. |
| 4 to 10 agents | Tighten ownership rules and formalize handoff patterns. |
| 10+ agents | Re-evaluate storage, observability, and whether some work should move to queues or services. |

Questions to ask before scaling further:

- Are agents waiting on the same shared files too often?
- Is the operator losing visibility into active work?
- Are handoffs too slow or too ambiguous?
- Do some tasks deserve stronger automation than a file-based pattern can provide?

## Security Boundaries

- Keep secrets out of the shared surface.
- Limit write access to shared paths wherever practical.
- Prefer isolated browser sessions and credentials per agent.
- Keep shared artifacts public-by-design and low sensitivity.
- Treat agents as replaceable units with a bounded blast radius.

## Failure Model

This architecture assumes failures will happen.

- If one agent fails, other agents should still operate.
- If shared storage is unavailable, coordination degrades immediately.
- If chat is unavailable, agents can continue local work but lose supervision and quick coordination.
- If a multi-writer file is updated without locking, data loss is possible.

Design for recovery:

- re-create agents from a clean template
- keep shared artifacts simple enough to inspect manually
- separate durable knowledge from local scratch space

## Concurrency

Any shared file with multiple possible writers needs an explicit update protocol. The default pattern in this repo is a lock directory created next to the target file before writing.

See [LOCKFILE-PROTOCOL.md](LOCKFILE-PROTOCOL.md) for the full contract and reference script.

## Adoption Checklist

- Private workspace per agent
- Shared surface for public coordination
- Clear single-owner versus multi-writer rules
- Chat path for operator oversight
- Documented lock protocol for shared writes
- Agent instructions copied from the shared template
