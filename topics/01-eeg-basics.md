# 01. EEG basics

**Needed in:** Phase 0
**In one sentence:** EEG measures tiny voltage fluctuations on the scalp produced by
the synchronised activity of millions of neurons, and almost every property of the
signal follows from how badly the skull blurs it.

## Why this matters here

Every design choice later in the project (why 4 channels is limiting, why you filter
8 to 30 Hz, why you need dozens of repeated trials) is a consequence of the physics of
how EEG is generated and measured. If you understand this section, most of the rest
stops feeling arbitrary.

## What the signal actually is

A single neuron firing produces nothing measurable at the scalp. What EEG picks up is
the summed **postsynaptic potentials** of large populations of cortical pyramidal
neurons that happen to be aligned in parallel and active in synchrony. Two conditions
must hold for a signal to reach your electrode:

1. **Spatial alignment.** Pyramidal neurons in cortex are oriented perpendicular to the
   surface, so their small dipoles add up rather than cancelling.
2. **Temporal synchrony.** Thousands of neurons must be doing roughly the same thing at
   the same time. Desynchronised activity cancels out, which is exactly the mechanism
   behind ERD (see [02](02-motor-imagery-erd.md)).

Consequence: EEG is blind to a great deal of brain activity. It sees synchronous,
superficial, radially-oriented cortical population activity, and very little else.

## Scale and units

* Scalp EEG is roughly **10 to 100 µV** peak to peak. That is about a hundred thousand
  times smaller than a AA battery.
* Eye blinks are **100 to 200 µV**, muscle artifacts can be larger. Your artifacts are
  routinely bigger than your signal. This is the defining practical problem of EEG.
* MNE stores data in **volts**; BrainFlow gives you **microvolts**. Getting this wrong
  by a factor of a million is a rite of passage.

## Volume conduction and why 4 channels hurts

The signal passes through cerebrospinal fluid, skull and scalp before reaching the
electrode. The skull in particular is a poor conductor and acts as a **spatial
low-pass filter**: it smears each cortical source over several centimetres of scalp.

The mechanism is worth having concretely, because it is the physical fact that limits
the whole project. Skull bone conducts much worse than the tissue on either side of it.
Older measurements put the ratio at roughly 1:80 against brain and scalp, modern ones
nearer 1:15, and either way the skull is the bottleneck in the path. Current leaving a
cortical source therefore takes the easier route: it spreads sideways through the
conductive cerebrospinal fluid and scalp rather than crossing the skull straight up to
the nearest electrode. By the time it reaches the surface, one small, focal source has
been blurred into a broad patch several centimetres across.

Two consequences that matter for this project:

* An electrode does not measure "the brain underneath it". It measures a weighted mix
  of many sources, near and far. This is called **volume conduction**, and it runs both
  ways: one source appears on many electrodes, and one electrode contains many sources.
* Because of the smearing, spatial resolution is poor (centimetres, not millimetres),
  and you need many electrodes to separate sources. With 4 electrodes you cannot do
  much spatial separation at all. This is the core reason
  [07 CSP](07-csp.md) will struggle on Muse data.

Note what volume conduction is **not**: it is not a delay. At EEG frequencies the head
behaves as a resistor rather than anything with meaningful capacitance, so the spreading
is effectively instantaneous. That is why EEG keeps its millisecond timing even while
losing spatial detail, and it is the trade-off the next section is about.

The upside of volume conduction: a source over motor cortex is not *completely*
invisible at TP9/TP10, just heavily attenuated and mixed with everything else. That is
the thin thread this project hangs on.

## What EEG buys, and what it costs

EEG and fMRI sit at opposite corners of the same trade-off. Knowing which corner you are
in explains most of this project's design.

| | EEG | fMRI |
|---|---|---|
| Time resolution | about 1 ms, limited only by the sampling rate | about 1 s, limited by the blood-flow response |
| Spatial resolution | centimetres, and ambiguous | millimetres, and unambiguous |
| What is measured | the electrical activity itself | blood oxygenation, an indirect proxy |
| Practicality | a headband on your desk | a shielded room and a physicist |

