# 03. Artifacts

**Needed in:** Phase 0, Phase 2, Phase 4
**In one sentence:** Almost everything your electrodes pick up that is larger than the
EEG is not EEG, and on a 4-channel headset an artifact that correlates with your task
is the single most likely explanation for a good classification result.

## Why this matters here

In a normal EEG study artifacts are a nuisance. In this project they are an existential
threat to the conclusion. The Muse's electrodes sit directly over the two worst artifact
sources on the head: AF7 and AF8 are next to the eyes, TP9 and TP10 are on top of the
temporalis muscle. If "imagine left" is systematically accompanied by a glance or a jaw
twitch, your classifier will happily learn that instead, and you will report a BCI that
is really an eye tracker.

Treat this document as the checklist behind [REPORT.md](../REPORT.md) section E.2.

## The main artifact types

### EOG: eyes

Blinks and eye movements. The eyeball is an electrical dipole (cornea positive relative
to retina), so rotating it sweeps a large potential across the front of the head.

* **Amplitude:** 100 to 200 µV for a blink. Often 10x your EEG.
* **Frequency:** low, mostly below 4 Hz.
* **Topography:** largest frontally, decreasing towards the back, but present
  everywhere because the Muse's reference (FPz) is itself near the eyes.
* **Look like:** slow, smooth, high-amplitude deflections. Blinks are sharp and
  roughly symmetric; horizontal saccades produce opposite-sign steps on the left and
  right frontal channels.

**Why it is dangerous here:** horizontal eye movement is inherently left/right
lateralised. It is the perfect confound for a left/right classification task.

**Mitigation:** your 8 to 30 Hz bandpass removes most of the energy, which helps a lot.
But blinks are sharp enough to have some high-frequency content, so filtering is not a
complete solution. Instruct the subject to fixate on a central cross and blink only
during designated rest periods.

### EMG: muscles

Muscle action potentials, mostly from jaw (temporalis), neck, and forehead (frontalis).

* **Amplitude:** highly variable, can exceed 1000 µV with a real clench.
* **Frequency:** broadband, dominant above 20 Hz, extending well past 100 Hz.
* **Topography:** near the muscle. Temporalis is right under TP9 and TP10.
* **Look like:** dense, spiky high-frequency "fuzz" riding on the signal.

**Why it is dangerous here:** EMG spectral energy overlaps the top of your beta band
(20 to 30 Hz), so an 8 to 30 Hz bandpass does **not** remove it. Combined with TP9/TP10
placement, this is probably your highest-risk confound. Jaw tension can also be
asymmetric and unconscious.

**Mitigation:** the standard diagnostic is to check whether power increases *monotonically*
with frequency in the 20 to 40 Hz range. Real beta has a peak; EMG rises steadily.
Compare your two classes' spectra above 30 Hz: if they differ there, you have a muscle
problem, because there is little legitimate EEG at those frequencies at the scalp.

### Mains noise

50 Hz in Europe (60 Hz in North America), plus harmonics at 100 Hz, 150 Hz, and so on.

* **Look like:** a razor-sharp spike at exactly 50 Hz in the power spectrum.
* **Mitigation:** a notch filter, and physically moving away from chargers, laptops on
  mains power, and unshielded cabling. With a battery-powered Bluetooth headset it is
  usually less severe than with wired research systems, but check anyway.
* Because 50 Hz is outside your 8 to 30 Hz band, the bandpass already handles it. The
  notch filter is mostly for looking at raw data and for making sure nothing aliases.

### Electrode and movement artifacts

* **Poor contact:** high impedance produces drifting, noisy, or flat channels. On a dry
  electrode headset like the Muse this is the most common practical problem. Hair under
  TP9/TP10 is the usual culprit.
* **Cable/headset movement:** large step changes or slow drifts.
* **Sweat:** very slow (below 1 Hz) drift as skin conductance changes over a session.
* **Heartbeat (ECG):** a periodic ~1 Hz spike, more common on mastoid/temporal
  electrodes, so TP9/TP10 are candidates.

## Detection

In roughly the order you should implement them:

1. **Look at the raw data.** Non-negotiable. Plot every recording and scroll through it
   before you analyse anything. You will learn to recognise all of the above by eye in
   about an hour, and this skill saves entire weeks later.
