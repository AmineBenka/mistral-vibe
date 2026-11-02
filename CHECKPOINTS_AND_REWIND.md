# Checkpoints & Rewind

> **Undo for agents.** Vibe snapshots every file an agent turn touches, so you can
> roll back both the conversation *and* your working tree to any earlier
> message — without ever losing the original session.

This guide covers `vibe/core/checkpoints/` (how snapshots are taken and stored)
and `vibe/core/rewind/` (how they are used to undo work), plus how both behave
across session resume.

**Related ADRs:** [0006 Local Sessions](docs/adr/0006-local-sessions.md) (rewind
persistence modes) · [0003 Event Driven Agent Loop](docs/adr/0003-event-driven-agent-loop.md)

---

## Contents

- [At a glance](#at-a-glance)
- [Architecture](#architecture)
- [Checkpoints — `vibe/core/checkpoints/`](#checkpoints--vibecorecheckpoints)
  - [Snapshots: `FileState`](#snapshots-filestate)
  - [The event log](#the-event-log)
  - [Turn lifecycle](#turn-lifecycle)
  - [Manual edits](#manual-edits)
  - [Hunks and review decisions](#hunks-and-review-decisions)
  - [Restore plans](#restore-plans)
- [Rewind — `vibe/core/rewind/`](#rewind--vibecorerewind)
  - [How a rewind runs](#how-a-rewind-runs)
  - [Message resets](#message-resets)
- [Snapshots and resume](#snapshots-and-resume)
- [Tests](#tests)

---

## At a glance

**Using it.** Type `/rewind`, or press `Esc` twice on an empty input, then pick
the user message to return to. You choose whether to restore files as well as
the conversation; the rewound message is placed back in the input box, ready to
edit and resend.

**What you can rely on:**

| Guarantee | How |
| --- | --- |
| **Nothing is lost by default** | Rewinds fork a new session; the original is saved untouched. |
| **Files and conversation are independent** | Restore the conversation alone, or both together. |
| **Your own edits are respected** | Hand edits made between turns are detected and tracked as `ManualEdit`s. |
| **Deletions are never invented** | A read error is never mistaken for a missing file. |
| **Shell side effects are captured** | Files touched by snapshot-less tools are re-read at turn boundaries. |
| **Single source of truth** | All state is derived from one append-only event log; nothing is cached on the side. |

---

## Architecture

Vibe follows a **pure core, impure shells** design. A single `Checkpointer`
instance is shared by the recorder, the rewind manager and the review manager
(wired in `AgentLoop.__init__`, `vibe/core/agent_loop/_loop.py`). Only
`FileStore` ever touches the disk.

```
                    ┌──────────────────────────┐
  agent loop ──────▶│ CheckpointRecorder       │  impure shell: turn lifecycle,
                    │ (recorder.py)            │  reads disk at boundaries
                    └────────────┬─────────────┘
                                 ▼
                    ┌──────────────────────────┐
                    │ Checkpointer             │  pure, in-memory, append-only
                    │ (checkpointer.py)        │  event log (never touches disk)
                    └────────────┬─────────────┘
                                 │ view()
                                 ▼
                    ┌──────────────────────────┐
                    │ History (history.py)     │  pure read model: projections,
                    │                          │  hunks, restore plans
                    └────────────┬─────────────┘
               ┌─────────────────┴────────────────┐
               ▼                                  ▼
   ┌──────────────────────┐          ┌──────────────────────────┐
   │ RewindManager        │          │ ReviewManager            │
   │ (rewind/manager.py)  │          │ (review/manager.py)      │
   │ undo to a user msg   │          │ per-hunk keep / revert   │
   └──────────┬───────────┘          └────────────┬─────────────┘
              └──────────────┬────────────────────┘
                             ▼
                    ┌──────────────────────────┐
                    │ FileStore → Filesystem   │  the only disk I/O
                    │ (file_store.py, fs.py)   │
                    └──────────────────────────┘
```

---

## Checkpoints — `vibe/core/checkpoints/`

| File | Responsibility |
| --- | --- |
| `models.py` | Public value types: `FileState`, `Region`, `OpaqueChange`, `RegionId`, owners (`AgentTurn`, `ManualEdit`), `Decision`, `TurnRegion`, `HunkAnchor`, errors. |
| `_events.py` | Private log events: `_TurnMark`, `_Edit`, `_Decide`. |
| `checkpointer.py` | `Checkpointer` — the append-only log and every mutation (turns, edits, decisions, truncation, fork). |
| `history.py` | `History` — pure read model computed from a list of events. |
| `recorder.py` | `CheckpointRecorder` + `FileSnapshot` — drives the per-turn snapshot lifecycle from the agent loop. |
| `file_store.py` | `FileStore` — reads `FileState`s and applies restore plans to disk. |
| `fs.py` | `Filesystem` port + `DiskFilesystem` adapter (fakeable in tests). |

### Snapshots: `FileState`

A snapshot is a whole-file byte copy, `FileState(data: bytes | None)`:

- **`data is None`** — the file did not exist; restoring it means **deleting** it.
- **`b"\x00" in data`** — treated as binary: not line-diffed, but handled as a
  single `OpaqueChange`.

> [!NOTE]
> `DiskFilesystem.read_bytes` returns `None` **only** for a missing file and
> raises for an unreadable one, so a transient read error is never mistaken
> for a deletion.

### The event log

`Checkpointer` keeps one ordered list of events, each stamped with a monotonic
`seq`:

| Event | Meaning |
| --- | --- |
| `_TurnMark(seq, turn_id, pre)` | Start of an agent turn. `turn_id` is the index of the user message in the conversation; `pre` maps each file the turn touched to its state **before** the turn's first edit. |
| `_Edit(seq, owner, path, before, after, deps)` | One file's `before → after` change, owned by an `AgentTurn` or a `ManualEdit`. `deps` records, per hunk, the earlier hunks it was built on. |
| `_Decide(seq, path, hunk, keep)` | A review decision (keep / revert) on one hunk. |

Everything else — current content, the approved baseline, hunks, restore
plans — is **derived** by `History` from this log.

### Turn lifecycle

`AgentLoop` drives three hooks around every user turn:

| Phase | Hook | What happens |
| --- | --- | --- |
| **1. Turn start** | `create_checkpoint()` | `begin_turn(len(messages))` appends a `_TurnMark`. Every file touched in the previous turn is re-read from disk as a pre-edit state, so files mutated by snapshot-less tools (e.g. a shell command) are still captured. |
| **2. Before each tool call** | `add_snapshot(snapshot)` | File-writing tools (`write_file`, `edit`; see `BaseTool.get_file_snapshot`) return a `FileSnapshot(path, state)` taken *before* they run. `record_pre_edit` keeps only the **first** snapshot per path in a turn. |
| **3. Turn end** (`finally`) | `seal_turn()` | Every touched path is re-read as its post-edit state; each `pre != post` pair becomes an `_Edit` owned by `AgentTurn(turn_id)`. Unchanged files emit nothing. A file that fails to read is logged and skipped without blocking the others. |

### Manual edits

Changes you make by hand between turns are detected by comparing live disk
with the log's projection:

- **`record_pre_edit`** — if a tracked file has drifted when a new turn first
  touches it, the drift is inserted as a `ManualEdit` **just before** that
  turn's mark (it happened before the turn).
- **`reconcile(path, current)`** — called by `ReviewManager` at each acting
  boundary; drift is appended as a `ManualEdit`. Idempotent, and skipped while
  a turn is open.

### Hunks and review decisions

`History` splits each `_Edit` into line hunks (`Region`, via `difflib`) or a
single `OpaqueChange`. A hunk's identity,
`RegionId(version_index=edit.seq, ordinal)`, stays stable as the log grows.
Decisions are made per hunk:

- **Keep** pulls in the pending hunks it depends on.
- **Revert** drags its dependents along (computed at read time).
- **No un-revert** — a reverted change comes back only by truncating the log or
  through a new edit.

`Checkpointer.atomic()` rolls the log back if the disk write following a
decision fails. Decisions are refused while a turn is open.

### Restore plans

`History.restore_plan(index)` answers one question: *what should each file look
like if the log were cut at event `index`?*

- A file touched by a dropped **turn** restores to the `pre` of the earliest
  dropped turn that touched it — its state when that turn began.
- A file touched only by a dropped **manual edit** restores to its projection
  of the kept prefix.
- `FileState(None)` means *delete the file*.

`restore_plan_to_turn(turn_id)` resolves the turn's mark (newest exact match,
else the next later turn) and delegates to `restore_plan`.
`FileStore.apply(plan)` writes the result, skips files already in the target
state, and returns `(errors, restored_paths)`.

---

## Rewind — `vibe/core/rewind/`

`RewindManager` (`manager.py`) is the impure shell that winds the conversation —
and optionally the files — back to an earlier user message.

| Method | Purpose |
| --- | --- |
| `get_rewindable_messages()` | `(index, content)` for every non-injected user message. |
| `index_for_message_id(id)` | Resolves a message id to its index, or raises `RewindError`. |
| `restorable_paths_at(index)` / `has_file_changes_at(index)` | Files whose disk content would actually change when rewinding to that turn (plan target ≠ current disk). Clients use this to decide whether to offer "restore files". |
| `rewind_to_message(index, *, restore_files, inplace=False)` | Performs the rewind; returns `(message_content, restore_errors, restored_paths)`. |

### How a rewind runs

1. **Validate** — `index` must point at a user message.
2. **Restore files** *(if `restore_files`)* — apply `restore_plan_to_turn(index)`.
   The turn started at that message and every later turn are undone, along with
   manual edits recorded after it.
3. **Truncate** — the conversation to `messages[:index]` and the log via
   `drop_turns_from(index)`. The rewound message content is returned so the
   client can put it back in the input box.
4. **Persist** according to the selected mode (ADR 0006):

| Mode | Behaviour | Original history |
| --- | --- | --- |
| **Fork** *(default, `inplace=False`)* | Save the full history to the current session, truncate, then `_reset_session()` creates a new session whose parent is the original. | ✅ Preserved |
| **In place** (`inplace=True`) | Truncate, then save under the same session (`allow_empty=True`, so rewinding to the first message saves an empty conversation). | ❌ Discarded for good |

> [!TIP]
> File restoration and conversation persistence are independent: rewinding with
> `restore_files=False` leaves the disk exactly as it is.

### Message resets

`RewindManager` subscribes to `MessageList.on_reset`. Any reset other than the
rewind's own truncation (session switch, `/clear`, …) clears the checkpoint log.
If a turn was open, a fresh one begins so the rest of the tool loop keeps
recording snapshots.

---

## Snapshots and resume

> [!IMPORTANT]
> The checkpoint log lives **only in memory** — it is never written to the
> session log. Conversation rewinds are always durable; **file** restores are
> only available for turns recorded in the current process.

- **Resuming a session** (`--resume` / `--continue`, session picker) calls
  `checkpoint_recorder.reset()` (`_reset_session_scoped_state`) and starts from
  an empty log. Rewinding to a message from before the resume truncates the
  conversation but restores **no files**: `restorable_paths_at` returns `[]`
  and the client does not offer a file restore.
- **Turns run after the resume** are recorded normally and can be rewound with
  files restored.
- **Forked rewinds** keep working within a process. On the app-server unified
  backend (`vibe/app_server/_unified_harness_snapshots.py`),
  `RewindFileSnapshots.forked()` copies the log with
  `Checkpointer.prefix_before(turn)`, which re-bases `seq` and keeps the
  manual-edit counter so inherited region ids never collide.

### App-server unified backend

`RewindFileSnapshots` reuses `Checkpointer` but records disk states at the turn
boundaries the adapter observes, since the harness runtime writes files itself.

> [!WARNING]
> Known gaps on this backend:
> - A file first touched by `edit` or a shell command has no pre-state.
> - Subagent writes are not recorded.
> - Sandboxed sessions use `UnavailableRewindFileSnapshots` and restore nothing.

---

## Tests

| Location | Covers |
| --- | --- |
| `tests/core/rewind/` | Rewind manager behaviour. |
| `client-e2e/scenarios/rewind/` + `client-e2e/goldens/rewind/` | End-to-end rewind scenarios for the Rust TUI. |
| `vibe/cli-rust/src/rewind/` | Rust TUI rewind UI (a thin client over the app-server). |
