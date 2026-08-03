# 02. Motor imagery and ERD

**Needed in:** Phase 0, Phase 4
**In one sentence:** When you move or imagine moving a limb, the mu and beta rhythms
over the opposite side's motor cortex lose power, and that lateralised power drop is
the only thing distinguishing "left" from "right" in this project.

## Why this matters here

This is the physical effect the entire project detects. Everything else (filtering,
CSP, LDA, the game) is machinery built to measure it. If you only properly understand
one topic document, make it this one.

## ERD and ERS

**ERD** is Event-Related Desynchronization: a *decrease* in band power, time-locked to
an event. **ERS** is the opposite, an *increase*.

The mechanism follows directly from [01](01-eeg-basics.md). EEG amplitude depends on
how synchronised a neural population is, not on how active it is:

* **Idling cortex** is synchronised. Large populations oscillate together in the
  8 to 13 Hz range, the individual dipoles add up, and you measure a large mu rhythm.
* **Active cortex** is desynchronised. Neurons start doing different things at
  different times for the task at hand, the summation breaks down, and measured
  amplitude *drops*.

So ERD means "this piece of cortex just got busy". Counterintuitively, activation shows
up as **less** signal, not more.

Timing, roughly:

* Mu and beta ERD begins about 0.5 to 2 s *before* a self-paced movement, and persists
  through it.
* After the movement ends there is a **beta rebound** (also called post-movement beta
  ERS), a sharp power increase in the beta band lasting a second or so. This is often
  one of the strongest and most reliable motor signals in the whole recording, and it
  is worth knowing about because it can be more detectable than the ERD itself.

## Somatotopy and why left vs. right is decodable at all

The motor cortex is laid out somatotopically: adjacent body parts map to adjacent
cortical strips (the "motor homunculus"). Crucially, control is **contralateral**:

* Imagining the **right** hand produces ERD over the **left** hemisphere, at **C3**.
* Imagining the **left** hand produces ERD over the **right** hemisphere, at **C4**.

This lateralisation is what makes the two classes separable. It is not that left-hand
imagery produces a different *kind* of signal; it produces the *same* signal in a
different *place*. Classification is therefore fundamentally a **spatial** problem,
which is precisely why [CSP](07-csp.md), a spatial filtering method, is the standard
approach.

**This is also why the Muse is a bad fit.** The discriminative information is a
left/right difference in power at C3 vs. C4. The Muse has no electrode near either. It
has TP9 and TP10, which are lateralised but several centimetres away, and AF7/AF8,
which are lateralised but frontal. Whatever you detect will be volume-conducted
leakage, not the effect itself. Say so plainly in the report.

The hand areas are also unhelpfully close together on the cortical surface, only a few
centimetres apart, which is near the limit of EEG's spatial resolution even with 64
channels. Hands are used anyway because they are the easiest body parts for a subject
to imagine vividly.

## Motor imagery vs. motor execution

**Motor execution** is actually moving. **Motor imagery** is imagining the movement
without producing it. They activate overlapping cortical areas, and imagery produces
ERD in the same place as execution, just weaker.

For this project the distinction matters practically:

* Execution gives a **much** stronger and more reliable signal. Start there in Phase 4
  as a sanity check on your whole chain.
* Execution also produces **EMG contamination** from the actual muscle activity, and
  possibly movement artifacts. A classifier can succeed on execution data by reading
  muscles rather than brain. That is fine for a sanity check, but never report it as a
  BCI result.
* Imagery is the real target: no muscle activity, so a positive result is more
  defensible. It is also considerably harder.

## Kinaesthetic vs. visual imagery

There are two distinct ways to imagine a movement, and they are not equivalent:

* **Visual motor imagery**: picturing the hand moving, as if watching it. Recruits
  visual areas. Produces weak ERD.
* **Kinaesthetic motor imagery**: *feeling* the movement from the inside, the sense of
  muscle tension and proprioception, without seeing anything. Recruits sensorimotor
  cortex. Produces much clearer ERD.

Neuper et al. (2005) showed kinaesthetic imagery gives substantially better single-trial
classification. **The instruction you give the subject is therefore an experimental
variable**, and it belongs in your method section. Something like "feel the sensation of
squeezing a ball in your left hand, do not picture it" rather than "think about your
left hand".

## BCI illiteracy

Somewhere around 15 to 30 percent of people produce motor imagery signals too weak to
control an SMR-based BCI, even with research-grade hardware and training. This is
usually called "BCI illiteracy" or "BCI inefficiency".

Blankertz et al. (2010) found that the strength of a person's resting mu rhythm
predicts their BCI performance, which gives you a cheap pre-test: record two minutes of
eyes-open rest and look for a peak around 10 Hz over sensorimotor channels. If there is
no mu rhythm to suppress, there is no ERD to detect.

For a single-subject project this is a real risk to name in the report. If your results
are poor, you cannot distinguish "the Muse cannot see it" from "this subject has weak
mu" without additional evidence. Running Phase 3 on public data and, if possible,
testing a second subject are how you address it.

## Common mistakes

