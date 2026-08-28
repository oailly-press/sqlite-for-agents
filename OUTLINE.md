# Durable State for Ephemeral Minds — proposal and evidence map

**Working title:** Durable State for Ephemeral Minds
**Subtitle:** SQLite as the memory of machine operators
**Shelf:** SYSTEMS & CRAFT (deltas: none — code must run, citations resolve)
**Tier:** Pocket (target ~26,000 measured words, 8 chapters)
**Proposed book-id:** rogerai-labs--sqlite-for-agents
**Status:** proposal + outline only; queued behind the publisher's one-in-pipeline
slot (Linux for Language Models, in review). Not yet written.
**Mascot request (draft):** leafcutter ant — an operator that cannot digest what it
harvests and instead builds a durable, structured store (the fungus garden) that
does the remembering for the colony; provisioning a memory outside the body is the
whole trick, and it is this book's trick too.

## The book-shaped hole

Every agent framework reinvents state badly. Operators whose sessions end — cron
jobs, CI steps, and language-model agents above all — need memory that outlives the
process: task ledgers, cursors, caches, run histories, knowledge scraps. Today that
memory is a midden of ad-hoc JSON files, dotfile scribbles, and pickle blobs:
unsearchable, un-transactional, corrupted by the first concurrent writer, silently
lost to a typo'd path. Meanwhile the most deployed database on earth sits in the
standard library of the language every one of these operators already carries.
SQLite's own documentation is excellent but tool-shaped — it teaches the engine,
not the *operator's discipline* for using it as machine memory: which state
deserves a table versus a file, how an ephemeral mind hands a database to its next
incarnation, how two uncoordinated operators share one file without eating each
other's writes, what verification-before-trust looks like when you did not write
the rows you are reading. Framework docs assume their own abstractions; database
books assume a server and a DBA. The operator's middle — one file, no server, no
memory of yesterday — is documented nowhere, and it is book-shaped because the
practices constrain each other exactly as the register's do: schema design decides
what handoff is possible; transactions decide what concurrency is safe; provenance
columns decide what future trust can be rebuilt.

## Reader

The developer building agents, unattended jobs, or self-hosted automations that
must remember things between runs — and, in second person where it earns it, the
operator itself deciding at 3 a.m. where a fact should live. Assumes basic SQL
reading knowledge and the shell literacy of a working developer; assumes no
database administration background.

## Boundaries (stated in chapter 1, held everywhere)

Claims are grounded in sqlite.org documentation and runnable listings (python3
stdlib `sqlite3` — every listing executes in the gate sandbox with no external
dependencies). The book does not cover client-server databases, does not argue
SQLite replaces them at scale, does not cover vector search or embedding stores
beyond honest pointers, and does not teach SQL from zero. Where SQLite's own docs
state a limit or a caveat, the book cites it rather than paraphrasing around it.

## Chapter architecture and evidence plan

1. **The Amnesiac's Estate** — the thesis: operators with no memory need state with
   guarantees; the file-midden failure modes (partial writes, lost updates,
   unsearchable history) demonstrated live; where SQLite sits (one file,
   in-process, transactional) and the state taxonomy: scratch (files), records
   (tables), artifacts (files + a table indexing them). Evidence: sqlite.org
   "appropriate uses", "how SQLite works"; runnable corruption-by-interruption
   demo against a naive JSON ledger vs. the same ledger in a transaction.
2. **One File, Whole Truths** — ACID for operators: what a transaction actually
   promises; atomic commit; `BEGIN IMMEDIATE` vs deferred; the write that either
   happened or did not — chapter-length development of the register's
   ask-and-verify against a store that can finally answer. Evidence: sqlite.org
   atomic-commit and transaction docs; runnable interrupted-write demonstrations.
3. **Schema Is the Handoff** — designing tables a stranger (your next incarnation)
   can read: naming, `STRICT` tables, CHECK constraints as executable
   documentation, the provenance columns (created_at UTC, created_by, source),
   schema_version and migrations applied idempotently. Evidence: sqlite.org STRICT
   and ALTER TABLE docs; a worked ledger schema built and migrated live.
4. **The Ledger Pattern and Friends** — the recurring shapes of operator memory:
   append-only ledgers, cursors/bookmarks, key-value config with history, run
   registries, content-addressed artifact indexes; each as a small worked schema
   with the queries that serve it. Evidence: runnable listings for every pattern;
   cross-references to the register book's ledger discipline.
5. **Two Operators, One File** — concurrency without a server: WAL mode and what
   it changes, busy_timeout, retry discipline, why long transactions starve
   writers, the single-writer truth and its consequences for agent fleets.
   Evidence: sqlite.org WAL and locking docs; runnable two-process contention
   demonstrations with real BUSY outcomes and their cures.
6. **Search Is Recall** — FTS5 for the operator's notes: building a searchable
   memory of transcripts, findings, and documents; ranking, prefix queries,
   highlight; what FTS is not (not embeddings, not semantics) and when a plain
   LIKE suffices. Evidence: sqlite.org FTS5 docs; a runnable searchable-journal
   build over real text.
7. **Trust, Verify, Repair** — reading a database you did not write: PRAGMA
   integrity_check as the opening move, quick_check economics, foreign_key_check;
   backup that is actually consistent (the backup API and VACUUM INTO, why `cp` on
   a live WAL database lies); corruption realities and the honest recovery
   ladder. Evidence: sqlite.org integrity/backup/corruption docs; runnable
   verify-and-backup listings.
8. **Where Memory Ends** — retention and forgetting as design (DELETE + VACUUM
   truths, the auto_vacuum tradeoff); when SQLite is the wrong answer (write-heavy
   multi-host, huge blobs, true multi-writer) with the honest handoff to bigger
   tools; and the closing frame: the estate handed to the next incarnation — one
   file, verified, searchable, explaining itself. Evidence: sqlite.org limits and
   when-to-use docs; the book's own schemas as the worked estate.

## Length and listing plan

8 chapters × ~3,300 measured words ≈ 26k body words. Listing policy
`executable_plus_marked_fragments`; nearly everything is runnable python3 stdlib
(`sqlite3`) in scratch directories — the gate sandbox limits (no network, 512 MB,
15 s CPU, 40-listing execution budget) shape listings exactly as they did for the
register book; `no-run` marks the overflow demos. All outputs real transcripts.

## Contamination note

Adjacent to *Linux for Language Models* (same author, same register) but disjoint
in substance: that book teaches reading and changing a machine; this one teaches
remembering. The ledger discipline is cross-referenced, not restated — chapter 4
builds schemas for it rather than re-arguing it — and the catalog-overlap gate is
the enforcement.
