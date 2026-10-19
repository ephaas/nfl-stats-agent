# Proposal

## Why

Phase 0 must choose a $0 host for nfl-mcp using criteria written down before any numbers exist. Task 0.1's local size, memory, and latency comparison of a 2022–2026 DuckDB against the full 2013–2026 file is the first evidence that record needs, and it must not touch the original database at `~/.nfl-mcp/nflread.duckdb`.

## What Changes

- Add `docs/decisions/0001-hosting.md` in two steps: first commit copies only the Phase 0 pass/fail criteria for options A (Render), B (Neon), and C (Azure write-up) from `PLAN.md`; a later edit records measured numbers, each with the exact command that produced it.
- Build a 2022–2026 copy of the nfl-mcp DuckDB at a separate path discovered from `nfl-mcp ingest --help` (with `NFL_MCP_DB_PATH` as the documented override if `--help` has no output-path flag). Compare that copy to the full 2013–2026 database on size, memory, and latency.
- Verify the original file at `~/.nfl-mcp/nflread.duckdb` is unchanged (checksum, size, mtime) before and after ingest and serve.

Out of scope: task 0.2 (AI SDK script and schema token counts), task 0.3 (provider tool-calling checks), the Render deploy and remote measurements in task 0.4, task 0.5 (ranking A/B/C after Render numbers), and any code under `web/`.

## Capabilities

### New Capabilities

- `hosting-decision`: Pre-register hosting pass/fail criteria before measuring, then fill the decision record with reproducible local nfl-mcp measurements without modifying the original DuckDB.

### Modified Capabilities

- None. `openspec/specs/` is empty.

## Impact

- New documentation only: `docs/decisions/0001-hosting.md`. No application code, no `web/`, no new npm or uv packages.
- Local filesystem: a second DuckDB at a path other than `~/.nfl-mcp/nflread.duckdb` (gitignored by the existing `*.duckdb` rule if it ever lands in the repo; the copy should live under `~/.nfl-mcp/` with a distinct filename).
- Uses already-installed nfl-mcp 0.3.3. Ingest downloads nflverse data for 2022–2026 into the copy only.
- Does not choose a host. Render/Neon runtime numbers and the A-then-B-then-C ranking stay in later Phase 0 work.
