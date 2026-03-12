# Lockfile Protocol — Shared File Concurrency

> How agents safely write to shared files without overwriting each other.

---

## The Problem

Multiple agents share `/shared/` via NFS. If two agents write to the same file at the same time, the last write wins and the other's changes are silently lost. NFS has no reliable built-in file locking for this use case.

## The Solution: Convention-Based Lockfiles

A **lockfile** is a directory (not a file) that acts as a "DO NOT DISTURB" sign. Before writing to a shared file, an agent creates a `.lock` directory next to it. Other agents see the lock and wait. When the writing agent is done, it deletes the lock.

There is no OS-level magic here. It works because **all agents agree to follow the protocol**.

---

## How It Works

### Visual Overview

```
BEFORE (no one writing):
/shared/knowledge/
└── market-intel.md

DWIGHT ACQUIRES LOCK:
/shared/knowledge/
├── market-intel.md
└── market-intel.md.lock/        ← directory created by Dwight
    └── info.json                ← metadata: who, when

DWIGHT WRITES CHANGES:
/shared/knowledge/
├── market-intel.md              ← Dwight edits this normally
└── market-intel.md.lock/

DWIGHT RELEASES LOCK:
/shared/knowledge/
└── market-intel.md              ← lock directory deleted, file updated
```

### The Rules

1. **Check** — Does `<filename>.lock/` exist?
2. **Acquire** — If no: create it with `mkdir` (atomic on NFS). If yes: wait and retry.
3. **Write** — Make your changes to the actual file.
4. **Release** — Delete the `.lock/` directory.

### Why a Directory, Not a File?

`mkdir` is **atomic** on NFS — if two agents try to create the same directory at the exact same millisecond, only one succeeds (the other gets `EEXIST`). Regular file creation (`touch`, `echo >`) is **not** atomic on NFS, so two agents could both "successfully" create the same lock file. The directory trick gives you a reliable mutex for free.

---

## Lock Metadata

The `.lock/` directory contains an `info.json` so other agents know who holds the lock and for how long:

```json
{
  "agent": "dwight",
  "timestamp": "2026-03-12T04:25:00Z",
  "operation": "appending market analysis"
}
```

This is used for:
- **Debugging** — "Why can't I write? Oh, Dwight has it."
- **Stale lock detection** — If the timestamp is older than 60 seconds, the lock is probably stale (agent crashed) and can be force-removed.

---

## Wrapper Script

All agents use this script instead of writing to `/shared/` directly.

### `/shared/tools/shared-write.sh`

```bash
#!/bin/bash
# shared-write.sh — Safe concurrent writes to shared files
#
# Usage:
#   shared-write.sh <file> <agent-name> append "content to append"
#   shared-write.sh <file> <agent-name> overwrite "full new content"
#   shared-write.sh <file> <agent-name> edit "old text" "new text"
#
# Examples:
#   shared-write.sh /shared/conference/standup/2026-03-12.md dwight append "## Dwight — 09:00\n- Did stuff"
#   shared-write.sh /shared/knowledge/lessons.md jim append "- Never trust 80%+ markets"
#   shared-write.sh /shared/agents/roster.md pam overwrite "$(cat updated-roster.md)"

set -euo pipefail

LOCK_TIMEOUT=60       # seconds before lock is considered stale
MAX_RETRIES=5         # number of retry attempts
RETRY_DELAY=2         # seconds between retries

TARGET_FILE="$1"
AGENT_NAME="$2"
OPERATION="$3"        # append | overwrite | edit
shift 3

# --- Lock Functions ---

acquire_lock() {
    local lockdir="${TARGET_FILE}.lock"

    for i in $(seq 1 $MAX_RETRIES); do
        if mkdir "$lockdir" 2>/dev/null; then
            # Got the lock — write metadata
            cat > "$lockdir/info.json" <<EOF
{
  "agent": "$AGENT_NAME",
  "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "operation": "$OPERATION $TARGET_FILE"
}
EOF
            echo "Lock acquired by $AGENT_NAME"
            return 0
        fi

        # Lock exists — check if stale
        if [ -f "$lockdir/info.json" ]; then
            lock_time=$(jq -r '.timestamp' "$lockdir/info.json" 2>/dev/null || echo "")
            if [ -n "$lock_time" ]; then
                lock_epoch=$(date -d "$lock_time" +%s 2>/dev/null || echo 0)
                now_epoch=$(date +%s)
                age=$(( now_epoch - lock_epoch ))

                if [ "$age" -gt "$LOCK_TIMEOUT" ]; then
                    holder=$(jq -r '.agent' "$lockdir/info.json" 2>/dev/null || echo "unknown")
                    echo "WARNING: Stale lock from $holder (${age}s old), forcing removal"
                    rm -rf "$lockdir"
                    continue
                fi

                holder=$(jq -r '.agent' "$lockdir/info.json" 2>/dev/null || echo "unknown")
                echo "Lock held by $holder (${age}s), retry $i/$MAX_RETRIES..."
            fi
        else
            echo "Lock dir exists but no info.json, retry $i/$MAX_RETRIES..."
        fi

        sleep $RETRY_DELAY
    done

    echo "ERROR: Failed to acquire lock after $MAX_RETRIES attempts"
    return 1
}

release_lock() {
    rm -rf "${TARGET_FILE}.lock"
    echo "Lock released"
}

# --- Ensure lock is released on exit (even on crash/error) ---
cleanup() {
    release_lock 2>/dev/null || true
}
trap cleanup EXIT

# --- Main ---

acquire_lock

case "$OPERATION" in
    append)
        echo -e "$1" >> "$TARGET_FILE"
        echo "Appended to $TARGET_FILE"
        ;;
    overwrite)
        echo -e "$1" > "$TARGET_FILE"
        echo "Overwrote $TARGET_FILE"
        ;;
    edit)
        old_text="$1"
        new_text="$2"
        if grep -qF "$old_text" "$TARGET_FILE"; then
            sed -i "s|$(echo "$old_text" | sed 's/[&/\]/\\&/g')|$(echo "$new_text" | sed 's/[&/\]/\\&/g')|g" "$TARGET_FILE"
            echo "Edited $TARGET_FILE"
        else
            echo "WARNING: Old text not found in $TARGET_FILE, no changes made"
            exit 1
        fi
        ;;
    *)
        echo "ERROR: Unknown operation '$OPERATION'. Use: append | overwrite | edit"
        exit 1
        ;;
esac

echo "Done: $OPERATION on $TARGET_FILE by $AGENT_NAME"
```

