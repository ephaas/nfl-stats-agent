# Design

## Context

See `proposal.md` for why. The repo has `PLAN.md`, `AGENTS.md`, `.gitignore` (`*.duckdb` already ignored), and no `docs/` tree. nfl-mcp 0.3.3 is already installed. The live database is `~/.nfl-mcp/nflread.duckdb` (673 MiB, seasons 2013–2026 per `PLAN.md`). `docs/decisions/` does not exist yet.

nfl-mcp's ingest CLI has `--start`, `--end`, `--dataset`, `--fresh`, `--skip-views`, and `--list`. It has no output-path flag. The write target is `get_duckdb_path()`: `NFL_MCP_DB_PATH` if set, otherwise `~/.nfl-mcp/config.json`'s `duckdb_path`, otherwise `~/.nfl-mcp/nflread.duckdb`. Apply still runs `nfl-mcp ingest --help` first and records what it prints; if help is silent on path, use `NFL_MCP_DB_PATH`.

`PLAN.md` section 2 already measured queries on this Mac with DuckDB capped at 256 MB and 1 thread and nfl-mcp server imports loaded. Task 0.1 asks to run `nfl-mcp serve` on a copy and compare size, memory, and latency. Render (out of scope here) is planned with `NFL_MCP_DUCKDB_MEMORY_LIMIT=256MB`.

## Goals / Non-Goals

**Goals:**

- Pre-register A/B/C criteria in git before any ingest or timing.
- Produce a 2022–2026 DuckDB beside the original, never as a rewrite of it.
- Fill the same decision file with size, memory, and latency for both databases, each number next to its command.
- Use the same query set and the same DuckDB caps on both files.

**Non-Goals:**

- Do not add application code, packages, or anything under `web/`.
- Do not call providers, do not deploy, do not rank A/B/C.
- Do not treat the subset file size vs 0.9 GB as a B pass/fail; record the size only.
- Do not install nfl-mcp, DuckDB, or Python packages.

## Decisions

### 1. Two edits of one file, criteria committed first

Write `docs/decisions/0001-hosting.md` with only the Phase 0 "Pre-registered criteria" block from `PLAN.md` (A, B, and the C write-up including Azure cost figures). Commit that file before ingest. After measurements, edit the same file in place (do not amend the criteria commit). Copy criteria text from `PLAN.md`; do not paraphrase the thresholds or dollar figures.

**Alternatives:** Separate `0001-criteria.md` and `0002-measurements.md` — rejected; `PLAN.md` names one file. Git commit `--amend` of the criteria commit after measuring — rejected; that would hide the pre-registration.

### 2. Copy path via env, never via config rewrite

Target: `~/.nfl-mcp/nflread-2022-2026.duckdb`. Ingest:

```bash
NFL_MCP_DB_PATH="$HOME/.nfl-mcp/nflread-2022-2026.duckdb" nfl-mcp ingest --start 2022 --end 2026
```

Do not change `~/.nfl-mcp/config.json` (that would retarget the default database). Do not run `nfl-mcp init`. Do not pass `--fresh`. Do not run ingest unless `NFL_MCP_DB_PATH` is set to a path other than `~/.nfl-mcp/nflread.duckdb`.

**Alternatives:** Copy the original file and DELETE old seasons — rejected; easy to corrupt the original and not what task 0.1 specifies. `/tmp/...` — rejected; ephemeral. A path inside the repo — unnecessary; `.gitignore` would hide it, and `~/.nfl-mcp/` is where nfl-mcp already writes.

### 3. Fingerprint the original before and after

Before ingest and after ingest plus serve, record for `~/.nfl-mcp/nflread.duckdb`: SHA-256 (`shasum -a 256`), byte size and mtime (`stat -f '%z %m %Sm'`). All three MUST match. If they do not, stop and do not keep measuring.

### 4. Measure size from files; measure queries in-process; serve the copy separately

**Size.** `stat -f%z` (bytes) and `ls -lh` on both `.duckdb` files. Record both commands.

**Caps (both databases).** `NFL_MCP_DUCKDB_MEMORY_LIMIT=256MB` and `NFL_MCP_DUCKDB_THREADS=1`, matching `PLAN.md` section 2 and the planned Render cap.

