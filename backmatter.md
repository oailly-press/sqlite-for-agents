# Back Matter

## Glossary

- **append-and-complete** — the ledger discipline: intent rows are inserted, outcome fields completed, nothing deleted or rewritten; corrections are new rows citing old ones.
- **artifact index** — the estate table vouching for files: path, content hash, origin, fetch date; the files themselves stay on the file system.
- **atomic commit** — the engine's guarantee that a transaction's changes become visible all at once or not at all, held across process death and (at full sync) power loss.
- **briefing** — the successor's opening read: schema version, storage audit, unresolved intents, unfinished runs, staleness — a to-do list with provenance.
- **busy timeout** — the per-connection bound on how long the engine waits for the write slot before returning BUSY; the estate's patience, set once in the open ritual.
- **CHECK constraint** — a schema-enforced predicate on rows; the estate's executable documentation.
- **checkpoint** — folding the write-ahead log back into the main database file; automatic by default, manually truncated at handoff.
- **cursor (estate)** — one row per consumed stream recording the opaque resume position and when it advanced.
- **estate** — the durable, queryable, verified state a session-bound operator leaves for its successors; this book's name for operator memory done properly.
- **flexible typing** — SQLite's historical default where declared column types are affinities, not contracts; retired in estates by STRICT.
- **generated column** — a column derived by the engine from its row's other columns; structure doing a writer's arithmetic.
- **idempotency key** — a stored, UNIQUE-constrained identity for a world-action, scoped to its once-ness, making retries visible as constraint refusals.
- **intent-then-outcome** — recording a world-action before performing it and completing the record after, so a death in the gap leaves a visible open intent instead of silence.
- **info table** — key-value rows naming the estate's purpose, owner, conventions, and backup location; the file's title page.
- **integrity_check / quick_check** — the engine's storage audits: full cross-checking versus the faster daily subset.
- **journal (estate)** — the FTS-indexed table of prose findings written at outcome time for future searchers.
- **ledger** — the estate's table of world-actions: idempotency key, action as composed, intent time, outcome with evidence, outcome time.
- **lost update** — the read-modify-write race where the last writer erases intervening updates; structurally absent under transactions.
- **midden** — the ad-hoc heap of state files this book replaces: unsearchable, un-transactional, corrupted by the first concurrent writer.
- **migration list** — the append-only sequence of DDL steps that builds any vintage of the estate to the current schema, applied idempotently at open.
- **no unintended truths** — the transaction property that every observable estate state was deliberately committed by some operator.
- **open ritual** — the single function every tenant opens the estate through: foreign keys on, busy timeout set, WAL on, migrations applied.
- **provenance block** — the columns every record owes the future: recorded_at (UTC ISO-8601, defaulted), recorded_by, source.
- **queue (estate)** — the ledger variant holding work that waits: atomic claim via UPDATE...RETURNING, completion as a second write, stale claims reclaimed.
- **read-only seat** — a connection opened `mode=ro`; reporting without the ability to write.
- **RETURNING** — the SQL clause handing back what a write changed, making claim-and-learn a single atomic statement.
- **rollback journal** — the classic atomic-commit sidecar holding original pages until commit; visible mid-transaction as `-journal`.
- **run registry** — the estate's table of sessions: operator, task, start, end, outcome; open rows are inherited unfinished business.
- **savepoint** — a named transaction-within-a-transaction; undo boundary for attempts that may not survive.
- **sidecar** — the `-journal`, `-wal`, or `-shm` file beside a database; part of the database, opened with the engine, never handled by hand.
- **single-writer queue** — SQLite's concurrency contract: one write transaction at a time, writers queued, readers (under WAL) unblocked.
- **snapshot isolation** — a WAL read transaction's stable view of the database as of its start, regardless of concurrent commits.
- **standing questions** — the named queries adopted alongside each pattern; the estate's interface and its cheapest schema review.
- **STRICT table** — the table option making declared types contracts the engine enforces at write time.
- **trigram tokenizer** — FTS5 tokenization by three-character windows, buying indexed substring search for identifiers, paths, and hashes.
- **trust ladder** — graded confidence in an inherited estate: opens → storage audits pass → application audits pass → spot re-verification against the world.
- **upsert** — INSERT that becomes UPDATE on key conflict; the idiom for current-state rows like cursors.
- **user_version** — the integer SQLite reserves in the file header for the application's schema version; the migration list's counterpart in the file.
- **VACUUM / VACUUM INTO** — rebuilding the database compactly in place, or writing a transactionally consistent compact copy to a new file (the estate's backup verb).
- **WAL (write-ahead log)** — the journal mode appending new pages to a log instead of rewriting in place; readers and writers stop blocking each other.

## References

1. Appropriate Uses For SQLite ("SQLite does not compete with client/server databases; SQLite competes with fopen()"). https://sqlite.org/whentouse.html
2. Most Widely Deployed and Used Database Engine. https://sqlite.org/mostdeployed.html
3. Well-Known Users of SQLite. https://www.sqlite.org/famous.html
4. SQLite As An Application File Format. https://sqlite.org/appfileformat.html
5. Atomic Commit In SQLite. https://sqlite.org/atomiccommit.html
6. Transaction documentation (BEGIN DEFERRED/IMMEDIATE/EXCLUSIVE). https://sqlite.org/lang_transaction.html
7. SAVEPOINT documentation. https://sqlite.org/lang_savepoint.html
8. Isolation In SQLite. https://sqlite.org/isolation.html
9. Datatypes In SQLite (type affinity). https://sqlite.org/datatype3.html
10. STRICT Tables. https://sqlite.org/stricttables.html
11. Quirks, Caveats, and Gotchas In SQLite (foreign keys off by default; flexible typing; boolean aliases). https://sqlite.org/quirks.html
12. ALTER TABLE documentation (deliberate minimalism; the sanctioned table rebuild). https://sqlite.org/lang_altertable.html
13. Date And Time Functions. https://sqlite.org/lang_datefunc.html
14. Generated Columns. https://sqlite.org/gencol.html
15. The RETURNING Clause. https://sqlite.org/lang_returning.html
16. Write-Ahead Logging. https://sqlite.org/wal.html
17. File Locking And Concurrency In SQLite Version 3. https://sqlite.org/lockingv3.html
18. Pragma statements (user_version, foreign_keys, busy_timeout, synchronous, integrity_check, quick_check, wal_checkpoint). https://sqlite.org/pragma.html
19. SQLite FTS5 Extension (query syntax, bm25, highlight/snippet, tokenizers, external content, optimize/rebuild). https://sqlite.org/fts5.html
20. The JSON Functions. https://sqlite.org/json1.html
21. VACUUM (and VACUUM INTO). https://sqlite.org/lang_vacuum.html
22. How To Corrupt An SQLite Database File (naive copies, deleted sidecars, fork with open connections, network filesystems). https://sqlite.org/howtocorrupt.html
23. Defense Against The Dark Arts: SQLite database files as untrusted input. https://sqlite.org/security.html
24. Uniform Resource Identifiers (mode=ro, immutable). https://sqlite.org/uri.html
25. How SQLite Is Tested. https://sqlite.org/testing.html
26. SQLite Is Self-Contained. https://sqlite.org/selfcontained.html
27. SQLite Is Serverless. https://sqlite.org/serverless.html
28. Frequently Asked Questions (INSERT speed and durable transaction cost). https://sqlite.org/faq.html
29. The Virtual Table Mechanism Of SQLite. https://sqlite.org/lang_createvtab.html
30. Python standard library: sqlite3 — DB-API interface (transaction handling, autocommit modes, iterdump, backup, thread affinity). https://docs.python.org/3/library/sqlite3.html
31. SQLite Database File Format (stability pledge through 2050). https://sqlite.org/fileformat2.html
32. *Linux for Language Models*, O'AILLY Systems & Craft (the companion volume; the register's disciplines this book extends to memory). https://oailly.com/read/rogerai-labs--linux-for-language-models/

## A note on measured outputs

Outputs printed in this book's listings are real transcripts from the authoring
machine (Gentoo Linux, kernel 6.18.31-gentoo-dist, Python 3.13.x with SQLite
3.51.3), captured 2026-08-28 under the publisher gate's environment. Quantities
that vary run to run (timings, temporary paths, timestamps, machine-load-
dependent figures) will differ on re-execution; statuses, refusals, and
behaviors are the reproducible claims.
