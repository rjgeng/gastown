# hq-999 completion report — polecat co_ops/rust, 2026-06-05

(Written to disk because town Dolt went down mid-session and the hook bead was
hard-deleted mid-flight; this file is the durable record.)

## Assignment

hq-999: deacon compaction orphaned wisp_deps cleanup fails — wrong column name
(`depends_on_id` vs `depends_on_wisp_id`/`depends_on_issue_id`). Mayor's
instructions: fix in /Users/rfvitis/temp/gastown, branch from #4172's base,
build + verify dry-run, stage branch + draft PR locally, DO NOT push (human
gate), report back to mayor.

## Status: STAGED — awaiting operator approval to push

- **Branch (local only):** `fix/compact-depends-on-column-names` @ `892eb069`,
  based on `c8130ad9` (same base as #4172)
- **Change:** `internal/cmd/compact.go` — `cleanOrphanedWispDeps()` DELETE now
  uses `depends_on_wisp_id`→wisps / `depends_on_issue_id`→issues, with
  `IS NOT NULL` guards so external-only deps are never swept (+doc comment).
- **Draft PR:** `DRAFT_PR_hq-999.md` (this directory) — title, body, and
  push/PR commands for after approval.

## Mayor's "check both" question answered

#4172's diff touches ONLY `internal/reaper/reaper.go`. It never covered the
compaction path — the deacon binary patch level is irrelevant; the compaction
SQL was simply never fixed.

## Verification

- `go build ./...`, `go vet ./internal/cmd/`,
  `go test ./internal/cmd/ -run 'Compact|Orphan'` — all pass
- Old predicate (read-only SELECT form) against live town DB reproduces the
  exact production error: `Error 1105: table "wisp_dependencies" does not have
  column "depends_on_id"`
- New predicate executes cleanly (251 wisp_dependencies rows, 0 currently
  orphaned — town had a recent wisp gc)
- `make build` + `gt compact --dry-run` against the town with the patched
  binary: 183 wisps scanned, no errors (orphan sweep is `!dryRun`-guarded by
  design; the SQL was verified read-only as above)
- Schema confirmed via `DESCRIBE wisp_dependencies` AND `DESCRIBE dependencies`:
  both have `depends_on_issue_id`/`depends_on_wisp_id`/`depends_on_external`,
  no `depends_on_id`

## Discovered work

- **hq-u8w** (filed): audit remaining `depends_on_id` references —
  plugins/dolt-snapshots/main.go:386, internal/cmd/convoy.go:484-497,
  internal/cmd/sling_convoy.go:31, internal/cmd/convoy_stage.go:1540,
  internal/doltserver/wisps_migrate.go:260,462-469 (CREATE TABLE template
  still emits the old single-column schema), 
  internal/doctor/misclassified_wisp_check.go:243

## Anomalies — CORRECTED 05:28 per mayor

1. ~~hq-999 + molecule hard-deleted mid-flight~~ **FALSE ALARM.** Mayor
   confirmed (mail hq-wisp-e6q9, 05:28): hq-999 survived intact (HOOKED,
   assigned to this polecat). The "vanished" beads at ~05:09 were **silent
   read failures during the Dolt outage** — `bd show`/`bd sql` returned
   empty results / "no issue found" instead of errors before the circuit
   breaker started failing fast. Worth noting for the audit trail: empty
   reads during a Dolt degradation are indistinguishable from deletions.
2. **Dolt outage ~05:09–05:2x**: `dolt circuit breaker is open: server
   appears down` on all bd reads+writes from this polecat, while `gt dolt
   status` still reported the server running. `gt escalate` also fails
   during such an outage (it writes a bead) — escalated via `gt nudge`
   instead. Per governance the polecat did not touch the server; resolved
   by operator-approved restart, all DBs verified (per mayor).
