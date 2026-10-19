# Tasks

## 1. Pre-register criteria

- [x] 1.1 Create `docs/decisions/0001-hosting.md` by copying the Phase 0 "Pre-registered criteria" block from `PLAN.md` for options A, B, and C (including B's known downside and the C Azure cost write-up). Verify the file contains those thresholds and dollar figures verbatim and contains no measured numbers.
- [x] 1.2 Commit only `docs/decisions/0001-hosting.md` (do not amend later). Verify `git log -1 --name-only` shows that file, `git show HEAD:docs/decisions/0001-hosting.md` has no size/memory/latency results, and this commit exists before any ingest or timing command is run.

## 2. Build the 2022–2026 copy without touching the original

- [x] 2.1 Run `nfl-mcp ingest --help` and record the path-override it documents (if none, use `NFL_MCP_DB_PATH` per design). Verify the chosen override is not a rewrite of `~/.nfl-mcp/nflread.duckdb` and that `~/.nfl-mcp/config.json` will not be edited.
- [x] 2.2 Fingerprint `~/.nfl-mcp/nflread.duckdb` with `shasum -a 256` and `stat -f '%z %m %Sm'`. Verify all three values are saved for comparison after ingest.
- [x] 2.3 Ingest only into `$HOME/.nfl-mcp/nflread-2022-2026.duckdb` using `NFL_MCP_DB_PATH` set to that path and `nfl-mcp ingest --start 2022 --end 2026` (no `--fresh`, no `nfl-mcp init`). Verify the copy exists, the original path was never the write target, and the original fingerprints from 2.2 are unchanged.

## 3. Size, serve, and query comparison

- [x] 3.1 Measure file size of both DuckDB files with `stat -f%z` and `ls -lh`. Verify both byte counts and human-readable sizes are on hand with those exact commands.
- [x] 3.2 Start `nfl-mcp serve` against the 2022–2026 copy only (`NFL_MCP_DB_PATH` set to the copy, `NFL_MCP_DUCKDB_MEMORY_LIMIT=256MB`, `NFL_MCP_DUCKDB_THREADS=1`), record idle RSS with `ps -o rss= -p <pid>`, then stop the server. Verify it listened and that serve was never pointed at `~/.nfl-mcp/nflread.duckdb`. Optionally repeat against the full file for like-for-like idle RSS, stopping it before 3.3.
- [x] 3.3 With serve stopped, for each database in a separate Python process (`NFL_MCP_DB_PATH`, `NFL_MCP_DUCKDB_MEMORY_LIMIT=256MB`, `NFL_MCP_DUCKDB_THREADS=1`), import nfl-mcp, warmup once, then run the six design.md queries five times. Verify p50 and max latency (`time.perf_counter`) and peak RSS (`resource.getrusage`; label macOS units) exist for both files from the same queries. If query 6 fails on schema, use the design.md fallback and keep that command.
- [x] 3.4 Re-fingerprint `~/.nfl-mcp/nflread.duckdb` with the same commands as 2.2. Verify SHA-256, size, and mtime match 2.2; if not, stop and do not write measurements.

## 4. Record measurements in the decision file

- [x] 4.1 Append to `docs/decisions/0001-hosting.md` without changing the criteria section: status (criteria pre-registered; local 0.1 filled; host not chosen), help quote and path used, fingerprints before/after, the six SQL strings, and a table of size, memory, and latency for both databases. Verify every number sits next to the exact command that produced it, there is no A/B/C selection, no Render/provider/token figures, no `web/` files, and subset size vs 0.9 GB is noted as evidence only, not a B verdict.
