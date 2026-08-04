# 11. Real-time processing

**Needed in:** Phase 5
**In one sentence:** Going from offline analysis to a live system is mostly an exercise
in making sure the data reaching your classifier at runtime is statistically identical
to the data it was trained on, while keeping the latency low enough to feel responsive.

## Why this matters here

This is where most BCI projects break, and the breakage is confusing: the offline
accuracy was 72 %, the real-time system behaves like a coin flip, and nothing has
obviously changed. The causes are almost always in this document.

## The sliding window

Offline, you have discrete labelled trials. Live, you have a continuous stream and no
cues. The standard solution is a sliding window:

* Keep the most recent **W** seconds of data (e.g. 2 s).
* Every **S** milliseconds (e.g. 250 ms), take that window, run the pipeline, produce a
  decision.
* Successive windows overlap heavily. With W=2 s and S=250 ms, consecutive windows share
  87.5 % of their samples.

Two parameters, and they trade off directly:

| Parameter | Larger means | Smaller means |
|---|---|---|
| **W** (window length) | More data per decision, better variance estimates, more accurate | Lower latency, more responsive, but noisier |
| **S** (update interval) | Less computation | Smoother-feeling control, but successive decisions are highly correlated |

W = 1 to 2 s is standard for motor imagery, because you need enough samples to estimate
band power reliably. At 256 Hz, a 2 s window is 512 samples, which is a reasonable
variance estimate for an 8 to 30 Hz band.

Note that because windows overlap, consecutive decisions are **not independent**. Four
decisions per second does not mean four independent pieces of evidence per second. This
matters when you compute information transfer rate.

## The train/deploy mismatch: the number one bug

Your classifier learned a decision boundary in feature space. If the features at runtime
are distributed even slightly differently from those in training, the boundary is in the
wrong place. Sources of mismatch, in rough order of how often they bite:

### 1. Different filtering

Offline you use `sosfiltfilt` (zero-phase). Live you must use `sosfilt` (causal). These
produce different signals: different phase, and a transient at the start of each block.
See [05](05-digital-filtering.md).

**Two ways to handle it, and you should pick deliberately:**

* Train on causally filtered data too, so training and deployment match exactly. Costs
  you a little offline accuracy, gains you consistency. Generally the right choice.
* Keep `filtfilt` offline and accept the mismatch, but then measure it: run your
  offline test set through the causal path and see how much accuracy drops.

Either way, document the choice in the report. Most projects never notice the issue
exists.

### 2. Filter state not carried between blocks

If you filter each 2 s window independently, every window begins with a filter transient
because the filter has no history. That transient is a large artifact right at the edge
of your data.

Fix: maintain the filter state (`zi`) across calls, so the filter runs continuously over
the stream:

```python
zi = sosfilt_zi(sos) * x[0]        # initialise once
y, zi = sosfilt(sos, block, zi=zi) # carry state forward each block
```

Alternatively, keep a longer buffer than your analysis window, filter the whole buffer,
and take only the last W seconds. Wasteful but simple, and it avoids the edge transient
by construction. Either works; doing neither does not.

### 3. Different window length

If you trained on 3 s epochs and classify 2 s windows, your variance estimates come from
different amounts of data and are distributed differently. Log-variance features shift
systematically. **Train on windows the same length as the ones you will classify.**

A good approach: after Phase 4, re-epoch your training data into overlapping W-second
windows drawn from within the imagery period, and train on those. Now training data and
runtime data are the same kind of object.

### 4. Different preprocessing order

Any step you do offline but skip live (detrending, rejection, baseline correction) is a
mismatch.

**The structural fix for all of these:** put the entire chain into one saved object.
`joblib.dump(pipe, ...)` on a sklearn `Pipeline` that includes every learned step, and
write the filtering as a single function used by both the offline script and the live
loop. If offline and online use the same code path, they cannot diverge.

## Smoothing decisions

A raw per-window classification is jittery. At 4 decisions per second with 70 %
accuracy, the output flips constantly and the game is unplayable.

Options, roughly in increasing sophistication:

* **Majority vote** over the last N decisions. Simple, works.
* **Exponential moving average of probabilities.** Better, because it uses confidence
  rather than hard labels, and it degrades gracefully:
  `p_smooth = alpha * p_new + (1 - alpha) * p_smooth`. Lower `alpha` means more
  smoothing and more lag.
* **Dwell time / threshold.** Only act when the smoothed probability exceeds a threshold
  (e.g. 0.7) and stays there for a minimum duration. Adds an explicit "uncertain" state.

**Smoothing trades accuracy against latency**, and there is no universally right
setting. Quantify it: plot the false-activation rate during rest against the response
time, for several `alpha` values. That is a good figure for
[REPORT.md](../REPORT.md) D.3 and it demonstrates you understand the trade-off rather
than having tuned it by feel.

## The latency budget

Add up everything between the subject's intention and the game responding:

| Component | Typical |
|---|---|
| Physiological ERD onset | 300 to 500 ms |
| Window length (you need the window to fill) | 1000 to 2000 ms |
| Bluetooth and buffering | 20 to 100 ms |
| Processing (filter, CSP, LDA) | under 10 ms |
| Smoothing lag | 250 to 1000 ms |
| Display | 16 to 30 ms |

