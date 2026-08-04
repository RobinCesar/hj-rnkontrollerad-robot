# Topics

One document per concept the project depends on. Each explains the idea, lists the
mistakes people make with it, and ends with sources for learning it properly.

These are meant to be read *before* the corresponding phase in
[LEARNING_PLAN.md](../LEARNING_PLAN.md), not as a reference to look things up in.
For API calls and signatures, use [SYNTAX.md](../SYNTAX.md) instead.

| # | Topic | Needed in phase |
|---|---|---|
| 01 | [EEG basics](01-eeg-basics.md) | 0 |
| 02 | [Motor imagery and ERD](02-motor-imagery-erd.md) | 0, 4 |
| 03 | [Artifacts](03-artifacts.md) | 0, 2, 4 |
| 04 | [Data acquisition](04-data-acquisition.md) | 1 |
| 05 | [Digital filtering](05-digital-filtering.md) | 2, 5 |
| 06 | [Epoching](06-epoching.md) | 2 |
| 07 | [CSP](07-csp.md) | 3 |
| 08 | [Classification](08-classification.md) | 3 |
| 09 | [Validation and statistics](09-validation.md) | 3, 4 |
| 10 | [Experimental design](10-experimental-design.md) | 4 |
| 11 | [Real-time processing](11-realtime.md) | 5 |
| 12 | [Game design for BCI](12-game-design.md) | 6 |
| 13 | [Visualisation](13-visualisation.md) | 7 |
| 14 | [Extending the montage](14-electrode-hardware.md) | 4B (optional) |

## The two source tiers

Every document ends in a `## Sources` section split in two. The split is a filter, not a
ranking, and it exists because the fastest way to stall this project is to queue up a
reading list instead of writing code.

* **Tier 1** is what you read before starting the corresponding phase. It is short
  everywhere, usually two or three items, and a few topics have a paper in it because on
  that particular subject a paper really is the thing to read. If a topic's Tier 1 takes
  more than an evening, that is the exception and it says so.
* **Tier 2** is what you open when a Tier 1 source assumed something you did not have,
  when a design decision needs justifying in [REPORT.md](../REPORT.md), or when the basic
  version works and you want the better one. Not homework. Several topics tell you
  explicitly that there is nothing here worth reading, which is itself information.

Everything that was in the old "Papers" and "Video" lists and is not in one of these two
tiers was cut deliberately. It was not wrong, it was just more reading than the project
needs.

## Answering the questions

Every document ends in a `## Questions` section. The questions come from two places:

* **This document**, i.e. things the explanation above should have taught you.
* **The sources**, i.e. things you only find by actually opening a Tier 1 or Tier 2 link.
  Those are marked with which source answers them, papers included. Every paper that
  survived the cut has at least one question attached, which is the check on whether it
  earned its place.

Each question looks like this:

```
**Q3.** Why does the Muse reference at FPz make eye artifacts appear on every channel?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_
```

Replace `_(unanswered)_` with your answer, in your own words. Short is fine; the point
is to find out whether you can reconstruct the reasoning, not to write an essay. Leave
the `reviewed:` tag alone.

**Getting them checked.** Ask Claude in any session to check your answers. It looks for
questions that are answered but still tagged `reviewed: no`, marks each answer correct,
partly correct or wrong with a short explanation, and then flips the tag to
`reviewed: yes`. Questions already tagged `reviewed: yes` are skipped, so you can answer
a few at a time and only pay for what is new. The full procedure is in
[CLAUDE.md](../CLAUDE.md).

## Reading order if you only have time for four

1. [Motor imagery and ERD](02-motor-imagery-erd.md), because it is the physical effect
   the whole project rests on.
2. [Digital filtering](05-digital-filtering.md), because almost every early bug is a
   filtering bug.
3. [CSP](07-csp.md), because it is the one genuinely non-obvious algorithm here.
4. [Validation and statistics](09-validation.md), because without it you cannot tell a
   real result from a broken one.

## The sources you will keep opening

Four of the Tier 1 entries recur across several topics, so they are worth bookmarking
rather than re-finding each time:

* **MNE-Python tutorials**, https://mne.tools/stable/auto_tutorials/index.html
  The best free practical EEG-analysis teaching material that exists.
* **MNE's "Decoding motor imagery with CSP" example**, which is simultaneously the target
  of Phase 3, the source of your epoch window, and a worked CSP pipeline. It appears in
  Tier 1 of [02](02-motor-imagery-erd.md), [06](06-epoching.md) and [07](07-csp.md).
* **scipy.signal**, https://docs.scipy.org/doc/scipy/reference/signal.html
  Five or six docstrings that constitute your entire filtering implementation.
* **scikit-learn's cross-validation and pipeline pages**, which is where the honesty of
  every number in the report comes from.

## The papers that survived

Eight papers across fourteen topics, and each is attached to a question so you can tell
whether reading it landed. In rough order of how much they matter here:

| Paper | Topic | Why it is in |
|---|---|---|
| Pfurtscheller & Lopes da Silva (1999) | [02](02-motor-imagery-erd.md) | The effect the project detects, and the ERD formula |
| Combrisson & Jerbi (2015) | [09](09-validation.md) | Stops you reporting a chance result as a finding |
| Blankertz et al. (2008) | [07](07-csp.md) | The one good explanation of CSP |
| Goncharova et al. (2003) | [03](03-artifacts.md) | EMG at exactly your electrode positions |
| Haufe et al. (2014) | [07](07-csp.md) | Stops you misreading your own report figure |
| de Cheveigné & Nelken (2019) | [05](05-digital-filtering.md) | Filtering creates effects as well as removing them |
| Wolpaw et al. (2002) | [12](12-game-design.md) | What is and is not a BCI, plus ITR |
| Krigolson et al. (2017) | [14](14-electrode-hardware.md) | What the stock Muse can actually do |

Tier 2 adds nine more (Buzsáki 2012, Neuper 2005, Blankertz 2010, Muthukumaraswamy 2013,
Lotte 2018, Varoquaux 2018, Shenoy 2006, Vidaurre 2011, Kappenman & Luck 2010), none of
which you need before starting the phase they belong to. Seventeen papers total across
the whole project, down from about forty.

## A note on links

Every URL in these documents was checked and returned 200 when written (2026-08-03).
Documentation sites reorganise, so if a link breaks, search for the quoted page title
rather than assuming the page is gone.

Papers are cited by author, year and title. Most are findable on Google Scholar; many
of the BCI papers have free PDFs on the authors' university pages.
