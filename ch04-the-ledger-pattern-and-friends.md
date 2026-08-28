# Chapter 4 — The Ledger Pattern and Friends

*Draft status: author draft, gate-checked; human verification pending. Outputs are
real transcripts; the dead run in the registry listing is a real mid-task process
death.*

## Five shapes, most of memory

Operator memory feels endlessly various until you sort a few months of it, at
which point it collapses into a handful of recurring shapes. Things done. Places
reached in streams being read. Choices made and revised. Sessions begun and
ended. Files fetched or produced. This chapter gives each shape its table — the
ledger, the cursor, the config history, the run registry, the artifact index —
worked as running code with the chapter 3 disciplines already applied. They are
patterns, not a framework: no library to adopt, no dependency to carry, just
shapes to copy and adapt, which for estates meant to outlive their tooling is a
feature and not a modesty. Each section states the shape's contract — what it
promises a successor — because the contract, not the columns, is what makes a
pattern transferable.

A word on how the five relate before meeting them singly. The run registry is
the spine: everything else that happens, happens *during* some run, and rows
elsewhere carry the run's id so the estate can answer "what else did the session
that did this also do?" — the question incident reviews are made of. The ledger
records the runs' outward acts; the cursor and config tables record their
resumable inward state; the artifact index binds the file system's holdings into
the same web of provenance. One estate, five tables, joined — the composition
section at the end runs the queries that only the joined whole can answer.

## The ledger: things done, once, with their fates

The estate's centerpiece is the pattern chapter 2 previewed twice, now
assembled. A ledger row is an *operation with a fate*: what was to be done,
proof it was decided (the intent, committed before acting), and what became of
it (the outcome, committed after). The idempotency key makes the row a guard as
well as a record:

```python
import sqlite3
db = sqlite3.connect("estate.db")
db.execute("""
CREATE TABLE ledger (
  id INTEGER PRIMARY KEY,
  op_key TEXT NOT NULL UNIQUE,      -- idempotency key: this operation, ever, once
  action TEXT NOT NULL,
  intent_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ','now')),
  outcome TEXT,                     -- NULL = fate unknown; successor must resolve
  outcome_at TEXT,
  CHECK (NOT (outcome IS NOT NULL AND outcome_at IS NULL))
) STRICT
""")
with db:
    db.execute("INSERT INTO ledger (op_key, action) VALUES (?, ?)",
               ("restart-nginx-2026-08-28", "systemctl restart nginx"))
with db:
    db.execute("""UPDATE ledger SET outcome = 'is-active reported active',
                  outcome_at = strftime('%Y-%m-%dT%H:%M:%SZ','now') WHERE op_key = ?""",
               ("restart-nginx-2026-08-28",))
try:
    with db:
        db.execute("INSERT INTO ledger (op_key, action) VALUES (?, ?)",
                   ("restart-nginx-2026-08-28", "systemctl restart nginx"))
except sqlite3.IntegrityError:
    print("retry recognized: this operation is already in the ledger")
print(db.execute("SELECT op_key, outcome FROM ledger").fetchone())
```

```output
retry recognized: this operation is already in the ledger
('restart-nginx-2026-08-28', 'is-active reported active')
```

The contract, spelled out. Every world-touching act appears here before it
happens, so a successor never inherits invisible history. A NULL outcome is a
promise of honesty, not a gap: it marks exactly the operations whose fate must
be resolved by reading the world, and `WHERE outcome IS NULL` is the successor's
first ledger query. The UNIQUE refusal is the pattern's quiet triumph — the
retried operator in the listing learned it was a retry from the schema, at
insert time, *before* running the restart again. And the discipline that keeps
all this true is append-and-complete: intent rows are inserted, their outcome
fields are completed, and nothing is ever deleted or rewritten — corrections
are new rows referencing old ones, the same append-only covenant the register's
book demanded of ledgers in prose, now held by habit and CHECK together.

Two design notes earn their space. The op_key is chosen, not generated, when
the operation has a natural once-ness — "rotate credentials for host X during
window W" — and generated (and stored with the task that carries it) when it
does not; what matters is that the key's scope match the once-ness you mean,
which is a decision the pattern forces into the open. And the action column
records the *command as composed*, because the successor auditing an incident
wants what was actually dispatched — the register's exact-transcripts rule,
applied to memory.

## The cursor: where reading stopped

The second shape is the one this book's own predecessor kept in a flat file
and called a bookmark: for any stream consumed incrementally — a journal, a
feed, a log directory, an API's paginated history — the estate records how far
reading got, so the next session reads only what is new. The cursors table from
chapter 3 gets its writer, the upsert:

```python
import sqlite3
db = sqlite3.connect("estate.db")
db.execute("""CREATE TABLE cursors (stream TEXT PRIMARY KEY, position TEXT NOT NULL,
              advanced_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ','now'))) STRICT""")
def advance(stream, position):
    with db:
        db.execute("""INSERT INTO cursors (stream, position) VALUES (?, ?)
                      ON CONFLICT(stream) DO UPDATE SET
                        position = excluded.position,
                        advanced_at = excluded.advanced_at""", (stream, position))
advance("journal:nginx.service", "cursor=s=abc;i=44f0")
advance("journal:nginx.service", "cursor=s=abc;i=4512")
print(db.execute("SELECT stream, position FROM cursors").fetchall())
print("rows:", db.execute("SELECT count(*) FROM cursors").fetchone()[0])
```

```output
[('journal:nginx.service', 'cursor=s=abc;i=4512')]
rows: 1
```

One row per stream, always current, atomically replaced — the upsert (INSERT
that becomes UPDATE on key conflict) is the exact tool for state whose history
does not matter, and the listing's second call landing as an update, not a
second row, is the semantics on display. The contract has three clauses worth
enforcing by convention. Positions are *opaque*: the cursor stores whatever
resume token the stream's own tooling emits — a journald cursor string, an HTTP
ETag, a line offset — and no consumer ever parses it, so streams can change
their token format without breaking the estate. Advancement is *transactional
with processing*: the cursor moves in the same transaction that records what
was done with the new entries (chapter 2's units-of-meaning rule; a cursor
advanced before its entries are handled is data loss wearing a bookmark). And
staleness is *the reader's first question*: `advanced_at` exists so a successor
can distinguish a stream read minutes ago from one abandoned in June — the
difference between resuming and re-validating.

## Configuration with a memory

Operators make choices — polling intervals, thresholds, target lists — and the
midden stores them as bare current values, which answers "what is the setting?"
and is mute before the questions that actually arise: what was it before, who
changed it, and *why*? The estate stores configuration as history and derives
the present from it:

```python
import sqlite3
db = sqlite3.connect("estate.db")
db.executescript("""
CREATE TABLE settings (
  id INTEGER PRIMARY KEY,
  key TEXT NOT NULL,
  value TEXT NOT NULL,
  set_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ','now')),
  set_by TEXT NOT NULL,
  reason TEXT NOT NULL
) STRICT;
CREATE VIEW settings_current AS
  SELECT key, value, set_at, set_by, reason FROM settings s1
  WHERE id = (SELECT max(id) FROM settings s2 WHERE s2.key = s1.key);
""")
rows = [("poll_interval", "300", "author-session", "default"),
        ("poll_interval", "60", "author-session", "expedite request 2026-08-28"),
        ("retention_days", "90", "author-session", "default")]
with db:
    db.executemany("INSERT INTO settings (key, value, set_by, reason) VALUES (?,?,?,?)", rows)
print("current:", db.execute("SELECT key, value FROM settings_current ORDER BY key").fetchall())
print("history of poll_interval:",
      db.execute("SELECT value, reason FROM settings WHERE key='poll_interval' ORDER BY id").fetchall())
```

```output
current: [('poll_interval', '60'), ('retention_days', '90')]
history of poll_interval: [('300', 'default'), ('60', 'expedite request 2026-08-28')]
```

