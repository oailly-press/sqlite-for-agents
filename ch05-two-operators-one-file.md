# Chapter 5 — Two Operators, One File

*Draft status: author draft, gate-checked; human verification pending. Outputs are
real transcripts; the two-process counter at the chapter's end is a genuine
concurrent run.*

## The second operator arrives

Every listing so far wrote from one connection, which flattered a fiction: that
the estate has one tenant. Real estates do not. The timer fires its report job
while the interactive session is mid-task; a supervisor dispatches two agents
whose work overlaps; yesterday's run, believed dead, turns out to be alive and
finishing. The register's previous book met this world with `flock` and taught
the honest limits of advisory locking: it protects the writers who remember to
take the lock. The estate can do better, because the coordination this chapter
needs is not bolted onto the file — it *is* the file. SQLite's locking is
mandatory for everyone who comes through the library, which is everyone; there
is no code path that writes the database around it. Two operators that have
never heard of each other, sharing nothing but a path, get correctness anyway.
What they do not get is freedom from each other's *timing* — and this chapter
is about the difference: what the engine guarantees unasked, what it asks the
operator to decide (chiefly: how long to wait), and which famous SQLite
complaint — `database is locked` — is not a malfunction but a question
addressed to you.

The mental model to install first is the single-writer truth. However many
connections hold the estate open, SQLite permits exactly one write transaction
at a time; writers *queue*, they never interleave. Everything else in the
chapter — the refusal, the timeout, WAL's reader liberation, the throughput
ceiling — is a consequence of that one design decision, which is also the
decision that makes chapter 2's promises cheap enough to keep. Multi-writer
engines pay for their concurrency in machinery (row locks, MVCC vacuuming,
conflict resolution) and in sharper failure modes; the estate's engine chose
instead to make the common case — few operators, brief writes — simple and
bulletproof. The design fit is worth noticing: operator estates are almost
definitionally low-contention. Sessions write in bursts, ledgers take a row at
a time, nothing holds transactions across human-scale pauses (chapter 2's
boundaries rule already forbade it). The single-writer queue is not a limit the
estate suffers; it is the contract the estate's workload was born matching.

## The refusal, witnessed

Here is the collision, staged small and read closely, because everything the
operator must decide follows from its anatomy:

```python
import sqlite3
a = sqlite3.connect("estate.db")
a.execute("CREATE TABLE t (x INTEGER) STRICT"); a.commit()
b = sqlite3.connect("estate.db")
b.execute("PRAGMA busy_timeout = 0")     # refuse to wait, so the refusal is visible
a.execute("BEGIN IMMEDIATE")
a.execute("INSERT INTO t VALUES (1)")
try:
    b.execute("BEGIN IMMEDIATE")
except sqlite3.OperationalError as e:
    print("second operator:", e)
a.commit()
b.execute("BEGIN IMMEDIATE"); b.execute("INSERT INTO t VALUES (2)"); b.commit()
print("rows after both:", a.execute("SELECT count(*) FROM t").fetchone()[0])
```

```output
second operator: database is locked
rows after both: 2
```

Operator A holds the write slot mid-transaction; operator B asks for it and is
told no; A finishes; B asks again and succeeds; both rows land. Nothing was
corrupted, nothing was lost, nobody's write interleaved with anybody's — the
lost-update accident of chapter 1 is structurally absent. What B received was
not an error in the register's sense but a *status*: `SQLITE_BUSY`, surfaced by
Python as that famous message, meaning precisely "the slot is taken; try again
later." The listing forced the refusal into view by setting the busy timeout to
zero — B declared it would not wait, so it didn't. That declaration is the
operator's real decision surface, and the default answer is wrong for estates:
a fresh connection's timeout is effectively no patience at all, which converts
every routine collision into an exception. The registers' operators know this
error's cousin from package managers — "could not get lock" — and know the
diagnosis is usually *someone else is legitimately working*, not *something is
broken*. The estate's version deserves the same reading.

## Patience is configuration

What B should have done is wait — briefly, boundedly, and without any code for
it, because waiting is built in:

```python
import sqlite3, threading, time
a = sqlite3.connect("estate.db")
a.execute("CREATE TABLE t (x INTEGER) STRICT"); a.commit()
a.execute("BEGIN IMMEDIATE")
a.execute("INSERT INTO t VALUES (1)")            # first operator holds the write lock
result = {}
def second_operator():
    b = sqlite3.connect("estate.db")             # its own connection, its own thread
    b.execute("PRAGMA busy_timeout = 2000")      # willing to wait up to 2 s
    t0 = time.monotonic()
    with b:
        b.execute("INSERT INTO t VALUES (2)")
    result["waited"] = time.monotonic() - t0
th = threading.Thread(target=second_operator); th.start()
time.sleep(0.4)
a.commit()                                       # first operator releases
th.join()
print(f"second operator wrote after waiting {result['waited']:.1f}s")
print("rows:", a.execute("SELECT count(*) FROM t").fetchone()[0])
```

```output
second operator wrote after waiting 0.4s
rows: 2
```

B asked, was refused, and simply *stayed in line*; four-tenths of a second
later A released, B wrote, and no exception ever surfaced. The
`busy_timeout` pragma is the whole mechanism: below it, the engine retries
acquisition for up to the stated bound before giving up and returning BUSY.
Choosing the bound is register economics, and the register's own rules apply
verbatim. The wait must exist (zero patience turns normal coexistence into
failure), must be bounded (infinite patience is the hang chapter 1 of the
previous book banned), and the bound should be derived from the neighbors: a
touch longer than the longest write transaction any well-behaved tenant runs —
which chapter 2's boundary discipline already made short. This book's default
is five seconds, set in `open_estate()` beside the foreign-keys pragma, one
decision made once. And when the bound is genuinely exceeded — BUSY *after*
five seconds — the correct reading changes: now something probably is wrong (a
tenant died holding nothing, since crashed processes release locks with their
file handles; more likely a tenant is violating the short-transaction covenant)
and the register's failure discipline takes over: record the refusal in the
run's own account, read the world, do not hammer.

Two mechanics footnotes belong here because their absence causes real
confusion. First, the incidental lesson the listing's shape teaches: Python's
sqlite3 binds each connection to its creating thread by default — the waiting
operator built its own connection inside its thread because sharing one across
threads is refused by the module. Connections are cheap; the pattern is one
per thread, or `check_same_thread=False` accepted knowingly with external
serialization. Second, `BEGIN IMMEDIATE` is what makes waiting *work*: chapter
2 installed it to convert mid-transaction refusals into at-entry waits, and
this is the payoff — the busy handler can only wait politely at moments where
waiting is safe, and a deferred transaction that already read and now wants to
write is not such a moment (the engine returns BUSY immediately there,
timeout notwithstanding, precisely because waiting could deadlock two
half-done readers forever). Say IMMEDIATE; wait at the door, not on the
stairs.

## WAL: the readers go free

The classic journal mode has one genuinely operator-hostile trait left: writers
and readers contend — a long read can hold off a writer, a committing writer
excludes readers at the wrong moment. The modern cure is one pragma, and it is
the single most valuable configuration line in this book:

```python
import sqlite3, os
w = sqlite3.connect("estate.db")
print("journal mode now:", w.execute("PRAGMA journal_mode = WAL").fetchone()[0])
w.execute("CREATE TABLE facts (n INTEGER) STRICT")
with w:
    w.executemany("INSERT INTO facts VALUES (?)", [(i,) for i in range(3)])
r = sqlite3.connect("estate.db")
r.execute("BEGIN")                                   # reader opens its snapshot
before = r.execute("SELECT count(*) FROM facts").fetchone()[0]
with w:
    w.execute("INSERT INTO facts VALUES (99)")       # writer commits DURING the read txn
during = r.execute("SELECT count(*) FROM facts").fetchone()[0]
r.commit()
after = r.execute("SELECT count(*) FROM facts").fetchone()[0]
print(f"reader saw: {before} rows, then {during} inside the same snapshot, then {after} in a new one")
print("sidecars:", sorted(f for f in os.listdir(".") if f.startswith("estate.db-")))
```

```output
journal mode now: wal
reader saw: 3 rows, then 3 inside the same snapshot, then 4 in a new one
sidecars: ['estate.db-shm', 'estate.db-wal']
```

