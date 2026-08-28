# Chapter 7 — Trust, Verify, Repair

*Draft status: author draft, gate-checked; human verification pending. Outputs are
real transcripts; the corruption in the final listing is real damage, inflicted on
a scratch database by the listing itself.*

## Inherited trust is not trust

Every chapter so far wrote the estate; this one inherits it. The successor
operator opens a file it did not create, holding records it did not witness,
and must decide how much weight the file can bear — before building on it, not
after. The register's previous volume made this posture a reflex for machines
("the four-question routine", "proof of target"); the estate needs the same
reflex with different instruments, because a database can fail in ways a
transcript cannot: bytes rot, copies go stale, sidecars get separated from
their files, and — the quiet majority of real incidents — the backup everyone
trusted turns out to have been wrong every night for a year. The good news is
proportionate: the estate's engine ships verification as a first-class
operation, cheap enough to run on schedule, and the correct-backup problem has
exact, documented answers. This chapter is those instruments, ordered as the
successor meets them: verify what you inherited, back up what you verified,
and know the recovery ladder before you need its rungs.

The posture to install is the same one the previous volume gave outward
reports: *claims sized to evidence*. An estate is not "fine" because it opens
— chapter 2 showed opening does almost nothing — and not "backed up" because
a file named backup exists. It is fine because `integrity_check` said `ok`
recently; it is backed up because a restore was *drilled*. Everything below
mechanizes those two sentences.

## The opening move

SQLite's self-audit walks the entire file — every page, every index, every
constraint's storage — and returns either the single row `ok` or a list of
what is wrong:

```python
import sqlite3, pathlib
db = sqlite3.connect("estate.db")
db.execute("CREATE TABLE facts (id INTEGER PRIMARY KEY, fact TEXT) STRICT")
with db:
    db.executemany("INSERT INTO facts (fact) VALUES (?)", [(f"fact {i}",) for i in range(200)])
print("healthy file says:", db.execute("PRAGMA integrity_check").fetchone()[0])
db.close()
raw = bytearray(pathlib.Path("estate.db").read_bytes())
raw[4096:4160] = b"\xde\xad" * 32                    # 64 bytes of damage mid-file
pathlib.Path("estate.db").write_bytes(raw)
db = sqlite3.connect("estate.db")
try:
    verdict = db.execute("PRAGMA integrity_check").fetchall()
    print("damaged file says:", verdict[0][0][:70])
except sqlite3.DatabaseError as e:
    print("damaged file says:", e)
```

```output
healthy file says: ok
damaged file says: database disk image is malformed
```

Sixty-four bytes of damage, surgically inflicted mid-file, and the audit caught
it — where a JSON midden with the same wound would have parsed cleanly or
failed confusingly depending on where the bytes landed, and a naive estate
consumer might have read plausible garbage for weeks. Operating doctrine for
the check: it runs at *inheritance* (a successor's first act on an estate it
did not close), at *backup* (below — verifying the copy, which is the copy
that matters), and on *schedule* for long-lived estates. Its cost scales with
file size; the lighter `PRAGMA quick_check` skips the slowest cross-checks and
is the right compromise for large estates checked often, with the full check
reserved for backups and suspicion. Two companions complete the audit
toolkit: `PRAGMA foreign_key_check` reports orphaned references (chapter 3's
switch enforces them per-connection going forward, but rows written by some
past pragma-forgetting tenant are findable only by asking), and the
application-level audits this book has been accumulating all along — chapter
4's artifact hashes, chapter 6's index-sync counts — run beside the engine's,
because the engine can only vouch for storage, never for meaning.

