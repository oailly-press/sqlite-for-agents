# Chapter 3 — Schema Is the Handoff

*Draft status: author draft, gate-checked; human verification pending. Outputs are
real transcripts from the authoring machine.*

## The stranger at the door

Whoever next opens the estate — tomorrow's session, a different tool, a human
with a database browser, some future model that does not exist yet — arrives
knowing nothing except what the file itself can teach. There will be no
walkthrough, no chat with the author, no institutional memory; the register's
operators do not get onboarding. Every hope of a good handoff therefore rests on
one artifact: the schema. A schema is usually described as the structure data is
stored in, which is true and misses the point that matters here. For the
amnesiac's estate, the schema is *the documentation that executes* — the one
description of the data that cannot drift from the data, because the engine
enforces it on every write. Prose documentation describes what writers intended;
schemas constrain what writers could do. A stranger can trust the second kind
without trusting anyone.

This chapter is therefore written as a craft of hospitality: every choice —
types, constraints, provenance columns, versioning, naming — is judged by what
it tells or guarantees to a reader who was not there. The test to hold
throughout is concrete: *could a competent stranger, given only the file,
reconstruct what each table means, how much to trust each row, and how the
whole thing has changed over its life?* Each section below closes one gap
between today's estates and a yes.

## Types that mean it

The first surprise SQLite deals a newcomer is that, by historical default, its
column types are suggestions. The engine's "flexible typing" — documented
candidly in its datatype and quirks pages — declares a column's type an
*affinity*, a preference the engine will try to honor and cheerfully override
when a value disagrees:

```python
import sqlite3
db = sqlite3.connect("estate.db")
db.execute("CREATE TABLE loose (host TEXT, cpus INTEGER)")
db.execute("INSERT INTO loose VALUES (42, 'sixty-four')")   # both wrong, both accepted
db.execute("CREATE TABLE strict (host TEXT, cpus INTEGER) STRICT")
try:
    db.execute("INSERT INTO strict VALUES ('RogGentoo', 'sixty-four')")
except sqlite3.IntegrityError as e:
    print("strict refused:", e)
print("loose table holds:", db.execute("SELECT host, cpus FROM loose").fetchone())
```

```output
strict refused: cannot store TEXT value in INTEGER column strict.cpus
loose table holds: ('42', 'sixty-four')
```

The loose table accepted a numeric host name and a spelled-out CPU count without
a murmur, and will hand them to every future reader who asked, by reading the
schema, for text and an integer. In an application with a single disciplined
writer this laxity is survivable; in an estate written by generations of
operators — some of them language models assembling INSERT statements from
prose — it is a slow poison, because every reader must now defend against every
past writer's accidents. The cure costs seven characters: the `STRICT` table
option, added to the language in 2021 precisely for schemas that mean what they
say, makes the declared type a contract and a violating write an error at the
write site — where the operator that caused it is still present to read the
refusal, instead of at the read site months later, where nobody is. This book's
rule needs no nuance: **every estate table is STRICT.** The loose demonstration
above is the last non-STRICT table in it.

Two consequences of STRICT deserve a sentence each. Declared types must come
from the engine's real repertoire (INT, INTEGER, REAL, TEXT, BLOB, ANY) — the
compatibility aliases that flexible typing tolerated are refused, which is
itself documentation-by-enforcement. And where a column legitimately holds
mixed types — rare, but real — `ANY` declares that honestly, telling the
stranger "expect anything here" instead of lying with a specific type the
engine will not police.

## Constraints: the rules that outlive their authors

Types police form; constraints police meaning, and they are where the estate's
discipline stops being conduct and becomes structure. Chapter 2 established the
invariant "no change recorded without its proof" by transaction shape — a
discipline each writer must remember. A CHECK constraint moves it into the
schema, where no writer can forget it:

