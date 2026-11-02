# Checkpoints and Rewind

This document explains `vibe/core/checkpoints/` and `vibe/core/rewind/`: how Vibe
snapshots files during agent turns, and how those snapshots are used to undo
work (rewind) and how they behave when a session is resumed.

Related reading: [ADR 0006 Local Sessions](docs/adr/0006-local-sessions.md)
(rewind persistence modes) and [ADR 0003 Event Driven Agent Loop](docs/adr/0003-event-driven-agent-loop.md).

## Overview

```
                    ┌──────────────────────────┐
  agent loop ──────▶│ CheckpointRecorder       │  write shell: turn lifecycle,
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

The design is "pure core, impure shells": one `Checkpointer` instance is shared
by the recorder, the rewind manager and the review manager (wired in
`AgentLoop.__init__`, `vibe/core/agent_loop/_loop.py`). Only `FileStore` reads
or writes files.

## `vibe/core/checkpoints/`

| File | Role |
| --- | --- |
| `models.py` | Public value types: `FileState`, `Region`, `OpaqueChange`, `RegionId`, owners (`AgentTurn`, `ManualEdit`), `Decision`, `TurnRegion`, `HunkAnchor`, errors. |
| `_events.py` | Private log events: `_TurnMark`, `_Edit`, `_Decide`. |
| `checkpointer.py` | `Checkpointer`: the append-only log and all mutations (turns, edits, decisions, truncation, fork). |
| `history.py` | `History`: pure read model computed from a list of events. |
| `recorder.py` | `CheckpointRecorder` + `FileSnapshot`: drives the per-turn snapshot lifecycle from the agent loop. |
| `file_store.py` | `FileStore`: reads `FileState`s and applies restore plans to disk. |
| `fs.py` | `Filesystem` port + `DiskFilesystem` (fakeable in tests). |

### The snapshot: `FileState`

A snapshot is a whole-file byte copy, `FileState(data: bytes | None)`:

- `data is None` → the file did not exist (restoring it means **deleting** it).
- `b"\x00" in data` → treated as binary; it is not line-diffed but handled as
  an `OpaqueChange` (whole-file unit).

`DiskFilesystem.read_bytes` returns `None` only for a missing file and raises
for an unreadable one, so a transient read error is never mistaken for a
deletion.

### The event log

`Checkpointer` keeps a single ordered list of events, each with a monotonic
`seq`:

| Event | Meaning |
| --- | --- |
| `_TurnMark(seq, turn_id, pre)` | Start of an agent turn. `turn_id` is the index of the user message in the conversation. `pre` maps each file the turn touched to its state **before** the turn's first edit. |
| `_Edit(seq, owner, path, before, after, deps)` | One file's `before → after` change, owned by an `AgentTurn` or a `ManualEdit`. `deps` records, for each hunk, the earlier hunks it was built on. |
| `_Decide(seq, path, hunk, keep)` | A review decision (keep/revert) on one hunk. |

Everything else — current content, the approved baseline, hunks, restore
plans — is **derived** by `History` from this log. Nothing is cached on the
side.

### Turn lifecycle (how snapshots are taken)

Driven by `AgentLoop` around each user turn:

1. **`create_checkpoint()`** (turn start): `begin_turn(len(messages))` appends a
   `_TurnMark`. Every file touched in the previous turn is re-read from disk and
   recorded as a pre-edit state, so files mutated by tools that produce no
   snapshot (e.g. a shell command) are still captured.
2. **`add_snapshot(snapshot)`** (before each approved tool call): tools that
   write files (`write_file`, `edit`, see `BaseTool.get_file_snapshot`) return a
   `FileSnapshot(path, state)` of the file *before* they run. `record_pre_edit`
   keeps only the **first** snapshot per path in a turn.
3. **`seal_turn()`** (turn end, in a `finally`): every touched path is re-read
   as its post-edit state; each `pre != post` pair becomes an `_Edit` owned by
   `AgentTurn(turn_id)`. Unchanged files produce no event. A file that fails to
   read is logged and skipped, without blocking the others.

### Manual edits

Changes the user makes by hand between turns are detected by comparing live
disk with the log's projection:

- In `record_pre_edit`: if a tracked file already drifted when a new turn first
  touches it, the drift is inserted as a `ManualEdit` **just before** that
  turn's mark (it happened before the turn).
- In `reconcile(path, current)` (called by `ReviewManager` at each acting
  boundary): drift is appended as a `ManualEdit`. It is idempotent and skipped
  while a turn is open.

### Hunks and review decisions

`History` splits each `_Edit` into line hunks (`Region`, via `difflib`) or one
`OpaqueChange`. A hunk's identity is `RegionId(version_index=edit.seq,
ordinal)`, stable as the log grows. Decisions are per hunk:

- **Keep** pulls in the pending hunks it depends on.
- **Revert** drags its dependents (computed at read time).
- There is no "un-revert"; a reverted change comes back only by truncating the
  log or with a new edit.

`Checkpointer.atomic()` rolls the log back if the disk write that follows a
decision fails. Decisions are refused while a turn is open.

### Restore plans

`History.restore_plan(index)` answers "what should each file look like if the
log were cut at event `index`?":

- A file touched by a dropped **turn** restores to the `pre` of the earliest
  dropped turn that touched it (its state when that turn began).
- A file touched only by a dropped **manual edit** restores to its projection
  of the kept prefix.
- `FileState(None)` means delete the file.

`restore_plan_to_turn(turn_id)` resolves the turn's mark (preferring the newest
exact match, else the next later turn) and calls `restore_plan`.
`FileStore.apply(plan)` writes it, skipping files already in the target state,
and returns `(errors, restored_paths)`.

## `vibe/core/rewind/`

`RewindManager` (`manager.py`) is the impure shell that undoes the
conversation, and optionally the files, back to an earlier user message.

| Method | Purpose |
| --- | --- |
| `get_rewindable_messages()` | `(index, content)` for every non-injected user message. |
| `index_for_message_id(id)` | Resolves a message id to its index, or raises `RewindError`. |
| `restorable_paths_at(index)` / `has_file_changes_at(index)` | Files whose disk content would actually change if rewinding to that turn (plan target ≠ current disk). Used by clients to offer "restore files". |
| `rewind_to_message(index, *, restore_files, inplace=False)` | Performs the rewind. Returns `(message_content, restore_errors, restored_paths)`. |

### Undo: `rewind_to_message`

1. Validates that `index` points at a user message.
2. If `restore_files`: applies `restore_plan_to_turn(index)` to disk. The turn
   started at that user message and every later turn are undone, as are manual
   edits recorded after it.
3. Truncates the conversation to `messages[:index]` and the log with
   `drop_turns_from(index)`. The rewound user message content is returned so the
   client can put it back in the input box.
4. Persists according to the mode (see ADR 0006):
   - **Fork (default, `inplace=False`)**: save the full history to the current
     session, truncate, then `_reset_session()` creates a new session whose
     parent is the original. Nothing is lost.
   - **In place (`inplace=True`)**: truncate, then save the truncated history
     under the same session (`allow_empty=True`, so rewinding to the first
     message saves an empty conversation). The rewound turns are gone for good.

File restoration and conversation persistence are independent: rewinding with
`restore_files=False` leaves the disk as it is.

### Message resets

`RewindManager` subscribes to `MessageList.on_reset`. Any reset that is not the
rewind's own truncation (session switch, `/clear`, …) clears the checkpoint log.
If a turn was open, a fresh one is begun so the rest of the tool loop keeps
recording snapshots.

## Snapshots and resume

The checkpoint log lives **only in memory**. It is not written to the session
log, so:

- **Resuming a session** (`--resume`/`--continue`, session picker) calls
  `checkpoint_recorder.reset()` (`_reset_session_scoped_state`) and starts with
  an empty log. Rewinding to a message from before the resume truncates the
  conversation but restores **no files**: `restorable_paths_at` returns `[]`
  and the client does not offer a file restore.
- Turns run after the resume are recorded normally and can be rewound with
  files restored.
- **Forked rewinds** keep working within a process. On the app-server unified
  backend (`vibe/app_server/_unified_harness_snapshots.py`),
  `RewindFileSnapshots.forked()` copies the log with
  `Checkpointer.prefix_before(turn)`, which re-bases `seq` and keeps the
  manual-edit counter so inherited region ids never collide.
- In short: the conversation side of a rewind (fork or in place) is always
  durable, because it goes through the session log; the file side is only
  available for turns recorded in the current process.

### App-server unified backend

`RewindFileSnapshots` reuses `Checkpointer` but records disk states at the turn
boundaries the adapter observes, because the harness runtime writes files
itself. Its documented gaps: a file first touched by `edit` or a shell command
has no pre-state; subagent writes are not recorded; sandboxed sessions use
`UnavailableRewindFileSnapshots` and restore nothing.

## Tests

- `tests/core/rewind/` — rewind manager behaviour.
- `client-e2e/scenarios/rewind/` + `client-e2e/goldens/rewind/` — end-to-end
  rewind scenarios for the Rust TUI.
- `vibe/cli-rust/src/rewind/` — Rust TUI rewind UI (a thin client over the
  app-server).
