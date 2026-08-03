# Report and lab notebook

This is where I collect the problems and solutions, limitations and results I run into
during the project, as material for the final report.

**How to use it:** the lab notebook (part A) is filled in continuously, newest entry
first. Parts B to F are the skeleton of the finished report and are filled in
afterwards, by lifting up whatever turned out to matter from the notebook.

# A. Lab notebook

Format per entry, keep it short:

```
## YYYY-MM-DD  Title
**What I tried:**
**What happened:**
**Why (hypothesis / cause):**
**Fix / next step:**
```

<!-- New entries go directly below this line -->

# B. Research questions

What the report should answer. Phrase them so they can be answered with a number or a
clear yes/no.

1. Is it possible to classify motor imagery (left vs. right hand) above chance level
   with a 4-channel consumer EEG (Muse), where the electrodes do not cover
   sensorimotor cortex?
2. How much worse is the result compared to the same pipeline run on a research
   dataset with electrodes over C3/C4?
3. How much of any performance comes from actual EEG, and how much from artifacts
   (eyes, jaw clenching, muscles)?
4. What latency and decision stability are needed for the game to feel controllable,
   and what does that cost in accuracy?

# C. Method

Written so that someone else could reproduce the experiment.

## C.1 Equipment
Model, number of channels, sampling rate, reference, connection.

## C.2 Experimental protocol (cue paradigm)
The timeline of a single trial, number of trials per class, number of sessions, breaks.
Reference: the Graz paradigm, i.e. fixation cross, cue (arrow), imagined movement, rest.

| Parameter | Value | Rationale |
|---|---|---|
| Trials per class | | |
| Trial length | | |
| Rest between trials | | |
| Number of sessions | | |

## C.3 Signal processing
Filter bands and order, notch, artifact method, epoch window relative to the cue,
resampling, referencing.

## C.4 Feature extraction
CSP: number of components, how the feature is computed (log-variance), regularisation.

## C.5 Classification and evaluation
Classifier, cross-validation scheme, metrics, how data leakage is avoided.

> **Critical methodological point:** the filter, CSP and classifier must all be fitted
> *inside* each CV fold. If CSP is fitted on all the data before cross-validation, the
> result will look too good and the report is worthless. Document how this is ensured.

# D. Results

## D.1 Baselines to compare against
Without these, no result can be interpreted.

| Baseline | Value | Comment |
|---|---|---|
| Chance level (2 classes) | 50 % | |
| Upper bound of chance (95 % CI, given N trials) | | Computed from the binomial distribution |
| Permutation test (p-value) | | Labels shuffled, model retrained |
| Same pipeline on a public dataset | | Shows the pipeline itself works |
| Own Muse data, motor *execution* | | Easier task, sanity check |
| Own Muse data, motor *imagery* | | The main result |

## D.2 Results table

| Experiment | Dataset | Features | Classifier | Accuracy (CV) | Comment |
|---|---|---|---|---|---|
| | | | | | |

## D.3 Real-time performance

| Metric | Value |
|---|---|
| Window length / update interval | |
| Latency from cue to decision | |
| Decision stability (switches per second at rest) | |

## D.4 Figures
* CSP patterns (topography or channel weights)
* Power spectrum per class, with the mu and beta bands marked
* Time-frequency / ERD curve across the trial
* Confusion matrix
* Accuracy as a function of the number of training trials (learning curve)

# E. Limitations

## E.1 Electrode placement
The Muse has 4 channels: TP9, AF7, AF8, TP10. Motor imagery produces ERD
(event-related desynchronization) in the mu (8:13 Hz) and beta (13:30 Hz) bands,
strongest over C3 (right hand) and C4 (left hand). None of the Muse electrodes sit
there. What might still be picked up is weak volume-conducted activity in TP9/TP10.

The consequence to investigate and report: how much of the performance ceiling is set
by the hardware rather than by the method.

## E.2 Artifacts that could explain a suspiciously good result
The frontal electrodes (AF7/AF8) pick up eye movements and blinks; the temporal ones
(TP9/TP10) pick up the temporalis muscle (jaw clenching). If the left/right thought is
systematically followed by an eye movement or asymmetric muscle tension, the classifier
can learn *that* instead of the motor cortex.

Controls to run:
* Inspect which frequencies and channels the CSP filters weight most heavily
  (EMG above 30 Hz, EOG below 4 Hz).
* Compare results with and without artifact removal.
* Run a control session where eyes and jaw are deliberately kept still.

## E.3 How would better hardware improve the result?
The Muse has 4 channels; research equipment has 32 to 64, with electrodes over
C3/C4/Cz. To quantify this: run the same pipeline on a public 64-channel dataset, and
then on a subset of channels resembling the Muse placement. The difference is a direct
estimate of the importance of the hardware.

## E.4 Other limitations
* One subject, so the results do not generalise.
* Session variability: electrode placement and impedance differ between days. Test
  training on one session and evaluating on another.
* "BCI illiteracy": a fraction of the population produces a weak motor imagery signal
  even with good equipment.

# F. Discussion and conclusion

What worked, what did not, what was surprising, and what the next version of the
project should do differently.

# G. References

| # | Source | Used for |
|---|---|---|
| | | |