```python
import sqlite3
db = sqlite3.connect("estate.db")
db.execute("""
CREATE TABLE tasks (
  id INTEGER PRIMARY KEY,
  title TEXT NOT NULL CHECK (length(title) > 0),
  status TEXT NOT NULL DEFAULT 'open'
      CHECK (status IN ('open', 'blocked', 'done', 'abandoned')),
  proof TEXT,
  CHECK (NOT (status = 'done' AND proof IS NULL))
) STRICT
""")
db.execute("INSERT INTO tasks (title) VALUES ('rotate backup credentials')")
try:
    db.execute("UPDATE tasks SET status = 'done' WHERE id = 1")
except sqlite3.IntegrityError as e:
    print("refused:", e)
db.execute("UPDATE tasks SET status = 'done', proof = 'restore drill passed 2026-08-28' WHERE id = 1")
print(db.execute("SELECT title, status, proof FROM tasks").fetchone())
```

```output
refused: CHECK constraint failed: NOT (status = 'done' AND proof IS NULL)
('rotate backup credentials', 'done', 'restore drill passed 2026-08-28')
```

Read the refused UPDATE as the stranger will read the schema: this estate does
not contain finished tasks without evidence, and that is not a hope, it is a
property. Each constraint in the table is doing double duty. `NOT NULL` and the
non-empty check on `title` refuse the classic degradation of ledgers into rows
of placeholders. The `status IN (...)` enumeration is the poor operator's enum,
and more: it is the complete, machine-readable list of states this workflow
admits — a stranger learns the lifecycle without a wiki. The table-level CHECK
encodes the proof invariant. And every one of these rules will still be
enforced, verbatim, on writers not yet written, running models not yet trained,
years after the author-session that chose them ended. Constraints are the only
documentation with that property, which is why the estate spends them
generously — while respecting their limit: a CHECK sees only its own row.
Cross-row truths (uniqueness, references) have their own instruments, one of
which comes with a trap.

## The referential switch everyone forgets

Foreign keys — the declaration that a `findings.run_id` must name a real row in
`runs` — are the cross-table constraint estates lean on constantly, and SQLite
ships them **off**. The syntax parses, the schema records the intention, and by
historical default nothing is enforced:

```python
import sqlite3
db = sqlite3.connect("estate.db")
db.executescript("""
CREATE TABLE runs (id INTEGER PRIMARY KEY, started_at TEXT);
CREATE TABLE findings (id INTEGER PRIMARY KEY, run_id INTEGER REFERENCES runs(id), note TEXT);
""")
db.execute("INSERT INTO findings (run_id, note) VALUES (999, 'orphan')")   # run 999 never existed
print("orphans accepted by default:", db.execute("SELECT count(*) FROM findings").fetchone()[0])
db.execute("PRAGMA foreign_keys = ON")
try:
    db.execute("INSERT INTO findings (run_id, note) VALUES (998, 'another orphan')")
except sqlite3.IntegrityError as e:
    print("with the pragma:", e)
```

```output
orphans accepted by default: 1
with the pragma: FOREIGN KEY constraint failed
```

The quirks documentation owns this frankly as a compatibility fossil: turning
enforcement on by default would break decades-old applications, so every new
connection starts with it off, and `PRAGMA foreign_keys = ON` must be issued
*per connection* — not once per database, which is the misunderstanding that
produces estates that were protected on Tuesdays. The operational consequence
is a ritual this book now installs and never abandons: estates are opened by
one function, and that function issues the pragmas. Scattered
`sqlite3.connect()` calls throughout a codebase are how one forgotten switch
quietly waives the constraints everywhere; a single `open_estate()` is how a
decision is made once. The migration listing below *is* that function, because
the open ritual and versioning belong together.

## Born versioned

Schemas change — a column proves missing, a pattern from chapter 4 gets
adopted, an index earns its keep — and the estate must survive its own
evolution across operators who cannot be gathered for a migration party. The
mechanism is small enough to read whole. The database carries its own version
number (SQLite reserves `PRAGMA user_version`, an integer stored in the file
header, for exactly this); the code carries an append-only list of migrations;
opening the estate applies whatever the file has not yet seen:

```python
import sqlite3
MIGRATIONS = [
    (1, "CREATE TABLE facts (id INTEGER PRIMARY KEY, fact TEXT NOT NULL) STRICT"),
    (2, "ALTER TABLE facts ADD COLUMN source TEXT NOT NULL DEFAULT 'unrecorded'"),
    (3, "CREATE INDEX facts_source ON facts(source)"),
]
def open_estate(path):
    db = sqlite3.connect(path)
    db.execute("PRAGMA foreign_keys = ON")
    applied = db.execute("PRAGMA user_version").fetchone()[0]
    for version, ddl in MIGRATIONS:
        if version > applied:
            with db:
                db.execute(ddl)
                db.execute(f"PRAGMA user_version = {version}")
            print(f"applied migration {version}")
    return db
db = open_estate("estate.db"); db.close()
print("reopening…")
db = open_estate("estate.db")
print("schema version now:", db.execute("PRAGMA user_version").fetchone()[0])
```

```output
applied migration 1
applied migration 2
applied migration 3
reopening…
schema version now: 3
```

The second opening applied nothing — the file said 3, the list said 3, done —
and that idempotence is the whole trick: any operator, any generation, opening
any vintage of the estate, arrives at the same schema, and a fresh file builds
itself from nothing by the same path. Three rules keep the mechanism honest.
Migrations are *append-only*: a shipped migration is history, and fixing a bad
one means appending a corrective, never editing the past — the same
no-history-rewrites covenant this book's own publisher enforces on manuscripts,
for the same reason (someone already built on the past). Each migration runs in
its own transaction with the version bump inside it, so a crash mid-migration
leaves the file honestly at the old version, ready to retry, never half-moved.
And migrations are pure DDL applied at open, before any data work — chapter 2's
separation of schema phase from data phase, now with an address. (SQLite's
`ALTER TABLE` is deliberately minimal — add, rename, drop; no type changes —
and the documentation's sanctioned workaround for bigger reshapes, build-new,
copy, swap inside a transaction, is chapter 5's atomic-replace instinct applied
to tables. Design so you rarely need it; the migration list makes even that
reshaping a recorded, replayable event.)

## The estate's value conventions

Between types and constraints sits a layer of conventions the schema cannot
fully enforce but the stranger must be able to assume, and stating them once —
in the estate's own documentation table or its schema comments — spares every
future writer a private decision and every future reader a private guess.
Dates and times: SQLite has no datetime type; the engine stores what you give
it and supplies functions for several representations. The estate's convention
is the one both prior chapters already used — TEXT, UTC, ISO-8601, seconds
precision, trailing `Z` — because it sorts as text, compares as text, reads
without conversion, and joins across estates without timezone archaeology; a
CHECK (`length(recorded_at) = 20`) pins the shape cheaply where drift would
hurt. Booleans: INTEGER 0 and 1, with a CHECK constraining to those two, since
SQLite's own quirks page notes the keywords are mere aliases. Numbers that are
really identifiers — ports, PIDs, version strings — stay TEXT, because
arithmetic on them is always a bug and leading zeros have died for less.

Two lower-level conventions complete the set, each a sentence of policy
against an hour of future confusion. Text is UTF-8, always — the engine
stores what it is handed, Python hands it UTF-8, and the estate declares
the encoding in its info table so no future tool guesses; the one encoding
accident worth naming is bytes-that-are-not-text, which belong in BLOB
columns honestly rather than in TEXT columns hopefully. And REAL is for
measurements, never for money or counts-of-things: floating point is the
right shape for a CPU temperature and the classic wrong shape for anything
that must sum exactly, where the convention is integers in the smallest
unit (cents, bytes, milliseconds) with the unit in the column name or its
comment — `duration_ms INTEGER`, self-documenting at every read site. Both
rules exist in every database tradition; they are restated here because
estates are written by operators assembling schemas at 3 a.m. from prose
intentions, which is exactly when a stated convention outperforms a
remembered one.

