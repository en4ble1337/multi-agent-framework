# The Office — Multi-Agent Architecture

> A framework for running multiple autonomous AI agents that share knowledge, coordinate through files and Discord, and scale by simply adding new VMs.

---

## Table of Contents

- [Philosophy](#philosophy)
- [Core Concepts](#core-concepts)
- [Infrastructure Layer](#infrastructure-layer)
- [Agent Anatomy](#agent-anatomy)
- [Shared Memory (NFS)](#shared-memory-nfs)
- [Discord Architecture](#discord-architecture)
- [Agent Lifecycle](#agent-lifecycle)
- [Communication Patterns](#communication-patterns)
- [Scaling](#scaling)
- [Security Considerations](#security-considerations)
- [Example: Two-Agent Setup](#example-two-agent-setup)

---

## Philosophy

Most multi-agent frameworks are over-engineered. They introduce custom protocols, message queues, orchestration layers, and coordination services — all of which add complexity and failure modes.

**The Office** takes a different approach: agents are independent entities that share a filesystem and communicate through Discord channels. No custom protocols. No message brokers. Just files and chat.

### Why "The Office"?

The metaphor maps perfectly:

| Office Concept | Agent Equivalent |
|---|---|
| Employee | Autonomous AI agent on its own VM |
| Desk | Agent's private workspace (`~/.openclaw/workspace/`) |
| Conference Room | Shared Discord channel + shared filesystem |
| Bulletin Board | `/shared/bulletin-board.md` |
| Filing Cabinet | `/shared/knowledge/` directory |
| Boss's Office | Your DM with each agent (Telegram, Discord, etc.) |
| Company Policy | `/shared/company/POLICY.md` |
| Employee Directory | `/shared/agents/roster.md` |
| Intercom | Discord `@mentions` across channels |

### Design Principles

1. **Independence over coordination** — Each agent runs autonomously. If one goes down, the others keep working. No single point of failure (except the NFS share, which is trivially redundant).

2. **Files over protocols** — Shared state lives in plain text files on an NFS mount. Any agent can read them. No custom APIs, no serialization formats, no versioning headaches. Markdown is the universal language.

3. **Discord as the UI layer** — Humans interact through Discord. Agents post updates, receive commands, and coordinate through channels. Discord is the "office floor" where everyone can see what's happening.

4. **Clone to scale** — Adding a new agent means cloning a VM, changing the identity, mounting the share, and joining Discord. 15 minutes from zero to productive.

5. **The Boss is the Boss** — Agents are autonomous within their scope, but the human (you) has final authority. Shared announcements override individual agent decisions. Chain of command is clear.

---

## Core Concepts

### Agent
An autonomous AI assistant running on its own VM with its own OpenClaw instance, personality (SOUL.md), memory, and assigned tasks. Each agent has a name, a role, and one or more projects.

### Workspace
An agent's private directory (`~/.openclaw/workspace/`). Contains their soul, memory, tools config, and project files. **Other agents cannot access this.** This is their desk — private, personal, customized.

### Shared Drive
An NFS-mounted directory (`/shared/`) accessible by all agents. Contains company knowledge, shared memory, the agent roster, and inter-agent communication files. Think of it as the shared filing cabinet and bulletin board.

### Channel
A Discord channel dedicated to an agent or a project. Agents post status updates, logs, and reports to their channels. The boss and other agents can read and respond.

### Conference Room
A shared Discord channel where all agents and the boss can communicate. Used for announcements, cross-agent coordination, and group discussions.

---

## Infrastructure Layer

### Proxmox Host

All agent VMs run on a Proxmox hypervisor. This gives you:

- **Snapshots** — Roll back any agent to a known-good state
- **Cloning** — Spin up new agents from a template VM in minutes
- **Resource control** — Allocate CPU/RAM per agent as needed
- **Isolation** — Each agent is fully sandboxed at the VM level

### Template VM

One "golden image" VM that all agents are cloned from:

```
Ubuntu 24.04 LTS
├── OpenClaw node installed & configured
├── Chrome browser (for web/social automation)
├── NFS client configured (mount at /shared)
├── Display :0 (for browser-visible Proxmox console)
├── xhost +local: configured
└── Base workspace with placeholder SOUL.md, AGENTS.md
```

**When cloning, change:**
1. Hostname → agent name
2. Static IP → new IP from your range
3. `/etc/machine-id` → regenerate (Proxmox "unique" option)
4. OpenClaw node name → agent name
5. `SOUL.md` → agent's personality
6. `IDENTITY.md` → agent's name, role, emoji
7. Discord bot token → new bot for this agent
8. Chrome profile → log into the agent's accounts

### NFS Server

A lightweight NFS export on your Proxmox host (or a dedicated NAS/VM):

```bash
# /etc/exports on NFS server
/srv/office  10.1.20.0/24(rw,sync,no_subtree_check,no_root_squash)
```

```bash
# /etc/fstab on each agent VM
10.1.20.1:/srv/office  /shared  nfs  defaults,_netdev  0  0
```

**Why NFS?**
- Zero configuration per agent (just mount)
- Real-time — no sync delays
- Battle-tested, boring technology (boring is good)
- Works with plain files — no database, no API
- Any agent can read/write immediately

**Alternatives considered:**
- **Syncthing** — Adds sync delay, conflict resolution complexity. Overkill for LAN.
- **Git repo** — Requires commit/push/pull cycles. Too slow for real-time status.
- **Database** — Requires a service, schema, queries. Files are simpler.
- **S3/Object storage** — Adds latency, API complexity. Not needed on LAN.

---

## Agent Anatomy

Each agent VM has the following structure:

```
/home/openclaw/.openclaw/
├── workspace/                          # Agent's private desk
│   ├── AGENTS.md                       # Operating instructions
│   ├── SOUL.md                         # Personality & identity
│   ├── IDENTITY.md                     # Name, emoji, role
│   ├── USER.md                         # About the boss
│   ├── MEMORY.md                       # Personal long-term memory
│   ├── TOOLS.md                        # Credentials & config (PRIVATE)
│   ├── HEARTBEAT.md                    # Periodic task checklist
│   ├── memory/                         # Daily journals
│   │   └── 2026-03-10.md
│   └── projects/                       # Assigned work
│       ├── project-a/
│       └── project-b/
│
├── agents/main/agent/
│   ├── channels.json                   # Discord + other channels
│   └── mcp.json                        # MCP tool config
│
└── openclaw.json                       # Gateway config

/shared/                                # NFS mount — shared across all agents
└── (see Shared Memory section)
```

### What's Private vs Shared

| File | Scope | Reasoning |
|---|---|---|
| `SOUL.md` | Private | Each agent has a unique personality |
| `MEMORY.md` | Private | Personal experiences and context |
| `TOOLS.md` | Private | API keys, credentials — NEVER shared |
| `HEARTBEAT.md` | Private | Agent-specific task schedule |
| `projects/` | Private | Unless project is cross-agent |
| `/shared/knowledge/` | Shared | Company-wide knowledge base |
| `/shared/conference/` | Shared | Announcements, meeting notes |
| `/shared/agents/` | Shared | Public profiles and status |

---

## Shared Memory (NFS)

### Directory Structure

```
/shared/
├── company/                            # Company-wide governance
│   ├── POLICY.md                       # Rules all agents follow
│   ├── BRAND.md                        # Brand guidelines, tone, assets
│   ├── CONTACTS.md                     # Shared contact directory
│   └── ACCOUNTS.md                     # Which agent owns which accounts
│
├── conference/                         # The conference room
│   ├── announcements.md                # Boss posts, all agents read
│   ├── standup/                        # Daily standups
│   │   └── 2026-03-10.md              # Each agent appends their status
│   └── meetings/                       # Meeting notes / decisions
│       └── 2026-03-10-strategy.md
│
├── knowledge/                          # Collective intelligence
│   ├── market-intel/                   # Research contributions
│   │   ├── gpu-hosting-landscape.md
│   │   └── competitor-analysis.md
│   ├── lessons-learned.md              # Mistakes anyone can learn from
│   ├── playbooks/                      # Shared how-to guides
│   │   ├── twitter-engagement.md
│   │   └── discord-community.md
│   └── research/                       # Deep dives
│       └── topic-name.md
│
├── agents/                             # Agent directory
│   ├── roster.md                       # Who's who — name, role, VM IP, status
│   ├── dwight/                         # Dwight's public profile
│   │   ├── PROFILE.md                  # Role, capabilities, current focus
│   │   ├── status.md                   # Current status (auto-updated)
│   │   └── outbox/                     # Messages TO other agents
│   │       └── to-agent-b.md
│   ├── agent-b/
│   │   ├── PROFILE.md
│   │   ├── status.md
│   │   └── outbox/
│   │       └── to-dwight.md
│   └── ...
│
└── projects/                           # Cross-agent projects
    └── project-name/
        ├── README.md
        ├── tasks.md                    # Who's doing what
        └── shared-data/
```

### File Conventions

**Status files** (`status.md`) — Updated by each agent on heartbeat:
```markdown
# Dwight — Status
**Updated:** 2026-03-10 15:00 UTC
**State:** Active
**Current task:** GPU Autopilot Twitter — afternoon engagement round
**Blockers:** None
**Next action:** 5pm CST post scheduled
```

**Standup files** (`standup/YYYY-MM-DD.md`) — Each agent appends:
```markdown
## Dwight — 09:00 CST
- Yesterday: Posted 3 tweets, engaged 12 accounts, gained 4 followers
- Today: Afternoon engagement, reply to @VoskCoin thread
- Blockers: None

## Agent B — 09:05 CST
- Yesterday: ...
```

**Outbox pattern** — Async agent-to-agent messaging:
```markdown
# Message to Agent B
**From:** Dwight
**Date:** 2026-03-10 14:30 UTC
**Priority:** Normal
**Read:** false

Hey, I found a thread about GPU pricing on Twitter that's relevant to your
Discord community management. Link: https://x.com/...

Can you share this in the community #gpu-market channel?
```

Receiving agent checks their inbox (`/shared/agents/<name>/outbox/`) on heartbeat, processes messages, marks as read.

---

## Discord Architecture

### Server Structure

The Discord server mirrors the office. Each agent is a **category** (top-level grouping) with sub-channels for their projects and ops. Shared spaces sit at the top.

```
🏢 THE OFFICE (Discord Server)
│
├── 📋 FRONT DESK                          [Category — Shared Spaces]
│   ├── #michaels-office                   The Boss's office. Directives, announcements, policy.
│   └── #conference-room                   All agents + boss. Group discussions & coordination.
│
├── 👓 DWIGHT                              [Category — Agent]
│   ├── #dwight-general                    Primary channel. Boss ↔ Dwight direct.
│   ├── #dwight-timpi-hosting              Timpi hosting tracking & collections.
│   ├── #dwight-polymarket                 Polymarket trading (paused).
│   ├── #dwight-gpuaas                     GPU-as-a-Service ops.
│   ├── #dwight-vast-bot                   Vast.ai monitoring.
│   └── #dwight-logs                       Automated reports, cron output, errors.
│
├── 😏 JIM                                 [Category — Agent]
│   ├── #jim-general                       Primary channel. Boss ↔ Jim direct.
│   ├── #jim-logs                          Automated reports.
│   └── (project channels added as assigned)
│
├── 🎀 PAM                                 [Category — Agent]
│   ├── #pam-general                       Primary channel. Boss ↔ Pam direct.
│   ├── #pam-logs                          Automated reports.
│   └── (project channels added as assigned)
│
├── 💅 KELLY                               [Category — Agent]
│   ├── #kelly-general                     Primary channel. Boss ↔ Kelly direct.
│   ├── #kelly-logs                        Automated reports.
│   └── (project channels added as assigned)
│
└── (More agents added as needed — one category per agent)
```

### Why Categories = Agents

Discord categories are the perfect abstraction for agents:

1. **Visual grouping** — You see all of an agent's channels collapsed under their name
2. **Permission scoping** — Each agent's bot only needs access to their own category + shared spaces
3. **Scalable** — Adding an agent = adding a category. No restructuring.
4. **Collapsible** — Don't care what Dwight's doing right now? Collapse the category. Clean.
5. **Familiar** — Anyone who uses Discord understands categories instantly.

### Channel Naming Convention

```
#<agent-name>-<project-or-function>
```

Examples:
- `#dwight-general` — Dwight's main channel
- `#dwight-gpu-autopilot-twitter` — Specific project
- `#dwight-logs` — Every agent gets a logs channel
- `#agent-b-twitter` — Agent B's Twitter ops

**Why prefix with agent name?**
- Discord search works globally — prefixing makes it obvious which agent a channel belongs to
- When viewing "All Channels" or notifications, you immediately know the source
- Prevents name collisions (`#twitter` is ambiguous, `#dwight-twitter` is not)

### Bot Permissions per Agent

Each agent's Discord bot should have:

| Permission | Their Category | Shared Spaces | Other Agent Categories |
|---|---|---|---|
| Read Messages | ✅ | ✅ | ❌ (optional read-only) |
| Send Messages | ✅ | ✅ (conference, standup) | ❌ |
| Manage Threads | ✅ | ❌ | ❌ |
| Read History | ✅ | ✅ | ❌ |

**Reasoning:** Agents should have full control of their own channels, limited access to shared spaces, and no write access to other agents' channels. This prevents agents from stepping on each other.

---

## Agent Instructions

Each agent needs standard operating instructions so they know how The Office works — which channels are theirs, how to use the shared drive, when to speak in the conference room, and how to communicate with other agents.

A ready-to-use template is provided in **[AGENT-INSTRUCTIONS-TEMPLATE.md](AGENT-INSTRUCTIONS-TEMPLATE.md)**.

Copy it into each new agent's `AGENTS.md` and replace the `{{PLACEHOLDER}}` values:

| Placeholder | Example |
|---|---|
| `{{AGENT_NAME}}` | Jim |
| `{{AGENT_NAME_LOWER}}` | jim |
| `{{AGENT_ROLE}}` | Community & Social Media |
| `{{AGENT_EMOJI}}` | 😏 |
| `{{AGENT_MODEL}}` | Claude Sonnet |
| `{{AGENT_VM_IP}}` | 10.1.20.195 |
| `{{AGENT_PROJECTS}}` | New X account, Discord community |
| `{{PROJECT_CHANNELS}}` | `#jim-twitter`, `#jim-community` |

The template covers:
- **Discord behavior** — When to speak, when to stay silent, where to post
- **Shared drive usage** — What to read, what to write, what to contribute
- **Inter-agent communication** — Outbox pattern for async messaging
- **Heartbeat checklist** — What to check on each cycle
- **Safety boundaries** — Credential isolation, channel boundaries, escalation

---

## Agent Lifecycle

### Creating a New Agent

```
Step 1: Clone Template VM
├── Proxmox → Clone → Full Clone → Tick "Unique"
├── Allocate: 2 CPU, 2-4GB RAM, 20GB disk
└── Boot & verify SSH access

Step 2: Configure Identity
├── Set hostname:           hostnamectl set-hostname agent-name
├── Set static IP:          /etc/netplan/...
├── Regenerate machine-id:  rm /etc/machine-id && systemd-machine-id-setup
├── Update OpenClaw node:   edit ~/.openclaw/openclaw.json → node name
└── Restart OpenClaw:       openclaw gateway restart

Step 3: Mount Shared Drive
├── mkdir /shared
├── Add to /etc/fstab:     10.1.20.1:/srv/office /shared nfs defaults,_netdev 0 0
├── mount /shared
└── Verify: ls /shared/agents/roster.md

Step 4: Set Agent Identity
├── Edit SOUL.md            → Agent's personality, voice, style
├── Edit IDENTITY.md        → Name, emoji, role description
├── Edit USER.md            → Same boss info across all agents
├── Edit HEARTBEAT.md       → Agent-specific periodic tasks
└── Edit TOOLS.md           → Agent-specific credentials

Step 5: Create Discord Presence
├── Create Discord bot      → Discord Developer Portal
├── Add bot to server       → With scoped permissions
├── Create category         → Named after agent with emoji
├── Create channels         → #agent-general, #agent-logs, + project channels
└── Configure channels.json → Point to correct guild + channels

Step 6: Register in The Office
├── Add to /shared/agents/roster.md
├── Create /shared/agents/<name>/PROFILE.md
├── Create /shared/agents/<name>/status.md
├── Post introduction in #conference-room
└── First standup entry in /shared/conference/standup/

Step 7: Go Live
├── Start OpenClaw gateway
├── Verify heartbeat cycle
├── Send test message to #agent-general
└── Confirm NFS read/write works
```

### Decommissioning an Agent

```
1. Post farewell in #conference-room (optional, but classy)
2. Archive Discord category (or delete channels)
3. Remove from /shared/agents/roster.md
4. Archive /shared/agents/<name>/ → /shared/agents/_archived/<name>/
5. Stop OpenClaw on VM
6. Shut down VM (keep snapshot for 30 days, then delete)
```

---

## Communication Patterns

### Boss → One Agent
**Channel:** Agent's `#general` channel or Telegram DM
**Use case:** Direct instructions, questions, feedback

### Boss → All Agents
**Channel:** `#announcements` or `#conference-room`
**Shared file:** `/shared/conference/announcements.md`
**Use case:** Policy changes, company-wide directives

### Agent → Boss
**Channel:** Agent's `#general` channel, Telegram DM, or `#alerts` (urgent)
**Use case:** Status updates, questions, escalations

### Agent → Agent (Async)
**Method:** File-based outbox (`/shared/agents/<target>/outbox/from-<sender>.md`)
**Use case:** Non-urgent cross-agent requests, sharing intel

### Agent → Agent (Semi-Sync)
**Method:** Discord `#conference-room` with `@mention`
**Use case:** Time-sensitive coordination, group decisions

### Daily Standup (Automated)
**When:** Each agent posts during their first heartbeat of the day
**Where:** `#daily-standup` channel + `/shared/conference/standup/YYYY-MM-DD.md`
**Format:**
```
**Dwight — Daily Standup**
📅 2026-03-10 | 09:00 CST

**Yesterday:**
• Posted 3 tweets, engaged 12 accounts
• Gained 4 new followers

**Today:**
• Afternoon engagement round
• Reply to @VoskCoin GPU pricing thread

**Blockers:** None
```

---

## Scaling

### Resource Requirements per Agent

| Component | Minimum | Recommended |
|---|---|---|
| CPU | 1 vCPU | 2 vCPU |
| RAM | 1 GB (no browser) | 2-4 GB (with Chrome) |
| Disk | 10 GB | 20 GB |
| Network | LAN access + internet | Static IP on 10.1.20.x |

### Scaling Milestones

| Agents | Notes |
|---|---|
| 1-3 | Single Proxmox host handles easily. NFS on host. |
| 4-10 | Consider dedicated NFS VM/NAS. Monitor host resources. |
| 10-20 | Multiple Proxmox hosts or move to cloud. Shared drive becomes critical path — add redundancy. |
| 20+ | You're running a company. Consider Kubernetes, proper orchestration, or hiring humans. |

### Cost Scaling (Approximate)

Each agent's ongoing cost is primarily **LLM API usage**:

| Component | Cost |
|---|---|
| VM resources | ~Free (on your own Proxmox hardware) |
| LLM API (Sonnet, light use) | ~$5-15/month |
| LLM API (Opus, heavy use) | ~$30-100/month |
| Discord bot | Free |
| NFS storage | Negligible |

**Optimization:** Use cheaper models (Sonnet) for routine agents, Opus for agents doing complex work. Each agent can run on a different model.

---

## Security Considerations

### Credential Isolation
- `TOOLS.md` is **never** on the shared drive — each agent has their own credentials
- Agents should never write API keys, tokens, or passwords to `/shared/`
- If an agent needs a shared service credential, the boss provisions it individually

### Network Isolation
- All agent VMs on the same LAN subnet (10.1.20.x)
- NFS restricted to that subnet
- Each VM has UFW enabled — only necessary ports open
- No agent VM should be directly exposed to the internet (use reverse proxy if needed)

### Permission Boundaries
- Agents can read all of `/shared/` but should only write to:
  - `/shared/agents/<own-name>/`
  - `/shared/conference/standup/` (append only)
  - `/shared/knowledge/` (contributions)
- Enforce via Unix file permissions + NFS uid mapping if you want hard boundaries

### Blast Radius
- A compromised agent can only affect:
  - Its own VM
  - Its own Discord channels
  - Shared files it can write to
- It **cannot** affect other agents' private workspaces, credentials, or VMs
- Worst case: wipe the VM, re-clone from template, restore from snapshot

---

## Example: Two-Agent Setup

### Agent 1: Dwight 👓
- **Role:** Operations & Marketing
- **VM:** 10.1.20.194 (existing GPU Autopilot Browser)
- **Projects:** GPU Autopilot Twitter, Timpi Hosting, Vast monitoring
- **Model:** Claude Opus (complex tasks, trading analysis)
- **Channels:** Telegram DM + Discord category

### Agent 2: Jim 😏 (example)
- **Role:** Community & Social
- **VM:** 10.1.20.195 (cloned from template)
- **Projects:** New X account campaign, Discord community management
- **Model:** Claude Sonnet (engagement, social posts)
- **Channels:** Discord category only

### Setup Checklist

```
[x] Proxmox template VM ready
[ ] NFS server configured on Proxmox host
[ ] /shared/ directory structure created
[ ] Dwight's VM updated to mount /shared/
[ ] Jim's VM cloned from template
[ ] Jim's identity configured (SOUL.md, IDENTITY.md)
[ ] New Discord bot created for Jim
[ ] Discord categories + channels created
[ ] Roster updated
[ ] Both agents posting to #daily-standup
[ ] Test: Boss posts in #conference-room, both agents respond
```

---

## Concurrency & Lockfile Protocol

When multiple agents share writable files, simultaneous writes can cause data loss. The Office uses a **convention-based lockfile protocol** to prevent this.

**Full specification:** [LOCKFILE-PROTOCOL.md](LOCKFILE-PROTOCOL.md)

**Quick summary:**
- Agents use `/shared/tools/shared-write.sh` instead of writing to `/shared/` directly
- The script creates a `.lock/` directory (atomic via `mkdir`) before writing
- Other agents see the lock and wait/retry
- Stale locks (>60s) are auto-cleaned
- Single-owner files (agent's own status, outbox) skip locking

---

## Future Enhancements

These are not needed now, but worth thinking about as the office grows:

1. **Skill sharing** — Agents could share skills via `/shared/skills/`. One agent develops a playbook, all agents can use it.

2. **Task queue** — `/shared/tasks/pending/` with files that any available agent can pick up. First to claim it, works it.

3. **Voting / consensus** — For decisions that affect multiple agents, a simple voting file where each agent appends their vote.

4. **Agent health monitoring** — A cron job on the Proxmox host that checks each agent's `status.md` timestamp. If stale > 1 hour, alert the boss.

5. **Shared calendar** — `/shared/calendar/YYYY-MM-DD.md` for coordinating schedules and avoiding conflicts (e.g., two agents tweeting about the same thing simultaneously).

6. **Agent-to-agent delegation** — Agent A can write a task to Agent B's outbox. Agent B picks it up, executes, and writes the result back. Crude but functional.

---

## Quick Reference

| Action | How |
|---|---|
| Add new agent | Clone VM → configure identity → mount NFS → create Discord category |
| Talk to one agent | Their `#general` channel or DM |
| Talk to all agents | `#conference-room` or `/shared/conference/announcements.md` |
| Check agent status | `#daily-standup` or `/shared/agents/<name>/status.md` |
| Share knowledge | Write to `/shared/knowledge/` |
| Agent-to-agent message | Write to `/shared/agents/<target>/outbox/` |
| Decommission agent | Archive channels, remove from roster, shut down VM |
| Emergency shutdown | Stop OpenClaw on VM, or shut down VM in Proxmox |

---

*Built for [OpenClaw](https://github.com/openclaw/openclaw) agents running on Proxmox. Designed to be simple, file-based, and scalable without over-engineering.*

*Last updated: 2026-03-10*