Verification's price list keeps the schedule honest, so the costs go on
record with the doctrine. `integrity_check` reads every page and checks
every index against its table — I/O-bound, linear in file size, seconds
for the megabyte estates this book's patterns produce and minutes only
when an estate has ignored chapter 8's retention for years. `quick_check`
skips the slow cross-checks (index-to-table consistency among them) for
roughly order-of-magnitude savings, which is why the daily seat belongs to
it and the full check rides the backup cadence, where its cost disappears
into an operation that reads the whole file anyway. `foreign_key_check`
scales with the referencing tables it scans; the application audits cost
whatever their queries cost, which chapter 3's index discipline already
bounded. None of it approaches the price of the alternative — the
register's economics, one last time: a verification is a read, reads are
cheap, and the one commodity that cannot be bought back after the fact is
the confidence that yesterday's estate was sound *yesterday*, attested on
the record, by a check that ran when nobody was worried.

One boundary of the engine's audit deserves explicitness before the backup
sections rely on it: `integrity_check` proves the file is a well-formed
database; it does not prove it is *your* database. A backup restored from
the wrong generation, an estate swapped by mistake, rows deleted by an
authorized-but-wrong session — all audit `ok`, because storage soundness
was never identity or completeness. The estate's own layers carry that
weight where it matters: the info table names the estate and its lineage
(chapter 3), the artifact index's hash pins each backup generation to its
recorded identity, and — for estates whose threat model includes tampering
rather than mere accident — the ledger's append-only shape extends
naturally to a verification chain, each row carrying a hash over its
content plus its predecessor's hash, making silent rewriting of history
detectable by one walk. Most operator estates stop well short of that
last measure, and should; the point of naming the layers is the habit of
asking, for each trust question, *which* instrument actually answers it —
the engine for bytes, the schema for meaning, provenance for identity,
and, at the far end, cryptography for adversaries. Chapter 5's threat was
concurrency and the engine answered it; this chapter's is decay and
mistake; the adversarial case is real but rarer, and an estate that
reaches it has usually outgrown one file for reasons chapter 8 already
catalogs.

## The backup that lies

Chapter 5 promised that copying a live WAL database loses data; here is the
loss, measured. The scenario is the commonest backup bug in SQLite's world: a
cron job that `cp`s the main file, unaware that recent commits — sometimes
all commits — live in the `-wal` sidecar until a checkpoint folds them in:

```python
import sqlite3, shutil
db = sqlite3.connect("estate.db")
db.execute("PRAGMA journal_mode = WAL")
db.execute("PRAGMA wal_autocheckpoint = 0")          # keep recent commits in the -wal
db.execute("CREATE TABLE ledger (id INTEGER PRIMARY KEY, entry TEXT) STRICT")
with db:
    db.executemany("INSERT INTO ledger (entry) VALUES (?)",
                   [(f"entry {i}",) for i in range(500)])
print("live estate holds:", db.execute("SELECT count(*) FROM ledger").fetchone()[0])
shutil.copyfile("estate.db", "naive-backup.db")      # cp on a live WAL database
naive = sqlite3.connect("naive-backup.db")
try:
    print("naive copy holds:", naive.execute("SELECT count(*) FROM ledger").fetchone()[0])
except sqlite3.OperationalError as e:
    print("naive copy says: ", e)
```

```output
live estate holds: 500
naive copy says:  no such table: ledger
```

Worse than losing rows: the copy lost the *table*. Every commit of this young
database still lived in the write-ahead log, so the main file held little more
than a header, and the backup — well-named, timestamped, dutifully rotated —
would restore to an empty estate. The demonstration pins the extreme case by
disabling auto-checkpointing, but the production version differs only in
degree: with default checkpointing the naive copy is missing *whatever
committed since the last checkpoint*, an amount that varies invisibly from
minute to minute — a backup whose completeness is a coin toss weighted by
timing. And the rollback-journal sibling of this bug is nastier still: a `cp`
taken mid-transaction can capture a half-committed page set that *the copy has
no journal to repair*, yielding not a stale backup but a corrupt one. The rule
absorbs in one line — **a database in use cannot be copied by copying its
file** — and the honest versions cost nothing, as the next listing shows.