Total: comfortably **1.5 to 3 seconds**. That is inherent to the paradigm, not a defect
in your implementation, and it is the single most important constraint on the game
design ([12](12-game-design.md)).

Measure your actual figure rather than estimating it. A simple procedure: have the
subject start imagery on a beep, log the timestamp, and log when the smoothed decision
crosses threshold. Repeat 20 times, report mean and spread.

Note that processing time is negligible here. CSP and LDA on a 4x512 array take
microseconds. Do not optimise the maths; the latency is in physiology and windowing.

## Threading

Acquisition must not block the game loop, and vice versa. A workable structure:

* **Thread 1:** poll BrainFlow at a steady rate, push into a shared buffer.
* **Thread 2 (or the main loop):** every S ms, read the latest W seconds, classify,
  update a shared decision variable.
* **Main loop:** pygame at 60 fps, reading the current decision.

Use `queue.Queue` (thread-safe by design) or a lock around a shared numpy ring buffer.
Use `threading.Event` for clean shutdown rather than killing threads.

Python's GIL is not a problem here, because the workload is I/O-bound waiting on
Bluetooth and the numeric work is tiny and releases the GIL inside numpy anyway.

**Always shut down cleanly.** `board.release_session()` in a `finally` block. An
abandoned session can leave the Muse unable to reconnect until it is power-cycled, and
you will lose ten minutes each time to a problem that looks like broken hardware.

## Debug before you play

Build a minimal visual output first: a bar whose length is the smoothed probability, or
a dot that moves left and right. No game logic, no score, no graphics.

You need to be able to watch the classifier's raw behaviour without game mechanics in
between. Once the game exists, every problem becomes ambiguous: is the classifier bad,
or is the mapping to game actions wrong? The debug view removes that ambiguity, and it
takes twenty minutes to write.

## Common mistakes

* Using `get_board_data()` (which empties the buffer) instead of
  `get_current_board_data(n)` in the loop. See [04](04-data-acquisition.md).
* Filtering each window independently, producing an edge transient every time.
* Training on 3 s epochs and classifying 2 s windows.
* Using `filtfilt` offline and `sosfilt` online without accounting for it.
* Reimplementing preprocessing in the live loop instead of reusing the training code.
* No smoothing, so the output is unusable.
* Too much smoothing, so latency exceeds three seconds and control feels disconnected.
* Going straight to the game without a debug view.
* Not releasing the board session on exit.

## Questions

Write your answers in the boxes. See
[the convention](README.md#answering-the-questions).

**Q1.** Why is `sosfiltfilt` impossible in a real-time loop?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q2.** What happens if you filter each 2 s window independently, and what are two
fixes?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q3.** Why must the training epoch length match the runtime window length?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q4.** Your offline accuracy is 72 % and live performance feels random. List four
things to check, in order.
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q5.** Why is optimising the CSP computation a waste of time here?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q6.** With W=2 s and S=250 ms, how much data do two consecutive windows share, and why
does that matter for measuring information transfer rate?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q7.** Go through the latency budget table and add it up for a system with W=2 s and a
smoothing lag of 500 ms. Which single component is the largest, and is it something you
can engineer away?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q8.** In `scipy.signal.sosfilt`, what does the `zi` argument represent physically, what
does the function return when you pass it, and what shape must `zi` have for an SOS
filter with `n` sections?
*Source: scipy.signal reference. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q9.** What is the difference between `Queue.get()` and `Queue.get_nowait()`? If you set
`maxsize` and your acquisition thread produces faster than your classifier consumes, what
happens on `put()`, and which behaviour do you actually want for EEG?
*Source: Python `queue` documentation. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q10.** What does `threading.Event` give you that a plain boolean flag does not, and
which of its methods would you use to make an acquisition thread exit cleanly?
*Source: Python `threading` documentation. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q11.** Design the measurement that gets you an actual latency number rather than the
estimate in the table above. What do you log, how many repetitions, and what two
statistics do you report? Which of them can you compensate for and which must simply be
stated?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q12.** Smoothing trades accuracy against latency, and there is no correct setting in
the abstract. Describe the experiment that turns that trade-off into a figure, name the
two axes, and say what you would look for on it to choose an `alpha`.
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

## Sources

Nothing but reference documentation, and that is the right shape for this phase. The
failure modes here are not conceptual gaps, they are mismatches between two pieces of
your own code, so the way to learn this topic is to build the debug view and watch it
misbehave. The one paper worth reading for the calibration-to-online distribution shift
is Shenoy et al. (2006), listed in [10](10-experimental-design.md) Tier 2.

### Tier 1

* **BrainFlow, Code Samples**, https://brainflow.readthedocs.io/en/stable/Examples.html
  Includes streaming loops that show the `get_current_board_data` pattern, which is the
  one call that separates a working real-time loop from a broken one.
* **scipy.signal**, https://docs.scipy.org/doc/scipy/reference/signal.html
  Read the docstrings for `sosfilt` and `sosfilt_zi` carefully. The `zi` parameter is the
  entire mechanism for continuous filtering across blocks, and the docs are the clearest
  explanation of it that exists.
* **Python `threading` and `queue` docs**, https://docs.python.org/3/library/queue.html
  `queue.Queue` is thread-safe by construction and removes a whole class of bugs. Use it
  rather than managing locks yourself, and read `threading.Event` for clean shutdown.
