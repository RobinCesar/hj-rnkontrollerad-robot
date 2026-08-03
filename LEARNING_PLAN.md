# Learning plan and TODO

An ordered path from zero to a finished project. The phases build on each other, so do
not skip one because it feels boring: debugging the next phase becomes impossible if
you do.

**How to read it:** each phase links to the [topic documents](topics/) that explain the
concepts, then lists *Do* (concrete tasks), *Pitfalls* and *Done when*. Read the linked
topics before starting the phase.

| Document | Purpose |
|---|---|
| [topics/](topics/) | What each concept is, why it matters, and sources to learn it from |
| [SYNTAX.md](SYNTAX.md) | API calls and signatures for every package |
| This file | What to do, in what order |

Starting point: Python, numpy, matplotlib. Everything else is new.

# Phase 0: Setup and fundamentals

**Read first**
* [01. EEG basics](topics/01-eeg-basics.md)
* [02. Motor imagery and ERD](topics/02-motor-imagery-erd.md)
* [03. Artifacts](topics/03-artifacts.md)

**Do**
* [ ] Read through [README.md](README.md) and [REPORT.md](REPORT.md) so the structure sticks.
* [ ] Create the folder structure from the README (`src/`, `data/raw/`, `scripts/`, ...).
* [ ] Add a `.gitignore` containing `.venv/`, `__pycache__/`, `data/`, `models/`.
      See the note below on why each entry is there.
* [ ] **Uninstall the `csp` package**: `pip uninstall csp`. It is a stream-processing
      library from Point72, not Common Spatial Patterns. CSP for BCI lives in
      `mne.decoding.CSP`. Update `requirements.txt`.
* [ ] Learn the `venv` + `pip freeze > requirements.txt` workflow.
* [ ] Revise numpy: `shape`, axes, slicing, broadcasting, `np.mean(x, axis=...)`. You
      will work with 3D arrays of shape `(trials, channels, samples)` for the whole
      project. If axis handling feels shaky now, everything after this gets much harder.
* [ ] Draw the 10-20 positions C3, C4 and Cz on a head, then add the four Muse
      electrodes. Keep the drawing; it explains the whole project.

**On `.gitignore`**

A `.gitignore` does not create or delete anything. It is a list of paths that git should
refuse to track, so they never enter a commit. The entries are there precisely *because*
those things already exist or will create themselves:

* `.venv/` exists, and that is the problem. Without the ignore, `git add .` would stage
  several thousand library files, and your repo would contain a copy of numpy that only
  works on your machine and your Python version. `requirements.txt` is the portable
  replacement.
* `__pycache__/` does create itself, repeatedly, and git would commit the `.pyc` files
  and then show them as modified every time you run anything.
* `data/` and `models/` hold recordings and trained models. These are large, they are
  regenerable, and git stores every version of every file forever, so a repo that has
  ever contained a 200 MB recording stays 200 MB larger permanently even after you
  delete the file. EEG recordings are also personal data, which is a second reason not
  to push them to a public repository.

The point is not that these files are unimportant. It is that git is the wrong tool for
storing them.

**Done when:** you can draw the Muse electrodes on a head and explain why the placement
is a problem for classifying hand movement.

# Phase 1: Get data out of the Muse

The goal is only to see numbers moving. Nothing more.

**Read first**
* [04. Data acquisition](topics/04-data-acquisition.md)

**Do**
* [ ] Pair the Muse with the computer over Bluetooth. Find out which `BoardIds` value
      your model has (native BLE vs. BLED112 dongle are different IDs).
* [ ] Write a minimal script that connects, streams for 10 seconds and prints
      `data.shape`. Understand what every number means.
* [ ] Plot 10 seconds of raw EEG from all 4 channels in matplotlib.
* [ ] Blink hard and clench your jaw during a recording. Find the artifacts in the
      plot. Note which channels are affected most: that becomes a figure in the report.
* [ ] Write a signal-quality check: if a channel's standard deviation is extreme, the
      electrode is seated badly.
* [ ] Save to file and read it back. Decide on a format now and stick to it, and save a
      small metadata file alongside each recording.
* [ ] Measure your cue-to-marker latency using the blink method in topic 04. Most
      student projects skip this; it belongs in your report.

**Pitfalls**
* BrainFlow buffers internally. If you call `get_board_data()` too rarely you lose data.
* The unit is microvolts. If the numbers are in the thousands, the electrodes are wrong.
* The first few seconds after connecting are usually garbage, so throw them away.
* Always `release_session()`, including on error, or the headset may refuse to reconnect.

**Done when:** you can record, save, load and plot EEG from the Muse, and you know your
timing delay.

