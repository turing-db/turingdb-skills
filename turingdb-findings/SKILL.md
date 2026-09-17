---
name: turingdb-findings
description: >
  Produce an actionable defect and feature-gap register for TuringDB from real usage — severity-banded,
  every item carrying a reproduction, the exact engine message and the version it was last seen on.
  TRIGGER when: the user is testing, evaluating, benchmarking or stress-testing TuringDB; asks to
  report, collect, track or write up TuringDB bugs, crashes, limitations, missing features or
  performance problems; says something "doesn't work", "isn't supported" or "is slow" in TuringDB and
  wants it recorded; asks for a findings report, defect register, bug report or engine feedback for the
  TuringDB team.
  SKIP: routine TuringDB query or ingest work with no intent to report (use the `turingdb` skill);
  bug reports about anything other than the TuringDB engine, SDK, CLI or its documentation.
---

# TuringDB findings

Turning "this was annoying" into something an engine team can act on.

The output is a **register**, not a bug list: every item carries a reproduction,
the exact message the engine produced, and the version it was last seen on —
ordered by what it costs the person who hits it, not by when you found it.

## The one rule

**Every finding is confirmed against a running server before it is recorded.**

Not from memory, not from the docs, not from a skill file. The dialect moves in
both directions between releases — 1.36 made `type` a reserved word, 1.37
*relaxed* four rules that earlier versions rejected — so a limitation you
"know about" may have been fixed, and a capability you rely on may have gone.
A register full of stale entries is worse than no register: it trains the
engine team to discount it.

If you cannot reproduce it, it is not a finding yet. Write it as an open
question instead and say what you could not pin down.

## Severity

Band by **what it costs the person who hits it**, not by how hard it looks to fix.

| | | |
|---|---|---|
| **S1** | Crashes or loses data | The instance goes down, or bytes are at risk |
| **S2** | Silently returns a wrong answer | No error. The caller cannot tell |
| **S3** | Blocks a real use case | Errors honestly, but the thing cannot be done |
| **S4** | Costs time or forces a workaround | Expressible, just badly |

**S2 outranks S3 deliberately.** A query that errors is a nuisance; a query that
returns a plausible wrong number ends up in a paper. When you are unsure between
the two, ask: *would the user find out?* If the answer is no, it is S2.

A missing feature is usually S3 or S4. It becomes S2 when its absence pushes
people into a pattern that is quietly wrong — no bound parameters is S3 for
expressiveness and closer to S1 for anything taking user input, because the only
available workaround is string interpolation.

## What every finding needs

Four fields. A finding missing any of them will be bounced back.

1. **A reproduction** small enough to paste. Reduce it — the `shortestPath`
   crash was found on a real graph and reduced to eight nodes and five variants,
   which is what made it diagnosable.
2. **The exact engine output**, copied not paraphrased. `PARSE_ERROR`,
   `ANALYZE_ERROR`, `PLAN_ERROR` and `EXEC_ERROR` mean different things here —
   they are raised by different stages — so the class is diagnostic information,
   not noise.
3. **The version**, and the SDK/CLI split if they differ.
4. **Why it matters** — one sentence on the consequence, separate from the
   description. For most findings the defect is obvious and the consequence is
   not. "Introspection over-reports" sounds cosmetic until you say that
   introspection is what a client uses to discover a schema it did not write.

Add a **measurement** wherever the finding is about cost. "Slow" is not
reportable; "157 ms against a query that did not return in 900 s, at 4.7 M
nodes" is.

## Probing: where the findings actually are

Work through these against a graph with real shape and real scale. A toy graph
hides everything interesting — several of the worst findings only appear past a
few million edges.

**Read the server log, always.** `<turing-dir>/logs/` routinely carries a
diagnostic the API response omits. A `LOAD JSONL` of a 3.5 GB file returns only
`Failed to load JSONL graph <file>`, while the log names the line, column and
offending token.

**Check the error field, not the status code.** Failures arrive as HTTP 200 with
a non-null top-level `error`. Any harness that checks status alone will record
every failure as a pass.

### The classes worth sweeping