The mechanics are two ideas stacked. Writes are pure appends — nothing
UPDATEs, so no choice is ever erased by the next one — and the *view* derives
the current value as "the latest row per key", giving every consumer a table
that reads exactly like the flat config it replaced. (Views are the estate's
politeness layer generally: a stored query wearing a table's name, letting the
schema serve the stranger's common questions pre-composed.) The required
`reason` column is the pattern's soul, and it is required precisely because it
is what nobody records voluntarily. Every debugging session that ever ended
with "who set this to 60?!" was mourning this column. The row that answers it
here — an expedite request, dated, attributed — is this book's own production
history, recorded the way the pattern demands.

## The run registry: sessions and their ends

The fourth shape records the operators themselves. A run row marks a session's
birth (operator identity, task, start time) and — completed at exit, honestly —
its end and outcome. Its power is what *incomplete* rows mean. Because the
start is committed at startup and the end only at a clean exit, a row with
`ended_at NULL` whose operator is no longer alive is a session that died
mid-work, and the registry makes that inheritance visible instead of
archaeological. Demonstrated with a genuinely killed run:

```python
import sqlite3, subprocess, sys
db = sqlite3.connect("estate.db")
db.execute("""
CREATE TABLE runs (
  id INTEGER PRIMARY KEY,
  operator TEXT NOT NULL,
  task TEXT NOT NULL,
  started_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ','now')),
  ended_at TEXT,
  outcome TEXT CHECK (outcome IN ('ok','failed','abandoned') OR outcome IS NULL)
) STRICT
""")
db.commit(); db.close()
child = '''
import sqlite3
db = sqlite3.connect("estate.db")
with db:
    db.execute("INSERT INTO runs (operator, task) VALUES ('session-77', 'rotate logs')")
import os; os._exit(1)   # died mid-task; ended_at and outcome never written
'''
subprocess.run([sys.executable, "-c", child])
db = sqlite3.connect("estate.db")
open_runs = db.execute("""SELECT id, operator, task, started_at FROM runs
                          WHERE ended_at IS NULL""").fetchall()
print("unfinished business inherited by the successor:")
for r in open_runs: print(" ", r)
```

```output
unfinished business inherited by the successor:
  (1, 'session-77', 'rotate logs', '2026-08-28T18:00:49Z')
```

Session 77 died between its first commit and its last, and the registry holds
exactly the truth: a rotate-logs run began at 18:00 and never reported back.
The successor's protocol writes itself from the row: read the world (were the
logs rotated?), consult the ledger for session 77's intents (chapter 2's gap,
now navigable by join), then close the row honestly — `outcome = 'abandoned'`,
with a note — so the registry converges to a complete history instead of
accreting mysteries. The registry's second dividend is aggregate: because
every run lands here, "how have runs been ending lately" is one GROUP BY —
failure rates by task, duration drift, the trend that distinguishes a flaky
week from a broken change. The register's previous book put calibration in the
operator's conduct; the registry is where the calibration data has been
accumulating all along.

## The artifact index: files, vouched for

The fifth shape closes the loop chapter 1's taxonomy opened. Artifacts live in
the file system; the estate holds their papers — identity, origin, and a
content hash that converts "I think this is the file" into arithmetic:

```python
import sqlite3, hashlib, pathlib
def sha256(p): return hashlib.sha256(pathlib.Path(p).read_bytes()).hexdigest()
pathlib.Path("model-config.yaml").write_text("layers: 32\n")
db = sqlite3.connect("estate.db")
db.execute("""CREATE TABLE artifacts (path TEXT PRIMARY KEY, sha256 TEXT NOT NULL,
              origin TEXT NOT NULL, fetched_at TEXT NOT NULL) STRICT""")
with db:
    db.execute("INSERT INTO artifacts VALUES (?,?,?,?)",
               ("model-config.yaml", sha256("model-config.yaml"),
                "generated by session-77", "2026-08-28T18:40:00Z"))
path, recorded = db.execute("SELECT path, sha256 FROM artifacts").fetchone()
print("verify:", path, "MATCHES" if sha256(path) == recorded else "DRIFTED")
pathlib.Path(path).write_text("layers: 32\nquantized: true\n")   # someone touched it
print("verify:", path, "MATCHES" if sha256(path) == recorded else "DRIFTED")
```

```output
verify: model-config.yaml MATCHES
verify: model-config.yaml DRIFTED
```

