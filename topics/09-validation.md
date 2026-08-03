# 09. Validation and statistics

**Needed in:** Phase 3, Phase 4
**In one sentence:** An accuracy number means nothing on its own; it is only
interpretable next to a chance level, a confidence interval, and evidence that your
evaluation procedure did not let information leak from the test set into training.

## Why this matters here

This is the difference between a project that demonstrates competence and one that
demonstrates enthusiasm. With 60 trials and 4 channels, the two most likely outcomes
are a genuine weak result and a spurious strong one, and only a rigorous validation
setup lets you tell them apart. Everything in this document is what a reviewer, or an
interviewer at a neurotech company, would probe first.

## Cross-validation

You cannot evaluate on the data you trained on; the model has seen the answers.
With only 60 trials you also cannot afford a large held-out test set. Cross-validation
resolves this by splitting the data k ways, training on k-1 folds and testing on the
remaining one, rotating through all folds, and averaging.

* **StratifiedKFold** preserves the class ratio in every fold. Use this rather than
  plain `KFold`. With small, potentially imbalanced data, an unstratified split can
  produce a fold with a badly skewed class ratio.
* **k=5 or k=10.** More folds means more training data per fold (less pessimistic bias)
  but higher variance between folds and more compute. 5 is fine here.
* **Report the standard deviation across folds**, not only the mean. A result of
  68 % ± 3 % is a very different claim from 68 % ± 15 %, and with 60 trials the second
  is common. Each fold's test set has only ~12 trials, so fold-to-fold variability is
  large by construction.

### Shuffling and temporal structure

`shuffle=True` randomises which trials go in which fold. This is usually right, but be
aware of what it assumes: that trials are independent. In EEG they are not entirely
independent, because electrode drift, fatigue, and impedance changes make temporally
adjacent trials more similar to each other. Random shuffling lets the model train on
trials recorded seconds before its test trials, which is mildly optimistic.

A stricter alternative is to split by block or by time, which is closer to how the
system will actually be used (calibrate now, play afterwards). Running both and
reporting the difference is a good, honest result.

## Data leakage: the failure mode that matters most

**Leakage is when information from the test set influences the training process.** It
produces inflated, meaningless accuracy, and it is very easy to introduce accidentally.

The canonical version in this project:

```python
# WRONG: CSP sees every trial's labels before the split
X_feat = csp.fit_transform(X, y)
scores = cross_val_score(clf, X_feat, y, cv=cv)
```

CSP uses `y`. Fitting it on all the data means the spatial filters were optimised using
the labels of the trials you are about to test on. The reported accuracy will be
substantially too high, and the effect is larger the fewer trials you have.

```python
# RIGHT: everything that learns is inside the pipeline
pipe = Pipeline([('csp', CSP(4, log=True)), ('clf', LDA(solver='lsqr', shrinkage='auto'))])
scores = cross_val_score(pipe, X, y, cv=cv)
```

The rule: **anything that learns anything from the data belongs inside the `Pipeline`.**
That includes CSP, scalers, feature selection, and dimensionality reduction. This is
what `Pipeline` is for; it is not a convenience wrapper.

Other leakage sources to watch for in this project:

* **Filtering across trial boundaries.** Filtering the continuous recording is correct
  and standard, but note that a long filter can smear information across the boundary
  between what will become a training trial and a test trial. In practice this is small
  for the filters used here, but it is worth knowing the concern exists.
* **Choosing the epoch window, the frequency band, or `n_components` by looking at
  cross-validated accuracy, then reporting that same accuracy.** This is the subtlest
  and most common form. Every choice you make by consulting the test score costs you
  some of the score's validity. The clean solution is nested cross-validation; the
  practical solution for this project is to make those choices on the *public dataset*
  in Phase 3, fix them, and then apply them unchanged to your own data. Say that this
  is what you did.
* **Normalising across the whole dataset before splitting.**

## Baselines: what to compare against

An accuracy is meaningless without a reference point.

### Theoretical chance level

For two balanced classes, 50 %. This is the *expected* value under the null hypothesis,
not a threshold you need to beat.

### The confidence interval around chance

