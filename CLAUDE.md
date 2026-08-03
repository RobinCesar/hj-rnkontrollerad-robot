# CLAUDE.md

Context and working rules for AI assistance in this repository.

**This file is loaded automatically at the start of every session in this directory.**
Anything written here is known to the assistant without Robin having to repeat it. That
is why the standing conventions, the project structure and the question-checking
procedure all live here rather than being explained again each time.

# The most important rule: teach, do not build

**This project exists so that Robin learns BCI and signal processing. Writing the code
for him defeats the entire purpose.**

Robin's starting point: solid basic Python, numpy and matplotlib. Everything else
(BrainFlow, MNE, scikit-learn, scipy.signal, pygame, EEG theory) is new.

## What to do

* Explain concepts, the theory behind them and *why* something is done a certain way.
* Give API signatures, function names, argument meanings and small isolated snippets
  that demonstrate a single call.
* Point at the right function, module or documentation page and let him assemble it.
* Ask leading questions when he is stuck: "what shape is your array at that point?",
  "what happens to the phase when you filter?"
* Review code he has written: point out bugs, methodological mistakes (especially data
  leakage and artifact contamination) and better idioms, with reasoning.
* Debug alongside him: help read a traceback, suggest what to print, narrow down where
  the problem is. Explain the cause, then let him apply the fix.
* Be blunt about methodological errors. A silently wrong result is worse than no result.

## What not to do

* Do not write complete functions, modules or scripts unless he explicitly asks for
  that specific file to be written for him.
* Do not implement an entire pipeline stage. Preprocessing, epoching, the CSP feature
  step, the real-time loop and the game are all his to write.
* Do not fix his code by rewriting it. Say what is wrong and where.
* Do not hand over a finished solution because it is faster.
* Do not put big blocks of project code into README.md, REPORT.md or LEARNING_PLAN.md.
  Those documents hold plans, theory and API reference only.

## Where the line goes

Fine to write directly: `.gitignore`, folder scaffolding, `requirements.txt`,
documentation, plotting boilerplate that is not part of what he is learning, and
throwaway diagnostic snippets used to answer a question.

Everything under `src/` and `scripts/` is his.

If he asks for a full solution anyway, first check whether he wants to be walked
through it instead. If he confirms, then write it, and explain the code afterwards.

# Project

A brain-computer interface that classifies motor imagery (left vs. right hand) from a
4-channel Muse 2 headband and uses it to control a game.

Pipeline: BrainFlow acquisition, 8:30 Hz bandpass + 50 Hz notch + artifact removal,
epoching, CSP feature extraction, LDA (and SVM as a comparison), sliding-window
classification in real time, mapped to game input.

**The ladder.** The same pipeline is run unchanged across a set of conditions ordered
from "no signal by construction" to "signal that is definitely present", listed in
REPORT.md D.1. Motor imagery on Muse data is the scientific question; deliberate
eye/jaw control is the top rung, measuring what non-neural control achieves on this
hardware. No single accuracy in this project is interpretable on its own, only the gaps
between rungs. When Robin reports a number, ask which rung it is and what it sits next
to.

**Eye/jaw control is not a BCI** and must never be presented as one. Wolpaw's definition
requires measuring central nervous system activity and not depending on peripheral
nerves and muscles. It is in the project as a measured comparison and as the game's
reliable input source, and it is labelled as non-neural in README.md, REPORT.md and
on screen during play. If Robin's phrasing anywhere blurs this, say so.

Key documents:

| File | Contents |
|---|---|
| [README.md](README.md) | Overview, hardware, workflow, repo structure |
| [REPORT.md](REPORT.md) | Lab notebook and the skeleton of the final report |
| [LEARNING_PLAN.md](LEARNING_PLAN.md) | Phases 0 to 8, tasks and pitfalls |
| [topics/](topics/) | One document per concept, each ending in curated sources |
| [SYNTAX.md](SYNTAX.md) | API signatures for BrainFlow, MNE, sklearn, scipy, pygame |

When explaining a concept, check whether `topics/` already covers it and point there
rather than re-explaining from scratch. If the explanation there is wrong or missing
something, fix the topic document.

## Repository structure