**Latency and peak memory.** One Python process per database: set `NFL_MCP_DB_PATH` to that file, import nfl-mcp the same way section 2 did ("server imports loaded"), run a one-query warmup, then five timed runs of each of the six queries below. Record p50 and max wall time (`time.perf_counter`) and peak RSS (`resource.getrusage(RUSAGE_SELF).ru_maxrss`; on macOS this is bytes). Stop serve against a file before opening it in this process so DuckDB locking does not interfere.

**Serve.** `NFL_MCP_DB_PATH=<copy> NFL_MCP_DUCKDB_MEMORY_LIMIT=256MB NFL_MCP_DUCKDB_THREADS=1 nfl-mcp serve` against the 2022–2026 copy (task 0.1). After it listens, record idle RSS with `ps -o rss= -p <pid>` (KB on macOS) and stop it. Optionally repeat against the full file for a like-for-like idle RSS. Do not add an MCP HTTP client; that is task 0.2.

**Alternatives:** Time queries over Streamable HTTP — closer to production, but needs a session client and overlaps 0.2. DuckDB CLI only — would skip nfl-mcp imports and serve. HTTP plus in-process — extra numbers without a new decision.

### 5. Fixed six queries, valid on both files, no CTEs

Queries MUST be the same on both databases. Seasons used in filters MUST be inside 2022–2026 so the subset can run them. No `WITH` (nfl-mcp rejects CTEs). No keyword-blocklist hits such as `replace()`.

1. `SELECT COUNT(*) AS n FROM plays`
2. `SELECT COUNT(*) AS n FROM plays WHERE season BETWEEN 2022 AND 2026`
3. `SELECT season, COUNT(*) AS n FROM plays WHERE season BETWEEN 2022 AND 2026 GROUP BY season ORDER BY season`
4. `SELECT posteam, ROUND(AVG(epa), 4) AS avg_epa FROM plays WHERE season = 2024 AND season_type = 'REG' AND play_type IN ('pass', 'run') GROUP BY posteam ORDER BY avg_epa DESC LIMIT 10`
5. `SELECT * FROM plays LIMIT 5`
6. `SELECT p.season, COUNT(*) AS n FROM plays p JOIN schedules s ON p.game_id = s.game_id WHERE p.season BETWEEN 2022 AND 2026 GROUP BY p.season ORDER BY p.season`

If query 6 fails because `game_id` is absent on `schedules`, replace the join with `SELECT season, week, COUNT(*) AS n FROM plays WHERE season BETWEEN 2022 AND 2026 GROUP BY season, week ORDER BY n DESC LIMIT 20` and record that substitution and its command.

### 6. Decision record shape after measurements

Keep the criteria section unchanged. Append:

- Status: criteria pre-registered; local 0.1 numbers filled; hosting route not chosen.
- Help output (or a short quote) showing how the copy path was selected.
- Original-file fingerprints before and after.
- A table: metric, full 2013–2026 value, 2022–2026 value, exact command.
- The six SQL strings.
- A note that subset size vs 0.9 GB is evidence for one B condition only, not a B verdict.

No "Decision: A/B/C" heading.

### 7. No new repo files except the decision record

Commands live in `0001-hosting.md`. Do not add `scripts/`, `web/`, or a Python helper. Ask before installing any package (`AGENTS.md`).

## Risks / Trade-offs

- **Ingest without `NFL_MCP_DB_PATH` writes to the original.** → Help first; refuse to run ingest unless the env var is a different path; fingerprint before/after.
- **`--fresh` on a ranged ingest can drop the whole `plays` table** (nfl-mcp AGENTS.md). → Never pass `--fresh`.
- **Ingest while serve holds the same file.** DuckDB is one writer, no concurrent readers with a writer. → Never ingest the original; stop serve before in-process query runs on that path.
- **Subset ingest downloads nflverse (network, time, disk).** → Expected; copy stays under `~/.nfl-mcp/`. If disk is short, stop rather than touching the original.
- **macOS vs Linux RSS units.** `ru_maxrss` is bytes on Darwin; `ps -o rss=` is KB. → Label units in the record next to the command.
- **Local latency is not Render warm p95.** → Record as local 0.1 only; do not score A.

## Migration Plan

Docs-only. Rollback is `git revert` of the criteria commit and/or the measurements commit. The 2022–2026 file can be deleted; it is not a git object. Do not delete `~/.nfl-mcp/nflread.duckdb`.
