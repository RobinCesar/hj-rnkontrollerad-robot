# 08. Classification

**Needed in:** Phase 3
**In one sentence:** With a handful of CSP features and fewer than a hundred trials,
simple linear classifiers are not a compromise, they are the correct choice, and LDA
with shrinkage is the standard for BCI for well-understood reasons.

## Why this matters here

This is the smallest and least risky part of the pipeline, which is worth knowing so
you do not waste time on it. Almost all of the achievable performance in a motor
imagery BCI comes from the preprocessing and feature extraction. Swapping LDA for
something fancier typically moves accuracy by a couple of percent; fixing your
electrode placement or your epoch window moves it by twenty.

## The setting

After [CSP](07-csp.md) you have:

* `X` of shape `(n_trials, n_components)`, maybe `(60, 4)`
* `y` of shape `(n_trials,)`, values 1 and 2

That is an extremely small dataset by machine learning standards. **The
trials-to-features ratio is what determines which classifier is appropriate**, and with
60 trials and 4 features you are in the regime where simple linear models win and
anything with many parameters will overfit.

## LDA: Linear Discriminant Analysis

The default choice in BCI. LDA assumes each class is a Gaussian cloud, and that both
classes have the **same covariance** (same shape and orientation, different centres).
Under those assumptions the optimal decision boundary is a straight line (a hyperplane),
and LDA finds it analytically.

Why it suits this problem:

* **Very few parameters**, so little to overfit with. This matters enormously at 60
  trials.
* **No hyperparameters to tune** in the basic form, which removes an entire category of
  methodological error (see [09](09-validation.md) on tuning on your test set).
* **Fast**, both to fit and to predict, which matters for the real-time loop.
* **The Gaussian assumption is roughly satisfied**, because the log transform in CSP
  was chosen specifically to make the features approximately normal.

### Shrinkage: the one thing to get right

LDA needs to estimate a covariance matrix from your data. With few trials this estimate
is noisy, and inverting a noisy covariance matrix produces an unstable classifier.

**Shrinkage regularisation** pulls the estimated covariance towards a simple, stable
target (a scaled identity matrix), trading a little bias for a lot of variance
reduction. The Ledoit-Wolf method computes the optimal amount analytically, so there is
nothing to tune.

```python
LDA(solver='lsqr', shrinkage='auto')
```

Note `solver='lsqr'` is required; the default `'svd'` solver does not support shrinkage.
Blankertz et al. (2011) is the paper establishing that shrinkage LDA is the right
default for BCI, and it is worth citing in your method section.

Shrinkage matters more the more features you have. With 4 CSP components it may make
little difference; with 12 it will make a lot. Use it regardless.

## SVM: Support Vector Machines

An SVM finds the hyperplane with the largest **margin**, i.e. the widest gap between
the classes, and its position is determined only by the trials nearest the boundary
(the support vectors).

* **Linear SVM** is a reasonable alternative to LDA. Sometimes slightly better,
  sometimes slightly worse. Worth running as your comparison in Phase 3.
* **RBF-kernel SVM** allows curved boundaries. With 60 trials it will usually overfit,
  and it introduces two hyperparameters (`C` and `gamma`) that you must tune inside a
  nested cross-validation to avoid cheating. Not worth it here, but worth understanding
  why not.
* **SVMs need scaled features.** Put a `StandardScaler` in the pipeline before the SVM.
  LDA does not need this. Forgetting it is a very common cause of "the SVM is much
  worse", which is a scaling bug rather than a real result.

`C` controls the trade-off between margin width and training errors. Low `C` means a
wider margin and more tolerance of misclassified training points, i.e. more
regularisation. Start at `C=1.0`.

## LDA vs. logistic regression: the same boundary, fitted differently

These two produce a decision boundary of exactly the same functional form, a hyperplane,
and beginners often assume they are the same method under two names. They are not, and
the difference is precisely the one that matters at 60 trials.

* **LDA is generative.** It models each class as a Gaussian, estimates the class means
  and the shared covariance, and *derives* the boundary from those. It is making a strong
  assumption about how the data was produced.
* **Logistic regression is discriminative.** It models the probability of the class given
  the features directly, and fits the boundary by maximum likelihood. It assumes nothing
  about the shape of the class distributions.

The trade-off is the classic one between assumptions and data. When the Gaussian
assumption is roughly true, LDA extracts more information per trial, because it is using
structure that logistic regression ignores, and it therefore reaches a good boundary from
fewer samples. When the assumption is false, LDA is biased and logistic regression
eventually wins, but "eventually" means once you have enough data for the weaker method
to catch up.

You have 60 trials, and the `log` step in CSP was chosen specifically to make the
features approximately Gaussian. That combination is the regime where LDA is favoured,
and it is the actual reason LDA is the BCI default rather than mere convention. Logistic
regression remains worth having in the pipeline for one practical reason: its
probabilities are the best calibrated of the three options here, which matters for the
game's confidence display and its abstention threshold.

## Other options and when to consider them

| Method | Verdict for this project |
|---|---|
| **Logistic regression** | Perfectly reasonable, gives well-calibrated probabilities, useful for the confidence display in the game |
| **Random forest / gradient boosting** | Will overfit at this trial count, and gives no interpretability advantage here |
| **k-NN** | Poor in low-trial, moderate-dimensional settings |
| **Neural networks (EEGNet etc.)** | Need thousands of trials. Interesting for the report's future-work section, not for the implementation |
| **Riemannian tangent space + logistic regression** | Genuinely better than CSP+LDA and not much harder. See [07](07-csp.md) |

