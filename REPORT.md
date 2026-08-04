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
   with a 4-channel consumer EEG (Muse 2), where the electrodes do not cover
   sensorimotor cortex?
2. How much worse is the result compared to the same pipeline run on a research
   dataset with electrodes over C3/C4?
3. How much of any performance comes from actual EEG, and how much from artifacts
   (eyes, jaw clenching, muscles)? Answered by measuring deliberate eye and jaw control
   through the identical pipeline, which gives the ceiling that non-neural control can
   reach on this hardware.
4. What latency and decision stability are needed for the game to feel controllable,
   and what does that cost in accuracy?
5. How does a non-neural control signal (eye movement, jaw clench) compare with the
   motor imagery signal on the same hardware, the same features and the same validation,
   in accuracy, latency and effort? This is what makes the game playable, and it is the
   reference point that makes question 3 answerable rather than rhetorical.
6. What does adding a single electrode over C3, through the Muse's auxiliary input,
   change? Answered by recording one session with the aux attached and analysing it
   twice, once with five channels and once with the aux dropped, so that the comparison
   is within session, within subject and within pipeline. This separates "motor imagery
   is not measurable at these electrode positions" from "motor imagery is not measurable
   here at all", which questions 1 to 5 cannot distinguish on their own.

> **On terminology:** eye and jaw control is **not** a BCI. Wolpaw's definition requires
> the system to measure central nervous system activity and not to depend on peripheral
> nerves and muscles. The eye/jaw condition is included as a measured comparison and as a
> working game input, and the report must say so plainly wherever its numbers appear.
> The BCI claim rests on the motor imagery condition alone.

# C. Method

Written so that someone else could reproduce the experiment.

## C.1 Equipment
Model, number of channels, sampling rate, reference, connection. If the auxiliary
electrode is used, state the electrode type, the paste, how the position was measured,
how the lead was fixed, and the BrainFlow preset used to enable the fifth channel.

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

## D.1 The ladder

Every row uses the **same** pipeline, features, cross-validation scheme and metrics.
Only the data changes. That is what makes the rows comparable, and comparability is the
entire point: no single number here means anything, the differences between them do.

Ordered from "no signal by construction" to "signal that is definitely present":

| # | Condition | Expected | Value | What it establishes |
|---|---|---|---|---|
| 0 | Chance level (2 balanced classes) | 50 % | | Reference point, not a threshold |
| 1 | Binomial 95 % upper bound for your N | | | The number you must actually beat |
| 2 | Permutation test on the real result | p | | Empirical significance, and a leakage detector |
| 3 | Motor imagery, public 64-ch dataset | 70 to 90 % | | The pipeline itself works |
| 4 | Same, restricted to Muse-like channels | | | What electrode placement costs, in points |
| 5 | Motor imagery, own Muse data | 50 to 65 % | | **The main result** |
| 5b | Same session, aux electrode at C3, left vs. right | | | What one motor-site electrode adds to the main result |
| 5c | Same session, aux at C3, imagery vs. rest | | | The task one central electrode can actually support |
| 6 | Motor execution, own Muse data | higher | | Recording and labelling chain works |
| 7 | Rest vs. rest, own Muse data | 50 % | | Should be chance. If not, the setup leaks |
| 8 | Deliberate eye/jaw control, own Muse | 95 %+ | | Ceiling of non-neural control on this hardware |

**How to read rows 5 and 8 together.** Row 8 is what the hardware achieves when the
signal is large, lateralised and unambiguous. If row 5 approaches row 8, the imagery
result is almost certainly artifacts wearing a costume, and the controls in E.2 should
find out which. If row 5 sits far below row 8 but above rows 0 to 2, that is a weak but
credible neural result and it is the honest outcome this project is most likely to have.

Row 8 is also the game's actual input source (see D.3 and section 12 of the topics).

**Rows 5b and 5c** exist only if Phase 4B is run, and they are compared against row 5
from the *same session*, not against the row 5 you already have. The recording is made
once with the aux electrode attached and analysed twice, with and without that channel,
so the only thing that differs between the two numbers is the electrode. Any comparison
across sessions is confounded by placement, impedance and day-to-day variation, which is
worth several points on its own. Row 5c changes the task as well as the montage, so it is
not directly comparable to row 5; it is a separate claim about what this hardware can
support, and it is the one most likely to come out positive. Method in
[topics/14-electrode-hardware.md](topics/14-electrode-hardware.md).

