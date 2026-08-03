# 13. Visualisation

**Needed in:** Phase 7
**In one sentence:** In EEG, plotting is not presentation, it is the primary debugging
tool, and most of the figures that end up in the report are ones you originally drew to
work out why something was broken.

## Why this matters here

Your README lists "visualisations of how the brain is used" as a feature. This document
turns that into specific plots. But the more important point is that you should be
plotting constantly from Phase 1 onward, not saving it for Phase 7. Numerical checks
tell you a value is wrong; a plot usually tells you *why*.

## The debugging plots (make these early)

### Raw time series

The most important plot in the project. Every channel, scrolling, in microvolts.

Use it to spot: flat channels, drifting channels, blinks, jaw clenches, 50 Hz buzz,
and the electrode that came loose halfway through the session. You should look at every
recording this way before analysing it, and it should take thirty seconds.

`raw.plot()` in MNE gives you an interactive scrollable viewer with sensible scaling.
Learn to use it early; it is much better than rolling your own with matplotlib.

### Power spectrum

Power against frequency, per channel, usually log-scaled on the y axis. Computed with
**Welch's method**: split the signal into overlapping segments, take the periodogram of
each, average them. Averaging reduces variance, which is why a Welch spectrum is smooth
and readable where a single FFT is noise.

The key parameter is `nperseg`, the segment length. It sets your frequency resolution
(`fs / nperseg`). At 256 Hz with `nperseg=256` you get 1 Hz resolution, which is fine
for identifying bands, and 1 s segments averaged over a long recording gives a clean
estimate.

What to look for:

* **An alpha/mu peak around 10 Hz.** If you see it, your electrodes are working and
  there is a rhythm to modulate. If there is no peak at all, that is a warning sign for
  the whole project (see BCI illiteracy in [02](02-motor-imagery-erd.md)).
* **A sharp 50 Hz spike.** Mains noise.
* **Power rising steadily above 20 Hz.** Muscle contamination
  ([03](03-artifacts.md)). Real EEG power falls off with frequency; EMG rises.
* **Huge low-frequency power.** Drift or eye artifacts.

Use it to verify your filters, too: plot the spectrum before and after filtering and
confirm the band you asked for is the band you got. Do this in Phase 2.

### Per-class spectra

The same plot, but averaged separately for left and right trials, overlaid.

This is where you *see* ERD directly, if it exists. A lateralised power difference in
the 8 to 13 Hz range is the effect. It is also your artifact check: any class difference
above 30 Hz is muscle, not brain.

If you cannot see any difference here, your classifier is unlikely to find one either,
and you have learned that before spending a day on cross-validation.

## The result plots

### Time-frequency / ERD

Power as a function of both time and frequency, across the trial, usually shown as a
heatmap with time on the x axis, frequency on the y, and colour as power change relative
to the pre-cue baseline.

This is the canonical motor imagery figure. Done right you should see a blue (power
decrease) patch in the 8 to 30 Hz range starting shortly after the cue, and possibly a
red patch (the beta rebound) after imagery ends.

Two things to get right:

* **Baseline-normalise.** Show power as a percentage change or decibel change relative
  to the pre-cue period, not absolute power. Absolute power is dominated by the 1/f
  background and you will see nothing.
* **Understand the time-frequency trade-off.** You cannot have arbitrarily good
  resolution in both time and frequency simultaneously; this is a mathematical
  constraint, not an implementation detail. Morlet wavelets handle it by using shorter
  windows at higher frequencies, which is why they are the default for this kind of
  plot. MNE's `tfr_morlet` / `compute_tfr` does this.

Compare the C3 and C4 channels on the public dataset in Phase 3, and TP9 vs TP10 on your
own data. The contrast between those two figures may be the most honest illustration of
the hardware limitation you can produce.

### CSP patterns

`csp.plot_patterns(epochs.info, ch_type='eeg')`. Topographic maps showing where each CSP
component's source projects onto the scalp.

**Plot patterns, not filters.** See [07](07-csp.md) for why this distinction matters.

On the 64-channel data in Phase 3, focal spots near C3 and C4 confirm the pipeline is
finding motor cortex. On 4-channel Muse data the topography will be nearly meaningless
(you cannot interpolate a scalp map from 4 points), so a bar chart of the four channel
weights is more honest than a smooth-looking topomap that implies spatial detail you do
not have. Resist the pretty version.

### Confusion matrix

`ConfusionMatrixDisplay.from_estimator()`. Shows which class is confused with which.
Reveals class bias that a single accuracy number hides.

### Learning curve

Accuracy against the number of training trials. Tells you whether collecting more data
would help, which directly informs how long your calibration phase should be. A curve
that has plateaued says more trials will not help; one still rising says your calibration
is too short.

`sklearn.model_selection.learning_curve` produces this.

### Permutation null distribution

A histogram of the permuted accuracies with your real accuracy marked. Visually
communicates significance far better than a p-value, and it makes leakage obvious if the
null is centred above chance. See [09](09-validation.md).

