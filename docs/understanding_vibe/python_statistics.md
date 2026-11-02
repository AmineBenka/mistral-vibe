# Mistral Vibe — Python Codebase Analysis

Last updated: 2026-10-19 (v2.26.1)

Deep-dive into the Python side of the codebase. See [Rust Statistics](rust_statistics.md) for the Rust counterpart, and [Statistics](statistics.md) for the cross-language overview.

## Overview

| Metric | Value |
|---|---|
| Total Python files | **2,049** |
| Total Python LOC | **480,254** |
| Source files (`vibe/` + `scripts/`) | 647 |
| Source LOC | 172,576 |
| Test files (`tests/`) | 708 |
| Test LOC | 235,802 |
| Avg lines/file (all) | 234 |
| Test-to-source ratio (lines) | **1.37:1** |

## Code Density (`vibe/` source, 170,846 lines)

| Category | Lines | % |
|---|---|---|
| Blank lines | 22,684 | 13.3% |
| Comment lines (`#`) | 4,793 | 2.8% |
| Docstring markers (`"""`) | 2,468 | 1.4% |
| Actual code | ~140,901 | 82.5% |

Compared to Rust source (80.6% code, 7.2% blank, 4.7% doc comments), Python source has more blank lines and fewer doc comments — consistent with Python's convention of using blank lines for visual separation and Rust's `///` doc-comment culture.

## Language Constructs (`vibe/` source)

| Construct | Count |
|---|---|
| `class` definitions | 2,342 |
| `def` / `async def` functions | 8,916 |
| `async def` functions | 2,091 (23% of all functions) |
| Decorators (`@`) | 1,578 |
| Import statements (`import` / `from`) | 5,771 |

High class count (2,342) reflects the Pydantic-model-heavy architecture. Nearly a quarter of all functions are async, driven by the agent loop and app-server RPC handlers.

## File Size Distribution

### Source files (`vibe/`, 633 files)

| Bucket | Count | % |
|---|---|---|
| < 20 lines | 72 | 11% |
| 20–50 lines | 100 | 16% |
| 50–100 lines | 99 | 16% |
| 100–200 lines | 138 | 22% |
| 200–500 lines | 143 | 23% |
| > 500 lines | 82 | 13% |

### Test files (`tests/`, 708 files)

| Bucket | Count | % |
|---|---|---|
| < 20 lines | 24 | 3% |
| 20–50 lines | 70 | 10% |
| 50–100 lines | 129 | 18% |
| 100–200 lines | 168 | 24% |
| 200–500 lines | 204 | 29% |
| > 500 lines | 114 | 16% |

Test files skew larger than source files: 45% of test files are >200 lines vs 36% of source files. The modal bucket for tests is 200–500 lines.

## Source Modules

### Top-level (`vibe/` + `scripts/`)

| Module | Files | Lines | % of source |
|---|---|---|---|
| `vibe/core` (engine, agent loop, tools, config) | 268 | 65,375 | 38.1% |
| `vibe/app_server` (RPC server, session backends) | 110 | 56,768 | 33.1% |
| `vibe/cli` (Textual TUI) | 177 | 36,117 | 21.1% |
| `vibe/acp` (Agent Client Protocol bridge) | 20 | 5,398 | 3.1% |
| `vibe/setup` (first-run wizards) | 23 | 4,410 | 2.6% |
| `vibe/utils` | 22 | — | — |
| `vibe/observability` | 3 | — | — |
| `vibe/plugins` | 2 | — | — |
| `scripts/` | 6 | 1,599 | 0.9% |
| root (`pytest_vibe.py`) | 1 | 131 | 0.1% |

`vibe/core` and `vibe/app_server` together account for 71% of all Python source. `vibe/app_server` has fewer files (110) than `vibe/core` (268) but nearly as many lines — its files average ~516 lines vs core's ~244 lines.

### `vibe/core` submodules (268 files, 65,375 lines)

| Submodule | Files | Lines |
|---|---|---|
| `tools/` (built-in tools) | 44 | 14,556 |
| `plugins/` | 20 | 9,147 |
| `config/` | 30 | 5,943 |
| `llm/` (model backends) | 16 | 4,378 |
| `agent_loop/` | 4 | 4,313 |
| `skills/` | 16 | 4,193 |
| `session/` | 16 | 3,477 |
| `git/` | 11 | 3,471 |
| `utils/` | 16 | 1,377 |
| `checkpoints/` | 8 | 1,364 |
| `hooks/` | 9 | 1,275 |
| `vibe_code_project/` | 6 | 1,105 |
| `teleport/` | 7 | 1,089 |
| `telemetry/` | 5 | 1,011 |
| `auth/` | 2 | 919 |

Tools are the largest submodule by LOC (14.5K lines across 44 files). Notably, `agent_loop/` is only 4 files but 4.3K lines — the loop is concentrated in `_loop.py` (4,084 lines).

### `vibe/cli` submodules (177 files, 36,117 lines)

| Submodule | Files | Lines |
|---|---|---|
| `textual_ui/` (Textual widgets) | 98 | 26,210 |
| `autocompletion/` | 12 | 2,062 |
| `update_notifier/` | 11 | 993 |
| Root-level `.py` files (~56) | ~56 | ~6,800 |

