# Chapter 2 — One File, Whole Truths

*Draft status: author draft, gate-checked; human verification pending. Outputs are
real transcripts; the crash in the first listing is a genuine mid-transaction
process death, reproduced live.*

## The promise

Everything the estate offers rests on a single guarantee, so this chapter earns it
properly before the book builds on it. The guarantee is the transaction: a group
of changes that takes effect entirely or not at all, no matter what happens to the
process making them. Chapter 1 showed the file-midden's partial write — half a
ledger, unreadable, history gone. Here is the same death, mid-write, against the
estate's engine. The listing forks a child operator that opens a transaction,
inserts two ledger entries, and is killed before commit — `os._exit(1)`, no
cleanup handlers, no goodbye, as close to a real timeout-kill as a demonstration
can honestly get:

```python
import sqlite3, subprocess, sys
db = sqlite3.connect("estate.db")
db.execute("CREATE TABLE ledger (id INTEGER PRIMARY KEY, entry TEXT NOT NULL)")
db.commit(); db.close()
child = '''
import sqlite3, os
db = sqlite3.connect("estate.db")
db.execute("BEGIN IMMEDIATE")
db.execute("INSERT INTO ledger (entry) VALUES ('step 1 of 3 done')")
db.execute("INSERT INTO ledger (entry) VALUES ('step 2 of 3 done')")
os._exit(1)   # killed mid-transaction, no commit, no cleanup
'''
r = subprocess.run([sys.executable, "-c", child])
print("child exit:", r.returncode)
db = sqlite3.connect("estate.db")
print("rows visible to the successor:", db.execute("SELECT count(*) FROM ledger").fetchone()[0])
```

```output
child exit: 1
rows visible to the successor: 0
```

Zero rows — not one row, not a corrupted row and a half. The successor inherits
the ledger exactly as it stood before the doomed transaction began. Compare this
carefully with what it replaces. The midden's failure left *no* history; the
naive hope ("surely it wrote the first insert") would have left *wrong* history —
a ledger asserting step one completed with no record that a step two was ever in
flight. The transaction's all-or-nothing is better than both, and better in the
specific currency the register's operators trade in: the estate never contains a
state that no operator ever intended to be true. Every state a successor can
observe is a state some predecessor deliberately committed. That property — call
it *no unintended truths* — is the foundation everything else in this book stands
on, and it was purchased in the listing above for the price of one `BEGIN`.

## The visible mechanics

The guarantee is not magic, and seeing its machinery once makes its edge cases
legible forever. SQLite's classic implementation of atomic commit is the rollback
journal: before touching the database file, the engine writes the *original*
content of every page it is about to change into a sidecar file, so that a crash
at any instant leaves either an untouched database or enough information to
restore one. The sidecar is an ordinary file, and you can catch it existing:

```python
import sqlite3, os
db = sqlite3.connect("estate.db")
db.execute("CREATE TABLE t (x)")
db.commit()
db.execute("BEGIN IMMEDIATE")
db.execute("INSERT INTO t VALUES (1)")
print("mid-transaction files:", sorted(f for f in os.listdir(".") if f.startswith("estate")))
db.commit()
print("after commit:       ", sorted(f for f in os.listdir(".") if f.startswith("estate")))
```

```output
mid-transaction files: ['estate.db', 'estate.db-journal']
after commit:        ['estate.db']
```

There is the promise, incarnate as `estate.db-journal`. If the process dies with
that journal present, the *next* connection to open the database — tomorrow's
operator, a different program, the sqlite3 shell — finds it, replays the original
pages back, and only then proceeds: recovery is automatic, unavoidable, and
requires nothing from the successor but opening the file. The engine's
atomic-commit documentation walks the full choreography, including the fsync
barriers that make it hold across power loss, and it repays one careful read.
Chapter 5 introduces the journal's modern sibling — write-ahead logging, which
inverts the arrangement to buy concurrency — but the contract seen from outside
is identical, and so is the operator's one obligation, which this glimpse makes
concrete: **the sidecar files are part of the database.** A `-journal` (or, under
WAL, a `-wal`) file sitting beside the estate is not litter to clean up; deleting
it, or copying the main file without it, is how "atomic" becomes "corrupted" —
the precise mistake chapter 7 teaches backup to avoid.

## Transactions are units of meaning

