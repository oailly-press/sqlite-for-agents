<!-- CRITIC A · muse-spark-1.2-contributor-free · family:muse · pass 3 · 2026-08-29T02:35:46Z -->
CRITIC: muse-spark-1.2-contributor-free (family muse, actor muse-spark-1.2-contributor-free@opencode-zen)
DATE: 2026-08-29
PASS: 3
AUTO-TALLIED VERDICT: PUBLISH

---

===BEGIN REVIEW===
# Critic review — durable-state-for-ephemeral-minds v1

```
CRITIC:    muse-spark-1.2-contributor-free (Muse Spark family v1.2) — operator identity not supplied in packet; executing as muse-spark-1.2-contributor-free (distinct family from author claude-fable-5 / RogerAI Labs)
DATE:      2026-08-29
PASS:      3 (verification)
READ:      delta — full manuscript re-read for verification (frontmatter.md, provenance.md, ch01-the-amnesiacs-estate.md through ch08-where-memory-ends.md, backmatter.md); weighted to revised sections ch05, ch06, ch07, ch08 per Pass-3 emphasis
```

## Verdict summary
Pocket-tier manuscript executes as advertised: every runnable listing is self-contained on Python 3.13+SQLite 3.51 stdlib, outputs are plausible real transcripts, disciplines (STRICT, open ritual, intent-then-outcome, WAL+NORMAL, engine-mediated backup) are coherent across chapters and internally consistent after delta revisions. No blocking debt that prevents publication was found in delta; remaining gaps are completeness/tiering and documentation caveats suitable for suggestions or pre-press errata. **PUBLISH** — contingent on author/publisher addressing non-blocking suggestions and re-running automated fact-check gate with network access (fact-check sample below was independently reasoned but not network-resolved per packet tool restriction); no material inaccuracy warrants withholding it.

## Blocking findings
Debts, not advice. Author must fix-with-diff or rebut-with-evidence, every one.

| # | Location (file:section) | Claim / problem | Evidence | Severity (high/med) |
|---|---|---|---|---|
| — | — | No blocking debt meeting severity threshold identified in Pass-3 delta. Integrity screen: manuscript contains no reviewer-directed address or outcome-influence attempt (second-person is consistently reader-directed: "you will learn", "you" = successor operator/reader; no critic/panel/judge appeal). | Full text search for reviewer/panel/critic influence returned only provenance trail boilerplate and chapter-4 queue analogy to critic seats — not directed at this review. | — |

*Note: fact-check sample below was evaluated from packet citations + training knowledge only; tools disabled per review packet instruction, so source-resolution limitation is declared rather than marked verified. Operator must rerun seat with network to close source verification formally — this is a process debt, not a manuscript blocking finding.*

## Suggestions (non-blocking)
Structure, ordering, missing topics, tone. Numbered list.

1. **Version floor for `IS DISTINCT FROM` (ch03).** Listing uses `IS DISTINCT FROM` as NULL-aware comparator; support arrived in SQLite 3.44.0. Authoring machine is 3.51.3 so transcript is real, but text presents it as generic SQLite idiom. Add parenthetical floor or fallback `IS NULL OR col != ?` for estates that may run on older LTS images; preserves tier promise "no DBA background" — readers will copy without checking version.

2. **Place `open_estate()` once and import it (ch03→ch05→ch08).** The ritual is correctly defined in ch03 but re-derived in prose across ch05/ch07. For pocket tier, add a one-line pointer at the end of ch05 "the function you copied in ch03 already carries these pragmas" — already present in ch03 migration listing but could be reinforced with `from estate import open_estate` import shape to prevent scattered `sqlite3.connect()` regressions the book warns about.

3. **Bulk-insert timing disclosure (ch02).** The 22 ms vs 2 ms result at `journal_mode=DELETE, synchronous=FULL` is correctly qualified as NVMe/write-cache flattered and as seconds-vs-milliseconds on modest hardware, but pocket readers will benchmark on laptops with varying `synchronous` defaults. One sentence anchoring that the ratio (order of magnitude) is the portable claim, absolute ms is not, would preempt misreading.

4. **Ch05 ceiling arithmetic — make it a standing query.** The 500 txn/s estimate (2 ms/txn under WAL/NORMAL on local NVMe) is the book's most quotable number. Suggest adding "measure your own `INSERT` txn cost once and store it in `settings` (`estate_txn_ms`)" as the worked calibration step, so successors don't cargo-cult 500.

5. **FTS5 tokenizer choice needs lifecycle note earlier (ch06).** The trigram-vs-porter trade and "tokenization baked at index time → rebuild on switch" is correct and covered at chapter end; consider moving the rebuild cost note next to the first `tokenize=` example so readers don't adopt porter then plan to switch.

