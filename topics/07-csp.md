# 07. CSP: Common Spatial Patterns

**Needed in:** Phase 3
**In one sentence:** CSP learns a set of channel weightings that make one class's
signal as variable as possible while making the other class's signal as flat as
possible, turning a spatial difference into a simple number you can classify.

## Why this matters here

CSP is the one genuinely non-trivial algorithm in the project, and it is the standard
method for motor imagery specifically because motor imagery is a **spatial** problem
(see [02](02-motor-imagery-erd.md)): the same mu/beta power drop appears at C3 or at C4
depending on the class. CSP is purpose-built to find exactly that kind of difference.

## The problem it solves

You have 4 channels by 1000 samples per trial. You need a handful of numbers per trial
to feed a classifier. The obvious approach, band power per channel, mostly fails,
because volume conduction ([01](01-eeg-basics.md)) means every channel sees a mixture of
every source. The discriminative source is smeared across all your channels, and none of
them cleanly represents it.

A **spatial filter** is a weighted sum of channels:

```
s(t) = w1*ch1(t) + w2*ch2(t) + w3*ch3(t) + w4*ch4(t)
```

By choosing the weights well you can partially undo the mixing and recover something
closer to the underlying source. CSP is a method for choosing those weights, and
crucially it chooses them **using the class labels**, so they are optimised for
discrimination rather than for anything else.

## The core idea

After bandpass filtering to 8 to 30 Hz, "power in the mu/beta band" is just **variance
over time**. So ERD, a power decrease, is a variance decrease.

CSP finds spatial filters `w` that maximise the ratio:

```
        variance of w-filtered signal for class 1
  J(w) = -----------------------------------------
        variance of w-filtered signal for class 2
```

The first filter maximises this ratio, giving a signal with high variance for left-hand
trials and low variance for right-hand trials. The last filter minimises it, doing the
opposite. Intermediate filters are less useful, which is why you keep only the extremes.

`n_components=4` means "keep the 2 most extreme filters from each end".

Mechanically, this is solved as a **generalised eigenvalue problem** on the two classes'
covariance matrices. You compute the average spatial covariance matrix for each class,
then simultaneously diagonalise them. The eigenvalues are the variance ratios and the
eigenvectors are the filters. This is worth reading through once, because it explains
several practical properties: why CSP needs enough trials to estimate covariance
reliably, why it is sensitive to outliers (one huge artifact trial dominates the
covariance estimate), and why regularisation helps.

## From filters to features

For each trial:

1. Apply the spatial filters: `(n_channels, n_samples) -> (n_components, n_samples)`
2. Compute the variance over time of each component: `(n_components,)`
3. Normalise by the total variance across components
4. Take the log

The log matters. Variance values are strongly skewed and span orders of magnitude;
`log` makes the distribution roughly Gaussian, which is exactly what LDA assumes
([08](08-classification.md)). `log=True` in MNE's `CSP` does this for you.

Result: `(n_trials, n_components)`, a small, well-behaved feature matrix. This is what
goes into the classifier.

## Filters vs. patterns (the distinction people get wrong)

CSP produces two related matrices, and they mean different things:

* **Filters** (`csp.filters_`) are what you multiply the data by to extract a component.
  Use them for computation.
* **Patterns** (`csp.patterns_`) describe how the extracted source projects back onto
  the scalp. Use them for **interpretation and plotting**.

Filter weights are *not* interpretable as "where the activity is". A filter can put a
large weight on a channel purely to subtract out noise from it. Plotting filters and
concluding "the signal is at TP9" is a real and common error.

`csp.plot_patterns()` is the correct call for your report figure. Haufe et al. (2014)
is the paper that established this distinction; it is worth knowing about because the
same mistake appears throughout the decoding literature.

**In Phase 3, plot the patterns on the 64-channel data.** If they show focal spots near
C3 and C4, you have confirmed that your pipeline is finding real motor cortex activity.
That single figure validates everything upstream of it.

## Why bandpass filtering must come first