The schema can also *compute* for its writers. Beyond the DEFAULT
expressions the provenance block already leans on, generated columns —
`GENERATED ALWAYS AS (expression)` — derive a value from the row's other
columns, kept current by the engine itself: a `year` column generated from
`recorded_at` for cheap grouping, a normalized lowercase key generated
from a mixed-case source, a size bucket derived from a byte count. The
estate's use for them is the same as for defaults: moving invariants out
of writer discipline and into structure, so that a value which *must*
track another value cannot be updated into disagreement by a forgetful
session. The restraint that keeps them honest: generated columns derive,
they never import — an expression reaching beyond its own row (dates
"now", random values, subqueries) is a trap the syntax mostly forbids and
the design rule finishes: derivation is structure, acquisition is a
writer's act, and the provenance block exists to record the second.

NULL deserves its own paragraph, because it is the one value that behaves
differently from every intuition text formats build, and estates use it
deliberately (chapter 4's "fate unknown"). NULL is not zero, not empty string,
and — the sharp edge — not equal *or unequal* to anything, which makes it
invisible to comparisons that feel exhaustive:

```python no-run
import sqlite3
db = sqlite3.connect(":memory:")
db.execute("CREATE TABLE t (host TEXT, env TEXT)")
db.executemany("INSERT INTO t VALUES (?,?)",
               [("web-1","prod"), ("web-2","staging"), ("web-3", None)])
print("hosts where env != 'staging':",
      db.execute("SELECT host FROM t WHERE env != 'staging'").fetchall())
print("with NULL handled:           ",
      db.execute("SELECT host FROM t WHERE env IS DISTINCT FROM 'staging'").fetchall())
```

```output
hosts where env != 'staging': [('web-1',)]
with NULL handled:            [('web-1',), ('web-3',)]
```

The host with no recorded environment vanished from a query that asked, in
plain English, for everything that is not staging — because `NULL !=
'staging'` is neither true nor false, and WHERE keeps only true. Three-valued
logic is not a flaw to route around but the correct semantics for "unknown";
the operator's obligations are two. Query with the NULL-aware forms when
unknowns must be included (`IS NULL`, `IS DISTINCT FROM`, `coalesce`). And
constrain NULL to mean exactly one thing per column — chapter 4's ledger
allows it in `outcome` *as* the fate-unknown marker and forbids it everywhere
meaning would blur, which is the general rule: a nullable column is a column
whose NULL has a documented reading, and any other nullable column is a guess
someone deferred.

## Migrations that move data

The migration list shown above contains only structure, and most migrations
are structural — but the honest catalog includes the other kind, because
sooner or later a migration must *reshape rows*: backfill a new column from
an old one's contents, split a field, normalize a unit. The mechanism needs
no extension — a migration entry is SQL, and UPDATE is SQL — but the
discipline tightens in three ways. A data migration states its scope in a
WHERE clause exactly as the previous volume's edits anchored their `sed`
patterns, and rehearses as a SELECT count of that scope before shipping (the
dry-run doctrine, unchanged). It remains inside the migration's transaction,
so a failure mid-backfill leaves the version number honestly unmoved. And it
never destroys its input in the same migration that derives from it — the
old column survives until a later migration retires it, one version after
the new column has been read in anger. For reshapes beyond ALTER TABLE's
deliberate minimalism — type changes, constraint additions to existing
columns — the engine's documentation prescribes the rebuild: create the new
table, copy with a transforming SELECT, drop the old, rename — all inside
one transaction, the atomic-swap instinct applied to tables:

```python fragment
# The sanctioned reshape, per the ALTER TABLE documentation — one transaction:
# CREATE TABLE facts_new (...corrected shape...) STRICT;
# INSERT INTO facts_new SELECT ...transformed... FROM facts;
# DROP TABLE facts;
# ALTER TABLE facts_new RENAME TO facts;
```

A reshape is the most invasive act an estate performs on itself, which is why
it lives in the migration list — versioned, transactional, replayed
identically by every opener — rather than in any session's ad-hoc hands. The
stranger's guarantee survives even this: whatever generation of the schema
they open, the road from there to current is recorded, ordered, and runs
itself.

## Indexes: the reader's courtesy

The stranger inherits not only meanings but *costs*, and one more schema
instrument decides whether the estate's questions stay cheap as it grows. A
table is, physically, rows in row order; a query that filters on anything else
must, absent help, examine every row. The help is an index, and the engine will
tell you — before any harm is done — whether a given question has one, through a
statement every estate author should reflexively use:

```python
import sqlite3
db = sqlite3.connect("estate.db")
db.execute("CREATE TABLE facts (id INTEGER PRIMARY KEY, subject TEXT NOT NULL, fact TEXT NOT NULL) STRICT")
with db:
    db.executemany("INSERT INTO facts (subject, fact) VALUES (?, ?)",
                   [(f"host-{n % 50}", f"observation {n}") for n in range(5000)])
q = "SELECT count(*) FROM facts WHERE subject = 'host-7'"
print("before:", db.execute("EXPLAIN QUERY PLAN " + q).fetchone()[3])
db.execute("CREATE INDEX facts_subject ON facts(subject)")
print("after: ", db.execute("EXPLAIN QUERY PLAN " + q).fetchone()[3])
print("answer:", db.execute(q).fetchone()[0])
```

```output
before: SCAN facts
after:  SEARCH facts USING COVERING INDEX facts_subject (subject=?)
answer: 100
```

`SCAN` is the plan that reads everything; `SEARCH ... USING INDEX` is the plan
that walks straight to the hundred matching rows among five thousand. At five
thousand rows the difference is microseconds and nobody cares; the reason the
habit matters is that estates *age*, and the queries written today run against
the table sizes of years hence, by operators who will experience a missing
index not as a design gap but as "the estate got slow" — a diagnosis away from
the cause. The register's two-question test settles what to index: whatever
columns the estate's *standing questions* filter or join on — `subject` here,
`recorded_at` for every retention and staleness query, foreign-key columns on
the many side. And the cost side stays honest: each index is paid for on every
write, which at operator scales is negligible and at bulk-import scales is
exactly why the migration list creates indexes *after* chapter 4's patterns
settle what the standing questions are. `EXPLAIN QUERY PLAN` is the audit that
keeps both sides truthful — the estate's equivalent of the register's dry run,
asking the engine what it *would* do while everything is still cheap.

## Shapes that mislead strangers

Hospitality also means declining certain shapes that schemas admit but
strangers regret. Three recur in operator estates often enough to name. The
JSON blob column — a TEXT field holding a serialized object — reintroduces the
midden *inside* the database: unqueryable without unpacking, unconstrainable by
CHECK in any depth, invisible to STRICT. SQLite's JSON functions make blobs
tolerable at the edges (a genuinely irregular payload, kept whole for
fidelity, with the *queried* fields lifted out into real columns beside it),
and the discipline is that lift: anything a standing question touches gets a
column; the blob is an artifact, not a record. The attribute-soup table —
`(entity, attribute, value)` triples, endlessly flexible — trades every
guarantee this chapter built for schema-free convenience: no types, no CHECKs,
no meaningful constraints, and every real query a self-join puzzle. It is the
shape estates reach for when their authors have not yet decided what they are
recording; the decision, not the soup, is the work. And the wide-null table —
one row type wearing forty mostly-NULL columns because several distinct kinds
of record were crowded into one table — fails the stranger at the first
question ("which columns apply to which rows?") that the schema, its one job,
can no longer answer. Each anti-shape has the same cure: tables that hold one
kind of thing, named for it, with the columns that kind actually owes — which
is chapter 4's whole agenda.

## The columns every record owes the future

With types strict, constraints meaningful, references enforced, and birth
versioned, what remains is the estate's signature habit, promised since chapter
1: no fact without its papers. Concretely, record tables carry a provenance
block — `recorded_at`, defaulted by the schema to UTC ISO-8601 from the
engine's own clock so no writer can forget or localize it; `recorded_by`,
required, naming the operator (session, script, model — whatever identity the
successor can act on); and `source`, required, naming where the fact came from
— a file path, a URL, a command, a transcript reference. The choice of TEXT
ISO-8601 over numeric epochs is deliberate hospitality: it sorts correctly as
text, reads correctly to humans and models without conversion, and survives
tool changes — the register book's determinism rule, applied to storage. The
stranger's payoff compounds: any row can be weighed (how old? whose claim? from
what evidence?), re-verified (follow `source`), or aged out (chapter 8's
retention queries key on `recorded_at`). And the discipline prices honesty
correctly on the write side too — an operator that cannot fill `source` is
holding a rumor, and the schema just asked it to notice that before the rumor
entered the record.

