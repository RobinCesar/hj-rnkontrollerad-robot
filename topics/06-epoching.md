# 06. Epoching

**Needed in:** Phase 2
**In one sentence:** Epoching turns one long continuous recording into a stack of
labelled trials, and choosing the time window is a real experimental decision that
directly determines whether your effect is visible.

## Why this matters here

Everything downstream operates on epochs. CSP needs `(trials, channels, samples)`,
sklearn needs one row per trial, and the labels only exist because you cut the data at
the right places. Epoching is also where a timing error from
[04](04-data-acquisition.md) turns into a silent, permanent misalignment.

## The transformation

```
continuous:  (n_channels, n_samples)          e.g. (4, 153600)  = 10 minutes at 256 Hz
    +  events: (n_events, 3)                  e.g. (80, 3)
    +  a time window (tmin, tmax)
    ->
epoched:     (n_trials, n_channels, n_samples)  e.g. (80, 4, 1280)
    +  labels y: (n_trials,)
```

`n_samples` per epoch is `(tmax - tmin) * sampling_rate`. A window from -1.0 to +4.0 s
at 256 Hz gives 1280 samples.

## The event array

MNE's convention is an `(n_events, 3)` integer array:

| Column | Meaning |
|---|---|
| 0 | Sample index where the event occurred |
| 1 | Previous event value (legacy, usually 0) |
| 2 | Event ID, i.e. which class |

Note it is **sample index**, not time in seconds. Converting from your marker
timestamps to sample indices is where off-by-a-lot errors creep in. If you have event
times in seconds and a timestamp array, `np.searchsorted(timestamps, cue_time)` gives
you the index.

## Choosing tmin and tmax

This is the decision that matters. Consider a Graz-style trial: cue at t=0, subject
imagines for 4 s, then rest.

You want the window to contain the ERD and as little else as possible.

* **Start too early (t=0):** you include the moment the cue appeared. The cue is a
  visual stimulus, so it produces a visual evoked response, and the subject takes
  several hundred milliseconds to start imagining anything. Both are noise for your
  purpose. Worse, if the cue arrow points left or right, the *visual* response differs
  between your two classes, which is a confound that has nothing to do with motor
  cortex.
* **Start too late:** you miss the ERD onset, which is often strongest shortly after
  imagery begins.
* **End too late:** you catch the post-movement beta rebound and the return to rest,
  diluting the effect. Though note the rebound is itself informative, so this is a
  genuine trade-off worth testing rather than assuming.

**A common choice for motor imagery is 0.5 to 3.5 s after the cue.** That is where the
MNE tutorial lands, and it is a defensible default. But do not treat it as fixed:
**sweep the window and report the sensitivity**. Running your pipeline at several
windows and plotting accuracy against window position is a good, cheap figure for the
report, and it tells you something real about your own data.

**Practical tip:** epoch generously (say -1.0 to 4.0 s), then use `epochs.copy().crop()`
to test narrower analysis windows. That way you can explore without re-epoching, and you
keep the pre-cue baseline period available.

## Baseline correction

Subtracting the mean of a pre-cue rest period from the whole epoch, per channel.

* **Purpose:** removes slow drift and the arbitrary DC offset each channel carries, so
  trials become comparable.
* **Typical baseline:** the second before the cue, `baseline=(-1.0, 0.0)`.
* **Caveat:** the baseline period must genuinely be rest. If your inter-trial interval
  is too short, the previous trial's beta rebound is still active and you are
  subtracting signal. This is one reason varied, adequate rest periods matter in
  [10](10-experimental-design.md).
* **For CSP specifically**, baseline correction matters less than it does for ERP
  analysis, because CSP works on *variance* within the band, and subtracting a constant
  does not change variance. It still helps with drift. Do it, but understand why its
  effect is smaller here.

There is also **spectral** baselining, expressing power as a change relative to a rest
period. That is the right approach for the ERD *visualisations* in
[13](13-visualisation.md), where you want to show a percentage power decrease rather
than absolute power.

### How long should the baseline be

Both directions are wrong, so this is a real choice rather than a default.

* **Too short** and the baseline estimate is itself noisy. You are subtracting a number
  computed from a handful of samples, and that noise gets added to every point in the
  epoch. Worse, it is *correlated* noise: the same erroneous value is subtracted from the
  whole trial, which shifts the entire epoch rather than jittering it. For spectral
  baselining the problem is sharper still, because estimating power in an 8 to 13 Hz band
  needs at least a few cycles of that band, so a 200 ms baseline cannot estimate mu power
  at all. Half a second is a floor; a full second is safer.
* **Too long** and you run out of genuine rest to sit in. The baseline has to be a period
  where nothing task-related is happening, and in a Graz-style trial the only such window
  is between the previous trial's activity dying down and the current trial's cue.