Here is the point almost everyone misses. With a small test set, an accuracy well above
50 % can easily occur by pure chance.

Under the null hypothesis, the number of correct classifications follows a **binomial
distribution** with n = number of test trials and p = 0.5. For n = 60, the 95 % upper
bound is around **60 %**. For n = 20, it is around **65 %**.

So "I got 58 % with 60 trials" is **not** evidence of anything. It is inside the range
you would expect from a coin flip. Müller-Putz et al. (2008) and Combrisson & Jerbi
(2015) exist entirely because so many published BCI papers made this error.

Compute this bound for your own trial count and put it in the results table. It takes
one call to `scipy.stats.binom.ppf`.

### Permutation testing

The empirical version, and more trustworthy because it makes no distributional
assumptions and it accounts for your entire pipeline:

1. Randomly shuffle the labels `y`.
2. Run the **complete** cross-validation procedure, refitting everything.
3. Record the accuracy.
4. Repeat 1000 times to build the null distribution.
5. Your p-value is the fraction of permuted runs that scored at least as high as your
   real run.

`sklearn.model_selection.permutation_test_score` does this.

Two reasons this is the best single check you can run:

* It gives you a real p-value for "is this better than chance".
* **It detects leakage.** If your pipeline is leaky, the permuted runs will also score
  above chance, because the leak provides information regardless of whether the labels
  are meaningful. A null distribution centred well above 50 % is a bug report.

Run it on the public dataset in Phase 3 first, where you know the answer, so you learn
what a correct null distribution looks like.

### Comparative baselines