CSP maximises a variance ratio, and variance is total power across whatever
frequencies are present. If you feed CSP broadband data, it will happily find the
spatial pattern that best separates the classes using **any** frequency, including the
0.5 Hz drift or the 45 Hz EMG. The 8 to 30 Hz filter is what makes "variance" mean
"mu and beta power" rather than "power in general".

This also means CSP is only as good as your band choice. Standard extensions
(Filter Bank CSP) run CSP in several sub-bands and let the classifier pick, which
handles subject-to-subject variation in peak mu frequency. That is a good extension for
this project if the basic version works.

## Practical parameters

| Parameter | Guidance |
|---|---|
| `n_components` | 4 or 6 with many channels. **With Muse's 4 channels you have at most 4 filters total**, so `n_components=2` or `4` |
| `log` | `True`, for the reason above |
| `reg` | Regularisation. `None` to start; `'ledoit_wolf'` or `'oas'` if you have few trials or unstable results |
| `transform_into` | `'average_power'` (default) gives one number per component per trial. `'csp_space'` gives the filtered time series, which you need if you want time-resolved output |

## The limitation for this project

CSP separates sources by exploiting differences between channels. With 4 channels you
have a 4-dimensional space to work in, and at most 4 spatial filters. Research systems
have 64 channels and 64 filters, and can isolate a source at C3 quite precisely.

You cannot spatially separate a source you have no coverage of. **Expect CSP to
underperform badly on Muse data**, and expect the Phase 3 channel-subset experiment
(rerunning the 64-channel dataset using only Muse-like positions) to show you roughly
how much you lose. That experiment is the honest way to quantify the limitation, and it
is more informative than any amount of tuning.

## Modern alternatives worth knowing

CSP is from 1990 to 2000 and is still the standard baseline, but the field has moved:

* **Riemannian geometry methods** (tangent space classification of covariance matrices)
  now generally outperform CSP, are more robust with few trials, and need less tuning.
  The `pyriemann` library implements them with a scikit-learn API, so swapping it in is
  a small change once your pipeline exists. This is an excellent Phase 3 extension and
  it would strengthen the report considerably.
* **FBCSP** (Filter Bank CSP), CSP across multiple frequency bands, won BCI Competition IV.
* **Deep learning** (EEGNet, ShallowConvNet) works but needs far more data than you will
  have.

Do CSP first. It is the baseline everything else is compared against, and you need to
understand it to understand why the alternatives are better.

## Common mistakes

* Not bandpass filtering first, so CSP optimises on drift or EMG.
* Fitting CSP on all the data before cross-validation. **This is data leakage** and it
  will inflate your accuracy substantially. CSP uses the labels, so it must be inside
  the sklearn `Pipeline`. See [09](09-validation.md).
* Plotting filters instead of patterns and misinterpreting where the activity is.
* Requesting more components than you have channels.
* Forgetting `log=True` and feeding raw, skewed variances to LDA.
* Not checking for outlier trials before fitting. One artifact-laden trial can dominate
  the covariance estimate and wreck the filters.

## Questions