`textual_ui/` is 73% of the TUI module — 98 widget files totaling 26K lines. This is the module being replaced by the Rust Ratatui TUI.

### `vibe/app_server` (110 files, 56,768 lines)

This module is notably flat — 110 files with no deep subdirectory nesting. The largest files are the unified harness adapters and runtime:

| Lines | File |
|---|---|
| 10,750 | `_unified_harness_backend_adapter.py` |
| 3,217 | `_runtime.py` |
| 2,510 | `protocol.py` |

## Tests (`tests/`, 708 files, 235,802 lines)

### By area

| Area | Files | Lines | Test functions |
|---|---|---|---|
| `tests/app_server` | 110 | 67,542 | ~1,200+ |
| `tests/core` | 143 | 45,353 | — |
| `tests/cli` | 150 | 40,527 | — |
| `tests/tools` | 23 | 15,236 | — |
| `tests/agent_loop` | 39 | 10,706 | — |
| `tests/acp` | 24 | 7,516 | — |
| `tests/backend` | 15 | 6,586 | — |
| `tests/snapshots` | 46 | 5,277 | — |
| `tests/session` | 9 | 4,637 | — |
| `tests/e2e` | 19 | 4,498 | — |
| `tests/skills` | 11 | 2,955 | — |
| `tests/stubs` | 24 | 2,236 | — |
| Other root-level | ~120 | ~15,000 | — |

### Test framework usage

| Metric | Count |
|---|---|
| `def test_*` functions | 5,326 |
| `async def test_*` functions | 4,124 (44% of tests) |
| Total test functions | **9,450** |
| `@pytest.mark` usages | 4,876 |
| `@pytest.fixture` definitions | 269 |
| `@pytest.mark.parametrize` usages | 640 |

Nearly half of all tests are async — driven by the async-heavy agent loop and app-server code. 640 parametrized tests indicate good use of data-driven testing.

### Largest test files

| Lines | Test functions | File |
|---|---|---|
| 20,309 | 432 | `tests/app_server/test_unified_harness_backend_adapter.py` |
| 4,032 | 104 | `tests/app_server/test_session.py` |
| 3,737 | 151 | `tests/tools/test_bash.py` |
| 3,591 | 111 | `tests/app_server/test_local.py` |
| 2,827 | 138 | `tests/core/test_worktree.py` |
| 2,296 | 115 | `tests/core/config/test_config_resolution.py` |
| 2,135 | — | `tests/session/test_session_logger.py` |

### Test files with most test functions

| Tests | File |
|---|---|
| 432 | `tests/app_server/test_unified_harness_backend_adapter.py` |
| 151 | `tests/tools/test_bash.py` |
| 138 | `tests/core/test_worktree.py` |
| 115 | `tests/core/config/test_config_resolution.py` |
| 111 | `tests/app_server/test_local.py` |
| 104 | `tests/app_server/test_session.py` |
| 89 | `tests/tools/test_mcp.py` |
| 87 | `tests/core/hooks/test_hooks.py` |
| 85 | `tests/tools/test_windows_shell.py` |
| 83 | `tests/tools/test_granular_permissions.py` |

## Largest Source Files

| Lines | File | Module |
|---|---|---|
| 10,750 | `vibe/app_server/_unified_harness_backend_adapter.py` | app_server |
| 6,730 | `vibe/cli/textual_ui/app.py` | cli |
| 4,084 | `vibe/core/agent_loop/_loop.py` | core |
| 3,217 | `vibe/app_server/_runtime.py` | app_server |
| 2,510 | `vibe/app_server/protocol.py` | app_server |
| 2,493 | `vibe/core/tools/builtins/experimental_bash.py` | core |
| 2,127 | `vibe/core/plugins/_native.py` | core |

## Observations

1. **`_unified_harness_backend_adapter.py` is the largest file in the entire repo** at 10,750 lines — 60% larger than the next Python file. It's a god-object that adapts the unified harness backend to the app-server interface.
2. **`vibe/app_server` is dense**: 110 flat files averaging ~516 lines each. The module has no subdirectory nesting, suggesting rapid growth without deep reorganization.
3. **`textual_ui/` dominates the Python TUI**: 98 files / 26K lines (73% of `vibe/cli`). This is the code being replaced by the Rust Ratatui TUI.
4. **The agent loop is concentrated**: `agent_loop/_loop.py` alone is 4,084 lines across just 4 files in the module. The loop logic lives in one large file.
5. **Tests outnumber source files**: 708 test files for 633 source files. 44% of tests are async, matching the async-heavy architecture.
6. **The largest test file mirrors the largest source file**: `test_unified_harness_backend_adapter.py` (20,309 lines, 432 tests) tests the 10,750-line adapter — both are outliers.
7. **Python source files average more lines than Rust**: 271 lines/file (Python source) vs 213 lines/file (Rust source). Python also has more tiny files (<20 lines: 11% vs 4%) and a fatter tail (>500 lines: 13% vs 7%), reflecting both config/data modules and large adapter files.