Knowing that transactions group changes, the design question is *which* changes
belong grouped, and the answer gives operators a tool the file-midden never
offered: invariants. A transaction boundary should enclose exactly the set of
statements that must be true *together* — that make no sense, or make a lie,
if only some of them land. The register's previous book taught that a change
without a printed verification is a rumor; the estate can now enforce a stronger
form structurally. Suppose the discipline is that no change is recorded without
its proof. Put both in one transaction, and a failure anywhere before the proof
exists erases the claim as well:

```python
import sqlite3
db = sqlite3.connect("estate.db")
db.execute("CREATE TABLE changes (id INTEGER PRIMARY KEY, action TEXT, proof TEXT)")
db.commit()
try:
    with db:
        db.execute("INSERT INTO changes (action, proof) VALUES ('edited sshd_config', NULL)")
        raise RuntimeError("verification probe failed")   # proof never obtained
except RuntimeError as e:
    print("caught:", e)
print("half-recorded changes:", db.execute("SELECT count(*) FROM changes").fetchone()[0])
with db:
    db.execute("INSERT INTO changes (action, proof) VALUES ('edited sshd_config', 'sshd -t exit 0')")
print("fully-recorded changes:", db.execute("SELECT count(*) FROM changes").fetchone()[0])
```

```output
caught: verification probe failed
half-recorded changes: 0
fully-recorded changes: 1
```

