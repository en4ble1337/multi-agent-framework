# The Office: Multi-Agent Framework

![License: MIT](https://img.shields.io/badge/license-MIT-yellow.svg)
![Docs](https://img.shields.io/badge/type-reference%20docs-blue)
![Infra](https://img.shields.io/badge/infrastructure-agnostic-brightgreen)
![OpenClaw](https://img.shields.io/badge/OpenClaw-compatible-black)

![The Office architecture](docs/images/office-architecture.png)

Documentation-first reference architecture for running multiple autonomous AI agents that keep private workspaces, share a common knowledge surface, coordinate through plain files and chat, and scale horizontally without adding a heavy control plane.

This repo is opinionated about the pattern, not the infrastructure. The examples use OpenClaw, Discord, Proxmox, Ubuntu VMs, and Synology-backed NFS, but the core ideas also map cleanly to containers, cloud VMs, shared volumes, Slack, Matrix, Telegram, or a single host.

## Why This Exists

Most multi-agent setups reach for orchestration too early: brokers, task queues, service meshes, agent routers, custom protocols, and databases for problems that do not need them yet.

The Office takes a simpler position:

- Each agent is an independent worker with its own identity, memory, credentials, and scope.
- Shared state lives in plain files that every relevant agent can inspect.
- Humans supervise through chat, not through a custom dashboard.
- Cross-agent coordination is mostly asynchronous.
- Concurrency on shared files is handled with a lightweight lockfile convention.
- Scaling means adding another agent unit, not inventing another control plane.

## Architecture At A Glance

```text
                     Human / Boss
                           |
                DMs, directives, escalation
                           |
                       Chat layer
             (Discord, Slack, Matrix, etc.)
                           |
      +--------------------+--------------------+
      |                    |                    |
      v                    v                    v
   Agent A              Agent B              Agent C
  private desk         private desk         private desk
      \                    |                    /
       \                   |                   /
        +------------------+------------------+
                           |
                       Shared drive
      knowledge/  conference/  agents/  projects/
                           |
                 lockfile protocol for
                  multi-writer documents
```

## Core Principles

- Independence over orchestration. Agents should keep working when another agent goes down.
- Files over protocols. Shared state should be inspectable without a custom client.
- Private by default. Identity, memory, credentials, and local project context stay with the agent.
- Shared by convention. Public status, research, cross-agent notes, and group artifacts live on the shared surface.
- Chat as the operations layer. Humans give direction, agents report status, and coordination stays visible.
- Boring infrastructure wins. Use components that are easy to reason about before adding more machinery.

## Who This Is For

- Builders running two or more semi-autonomous agents with different roles.
- People who want agents to share knowledge without standing up a database or broker.
- Operators who need a pattern they can adapt to VMs, containers, cloud instances, or one machine.
- Teams using OpenClaw or any other agent runtime that can read files and post messages.

## What This Is Not

- Not an SDK, framework, or install script.
- Not a requirement to use Proxmox, NFS, Discord, or OpenClaw.
- Not a claim that every multi-agent problem should be solved with files.
- Not a turnkey production platform with automation included.

## Repo Map

| File | Purpose |
|---|---|
| [ARCHITECTURE.md](ARCHITECTURE.md) | Core concepts, operating model, shared layout, communication patterns, scaling, and security. |
| [LOCKFILE-PROTOCOL.md](LOCKFILE-PROTOCOL.md) | How to prevent agents from clobbering shared files during concurrent writes. |
| [AGENT-INSTRUCTIONS-TEMPLATE.md](AGENT-INSTRUCTIONS-TEMPLATE.md) | Copy-paste operating instructions for a new agent. |
| [EXAMPLES.md](EXAMPLES.md) | One concrete implementation plus valid alternatives for different environments. |

## Start Here

1. Read [ARCHITECTURE.md](ARCHITECTURE.md) to understand the model.
2. Read [EXAMPLES.md](EXAMPLES.md) to map the model to your environment.
3. Read [LOCKFILE-PROTOCOL.md](LOCKFILE-PROTOCOL.md) before letting multiple agents write to the same files.
4. Copy [AGENT-INSTRUCTIONS-TEMPLATE.md](AGENT-INSTRUCTIONS-TEMPLATE.md) into each agent workspace and adapt it to your channels, directories, and operating rules.

## Adapt The Pattern, Not The Stack

The Office is a set of stable concepts that can be implemented in different ways:

| Concept | This repo's reference example | Other valid choices |
|---|---|---|
| Agent isolation | One VM per agent | Containers, one process per agent, cloud instances |
| Shared writable surface | NFS share | SMB, Syncthing, Docker volume, EFS, local folder on one host |
| Human coordination | Discord | Slack, Matrix, Telegram, email, internal chat |
| Agent runtime | OpenClaw | Any agent runtime with filesystem and chat access |
| Shared write safety | `mkdir`-based lock directories | Another atomic lock convention appropriate to your storage |

If your environment preserves the same boundaries, the pattern still works:

- Agents have a private workspace.
- Agents have a shared surface for public coordination.
- Humans have a visible communication layer.
- Multi-writer files use a locking rule.
- Ownership rules are clear enough that agents do not step on each other.

## Why OpenClaw

This work was designed around [OpenClaw](https://github.com/openclaw/openclaw), but the repository intentionally keeps the architectural ideas portable. If your agents can maintain private memory, read and write shared files, and communicate through chat, you can use the same structure.

## License

This repository is licensed under the [MIT License](LICENSE).
