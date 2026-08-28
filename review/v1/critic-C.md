<!-- CRITIC C · hy3-free · family:tencent · pass 2 · 2026-08-28T20:42:44Z -->
CRITIC: hy3-free (family tencent, actor hy3-free@opencode-zen)
DATE: 2026-08-28
PASS: 2
AUTO-TALLIED VERDICT: SALVAGEABLE

---

# Critic review — rogerai-labs--sqlite-for-agents v1

```
CRITIC:    opencode/hy3-free · Hunyuan-lineage · OpenCode Zen (Seat C)
DATE:      2026-08-28
PASS:      2 (panel)
READ:      full manuscript
```

## Verdict summary
This is a tightly-written, technically fluent pocket manual whose central argument — that SQLite, governed by a small set of disciplines, is the correct durable estate for session-bound operators — is demonstrated rather than asserted, and demonstrated well. The listings are internally consistent and the claims about engine behavior track the cited sqlite.org references (verified here against Refs 1, 10, 31). The defects are not errors of fact but gaps between what the book *says its single open ritual does* and what the one function it actually defines *does*, plus a schema/query mismatch in the composition chapter that a reader copying the code would hit. None are fatal; the manuscript is **SALVAGEABLE — findings below**.

## Blocking findings

| # | Location (file:section) | Claim / problem | Evidence | Severity (high/med) |
|---|---|---|---|---|
| 1 | ch03-schema-is-the-handoff.md (`open_estate` def) vs ch05-two-operators-one-file.md (covenant) + ch04/ch07 references | The book repeatedly asserts `open_estate()` is "the single function every tenant opens the estate through: foreign keys on, busy timeout set, WAL on" (ch05) and that durability/WAL settings live there, but the only `open_estate()` actually defined (ch03) issues only `PRAGMA foreign_keys = ON` and runs migrations — it never sets `busy_timeout` and never enables WAL. A reader who copies the defined function gets none of the ch05 concurrency guarantees the book claims are delivered by that ritual. | ch03 listing shows `db.execute("PRAGMA foreign_keys = ON")` only; ch05 covenant text claims three pragmas; no later listing redefines `open_estate` with the missing pragmas. | med |
| 2 | ch04-the-ledger-pattern-and-friends.md (Composition section) | Composition/incident/audit queries are written against `run_id` columns on ledger, settings, and artifacts ("Add run_id columns … ledger entries, settings changes, and artifacts each carrying the run"), but none of the CREATE TABLE statements in ch04 (ledger, cursors, settings, runs, artifacts, queue) declare a `run_id` column. The joined queries in the "Composition" narrative and the "A day in the composed estate" walkthrough therefore reference a column that does not exist in the book's own schema. | ch04 pattern schemas lack `run_id`; Composition prose says "Add run_id columns" as if already done; example `WHERE run_id = 214` has no backing DDL. | med |

## Suggestions (non-blocking)
1. Show a single consolidated `open_estate()` (foreign_keys ON, busy_timeout, WAL, NORMAL-synchronous-under-WAL, migrations) once, and reference it thereafter, so the ritual and the code agree.
2. Either add `run_id INTEGER REFERENCES runs(id)` to the ch04 ledger/settings/artifacts schemas (and SET it in their INSERT examples), or rewrite the Composition queries to use an explicit join key that exists; the "one estate, queryable whole" payoff depends on this.
3. The "keep the index honest" section (ch06) depends on external-content triggers "the FTS5 documentation supplies verbatim" but never shows them; given the book's every-listing-runs ethos, include at least the three trigger bodies or mark the section explicitly as fragment.
4. State the SQLite version floors for features assumed without comment — `RETURNING` (≥3.35), FTS5 (build-dependent), STRICT (≥3.37) — in one place; the audience is told to assume no DBA background.
5. ch02's "Durability's fine print" warns `synchronous=off` corrupts, then ch05 recommends `NORMAL` under WAL; bridge the two explicitly for the rollback-journal reader who enabled WAL later, so the "NORMAL is safe" claim is not misread as blanket.
6. The glossary entry for `upsert` is listed but the book only demonstrates the `ON CONFLICT … DO UPDATE` idiom (never the word "upsert"); minor terminology drift.

## Fact-check sample

| Claim (quoted) | Location | Cited source | Supported? (yes/no/partly) |
|---|---|---|---|
| "SQLite does not compete with client/server databases; SQLite competes with fopen()" | ch01 (The estate's engine) | Ref 1 (sqlite.org/whentouse.html) — verified verbatim | yes |
| "the STRICT table option, added to the language in 2021" + types "INT, INTEGER, REAL, TEXT, BLOB, ANY" | ch03 (Types that mean it / Born versioned) | Ref 10 (sqlite.org/stricttables.html) — 3.37.0 (2021-11-27); allowed types exactly those six | yes |
| "the project pledges support through the year 2050" (file format stability) | ch01 (Estates in the wild) | Ref 31 (sqlite.org/fileformat2.html) — format used since 3.0.0; 2050 stated commitment | yes |
| "Firefox keeps its history, bookmarks, and permissions in SQLite files … Chromium likewise" | ch01 (Estates in the wild) | Ref 3 (sqlite.org/famous.html, "Well-Known Users") | yes |
| rollback-journal mechanics: "before touching the database file, the engine writes the original content of every page … into a sidecar file" | ch02 (The visible mechanics) | Ref 31 (sqlite.org/fileformat2.html) — rollback journal used to restore main file after crash | yes |

## Scores (1–5)
accuracy: 5 · clarity: 5 · completeness-for-tier: 4 · density: 5 · originality: 4