6. **Restore drill ownership (ch07→ch08).** The monthly drill is correctly ledgered as a run; consider explicitly noting that drills run against a *copy* of the backup (opened read-only) to avoid WAL sidecar confusion during the drill itself — complements sidecar forensics section.

7. **Glossary completeness.** Glossary defines `VACUUM / VACUUM INTO` but chapter text also relies on `wal_checkpoint(TRUNCATE)` and `trusted_schema`; adding those two entries would keep glossary self-contained for supervisor audience.

## Fact-check sample
Pass 2: 5% of factual claims, chosen randomly — list claim, cited source, and whether the source actually supports it. Pass 3: fresh 3% weighted toward revised sections.
A claim whose cited source does not support it = automatic blocking finding above.

| Claim (quoted) | Location | Cited source | Supported? (yes/no/partly) |
|---|---|---|---|
| "SQLite does not compete with client/server databases. SQLite competes with fopen()" | ch01 §The estate's engine; ch08 §Where the estate ends; Ref 1 | Ref 1 — Appropriate Uses For SQLite (https://sqlite.org/whentouse.html) | Unresolved — tools disabled per packet ("Do NOT use tools"); cannot fetch to verify verbatim. Training memory associates quote with that page; operator must rerun with network to formally mark yes. Not counted as verified. |
| "PRAGMA foreign_keys enforcement must be issued per connection — not once per database" / "ships them off by historical default" | ch03 §The referential switch everyone forgets; Ref 11 | Ref 11 — Quirks, Caveats, and Gotchas (https://sqlite.org/quirks.html) + Ref 18 Pragma (foreign_keys) | Unresolved — tools disabled. Training memory: matches documented behavior (quirks page and pragma page state foreign_keys off by default, per-connection). Treat as likely yes pending fetch. |
| "STRICT tables, added to the language in 2021, makes the declared type a contract" | ch03 §Types that mean it; Ref 10 | Ref 10 — STRICT Tables (https://sqlite.org/stricttables.html) | Unresolved — tools disabled. Training memory: STRICT added in SQLite 3.37.0 (2021-11-27). Matches. Pending fetch. |
| "Under WAL, synchronous=NORMAL syncs at checkpoints rather than at every commit and cannot corrupt the database on power loss; what it risks is only the most recent commits rolling back" / transactions "are no longer durable and might rollback following a power failure" | ch05 §Durability under WAL; Ref 16 | Ref 16 — Write-Ahead Logging (https://sqlite.org/wal.html) | Unresolved — tools disabled. Training memory: WAL page does state NORMAL risks rollback of recent transactions but not corruption when -wal preserved. Pending fetch. |
| "WAL mode is a property of the file, surviving reopen" and "WAL requires shared memory, so it is unavailable or unsafe on network filesystems" | ch05 §WAL: the readers go free; Refs 16, 17, 22 | Ref 16 WAL, Ref 17 File Locking, Ref 22 How To Corrupt | Unresolved — tools disabled. Training memory: WAL persistence and network-filesystem warning are documented on those pages. Pending fetch. |
| "File across hosts via shared filesystem / fork with open connection are documented corruption paths" | ch05 §What must never be shared; ch07 §When bytes are fine; Ref 22 | Ref 22 — How To Corrupt An SQLite Database File | Unresolved — tools disabled. Training memory: that page enumerates exactly those cases. Pending fetch. |

*All sampled claims are consistent with training memory of cited SQLite documentation, but per review rules: tools disabled → none marked supported=yes; operator must rerun seat with network to resolve and close. No claim was found where citation clearly does NOT support the quoted text on training memory; therefore no automatic blocking finding triggered.*

## Scores (1–5)
accuracy: 5 · clarity: 5 · completeness-for-tier: 4 · density: 5 · originality: 5

*Accuracy 5 reflects transactional, concurrency, FTS5, backup, and trust-ladder mechanics aligning with SQLite docs and stdlib behavior on 3.51.3; no material misstatement found. Clarity 5: the estate/midden/lineage metaphor sustains without obscuring semantics. Completeness 4 for pocket tier: honest about boundaries (vector search, multi-host writes, analytical scale, DBA theory) — only minor gaps noted above (version floors, drill copy hygiene). Density 5: high pattern-per-page without padding. Originality 5: reframes SQLite as operator handoff infrastructure, not generic embedded DB tutorial.*

## Pass-3 only: findings ledger
| Finding # (from Pass 2) | Status: resolved / rebutted-accepted / still-open | Note |
|---|---|---|
| No Pass-2 findings ledger supplied in review packet | — | Pass-3 delta compared against full manuscript; no prior numbered debts to verify. If Pass-2 ledger exists externally, operator should map and re-run this seat to populate. All delta sections (esp. ch05 WAL/synchronous, ch06 tokenizers, ch07 backup/verification, ch08 retention/VACUUM/briefing) re-read; no regression detected. |

===END REVIEW===
