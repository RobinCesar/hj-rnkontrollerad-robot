# Brain-Controlled Game

A BCI (Brain-Computer Interface) project that classifies **motor imagery**, i.e.
whether the user is imagining a movement with the left or the right hand, and maps
that classification to control in a game.

The goal is twofold: a working real-time system, and a written report
([REPORT.md](REPORT.md)) documenting method, results and limitations.

The same pipeline is also run on a **deliberate eye/jaw control** condition. That is not
a BCI, since it depends on eye and jaw muscles rather than on brain activity, and it is
labelled as such throughout. It serves two purposes: it measures the ceiling that
non-neural control reaches on this hardware, which is what the motor imagery result has
to be distinguished from, and it gives the game an input source that reliably works.
See [REPORT.md](REPORT.md#d1-the-ladder) for how the conditions compare.

## Hardware

| | |
|---|---|
| Headset | **Muse 2** |
| Channels | 4 EEG: **TP9, AF7, AF8, TP10** (reference FPz) |
| Sampling rate | 256 Hz |
| Connection | Bluetooth LE, via BrainFlow (`MUSE_2_BOARD` = 38 native, `MUSE_2_BLED_BOARD` = 22 with the BLED112 dongle) |

> **Important limitation:** the Muse electrodes sit temporally and frontally, not over
> sensorimotor cortex (C3/C4/Cz) where mu/beta ERD during hand movement is strongest.
> This is the project's main scientific challenge and is covered in
> [REPORT.md](REPORT.md#e-limitations).

## Workflow

1. **Setup.** The Muse is placed on the head and signal quality is checked. The
   program shows instructions through the game.
2. **Acquisition.** The Muse transmits over Bluetooth and the computer picks up the
   signal via BrainFlow. Every cue is written into the stream as an event marker so
   the data can be labelled afterwards.
3. **Preprocessing**
   * Bandpass filter 8:30 Hz (mu and beta bands)
   * Notch filter 50 Hz (mains noise)
   * Artifact removal (eye blinks, jaw and muscle movement)
   * Epoching: the data is cut into windows around each event
4. **Feature extraction (CSP).** Common Spatial Patterns builds spatial filters from
   the raw data that maximise the variance difference between the classes (left vs.
   right). The feature is the log-variance per CSP component.
5. **Classification.** LDA as the baseline, optionally cross-checked against an SVM.
   In real time: a sliding window of 1:2 s updated every 250 ms, where each window is
   classified separately and the decisions are smoothed over time.
6. **Game control.** The output is mapped to movement in the game (game not chosen yet).
   The game reads one input interface with four interchangeable sources behind it:
   keyboard, replay of recorded data, the eye/jaw classifier and the motor imagery
   classifier. The game itself never imports BrainFlow or sklearn, so it can be built
   and tested without the headset. The active source is always shown on screen.
7. **Visualisation.** Real-time plots of the signal, band power and classifier
   confidence, plus topographies and weights for the CSP components.
8. **Report.** Running notes in [REPORT.md](REPORT.md).

## Repository structure

```
hjärnkontrollerat_spel/
├── README.md              # This document: overview and workflow
├── REPORT.md              # Lab notebook + material for the report
├── LEARNING_PLAN.md       # Ordered TODO and learning plan, phase by phase
├── SYNTAX.md              # API reference for every package used
├── CLAUDE.md              # Context + working rules for AI assistance
├── requirements.txt
├── config.yaml            # Parameters: filter bands, window length, board_id, etc.
│
├── topics/                # One document per concept, with sources for learning it
│   ├── 01-eeg-basics.md
│   ├── 02-motor-imagery-erd.md
│   ├── ...
│   └── 13-visualisation.md
│
├── data/
│   ├── raw/               # Untouched recordings (always keep the original)
│   └── processed/         # Epoched / filtered datasets
├── models/                # Saved, trained pipelines (.joblib)
├── notebooks/             # Exploratory analysis
│
├── src/
│   ├── acquisition.py     # BrainFlow: connect, stream, markers
│   ├── preprocessing.py   # Filtering, artifact removal, epoching
│   ├── features.py        # CSP and feature extraction
│   ├── classify.py        # Training, cross-validation, evaluation
│   ├── realtime.py        # Sliding-window loop, decision smoothing
│   ├── viz.py             # Plots and real-time visualisation
│   └── game/              # The game (pygame)
│
└── scripts/
    ├── record_session.py  # Run the cue paradigm and record training data
    ├── train_model.py     # Train and save a model from recorded data
    └── play.py            # Load a model and play
```

## Installation

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

## Usage

```bash
python scripts/record_session.py    # 1. Collect calibration data
python scripts/train_model.py       # 2. Train and evaluate the model
python scripts/play.py              # 3. Play
```

## Status

The project is in its startup phase. See [LEARNING_PLAN.md](LEARNING_PLAN.md) for the
current phase and next steps, and [topics/](topics/) for the background reading each
phase depends on.

| Phase | Content | Status |
|-----|----------|--------|
| 0 | Setup, EEG fundamentals | 🔜 |
| 1 | Data acquisition from the Muse | ⬜ |
| 2 | Preprocessing | ⬜ |
| 3 | Offline pipeline on a public dataset | ⬜ |
| 4 | Offline pipeline on own Muse data | ⬜ |
| 5 | Real-time classification | ⬜ |
| 6 | The game | ⬜ |
| 7 | Visualisation | ⬜ |
| 8 | Report | ⬜ |