## The backup that tells the truth

The engine offers two correct paths, both usable while the estate is live.
`VACUUM INTO` writes a complete, transactionally consistent, freshly-packed
copy to a new file in one statement; the online backup API (exposed in Python
as `Connection.backup`) streams a consistent copy page by page, politely
yielding to concurrent writers. The first is the estate's default for its
simplicity and the compaction it throws in free:

```python
import sqlite3
db = sqlite3.connect("estate.db")
db.execute("PRAGMA journal_mode = WAL")
db.execute("PRAGMA wal_autocheckpoint = 0")
db.execute("CREATE TABLE ledger (id INTEGER PRIMARY KEY, entry TEXT) STRICT")
with db:
    db.executemany("INSERT INTO ledger (entry) VALUES (?)",
                   [(f"entry {i}",) for i in range(500)])
db.execute("VACUUM INTO 'true-backup.db'")           # transactionally consistent copy
bak = sqlite3.connect("true-backup.db")
print("backup holds:      ", bak.execute("SELECT count(*) FROM ledger").fetchone()[0])
print("backup integrity:  ", bak.execute("PRAGMA integrity_check").fetchone()[0])
```

```output
backup holds:       500
backup integrity:   ok
```

Same live database, same uncheckpointed WAL, and the copy holds all five
hundred rows with a clean audit — because the copy was made *through the
engine*, which knows where the truth currently lives, instead of through the
file system, which does not. The listing also models the doctrine's second
clause: the verification ran *on the backup*, in the same breath as its
creation. An unverified backup is a hope with a filename; this one leaves the
session as a claim with evidence, ready for chapter 8's handoff. Around the
mechanism goes ordinary register discipline, inherited wholesale from the
previous volume: timestamped backup names (its reversibility chapter),
retention by schedule (its timer chapter), the backup recorded in the ledger
with outcome and integrity verdict (chapter 4), and — the clause institutions
skip until an incident teaches it — the *restore drill*: on some schedule, a
backup is actually opened, actually queried, actually compared against
expectations, because "backup succeeded" and "restore works" are different
facts and only the second one was ever the point. The drill, too, is one
`open_estate()` against yesterday's file plus a handful of the standing
queries — for an unattended operator, a scheduled job like any other.

## The trust ladder for inherited estates

Verification instruments in hand, the successor's opening posture can be
graded rather than binary — a ladder of earned trust, climbed as far as the
work requires, mirroring the evidence disciplines the previous volume built
for machines. Rung zero: the file opens and identifies itself (`user_version`,
`sqlite_schema`) — enough to *look*, nothing more. Rung one: `quick_check`
says ok — storage is sound; the successor may *read* and build provisional
plans. Rung two: the application audits pass — foreign keys check out,
artifact hashes match, the FTS row counts agree — the estate's *internal*
claims cohere, and routine work may proceed on them. Rung three, for records
about to bear real weight: spot re-verification against the *world* — the
config row against the actual file it describes, the "service healthy"
outcome against a fresh probe — because storage integrity never promised the
world held still, and rung three is where the estate's `recorded_at` columns
earn their keep by pricing each fact's staleness. The ladder's use is
economic, in the register's sense: climbing costs turns, and the height
required is set by what the session will do — a read-only report trusts at
rung one; a purge keyed on artifact rows climbs to three for exactly the
rows it will act on. What the ladder forbids is only the midden habit:
acting at rung three's stakes on rung zero's evidence because the file
opened and looked plausible.

## Cadence, copies, and the cold rule

