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

## Check yourself

1. You epoch -0.5 to 3.5 s at 256 Hz. What is the shape of one epoch?
2. Why is including the cue onset in the analysis window a problem for a left/right
   task specifically?
3. Why does baseline correction matter less for CSP than for ERP analysis?
4. You delivered 80 cues but `X.shape[0]` is 78. Name two possible causes.
5. Your rejection step removes 25 % of one class and 4 % of the other. What are the two
   separate problems this creates?

## Sources

### Start here

* **MNE-Python, "The Epochs data structure"**,
  https://mne.tools/stable/auto_tutorials/epochs/10_epochs_overview.html
  How epochs are constructed, indexed, cropped, and how rejection and metadata work.
  The `Epochs` object handles a lot of the bookkeeping that is easy to get wrong by hand.
* **MNE-Python, "Decoding motor imagery with CSP"**,
  https://mne.tools/stable/auto_examples/decoding/decoding_csp_eeg.html
  See the concrete `tmin`/`tmax` choices made for exactly this paradigm, and the
  reasoning in the comments.

### Go deeper

* **Steven Luck, "An Introduction to the Event-Related Potential Technique"**
  (MIT Press). The chapters on epoching, baseline correction and artifact rejection are
  the standard treatment. Written for ERPs, but the epoching logic is identical and
  Luck is unusually explicit about the ways it goes wrong.
* **MNE-Python, "Parsing events from raw data"**,
  https://mne.tools/stable/auto_tutorials/intro/20_events_from_raw.html
  How to get from annotations or a marker channel to the events array.

### Papers

* Tanner, D., Morgan-Short, K., & Luck, S. J. (2015). "How inappropriate high-pass
  filters can produce artifactual effects and incorrect conclusions in ERP studies."
  *Psychophysiology*, 52(8), 997-1009. On the interaction between filtering and
  epoching, and why order of operations is not a detail.
* Delorme, A., Sejnowski, T., & Makeig, S. (2007). "Enhanced detection of artifacts in
  EEG data using higher-order statistics and independent component analysis."
  *NeuroImage*, 34(4), 1443-1449. On automated epoch rejection criteria.

### Video

* **MNE-Python Tutorial: Epoching and Evoked Responses**,
  https://www.youtube.com/watch?v=eJmO-A1Kx6Q
  Events, the `Epochs` object and averaging into an evoked response, demonstrated in
  code. Your project averages band power rather than voltage, but the epoching
  machinery is exactly the same.