The second verification caught the edit — someone (here, the listing itself,
playing the world's usual role) changed the file after it was indexed, and the
hash said so. That one bit, MATCHES or DRIFTED, is the difference between an
estate that *describes* its files and one that *vouches* for them: the
register's proof-of-target discipline, precomputed and stored. The index's
columns follow the provenance rules of chapter 3 (`origin` answers "where
from", `fetched_at` answers "how stale"), and its verification query — every
row, hash recomputed, mismatches reported — is a standing job chapter 7 will
fold into the estate's larger trust apparatus. Deliberately absent: the file
*contents*. The blob column exists and the index declines it, because chapter
1's taxonomy holds — streaming bytes is the file system's talent, vouching is
the database's, and the hash marries them without confusing them.

## The sixth shape: work that waits

One variation on the ledger earns shape status of its own, because it turns
the estate from memory into *coordination*: the queue. Where the ledger
records work already decided, a queue holds work waiting for a worker — and
the estate can serve it to concurrent claimants without a broker, using the
atomic read-modify-write that chapter 1 introduced, now with the modern
`RETURNING` clause handing back what was claimed:

```python
import sqlite3
db = sqlite3.connect("estate.db")
db.execute("""CREATE TABLE queue (id INTEGER PRIMARY KEY, task TEXT NOT NULL,
  claimed_by TEXT, claimed_at TEXT, done_at TEXT) STRICT""")
with db:
    db.executemany("INSERT INTO queue (task) VALUES (?)",
                   [("verify backups",), ("prune graveyard",), ("rotate keys",)])
def claim(worker):
    with db:
        row = db.execute("""UPDATE queue SET claimed_by = ?, claimed_at = strftime('%Y-%m-%dT%H:%M:%SZ','now')
                            WHERE id = (SELECT min(id) FROM queue WHERE claimed_by IS NULL)
                            RETURNING id, task""", (worker,)).fetchone()
    return row
print("worker A claims:", claim("A"))
print("worker B claims:", claim("B"))
print("worker A claims:", claim("A"))
print("worker B claims:", claim("B"))
```

```output
worker A claims: (1, 'verify backups')
worker B claims: (2, 'prune graveyard')
worker A claims: (3, 'rotate keys')
worker B claims: None
```

Each claim is one transaction — find the oldest unclaimed task, stamp it
with the claimant, return it — so two workers arriving simultaneously
cannot claim the same row (the single-writer queue of chapter 5 serializes
them), and the drained queue answers `None`, the affirmative nothing the
register's previous volume taught shots to say. The shape's obligations
follow the ledger's family line: completion is a second write (`done_at`,
plus outcome evidence), so a claimed-but-never-completed row is the queue's
version of the unfinished run — visible inheritance, reclaimed by a
staleness rule (claimed more than an hour ago by a worker whose registry
row has ended: back to the pool, with a note). This is not a message broker
and does not pretend to be — no pub/sub, no cross-host delivery, chapter
8's boundaries apply — but for the workload it fits — one machine's workers
sharing a task list through the file they already share — it replaces a
broker service with twelve lines, and the claim-lock at its center is the
same one this book's own publisher documents for its critic seats:
self-service, claim-locked, no coordinator.

## Choosing keys: the once-ness decision

The ledger's op_key looked like a detail and is actually the pattern's hardest
design question, so it earns a worked treatment. The key's job is to make the
schema refuse a *second* recording of the *same* operation — which means the
key must encode what "same" means, and "same" is a decision about the world,
not the database. Three scenarios, three different correct keys. A nightly
certificate renewal: the operation recurs by design, so the key includes the
occasion — `renew-web-cert:2026-08-28` — and a retry within the night is
refused while tomorrow's run is new. A migration applied to a host: once ever,
so the key is timeless — `apply-schema-v7:db-host-2` — and any future attempt,
weeks later, is correctly recognized as already done. A user-requested
one-off: once *per request*, so the key carries the request's identity —
`purge-quarantine:req-4415` — and the same user asking again tomorrow is a
new request, new key, new row. Get the scope too narrow and retries slip
through (a key with a timestamp to the second refuses nothing, since every
retry mints a fresh second); too broad and legitimate recurrences are refused
(the migration key without the host would block host 3 because host 2 was
done). The test that settles every case: *if two rows carried this key, would
the second necessarily be a mistake?* — and the key is built from exactly the
facts that make the answer yes. Surrogate ids (the `INTEGER PRIMARY KEY` every
table carries) answer a different question — row identity for joins — and the
two must not be conflated: the surrogate is for the *estate's* bookkeeping,
the op_key is for the *world's*.

## What the ledger refuses to hold

Patterns are defined by their exclusions as much as their columns, and two
exclusions keep ledgers healthy. Reads stay out — the previous volume's rule
("only writes get ledger lines") carries over with its reasoning intact: an
estate that ledgers its reads drowns its writes in noise, and the registry
already accounts for sessions wholesale. The judgment call arrives with
*consequential* reads — the probe that decided a failover, the check that
justified a purge. Those enter the record not as ledger rows but as evidence
*on* the write they motivated: the outcome column of the action they
triggered, or a fact row with provenance, keeping the ledger's every line an
act upon the world.

Secrets stay out absolutely, and the rule needs stating because ledger columns
attract them — the action that ran with a token, the config value that is a
password. The estate is one readable file; it travels in backups, gets opened
by strangers (that is its *purpose*), and mixes lifetimes (chapter 8's
retention will happily keep a ledger row for years past any credential's
rotation). Secrets therefore appear in the estate only as *references* — the
name of the key in the system keyring, the path to the credentials file, the
identity of the vault entry — never as values; the action column records the
command with the secret's reference, exactly as the previous volume's
transcripts learned to show `$TOKEN` rather than its expansion. The stranger
inheriting the estate learns where every secret lives and holds none of them,
which is the correct shape of that inheritance.

## The standing questions are part of the pattern

Each shape shipped with example queries, and the framing deserves promotion:
a pattern is not adopted until its standing questions are written down beside
it — named, tested, kept with the schema the way chapter 3 keeps comments.
The ledger's four: what is unresolved (`outcome IS NULL`, oldest first)?
what did run N do? has this op_key been seen? what failed in the last week?
The cursor's two: where is stream S? which streams have gone stale? The
config table's three: current values (the view); history of key K; what
changed since date D? The registry's three: open runs; outcomes by task over
window; duration drift. The artifact index's two: verify everything; what
did run N produce? Fourteen queries, each a line or two, and together they
are the estate's *interface* — the successor's briefing (chapter 8 composes
it), the handoff message's evidence, the monitoring hooks. Writing them down
at adoption time costs minutes and does something subtler than convenience:
it *tests the schema against its purpose* while the schema is still cheap to
change. A shape whose standing questions turn out awkward to write — a join
that needs a column nobody stored, a filter on a field inside a blob — is a
shape caught misdesigned on day one instead of month six, which is the
cheapest schema review an unattended operator will ever get.

## Order, and where it really comes from

One subtlety spans all five shapes and surfaces in incident reviews at the
worst moments: what orders the history? The intuitive answer — the
timestamp columns — is the fragile one. Timestamps tie (the second is this
book's stated precision, and a busy session commits several truths per
second), and clocks move (NTP corrections, timezone accidents on machines
less disciplined than chapter 3 demands), so two rows' timestamps can
disagree with the order the estate actually experienced. The reliable
answer is already in every table: the `INTEGER PRIMARY KEY` is allocated
monotonically as rows commit, so *id order is commit order* within an
estate, and every "what happened next" question — the incident walk, the
correction chain, the settings view's "latest per key" — keys on id, with
timestamps serving their real purposes: humans, staleness pricing, and
joins against the world's clocks (logs, journals) that ids cannot reach.
The convention costs nothing to adopt and one bad afternoon to retrofit,
which is why it is stated here, between the shapes it quietly orders.
(Its boundary is the estate itself: ids order one file's history; across
estates or against the world, timestamps — pinned UTC, chapter 3 — are
the only shared clock, carrying exactly the caveats above.)

## Composition: one estate, queryable whole

The five shapes pay their real dividend joined. Add `run_id` columns —
ledger entries, settings changes, and artifacts each carrying the run that
made them — and the estate becomes a single navigable account of the
operator's whole history. The incident query: everything session 77 did —
its ledger intents, its setting changes, its artifacts — in one pass, from
the run id the registry handed you. The audit query: every world-action whose
outcome is NULL, oldest first, with the run that owes it. The trust query:
every artifact fetched by runs that later failed, for re-verification. The
calibration query: median run duration by task, this month against last.
None of these is an engineering project; each is a SELECT against tables this
chapter already built, which is the payoff chapter 1 promised when it said
state you cannot query is barely state at all. The midden could not answer
one of them.

A day in the composed estate makes the joins concrete. A session wakes,
registers its run (registry row 214, operator session-92, task "monthly cert
sweep"), and asks the ledger whether the sweep's op_key has been seen — new
month, new key, clean insert: intent recorded. It reads the cursor for the
certificate transparency stream, fetches what is new, and finds one
certificate nearing expiry. The renewal is a world-action: intent row with
op_key `renew-mail-cert:2026-09`, the renewal runs, the functional probe
passes, outcome completed — one transaction per truth, exactly as chapter 2
drew the boundaries. The new certificate file lands in the artifact index
with its hash and origin; the cursor advances in the same transaction that
recorded what the new entries produced; a journal entry (chapter 6) writes
the sentence a future searcher will want. The session ends; the registry row
closes with outcome ok. Nothing in the day required coordination, and yet
every question a supervisor, successor, or incident review could ask — what
ran, what changed, what proves it, what was produced, where reading stopped —
has one answer, in one file, joined by run id 214. That is the composition
argument in narrative form: not that five tables are tidier than five files,
but that the day's *whole shape* became queryable because its parts agreed
on keys.

The patterns also compose *downward* into discipline the register's book left
as conduct. Its evidence blocks now have an address (outcome columns); its
change ledger has a schema instead of a format convention; its handoff
message's five answers are five queries. And one estate serves one operator
lineage at a time so far — every listing in this chapter wrote from a single
connection. Real estates get written by concurrent generations: the timer
firing while the interactive session works, the second agent dispatched in
parallel. Two operators, one file, no coordinator — that is chapter 5, and
the engine has been waiting for it.