The failed attempt vanished — action and all — because the exception unwound the
transaction before commit; the successful attempt landed whole. Notice what this
does to the estate's epistemics: a `changes` row *cannot exist* in the
action-recorded-but-unproven state, not because writers are disciplined but
because the schema of commitment forbids it. (Whether an *unrecorded but
performed* action can exist is the operator's conduct problem, and chapter 4's
ledger pattern narrows it; no database can close it alone.) The design habit
that follows is worth stating as a rule: **choose transaction boundaries by
asking what a successor must never half-see.** A multi-row config change, a
cursor advance paired with the processing it acknowledges, an artifact row
paired with its hash — each is one transaction because each is one truth.
The anti-pattern is equally shaped: transactions drawn around *convenience*
(one per function, one per loop iteration, one giant one around an hour's work)
group statements by accident, and chapter 5 will add the concurrency reason
why the giant sort is actively harmful.

## The Python seam

Between the operator and the engine sits the `sqlite3` module, and its seams are
where estates actually leak, so this section is blunt about them. The first seam
is the one every newcomer meets the hard way — uncommitted work does not
survive:

```python
import sqlite3
db = sqlite3.connect("estate.db")
db.execute("CREATE TABLE facts (id INTEGER PRIMARY KEY, fact TEXT)")
db.execute("INSERT INTO facts (fact) VALUES ('this machine has 64 cpus')")
db.close()                      # ended without commit
db = sqlite3.connect("estate.db")
print("facts the successor inherits:", db.execute("SELECT count(*) FROM facts").fetchone()[0])
```

```output
facts the successor inherits: 0
```

The fact was inserted, the statement succeeded, the process exited cleanly — and
the successor inherits nothing, because under the module's default (legacy)
transaction handling, that INSERT silently *opened* a transaction that nothing
ever committed. (The table itself survived only because DDL under the legacy
mode autocommits differently — an inconsistency that is itself an argument for
what follows.) Closing, in SQLite semantics, resolves an open transaction by
rolling it back: correct by the engine's lights, catastrophic by the operator's.
Three disciplines close the seam. Prefer the context manager for every write —
`with db:` commits on success and rolls back on exception, turning the previous
section's invariant pattern into the path of least resistance. Know its one
surprise: `with db:` does **not** close the connection, only ends the
transaction; the estate connection can and should live across many such blocks.
And on modern Python, consider declaring intentions explicitly — the module now
offers an `autocommit` attribute whose explicit modes replace the legacy
implicit-transaction behavior; the standard library documentation marks the
legacy mode as the compatibility default, not the recommendation. Whichever mode
an estate's tooling picks, it should pick *on purpose*, in one place, and write
it down — chapter 3 gives estate metadata a table, and the connection discipline
belongs in it.

The second seam is quieter: `executescript` issues an implicit COMMIT before
running, and DDL's interaction with open transactions has version-dependent
subtleties — reason enough for a simpler rule that sidesteps the whole area:
schema changes happen at estate-open time, alone, before any data transaction
begins (chapter 3's migration pattern does exactly this), and data transactions
never mix DDL in. Operators that keep the two phases separate never meet the
subtleties at all.

Error reading completes the seam-sealing, because the module speaks in
exception classes the way commands speak in exit codes, and the register's
number-first discipline translates directly. `IntegrityError` is the schema
talking: a constraint refused the write, the estate is *working* — chapter
3 will make these refusals load-bearing and chapter 4's idempotency
pattern will treat one as an answer rather than a failure. `OperationalError`
is the circumstances talking: locked, busy, missing table, read-only —
conditions to diagnose, several of which chapter 5 converts to routine.
`ProgrammingError` is the operator talking to itself: malformed SQL,
wrong parameter counts — a composition bug, never retried, always fixed.
And `DatabaseError`'s corruption face ("malformed") is chapter 7's
department, met there with its own protocol. Catching broadly
(`except Exception`) around estate writes collapses these four distinct
sentences into one shrug — the transcript-mode operator's oldest sin,
parsing prose instead of reading the channel built for machines — and the
estate discipline is the same as the register's: catch the narrow class
the logic actually answers, let the rest surface loudly, and record what
surfaced.

## Saying IMMEDIATE, and meaning it

The listings above wrote `BEGIN IMMEDIATE` where plain `BEGIN` would seem to do,
and the difference deserves its own section because it is the first place
concurrency intrudes on even a single operator's thinking. A plain (deferred)
BEGIN acquires no lock at all: the transaction is notional until the first
actual read or write, and — the sharp edge — a transaction that *reads first and
writes later* acquires a read lock first and must upgrade to a write lock at the
first write. If, between the read and the write, some other connection has begun
its own write, the upgrade can find itself in a deadlock the engine resolves by
refusing: `database is locked`, delivered not at BEGIN, where the operator was
prepared to wait, but midway through the transaction's logic, where it was not.
`BEGIN IMMEDIATE` takes the write intention out loud at the start: it acquires
the write reservation up front, converting a mid-flight refusal into an at-entry
wait — and an at-entry wait is exactly the shape the register's operators know
how to handle, with a timeout and a bounded retry. The rule of thumb this book
uses everywhere: **a transaction that will write says IMMEDIATE at BEGIN.**
Read-only transactions stay deferred and cost nothing. Chapter 5 measures the
contention behavior for real, two operators against one file; the habit is
installed now because retrofitting it later means auditing every write site.

## Reading is transactional too

Transactions entered this chapter as the writer's tool, and their quieter half
belongs to readers. A report composed from several SELECTs — count the open
intents, then sum the week's failures, then list the stale cursors — is
implicitly claiming that its lines describe *one moment*. Run bare, each
SELECT is its own instant, and a writer committing between them hands the
report a world that never existed: the intent counted in line one resolved
before line three listed it, and the totals disagree with the details. Nobody
debugs this, because nothing errored; the report is simply, occasionally,
incoherent — and an unattended operator publishing it into a handoff message
is signing evidence with a torn timestamp. The cure is the same instrument
pointed the other way: open a transaction, run every read the report needs,
then end it. Inside, the reads share one consistent view (under WAL, chapter
5 shows this costs concurrent writers nothing at all), and the report's
implicit claim becomes true. The habit is cheap to install — the estate's
reporting queries live behind one function, the function wraps itself in a
read transaction — and it retires a failure class whose signature
(aggregates that almost agree) otherwise costs an afternoon the first time
it is met. The register's rule about evidence blocks said every figure is
measured fresh at the end; the estate's version adds: and all of them
through one snapshot, so "the end" is a moment rather than a smear.

## The price of a promise, and buying in bulk

Every COMMIT pays for its guarantee in the coin of chapter 1's register: real
work at the storage layer — journal bookkeeping and, at full durability, sync
barriers that wait for the disk. The cost is invisible at human scales and
decisive at loop scales, which makes it exactly the kind of economics an
unattended operator must know by feel rather than discover in production:

```python
import sqlite3, time
db = sqlite3.connect("estate.db")
db.execute("CREATE TABLE readings (id INTEGER PRIMARY KEY, v INTEGER)")
db.commit()
t0 = time.monotonic()
for i in range(2000):
    with db:                                   # one transaction per row
        db.execute("INSERT INTO readings (v) VALUES (?)", (i,))
per_row_ms = (time.monotonic() - t0) * 1000
t0 = time.monotonic()
with db:                                       # one transaction for the whole batch
    for i in range(2000):
        db.execute("INSERT INTO readings (v) VALUES (?)", (i,))
batched_ms = (time.monotonic() - t0) * 1000
print(f"2000 rows, 2000 commits: {per_row_ms:.0f} ms")
print(f"2000 rows, 1 commit:     {batched_ms:.0f} ms")
print("rows landed:", db.execute("SELECT count(*) FROM readings").fetchone()[0])
```

```output
2000 rows, 2000 commits: 22 ms
2000 rows, 1 commit:     2 ms
rows landed: 4000
```

An order of magnitude on the authoring machine — whose NVMe storage and write
caching flatter the per-commit case enormously; on modest hardware with honest
sync barriers the same experiment runs seconds against milliseconds, and the
engine's own FAQ answer on insertion speed explains why: a durable transaction
cannot outrun the platter or the flash erase block it waits on. The design
consequence is not "avoid commits" but the same boundary rule as before, read
from the other side: since a transaction is a unit of meaning, *bulk work whose
rows form one truth should arrive as one transaction* — an import, a scan's
findings, a batch of samples — and the meaning rule and the economics rule
converge on the same code. Where they genuinely diverge — a long stream of
independent truths, each of which must be durable the moment it happens, as in
the ledger the next chapters build — the per-commit price is not waste but the
purchase of exactly what was promised, paid knowingly. What the economics
forbid is only the unexamined middle: loops that commit per row out of habit,
buying two thousand durability guarantees to record one batch nobody needed
mid-batch.

## Rehearsal inside the transaction

One more instrument completes the transactional toolkit, and it answers a shape
of work the register's operators meet constantly: the attempt that may not
survive. A session's outer transaction holds the truths it is sure of; inside
it, an exploratory step — try strategy A, and if its verification fails, fall
back — needs an undo boundary of its own that does not forfeit the whole
session. SQLite's savepoints are transactions-within-transactions built for
exactly this:

```python
import sqlite3
db = sqlite3.connect("estate.db")
db.execute("CREATE TABLE ledger (id INTEGER PRIMARY KEY, entry TEXT)")
db.commit()
with db:
    db.execute("INSERT INTO ledger (entry) VALUES ('session opened')")
    db.execute("SAVEPOINT attempt")
    db.execute("INSERT INTO ledger (entry) VALUES ('tried strategy A')")
    strategy_a_worked = False
    if not strategy_a_worked:
        db.execute("ROLLBACK TO attempt")      # undo the attempt, keep the session
    db.execute("RELEASE attempt")
    db.execute("INSERT INTO ledger (entry) VALUES ('used strategy B instead')")
for row in db.execute("SELECT id, entry FROM ledger"):
    print(row)
```

```output
(1, 'session opened')
(2, 'used strategy B instead')
```

The failed attempt's row is gone; the session's opening row and the fallback
survived, all inside one outer commit — and the id sequence (1 then 2, no gap
in this run) is the successor's view of a history in which strategy A was
never recorded as tried. Whether that erasure is *correct* is a design
decision the pattern forces you to make explicitly, which is its second
virtue: an operator whose discipline says failed attempts are themselves
findings should record the failure as a fact — a committed row saying strategy
A was tried and did not verify — rather than leaving it inside the savepoint
to vanish. Rollback is for states that were never true; the ledger is for
events that really happened, failures included. The savepoint gives you the
mechanism for both readings and the obligation to choose between them, and
the register's honesty rules, as usual, decide the default: when in doubt,
the attempt happened, so the record stays.

## Durability's fine print

One last honesty layer, because "committed" is doing load-bearing work in this
chapter and its precise content should be on the table. What COMMIT promises
against *process* death is absolute and was demonstrated above. What it promises
against *power* death depends on the `synchronous` pragma and, beneath that, on
the storage stack telling the truth about flushes. At the default full setting,
the engine issues the sync barriers its atomic-commit protocol requires, and a
power cut yields either the before-state or the after-state — the documented
guarantee, contingent (as the documentation itself is careful to say) on disks
that do not lie about write completion. Operators tempted to trade this away
will find `synchronous = off` delivers real speed and a real risk: a badly timed
power loss can corrupt the database, not merely lose the last commit. The
estate's position is conservative and simple: leave `synchronous` at its
default; take the free and safe concurrency win of WAL mode when chapter 5
introduces it; and treat any tuning beyond that as requiring the engine
documentation's own "how to corrupt" page read in full first — the page exists
precisely because most corruption in the wild is operators defeating their own
guarantees. The register's blast-radius chapter taught that safety is
composition-time work; here, composition time is configuration time, and the
correct composition is mostly to decline to compose.

## What the transaction cannot promise

This chapter closes on the guarantee's honest boundary, because the estate's
worst failures live just past it. The transaction makes the *record* atomic. It
cannot make the record and the *world* atomic, and an operator's work is mostly
in the world: the service restarted, the email sent, the file deleted. Between
"the action happened" and "the row committed" there is always a gap — the
process can die after acting and before recording, or after recording an intent
and before acting on it — and no database on either side of the gap can close
it, because the gap is between two systems that share no transaction. This is
the estate's local edition of an old distributed-systems truth, and the
register's previous book met its behavioral half in the blast-radius chapter's
rule that a failed write is followed by a read. The estate adds the structural
half: design the records so that the gap, when it happens, is *detectable and
survivable* rather than silent.

Two patterns carry most of that weight, and both are schema patterns as much as
conduct patterns. The first is intent-then-outcome. An operation that touches
the world gets *two* writes: a committed row recording the intent before the
action ("about to restart nginx, reason, timestamp"), and a second write
recording the outcome after. A successor that finds an intent with no outcome
knows exactly what it inherits: an action whose fate is unknown, to be resolved
by reading the world — the service's actual state — before anything else
proceeds. Compare the alternatives. Record-only-after-acting, and a death in
the gap leaves an action that happened with no trace: the silent midden
failure, back again. Record-only-intent, and the ledger fills with plans
indistinguishable from history. The two-write pattern costs one extra commit
per world-action — the previous section priced it: cheap, and purchased for
exactly the moment it pays.

The second pattern is the idempotency key, and it turns the estate into a guard
against the retry accidents the register's operators are prone to by
constitution. Give every world-action a stable identity — the operation's
natural key, or a generated one carried in the task — and record it in a column
with a UNIQUE constraint. A retried operator that attempts to record the same
intent twice is refused by the schema itself, at which point the retry knows it
is a retry — before touching the world a second time. The pattern converts "did
I already send this?" from an unanswerable memory question into an INSERT whose
failure *is* the answer. Chapter 4 builds it into the ledger schema properly;
it is previewed here because it is transactional thinking applied at the
design layer: the uniqueness constraint is a transaction boundary drawn around
all time, not just around one session's statements — this action, ever, once.

The gap patterns also hand the estate's author a testing method worth
naming, because transactional code has the classic property of working
perfectly until the one moment it matters. The crash demonstration that
opened this chapter — a child process killed mid-transaction by `os._exit`
— is not just a teaching device; it is a reusable harness. Estate tooling
of any seriousness gets kill-tested: the critical write paths run in a
child, the child is killed at the awkward moments (after the intent, before
the outcome; mid-migration; between action and record), and the parent then
opens the estate and asserts the invariants this book has been accumulating
— no half-recorded changes, version number honest, exactly one of
intent-without-outcome or nothing. The harness costs twenty lines once and
converts this chapter's promises from believed to *demonstrated on your own
schemas* — which is, the reader will notice, precisely the relationship
this press's gate has to this book's listings. Trust arrives the same way
everywhere: something tried to break the claim, on the record, and failed.

Held together, the boundary reads like this. Inside the file, the transaction
gives you *no unintended truths*. At the file's edge, intent-then-outcome gives
you *no silent gaps* — every uncertainty is visible as an open intent. Across
runs, idempotency keys give you *no accidental repeats*. None of the three is
the others' substitute, and the estate needs all three precisely because its
operators end without warning. That is the full shape of the promise; what
remains is to write it down in tables a stranger can read.

A note on scope keeps the three-guarantee summary honest for the reader
building multi-file arrangements: the transaction's boundary is the
database it runs in. Two estates changed "together" by one session are two
transactions, with the gap between them exactly as real as the world-gap
above — one more argument for chapter 5's one-estate-per-lineage default,
and, where a split is genuinely earned, for treating cross-file
consistency by the same intent-then-outcome bookkeeping rather than by
hoping the two commits land as one. (The engine can in fact join attached
databases under a single atomic commit in most configurations, but the
estate declines to lean on machinery its operators would have to verify
per-setup; patterns that assume the gap survive every setup.)

The transaction, then: all-or-nothing against crashes, demonstrated; mechanics
visible on disk; boundaries drawn by meaning; the Python seams sealed; write
intent declared at entry; durability's contingencies stated. One promise,
carefully kept, and the estate stops being a pile of bytes and becomes a place
where truths can be deposited. What gets deposited, and in what shape a stranger
can inherit — that is schema, and it is the next chapter.