## D.2 Results table

| Experiment | Dataset | Features | Classifier | Accuracy (CV) | Comment |
|---|---|---|---|---|---|
| | | | | | |

## D.3 Real-time performance

Measured per input source, because they differ by an order of magnitude and quoting a
single figure would be misleading.

| Metric | Motor imagery | Eye/jaw |
|---|---|---|
| Window length / update interval | | |
| Latency from intention to decision | | |
| Decision stability (switches per second at rest) | | |
| False activations per minute during intentional rest | | |
| Information transfer rate (bits/min) | | |

Which source the demo video uses, and why, belongs in section F.

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

**The one thing that can be done about it.** The Muse has a single auxiliary EEG input
on the micro-USB port, so exactly one extra electrode can be added, at C3 or C4 but not
both. That rules out the standard C3 versus C4 lateralisation feature and leaves two
usable designs: left versus right with one motor site added (row 5b), and imagery versus
rest at C3, which a single channel supports well and which is still a BCI by Wolpaw's
definition (row 5c). Whether the modification helped is answered by rows 5b and 5c
against their same-session four-channel twin, and the cost side is answered by the noise
comparison in the next paragraph. Method and acceptance tests in
[topics/14-electrode-hardware.md](topics/14-electrode-hardware.md).

**What the modification costs.** A hand-placed electrode on an unshielded lead, with no
impedance readout and no mechanical support from the headband, is worse instrumentation
than the four built-in channels. Report the ratio of the 50 Hz peak and of the
low-frequency drift on the aux channel against TP9, and report how many trials the aux
channel alone caused to be rejected. A fifth channel with poor contact can lower
accuracy rather than raise it, and that outcome must be reported as readily as the other
one.

## E.2 Artifacts that could explain a suspiciously good result
The frontal electrodes (AF7/AF8) pick up eye movements and blinks; the temporal ones
(TP9/TP10) pick up the temporalis muscle (jaw clenching). If the left/right thought is
systematically followed by an eye movement or asymmetric muscle tension, the classifier
can learn *that* instead of the motor cortex.

This is not a hypothetical worry here, because row 8 of the ladder in D.1 measures
exactly how well those artifacts classify on this hardware. That number is the ceiling
the imagery result must be distinguished from.

Controls to run:
* Inspect which frequencies and channels the CSP filters weight most heavily
  (EMG above 30 Hz, EOG below 4 Hz).
* Compare class-average power spectra above 30 Hz. Any class difference there is muscle.
* Rerun the pipeline narrowband, 8 to 13 Hz only. Cheapest and most informative control,
  because there is very little EMG below 13 Hz. Do this one first.
* Compare results with and without artifact removal.
* Run a control session where eyes and jaw are deliberately kept still, with a central
  fixation cross and blinking restricted to the rest periods.
* Compare the imagery CSP patterns against the eye/jaw CSP patterns from row 8. If they
  weight the same channels in the same way, the two conditions are measuring the same
  thing and the imagery claim fails.

## E.3 How would better hardware improve the result?
The Muse has 4 channels; research equipment has 32 to 64, with electrodes over
C3/C4/Cz. To quantify this: run the same pipeline on a public 64-channel dataset, and
then on a subset of channels resembling the Muse placement. The difference is a direct
estimate of the importance of the hardware.

Three estimates of the same thing, from three directions, and the report is stronger for
having all of them:

* **Simulated**, from the public dataset restricted to Muse-like channels (row 4). Costs
  nothing, but it is a simulation of the placement and not of the amplifier.
* **Measured, one electrode**, from the auxiliary channel at C3 (rows 5b and 5c). Costs
  about 50 CAD and gives a real number on real hardware for a real motor site.
* **Not attempted here**: a multi-channel amplifier such as an OpenBCI Ganglion or
  Cyton, which BrainFlow supports through the same API and which would allow C3, C4 and
  Cz simultaneously. This is the correct answer to the placement problem and it costs one
  to two orders of magnitude more. State the estimated gain and leave it as future work.

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