- **Dialect conformance.** Take the cases from
  `docs.turingdb.ai/query/cypher_subset`, not from memory, and tag each with
  what the docs claim. Report four outcomes: `OK`, `BROKEN` (docs say yes, it
  fails), `MISSING` (confirmed unsupported), `EXTRA` (works but undocumented —
  a workaround you can now delete). `EXTRA` findings are as valuable as bugs.
- **Read-after-write inside a change.** Write, then read back in the same
  change, and compare against the truth. This is where the silent ones live.
- **Introspection accuracy.** Compare `db.edgeTypes()` and `db.propertyTypes()`
  against actual `count()` per type. They read a registry that keeps names from
  earlier builds of the same graph.
- **Query phrasing.** Write the same question two ways and time both. The
  planner follows the pattern as written, so phrasing differences are worth
  orders of magnitude, not percentages.
- **Scale behaviour.** Measure the same operation at two corpus sizes and check
  whether the cost is linear. Several costs here are quadratic or
  payload-independent in ways that only show up on the second measurement.
- **Restart and lifecycle.** Stop the server, start it, and query immediately.
  Graphs are not resident after a restart, and an unloaded graph answers
  `GRAPH_NOT_FOUND` rather than returning nothing.
- **Version bumps.** Re-run the whole sweep and **diff it against the last run**.
  That is the only reliable way to see what moved, in either direction.

## Traps that make your testing lie

These cost real time. They are listed because each one produced a result that
looked clean and was not.

**A check that cannot fail reads exactly like a check that passed.** Before
trusting any green run, inject a known-bad case and confirm it goes red. A query
gate here once reported "all shipped queries execute cleanly" while extracting
12 of 127 queries; its negative control ran *after* extraction, so it exercised
the transport and proved nothing about the broken part. **Put the control
through the component you doubt.**

**Silence is not success.** A validator that analyses zero rows and prints
`PASS` is the same failure in a different costume. Assert on the population
size, not only on the check.

**Measure before concluding, and measure again after a version bump.**
`MERGE_DATAPARTS` was recorded as a no-op twice on reasoning about write
patterns, then measured on the one shape the docs actually aim it at — a
21 M-element single-shot bulk load — and found to be a no-op there too. The
measurement is what makes it reportable; the reasoning only made it plausible.

**Do not trust a count you did not derive.** Re-derive figures from the live
graph rather than quoting an earlier run. A deduplication guard that scanned
only the last 400 edges inflated a motif count elevenfold, and every number
downstream of it looked reasonable.

**Separate a defect from a documented limitation.** Both belong in the register,
but conflating them wastes the engine team's attention and makes the urgent
items harder to see. If the docs say it is unsupported, it is a gap, not a bug —
unless the docs are wrong, which is itself a finding.

## Writing it up

Order by severity, then by consequence. Group by what the absence costs, not
alphabetically — "blocks common questions", "forces a rewrite", "absent from the
subset" tells a reader where to look; an A–Z list does not.

Give S1 and S2 items room: full entries with the reproduction and the exact
message. Compress S3 and S4 into tables. Spending equal space on a crash and a
missing `collect()` buries the crash.

**Keep the tone factual.** No editorialising about the product, no rhetoric
about how bad something is. The measurements carry the argument; language on top
of them reads as advocacy and gets discounted. Where you did not test something,
say so rather than implying coverage.

**Close with a short prioritised list** — five at most — and give the reason for
each ordering. The engine team will disagree with some of it, and that is fine;
what they cannot do is infer your reasoning from a flat list.

**State the scope honestly at the end.** A register is what one team hit while
doing their own work, not an audit. Name the areas you never exercised.

## Keeping it alive

A register decays the moment it stops being updated. Record findings **as they
happen**, in the project's own notes file, with severity and version attached —
reconstructing them weeks later is where the exact messages get lost.

When a version bump fixes something, **move it rather than deleting it**, noting
the version that fixed it. Knowing when something was fixed is what makes the
next surprise legible, and it is the evidence that the register is maintained.

## Publishing

For a report going to the engine team, a page reads better than a document: the
severity banding is visual, the reproductions want monospace, and the reader is
scanning for their component rather than reading start to finish. Offer it, and
keep the underlying register in the repo so the page can be regenerated.
