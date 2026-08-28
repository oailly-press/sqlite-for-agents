# Chapter 6 — Search Is Recall

*Draft status: author draft, gate-checked; human verification pending. Outputs are
real transcripts; the journal entries indexed below are drawn from this book's own
production history.*

## Memory that cannot be recalled is storage

The estate so far remembers perfectly and recalls narrowly. Every query in
chapters 4 and 5 addressed rows by their structure — by key, by status, by
date — which serves the questions an operator knows it will ask. But a working
memory accumulates a second kind of holding: prose. Findings written in
sentences, incident notes, decisions with their reasoning, excerpts from
documents that mattered once and might again. Structure cannot address these,
because their content *is* their address: the future question will be "what do
I know about power capping?", asked in words, answerable only by matching
words against words. A memory that cannot answer that question does not really
hold its prose; it stores it, the way the midden stored everything —
present, and unreachable.

The register's operators feel this gap acutely because their native recall is
so poor. A human admin half-remembers last month's incident and greps her
shell history; a session-bound operator has no half-memories to steer by. For
it, recall *is* search — whatever the estate's search can surface is,
functionally, everything the operator has ever known. That makes the quality
of the estate's text search a first-order property of the operator itself,
and it makes the right tool worth learning properly. SQLite ships that tool:
FTS5, a full-text index that lives in the same file, transacts with the same
transactions, and needs nothing installed. This chapter builds the operator's
journal on it, then draws the tool's honest boundaries — because "full-text
search" sits near enough to fashionable retrieval technology that confusing
them costs estates real design mistakes in both directions.

## The searchable journal

The pattern chapter 4 might have called the sixth shape: a journal of prose
entries, indexed for content. FTS5 tables are declared virtual — the engine
maintains the index structures behind an interface that reads like a table:

```python
import sqlite3
db = sqlite3.connect("estate.db")
db.execute("""CREATE VIRTUAL TABLE journal USING fts5(entry, written_at UNINDEXED)""")
notes = [
    ("gpu-power-cap.service fails at boot; exit status 2; journal unreadable from this seat", "2026-08-24"),
    ("rotated backup credentials; restore drill passed; old key revoked", "2026-08-25"),
    ("nginx upgraded; config drop-in preserved; is-active confirmed after restart", "2026-08-26"),
    ("disk pressure on /mnt/train at 98 percent; quarantined stale checkpoints", "2026-08-27"),
    ("power cap applied to RTX PRO 6000 after PSU transient trip; verified with vendor tool", "2026-08-28"),
]
with db:
    db.executemany("INSERT INTO journal VALUES (?, ?)", notes)
print("query: power NEAR cap")
for row in db.execute("""SELECT written_at, highlight(journal, 0, '[', ']')
                         FROM journal WHERE journal MATCH 'NEAR(power cap)'
                         ORDER BY rank"""):
    print(" ", row[0], "|", row[1])
print("query: credential*")
for row in db.execute("SELECT written_at, entry FROM journal WHERE journal MATCH 'credential*'"):
    print(" ", row[0], "|", row[1][:60])
```

```output
query: power NEAR cap
  2026-08-24 | gpu-[power]-[cap].service fails at boot; exit status 2; journal unreadable from this seat
  2026-08-28 | [power] [cap] applied to RTX PRO 6000 after PSU transient trip; verified with vendor tool
query: credential*
  2026-08-25 | rotated backup credentials; restore drill passed; old key re
```

The five entries are this book's own production history — the failed unit from
the previous volume's postmortem, the disk pressure this machine really
carried — and the queries against them exercise the toolkit an operator's
recall actually needs. `MATCH` takes a query language, not a substring:
`NEAR(power cap)` found the incident whether the words appeared hyphenated in
a unit name or spaced in prose, and would have found them sentences apart.
`credential*` is prefix search — the recall question rarely knows the exact
inflection it stored. `ORDER BY rank` sorts by relevance (BM25, the standard
lexical ranking, built in), which begins to matter the day the journal holds
five thousand entries instead of five. And `highlight()` returns the entry
*with the matches marked* — for this book's reader the killer feature, because
an operator budgeting transcript volume (register rule, chapter 1 of the
previous volume) wants evidence of *why* a result matched without re-reading
it whole; the companion `snippet()` function goes further and excerpts just
the matching neighborhood. The one schema note: `written_at UNINDEXED` stores
the date alongside without polluting the text index — metadata rides along,
content gets searched.