# Phase 2: Preprocessing

**Read first**
* [05. Digital filtering](topics/05-digital-filtering.md)
* [06. Epoching](topics/06-epoching.md)

**Do**
* [ ] Implement a 8:30 Hz bandpass. Verify it by plotting the power spectrum before and
      after: everything outside the band should be attenuated.
* [ ] Implement a 50 Hz notch. Verify the same way (you should see a 50 Hz peak in the
      raw data disappear).
* [ ] Write an epoching function: in comes continuous data plus marker times, out comes
      an array of shape `(n_trials, n_channels, n_samples)`.
* [ ] Artifact removal, start simple: reject trials where the amplitude exceeds a
      threshold (e.g. ±100 µV). Log how many trials you lose, per class.
* [ ] Read about ICA for eye artifacts, but do not implement it: with 4 channels there
      is little room for ICA. Write the reasoning into REPORT.md. Knowing when a
      standard method does *not* apply is worth as much as knowing how to use it.

**Pitfalls**
* Filter **before** epoching, otherwise you get edge effects in every epoch.
* A high filter order can become unstable. Start with order 4.
* `sosfiltfilt` doubles the effective filter order.

**Done when:** you can go from a raw recording to a clean `(trials, channels, samples)`
array.

# Phase 3: The whole pipeline on a public dataset (the most important phase)

Before you trust your own Muse data, prove that the pipeline works on data where the
answer is known. Otherwise you will never know whether a bad result comes from your
code, your protocol or the hardware.

MNE ships a dataset: **EEG Motor Movement/Imagery** (PhysioNet, 109 subjects, 64
channels, left/right hand, both executed and imagined movement). CSP + LDA typically
reaches around 70:90 % on it.

**Read first**
* [07. CSP](topics/07-csp.md)
* [08. Classification](topics/08-classification.md)
* [09. Validation and statistics](topics/09-validation.md)

**Do**
* [ ] Download the dataset via `mne.datasets.eegbci`.
* [ ] Work through MNE's official CSP tutorial, but rewrite it in your own words and
      your own structure rather than copying it.
* [ ] Reproduce a reasonable accuracy (above 70 %) with CSP + LDA.
* [ ] Plot the CSP **patterns** as topographies. Do they sit over C3/C4? If so, you have
      found real motor cortex activity and you know what it should look like.
* [ ] Run a permutation test: shuffle the labels and retrain. The result should drop to
      around 50 %. If it does not, you have leakage in your pipeline.
* [ ] Compute the binomial 95 % chance bound for your trial count, so you know what
      number you actually have to beat.
* [ ] **Key experiment for the report:** rerun everything using only the channels that
      correspond to the Muse placement (TP9/TP10, frontal). How much does accuracy
      drop? That directly answers research question 2 in REPORT.md.
* [ ] Swap LDA for an SVM and compare. Remember the `StandardScaler`.
* [ ] Fix your epoch window, frequency band and `n_components` **here**, on public data,
      before touching your own recordings. Choosing them later by looking at your own
      cross-validation score invalidates that score.
* [ ] Optional but strong: try a Riemannian pipeline via `pyriemann` and compare.

**Done when:** you have a working, leakage-free pipeline and a number for what
Muse-like electrode placement costs.

# Phase 4: Own data, recording protocol and offline training

**Read first**
* [10. Experimental design](topics/10-experimental-design.md)
* Re-read [03. Artifacts](topics/03-artifacts.md), especially the control experiments

**Do**
* [ ] Decide and document the protocol in REPORT.md **before** you record anything.
* [ ] Design the cue so it does not induce lateralised eye movement. This is the design
      flaw most likely to invalidate your whole result.
* [ ] Write `scripts/record_session.py`: show the cue, insert the marker, record, save.
* [ ] **Start with motor execution**, i.e. actually clench your fist. Stronger signal,
      and if even that cannot be classified, something is broken in the chain.
* [ ] Record at least 40 trials per class, preferably 60, split into blocks with breaks.
* [ ] Run your Phase 3 pipeline on your own data, unchanged.
* [ ] Move on to motor imagery and compare.
* [ ] Run the artifact control experiments from REPORT.md section E.2. The narrowband
      8:13 Hz rerun is the cheapest and most informative; do that one first.
* [ ] Record a second session on a different day. Train on session 1, test on session 2.
      The difference from within-session CV is an important result.
* [ ] Keep a written session log: date, electrode issues, how you felt, anything odd.

**Pitfalls**
* If accuracy on Muse data looks suspiciously high (above 90 %), hunt for artifacts
  rather than celebrating.
