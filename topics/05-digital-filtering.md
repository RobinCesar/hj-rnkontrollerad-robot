# 05. Digital filtering

**Needed in:** Phase 2, Phase 5
**In one sentence:** Filtering removes frequencies you do not want, but it also
distorts the ones you keep, and the trade-off between how sharp the filter is and how
much it smears your signal in time is something you have to choose deliberately.

## Why this matters here

Filtering is where most early bugs live, and worse, filtering bugs produce *plausible*
output. A badly filtered signal still looks like EEG. You only find out when your
classifier fails, or when it succeeds for the wrong reason. Understanding filters is
also what separates people who use signal processing from people who copy it.

## What a filter does

A filter attenuates some frequencies and passes others. Four basic shapes:

| Type | Passes | Used here for |
|---|---|---|
| Low-pass | below a cutoff | anti-aliasing before downsampling |
| High-pass | above a cutoff | removing drift and sweat artifacts |
| **Band-pass** | between two cutoffs | **isolating mu and beta, 8 to 30 Hz** |
| **Band-stop / notch** | everything except a narrow band | **removing 50 Hz mains** |

## The three properties you have to reason about

### 1. The transition is not a cliff

A "8 to 30 Hz bandpass" does not pass 30 Hz fully and 31 Hz not at all. There is a
**transition band** where attenuation ramps up gradually. How steep that ramp is
depends on the filter **order**.

* Higher order = steeper transition = closer to the ideal.
* Higher order also = more ringing, more time-domain distortion, and potential
  numerical instability.
* Order 4 is a sensible default for EEG. Start there.

Always **verify** your filter by plotting the power spectrum before and after. If you
cannot see the band you asked for, the filter is not doing what you think it is. Do
this in Phase 2; it takes five minutes and it is the only way to be sure.

### 2. Filters shift signals in time (phase response)

This is the property people forget. A causal filter must produce output using only past
samples, and that inevitably delays the signal. Worse, the delay is usually **different
at different frequencies** (non-linear phase), which distorts the waveform shape.

For EEG this matters because your analysis is time-locked to an event. If your 10 Hz
component is delayed by 80 ms and your 25 Hz component by 30 ms, the temporal
relationship between your cue and the ERD onset is corrupted.

**The offline solution: zero-phase filtering.** Run the filter forwards, then run it
backwards over the result. The two delays cancel exactly, giving zero net phase shift.
This is `filtfilt` (or `sosfiltfilt`).

The catch, and it is a real one: running the filter backwards means output at time *t*
depends on input at times *after* *t*. The filter is **non-causal**. It can smear an
artifact *backwards* in time, so a blink at 3.0 s can produce apparent activity at
2.8 s. When you are looking for a pre-movement ERD, that can manufacture an effect that
was not there. This is the central warning in de Cheveigné & Nelken (2019), and it is
worth taking seriously.

Also note `filtfilt` applies the filter twice, so the effective order is doubled and the
attenuation is squared. A "4th order" `filtfilt` is really 8th order.

**In real time you cannot use `filtfilt` at all**, because the future has not happened
yet. You use a causal filter (`sosfilt`) and accept the delay as part of your latency
budget. See [11](11-realtime.md).

**This offline/online asymmetry is a genuine methodological issue for your project**:
your classifier is trained on zero-phase-filtered data but deployed on causally
filtered data. The distributions differ. Note it in the report; discuss whether to
train on causally filtered data instead so that training and deployment match.

### 3. Edge effects

At the start and end of any filtered segment there is a transient, because the filter
has no history to work with. It lasts roughly as long as the filter's impulse response.

Two consequences:

* **Filter continuous data before epoching, not after.** If you epoch first, every
  single epoch gets a transient at both ends, right where your data of interest is.
  This is one of the most common EEG processing errors.
* When you do filter continuously, discard the first second or so of the recording.

## Butterworth, IIR, FIR, and SOS

* **IIR** (Infinite Impulse Response) filters are recursive: output depends on previous
  outputs. Efficient, low order for a given sharpness, but non-linear phase. Butterworth
  is an IIR design characterised by a maximally flat passband (no ripple), which is why
  it is the default choice in EEG. Both BrainFlow and `scipy.signal.butter` give you this.
* **FIR** (Finite Impulse Response) filters use only past inputs. They can be designed
  with exactly linear phase (a constant delay at all frequencies, correctable by simply
  shifting), and they are unconditionally stable. They need a much higher order for the
  same sharpness. MNE defaults to FIR for good reasons.
* **SOS** (Second-Order Sections) is a *representation* of an IIR filter as a cascade of
  small second-order stages rather than one big polynomial. Mathematically identical,
  numerically far more stable. **Always use `output='sos'`** with scipy; the classic
  `b, a` form accumulates floating-point error badly above about order 4 and can produce
  garbage without warning.

## Downsampling and aliasing

If you reduce the sampling rate, any frequency above the *new* Nyquist limit does not
disappear. It **folds back** and reappears disguised as a lower frequency, permanently
corrupting your data. Before downsampling you must low-pass filter below the new
Nyquist frequency.

`raw.resample()` in MNE and `scipy.signal.decimate` do this for you. `x[::2]` does
**not**. Never downsample by slicing.

Do you need to downsample here? Not really. 256 Hz for an 8 to 30 Hz analysis is fine,
and the data volume is trivial. It becomes relevant only if you want to reduce
computation in the real-time loop.

## Order of operations

The order that works for this project:

1. Load raw, continuous data
2. Remove the first few seconds (electrode settling)
3. Notch 50 Hz (optional if bandpassing to 30 Hz, but do it for raw-data inspection)
4. Bandpass 8 to 30 Hz on the **continuous** data
5. Epoch around events ([06](06-epoching.md))
6. Reject bad epochs by amplitude ([03](03-artifacts.md))
7. Crop to the analysis window
8. CSP ([07](07-csp.md))

Steps 3 and 4 must precede step 5. Step 4 must precede step 8, because CSP works on
band-limited variance.

## Common mistakes

* Filtering after epoching, producing edge transients in every trial.
* Using `b, a` instead of `sos` at high order and getting silently wrong output.
* Using `filtfilt` in the real-time loop (it cannot work) or, more subtly, training
  offline with `filtfilt` and deploying with `sosfilt` without acknowledging the
  mismatch.
* Downsampling by slicing.
* Never plotting the spectrum to verify the filter did what was asked.
* Cranking up the order to get a sharper cutoff and introducing ringing or instability.
* Forgetting that `filtfilt` doubles the effective order.

## Questions

Write your answers in the boxes. See
[the convention](README.md#answering-the-questions).

**Q1.** Why can you not use `sosfiltfilt` in a real-time loop?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q2.** What specifically goes wrong if you epoch before filtering?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q3.** You bandpass 8 to 30 Hz with `sosfiltfilt` and order 6. What is the effective
order?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q4.** Why is `output='sos'` preferred over `b, a`?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q5.** You resample from 256 Hz to 64 Hz. What must happen first, and what happens if
it does not?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q6.** Why might a non-causal filter make you believe in a pre-movement effect that is
not real?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q7.** Name the trade-off that filter order controls, in both directions. Why is order
4 a sensible starting point rather than order 20?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q8.** MNE's filtering tutorial explains the relationship between filter length,
transition bandwidth and ringing. State it: what happens to the transition band as the
filter gets longer, and what does that cost you in the time domain?
*Source: MNE, "Background information on filtering". `reviewed: no`*

> **Answer:** _(unanswered)_

**Q9.** The same tutorial is emphatic about high-pass cutoffs. What does it say goes
wrong when you high-pass above roughly 0.1 to 1 Hz, and why does that matter less for
this project than for an ERP study?
*Source: MNE, "Background information on filtering". `reviewed: no`*

> **Answer:** _(unanswered)_

**Q10.** What is a filter's impulse response, what is its frequency response, and what
is the operation that connects them to the act of filtering a signal?
*Source: Smith, "The Scientist and Engineer's Guide to DSP", ch. 14 to 21.
`reviewed: no`*

> **Answer:** _(unanswered)_

**Q11.** Smith frames the FIR vs. IIR choice as two different design goals rather than
one being better. What are the two goals, and which one does your offline 8 to 30 Hz
bandpass care about most?
*Source: Smith, "The Scientist and Engineer's Guide to DSP". `reviewed: no`*

> **Answer:** _(unanswered)_

**Q12.** What does `scipy.signal.sosfilt_zi` return, and what do you multiply it by
before using it on a real signal? Separately: what does `sosfreqz` let you check about a
filter *before* you apply it to any data?
*Source: scipy.signal reference. `reviewed: no`*

> **Answer:** _(unanswered)_

## Sources

### Start here

* **MNE-Python, "Background information on filtering"**,
  https://mne.tools/stable/auto_tutorials/preprocessing/25_background_filtering.html
  The best single document on this topic anywhere. Written specifically for EEG, covers
  causal vs. zero-phase, filter order, ringing, and edge effects, with figures showing
  exactly what goes wrong in each case. **Read this one properly, twice if needed.**
* **Steven W. Smith, "The Scientist and Engineer's Guide to DSP"**, free at
  https://www.dspguide.com/
  Chapters 14 to 21 cover filters from the ground up with almost no mathematical
  prerequisites. If MNE's tutorial assumes too much, start here instead.

### Go deeper

* **scipy.signal reference**, https://docs.scipy.org/doc/scipy/reference/signal.html
  and the tutorial at https://docs.scipy.org/doc/scipy/tutorial/signal.html
  Read the docstrings for `butter`, `sosfilt`, `sosfiltfilt`, `sosfilt_zi` and
  `freqz` carefully. `freqz` lets you plot the frequency response of a filter you
  designed, which is how you check it before applying it to anything.
* **Mike X Cohen, "Analyzing Neural Time Series Data"** (MIT Press, 2014), the
  filtering chapters. Also his free video series at
  https://www.youtube.com/@mikexcohen1, which includes a full set of lectures on
  filtering neural data with worked examples.

### Papers

* de Cheveigné, A., & Nelken, I. (2019). "Filters: When, Why, and How (Not) to Use
  Them." *Neuron*, 102(2), 280-293. A short, readable, and slightly alarming paper on
  the artifacts filtering introduces. The source of the warning about non-causal
  filters creating apparent pre-stimulus effects.
* Widmann, A., Schröger, E., & Maess, B. (2015). "Digital filter design for
  electrophysiological data: a practical approach." *Journal of Neuroscience Methods*,
  250, 34-46. Concrete recommendations for filter type, order and cutoffs in EEG, with
  the reasoning behind each. This is the paper to cite in your method section.
* Rousselet, G. A. (2012). "Does filtering preclude us from studying ERP time-courses?"
  *Frontiers in Psychology*, 3, 131. Short, on how high-pass filtering distorts the
  timing of effects.

### Video

* **Band-pass filtering and the filter-Hilbert method**,
  https://www.youtube.com/watch?v=ljw3gW-nL0E
  Band-pass filter design and then the Hilbert transform for extracting band power.
  The filter-Hilbert method is directly relevant: it is one of the two ways to get
  the mu band envelope that your classifier will feed on.
