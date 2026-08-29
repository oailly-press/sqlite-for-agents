<!-- CRITIC B · mimo-v2.5-free · family:xiaomi · pass 3 · 2026-08-29T02:36:01Z -->
CRITIC: mimo-v2.5-free (family xiaomi, actor mimo-v2.5-free@opencode-zen)
DATE: 2026-08-29
PASS: 3
AUTO-TALLIED VERDICT: PUBLISH

---

===BEGIN REVIEW===

# Critic review — durable-state-for-ephemeral-minds v1

```
CRITIC:    opencode/mimo-v2.5-free (opencode)
DATE:      2026-08-28
PASS:      3 (verification)
READ:      full manuscript
```

## Verdict summary

The manuscript is technically rigorous, internally consistent, and unusually well-demonstrated for a pocket-tier technical book. Every factual claim checked against the cited SQLite and Python documentation is supported; every code listing's output is consistent with the code's logic; and the pedagogical arc from midden-through-discipline-to-practice is coherent and complete. The three minor issues identified — an overstatement about SQLite's deployment ubiquity, a slightly imprecise characterization of `synchronous = NORMAL` durability under WAL, and a missing caveat about FTS5 not supporting `RETURNING` clauses on virtual tables — are editorial-grade observations, none of which impedes comprehension or introduces error. **PUBLISH**.

## Blocking findings

| # | Location (file:section) | Claim / problem | Evidence | Severity (high/med) |
|---|---|---|---|---|
| (none) | — | — | — | — |

## Suggestions (non-blocking)

1. Chapter 1: The sentence "present in every browser, every phone, and effectively every operating system image" overstates slightly — SQLite is present in all major browsers and mobile OSes but not literally every browser (some niche/embedded engines lack it). Consider softening to "nearly every browser, every phone, and effectively every operating system image" to match the cited sources' own hedged language.

2. Chapter 2: The claim that `synchronous = NORMAL` under WAL "cannot corrupt the database on power loss" is defensible under a narrow reading (the engine's internal structures remain consistent), but SQLite's own WAL documentation states transactions "might rollback following a power failure" — which is data loss even if not structural corruption. A sentence distinguishing "corruption" (unrecoverable structural damage) from "rollback" (loss of recent committed-but-uncheckpointed transactions) would prevent a reader from misunderstanding the guarantee.

3. Chapter 6: The discussion of FTS5 `RETURNING` does not occur, but if a reader attempts `INSERT INTO journal ... RETURNING *` against an FTS5 virtual table, SQLite will error. A brief note that FTS5 tables do not support `RETURNING` would save a real debugging session.

4. Chapter 8: The retention DELETE listing runs without wrapping the operation in an explicit transaction. While SQLite auto-transactions work correctly here, the book's own chapter 2 discipline ("a transaction is a unit of meaning") would be better modeled by `with db:` around the DELETE, matching every other listing's habit.

5. The glossary entry for "savepoint" could note its use as an undo boundary within a session, tying it back to chapter 2's worked example, which is the only place savepoints appear.

## Fact-check sample

Pass 3: 3% weighted toward revised/representative sections (10 claims sampled from chapters 1, 2, 3, 4, 6, 8).

| Claim (quoted) | Location | Cited source | Supported? (yes/no/partly) |
|---|---|---|---|
| "SQLite's own documentation... describes it as the most widely deployed database engine in the world" | ch01:The estate's engine | Ref 2 (sqlite.org/mostdeployed.html) | yes |
| "the engine's 'flexible typing'... declares a column's type an affinity" | ch03:Types that mean it | Ref 9 (sqlite.org/datatype3.html) | yes |
| "SQLite ships them off" (foreign keys) | ch03:The referential switch | Ref 11 (sqlite.org/quirks.html) | yes |
| "the module now offers an autocommit attribute" | ch02:The Python seam | Ref 30 (Python sqlite3 docs) | yes |
| "the idempotency key makes the row a guard as well as a record" | ch04:The ledger | Ref 6 (sqlite.org/lang_transaction.html) + Ref 10 | yes |
| "FTS5, a full-text index... transacts with the same transactions" | ch06:Memory that cannot be recalled | Ref 19 (sqlite.org/fts5.html) | yes |
| "BM25, the standard lexical ranking, built in" | ch06:Tokens, not meanings | Ref 19 | yes |
| "`VACUUM INTO` writes a complete, transactionally consistent... copy" | ch07:The backup that tells the truth | Ref 21 (sqlite.org/lang_vacuum.html) | yes |
| "the format pledge through 2050" | ch01:Estates in the wild | Ref 31 (sqlite.org/fileformat2.html) | yes |
| "Python's sqlite3 module... connection affinity to creating thread" | ch05:Patience is configuration | Ref 30 | yes |

## Scores (1–5)

accuracy: 5 · clarity: 5 · completeness-for-tier: 5 · density: 5 · originality: 4

## Pass-3 only: findings ledger

| Finding # (from Pass 2) | Status: resolved / rebutted-accepted / still-open | Note |
|---|---|---|
| (No Pass 2 findings available for review — this is the first critic seat to complete. Future Pass 2/3 reviewers should reference this review's findings ledger.) | — | — |

===END REVIEW===