## Probabilities, not just labels

For the game you want `predict_proba()` rather than `predict()`. Two reasons:

1. **Smoothing.** Averaging probabilities over successive windows is better behaved
   than majority-voting hard labels. See [11](11-realtime.md).
2. **Abstention.** If the probability is near 0.5 the classifier is guessing. A game
   that does nothing when uncertain feels far better than one that moves randomly.
   This is a real design lever, and it is worth measuring: plot accuracy against the
   fraction of trials you refuse to classify.

A caveat: `predict_proba` outputs are not necessarily well-calibrated probabilities.
LDA's are reasonable. SVM's come from Platt scaling (`probability=True`), which
requires internal cross-validation, makes fitting slower, and can be inconsistent with
`predict()`. Logistic regression gives the best-calibrated outputs of the three.

## Interpreting the model

A linear classifier's weights tell you which CSP components mattered. As with CSP
filters, be careful: classifier weights are not directly interpretable as importance
(the Haufe et al. 2014 point again). But checking that the model relies on the extreme
CSP components rather than the middling ones is a useful sanity check.

The **confusion matrix** is more informative than accuracy alone. It tells you whether
errors are symmetric or whether the model is biased towards one class, which happens
easily with imbalanced data.

## Common mistakes

* Reaching for a complex model to fix a weak signal. It cannot; it will only overfit.
* Forgetting `StandardScaler` before an SVM and concluding SVMs are bad.
* Using the default `svd` solver with `shrinkage='auto'`, which silently fails.
* Tuning `C` by trying values and picking the best cross-validated score, then
  reporting that score. That is optimistically biased. See [09](09-validation.md).
* Reporting only accuracy without the confusion matrix or the class balance.
* Using hard labels in real time when probabilities would let you smooth and abstain.

## Questions

Write your answers in the boxes. See
[the convention](README.md#answering-the-questions).

**Q1.** Why is LDA a better fit than a random forest for 60 trials and 4 features?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q2.** What does shrinkage do, and why does it matter more with more features?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q3.** Which classifiers need feature scaling and which do not?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q4.** Why is `predict_proba` more useful than `predict` for the game?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q5.** You get 70 % accuracy. What does the confusion matrix tell you that accuracy
does not?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q6.** Why is tuning `C` on your cross-validation score and then reporting that score
dishonest, and what is the fix?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q7.** What are the two assumptions LDA makes about the class distributions, and which
step of the CSP feature extraction was chosen partly to make one of them hold?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q8.** What assumption does QDA drop compared with LDA, and what does that cost you in
number of parameters to estimate? Why is that a bad trade at 60 trials?
*Source: scikit-learn, "Linear and Quadratic Discriminant Analysis". `reviewed: no`*

> **Answer:** _(unanswered)_

**Q9.** Which `LinearDiscriminantAnalysis` solvers support `shrinkage`, and which one
also supports `transform()` for dimensionality reduction? What happens if you pass
`shrinkage='auto'` with the default solver?
*Source: scikit-learn, "Linear and Quadratic Discriminant Analysis". `reviewed: no`*

> **Answer:** _(unanswered)_

**Q10.** The "Tips on Practical Use" section of the SVM page gives a specific
recommendation about `C` when your data is noisy. What is it, and which direction does
it push `C`?
*Source: scikit-learn, "Support Vector Machines". `reviewed: no`*

> **Answer:** _(unanswered)_

**Q11.** What is the difference between `Pipeline` and `make_pipeline`? If your pipeline
step is named `csp`, how do you refer to its `n_components` parameter inside a
`GridSearchCV` parameter grid?
*Source: scikit-learn, "Pipelines and composite estimators". `reviewed: no`*

> **Answer:** _(unanswered)_

**Q12.** Lotte et al. (2018) give recommendations by data regime rather than a single
best method. What do they recommend for a small-sample, few-feature, two-class motor
imagery problem, and what do they say about Riemannian methods? Does their advice match
what this document told you to do?
*Source: Lotte et al. (2018). `reviewed: no`*

> **Answer:** _(unanswered)_

## Sources

This is the least risky part of the pipeline, so it gets the smallest reading list in the
folder. **Tier 1** is two scikit-learn pages. Resist the temptation to go deeper here;
the accuracy is not hiding in the classifier.

### Tier 1

* **scikit-learn, "Linear and Quadratic Discriminant Analysis"**,
  https://scikit-learn.org/stable/modules/lda_qda.html
  Includes a section specifically on shrinkage and why it helps, with a figure showing
  the effect growing as the feature count grows. That figure is the argument for the one
  parameter choice that matters in this topic.
* **scikit-learn, "Pipelines and composite estimators"**,
  https://scikit-learn.org/stable/modules/compose.html
  How to chain CSP, scaling and the classifier into one object. This is not a convenience
  topic: it is the mechanism that keeps your validation honest, and
  [09](09-validation.md) depends on it entirely.

### Tier 2

* **scikit-learn, "Support Vector Machines"**,
  https://scikit-learn.org/stable/modules/svm.html
  Read the "Tips on Practical Use" section only, which covers scaling and choosing `C`.
  You need this the day you run the SVM comparison in Phase 3, and not before.
* **Lotte, F., Bougrain, L., Cichocki, A., et al. (2018). "A review of classification
  algorithms for EEG-based brain-computer interfaces: a 10 year update."** *Journal of
  Neural Engineering*, 15(3), 031005. The survey of the field, with recommendations
  broken down by data regime. Read the recommendations and the summary tables rather than
  the whole thing, and use it as the citation justifying your classifier choice in the
  method section.