A single accuracy is uninterpretable. What you report is a **ladder** of conditions, all
run through the identical pipeline so that only the data differs, ordered from "no signal
by construction" to "signal that is definitely present". The canonical version lives in
[REPORT.md](../REPORT.md#d1-the-ladder) section D.1 and is filled in as you go; do not
maintain a second copy here.

The principle behind it is worth stating on its own, because it generalises far past this
project: **you learn from the gaps between conditions, not from any one number.** The
public-data row tells you the pipeline works. The channel-subset row tells you what the
hardware costs. The deliberate eye/jaw row tells you what a non-neural signal achieves on
the same equipment, which is the ceiling your real result has to be distinguished from.
An imagery accuracy sitting close to that ceiling is a warning, not a success.

Designing the comparison set is the actual skill here. Running cross-validation is the
easy part.

## Metrics beyond accuracy

* **Balanced accuracy** if your classes are unequal after rejection.
* **Confusion matrix**, always. It reveals class bias that accuracy hides.
* **Cohen's kappa** is the conventional BCI metric, correcting for chance agreement.
  Worth reporting because it makes your numbers comparable to published work.
* **Information transfer rate (ITR)**, bits per minute, combines accuracy with speed.
  This is the standard way BCIs are compared, and it is the right metric for the
  real-time system where a slower, more accurate classifier trades against a faster,
  noisier one.

## Common mistakes

* Fitting CSP outside the cross-validation loop.
* Reporting a mean without a standard deviation or a chance interval.
* Comparing against 50 % instead of against the binomial upper bound.
* Tuning parameters on cross-validated accuracy and reporting that same accuracy.
* Running the permutation test on the classifier only, rather than on the whole
  pipeline, which defeats its purpose as a leakage detector.
* Presenting a within-session cross-validation number as if it predicted real-world use.

## Questions

Write your answers in the boxes. See
[the convention](README.md#answering-the-questions).

**Q1.** Why must CSP be inside the `Pipeline` rather than applied beforehand?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q2.** You have 40 test trials and get 62 % accuracy. Is that significant? How would
you check?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q3.** Your permutation null distribution is centred at 61 % rather than 50 %. What
does that tell you?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q4.** Why is `StratifiedKFold` preferable to `KFold` here?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q5.** You try five epoch windows and report the best cross-validated score. What is
wrong, and what are two ways to fix it?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q6.** Why might within-session cross-validation overestimate how well the game will
work?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q7.** `shuffle=True` assumes something about your trials that is not quite true in
EEG. What is the assumption, why is it violated, and which direction does the violation
bias your accuracy?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q8.** What is the difference between `cross_val_score`, `cross_validate` and
`cross_val_predict`? The documentation warns against using one of them to compute a
generalisation metric. Which one, and why?
*Source: scikit-learn, "Cross-validation: evaluating estimator performance".
`reviewed: no`*

> **Answer:** _(unanswered)_

**Q9.** What does `GroupKFold` guarantee that `StratifiedKFold` does not? In your own
recordings, what would be the natural thing to use as a group, and what leak would that
prevent?
*Source: scikit-learn, "Cross-validation: evaluating estimator performance".
`reviewed: no`*

> **Answer:** _(unanswered)_

**Q10.** What three things does `permutation_test_score` return, and how exactly is the
p-value computed? There is a `+1` in the formula. What is it for?
*Source: scikit-learn, "Test with permutations the significance of a classification
score". `reviewed: no`*

> **Answer:** _(unanswered)_

**Q11.** Section 7.10.2 of ESL, "The Wrong and Right Way to Do Cross-validation", walks
through a concrete example. Describe what was done wrong, what accuracy it produced, and
what the true accuracy should have been given how the data was generated.
*Source: Hastie, Tibshirani & Friedman, "The Elements of Statistical Learning",
section 7.10.2. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q12.** MOABB defines three evaluation protocols. Name them, and say which one most
closely matches how someone would actually sit down and play your game.
*Source: MOABB documentation. `reviewed: no`*

> **Answer:** _(unanswered)_

## Sources

### Start here

* **scikit-learn, "Cross-validation: evaluating estimator performance"**,
  https://scikit-learn.org/stable/modules/cross_validation.html
  Covers every splitter, and has an explicit section on why transformations must be
  inside the pipeline. Read the whole page.
* **scikit-learn, "Test with permutations the significance of a classification score"**,
  https://scikit-learn.org/stable/auto_examples/model_selection/plot_permutation_tests_for_classification.html
  A worked permutation test with the null distribution plotted. This is the figure you
  want in your report.
* **scikit-learn, "Pipelines and composite estimators"**,
  https://scikit-learn.org/stable/modules/compose.html

### Go deeper

* **MOABB**, https://moabb.neurotechx.com/docs/index.html
  Exists because BCI results were not comparable across papers. Their evaluation
  protocols (within-session, cross-session, cross-subject) are the ones to imitate, and
  their published numbers tell you what accuracy is actually normal for CSP+LDA on
  motor imagery.
* **Hastie, Tibshirani & Friedman, "The Elements of Statistical Learning"**, free at
  https://hastie.su.domains/ElemStatLearn/
  Chapter 7 on model assessment, especially section 7.10.2, "The Wrong and Right Way to
  Do Cross-validation", which is the clearest statement of the leakage problem in print.

### Papers

* Combrisson, E., & Jerbi, K. (2015). "Exceeding chance level by chance: The caveat of
  theoretical chance levels in brain signal classification and statistical assessment of
  decoding accuracy." *Journal of Neuroscience Methods*, 250, 126-136. **Read this
  before you report any number.** It is exactly about the mistake you are most likely to
  make, with tables of the binomial thresholds by sample size.
* Müller-Putz, G. R., Scherer, R., Brunner, C., Leeb, R., & Pfurtscheller, G. (2008).
  "Better than random? A closer look on BCI results." *International Journal of
  Bioelectromagnetism*, 10(1), 52-55. Short, blunt, and specific to BCI.
* Varoquaux, G. (2018). "Cross-validation failure: Small sample sizes lead to large
  error bars." *NeuroImage*, 180, 68-77. Why cross-validation estimates are so noisy
  with few samples. Directly relevant to a 60-trial study, and the reason you must
  report variability.
* Kriegeskorte, N., Simmons, W. K., Bellgowan, P. S., & Baker, C. I. (2009). "Circular
  analysis in systems neuroscience: the dangers of double dipping." *Nature
  Neuroscience*, 12(5), 535-540. The general form of the leakage problem in
  neuroscience.

### Video

* **StatQuest: Cross-Validation**,
  https://www.youtube.com/watch?v=fSytzGwwBVw
  Why a single train and test split is not enough and what k-fold does about it.
  Short and worth watching before you interpret any accuracy number in this project.