2. **Per-channel statistics.** Compute standard deviation per channel per session. An
   outlier channel is a badly-seated electrode. Do this as an automatic check in
   Phase 1.
3. **Amplitude thresholding.** Reject epochs where any channel exceeds a threshold
   (±100 µV after filtering is a reasonable starting point). Simple, robust, and it
   will be enough for this project. **Log how many trials you reject and per class.**
   If you reject far more of one class than the other, that itself is a finding.
4. **Spectral checks.** Compare class-average power spectra above 30 Hz for EMG, and
   below 4 Hz for EOG.

## Rejection vs. correction

There are exactly two things you can do about a contaminated trial, and they are
different in kind. Getting the vocabulary right matters because the report has to say
which one you chose and why.

* **Rejection** throws the contaminated data away. You lose trials, your statistical
  power drops, and if the contamination is not equally likely in both classes you have
  also introduced a bias. What you keep is clean and unaltered.
* **Correction** keeps the trial and subtracts an estimate of the artifact from it. You
  keep your trial count, but every remaining sample is now partly a model output rather
  than a measurement, and if the estimate is wrong you have distorted your data in a way
  that is invisible afterwards.

The argument for correction is that rejection can bite hardest exactly where you can
least afford it: if a subject blinks more during one condition, rejection removes more of
that condition. The argument for rejection is that it cannot manufacture an effect, and
that a correction you cannot verify is worse than a smaller dataset. With few channels,
the second argument wins decisively.

### Why correction is not available to you: ICA

**ICA** (Independent Component Analysis) is the standard correction tool. It decomposes
the signal into statistically independent components, you identify the ones that look
like eyes or muscle, and you reconstruct the data without them.

The catch: ICA can extract at most as many components as you have channels. With
**4 channels you get 4 components**, and each will be a blurry mixture of brain and
artifact. Removing one throws away a quarter of your data along with the artifact. ICA
is a technique for 32+ channel systems.

**For this project: use rejection, not correction.** Threshold-reject bad epochs and
accept that you lose trials, and log the counts per class so the bias is visible rather
than hidden. It is more defensible than pretending 4-channel ICA cleaned anything. Read
about ICA anyway, understand why it does not apply here, and write that reasoning into
the report. Recognising when a standard technique does *not* fit is exactly the kind of
judgement the report should demonstrate.

## Where cleaning sits in the pipeline

Order matters, and two of these are commonly got wrong.

1. **High-pass or bandpass the continuous data first.** Drift is what makes amplitude
   thresholds fire on data that is otherwise fine, so thresholding unfiltered data mostly
   detects drift rather than the artifacts that will affect your features.
2. **Notch out mains** if you are going to look at raw data or anything above 30 Hz.
3. **Identify bad channels** across the whole recording, before epoching. A channel that
   never made contact is a property of the session, not of individual trials, and
   catching it early stops it silently poisoning every epoch. On four channels you cannot
   interpolate a bad channel from its neighbours the way a 64-channel montage would, so
   a dead channel means either dropping it for the whole session (and saying so) or
   re-recording.
4. **Epoch**, then **reject epochs by amplitude**.
5. Only then extract features.

The one that catches people: filtering must precede epoching, for the edge-transient
reason in [05](05-digital-filtering.md), *and* it must precede rejection, for the drift
reason above. Both point the same way, so there is a single correct order.

## The control experiments

These are what turn "I got 68 % accuracy" into a credible claim. Run them in Phase 4.

| Control | What it tests |
|---|---|
| Inspect CSP filter weights and their frequency response | If the discriminative energy is above 30 Hz it is EMG; below 4 Hz it is EOG |
| Compare class spectra above 30 Hz | Any class difference here is muscle, not brain |
| Rerun the pipeline with a narrower band, e.g. 8 to 13 Hz only | If accuracy survives, EMG cannot explain it, because there is very little EMG below 13 Hz |
| Deliberate-artifact session: blink or clench in time with the cues | Establishes the *upper bound* of what artifacts alone can achieve. If this scores about the same as your real result, your real result is artifacts |
| Rest/control condition with no imagery | Should classify at chance. If it does not, something in your setup leaks |
| Fixation-enforced session | Central cross, no blinking during trials |

The narrowband rerun is cheap and powerful. Do it early.

## Common mistakes

* Never looking at the raw data.
* Assuming the 8 to 30 Hz bandpass removed the artifacts. It removes EOG and mains; it
  does **not** remove EMG.
