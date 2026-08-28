<!-- CRITIC B · muse-spark-1.2-contributor-free · family:muse · pass 2 · 2026-08-28T20:41:08Z -->
CRITIC: muse-spark-1.2-contributor-free (family muse, actor muse-spark-1.2-contributor-free@opencode-zen)
DATE: 2026-08-28
PASS: 2
AUTO-TALLIED VERDICT: SALVAGEABLE

---

# Critic review — rogerai-labs--sqlite-for-agents v1

```
CRITIC:    opencode/muse-spark-1.2-contributor-free · Muse Spark 1.2 · OpenCode Zen (Seat B)
DATE:      2026-08-28
PASS:      2 (panel)
READ:      full manuscript
```

## Verdict summary
Durable State for Ephemeral Minds is a pocket-tier standout: narrow thesis, live transcripts, disciplined taxonomy (scratch/records/artifacts), and a coherent estate (ledger/cursor/config/registry/artifact/queue + FTS journal + trust suite) that composes end-to-end. Writing is crisp and source-grounded, with sqlite.org citations and a reproducible gate. The debts are real but surgical — a premature concurrency promise in ch1/ch4, an incomplete ledger invariant, an unsafe reshape recipe, and under-disclosed measurement/pledge context — all fixable without restructuring. SALVAGEABLE — findings below

## Blocking findings
Debts, not advice. Author must fix-with-diff or rebut-with-evidence, every one.

| # | Location (file:section) | Claim / problem | Evidence | Severity (high/med) |
|---|---|---|---|---|
| 1 | ch01-the-amnesiacs-estate.md: The estate's engine (lost-update fix) | Claim that `with db: db.execute("UPDATE counters SET value = value + 1 ...")` alone makes the lost-update "structurally absent" for "two operators... two writers" — single-connection sequential loop is shown as proof, but concurrent two-process/two-connection writers without `PRAGMA busy_timeout` and `BEGIN IMMEDIATE` receive `OperationalError: database is locked` instead of serialized success. Manuscript's own ch05 proves the refusal ( `busy_timeout=0` → `database is locked`, then `busy_timeout=2000` + IMMEDIATE required). Reader copying ch01 pattern to real concurrent case will fail. | ch01 lists `for operator in ("A","B"): with db: UPDATE...` on one `db` handle and prints 2; ch05-two-operators-one-file.md: The refusal, witnessed shows second `BEGIN IMMEDIATE` → `database is locked` without patience; Patience is configuration shows busy_timeout required. Core docs: File Locking And Concurrency (Ref 17) + pragma busy_timeout (Ref 18). | high |
| 2 | ch04-the-ledger-pattern-and-friends.md: The ledger (schema) | Ledger CHECK `CHECK (NOT (outcome IS NOT NULL AND outcome_at IS NULL))` enforces only half of intent-then-outcome invariant — allows nonsensical state `outcome IS NULL AND outcome_at NOT NULL` (timestamp without outcome) and thus a successor-visible fate-unknown row carrying an outcome time. | Schema in ch04: `outcome TEXT, outcome_at TEXT, CHECK (NOT (outcome IS NOT NULL AND outcome_at IS NULL))`; invariant stated prose: intent row + outcome completion together; successor query `WHERE outcome IS NULL` expects clean NULL marker. Fix requires symmetric check `CHECK ((outcome IS NULL) = (outcome_at IS NULL))` or two checks. | high |
| 3 | ch03-schema-is-the-handoff.md: Migrations that move data (fragment) | Sanctioned reshape recipe `CREATE TABLE facts_new ...; INSERT INTO facts_new SELECT ...; DROP TABLE facts; ALTER TABLE facts_new RENAME TO facts;` inside one transaction omits required `PRAGMA foreign_keys=OFF` / `foreign_key_check` and trigger/index handling prescribed by ALTER TABLE documentation (Ref 12). Running as written on an estate with FKs enforced (`PRAGMA foreign_keys=ON` per ch03 ritual) can abort on constraint or leave orphan checks unrun; also silently drops indexes/triggers on old table without recreating. | Text at `ch03: fragment` lists 4 statements; Ref 12 (ALTER TABLE) documents 12-step procedure including `PRAGMA foreign_keys=OFF`, `PRAGMA legacy_alter_table`, recreate indexes/triggers, `PRAGMA foreign_key_check`. Manuscript's open_estate ritual enforces FKs, making omission load-bearing. | high |
| 4 | ch03-schema-is-the-handoff.md: The estate's value conventions + Born versioned | Claims `PRAGMA user_version` bump inside same `with db:` transaction as DDL is atomic and crash-safe ("version bump inside it, so crash mid-migration leaves file honestly at old version"). `PRAGMA user_version` assignment is not transactional in all journal modes and is documented as a header field written outside normal rollback semantics; wrapping it with DDL in one transaction does not guarantee atomicity the text promises. Evidence of intent requires separate docs check or test harness. | Code `with db: db.execute(ddl); db.execute(f"PRAGMA user_version = {version}")` in ch03; Ref 18 Pragma user_version documentation; ch02 mechanics already warns DDL/transaction subtleties. Needs either two-phase proof or rephrase to "applied after DDL commits, retry idempotent" with citation. | med |
| 5 | ch02-one-file-whole-truths.md: The price of a promise, and buying in bulk | Timing transcript `2000 rows, 2000 commits: 22 ms / 2000 rows, 1 commit: 2 ms` presented as measured fact with "order of magnitude" interpretation, but listing never states `PRAGMA journal_mode` or `synchronous` in effect. On authoring machine (NVMe, write caching, default DELETE journal + FULL sync) per-commit cost is flattened vs honest sync hardware where gap is seconds vs milliseconds per ch02's own FAQ citation. Without pragma disclosure the reproducible claim is unfalsifiable and contradicts ch02 Durability's fine print. | Output block at ch02:price section; backmatter note says timings vary; Ref 28 FAQ on INSERT speed + Ref 18 synchronous pragma require disclosure. Fix: print `PRAGMA journal_mode; PRAGMA synchronous` in listing or add annotation. | med |
| 6 | ch05-two-operators-one-file.md: WAL / ch07-trust-verify-repair.md: The backup that lies | Overstates WAL safety as "cannot corrupt the database on power loss" under `synchronous=NORMAL` without the documented qualifier that `NORMAL` risks rollback of recent commits to last checkpoint and that corruption avoidance depends on `wal` persistence and no `PRAGMA journal_mode` change mid-transaction; ch02 correctly conditioned on disks not lying, ch05 drops qualifier. | ch05 Durability under WAL: "cannot corrupt... risks only most recent commits rolling back" — missing "provided WAL file is preserved and checkpoint semantics hold"; Ref 16 Write-Ahead Logging and Ref 22 How To Corrupt both condition this. Needs citation-qualified wording. | med |

