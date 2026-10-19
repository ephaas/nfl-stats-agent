# 0001. Hosting

Copied from `PLAN.md` Phase 0 before any measuring.

## Pre-registered criteria

- **A passes** if all three hold:
  - no out-of-memory restarts, with peak memory under 450 MB
  - warm p95 of 5 seconds or less, with zero 10-second timeouts
  - cold start of 90 seconds or less
  If cold start fails because of image size, try a trimmed image built by us.
- **B passes** if the subset fits in 0.9 GB or less, warm p95 is 2 seconds or less, and cold start is 5 seconds or less. Known downside: production would run a Postgres SQL tool instead of nfl-mcp, so production would no longer be the evaluated system.
- **C (Azure for Students) is a write-up only.** No sign-up and no deploy; you decide after seeing the numbers.
Cost of nfl-mcp's template (0.5 vCPU, 1 GiB, 24/7), at the [official prices](https://azure.microsoft.com/en-us/pricing/details/container-apps/) with the [idle-billing rules](https://learn.microsoft.com/en-us/azure/container-apps/billing):
  - **Expected (mostly idle):** about $10.20 a month, or about 9.8 months of the $100 credit ($10–12 a month realistically, 8–10 months).
  - **Worst case (billed active 24/7):** about $34 a month, or about 2.9 months.
  - **Other costs:** logs have 5 GB a month free, so they cost nothing here.
  - **Student status ends around June 2027**, so the demo likely stops then whatever credit is left.
  - **Never upgrade to pay-as-you-go; any card prompt means stop.**
  - **Open risk:** a $0.10/hour "Environment Management Hour" meter (about $73 a month). Check in the no-sign-in pricing calculator that it doesn't apply.

## Status

Criteria pre-registered (commit `fb680ae`, file criteria-only). Local Phase 0 task 0.1 numbers filled below. Hosting route is not chosen. These local figures are not Render warm p95 or cold start, so they do not score option A. Option B's 0.9 GB size condition is noted as evidence only; B's warm p95 and cold start were not measured here.

## Path override

Command:

```
nfl-mcp ingest --help
```

