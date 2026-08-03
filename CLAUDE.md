# CLAUDE.md

Context and working rules for AI assistance in this repository.

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
4-channel Muse headband and uses it to control a game.

Pipeline: BrainFlow acquisition, 8:30 Hz bandpass + 50 Hz notch + artifact removal,
epoching, CSP feature extraction, LDA (and SVM as a comparison), sliding-window
classification in real time, mapped to game input.

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

## Facts worth remembering

* Muse EEG channels are TP9, AF7, AF8, TP10 at 256 Hz. They do **not** cover
  sensorimotor cortex (C3/C4), which is where motor imagery ERD is strongest. This is
  the central limitation of the project and is discussed in REPORT.md section E.
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
