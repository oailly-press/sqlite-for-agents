<!-- CRITIC C · hy3-free · family:tencent · pass 3 · 2026-08-29T02:36:11Z -->
CRITIC: hy3-free (family tencent, actor hy3-free@opencode-zen)
DATE: 2026-08-29
PASS: 3
AUTO-TALLIED VERDICT: DONT-PUBLISH

---

===BEGIN REVIEW===
# Critic review — durable-state-for-ephemeral-minds v1

```
CRITIC:    hy3-free · opencode · operator: RogerAI Labs (seat run by Luis)
DATE:      2026-08-28
PASS:      3 (verification)
READ:      full manuscript (frontmatter, provenance, ch01–ch08, backmatter)
```

## Verdict summary
This is a technically strong, unusually disciplined pocket-tier manuscript: its
central claims (transactional atomicity, single-writer concurrency, WAL
isolation, FTS5 recall, verification/backup discipline) are correct as far as
they can be assessed from the packet, and the code listings are internally
consistent and free of the errors that usually discredit such books. However,
this is a **DON'T PUBLISH** at Pass 3 for three concrete, author-acknowledged
reasons rather than for quality: (1) the provenance page itself declares the
draft has *not* been human-verified and "ships nowhere until it has been,"
which is an author-stated ship-block that no critic verdict can override; (2)
the operator's own rules require the fact-check sample to be *independently
resolved*, and this seat was instructed not to use tools, so the 32 cited
sqlite.org/Python sources and the load-bearing version/output claims cannot be
verified here — the sample is therefore NOT verified and the seat must be
rerun with tool access; (3) the exact SQLite build version (3.51.3) underpinning
every "real transcript" claim is forward-dated and unconfirmable from this seat.
Resolve those three and the book is publishable; the writing and engineering are
ready.

## Blocking findings
Debts, not advice. Author must fix-with-diff or rebut-with-evidence, every one.

| # | Location (file:section) | Claim / problem | Evidence | Severity (high/med) |
|---|---|---|---|---|
| 1 | provenance.md (VERIFIED BY) | Draft ships nowhere until human-verified, but no human verification is evidenced; a Pass-3 PUBLISH cannot coexist with an author-declared ship-block. | Provenance: "human verification NOT yet performed. Nothing in this draft has been human-verified, and it ships nowhere until it has been." | high |
| 2 | provenance.md / backmatter "measured outputs" / ch02 "price of a promise" | Exact SQLite build "3.51.3" / "3.51 via Python 3.13" is forward-dated to 2026 and unconfirmable from this seat; every "real transcript" claim depends on it being the true gate version. | No tool access granted; version not independently resolvable. If gate truly ran 3.51.3 the claim stands, else every transcript's environment line is wrong. | med |
| 3 | (process) Fact-check sample | 32 cited sources + specific quotes (WAL/synchronous wording, "2050 pledge", "most deployed", FAQ insertion-speed) could not be independently resolved because the seat was barred from tools; per operator rules the sample is NOT verified. | Operator instruction: "If your tools cannot access them, state the limitation and do not call the sample verified; the operator must rerun the seat." | med |

## Suggestions (non-blocking)
Structure, ordering, missing topics, tone. Numbered list.
1. ch03 `open_estate()` sets `journal_mode = WAL` and `synchronous = NORMAL` (with a "(chapter 5)" comment) before ch05 justifies either; a reader who copies ch03 early runs below-FULL durability without the ch05 caveat in hand. Add a one-line inline warning at the pragma, not just a forward reference.
2. ch05's IMMEDIATE/timeout discussion would benefit from noting that Python's `with db:` issues a *deferred* BEGIN under legacy isolation, which is exactly why `BEGIN IMMEDIATE` must be explicit — the thread/connection note is there but the BEGIN-default is not.
3. The `no-run` and `fragment` markings are explained once (frontmatter) but a reader meeting the ch03 NULL `no-run` listing or the rebuild `fragment` mid-chapter has no reminder; a marginal tag would help.
4. ch06 semantic-vs-FTS boundary is well drawn; consider one explicit "when NOT to use FTS" one-liner in the glossary or ch06 opener for tier-appropriate scannability.
5. Minor: ch08 retention demo's `taken_at < '2026-08'` relies on ISO-text lexicographic ordering — correct, but a one-sentence note that this is *why* the ch03 date convention matters would close the loop for skimmers.

## Fact-check sample
Pass 3: fresh sample weighted toward revised/concurrency sections. Per seat
constraint, sources were NOT independently resolvable (no tool access); each is
marked unverified-with-assessment. A claim whose cited source does not support
it would be an automatic blocking finding — none are confirmed unsupported, but
none are confirmed supported either from this seat.

| Claim (quoted) | Location | Cited source | Supported? (yes/no/partly) |
|---|---|---|---|
| "the most widely deployed database engine in the world, present in every browser, every phone" | ch01 "The estate's engine" | Ref 2 (mostdeployed.html) | partly — plausible/asserted by sqlite.org; unverified (tool restriction) |
| "the STRICT table option, added to the language in 2021" | ch03 "Types that mean it" | Ref 10 (stricttables.html) | yes (training: SQLite 3.37.0, 2021-11, introduced STRICT) — unverified via source |
| "SQLite ships them [foreign keys] off" / "every new connection starts with it off" | ch03 "The referential switch" | Ref 11 (quirks.html) | yes (training: FK enforcement off by default) — unverified via source |
| "transactions 'are no longer durable and might rollback following a power failure or hard reset'" | ch05 "Durability under WAL" | Ref 16 (wal.html) | partly — conceptual claim (NORMAL under WAL: no corruption, tail may roll back) is correct per SQLite docs; exact quoted wording unverified |
| "the project pledges support through the year 2050" | ch01 "Estates in the wild" | Ref 31 (fileformat2.html) | yes (training: SQLite format stability pledge through 2050) — unverified via source |
| "Python's sqlite3 binds each connection to its creating thread by default" | ch05 "Patience is configuration" | Ref 30 (python sqlite3 docs) | yes (training: check_same_thread=True default) — unverified via source |

## Scores (1–5)
accuracy: 4 · clarity: 5 · completeness-for-tier: 5 · density: 4 · originality: 4

## Pass-3 only: findings ledger
No Pass-2 findings were supplied in the review packet, so delta verification
against a prior pass is impossible from this seat. All findings above are raised
fresh at Pass 3 as a full review; the operator should attach the Pass-2 ledger
and rerun this seat (with tool access) to confirm the three blocking items are
closed before any publish decision.

| Finding # (from Pass 2) | Status: resolved / rebutted-accepted / still-open | Note |
|---|---|---|
| (none provided) | N/A | Pass-2 findings not included in packet; delta verification could not be performed. |
===END REVIEW===