Mechanism without schedule is chapter 7's own evidence theater, so the
backup doctrine gets its operational shape. Cadence follows value density:
an estate absorbing a session's work backs up at session end (the handoff's
natural moment — the briefing cites the backup it just verified); estates
under continuous unattended write take a timer (previous volume, chapter 4)
on a period priced by the acceptable loss window. Retention follows the
previous volume's timestamped-graveyard pattern: dated backup files, a
purge schedule, and at least one *drilled* generation always outside the
blast radius of the newest mistake — because the incident that corrupts an
estate at 3 a.m. is also the incident most likely to have corrupted
*tonight's* backup at 2:55. Distribution follows one clarifying rule: cold
copies travel freely, live files never. The naive-copy prohibition binds
only the *live* database; a `VACUUM INTO` product is a closed, complete,
ordinary file, and every file tool the register knows — rsync to another
host, checksum manifests, the artifact index itself (the estate's backups
belong in the estate's *successor's* artifact table, hash and all) —
applies to it without caveat. The 3-2-1 folklore translates directly once
that distinction is held: three copies, two media, one elsewhere — all of
them cold, all of them born from the engine, none of them a cp of a file
something might be writing.

## Sidecar forensics

Because chapter 5's sidecars are where naive tooling does its damage, the
successor needs the reading list for files found beside an inherited
estate. A `-wal` file present at rest: normal — the last tenant exited
without a final checkpoint; the engine will fold it in on open, and the
only wrong move is "cleaning it up" first, which discards committed
transactions. A `-shm` file: coordination scaffolding, meaningless at
rest, recreated on demand — its presence signals nothing. A `-journal`
file: an interrupted rollback-journal transaction awaiting automatic
recovery on open — same rule, sharper stakes: deleting it converts a
recoverable interruption into corruption. The general law covers every
case and fits on one line: **sidecars are opened with the engine, never
interpreted, moved, or deleted by hand** — and its corollary from chapter
5, that a live estate's directory is copied only through the engine,
completes the pair of rules that would, between them, have prevented the
majority of the corruption stories the documentation's post-mortems
collect. The one legitimately hands-on act — archiving a *cold* estate
directory whose tenant is confirmed gone — is the previous volume's
proof-of-target discipline: `fuser` says no holders, then the whole
directory travels together, sidecars included, as one artifact.

## The ladder down