You are buying time resolution and paying for it in space. For an 8 to 30 Hz rhythm that
rises and falls over hundreds of milliseconds, that is the right trade. For localising
which square centimetre of cortex did it, it is the wrong one, which is exactly the
project's central limitation.

The part that shapes your experiment is the **signal-to-noise ratio**. A single trial
contains the effect you want buried under ongoing activity that is usually larger than
it. No analysis step fixes that; only repetition does. Averaging n trials improves SNR by
roughly the square root of n, so going from 10 trials to 40 buys you a factor of two.
That arithmetic is why protocols call for 40 to 60 trials per class rather than 5, and
why [10 experimental design](10-experimental-design.md) treats trial count as a hard
requirement rather than a preference. A classifier pools trials more cleverly than a
plain average does, but it does not escape the same arithmetic.

## Reference, ground, and why it matters

Voltage is always a difference between two points. There is no such thing as "the
voltage at TP9"; there is only "TP9 minus the reference". This means:

* Changing the reference changes the shape of your signal everywhere. A "signal" you
  see at one electrode may actually be activity at the reference.
* The Muse references to **FPz** (on the forehead). That location is close to the eyes,
  so eye artifacts contaminate *all* Muse channels, not just the frontal ones.
* Common re-referencing schemes include average reference, linked mastoids, and
  Laplacian. With 4 channels your options are limited, but be aware the choice exists.

**There is no neutral reference, including the average reference.** It is tempting to
treat the average of all electrodes as "no reference at all", a true zero. It is not.
Every scheme subtracts *something*, and that something is itself a signal recorded on a
head. The average reference approximates a neutral point only if your electrodes sample
the whole head evenly, which requires many of them spread all over it; averaging four
electrodes clustered on the front and sides of the head gives you the mean of the front
and sides, not the mean of the head.

Two practical consequences:

* A waveform is only meaningful together with the reference it was computed against.
  "There is a 10 µV peak at TP9" is an incomplete statement.
* A number from a published study that used linked mastoids is **not** directly
  comparable with yours from FPz, because the two are measuring different differences.
  When you compare against the literature, compare effects, directions and relative
  changes, not absolute amplitudes. This matters in [REPORT.md](../REPORT.md) whenever
  you put your numbers next to somebody else's.

## Frequency bands

EEG is conventionally split into bands. The boundaries are conventions, not physical
constants, and different papers use slightly different ones.

| Band | Range | Loosely associated with |
|---|---|---|
| Delta | below 4 Hz | Deep sleep. Also where most drift and sweat artifacts live |
| Theta | 4 to 8 Hz | Drowsiness, memory encoding |
| **Alpha** | **8 to 13 Hz** | Relaxed wakefulness, strongest over occipital cortex with eyes closed |
| **Mu** | **8 to 13 Hz** | Same frequency as alpha but over **sensorimotor** cortex, and it responds to movement rather than vision |
| **Beta** | **13 to 30 Hz** | Active concentration; also motor cortex |
| Gamma | above 30 Hz | Contested; heavily contaminated by muscle activity at the scalp |

Alpha and mu occupy the same frequency range but are different rhythms with different
generators and different triggers. Confusing them is common. Mu is your target.

The 8 to 30 Hz bandpass in this project is simply "mu plus beta", i.e. the two bands
that show motor-related changes.

## The 10-20 system

The standard naming scheme for electrode positions. Letters give the lobe, numbers give
the position:

* **F** frontal, **C** central, **P** parietal, **O** occipital, **T** temporal
* Odd numbers on the **left**, even numbers on the **right**, **z** on the midline
* Combined letters mark intermediate positions: **AF** is between frontal and frontopolar,
  **TP** is between temporal and parietal

The positions you care about:

| Position | Where | Relevance |
|---|---|---|
| **C3** | Left central | Over left motor cortex, controls the **right** hand |
| **C4** | Right central | Over right motor cortex, controls the **left** hand |
| **Cz** | Vertex | Over foot/leg motor area |
| **TP9, TP10** | Behind the ears | Muse's rear channels. Also over the temporalis muscle |
| **AF7, AF8** | Forehead | Muse's front channels. Close to the eyes |

