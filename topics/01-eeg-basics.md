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

Two consequences that matter for this project:

* An electrode does not measure "the brain underneath it". It measures a weighted mix
  of many sources, near and far. This is called **volume conduction**.
* Because of the smearing, spatial resolution is poor (centimetres, not millimetres),
  and you need many electrodes to separate sources. With 4 electrodes you cannot do
  much spatial separation at all. This is the core reason
  [07 CSP](07-csp.md) will struggle on Muse data.

The upside of volume conduction: a source over motor cortex is not *completely*
invisible at TP9/TP10, just heavily attenuated and mixed with everything else. That is
the thin thread this project hangs on.

## Reference, ground, and why it matters

Voltage is always a difference between two points. There is no such thing as "the
voltage at TP9"; there is only "TP9 minus the reference". This means:

* Changing the reference changes the shape of your signal everywhere. A "signal" you
  see at one electrode may actually be activity at the reference.
* The Muse references to **FPz** (on the forehead). That location is close to the eyes,
  so eye artifacts contaminate *all* Muse channels, not just the frontal ones.
* Common re-referencing schemes include average reference, linked mastoids, and
  Laplacian. With 4 channels your options are limited, but be aware the choice exists.

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

## Check yourself

1. Why does a single neuron's firing not show up in EEG?
2. Your Muse data has values around 15000. What went wrong?
3. Which hemisphere's motor cortex controls your left hand, and which 10-20 electrode
   sits over it?
4. Why might an eye blink appear on TP9, which is behind your ear?
5. What is the difference between alpha and mu?

## Sources

### Start here

* **MNE-Python, "Overview of MEG/EEG analysis"**,
  https://mne.tools/stable/auto_tutorials/intro/10_overview.html
  Gets you from raw data to results in one page, with real data.
* **Malmivuo & Plonsey, "Bioelectromagnetism"**, free full text at
  https://www.bem.fi/book/
  Chapters 13 and 14 cover the physics of EEG generation and volume conduction
  properly. Dense, but it is the actual physics rather than a hand-wave.
* **BrainFlow, "Supported Boards"**,
  https://brainflow.readthedocs.io/en/stable/SupportedBoards.html
  Confirms the Muse channel names, sampling rate and reference.

### Go deeper

* **Mike X Cohen, "Analyzing Neural Time Series Data"** (MIT Press, 2014), chapters 1
  to 5. The best single explanation of what the signal is and what you can legitimately
  conclude from it. Free companion lectures at https://www.youtube.com/@mikexcohen1
* **Steven Luck, "An Introduction to the Event-Related Potential Technique"**
  (MIT Press). Chapter 1 and the appendix on electrical basics are worth reading even
  though this project is not about ERPs. Luck is unusually good on referencing.
* **Nunez & Srinivasan, "Electric Fields of the Brain"** (Oxford University Press).
  The authoritative treatment of volume conduction. Reference, not a read-through.

### Papers

* Jasper, H. H. (1958). "The ten-twenty electrode system of the International
  Federation." The original definition of the electrode naming scheme.
* Buzsáki, G., Anastassiou, C. A., & Koch, C. (2012). "The origin of extracellular
  fields and currents: EEG, ECoG, LFP and spikes." *Nature Reviews Neuroscience*,
  13(6), 407-420. The modern reference on where the signal comes from.

### Video

* **Essentials of Neuroscience with MATLAB: Module 2-3 (EEG)**,
  https://www.youtube.com/watch?v=mGudmVJQrLg
  A short lecture on what the EEG signal is, where it comes from and what the
  classical frequency bands mean. The examples are in MATLAB, but nothing in the
  concepts depends on the language.