```
hjärnkontrollerat_spel/
├── CLAUDE.md              # This file. Working rules, project facts, session notes
├── README.md              # Overview, hardware, workflow, intended repo layout
├── REPORT.md              # Lab notebook (part A) + skeleton of the final report (B to G)
├── LEARNING_PLAN.md       # Phases 0 to 8: tasks, pitfalls, "done when"
├── SYNTAX.md              # API signatures, verified against the installed versions
├── requirements.txt
├── .gitignore             # .venv/, __pycache__/, data/, models/
│
├── topics/                # 13 concept documents + README with the reading order
│   ├── README.md          # Index, reading order, the question-answering convention
│   └── 01..13-*.md        # Explanation, common mistakes, Questions, Sources
│
├── data/raw/              # Untouched recordings. Git-ignored
├── data/processed/        # Epoched / filtered datasets. Git-ignored
├── models/                # Saved sklearn pipelines (.joblib). Git-ignored
├── src/                   # Robin's code (empty so far)
└── scripts/               # Robin's entry points (empty so far)
```

The `src/` and `scripts/` file layout planned in README.md (`acquisition.py`,
`preprocessing.py`, `features.py`, `classify.py`, `realtime.py`, `viz.py`, `game/`, and
the three scripts) does not exist yet. Nothing under `src/` or `scripts/` has been
written. `config.yaml` and `notebooks/` from the README layout do not exist either.

**Document roles, so nothing is duplicated:**

| Document | Answers |
|---|---|
| README.md | What is this project and how do I run it |
| LEARNING_PLAN.md | What do I do next, in what order |
| topics/ | Why does it work that way, and where do I learn it |
| SYNTAX.md | What is the exact call signature |
| REPORT.md | What happened, and what goes in the write-up |
| CLAUDE.md | How the assistant should behave, and project state |

## The questions in topics/

Every topic document ends in a `## Questions` section. Format per question:

```
**Q7.** Question text.
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_
```

Roughly half of each set comes from the document's own explanation, the rest from the
"Start here" and "Go deeper" sources, named in the `Source:` line. Nothing is drawn from
the "Papers" sections. There are 151 questions in total.

### Checking answers when Robin asks

He answers over time and asks for a check in a later session. The procedure:

1. Find the work: `grep -n "reviewed: no" topics/*.md`, then look at each hit and skip
   any whose answer is still `_(unanswered)_`. **Only grade questions that are both
   answered and still tagged `reviewed: no`.** Anything tagged `reviewed: yes` has
   already been graded, in this session or an earlier one, and must not be re-graded.
2. Grade each one as **correct**, **partly correct** or **wrong**. Be honest; a wrong
   answer waved through is worse than no check at all. Say specifically what is missing
   or mistaken, and why it matters downstream.
3. Write the verdict into the file, directly under his answer, as a second blockquote
   line beginning `> **Check:**`. Keep it to a few sentences. Do not rewrite his answer.
4. Change that question's tag to `reviewed: yes`.
5. Summarise in chat: how many were checked, which ones were wrong, and any pattern
   worth addressing (e.g. "the reference/volume-conduction distinction is not landing").

If his answer reveals that the topic document itself is unclear or wrong, fix the topic
document too, and say so.

## Facts worth remembering

* The headset is a **Muse 2**. BrainFlow board IDs: `MUSE_2_BOARD` = 38 for native BLE,
  `MUSE_2_BLED_BOARD` = 22 for the BLED112 dongle. Different IDs for the same hardware.
* Muse EEG channels are TP9, AF7, AF8, TP10 at 256 Hz, referenced to FPz. They do
  **not** cover sensorimotor cortex (C3/C4/Cz), which is where motor imagery ERD is
  strongest. This is the central limitation of the project and is discussed in
  REPORT.md section E.
* BrainFlow returns microvolts, MNE expects volts.
* CSP comes from `mne.decoding.CSP`. The PyPI package named `csp` is an unrelated
  stream-processing library and should not be installed.
* Suspiciously high accuracy on Muse data usually means artifact contamination (jaw
  EMG at TP9/TP10, eye movement at AF7/AF8), not success.
* CSP and any fitted filter must sit inside the sklearn `Pipeline` so they are refitted
  per cross-validation fold, otherwise the results leak.

# Conventions

* Documentation in this repo is written in English.
* Follow the global preferences in `~/.claude/CLAUDE.md`: a short comment above every
  function, and a `#` note on every import explaining why it is there.
* Do not use `---` horizontal rules or em dashes in the documentation files.

# Session notes

## 2026-08-03

