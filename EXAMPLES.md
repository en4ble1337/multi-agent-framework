# Examples

This document shows one concrete implementation of The Office and then maps the same pattern to other environments.

The point is not to copy one stack exactly. The point is to preserve the operating model:

- private workspace per agent
- shared writable surface for public coordination
- visible chat layer for humans and agents
- a lock rule for shared files with multiple writers

## Example: Proxmox + Ubuntu VMs + Synology NFS + Discord

This is the reference implementation the architecture was written from.

| Layer | Implementation |
|---|---|
| Hypervisor | Proxmox |
| Agent isolation | One VM per agent |
| Guest OS | Ubuntu |
| Agent runtime | OpenClaw node on each VM |
| Shared storage | Synology-backed NFS mounted at `/shared` |
| Human and agent chat | Discord |
| Browser-capable agents | Chrome installed on the VM where needed |
| Shared coordination model | Markdown files on `/shared` |
| Shared write protection | Lock directories created with `mkdir` |

### Why This Version Works Well

- VM boundaries are easy to reason about. Each agent gets its own machine identity, local credentials, browser session, and failure domain.
- NFS keeps the shared surface simple. Every agent sees the same files immediately.
- Discord works as the office floor. Humans can watch what is happening without building a dashboard.
- Proxmox makes cloning cheap. New agents can be created from a known-good template quickly.

### Example Mapping

| Office concept | Concrete implementation |
|---|---|
| Agent | One OpenClaw node running in its own Ubuntu VM |
| Desk | `~/.openclaw/workspace/` inside that VM |
| Shared drive | Synology NFS mount at `/shared` |
| Conference room | Discord shared channels |
| Boss's office | DM or per-agent Discord channel |
| Bulletin board | `/shared/conference/announcements.md` |
| Employee directory | `/shared/agents/roster.md` |

### Example Topology

```text
Proxmox host
  |
  +-- VM: Dwight
  |     private workspace
  |     OpenClaw
  |     /shared -> Synology NFS
  |
  +-- VM: Jim
  |     private workspace
  |     OpenClaw
  |     /shared -> Synology NFS
  |
  +-- VM: Pam
  |     private workspace
  |     OpenClaw
  |     /shared -> Synology NFS
  |
  +-- VM: Kelly
        private workspace
        OpenClaw
        /shared -> Synology NFS

Synology NFS
  |
  +-- /shared/company
  +-- /shared/conference
  +-- /shared/knowledge
  +-- /shared/agents
  +-- /shared/projects

Discord
  |
  +-- shared channels
  +-- one category per agent
```

## Other Valid Implementations

### One machine, multiple agents

Use separate directories or users for each agent and a single local shared folder.

This works well when:

- you are experimenting
- your agents do not need browser isolation
- you want minimal operational overhead

Tradeoff: failure isolation is weaker, and shared resource contention is higher.

### Containers instead of VMs

Run one container per agent, mount a shared volume, and keep per-agent secrets isolated.

This works well when:

- you want faster provisioning than full VMs
- browser automation is optional or container-safe in your environment
- you already operate Docker or Kubernetes

Tradeoff: container boundaries are lighter than VM boundaries, so browser sessions and host integration need more care.

### Cloud VMs instead of self-hosted hypervisors

Replace Proxmox with EC2, Hetzner, DigitalOcean, or any other VM provider.

This works well when:

- you want geographic distribution
- you want managed compute instead of home-lab hardware
- you need to scale up or down quickly

Tradeoff: cost is higher, and shared storage choices need more attention.

### Shared volume instead of NFS

Use EFS, SMB, Syncthing, CephFS, or a local mounted volume depending on your environment.

This works well when:

- NFS is not available
- your agents are containerized
- your infrastructure already provides a shared filesystem

Tradeoff: not every storage system gives you the same locking guarantees, so adapt the lock protocol accordingly.

### Slack, Matrix, or Telegram instead of Discord

Replace Discord with any chat surface that supports direct messages, shared rooms, and bot participation.

This works well when:

- your team already lives somewhere else
- Discord permissions do not fit your environment
- you need enterprise controls or another communication style

Tradeoff: channel structure, permissions, and mention behavior will differ, but the coordination pattern stays the same.

## Translation Guide

If your environment differs, map the pattern layer by layer:

| If you do not have... | Use... | Keep this invariant |
|---|---|---|
| Proxmox | Any VM manager, containers, or isolated processes | Each agent still needs a distinct private workspace and failure boundary. |
| NFS | Any shared storage with predictable visibility | Shared artifacts must be visible to all relevant agents. |
| Discord | Any chat platform | Humans must be able to direct agents and observe coordination. |
| OpenClaw | Another agent runtime | Agents must be able to read files, write files, and communicate. |
| `mkdir` lock dirs | Another atomic lock primitive | Multi-writer files must never rely on blind concurrent writes. |

## Minimal Adoption Checklist

Before you implement this pattern anywhere, make sure you have:

1. A private workspace for each agent.
2. A shared directory or equivalent for public coordination.
3. A clear list of which files are private, shared, single-owner, and multi-writer.
4. A chat surface where the human operator and agents can coordinate.
5. A documented write-locking rule for shared files.

If those five conditions are true, you can adapt The Office to almost any environment.
