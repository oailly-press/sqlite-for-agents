# Durable State for Ephemeral Minds

## SQLite as the memory of machine operators

**O'AILLY Systems & Craft · REV 1.0 (draft)**

## Contents

- Chapter 1 — The Amnesiac's Estate
- Chapter 2 — One File, Whole Truths
- Chapter 3 — Schema Is the Handoff
- Chapter 4 — The Ledger Pattern and Friends
- Chapter 5 — Two Operators, One File
- Chapter 6 — Search Is Recall
- Chapter 7 — Trust, Verify, Repair
- Chapter 8 — Where Memory Ends

## Introduction

This book is for the developer building agents, unattended jobs, or self-hosted
automations that must remember things between runs — and, in second person where
it earns it, for the session-bound operator itself: the cron job, the CI step,
the language-model agent that wakes with no memory, works, and ends. It assumes
you can read basic SQL and hold your own in a shell; it assumes no database
administration background and no machine-learning background. Its claim is
narrow and demonstrated: SQLite, used with the disciplines this book teaches, is
the correct container for the records of operators whose sessions end — and the
ad-hoc file state it replaces fails in specific, reproducible ways that the book
reproduces live rather than asserts. Every listing runs on the standard library
alone; every printed output is a real transcript of the author's execution.
Listings carry one of three markings: plain runnable listings are re-executed by
the publisher's acceptance gate before publication; listings marked `no-run`
were executed by the author but sit outside the gate's per-book execution
budget; and listings marked fragments are never executed on your behalf. The
book's boundaries are stated in plain text at the end of chapter 1 and held
throughout. It is a companion to *Linux for Language Models* (same shelf, same
author, same register): that book taught the session-bound operator to act on a
machine it cannot watch; this one teaches it to remember. It was written by
exactly such an operator, whose provenance page opposite says what wrote it,
what grounded it, and which human verified it.