This is exactly where the epoching decision collides with the protocol design in
[10](10-experimental-design.md). The chain runs: the beta rebound after the previous
trial lasts about a second, you want a baseline of about a second, and you want a margin
so that anticipation of the next cue does not creep into it. That is what forces the rest
period to be three to five seconds rather than two, and it is why the rest duration is
varied rather than fixed. If you shorten the rest to fit more trials into the session,
you are not saving time, you are contaminating your baseline with the previous trial.

## Rejection

After epoching, drop trials contaminated by artifacts. See [03](03-artifacts.md) for
the methods. Two things to get right here:

1. **Reject after filtering, before CSP.** A threshold on unfiltered data mostly finds
   drift; a threshold on filtered data finds what will actually affect your features.
2. **Log the rejection counts per class.** If you reject 20 % of left trials and 5 % of
   right trials, you have introduced a class imbalance *and* revealed that something
   systematically differs between conditions. Both belong in the report.

Also drop the corresponding labels. Keeping `X` and `y` aligned through rejection is a
classic source of silent bugs; if you use MNE's `Epochs` object it handles this for you,
which is a good reason to use it rather than rolling your own arrays.

## Class balance

Aim for equal numbers of trials per class. Unequal classes make accuracy misleading
(90 % accuracy is unimpressive if 90 % of trials are one class) and bias most
classifiers towards the majority.

If rejection leaves you imbalanced, either subsample the majority class or use
`balanced_accuracy` as your metric. Say which you did.

## Common mistakes

* Epoching before filtering, so every trial has filter edge transients. See
  [05](05-digital-filtering.md).
* Using time in seconds where a sample index is expected, or vice versa.
* Including the cue onset in the analysis window, letting the visual response
  differentiate the classes.
* Baselining against a period that is not actually rest.
* Rejecting epochs from `X` but forgetting to drop the same entries from `y`.
* Picking a window once and never testing whether it was a good choice.
* Not checking `X.shape` after epoching. The number of trials should equal the number
  of cues you delivered, minus rejections. If it does not, stop and find out why.

## Questions

Write your answers in the boxes. See
[the convention](README.md#answering-the-questions).

**Q1.** You epoch -0.5 to 3.5 s at 256 Hz. What is the shape of one epoch?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q2.** Why is including the cue onset in the analysis window a problem for a left/right
task specifically?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q3.** Why does baseline correction matter less for CSP than for ERP analysis?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q4.** You delivered 80 cues but `X.shape[0]` is 78. Name two possible causes.
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q5.** Your rejection step removes 25 % of one class and 4 % of the other. What are the
two separate problems this creates?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q6.** The recommended approach is to epoch generously and crop afterwards rather than
epoching straight to the analysis window. What does that buy you?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q7.** In MNE's `Epochs`, what is the difference between `reject`, `flat` and
`drop_bad()`, and how do you find out afterwards *why* a particular epoch was dropped?
*Source: MNE, "The Epochs data structure". `reviewed: no`*

> **Answer:** _(unanswered)_

**Q8.** What does `epochs.get_data()` return, in shape and in units? What does
`preload=True` change about when the data is actually read?
*Source: MNE, "The Epochs data structure". `reviewed: no`*

> **Answer:** _(unanswered)_

**Q9.** MNE has two different ways of representing events: `Annotations` and the
`(n_events, 3)` events array. What is the difference, and which functions convert
between them?
*Source: MNE, "Parsing events from raw data". `reviewed: no`*

> **Answer:** _(unanswered)_

**Q10.** MNE's CSP decoding example constructs its `Epochs` with `baseline=None`. Given
what this document says about baseline correction and variance, why is that a defensible
choice there?
*Source: MNE, "Decoding motor imagery with CSP". `reviewed: no`*

> **Answer:** _(unanswered)_

**Q11.** A baseline period that is too short causes problems of its own. What goes wrong,
why is the problem worse for *spectral* baselining than for subtracting a mean, and how
does that constrain the rest period in your Phase 4 protocol?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

## Sources

There is no paper worth reading for this topic. Epoching is bookkeeping: the concepts are
all above, and the rest is knowing what MNE's `Epochs` object does for you. Read Tier 1
with the code open and skip everything else.

### Tier 1

* **MNE-Python, "The Epochs data structure"**,
  https://mne.tools/stable/auto_tutorials/epochs/10_epochs_overview.html
  How epochs are constructed, indexed, cropped, and how rejection and metadata work.
  The `Epochs` object handles a lot of the bookkeeping that is easy to get wrong by hand,
  above all keeping `X` and `y` aligned through rejection.
* **MNE-Python, "Decoding motor imagery with CSP"**,
  https://mne.tools/stable/auto_examples/decoding/decoding_csp_eeg.html
  The same example as in [02](02-motor-imagery-erd.md), read for a different reason here:
  the concrete `tmin`/`tmax` choices made for exactly this paradigm, and the reasoning in
  the comments.

### Tier 2

* **MNE-Python, "Parsing events from raw data"**,
  https://mne.tools/stable/auto_tutorials/intro/20_events_from_raw.html
  How to get from annotations or a marker channel to the events array. You need this the
  day you convert BrainFlow's marker row into something MNE will accept, and not before.