## Suggestions (non-blocking)
Structure, ordering, missing topics, tone. Numbered list.

1. Move `open_estate()` definition to ch03 once as canonical, then import/reuse in ch04–08 listings instead of re-defining `CREATE TABLE` inline — reduces drift and reinforces single-ritual point.
2. Add explicit forward reference in ch01 lost-update fix ("full patience recipe in ch5") so early copy-paste is not misused.
3. Add storage-cost sidebar for FTS5 (`secure-delete`, trigram) — pocket readers will underestimate index size for artifact paths.
4. Brief "one-estate-per-lineage" decision tree figure would replace long prose in ch05 Where the file lives — tier expects visuals.
5. Clarify `ITERDUMP` vs `VACUUM INTO` for interchange: note `iterdump` Python-level generation is slower and not transactional snapshot vs engine `VACUUM INTO`.
6. Tone: ch08 garden coda is strong but drifts into general systems philosophy; tighten by explicitly tying leafcutter analogy back to migration-baseline and provenance block.
7. Add index for `user_version` baseline pattern — currently buried in ch08 generation fifty; cross-link to ch03 migration list.

## Fact-check sample
Pass 2: 5% of factual claims, chosen randomly — list claim, cited source, and whether the
source actually supports it. Pass 3: fresh 3% weighted toward revised sections.
A claim whose cited source does not support it = automatic blocking finding above.

| Claim (quoted) | Location | Cited source | Supported? (yes/no/partly) |
|---|---|---|---|
| "SQLite does not compete with client/server databases. SQLite competes with fopen()." | ch01: The estate's engine / ch08 Where the estate ends; Ref 1 | https://sqlite.org/whentouse.html — Appropriate Uses For SQLite | yes — verbatim on page; also local storage / <100K hits/day context matches |
| "STRICT table option, added to the language in 2021 precisely for schemas that mean what they say" | ch03: Types that mean it; Ref 10 | https://sqlite.org/stricttables.html | yes — page states STRICT exists as of version 3.37.0 (2021-11-27) |
| "Declared types must come from the engine's real repertoire (INT, INTEGER, REAL, TEXT, BLOB, ANY)" | ch03: Types that mean it; Ref 10 | https://sqlite.org/stricttables.html | yes — allowed datatypes in STRICT are exactly INT, INTEGER, REAL, TEXT, BLOB, ANY |
| "SQLite's classic implementation of atomic commit is the rollback journal: before touching the database file, the engine writes the original content of every page it is about to change into a sidecar file" | ch02: The visible mechanics; Ref 5+31 | https://sqlite.org/atomiccommit.html + https://sqlite.org/fileformat2.html | yes — Ref 31 excerpt: during transaction SQLite writes rollback journal (or WAL) used to restore main file after crash |
| "the file format is cross-platform and backwards-compatible, and the project pledges support through the year 2050" | ch01: Estates in the wild; ch03; Ref 31 | https://sqlite.org/fileformat2.html (File Format stability) | partly — excerpt supplied verifies format stable since 3.0.0 (2004-06-18) and rollback/WAL mechanics; 2050 commitment is stated on sqlite.org (LTS/fileformat page footer) but not contained in the truncated excerpt provided for this seat; operator must rerun with full page to verify citation covers pledge verbatim |
| "WAL mode ... `PRAGMA journal_mode = WAL` ... WAL requires shared memory, so it is unavailable or unsafe on network filesystems" | ch05: WAL: the readers go free; Ref 16+22 | https://sqlite.org/wal.html + https://sqlite.org/howtocorrupt.html | yes — WAL docs state shared-memory WAL-index and network-filesystem locking caveats |

## Scores (1–5)
accuracy: 4 · clarity: 5 · completeness-for-tier: 4 · density: 4 · originality: 4

## Pass-3 only: findings ledger
| Finding # (from Pass 2) | Status: resolved / rebutted-accepted / still-open | Note |
|---|---|---|
| — | N/A — Pass 2 panel | Pass 3 verification not yet run |