### Window sensitivity

Accuracy against epoch window position and length. Shows you picked a defensible window
rather than a lucky one, and reveals when the discriminative information actually
occurs.

## The real-time displays

Different constraints: these must be fast, and they exist for the player, not the report.

* **Rolling signal trace** per channel, a few seconds wide.
* **Band power bars**, mu and beta per channel, updated a few times a second.
* **Classifier confidence**, the smoothed probability, as a bar or coloured indicator.
  This one is required, for the reasons in [12](12-game-design.md).

For real-time plotting, matplotlib is slow. Options: draw them yourself in pygame using
`pygame.draw.lines` (simplest, since pygame is already running and it is just a
polyline), or use `blit`-based matplotlib animation. Given that pygame is already in
your stack, drawing directly is the path of least resistance.

## Making figures readable

Since these go in a report:

* **Label axes with units.** "Amplitude" is not a label; "Amplitude (µV)" is.
* **Mark the event.** A vertical line at t=0 on every time-resolved plot.
* **Shade the analysis window** so the reader sees what you actually classified.
* **Use consistent colours** for left and right across every figure in the report.
* **Do not use a rainbow colormap** for time-frequency plots. It creates visual
  boundaries that are not in the data. Use a perceptually uniform map (`viridis`), or a
  diverging one centred at zero (`RdBu_r`) when showing baseline-relative change, since
  then the sign matters and zero should be visually neutral.
* **Say what N is.** "Average of 47 trials after rejection" belongs in the caption.

## Common mistakes

* Leaving all plotting to Phase 7 instead of using it to debug from Phase 1.
* Plotting time-frequency power without baseline normalisation and seeing only 1/f.
* Plotting CSP filters instead of patterns.
* Interpolating a topographic map from 4 electrodes and reading spatial detail into it.
* Rainbow colormaps.
* Using matplotlib inside the real-time loop and wondering why the game stutters.
* Unlabelled axes and missing units.

## Check yourself

1. Why does Welch's method give a cleaner spectrum than a single FFT?
2. You plot time-frequency power and see only a smooth gradient with nothing
   event-related. What did you forget?
3. What in a power spectrum tells you that you have muscle contamination?
4. Why is a 4-channel topomap misleading?
5. What does a learning curve tell you about how long your game's calibration should be?
6. Why is a diverging colormap the right choice for baseline-relative power change?

## Sources

### Start here

* **MNE-Python, "Frequency and time-frequency sensor analysis"**,
  https://mne.tools/stable/auto_tutorials/time-freq/20_sensors_time_frequency.html
  Covers PSD, induced power, and ERD/ERS visualisation on real data. This is the
  tutorial that produces the figures you want.
* **MNE-Python, visualisation tutorials**,
  https://mne.tools/stable/auto_tutorials/index.html
  The "Visualization" section covers `raw.plot`, `plot_topomap`, and the interactive
  browsers. MNE's plotting is far better than anything you would write yourself for
  these purposes.
* **scipy.signal**, https://docs.scipy.org/doc/scipy/reference/signal.html
  `welch` and `spectrogram` docstrings, including what `nperseg` and `noverlap` actually
  control.

### Go deeper

* **Mike X Cohen, "Analyzing Neural Time Series Data"** (MIT Press, 2014). Chapters 10
  to 19 are the definitive treatment of time-frequency analysis for EEG: wavelets, the
  time-frequency trade-off, baseline normalisation methods, and how to choose between
  them. His free lecture series at https://www.youtube.com/@mikexcohen1 covers the same
  material with code.
* **matplotlib, "Choosing colormaps"**,
  https://matplotlib.org/stable/users/explain/colors/colormaps.html
  Why `viridis` exists and why jet does not belong in a scientific figure.

### Papers

* Pfurtscheller, G., & Lopes da Silva, F. H. (1999). "Event-related EEG/MEG
  synchronization and desynchronization: basic principles." *Clinical Neurophysiology*,
  110(11), 1842-1857. Includes the standard method for computing and plotting ERD as a
  percentage power change relative to baseline. This is the definition to follow.
* Grandchamp, R., & Delorme, A. (2011). "Single-trial normalization for event-related
  spectral decomposition reduces sensitivity to noisy trials." *Frontiers in
  Psychology*, 2, 236. On the choice of baseline normalisation method, which affects
  what your ERD figures look like more than people expect.
* Crameri, F., Shephard, G. E., & Heron, P. J. (2020). "The misuse of colour in science
  communication." *Nature Communications*, 11, 5444. The evidence behind avoiding
  rainbow colormaps.

### Video

* **Power Spectral Density & Top-Map Visualization: An Advanced MNE-Python Workflow
  for EEG Data**, https://www.youtube.com/watch?v=V3ZqiNGE7FE
  PSD computation and topographic maps in MNE, in code. Note that topomaps with four
  Muse electrodes are close to meaningless as spatial maps; the PSD half is what
  applies to your data.