Help lists `--dataset`, `--start`, `--end`, `--fresh`, `--skip-views`, `--list`, and `--help`. It documents no output-path flag. The copy was therefore written with `NFL_MCP_DB_PATH` (nfl-mcp's documented env override). `~/.nfl-mcp/config.json` was not edited.

Copy path: `/Users/elihaas/.nfl-mcp/nflread-2022-2026.duckdb`

Ingest command (write target printed by nfl-mcp as `Database : /Users/elihaas/.nfl-mcp/nflread-2022-2026.duckdb`):

```
NFL_MCP_DB_PATH="$HOME/.nfl-mcp/nflread-2022-2026.duckdb" nfl-mcp ingest --start 2022 --end 2026
```

## Original database fingerprints

File: `/Users/elihaas/.nfl-mcp/nflread.duckdb`

Commands:

```
shasum -a 256 "$HOME/.nfl-mcp/nflread.duckdb"
stat -f '%z %m %Sm' "$HOME/.nfl-mcp/nflread.duckdb"
```

| When | SHA-256 | size (bytes) | mtime unix | mtime |
| --- | --- | --- | --- | --- |
| Before ingest | `116cbf85a48e2ed2ad243e5e1e43e8c5f1aed59185ccb69b2bc1e84f6d8cc388` | 705441792 | 1791860757 | Oct 12 20:05:57 2026 |
| After ingest, serve, and queries | `116cbf85a48e2ed2ad243e5e1e43e8c5f1aed59185ccb69b2bc1e84f6d8cc388` | 705441792 | 1791860757 | Oct 12 20:05:57 2026 |

Unchanged.

## Query set

Same six SQL strings on both databases. Query 6 ran as written (no fallback).

1. `SELECT COUNT(*) AS n FROM plays`
2. `SELECT COUNT(*) AS n FROM plays WHERE season BETWEEN 2022 AND 2026`
3. `SELECT season, COUNT(*) AS n FROM plays WHERE season BETWEEN 2022 AND 2026 GROUP BY season ORDER BY season`
4. `SELECT posteam, ROUND(AVG(epa), 4) AS avg_epa FROM plays WHERE season = 2024 AND season_type = 'REG' AND play_type IN ('pass', 'run') GROUP BY posteam ORDER BY avg_epa DESC LIMIT 10`
5. `SELECT * FROM plays LIMIT 5`
6. `SELECT p.season, COUNT(*) AS n FROM plays p JOIN schedules s ON p.game_id = s.game_id WHERE p.season BETWEEN 2022 AND 2026 GROUP BY p.season ORDER BY p.season`

Caps for serve and in-process queries: `NFL_MCP_DUCKDB_MEMORY_LIMIT=256MB`, `NFL_MCP_DUCKDB_THREADS=1`. Interpreter: `/Users/elihaas/.local/share/uv/tools/nfl-mcp/bin/python`. Host: macOS Darwin. p50 is the median of five timed runs after one warmup of query 1 (`time.perf_counter`, milliseconds). Peak RSS is `resource.getrusage(resource.RUSAGE_SELF).ru_maxrss` in **bytes** on macOS. `ps -o rss=` idle RSS is **kilobytes** on macOS.

## Measurements

Full path: `$HOME/.nfl-mcp/nflread.duckdb` (2013–2026). Copy path: `$HOME/.nfl-mcp/nflread-2022-2026.duckdb` (2022–2026).

| Metric | Full 2013–2026 | 2022–2026 copy | Command |
| --- | --- | --- | --- |
| File size (bytes) | 705441792 | 289419264 | `stat -f%z "$HOME/.nfl-mcp/nflread.duckdb"` and `stat -f%z "$HOME/.nfl-mcp/nflread-2022-2026.duckdb"` |
| File size (`ls -lh`) | 673M | 276M | `ls -lh "$HOME/.nfl-mcp/nflread.duckdb"` and `ls -lh "$HOME/.nfl-mcp/nflread-2022-2026.duckdb"` |
| Idle serve RSS (`ps -o rss=`, KB) | 77056 | 86784 | See serve commands below |
| Peak in-process RSS (`ru_maxrss`, bytes) | 311001088 | 282263552 | See query commands below |
| q1 p50 / max (ms) | 15.41 / 15.64 | 15.50 / 15.72 | See query commands below |
| q2 p50 / max (ms) | 15.85 / 17.30 | 15.48 / 15.80 | See query commands below |
| q3 p50 / max (ms) | 16.31 / 16.48 | 16.09 / 20.18 | See query commands below |
| q4 p50 / max (ms) | 17.31 / 20.28 | 17.24 / 20.82 | See query commands below |
| q5 p50 / max (ms) | 32.52 / 47.58 | 30.88 / 48.23 | See query commands below |
| q6 p50 / max (ms) | 19.86 / 20.24 | 19.09 / 19.87 | See query commands below |

Subset file size 289419264 bytes is 0.270 GiB (0.289 × 10^9 bytes). That is below B's "0.9 GB or less" size condition. This is **evidence for that one B condition only**, not a B pass/fail verdict (warm p95 and cold start for Neon were not measured).

### Serve idle RSS commands

Copy (listened on `http://127.0.0.1:8765/mcp`; `NFL_MCP_DB_PATH=/Users/elihaas/.nfl-mcp/nflread-2022-2026.duckdb`; never the original path):

```
NFL_MCP_DB_PATH="$HOME/.nfl-mcp/nflread-2022-2026.duckdb" NFL_MCP_DUCKDB_MEMORY_LIMIT=256MB NFL_MCP_DUCKDB_THREADS=1 nfl-mcp serve --port 8765
ps -o rss= -p 65816
```

Result: `86784` (KB).

Full (optional like-for-like; stopped before query measurements):

```
NFL_MCP_DB_PATH="$HOME/.nfl-mcp/nflread.duckdb" NFL_MCP_DUCKDB_MEMORY_LIMIT=256MB NFL_MCP_DUCKDB_THREADS=1 nfl-mcp serve --port 8766
ps -o rss= -p 65831
```

Result: `77056` (KB).

### Query latency and peak RSS commands

Copy:

```
NFL_MCP_DB_PATH="$HOME/.nfl-mcp/nflread-2022-2026.duckdb" NFL_MCP_DUCKDB_MEMORY_LIMIT=256MB NFL_MCP_DUCKDB_THREADS=1 \
  /Users/elihaas/.local/share/uv/tools/nfl-mcp/bin/python /tmp/measure_nfl_db.py
```

Full (separate process):

```
NFL_MCP_DB_PATH="$HOME/.nfl-mcp/nflread.duckdb" NFL_MCP_DUCKDB_MEMORY_LIMIT=256MB NFL_MCP_DUCKDB_THREADS=1 \
  /Users/elihaas/.local/share/uv/tools/nfl-mcp/bin/python /tmp/measure_nfl_db.py
```

`/tmp/measure_nfl_db.py` contents used for those two runs:

```python
import json
import os
import resource
import sys
import time

import nfl_mcp.server  # noqa: F401  # match PLAN.md "server imports loaded"
from nfl_mcp.config import get_duckdb_path
from nfl_mcp.tools import nfl_query

QUERIES = [
    ("q1", "SELECT COUNT(*) AS n FROM plays"),
    ("q2", "SELECT COUNT(*) AS n FROM plays WHERE season BETWEEN 2022 AND 2026"),
    (
        "q3",
        "SELECT season, COUNT(*) AS n FROM plays WHERE season BETWEEN 2022 AND 2026 GROUP BY season ORDER BY season",
    ),
    (
        "q4",
        "SELECT posteam, ROUND(AVG(epa), 4) AS avg_epa FROM plays WHERE season = 2024 AND season_type = 'REG' AND play_type IN ('pass', 'run') GROUP BY posteam ORDER BY avg_epa DESC LIMIT 10",
    ),
    ("q5", "SELECT * FROM plays LIMIT 5"),
    (
        "q6",
        "SELECT p.season, COUNT(*) AS n FROM plays p JOIN schedules s ON p.game_id = s.game_id WHERE p.season BETWEEN 2022 AND 2026 GROUP BY p.season ORDER BY p.season",
    ),
]
FALLBACK_Q6 = (
    "SELECT season, week, COUNT(*) AS n FROM plays WHERE season BETWEEN 2022 AND 2026 "
    "GROUP BY season, week ORDER BY n DESC LIMIT 20"
)


def has_error(result):
    if not isinstance(result, dict):
        return True
    if result.get("error"):
        return True
    text = json.dumps(result).lower()
    return "error" in result or '"error"' in text


def main():
    resolved = str(get_duckdb_path())
    warmup = nfl_query(QUERIES[0][1])
    if has_error(warmup):
        print(json.dumps({"ok": False, "phase": "warmup", "resolved": resolved, "result": warmup}))
        sys.exit(1)

    rows = []
    q6_used = QUERIES[5][1]
    q6_substituted = False
    for name, sql in QUERIES:
        used = sql
        if name == "q6":
            probe = nfl_query(sql)
            if has_error(probe):
                used = FALLBACK_Q6
                q6_used = used
                q6_substituted = True
        times_ms = []
        last = None
        for _ in range(5):
            t0 = time.perf_counter()
            last = nfl_query(used)
            t1 = time.perf_counter()
            times_ms.append((t1 - t0) * 1000)
            if has_error(last):
                print(
                    json.dumps(
                        {
                            "ok": False,
                            "phase": name,
                            "sql": used,
                            "resolved": resolved,
                            "result": last,
                        }
                    )
                )
                sys.exit(1)
        times_ms.sort()
        rows.append(
            {
                "name": name,
                "sql": used,
                "times_ms": times_ms,
                "p50_ms": times_ms[2],
                "max_ms": times_ms[-1],
                "row_count_hint": last.get("count") if isinstance(last, dict) else None,
            }
        )

    peak_rss = resource.getrusage(resource.RUSAGE_SELF).ru_maxrss
    print(
        json.dumps(
            {
                "ok": True,
                "resolved": resolved,
                "NFL_MCP_DB_PATH": os.environ.get("NFL_MCP_DB_PATH"),
                "NFL_MCP_DUCKDB_MEMORY_LIMIT": os.environ.get("NFL_MCP_DUCKDB_MEMORY_LIMIT"),
                "NFL_MCP_DUCKDB_THREADS": os.environ.get("NFL_MCP_DUCKDB_THREADS"),
                "peak_rss_ru_maxrss": peak_rss,
                "peak_rss_unit": "bytes on macOS Darwin",
                "q6_substituted": q6_substituted,
                "q6_sql": q6_used,
                "queries": rows,
            },
            indent=2,
        )
    )


if __name__ == "__main__":
    main()
```
