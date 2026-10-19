# Spec Delta

## Purpose

Records a hosting decision against pre-registered pass/fail criteria, with every local nfl-mcp measurement tied to the command that produced it and the original DuckDB left unmodified.

## ADDED Requirements

### Requirement: Criteria are committed before any measuring
The first committed version of `docs/decisions/0001-hosting.md` MUST contain only the Phase 0 pass/fail criteria for options A, B, and C copied from `PLAN.md`, and MUST contain no measured numbers, command outputs, or size, memory, or latency results.

#### Scenario: First commit has criteria only
- **WHEN** the decision record is first committed
- **THEN** the file includes the A, B, and C criteria from `PLAN.md` Phase 0 and includes no measured numbers

#### Scenario: Measuring starts only after that commit
- **WHEN** work to ingest a 2022–2026 copy or to measure size, memory, or latency is about to begin
- **THEN** that criteria-only commit already exists on the current branch

### Requirement: Option A pass conditions
The decision record MUST state that option A (Render) passes only if all three hold: no out-of-memory restarts with peak memory under 450 MB; warm p95 of 5 seconds or less with zero 10-second timeouts; and cold start of 90 seconds or less, with a trimmed image tried if cold start fails because of image size.

#### Scenario: A criteria are copied
- **WHEN** the criteria-only decision record is written
- **THEN** it includes those three A conditions and the trimmed-image fallback for image-size cold starts, matching `PLAN.md` Phase 0

### Requirement: Option B pass conditions
The decision record MUST state that option B (Neon) passes if the subset fits in 0.9 GB or less, warm p95 is 2 seconds or less, and cold start is 5 seconds or less, and MUST record B's known downside: production would run a Postgres SQL tool instead of nfl-mcp, so production would no longer be the evaluated system.

#### Scenario: B criteria are copied
- **WHEN** the criteria-only decision record is written
- **THEN** it includes those three B conditions and the known downside, matching `PLAN.md` Phase 0

### Requirement: Option C is a write-up only
The decision record MUST state that option C (Azure for Students) is a write-up only: no sign-up and no deploy. It MUST copy the Phase 0 Azure cost figures, student-status end date, never-upgrade-to-pay-as-you-go rule, card-prompt stop rule, and the open Environment Management Hour risk.

#### Scenario: C write-up is copied
- **WHEN** the criteria-only decision record is written
- **THEN** it includes the C write-up and cost figures from `PLAN.md` Phase 0 and does not instruct anyone to sign up or deploy on Azure

### Requirement: Every measured number cites its command
After local measurements, every number in the decision record MUST appear with the exact command that produced it so another person can reproduce the figure.

#### Scenario: Size, memory, and latency rows are reproducible
- **WHEN** the decision record is updated with local measurements
- **THEN** each recorded size, memory, and latency number is next to the exact command that produced it

#### Scenario: No unsourced figures
- **WHEN** a reader checks the measurements section
- **THEN** there is no measured number without an accompanying command

### Requirement: Original DuckDB is never modified
Work MUST NOT modify `~/.nfl-mcp/nflread.duckdb`. Before ingest and after ingest and serve, the record MUST include checksum, size, and mtime of that file showing they are unchanged.

#### Scenario: Original fingerprints match
- **WHEN** ingest of the 2022–2026 copy and local serve against that copy have finished
- **THEN** checksum, size, and mtime of `~/.nfl-mcp/nflread.duckdb` match the values taken before ingest

#### Scenario: Ingest is not aimed at the original path
- **WHEN** `nfl-mcp ingest --start 2022 --end 2026` is run
- **THEN** its write target is not `~/.nfl-mcp/nflread.duckdb`

### Requirement: Subset database is a separate 2022–2026 copy
The 2022–2026 database MUST be written to a path other than `~/.nfl-mcp/nflread.duckdb`. That path MUST be taken from `nfl-mcp ingest --help`; if `--help` documents no output-path flag, the path MUST be set with `NFL_MCP_DB_PATH`.

#### Scenario: Help is consulted first
- **WHEN** the subset ingest is prepared
- **THEN** `nfl-mcp ingest --help` has been run and the chosen path override is the one that help (or, if help is silent, `NFL_MCP_DB_PATH`) documents

#### Scenario: Copy covers 2022–2026 only
- **WHEN** the subset database exists
- **THEN** it was produced with `--start 2022 --end 2026` at the separate path and is not the original file

### Requirement: Local comparison covers size, memory, and latency
The decision record MUST compare the 2022–2026 copy to the full 2013–2026 database on file size, peak memory, and query latency, using the same query set and the same DuckDB memory and thread caps on both.

#### Scenario: Both databases are measured the same way
- **WHEN** measurements are recorded
- **THEN** the record includes size, peak memory, and latency for both the full 2013–2026 file and the 2022–2026 copy, from the same commands aside from the database path

### Requirement: This change does not pick a host
The decision record MUST NOT rank or select A, B, or C from these local measurements alone, and MUST NOT include Render deploy results, provider tool-calling checks, or AI SDK token counts.

#### Scenario: Hosting route stays undecided
- **WHEN** this change's measurements are written into the decision record
- **THEN** the record still has no chosen host and has no Render, Gemini, Groq, or qwen3 tool-calling numbers
