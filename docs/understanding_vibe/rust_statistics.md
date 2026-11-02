# Mistral Vibe — Rust Codebase Analysis

Last updated: 2026-10-19 (v2.26.1)

Deep-dive into the Rust side of the codebase. See [Python Statistics](python_statistics.md) for the Python counterpart, and [Statistics](statistics.md) for the cross-language overview.

## Overview

| Metric | Value |
|---|---|
| Total Rust files | **794** |
| Total Rust LOC | **153,857** |
| Source files (`src/`) | 560 |
| Source LOC | 119,347 |
| Test files | 234 |
| Test LOC | 34,510 |
| Avg lines/file (source) | 213 |
| Avg lines/file (all) | 194 |
| Test-to-source ratio (lines) | **0.29:1** |

Two crates: `vibe-rs` (Ratatui TUI) and `harness-core` (app-server harness).

## Crate Breakdown

| Crate | Path | Files | LOC | Source LOC | Test LOC |
|---|---|---|---|---|---|
| `vibe-rs` (Ratatui TUI) | `vibe/cli-rust/` | 650 | 102,163 | 67,653 | 34,510 |
| `harness-core` (app-server harness) | `harness/core/` | 144 | 51,694 | 35,323 | 16,371 |

## Code Density (source only, 119,347 lines)

| Category | Lines | % |
|---|---|---|
| Blank lines | 8,604 | 7.2% |
| Line comments (`//`) | 8,340 | 7.0% |
| Doc comments (`///`) | 5,626 | 4.7% |
| Module doc comments (`//!`) | 612 | 0.5% |
| Block comment starts (`/*`) | 9 | <0.1% |
| Actual code | ~96,156 | 80.6% |

Compared to Python source (82.5% code, 13.3% blank, 2.8% comments), Rust source is denser: fewer blank lines, more comments (especially doc comments at 4.7% vs Python docstrings at 1.4%).

## Language Constructs (source only)

| Construct | Count |
|---|---|
| `struct` declarations | 417 |
| `enum` declarations | 173 |
| `trait` declarations | 4 |
| `impl` blocks | 359 |
| `async fn` declarations | 145 |
| `#[derive(...)]` attributes | 649 |
| `pub use` re-exports | 86 |
| `macro_rules!` definitions | 1 |
| `unsafe` blocks (files containing) | 11 files, 39 mentions |

Notable: only 4 traits and 1 macro definition — the codebase favors concrete types and enums over trait-based abstraction. `unsafe` usage is minimal (11 files out of 560).

## File Size Distribution

### Source files (560 files)

| Bucket | Count | % |
|---|---|---|
| < 20 lines | 23 | 4% |
| 20–50 lines | 60 | 11% |
| 50–100 lines | 111 | 20% |
| 100–200 lines | 160 | 29% |
| 200–500 lines | 165 | 29% |
| > 500 lines | 42 | 7% |

### Test files (234 files)

| Bucket | Count | % |
|---|---|---|
| < 20 lines | 7 | 3% |
| 20–50 lines | 46 | 20% |
| 50–100 lines | 55 | 24% |
| 100–200 lines | 69 | 29% |
| 200–500 lines | 64 | 27% |
| > 500 lines | 18 | 8% |

Rust files skew larger than Python: the modal bucket is 100–500 lines (58% of source files), vs Python's modal bucket of 50–200 lines (37%).

## Crate: `vibe-rs` (Ratatui TUI)

### Source modules (`src/`, 416 files, 67,653 lines)

| Module | Files | Lines |
|---|---|---|
| `ui/` (widgets, transcript, markdown) | 108 | 16,226 |
| `setup/` (onboarding flow) | 31 | 4,972 |
| `server/` (app-server client) | 26 | 3,783 |
| `utils/` | 22 | 2,799 |
| `commands/` | 14 | 2,104 |
| `update_notifier/` | 10 | 2,102 |
| `mcp/` | 11 | 1,486 |
| `selection/` | 9 | 1,455 |
| `message_queue/` | 11 | 1,452 |
| `subagents/` | 7 | 1,379 |
| `startup/` | 7 | 1,241 |
| `event_loop/` | 8 | 1,184 |
| `transcript/` | 8 | 1,030 |
| `vibe_code_project/` | 6 | 989 |
| `question_app/` | 4 | 694 |
| `plugins/` | 5 | 657 |
| `observability/` | 6 | 650 |
| `voice/` | 5 | 525 |
| `rewind/` | 4 | 549 |
| `resume_picker/` | 3 | 547 |
| Root-level `.rs` files (~150) | ~150 | ~19,000 |

#### `ui/` submodules (108 files, 16,226 lines)

| Submodule | Files | Lines |
|---|---|---|
| `transcript/` (message rendering) | 22 | 3,211 |
| `markdown/` (syntax highlighting) | 12 | 1,971 |
| `banner/` | 3 | 1,025 |
| `theme/` | 4 | 858 |
| `approval/` (permission prompts) | 4 | 527 |
| `loading/` | 4 | 506 |
| `mcp/` | 2 | 364 |

The `ui/` module is the largest by far — 24% of all TUI source. Transcript rendering and markdown highlighting dominate.

### Tests (`tests/`, 234 files, 34,510 lines)

| Area | Files | Lines |
|---|---|---|
| `tests/units/` (unit tests) | 151 | 21,997 |
| `tests/` root (integration tests) | ~83 | ~12,513 |

Total test functions: **1,835** (`#[test]` + `#[tokio::test]`)

#### Largest unit test files

| Lines | Tests | File |
|---|---|---|
| 634 | 22 | `tests/units/subagent_transcript.rs` |
| 525 | 38 | `tests/units/markdown_parse.rs` |
| 490 | 28 | `tests/units/message_queue.rs` |
| 411 | 21 | `tests/units/subagent_list.rs` |
| 405 | 23 | `tests/units/mcp_browser.rs` |