* Restructured README.md and REPORT.md, and translated both to English.
* Added LEARNING_PLAN.md: an 8-phase plan with tasks, learning goals, pitfalls and a
  verified syntax reference for BrainFlow, MNE, scikit-learn, scipy.signal, pygame and
  numpy. All API signatures were checked against the installed package versions.
* Added the "teach, do not build" rule above at Robin's request.
* Main content added beyond what Robin had written: validating the pipeline on the
  public `mne.datasets.eegbci` dataset before trusting Muse data (Phase 3), the
  recording protocol and event synchronisation (Phase 4), baselines and permutation
  testing, and the artifact-contamination controls in REPORT.md section E.2.
* Open issue: `csp==0.18.0` in requirements.txt is the wrong package and should be
  uninstalled.
* Next: Phase 0, then getting data out of the Muse.

### 2026-08-03, later

* Split the syntax reference out of LEARNING_PLAN.md into [SYNTAX.md](SYNTAX.md).
* Added [topics/](topics/): 13 documents, one per concept, each with an explanation,
  common mistakes, self-check questions and curated sources (docs, books, papers).
  Every URL was checked for a 200 response on 2026-08-03.
* LEARNING_PLAN.md now links each phase to its prerequisite topic documents.
* Clarified the `.gitignore` task after Robin questioned it: the point is that git
  should not *track* `.venv/` or `__pycache__/`, and that committed data files bloat
  history permanently. Explanation kept in the Phase 0 section since it was a genuine
  misunderstanding worth recording.
* Added to Phase 3: fix hyperparameters on the public dataset before touching own data,
  so that later cross-validation scores stay valid.
* Added to Phase 6: a replay input source alongside keyboard and live classifier, for
  deterministic testing of the real-time chain without hardware.

### 2026-08-03, later still

* Robin read topic 01 and answered its five questions in the file. Not yet checked.
* Renamed `## Check yourself` to `## Questions` in all 13 topic documents and rebuilt
  the sets: existing questions kept, source-derived questions added, answer boxes and
  `reviewed:` tags added throughout. 151 questions total, 11 to 12 per document.
  Convention documented in [topics/README.md](topics/README.md#answering-the-questions)
  and the checking procedure in the section above.
* Added the repository structure and document-role table above, and pinned the hardware
  to Muse 2 with its board IDs.
* Answered Robin's question about the missing C3/C4/Cz coverage: it caps how well any
  method can work, makes CSP nearly pointless with 4 channels, and makes every artifact
  control in REPORT.md E.2 mandatory rather than optional. The Phase 3 channel-subset
  experiment is what turns the limitation into a measured number, which is the result
  that makes the report worth reading. Nothing in the plan changes because of it.
* Current state: nothing written under `src/` or `scripts/` yet. Phase 0 still open, and
  `pip uninstall csp` still outstanding.

### 2026-08-03, the ladder decision

Robin asked whether to switch the project's focus from motor imagery to eye movement and
jaw clenching, since he is not committed to hands. Recommendation given and accepted:
**do not swap, restructure.** A straight swap stops the project being a BCI at all,
deletes Phases 3 and 4 and most of the validation and CSP material, and leaves a report
with nothing to say. Instead:

* Motor imagery stays the scientific question.
* Eye/jaw becomes a fully recorded condition run through the identical pipeline, i.e.
  row 8 of the ladder, promoted out of REPORT.md E.2 where it had been a minor control.
* Eye/jaw is also the game's reliable input source, so the deliverable no longer depends
  on a signal that may not exist.

Changes made: REPORT.md gained research question 5, the terminology warning, the D.1
ladder table replacing the old baselines table, per-source D.3, and an expanded E.2.
LEARNING_PLAN.md Phase 4 gained the eye/jaw recording task and the honesty note, Phase 5
gained "bring the loop up on eye/jaw first", Phase 6 went from three input sources to
four with the on-screen source label, Phase 7 gained the paired CSP pattern figure and
the ladder bar chart. README.md and the Project section above were updated to match.

SSVEP was raised as the one genuine alternative where the Muse's placement is a handicap
rather than a disqualification, and declined for now. If Robin revisits it, it needs new
topic documents on SSVEP and CCA and a rewrite of Phases 5 to 7.

Not changed: the topic documents. Nothing in them became wrong, and topic 03 already
covers the eye and jaw signals in the detail the new condition needs.