* Applying 4-channel ICA and believing the result.
* Celebrating high accuracy instead of investigating it. In this project, above 90 % on
  imagery is a red flag, not a success.
* Rejecting epochs without recording how many and from which class.
* Cleaning artifacts differently in training and in real time, so the classifier sees a
  different data distribution when you actually play.

## Questions

Write your answers in the boxes. See
[the convention](README.md#answering-the-questions).

**Q1.** Why does an 8 to 30 Hz bandpass filter fail to remove jaw EMG?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q2.** You see a sharp peak at exactly 50 Hz. What is it, and does it threaten your
result?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q3.** Why is horizontal eye movement a particularly bad confound for *this*
classification task specifically?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q4.** Why is ICA a poor fit for a 4-channel headset?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q5.** Design an experiment that would prove your classifier is reading muscles rather
than brain.
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q6.** Real beta and EMG both put power in the 20 to 30 Hz range. What is the shape
difference between them in a power spectrum, and how do you use it as a diagnostic?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q7.** MNE's ICA tutorial recommends high-pass filtering at around 1 Hz before fitting
ICA, higher than you would use for the analysis itself. What is the reason, and what do
you do with the ICA solution afterwards so the analysis is not stuck with a 1 Hz
high-pass?
*Source: MNE, "Repairing artifacts with ICA". `reviewed: no`*

> **Answer:** _(unanswered)_

**Q8.** The same tutorial uses `find_bads_eog` to identify eye components automatically.
What does that function need in the recording, and why can you not use it on Muse data?
*Source: MNE, "Repairing artifacts with ICA". `reviewed: no`*

> **Answer:** _(unanswered)_

**Q9.** State the difference between artifact *rejection* and artifact *correction*, give
the argument in favour of each, and say which one this project uses and why the four
channels settle the question.
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q10.** Goncharova et al. recorded EEG while subjects deliberately contracted specific
muscles. What was the spectral shape of temporalis EMG, and how far across the scalp did
it spread? Given where TP9 and TP10 sit, what does their result predict for a jaw clench
on your headband, and what does it predict for the frontalis and neck muscles?
*Source: Goncharova et al. (2003). `reviewed: no`*

> **Answer:** _(unanswered)_

**Q11.** Muthukumaraswamy argues that a large fraction of published high-frequency
findings are muscle rather than brain. What is his central recommendation for telling
them apart, and what does it imply about ever reporting anything above 30 Hz from a Muse?
*Source: Muthukumaraswamy (2013). `reviewed: no`*

> **Answer:** _(unanswered)_

## Sources

Artifacts are the existential threat to this project's conclusion, so **Tier 1** here has
a paper in it. Read Goncharova before your first Phase 4 recording, not after you have a
result you want to believe.

### Tier 1

* **MNE-Python, "Overview of artifact detection"**,
  https://mne.tools/stable/auto_tutorials/preprocessing/10_preprocessing_overview.html
  Shows what each artifact type looks like in real data. This is largely a visual skill,
  so the figures are the point. Half an hour, and it pays for itself the first time you
  scroll through your own recording.
* **Goncharova, I. I., McFarland, D. J., Vaughan, T. M., & Wolpaw, J. R. (2003). "EMG
  contamination of EEG: spectral and topographical characteristics."** *Clinical
  Neurophysiology*, 114(9), 1580-1593. Written by the BCI2000 group specifically because
  EMG kept faking BCI results, and it measures exactly the muscles that sit under your
  electrodes. This is the most directly relevant paper in the entire topic folder to the
  hardware you own.

### Tier 2

* **MNE-Python, "Repairing artifacts with ICA"**,
  https://mne.tools/stable/auto_tutorials/preprocessing/40_artifact_correction_ica.html
  You are not going to use ICA here. Read it anyway, once, to see how many channels it
  assumes and to be able to say precisely why it does not apply, which is a paragraph the
  report needs.
* **Muthukumaraswamy, S. D. (2013). "High-frequency brain activity and muscle artifacts
  in MEG/EEG: a review and recommendations."** *Frontiers in Human Neuroscience*, 7, 138.
  Why EMG contaminates the beta and gamma bands and how to detect it. Open it when you
  are writing the artifact-control section of the report and want the general statement
  rather than the Muse-specific one.