### Key Safety Features

| Feature | How |
|---|---|
| **Atomic lock** | `mkdir` — only one agent wins |
| **Stale lock cleanup** | Locks older than 60s are force-removed |
| **Crash safety** | `trap cleanup EXIT` releases lock even if script errors |
| **Retry with backoff** | 5 attempts, 2s apart before giving up |
| **Metadata** | `info.json` tracks who holds the lock and why |

---

## Agent Instructions

Every agent's `AGENTS.md` (or the shared `POLICY.md`) must include:

```markdown
## Shared File Writing Protocol

NEVER edit files in /shared/ directly with echo, cat, sed, or any direct write.

ALWAYS use the shared-write script:

    /shared/tools/shared-write.sh <file> <your-name> append "content"
    /shared/tools/shared-write.sh <file> <your-name> overwrite "content"
    /shared/tools/shared-write.sh <file> <your-name> edit "old" "new"

WHY: Multiple agents share the same filesystem. Without locking, simultaneous
writes cause data loss. The script handles locking automatically.

If the script says "Failed to acquire lock" — DO NOT force it. Skip the write
and try again on your next heartbeat cycle.
```

---

## Which Files Need Locking?

### Lock required (multiple writers)

| File | Why |
|---|---|
| `/shared/conference/standup/YYYY-MM-DD.md` | All agents append their standup |
| `/shared/knowledge/*.md` | Any agent can contribute |
| `/shared/conference/announcements.md` | Boss + agents post |
| `/shared/agents/roster.md` | Updated when agents join/leave |

### No lock needed (single owner)

| File | Why |
|---|---|
| `/shared/agents/dwight/status.md` | Only Dwight writes this |
| `/shared/agents/dwight/outbox/*` | Only Dwight writes, others read |
| `/shared/agents/jim/PROFILE.md` | Only Jim updates their profile |
| Any file in agent's own subdirectory | Single writer by convention |

**Rule of thumb:** If only one agent ever writes to it, skip the lock. If multiple agents might write, always lock.

---

## Edge Cases

| Scenario | What Happens |
|---|---|
| Agent crashes mid-write | `trap EXIT` releases lock. If trap fails, stale timeout (60s) clears it. |
| Two agents lock at exact same ms | `mkdir` is atomic — one wins, one gets EEXIST and retries. |
| NFS server goes down | Everything breaks. Locks are the least of your problems. |
| Agent forgets to use the script | Data could be lost. This is a convention, not a hard barrier. |
| Lock directory exists but no info.json | Treated as locked, retried. Likely a partial creation. |
| Agent tries to lock a file it already locked | Deadlock. Don't do this. One lock per file per agent. |

---

## Setup

```bash
# On the NFS server, create the tools directory
mkdir -p /srv/office/tools

# Copy the script
cp shared-write.sh /srv/office/tools/shared-write.sh
chmod +x /srv/office/tools/shared-write.sh

# Ensure jq is installed on all agent VMs (for reading info.json)
apt install -y jq
```

---

## Enforcement Levels

| Level | How | Tradeoff |
|---|---|---|
| **Soft** (instructions only) | Tell agents the rules in AGENTS.md/POLICY.md | Simple, but agents can forget |
| **Medium** (script + instructions) | Wrapper script + clear docs ← **recommended** | Good balance, agents understand why |
| **Hard** (filesystem permissions) | Agents can't write to /shared/ directly, only through a service | Complex infrastructure, overkill for <10 agents |

For 2-5 agents: **Medium** is the sweet spot. The script handles locking, the instructions explain why, and the risk of an agent going rogue is low.

---

*Part of [The Office — Multi-Agent Architecture](ARCHITECTURE.md)*
*Last updated: 2026-03-12*