## Tokens, not meanings

What the index actually holds is tokens — words, as a tokenizer defines words
— and the operator who knows this predicts every search behavior from first
principles. The definition is configurable, and one configuration choice
illustrates the whole layer:

```python
import sqlite3
db = sqlite3.connect("estate.db")
db.execute("CREATE VIRTUAL TABLE plain USING fts5(entry)")
db.execute("CREATE VIRTUAL TABLE stemmed USING fts5(entry, tokenize = 'porter unicode61')")
for t in ("plain", "stemmed"):
    db.execute(f"INSERT INTO {t} VALUES ('deployed the staging build and verified the endpoints')")
for t in ("plain", "stemmed"):
    hits = db.execute(f"SELECT count(*) FROM {t} WHERE {t} MATCH 'deploy'").fetchone()[0]
    print(f"{t:8s} matches for 'deploy':", hits)
```

```output
plain    matches for 'deploy': 0
stemmed  matches for 'deploy': 1
```

The default tokenizer stores `deployed` as `deployed`, and the query `deploy`
misses it — surprising until the token model is explicit, obvious after. The
porter option stems English words to their roots at both index and query time,
so `deploy`, `deployed`, and `deploying` converge; the cost is the occasional
false collision and a mild English-centrism, which for operator journals is
usually the right trade. The general lesson outranks the specific knob:
**FTS matches token identity, nothing else.** It does not know that "rotated
credentials" and "changed the password" are the same event; no tokenizer
bridges vocabulary. The operator's countermeasure is a writing discipline, not
a search feature — journal entries name their subjects in stable terms (unit
names, paths, error strings verbatim: the register's exact-transcript rule
paying a second dividend), because the entry is written once and queried by a
stranger who can only guess words. Entries written for the searcher are the
estate's equivalent of the previous volume's labels-on-state.

This is also where the fashionable comparison belongs, stated plainly per
this book's boundaries. Embedding-based retrieval — vectors, semantic
similarity, the machinery behind modern RAG — solves the vocabulary problem
FTS cannot: it would land "changed the password" for the credentials query.
It pays in machinery (a model, its runtime, an index, all versioned and
maintained), in opacity (a match has no `highlight()` — *why* this result?),
and in exactness (the query `exit status 2` should match exit status 2, not
things shaped like it). Operator estates skew hard toward the exact: unit
names, error strings, hosts, keys. The honest architecture, when both needs
are real, is FTS as the estate's native recall with semantic search added
*beside* it, as its own indexed artifact — not a replacement, and never a
reason to skip the lexical index that costs nothing and explains itself. A
`LIKE '%pattern%'` scan, finally, keeps its small place: for a rare query
over a small table it is fine, and chapter 3's `EXPLAIN QUERY PLAN` will say
`SCAN` and remind you what it costs at scale.

## Writing for the searcher

The index amplifies whatever discipline the entries carry, which makes journal
*writing* half of recall quality, and the half entirely under the operator's
control. The disciplines are few and specific. Entries are written at outcome
time, not planning time — the journal records what happened and what it meant,
so a session's entry lands beside chapter 4's ledger completion, one truth in
two registers (the row for machines that query structure, the sentence for
searchers that query words). Entries name their subjects in the terms a
stranger will guess: unit names verbatim, paths absolute, error strings
pasted exactly — the previous volume's exact-transcript rule, now justified a
third way, since every paraphrase is a query that will someday miss. Entries
state outcomes and reasons, not just events — "quarantined stale checkpoints
*because* /mnt/train hit 98 percent" is findable from either end of the
causation, and the *because* is what the future searcher is usually really
hunting. And entries are atomic per subject: one entry about the disk
pressure, another about the nginx upgrade, because the searcher retrieving
one should not pay transcript volume for the other — the register's bounding
economics, applied to memory retrieval.

