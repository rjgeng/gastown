# DRAFT PR — staged, NOT pushed (human gate, hq-999)

**Do not push or open without explicit operator approval** (CLAUDE.md governance §A).

- **Branch (local only):** `fix/compact-depends-on-column-names` @ `892eb069`
- **Base:** `c8130ad9` (same base as #4172 / `fix/reaper-depends-on-column-names`)
- **Push target on approval:** `fork` remote (rjgeng/gastown), then PR to gastownhall/gastown
- **Commands on approval:**
  ```bash
  cd /Users/rfvitis/temp/gastown
  git push fork fix/compact-depends-on-column-names
  gh pr create --repo gastownhall/gastown --head rjgeng:fix/compact-depends-on-column-names \
    --title "fix(compact): use correct column names in orphaned wisp_dependencies sweep" \
    --body-file DRAFT_PR_hq-999.md   # (strip this header block first)
  ```

---

## Title

fix(compact): use correct column names in orphaned wisp_dependencies sweep

## Body

### Problem

`gt compact`'s post-compact sweep (`cleanOrphanedWispDeps` in
`internal/cmd/compact.go`) deletes orphaned `wisp_dependencies` rows with:

```sql
DELETE FROM wisp_dependencies WHERE
  NOT EXISTS (SELECT 1 FROM wisps WHERE id = wisp_dependencies.issue_id)
  OR NOT EXISTS (SELECT 1 FROM wisps WHERE id = wisp_dependencies.depends_on_id)
```

But `wisp_dependencies` has no `depends_on_id` column. The actual schema
(confirmed via `DESCRIBE wisp_dependencies`) splits the target across three
nullable columns: `depends_on_wisp_id`, `depends_on_issue_id`,
`depends_on_external`. Every compaction cycle the DELETE fails with:

```
Error 1105 (HY000): table "wisp_dependencies" does not have column "depends_on_id"
```

so orphaned dependency rows accumulate indefinitely. Observed in a deacon
compaction report on 2026-06-05.

This is the same bug pattern as #4172, which fixed seven references in
`internal/reaper/reaper.go` (reaper purge path) but did not touch the
compaction path.

### Fix

Check each set target column against its own table, mirroring the #4172
column mapping (`depends_on_wisp_id` → `wisps`, `depends_on_issue_id` →
`issues`):

```sql
DELETE FROM wisp_dependencies WHERE
  NOT EXISTS (SELECT 1 FROM wisps WHERE id = wisp_dependencies.issue_id)
  OR (depends_on_wisp_id IS NOT NULL AND NOT EXISTS (SELECT 1 FROM wisps WHERE id = wisp_dependencies.depends_on_wisp_id))
  OR (depends_on_issue_id IS NOT NULL AND NOT EXISTS (SELECT 1 FROM issues WHERE id = wisp_dependencies.depends_on_issue_id))
```

The `IS NOT NULL` guards matter: rows whose only target is
`depends_on_external` must never be swept by this cleanup.

### Verification

- `go build ./...`, `go vet ./internal/cmd/`, `go test ./internal/cmd/ -run 'Compact|Orphan'` — all pass
- Old predicate reproduced the exact production error against a live Dolt DB (read-only SELECT form)
- New predicate executes cleanly against the same DB (SELECT COUNT form; 251 rows total, 0 currently orphaned post-gc)
- `gt compact --dry-run` with the patched binary: 183 wisps scanned, no errors

### Related

- #4172 — same column bug in the reaper purge path
- Remaining `depends_on_id` references elsewhere in the tree (convoy, dolt-snapshots plugin, wisps_migrate CREATE TABLE template, misclassified_wisp_check) are tracked separately; this PR is scoped to the compaction sweep.
