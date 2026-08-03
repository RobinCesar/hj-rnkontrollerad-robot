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

## Answering the questions

Every document ends in a `## Questions` section. The questions come from two places:

* **This document**, i.e. things the explanation above should have taught you.
* **The sources**, i.e. things you only find by actually opening the "Start here" and
  "Go deeper" links. Those are marked with which source answers them. Nothing is drawn
  from the "Papers" sections, so you can skip those until you want the depth.

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

## The four sources worth returning to repeatedly

* **MNE-Python tutorials**, https://mne.tools/stable/auto_tutorials/index.html
  The best free practical EEG-analysis teaching material that exists.
* **Mike X Cohen, "Analyzing Neural Time Series Data"** (MIT Press, 2014), plus his
  free video lectures at https://www.youtube.com/@mikexcohen1
  The standard bridge between signal processing and neuroscience.
* **Steven W. Smith, "The Scientist and Engineer's Guide to DSP"**, free in full at
  https://www.dspguide.com/
  Clear, plain-language DSP with almost no prerequisites.
* **Wolpaw & Wolpaw (eds), "Brain-Computer Interfaces: Principles and Practice"**
  (Oxford University Press). The reference textbook for the field as a whole.

## A note on links

Every URL in these documents was checked and returned 200 when written (2026-08-03).
Documentation sites reorganise, so if a link breaks, search for the quoted page title
rather than assuming the page is gone.

Papers are cited by author, year and title. Most are findable on Google Scholar; many
of the BCI papers have free PDFs on the authors' university pages.