The anti-patterns mirror them. The diary entry ("busy session, lots of
firefighting, mostly done") indexes nothing a query will ever ask. The dump
entry — three hundred lines of pasted transcript — makes its keywords
findable and its retrieval cost absurd; the right decomposition stores the
transcript as a chapter-4 artifact and journals the three-sentence account
with the artifact's path, so search finds the summary and the summary points
at the evidence. And the secret-bearing entry violates chapter 4's exclusion
in the one table designed to be read broadly; the reference-not-value rule
binds hardest exactly here.

## Query craft: words and structure together

Recall questions in practice are rarely pure text — they are text *within
bounds*: what do I know about cert renewals, *from June*? The estate answers
hybrid questions in one statement, because the FTS table's indexed text and
its UNINDEXED metadata live in the same rows:

```python
import sqlite3
db = sqlite3.connect("estate.db")
db.execute("CREATE VIRTUAL TABLE journal USING fts5(entry, written_at UNINDEXED)")
rows = [("cert renewal failed: acme challenge timeout", "2026-06-02"),
        ("cert renewal succeeded after dns fix", "2026-06-03"),
        ("cert renewal succeeded", "2026-08-01")]
with db: db.executemany("INSERT INTO journal VALUES (?,?)", rows)
q = """SELECT written_at, snippet(journal, 0, '[', ']', '…', 8)
       FROM journal WHERE journal MATCH 'cert AND renewal'
         AND written_at >= '2026-06-01' AND written_at < '2026-07-01'
       ORDER BY written_at"""
for r in db.execute(q): print(r[0], "|", r[1])
```

```output
2026-06-02 | [cert] [renewal] failed: acme challenge timeout
2026-06-03 | [cert] [renewal] succeeded after dns fix
```

August's entry matched the words and fell to the date bound; June's two
arrived excerpted by `snippet()` with the matches marked. The MATCH clause
meanwhile carries its own small language, worth the ten minutes its
documentation costs: implicit AND between terms, explicit `OR` and `NOT`,
quoted phrases for exact sequences (`"exit status 2"` — invaluable for error
strings), `NEAR()` from the first listing for proximity without adjacency,
`^term` for entries *beginning* with a term, and column filters
(`title:deploy`) once journals grow structured fields. Two habits complete
the craft. Chapter 3's date conventions are what make the hybrid WHERE
clause work at all — ISO text comparing correctly is the payoff arriving —
and bounded output applies to recall exactly as to every read the register
ever taught: `ORDER BY rank LIMIT 10`, because a memory that answers with
everything it has is the unbounded journalctl of the previous volume,
reborn indoors.

Identifiers deserve a tokenizer note of their own, because operator prose
is full of them and word-shaped tokenization serves them poorly:
`gpu-power-cap.service` splits on its punctuation into pieces a searcher
must guess, and a path or commit hash is one opaque "word" findable only
whole. FTS5's trigram tokenizer answers the identifier case by indexing
overlapping three-character windows, buying indexed *substring* search —
`cap.serv` finds the unit, half a hash finds the commit — at the price of
a fatter index and no notion of words at all. The estate's arrangement,
when identifier recall matters enough: the journal keeps its word index
for prose, and the identifier-dense columns (ledger actions, artifact
paths) get a small trigram-indexed shadow — each index shaped to what it
searches, unioned at query time by the front-door view below. The general
tokenizer lesson closes where the section opened: tokenization is a
*declaration about the text's nature*, chosen per column at index time,
and the estate that declares it deliberately — words here, stems there,
trigrams for the machine-named — searches the way its content deserves
rather than the way the default guesses.

And recall's audience, like the estate's, includes the supervisor. The
journal a session keeps for its successors is, unchanged, the narrative an
incident review reads months later — ranked, dated, cause-carrying
sentences, each written when the knowledge was fresh, retrievable by the
reviewer's own words. The previous volume taught operators to report
plainly because reports are load-bearing; the journal extends the
principle across time: it is the report that never stopped being
queryable. Institutions pay technical writers for worse.

## The recall budget

Retrieval has the same economics as every read the register prices: results
pulled into a session's working context cost attention, and recall that
floods is recall that gets skipped next time. The budget discipline has
three dials. `LIMIT`, always — the ritual search opens with the top five by
rank, and widens only on a miss, the previous volume's cheap-aggregate-then-
drill rhythm applied to memory. `snippet()` over full entries at the survey
stage — excerpts with matches marked, at a tenth the volume, with the full
entry fetched only for the one or two results that survive triage. And
ranking *tuned to the estate's shape* where it earns it: BM25 accepts
per-column weights, so a journal that grows a `title` column can weight it
above the body (`bm25(journal, 10.0, 1.0)`), making the operator's own
one-line summaries the strongest signal — which quietly rewards exactly the
entry discipline the writing section asked for. The composed form of all
three dials is worth keeping as the estate's canonical recall query: top
five by weighted rank, snippets only, hybrid-filtered by any structural
bounds the task supplies. One query, bounded, explained — recall as a
disciplined shot rather than a rummage, which is what distinguishes an
operator consulting its memory from an operator lost in it.

## Searching everything at once

Estates accumulate more than one searchable surface — the journal here, the
ledger's action strings, the facts table's prose — and the recall ritual
should not require remembering which drawer holds what. The estate's answer
is a union view: each searchable table gets its FTS index, and one view
stitches their results into a single ranked stream tagged by origin
(`SELECT 'journal' AS kind, rank, snippet(...) FROM journal WHERE journal
MATCH :q UNION ALL SELECT 'ledger', ...`), so the operator's one query
sweeps the whole estate and the results say where each hit lives. The
pattern's discipline is to keep it *shallow*: the view unions indexes, it
does not try to merge scores across them into false precision (BM25 ranks
are comparable within an index, not between), so the composed query
interleaves by kind deliberately — top three journal hits, top three
ledger hits — rather than pretending one global ranking exists. Chapter
4's standing-questions rule then applies to recall itself: the union view
is written down with the schema, tested at adoption, and becomes the
estate's front door for every "what do I know about X" a successor will
ever ask.

## Keeping the index honest

One integration decision remains: the journal above *is* an FTS table — the
text lives in the index's own storage. That is the simplest correct
arrangement and the right default for a journal. When the text already lives
in a regular table (chapter 4's ledger actions, the facts table), FTS5's
external-content mode indexes it in place without duplicating storage — at
the price of a covenant: the index only learns what it is told, so the base
table and index must change together, conventionally via three small triggers
(insert, update, delete) the FTS5 documentation supplies verbatim. The estate
treats those triggers like schema (chapter 3: born in a migration, explained
by comments), and treats the covenant with chapter-7 suspicion: a
verification query — count of base rows vs count of indexed rows, plus a
spot-check MATCH for a recently inserted row — belongs in the estate's trust
suite, because an index that silently stopped syncing is recall quietly going
blind, which for this book's reader means *memory* quietly going blind.
The rebuild command (`INSERT INTO idx(idx) VALUES ('rebuild')`) is the
recovery ladder's one rung, cheap and total.

## Recall in the loop

A recall instrument earns its keep only if consulted, and session-bound
operators need the consultation *scheduled*, because the reflex humans call
"this feels familiar" is exactly what they lack. The discipline is a ritual
search at task start: before acting on any named subject — a unit, a host, a
procedure — the operator queries the journal for it, bounded and ranked, and
reads what its predecessors knew. The estate briefing of chapter 8 opens the
session; the recall query opens the *task*. What it changes is easiest to see
in the incident this book keeps returning to. The previous volume's operator
diagnosed a failed GPU power-cap unit from scratch, spending turns
establishing that the unit fails at boot, that exit 2 was the vendor tool's
"no device", that the journal was unreadable from its seat. Its successor,
facing the same unit after the next kernel upgrade, opens with `MATCH
'gpu-power-cap'` — and inherits the whole prior investigation in three ranked
entries: root cause candidate, the fix that worked, the verification that
proved it. The second diagnosis starts where the first one ended, which is
the entire economic argument for the journal in one example: every searched
session converts some past session's spent turns into this session's free
context. An operator that searches before acting compounds; one that does not
pays for the same knowledge repeatedly, which is the amnesiac's tax this book
exists to end.

The ritual has a write-side twin: promotion. Not everything a session
learns deserves the journal — transcripts are artifacts, scratch reasoning
is scratch — but any fact that *cost real turns to establish* and could
recur gets promoted to a journal entry at outcome time, written by the
searcher's rules above. The test is the compounding one: *would the
successor's ritual search want to find this?* Diagnoses, fixes-that-worked,
dead ends that looked promising (the previous volume's honest-failure
reporting, feeding recall), environmental facts expensively verified. The
promotion moment is the session's close, alongside the ledger completion
and the handoff — memory's last act before ending, and the first thing its
successor will thank it for.

One long-horizon honesty note completes the craft: vocabulary drifts.
The unit renamed in a refactor, the host re-addressed, the procedure's
informal name changed by a new supervisor — entries indexed under the old
terms quietly fall out of recall for operators searching the new ones. The
estate's countermeasures are modest and adequate: prefer stable identifiers
(paths, unit names) over informal descriptions in entries; when a rename
happens, journal the rename itself ("X is now Y") so either term's search
surfaces the bridge; and let the correction-citation habit carry the rest.
Recall systems decay by default; a memory meant for years gets tended like
one.

## What recall cannot do

The chapter closes on its instrument's honest edges, in the tradition both
volumes share. Recall retrieves what was *recorded*: a search that returns
nothing proves only that no entry matched, never that nothing happened — the
register's empty-output ambiguity, now at the scale of institutional memory,
and the reason chapter 4's structured tables carry the load-bearing history
(the ledger's completeness is a discipline with a CHECK; the journal's is a
courtesy). The searcher therefore treats journal silence as a prompt to
consult structure — the ledger by subject, the registry by date — before
concluding novelty; "the journal has nothing on X" and "X never happened"
are different sentences, and only middens confuse them. Symmetrically, what
recall returns is *testimony*, not ground truth: an entry is what some past
session believed at outcome time, aging from the moment it was written, and
chapter 7's staleness pricing applies to retrieved memories exactly as to
queried facts — the searcher checks `written_at` before betting anything
expensive on a reminiscence. None of this diminishes the instrument; it
locates it. The journal makes the estate's experience *findable*; the
structured tables make its claims *checkable*; the trust disciplines make
both *weighable*. An operator using all three in their places has something
neither databases nor diaries provide alone — a memory that can answer, and
can say how sure it is.

The index itself, finally, costs what it looks like it costs: roughly the
text again in storage (each token posted to its list), maintained
incrementally on every write, imperceptible at journal scales. Two
maintenance verbs cover its lifetime: `optimize` (an INSERT-command idiom
the FTS5 documentation specifies) consolidates the index's internal
segments after heavy write bursts, and `rebuild` — chapter's earlier rung —
remakes it wholesale from content, the recovery hammer that also serves
after tokenizer changes, since tokenization choices are baked in at index
time and a switch to porter mid-life reindexes or lies. Both are estate
maintenance acts like any other: scheduled, ledgered when they find work,
invisible otherwise.

Recall, then: prose holdings indexed in the same file, under the same
transactions; queries in words with ranking, prefixes, proximity, and marked
evidence; a token model understood rather than guessed; semantic tools
placed beside, not instead; and the index's honesty audited like everything
else the estate claims. The operator that keeps this chapter's journal owns
something the midden never offered and even most human admins never build —
a searchable account of everything it has ever known. What it does not yet
own is *confidence* in the file that holds all of this. Confidence is
manufactured, on schedule, by verification — and that is chapter 7.