**Do the drawing exercise in Phase 0.** Sketch a head from above, mark C3, C4, Cz, and
then the four Muse positions. The distance between C3 and the nearest Muse electrode is
the single most important fact about this project.

## Common mistakes

* Assuming an electrode reads the cortex directly beneath it. It does not.
* Forgetting that everything is relative to the reference.
* Mixing up alpha and mu because they share a frequency range.
* Trusting microvolt values without checking the unit convention of the library.
* Expecting to see anything meaningful in a single trial. EEG needs repetition and
  averaging (or a good classifier) to pull signal out of noise.

## Questions

Write your answers in the boxes. See
[the convention](README.md#answering-the-questions).

**Q1.** Why does a single neuron's firing not show up in EEG?
*Source: this document. `reviewed: yes`*

> **Answer:** The EEG measures the summed response of many neurons, a single neurons
> firing would just drown in the noise of the others so you cant accurately predict it
>
> **Check:** Partly correct. Summation is the right idea, but the reason is amplitude,
> not drowning in noise: one neuron's field is so small that it is unmeasurable at the
> scalp even in silence, because it has to cross skull and scalp. Two things to sharpen.
> First, EEG reflects **postsynaptic potentials**, not firing (action potentials); spikes
> are too brief and too asynchronous to sum. Second, "drown in the noise of the others"
> gets the mechanism backwards, since it is precisely the others being *in step* that
> creates the signal. Matters downstream: ERD is a change in synchrony, so if you think
> of the other neurons as noise you will read a power drop as "less activity".

**Q2.** Your Muse data has values around 15000. What went wrong?
*Source: this document. `reviewed: yes`*

> **Answer:** Mixed up volts and microvolts?
>
> **Check:** Correct, but be specific about the direction and the factor. Real scalp EEG
> is 10 to 100 µV, so 15000 is about a hundred times too large for a µV reading and
> absurdly large for a volt reading. The usual concrete case: BrainFlow hands you
> microvolts and you build an MNE `RawArray` from it without dividing by 1e6, so MNE
> labels the axis in volts and every plot scale is wrong by a million. One alternative
> worth ruling out before you blame units: an unfiltered channel carries a large DC
> offset, so 15000 can also be a real µV value that is mostly offset, which a high-pass
> removes. Check the peak-to-peak *after* filtering, not the raw values.

**Q3.** Which hemisphere's motor cortex controls your left hand, and which 10-20
electrode sits over it?
*Source: this document. `reviewed: yes`*

> **Answer:** C4, right hemisphere
>
> **Check:** Correct.

**Q4.** Why might an eye blink appear on TP9, which is behind your ear?
*Source: this document. `reviewed: yes`*

> **Answer:** Everything is measured against the reference, in this case FPz which is
> close to the eyes. The eyes send a significantly stronger signal then the one from
> the brain, so it can be confused with the reference signal
>
> **Check:** Correct on the mechanism that matters. One sharpening: it is not
> "confusion", it is arithmetic. Every channel is literally TP9 minus FPz, so whatever
> the eye does to FPz appears on TP9 with the opposite sign, on every channel, always.
> There is no signal-processing step that separates it, because it is not contamination
> added to the channel, it is part of the channel's definition. Volume conduction adds a
> smaller second route by which the blink reaches TP9 directly.

**Q5.** What is the difference between alpha and mu?
*Source: this document. `reviewed: yes`*

> **Answer:** mu responds to movement contra alpha responding to vision, and it is over
> the sensorimotor cortex
>
> **Check:** Correct. Worth stating the trap explicitly for the report: they occupy the
> *same* 8 to 13 Hz band, so a spectrum alone cannot tell them apart. Only the location
> (occipital vs. sensorimotor) and what modulates them (eyes closed vs. movement) can.
> On a Muse, with no sensorimotor electrode, a 10 Hz peak is far more likely to be
> occipital alpha volume-conducted forward than mu.

**Q6.** Two conditions must hold for neural activity to be visible at the scalp at all.
Name both, and say which one ERD depends on.
*Source: this document. `reviewed: yes`*

> **Answer:** Spatial alignment and Temporal synchrony: ERD depends on temporal synchrony
>
> **Check:** Correct.

**Q7.** MNE has three central data objects for continuous, segmented and averaged data.
Name them and say what each one holds.
*Source: MNE, "Overview of MEG/EEG analysis". `reviewed: no`*

> **Answer:** _(unanswered)_

**Q8.** According to BrainFlow's "Supported Boards" page, what does the Muse board
provide besides the four EEG channels, and what do you have to do before those extra
channels appear in the data matrix?
*Source: BrainFlow, "Supported Boards". `reviewed: no`*

> **Answer:** _(unanswered)_

**Q9.** The skull is a much poorer conductor than the scalp and the brain. What does
that difference do to a cortical source on its way to the electrode, and why does it
mean an electrode cannot be said to measure "the cortex underneath it"?
*Source: this document. `reviewed: yes`*

> **Answer:**  IDK
>
> **Check:** Wrong, but this one is worth learning properly because it is the physical
> reason the Muse cannot do what you want. Current takes the path of least resistance.
> The skull conducts far worse than the brain and scalp on either side of it, so instead
> of crossing it straight up to the nearest electrode, the current spreads sideways
> through the CSF and scalp first. One focal cortical source therefore arrives at the
> surface as a broad, blurred patch several centimetres across. That has two effects at
> once: every source lands on several electrodes, and every electrode sums several
> sources. So "the signal at TP9" is a weighted mixture whose weights you do not know,
> and no filter recovers the individual sources from it. Separating them needs *many*
> electrodes, which is exactly what CSP does and exactly what four channels deny you.
> The volume-conduction section above now covers this; reread it.

**Q10.** EEG and fMRI measure brain activity with opposite strengths. What does EEG buy
you, and what does it give up? Relate your answer to why this project needs repeated
trials.
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q11.** Buzsáki et al. argue that the extracellular field is dominated by *postsynaptic*
currents rather than by action potentials. Given that a spike is by far the larger event
at the cell itself, why does it contribute so little at the scalp? Name both reasons.
*Source: Buzsáki, Anastassiou & Koch (2012). `reviewed: no`*

> **Answer:** _(unanswered)_

## Sources

Two tiers. **Tier 1** is what you read before Phase 0, and it is short on purpose.
**Tier 2** is what you open when a Tier 1 source assumed something you did not have, or
when you are writing the report and want a citation.

### Tier 1

* **MNE-Python, "Overview of MEG/EEG analysis"**,
  https://mne.tools/stable/auto_tutorials/intro/10_overview.html
  Gets you from raw data to results in one page, with real data. The reason it is here
  rather than in a later topic is that it introduces the three objects every other MNE
  tutorial assumes you know. Read it at a keyboard, not on the sofa.
* **BrainFlow, "Supported Boards"**,
  https://brainflow.readthedocs.io/en/stable/SupportedBoards.html
  Find the Muse section. Five minutes, and it settles the channel names, sampling rate,
  reference and board IDs that the rest of the project depends on.

### Tier 2

* **Buzsáki, G., Anastassiou, C. A., & Koch, C. (2012). "The origin of extracellular
  fields and currents: EEG, ECoG, LFP and spikes."** *Nature Reviews Neuroscience*,
  13(6), 407-420. The modern answer to "what is this signal, physically". Read the first
  few sections only; the rest is about intracranial recording. Worth it because it is the
  citation for every claim in the "what the signal actually is" section above.
* **Malmivuo & Plonsey, "Bioelectromagnetism"**, free full text at
  https://www.bem.fi/book/
  Chapters 13 and 14 are the actual physics of volume conduction rather than a hand-wave.
  Open it if you want the conductivity numbers and the derivations behind the smearing
  argument. Dense, and not needed to do anything.