Last, the bad day: verification failed, on the live file. The recovery ladder,
descended in order and recorded in the run's account at every rung. First
rung: stop writing — every write to a corrupt database deepens the hole, so
the estate goes read-only the moment the audit speaks (the ledger's own
account of this event goes, per chapter 1's taxonomy, in a *different* file).
Second: preserve the evidence — copy the damaged file *and its sidecars*
(file-level copy is correct here precisely because the priority has inverted:
bytes, not consistency, are now the asset) before any recovery attempt
touches them. Third: restore from the newest verified backup, measure the gap
— the ledger's last rows in the backup date the loss — and let
intent-then-outcome (chapter 2) direct the re-verification of whatever world
actions fell in it. Fourth, only when backups fail the need: salvage — the
`.recover` command of the sqlite3 shell walks the wreck and emits everything
reconstructible, and `.dump` predates it as the cruder tool; both produce SQL
to rebuild a new file, never repairs in place. And the rung below salvage is
candor: some losses are losses, and the register's covenant — retractions
told, not hidden — applies to estates exactly as to publications. The
successor inherits the account of what was lost with the same provenance
discipline as any other fact, because the alternative — a gap wearing a calm
face — is the one failure this book's whole tradition refuses.

## The estate you did not expect

The trust ladder assumed the estate is *yours* — an inherited file from
your own lineage, suspect only of decay. Operators also meet the other
kind: a database file of unknown provenance, arriving as a download, an
attachment, another team's export. The engine's security documentation is
plain that a database file is an *input* like any other, and a crafted one
is an attack surface: deliberately corrupt structures probing the parser,
and — subtler — schema-borne behavior, because views and triggers execute
when touched, meaning a hostile schema can make an innocent-looking SELECT
do things its reader never wrote. The defensive posture for unknown files
costs three lines and the register's habits. Open read-only (`mode=ro` —
chapter 5's seat, now as armor). Leave `PRAGMA trusted_schema` at its
modern default of off, which refuses the schema-borne tricks the docs
enumerate, and run `PRAGMA integrity_check` plus `quick_check` before any
real query, as triage rather than trust. And read the schema *as text
first* (`sqlite_schema`, chapter 3's stranger query) before running
anything that would evaluate it — the previous volume's read-before-edit,
reincarnated as read-before-query. For the estate's own lineage this
paranoia is unnecessary by construction; the point of stating it is the
boundary: the disciplines that make your own estates trustworthy are
provenance disciplines, and a file without provenance gets the other
protocol, every time, no matter how much its tables look like home.

## The standing verification job

Instruments and cadences assembled, the chapter's doctrine compresses into
one scheduled session — the estate's health check, an unattended operator
like any other, whose own runs land in the registry it audits. Its shape,
as a template to adapt: daily, `quick_check` plus the application audits
(foreign keys, FTS sync counts, open-intent staleness — chapter 8's
briefing queries, run for alerting rather than orientation), each result a
ledger row only when it *finds* something, per the only-writes rule. At
backup cadence, the full sequence this chapter demonstrated: `VACUUM INTO`
a dated file, `integrity_check` on the product, hash into the artifact
index, retention purge of expired generations — one transaction of record
per backup, proof included. Monthly, the restore drill: yesterday's backup
opened, briefed, and spot-queried, with the drill's outcome ledgered
because "restores worked in August" is exactly the kind of claim an
incident in November wants dated. And on every schedule, the meta-check
the register's monitoring chapter taught: the health check's *own*
absence must be loud — a job that verifies everything but whose silence
looks like health is the calm-face failure again, so the briefing's
staleness queries watch the watcher too ("last verification run: when?").
None of this is machinery beyond what the book already built; it is
chapters 4 through 7 composed into a timer, which is the estate's whole
method arriving at its own maintenance.

## When the bytes are fine and the facts are not

One verification failure mode remains that no pragma detects: meaning-rot.
The storage audits clean, the hashes match, and the record is *wrong* —
because the world moved after the row was written. The config row describes
a file someone hand-edited last week; the "service healthy" outcome
predates the migration; the fact about the gate's memory cap was true of a
gate two versions ago. The estate's defenses are the provenance disciplines
laid down in chapter 3, now read as a freshness system. `recorded_at`
prices every fact's age, and the trust ladder's rung three — re-verify
against the world before acting — is *triggered* by that price crossing
the stakes at hand: a day-old fact backs a routine read; a season-old fact
backing a destructive write gets re-proven first, by the register's own
proof-of-target reflexes. `source` makes re-proving cheap: the row that
names its origin (a path, a command, a URL) carries its own re-verification
procedure. And the correction habit closes the loop: a fact found stale is
not UPDATEd into silence but corrected on the record — new row, correction
citing the old, journal entry for the searcher — so the estate's history
shows not only what was believed but when belief was revised, which is what
distinguishes a memory from a cache. Byte integrity the engine guarantees;
fact integrity is a practice, and it is the same practice this press runs
on manuscripts: dated claims, resolvable sources, corrections told.

Corruption itself deserves a closing word of proportion, because the engine's
reputation sometimes takes blame its documentation carefully allocates
elsewhere. SQLite's file format is famously durable; the documented paths to
corruption are dominated by the environment — storage that lies about syncs,
network filesystems with broken locking, *other processes* deleting or
copying sidecar files, backups taken the naive way — and by exotic
misconfiguration, not by the engine's bookkeeping. Which returns the chapter
to its theme with the emphasis correctly placed: the estate's trustworthiness
is mostly the operator's conduct — local disks, sidecars respected,
engine-mediated copies, checks on schedule, drills for real. The engine
holds up its half ruthlessly. The verification suite is how the operator
proves, on schedule, that both halves are still standing — and its verdicts
are precisely what the final chapter's handoff will cite.