Write your answers in the boxes. See
[the convention](README.md#answering-the-questions).

**Q1.** Why is variance the right feature after bandpass filtering to 8 to 30 Hz?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q2.** What would CSP latch onto if you gave it unfiltered data?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q3.** What is the difference between a CSP filter and a CSP pattern, and which do you
plot?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q4.** Why must CSP be fitted inside the cross-validation loop?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q5.** Why does the log transform help LDA specifically?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q6.** You have 4 channels. What is the maximum number of CSP components, and why?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q7.** CSP keeps the most extreme filters from each end of the eigenvalue spectrum and
throws away the middle ones. What is different about the middle filters that makes them
useless?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q8.** In `mne.decoding.CSP`, what are the shapes of `filters_` and `patterns_`? What
does the output shape become if you set `transform_into='csp_space'` instead of the
default?
*Source: MNE, `mne.decoding.CSP` API. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q9.** What values can `reg` take in `mne.decoding.CSP`, and what problem is
regularisation solving? Given that you have 4 channels and maybe 60 trials, do you
expect it to help much? Why?
*Source: MNE, `mne.decoding.CSP` API. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q10.** What does the `component_order` parameter control, and what is the difference
between `'mutual_info'` and `'alternate'`?
*Source: MNE, `mne.decoding.CSP` API. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q11.** pyRiemann classifies covariance matrices rather than CSP features. What is the
"tangent space", and why can you not simply flatten a covariance matrix and feed it to
LDA directly?
*Source: pyRiemann documentation. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q12.** Look up what accuracy MOABB reports for a CSP+LDA pipeline on a standard
two-class motor imagery dataset. What number is "normal"? What do MOABB's
within-session, cross-session and cross-subject evaluations each mean?
*Source: MOABB documentation. `reviewed: no`*

> **Answer:** _(unanswered)_

## Sources

### Start here

* **MNE-Python, "Decoding motor imagery with CSP"**,
  https://mne.tools/stable/auto_examples/decoding/decoding_csp_eeg.html
  The complete worked pipeline on the PhysioNet dataset, including pattern plotting and
  a sliding-window analysis. This is your Phase 3 target.
* **MNE-Python, `mne.decoding.CSP` API**,
  https://mne.tools/stable/generated/mne.decoding.CSP.html
  Every parameter documented, plus the `filters_` and `patterns_` attributes.

### Go deeper

* **pyRiemann**, https://pyriemann.readthedocs.io/en/latest/
  Riemannian geometry classifiers with a scikit-learn API. The documentation includes a
  good conceptual introduction to why covariance matrices need special treatment.
* **MOABB (Mother of All BCI Benchmarks)**, https://moabb.neurotechx.com/docs/index.html
  Benchmarks standard pipelines (including CSP+LDA) across many public motor imagery
  datasets. Extremely useful for calibrating expectations: it tells you what accuracy a
  given method actually achieves, so you know whether your 68 % is good or broken.

### Papers

* Ramoser, H., Müller-Gerking, J., & Pfurtscheller, G. (2000). "Optimal spatial
  filtering of single trial EEG during imagined hand movement." *IEEE Transactions on
  Rehabilitation Engineering*, 8(4), 441-446. **The paper that introduced CSP for motor
  imagery.** Short and readable.
* Blankertz, B., Tomioka, R., Lemm, S., Kawanabe, M., & Müller, K.-R. (2008).
  "Optimizing spatial filters for robust EEG single-trial analysis." *IEEE Signal
  Processing Magazine*, 25(1), 41-56. **The best tutorial explanation of CSP that
  exists**, written for people who want to understand rather than just use it. Covers
  the eigenvalue formulation, regularisation, and failure modes. Read this one.
* Haufe, S., Meinecke, F., Görgen, K., et al. (2014). "On the interpretation of weight
  vectors of linear models in multivariate neuroimaging." *NeuroImage*, 87, 96-110.
  The filters-vs-patterns distinction, and why misreading it leads to wrong conclusions.
* Ang, K. K., Chin, Z. Y., Zhang, H., & Guan, C. (2008). "Filter Bank Common Spatial
  Pattern (FBCSP) in Brain-Computer Interface." *IEEE IJCNN*. The multi-band extension.
* Barachant, A., Bonnet, S., Congedo, M., & Jutten, C. (2012). "Multiclass
  brain-computer interface classification by Riemannian geometry." *IEEE Transactions on
  Biomedical Engineering*, 59(4), 920-928. The modern alternative to CSP.
* Lotte, F., & Guan, C. (2011). "Regularizing common spatial patterns to improve BCI
  designs: unified theory and new algorithms." *IEEE Transactions on Biomedical
  Engineering*, 58(2), 355-362. On the `reg` parameter and when it helps.

### Video

* **Lecture 7.3 Common Spatial Patterns**,
  https://www.youtube.com/watch?v=zsOULC16USU
  A lecture derivation of CSP: the variance ratio being maximised and the generalised
  eigenvalue problem it reduces to. Watch this if the maths in the Blankertz tutorial
  is not landing.