* Fatigue degrades the signal, so split the recording into blocks with breaks.
* An accuracy of 58 % on 60 trials is not a result. Check it against the chance bound.

**Done when:** you have your own data, a trained model, and an honest number backed by a
permutation test and the artifact controls.

# Phase 5: Real time

**Read first**
* [11. Real-time processing](topics/11-realtime.md)

**Do**
* [ ] Re-epoch your training data into windows the same length as your runtime window,
      and retrain. Training and deployment must see the same kind of object.
* [ ] Decide how to handle the causal vs. zero-phase filter mismatch, and write down
      which you chose and why.
* [ ] Write the real-time loop: grab the last 2 s every 250 ms, run the pipeline, get a
      decision. Carry the filter state between blocks.
* [ ] Save the whole pipeline with `joblib` and use one shared preprocessing function
      for both offline and live code, so they cannot diverge.
* [ ] Use `predict_proba` rather than `predict`, so the system can say "don't know".
* [ ] Implement smoothing. Measure how often the decision flips while you rest.
* [ ] Measure actual latency and loop rate. Write it into REPORT.md D.3.
* [ ] Plot false-activation rate against response time for several smoothing settings.
* [ ] Build a neutral test display (a bar or a dot) before the game.

**Pitfalls**
* Real-time data must be processed **exactly** like the training data. This is the
  single most common real-time bug in BCI projects.
* `get_current_board_data(n)`, not `get_board_data()`, in the loop.

**Done when:** you can control a bar on screen with your head, at a known latency.

# Phase 6: The game

**Read first**
* [12. Game design for BCI](topics/12-game-design.md)

**Do**
* [ ] Choose the game. It must tolerate ~2.5 s latency, ~70 % accuracy and one binary
      axis. The auto-forward maze is probably the best fit.
* [ ] Define one input interface, then implement three sources behind it: keyboard,
      recorded-data replay, and live classifier. The game must never import BrainFlow.
* [ ] Build the game with **keyboard control first**. Make it fun without EEG.
* [ ] Add the replay source. It gives you a deterministic test of the whole live chain
      with no hardware, and it is the most useful debugging tool in the project.
* [ ] Switch in the live classifier.
* [ ] Add a calibration phase at the start that records training data.
* [ ] Display confidence to the player. This is required, not decorative.
* [ ] Log every session: time to target, path efficiency, false activations.

**Done when:** the game is playable with keyboard, with replay, and with your brain.

# Phase 7: Visualisation

**Read first**
* [13. Visualisation](topics/13-visualisation.md)

Most of these you will already have drawn while debugging. This phase is about turning
them into report-quality figures.

**Do**
* [ ] Real-time plot of the raw signal per channel.
* [ ] Real-time bars for band power (mu, beta) per channel.
* [ ] Power spectrum per class, with mu and beta marked.
* [ ] Time-frequency plot showing ERD across the trial, baseline-normalised.
      Do it for C3/C4 on the public data and TP9/TP10 on yours, side by side.
* [ ] CSP patterns (topography for the public data, bar chart for the Muse).
* [ ] Confusion matrix, learning curve, permutation null distribution.
* [ ] Go back and fix axis labels, units, event markers and colormaps on all of them.

# Phase 8: Report and wrap-up

* [ ] Lift lab-notebook entries up into the right chapters of REPORT.md.
* [ ] Make sure every number in the results table is reproducible with one command.
* [ ] Write the limitations chapter honestly. That is where you show you understand the
      method, and it is the part an employer reads most carefully.
* [ ] Clean up the repo, write run instructions, record a short demo video.
* [ ] Publish it publicly on GitHub.

# Glossary

| Term | Meaning |
|---|---|
| BCI | Brain-Computer Interface |
| ERD / ERS | Event-Related De-/Synchronization, power decrease/increase locked to an event |
| Mu rhythm | 8:13 Hz over sensorimotor cortex; attenuated by movement and imagined movement |
| CSP | Common Spatial Patterns, spatial filters maximising the variance difference between classes |
| LDA | Linear Discriminant Analysis |
| Epoch / trial | A time window cut out around an event |
| Cue | The signal telling the subject what to imagine |
| EOG / EMG | Eye and muscle artifacts respectively |
| Motor execution | Actual movement, as opposed to imagery |
| Contralateral | Opposite side; the right hand is controlled by the left motor cortex |
| Leakage | Test-set information influencing training, producing inflated accuracy |
| Volume conduction | The smearing of cortical sources across the scalp by skull and tissue |
| ITR | Information Transfer Rate, bits per minute, the standard BCI performance metric |
