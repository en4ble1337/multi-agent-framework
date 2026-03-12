# Lockfile Protocol for Shared Files

> A neutral pattern for protecting shared files from concurrent writes.

## The Problem

If multiple agents can write to the same file, blind writes will eventually overwrite each other. The exact storage backend does not change that risk.

Typical failure mode:

1. Agent A reads a shared file.
2. Agent B reads the same file.
3. Agent A writes an update.
4. Agent B writes an update based on stale content.
5. One update is lost.

## The Contract

Before modifying a multi-writer file, an agent must:

1. acquire a lock next to the target file
2. record who holds the lock and why
3. perform the write
4. release the lock

This is a convention, not magic. It only works if every writer follows it.

## Why Use a Lock Directory

A directory such as `<file>.lock/` is often a good lock primitive because directory creation can be atomic on many POSIX filesystems and shared storage backends.

Important:

- Validate that assumption on your actual storage backend.
- If your storage does not provide a reliable atomic directory create, choose another lock primitive.

## Example Flow

```text
Before:
/shared/knowledge/notes.md

Lock acquired:
/shared/knowledge/notes.md
/shared/knowledge/notes.md.lock/
  info.json

Write in progress:
/shared/knowledge/notes.md
/shared/knowledge/notes.md.lock/

After release:
/shared/knowledge/notes.md
```

## Lock Metadata

Store enough metadata for debugging and stale lock recovery:

```json
{
  "actor": "agent-a",
  "timestamp": "YYYY-MM-DDTHH:MM:SSZ",
  "operation": "append checkpoint"
}
```

Useful fields:

- `actor` for identifying the owner
- `timestamp` for stale lock detection
- `operation` for quick diagnostics

## Reference Wrapper Script

The script below is a reference implementation, not a required tool. Adapt it to your shell, runtime, and storage semantics.

```bash
#!/usr/bin/env bash
set -euo pipefail

LOCK_TIMEOUT=60
MAX_RETRIES=5
RETRY_DELAY=2

TARGET_FILE="$1"
ACTOR_NAME="$2"
MODE="$3"
shift 3

lockdir="${TARGET_FILE}.lock"

write_lock_metadata() {
  cat > "${lockdir}/info.json" <<EOF
{
  "actor": "${ACTOR_NAME}",
  "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "operation": "${MODE} ${TARGET_FILE}"
}
EOF
}

acquire_lock() {
  for attempt in $(seq 1 "${MAX_RETRIES}"); do
    if mkdir "${lockdir}" 2>/dev/null; then
      write_lock_metadata
      return 0
    fi

    if [ -f "${lockdir}/info.json" ]; then
      lock_time=$(jq -r '.timestamp' "${lockdir}/info.json" 2>/dev/null || echo "")
      holder=$(jq -r '.actor' "${lockdir}/info.json" 2>/dev/null || echo "unknown")
      lock_epoch=$(date -d "${lock_time}" +%s 2>/dev/null || echo 0)
      now_epoch=$(date +%s)
      age=$((now_epoch - lock_epoch))

      if [ "${age}" -gt "${LOCK_TIMEOUT}" ]; then
        rm -rf "${lockdir}"
        continue
      fi

      echo "Lock held by ${holder}; retry ${attempt}/${MAX_RETRIES}" >&2
    fi

    sleep "${RETRY_DELAY}"
  done

  echo "Failed to acquire lock for ${TARGET_FILE}" >&2
  return 1
}

release_lock() {
  rm -rf "${lockdir}"
}

cleanup() {
  release_lock 2>/dev/null || true
}

trap cleanup EXIT

acquire_lock

case "${MODE}" in
  append)
    printf "%s\n" "$1" >> "${TARGET_FILE}"
    ;;
  overwrite)
    printf "%s\n" "$1" > "${TARGET_FILE}"
    ;;
  *)
    echo "Unsupported mode: ${MODE}" >&2
    exit 1
    ;;
esac
```

## Files That Usually Need Locking

| File pattern | Why |
|---|---|
| `/shared/conference/standup/*.md` | Multiple agents append to the same daily file |
| `/shared/knowledge/*.md` | Shared notes and playbooks often have more than one writer |
| `/shared/agents/roster.md` | Registration changes can come from more than one workflow |
| `/shared/projects/*/tasks.md` | Shared task tracking is often multi-writer |

## Files That Usually Do Not Need Locking

| File pattern | Why |
|---|---|
| `/shared/agents/<name>/status.md` | One owning agent writes it |
| `/shared/agents/<name>/PROFILE.md` | Typically updated by one owner |
| agent-local files in a private workspace | Single writer by design |

Rule of thumb:

- one writer means explicit ownership
- many writers means locking

## Failure Cases

| Scenario | Expected response |
|---|---|
| Agent crashes while holding the lock | Use stale timeout or manual inspection before removal |
| Two agents try to lock at the same time | One should succeed, one should retry |
| Lock exists without metadata | Treat it as occupied and inspect before forcing removal |
| A writer bypasses the protocol | Data loss is possible |
| Shared storage goes offline | Coordination stops regardless of the lock design |

## Operational Guidance

- Keep lock time short.
- Do not hold a lock while performing slow external work.
- Write metadata so humans can diagnose stale locks quickly.
- Prefer append-only patterns when possible.
- Audit which files are truly multi-writer instead of locking everything.

## Minimal Policy Text

Copy a rule like this into shared policy docs or each agent workspace:

```markdown
Never write directly to a shared multi-writer file without using the lock protocol.

If you cannot acquire the lock, do not force the write unless the workflow
explicitly allows stale lock recovery.
```
