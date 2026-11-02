# Mistral Vibe — Codebase Statistics

Last updated: 2026-10-19 (v2.26.1)

## Overview

| Metric | Value |
|---|---|
| Total source files | **2,858** |
| Total lines of code | **~634,700** |
| Git commits | 96 |
| Python packages (dirs with `__init__.py`) | 70 |
| Rust crates | 2 (`vibe-rs`, `harness-core`) |

## By Language

| Language | Files | Total LOC | Source LOC | Test LOC | Avg lines/file | Test/Source ratio |
|---|---|---|---|---|---|---|
| Python | 2,049 | 480,254 | 172,576 | 235,802 | 234 | **1.37:1** |
| Rust | 794 | 153,857 | 119,347 | 34,510 | 194 | **0.29:1** |
| Shell + TCSS | 15 | ~3,977 | — | — | ~265 | — |

## Disk Usage

| Directory | Size |
|---|---|
| `vibe/` | 13 MB |
| `tests/` | 12 MB |
| `harness/` | 4.2 MB |
| `scripts/` | 108 KB |

## Project Metadata

- **Python**: 3.12+, managed with `uv`, Apache-2.0 license
- **Rust**: Edition 2021, two crates (`vibe-rs` for the Ratatui TUI, `harness-core` for the app-server harness)
- **Key Python deps**: `agent-client-protocol`, `httpx`, `textual`, `pydantic`, `gitpython`, `croniter`
- **Key Rust deps**: `ratatui 0.29`, `tokio`, `serde`, `clap 4`, `crossterm`
- **Version**: 2.26.1

## Deep Dives

- [Python Statistics](python_statistics.md) — module breakdowns, code density, language constructs, file size distribution, largest files
- [Rust Statistics](rust_statistics.md) — crate breakdowns, code density, language constructs, file size distribution, largest files

## Observations

1. **Dual-language codebase**: The project is mid-migration from a Python Textual TUI (`vibe/cli`, 177 files) to a Rust Ratatui TUI (`vibe/cli-rust`, 416 source files). The Rust TUI is already larger by file count.
2. **Python has heavier test coverage**: 1.37 test lines per source line vs Rust's 0.29:1. Python compensates for dynamic typing with more tests; Rust relies more on the type system.
3. **Python source is sparser**: 82.5% actual code vs Rust's 80.6%, but Python has more blank lines (13.3% vs 7.2%) and fewer doc comments (1.4% vs 4.7%).
4. **Both favor small files**: 27% of Python source and 17% of Rust source files are under 50 lines, consistent with a modular one-concept-per-file architecture.