#### Test files with most test functions

| Tests | File |
|---|---|
| 38 | `tests/worktree.rs` |
| 38 | `tests/units/markdown_parse.rs` |
| 30 | `harness/.../acceptance_scenarios/compaction.rs` |
| 29 | `tests/units/headless_logic.rs` |
| 28 | `tests/units/message_queue.rs` |

### Largest source files (`vibe/cli-rust/src/`)

| Lines | File |
|---|---|
| 1,236 | `app.rs` |
| 1,026 | `event_handler.rs` |
| 937 | `input.rs` |
| 868 | `transcript.rs` |
| 579 | `bin/replay.rs` |
| 464 | `cli.rs` |
| 457 | `main.rs` |
| 402 | `headless.rs` |

## Crate: `harness-core` (app-server harness)

### Source (`src/core/`, 113 files, 35,226 lines)

| Module | Files | Lines |
|---|---|---|
| `features/` (agent capabilities) | 41 | 14,876 |
| `checkpoint/` (session serialization) | 19 | 6,290 |
| `tools/` (tool execution) | 14 | 4,541 |
| `turn.rs` | 1 | 1,354 |
| `turn/` | 4 | 1,090 |
| `step_protocol/` | 5 | 886 |
| `wire/` (RPC protocol types) | 6 | 771 |
| Root-level `.rs` files (~60) | ~60 | ~5,500 |

#### `features/` submodules (41 files, 14,876 lines)

| Feature | Files | Lines |
|---|---|---|
| `programmatic_tool_calling/` (code-mode, V8) | 11 | 7,110 |
| `tool_discovery/` | 4 | 2,555 |
| `large_output/` | 2 | 1,283 |
| `compaction/` | 4 | 1,226 |
| `permissions/` | 6 | 964 |
| `subagents/` | 4 | 454 |
| `background_processes/` | 3 | 406 |
| `file_system/` | 1 | 325 |
| `skills/` | 3 | 299 |
| `notifications/` | 2 | 244 |

`programmatic_tool_calling` is nearly half the features module — it includes a V8 JavaScript engine binding (`v8.rs`, 3,646 lines) for sandboxed code-mode tool execution.

### Tests (`src/tests/`, 31 files, 16,371 lines)

| Area | Files | Lines |
|---|---|---|
| `acceptance_scenarios/` | 14 | 9,813 |
| Root-level test files | 10 | 4,558 |
| `mod.rs` | 1 | — |

#### Largest acceptance scenario files

| Lines | File |
|---|---|
| 1,683 | `compaction.rs` |
| 1,225 | `permissions/programmatic.rs` |
| 1,103 | `completion_and_tools.rs` |
| 1,005 | `runtime_driver.rs` |
| 935 | `permissions/direct.rs` |

### Largest source files (`harness/core/src/core/`)

| Lines | File |
|---|---|
| 3,646 | `features/programmatic_tool_calling/code_mode/v8.rs` |
| 1,717 | `features/programmatic_tool_calling/execution.rs` |
| 1,354 | `turn.rs` |
| 1,216 | `features/programmatic_tool_calling/state.rs` |
| 1,041 | `tools/execution/batch.rs` |
| 1,018 | `features/tool_discovery/mod.rs` |

## Largest Rust Files (entire repo)

| Lines | File | Crate |
|---|---|---|
| 3,646 | `harness/.../code_mode/v8.rs` | harness-core |
| 1,717 | `harness/.../execution.rs` | harness-core |
| 1,683 | `harness/.../compaction.rs` (test) | harness-core |
| 1,354 | `harness/.../turn.rs` | harness-core |
| 1,236 | `vibe/cli-rust/src/app.rs` | vibe-rs |
| 1,225 | `harness/.../permissions/programmatic.rs` (test) | harness-core |
| 1,216 | `harness/.../state.rs` | harness-core |
| 1,103 | `harness/.../completion_and_tools.rs` (test) | harness-core |
| 1,041 | `harness/.../batch.rs` | harness-core |
| 1,026 | `vibe/cli-rust/src/event_handler.rs` | vibe-rs |

## Observations

1. **Harness dominates by LOC-per-file**: harness-core averages 246 lines/file (source) vs vibe-rs at 163 lines/file. The harness is denser, more logic-heavy code; the TUI is more spread across many small widget files.
2. **`v8.rs` is the single largest Rust file** at 3,646 lines — nearly 3x the next largest source file. It's a V8 JavaScript engine binding for code-mode tool execution and is an extreme outlier.
3. **UI is the biggest TUI module**: `ui/` at 108 files / 16K lines is 24% of all TUI source. Transcript rendering (3.2K lines) and markdown highlighting (2K lines) are the two biggest sub-areas.
4. **Low trait usage**: Only 4 trait definitions across 560 source files. The codebase favors enums and concrete types over trait-based polymorphism — a pragmatic choice for a CLI app.
5. **Minimal `unsafe`**: 11 files out of 560 use `unsafe` (39 mentions total). The V8 bindings and possibly FFI for audio/terminal are the likely hotspots.
6. **Acceptance scenarios are the biggest test category** in harness-core: 14 files / 9.8K lines of end-to-end behavior tests, reflecting the harness's role as a testable app-server backend.
7. **Async is pervasive but not dominant**: 145 `async fn` declarations across 560 source files (~26%). The TUI uses tokio for the event loop; the harness is more synchronous with targeted async for I/O.
8. **Rust test ratio is lower than Python**: 0.29 test lines per source line vs Python's 1.37:1. However, Rust's 1,835 test functions across 234 test files is still substantial coverage, and Rust's type system catches more errors at compile time.