One last hospitality note, almost free and almost never used: SQL comments
survive. SQLite stores the literal text of every CREATE statement and returns
it verbatim to any tool that asks, so a comment written in the schema —
explaining a constraint's reason, a column's unit, a status's meaning — is
carried inside the database file itself, readable forever. The proof, on a
table chapter 4 will need anyway:

```python
import sqlite3
db = sqlite3.connect("estate.db")
db.execute("""
CREATE TABLE cursors (
  stream TEXT PRIMARY KEY,          -- what is being read (journal unit, feed URL, log path)
  position TEXT NOT NULL,           -- opaque resume token; meaning belongs to the stream
  advanced_at TEXT NOT NULL         -- UTC; staleness is the reader's first question
) STRICT
""")
print(db.execute("SELECT sql FROM sqlite_schema WHERE name = 'cursors'").fetchone()[0])
```

```output
CREATE TABLE cursors (
  stream TEXT PRIMARY KEY,          -- what is being read (journal unit, feed URL, log path)
  position TEXT NOT NULL,           -- opaque resume token; meaning belongs to the stream
  advanced_at TEXT NOT NULL         -- UTC; staleness is the reader's first question
) STRICT
```

The comments came back byte-for-byte, from the file, years-proof: the
`sqlite_schema` table every database carries is the estate describing itself,
and any stranger's first query. The estate's tables should be written like the drop-in files of the
register book: opening with two lines that say why they exist and who put them
there — except here, the note and the structure it explains travel in the same
artifact and cannot be separated. A schema written this way is not described by
its documentation. It *is* its documentation, enforced where it can be,
explained where it cannot.