* Expecting activation to look like *more* signal. It is less.
* Looking only at the imagery window and missing the beta rebound after it.
* Reversing the hemispheres. Right hand is left brain.
* Instructing the subject to "imagine" without specifying kinaesthetic imagery.
* Treating a good motor-execution result as evidence the imagery pipeline works.
* Assuming a null result means the method is broken, when it may be the subject or the
  hardware.

## Questions

Write your answers in the boxes. See
[the convention](README.md#answering-the-questions).

**Q1.** Why does cortical activation *reduce* EEG amplitude?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q2.** You see strong ERD at C4. Which hand was the subject imagining?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q3.** What is the beta rebound and when does it occur?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q4.** Why is left vs. right classification a spatial problem rather than a spectral
one?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q5.** You get 95 % accuracy on motor execution data from the Muse. What are the two
competing explanations, and how would you tell them apart?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q6.** What is the practical difference between kinaesthetic and visual motor imagery,
and what exactly would you say to a subject to get the first one?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q7.** In the PhysioNet EEG Motor Movement/Imagery dataset, which run numbers contain
*imagined* left vs. right fist movement, and what do the annotation codes T0, T1 and T2
mean in those runs?
*Source: PhysioNet, EEG Motor Movement/Imagery Dataset. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q8.** How many subjects, channels and samples per second does that dataset have, and
how long is a single task period? You will need all four numbers in Phase 3.
*Source: PhysioNet, EEG Motor Movement/Imagery Dataset. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q9.** In MNE's "Decoding motor imagery with CSP" example, which frequency band and
which `tmin`/`tmax` are used, and what classification accuracy does the page report?
*Source: MNE, "Decoding motor imagery with CSP". `reviewed: no`*

> **Answer:** _(unanswered)_

**Q10.** Wolpaw's definition of a BCI has a criterion that deliberately excludes systems
driven by muscle activity. State the criterion, and explain what it means for a Muse
result that turns out to be driven by jaw EMG.
*Source: Wolpaw & Wolpaw, "Brain-Computer Interfaces: Principles and Practice",
sensorimotor rhythm chapters. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q11.** Find one motor imagery project built on a consumer 4-channel headset (start
from NeuroTechX). What accuracy did they claim, and did they compare it against a chance
level computed for their trial count? What does that tell you about the numbers you will
see quoted online?
*Source: NeuroTechX. `reviewed: no`*

> **Answer:** _(unanswered)_

## Sources

### Start here

* **MNE-Python, "Decoding motor imagery with CSP"**,
  https://mne.tools/stable/auto_examples/decoding/decoding_csp_eeg.html
  The full pipeline on real motor imagery data. Run it in Phase 3.
* **PhysioNet, EEG Motor Movement/Imagery Dataset**,
  https://physionet.org/content/eegmmidb/1.0.0/
  109 subjects, 64 channels, both executed and imagined left/right hand. Read the
  dataset description page; it documents a working experimental protocol you can borrow.

### Go deeper

* **Wolpaw & Wolpaw (eds), "Brain-Computer Interfaces: Principles and Practice"**
  (Oxford University Press). The chapters on sensorimotor rhythms are the standard
  treatment of exactly this paradigm.
* **NeuroTechX**, https://neurotechx.com/
  Community around consumer neurotech, including many Muse-based projects. Useful for
  calibrating what is realistically achievable with 4 channels.

### Papers

* Pfurtscheller, G., & Lopes da Silva, F. H. (1999). "Event-related EEG/MEG
  synchronization and desynchronization: basic principles." *Clinical Neurophysiology*,
  110(11), 1842-1857. **The foundational paper on ERD/ERS.** Read this one properly.
* Pfurtscheller, G., & Neuper, C. (2001). "Motor imagery and direct brain-computer
  communication." *Proceedings of the IEEE*, 89(7), 1123-1134. The paper that framed
  motor imagery as a BCI control signal.
* Neuper, C., Scherer, R., Reiner, M., & Pfurtscheller, G. (2005). "Imagery of motor
  actions: Differential effects of kinesthetic and visual-motor mode of imagery in
  single-trial EEG." *Cognitive Brain Research*, 25(3), 668-677. The evidence behind
  the kinaesthetic instruction.
* Blankertz, B., Sannelli, C., Halder, S., et al. (2010). "Neurophysiological predictor
  of SMR-based BCI performance." *NeuroImage*, 51(4), 1303-1309. BCI illiteracy and
  how to predict it from resting mu.
* Pfurtscheller, G., & Solis-Escalante, T. (2009). "Could the beta rebound in the EEG
  be suitable to realize a 'brain switch'?" *Clinical Neurophysiology*, 120(1), 24-29.
  On using the post-movement rebound as a control signal.

### Video

* **Introduction to Motor-Imagery based Brain-Computer Interfaces**,
  https://www.youtube.com/watch?v=metlFBa_NdQ
  An overview of the motor imagery paradigm: the mu and beta rhythms, ERD and ERS,
  and how the whole thing is turned into a control signal. Good orientation before
  reading the Pfurtscheller papers below.