Write-ahead logging inverts chapter 2's journal: instead of saving old pages
aside and writing new ones in place, the engine appends new pages to a log and
leaves the main file alone, folding the log back in later ("checkpointing").
Three consequences, all visible in the transcript. Readers no longer block
writers or vice versa — the writer committed mid-read-transaction without
either party waiting. Readers get *snapshot isolation* for free: inside one
read transaction the reader saw 3 rows, then still 3 *after* the concurrent
commit — a stable world to compute over, the moving-substrate problem of the
register's network chapter solved outright at the estate's door — and the new
truth appeared only when the reader opened a new transaction. And the sidecar
population changed: `-wal` and `-shm` now accompany the database, persistently
(WAL mode is a property of the file, surviving reopen), with chapter 2's rule
unchanged and now sharper — *the sidecars are part of the database*, and
chapter 7 will show that copying around them is the classic way to lose
committed data. Two honest caveats bound the gift: WAL requires shared memory,
so it is unavailable or unsafe on network filesystems — estates live on local
disks, which they should anyway (the engine's corruption documentation has a
section on network filesystems that reads like a warning label) — and a read
transaction held open for a long time pins the WAL from checkpointing, so the
log grows; the short-transaction covenant turns out to bind readers too.

## Connections: how many, held how long

The chapter's demos juggled connections freely, and real estates need the
lifecycle stated. A connection is cheap to open — a file open and a header
read, microseconds locally — so the previous volume's instincts about
connection pooling (a server-database economics) do not transfer; an operator
that opens at session start and closes at exit is doing it right, and a
scheduled job that opens, works, and closes has nothing to optimize. The
rules that do matter are about *holding*. One connection per thread (the
module's binding rule, met above). One estate connection per operator
process, not per function — chapter 2's `with db:` blocks share it safely,
and a process that opens dozens of connections to one file is manufacturing
its own lock traffic. And nothing *holds a transaction* across a wait: not a
network call, not a subprocess, not a model inference, not user input. The
WAL section's caveat gives the reader's version teeth — a read transaction
held open pins the snapshot, so an operator that opens a read transaction
and then thinks for ten minutes is forcing the WAL to retain ten minutes of
history for a view nobody needed frozen — and the writer's version is
chapter 2's covenant with a sharper reason: the write slot is *exclusive*,
so a transaction held across a thirty-second inference is thirty seconds of
every other tenant's timeout budget. Transactions bracket *database work*,
never *thinking*; the estate connection lives long, its transactions live
milliseconds.

## Durability under WAL: one honest knob

Chapter 2 counseled leaving `synchronous` at its default and this chapter
must refine that advice once, because WAL changes the trade's terms in the
operator's favor. Under rollback journaling, lowering sync guarantees risks
corruption; under WAL, the engine documents a gentler middle: `synchronous =
NORMAL` syncs at checkpoints rather than at every commit, cannot corrupt the
database on power loss, and risks only the *most recent commits rolling
back* if power dies before the next checkpoint — process crashes lose
nothing either way. For an estate on a workstation or a battery-backed
machine, WAL + NORMAL is the documented sweet spot and this book's
recommendation, set in `open_estate()` with the reason recorded in the
settings history (chapter 4's pattern eating its own cooking). The estate
declines to go further: `synchronous = OFF` re-enters corruption territory,
and the ledger's whole value is that its last row can be believed. The
decision, either way, is one line — and the point of teaching it is less
the milliseconds than the method: durability settings are *estate policy*,
chosen once, recorded with reasons, never adjusted silently mid-incident by
whichever session is frustrated with a slow loop.

## The read-only seat

Not every tenant deserves the pen. Reporting sessions, dashboards, the
supervisor's audit, chapter 7's restore drills — all read; none should be
*able* to write, because ability is blast radius whether or not intent
exists (the previous volume's least-privilege doctrine, verbatim). The
engine provides the seat:

```python
import sqlite3
rw = sqlite3.connect("estate.db")
rw.execute("CREATE TABLE facts (fact TEXT) STRICT")
with rw: rw.execute("INSERT INTO facts VALUES ('reports read; they do not write')")
ro = sqlite3.connect("file:estate.db?mode=ro", uri=True)
print("read-only seat reads:", ro.execute("SELECT fact FROM facts").fetchone()[0])
try:
    ro.execute("INSERT INTO facts VALUES ('surely just this once')")
except sqlite3.OperationalError as e:
    print("read-only seat writes:", e)
```

```output
read-only seat reads: reports read; they do not write
read-only seat writes: attempt to write a readonly database
```

The URI form's `mode=ro` refuses writes at the connection, cheaply and
unconditionally — a report with a bug cannot corrupt the record it reports
on, which is the property that lets scheduled reporting run without the
scrutiny writes earn. Two strengthenings extend the seat. For genuinely
frozen files — archives, the backups chapter 7 verifies — `immutable=1`
goes further, promising the engine the file cannot change and skipping
lock traffic entirely (a promise that must be *true*; it is for cold
backups, never for live estates). And beneath the engine sits the outer
wall the register already taught: the estate file's unix permissions
decide who reaches it at all, and an operator lineage's estate belongs to
that lineage's user, mode 600, with the supervisor's audit seat granted
through group read — file-system enforcement backing engine politeness,
the same layering the previous volume built for every other durable
asset.

## Reading a BUSY like an operator

The refusal taxonomy, assembled for the diagnostic reflexes. BUSY at BEGIN
IMMEDIATE, within the timeout, resolving on retry: weather — a neighbor
writing, the queue working as designed; record nothing, proceed. BUSY
persisting past a generous timeout: a tenant is violating the
short-transaction covenant; the culprit is found not with database tools but
with the previous volume's — `fuser` on the estate file names the processes
holding it open (a fragment for the same PATH reasons as ever), and the run
registry says which *operator* each process claims to be; the fix is the
neighbor's transaction shape, never a longer timeout arms race. BUSY at a
*deferred* transaction's midpoint (the upgrade case): a design bug in the
asking code — the transaction read before declaring write intent; the fix is
IMMEDIATE, and no timeout would have helped. And the exotic
`SQLITE_BUSY_SNAPSHOT` under WAL — a writer whose snapshot has been
overtaken — resolves by restarting the transaction fresh. Four faces, four
different next moves, one diagnostic principle carried over whole from the
register: the refusal's *timing and persistence* carry the diagnosis, and
hammering retries without reading them converts information into noise.

One WAL mechanic belongs in the operator's model because it is the mode's
only moving part: the checkpoint, the act of folding the log back into the
main file. By default it happens automatically (around a thousand log
pages), opportunistically, and invisibly — the right arrangement, left
alone. The operational readings: the `-wal` file's *size* is the health
gauge (steady modest size: checkpointing is keeping up; monotonic growth:
something is pinning it — almost always the long-lived read transaction
the connection section indicted, found via the registry and cured by
fixing the reader, not by forcing checkpoints); and the one legitimate
manual intervention is maintenance-shaped — `PRAGMA
wal_checkpoint(TRUNCATE)` at a quiet moment (the backup session, the
handoff) folds everything in and shrinks the log to zero, leaving the
estate compact for the copy or the inheritance. What the operator never
does is treat checkpointing as a correctness lever: committed data is
equally durable in the log and the main file (chapter 7's engine-mediated
copies know where truth lives), so checkpoint management is housekeeping
economics — log size against fold-in cost — and the default economics are
already good. Know the gauge, fix the pinner, truncate at handoff; the
rest is the engine's business.

## What must never be shared

The file shares; two things above it must not, and both failure modes are
documented corruption paths rather than theory. The first is the connection
object across a `fork()`. An operator that opens the estate and then forks
— the daemonization dance, a multiprocessing pool created after connecting
— hands both processes the same open file descriptors and the same
in-library state, and the engine's how-to-corrupt documentation lists
exactly this as a way to break a database: two processes unknowingly
sharing what each believes is a private connection. The rule is mechanical:
*open after fork, never before* — child processes create their own
connections (the chapter's subprocess listings all do; multiprocessing
workers open inside the worker function), and a process that must fork
closes the estate first. The second is the file across *hosts* via a
shared filesystem — the network-mount warning of the WAL section,
generalized: SQLite's coordination is exactly as good as the filesystem's
locking, network filesystems' locking is historically exactly not good
enough, and the corruption documentation's dedicated section on it is the
most-cited page in this book's references for a reason. Same machine,
different processes: share freely, the whole chapter is the proof. Different
machines: the estate does not stretch; chapter 8 names what does. Between
those two rules and the covenant, the sharing story is complete — and
notably free of locks the operator must remember, which was the entire
point of paying an engine to remember them.

## The proof at scale

The chapter's claims assembled and stress-tested — two genuinely separate
processes, no shared state but the path, a hundred read-modify-writes each:

```python
import sqlite3, subprocess, sys
db = sqlite3.connect("estate.db")
db.execute("PRAGMA journal_mode = WAL")
db.execute("CREATE TABLE counters (name TEXT PRIMARY KEY, value INTEGER NOT NULL) STRICT")
db.execute("INSERT INTO counters VALUES ('runs', 0)"); db.commit()
worker = '''
import sqlite3
db = sqlite3.connect("estate.db")
db.execute("PRAGMA busy_timeout = 5000")
for _ in range(100):
    with db:
        db.execute("UPDATE counters SET value = value + 1 WHERE name = 'runs'")
'''
procs = [subprocess.Popen([sys.executable, "-c", worker]) for _ in range(2)]
for p in procs: p.wait()
print("two operators, 100 increments each, final count:",
      db.execute("SELECT value FROM counters WHERE name='runs'").fetchone()[0])
```

```output
two operators, 100 increments each, final count: 200
```

Two hundred, exactly — set beside chapter 1's flat-file counter, which lost
half its updates to two writers *in the same process being careful*. The
workers here share no locks of their own, coordinate nothing, and could have
been written by strangers; the busy timeout absorbed their collisions and the
single-writer queue serialized their arithmetic. This is the estate's
concurrency story in one number: correctness is the engine's job and arrived
by default; the operator's whole contribution was two pragmas and short
transactions.

## Where the file lives, and how many files there are

Concurrency questions are often placement questions wearing a disguise, so
the estate's geography gets settled here. *Where:* on a local filesystem,
always — the WAL caveat and the corruption documentation agree — and by
convention where the previous volume put durable operator state: under the
platform's state directory (`~/.local/state/<operator>/estate.db` for a
per-user operator, `/var/lib/<operator>/` for a system one), never in
scratch, never on a network mount, never in a synced-folder product whose
sync engine is precisely the naive copier chapter 7 indicts. *How many:*
one estate per *accountable lineage* — the operator identity that owns the
ledger's promises — which usually means one per agent-role per machine.
Splitting finer (per task, per session) shreds the composition dividend the
previous section just demonstrated; merging coarser (all operators on a
host in one file) couples unrelated write queues and makes chapter 8's
retention policy a negotiation. The two legitimate splits are the ones
already earned: high-rate sample tables (this chapter's ceiling section)
and scratch (never in the estate at all). And when split files must be
queried together, the engine's `ATTACH` joins them at read time — one
connection, two files, cross-database SELECTs — so the split costs analysis
nothing; a reporting session attaches the samples database read-only beside
the estate and the composed queries run as if the seam were not there. The
geography, like every estate policy, goes in the settings table with its
reason, because the successor's first question — *where is everything?* —
deserves a recorded answer rather than a convention remembered.

## The ceiling, and the covenant

Honesty about where the story ends. Writers queue, so write throughput has a
ceiling: roughly, one write transaction's duration times the queue's length is
everyone's latency, and a fleet of chatty writers against one estate will feel
it — first as waits, then as timeouts. The remedies escalate in order: batch
(chapter 2's economics — most "many writes" are one truth); shorten
transactions (the covenant again — nothing holds the write slot across a
network call or a model inference, ever); split estates along real seams (the
scratch/records boundary of chapter 1 often marks files that never needed to
be one — a high-rate sample log is its own database, attached when queried);
and, when a workload is genuinely many concurrent writers across many hosts,
concede the case to chapter 8, which is where this book hands such workloads
to server databases without embarrassment. What the ceiling almost never
justifies is the move folklore reaches for first — disabling the durability
that makes the estate worth having. The engine's own guidance pages order the
levers the same way: transaction shape first, WAL second, hardware third,
guarantees last and reluctantly.

The covenant that keeps the whole chapter working fits in three lines an
operator can hold: open through one ritual (foreign keys on, busy timeout
set, WAL on); say IMMEDIATE when you will write, and keep every transaction
— reader or writer — short; treat BUSY within the bound as weather, and BUSY
beyond it as a finding about a neighbor. Under that covenant, the estate
scales exactly as far as operator memory needs it to — and the file stays
one file, inheritable whole, which the next chapter finally teaches the
operator to *search*.