One table completes the self-description, and every estate should carry it
from birth: the info table — plain key-value rows naming what no column
can. What this estate is for, in a sentence. Which operator lineage owns
it. Where its conventions are written (this book's, or the successor
document that supersedes it). Where its backups land. Who the supervising
human is. Five to ten rows, written once, maintained on change — the
estate's title page, and the answer to the one question the briefing of
chapter 8 cannot compute: *what am I looking at?* The previous volume put
this note in a drop-in file's opening comments; the estate puts it where
nothing can separate it from the data it explains.

## The handoff review, in nine questions

The chapter compresses to a checklist an author can run against any estate
schema — its own, or an inherited one being judged. Each question maps to a
section above.

1. Is every table STRICT, with types from the real repertoire?
2. Does every enumerable column enumerate (CHECK ... IN), and every
   invariant that spans columns have its table-level CHECK?
3. Is `PRAGMA foreign_keys = ON` issued by the one shared open ritual —
   and is there exactly one open ritual?
4. Does the file carry its version, and does opening apply an append-only
   migration list idempotently, DDL alone, one transaction per step?
5. Do all record tables carry the provenance block — recorded_at (UTC ISO,
   defaulted), recorded_by, source — with NULL meanings documented?
6. Are the value conventions (dates, booleans, identifiers) stated once
   and pinned by CHECK where drift would hurt?
7. Does every standing question have its index, and has EXPLAIN QUERY
   PLAN confirmed it?
8. Are the anti-shapes absent — queried fields inside JSON blobs,
   attribute soup, wide-null crowding?
9. Could the stranger answer "what is this?" from the file alone — schema
   comments present, info table filled?

Nine yeses is a schema that will survive its authors, which is the only
kind worth writing. What the stranger does next — the shapes of the tables
an operator actually keeps — is chapter 4.
