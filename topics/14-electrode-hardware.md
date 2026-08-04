# 14. Extending the montage: adding an electrode to the Muse

Every other document in this folder takes the hardware as given: four electrodes at TP9,
AF7, AF8 and TP10, none of them over sensorimotor cortex. This one asks what happens if
you change that, because the Muse has one input you have not used yet.

Read this before Phase 4B in [LEARNING_PLAN.md](../LEARNING_PLAN.md). It assumes
[01. EEG basics](01-eeg-basics.md) (reference, montage, impedance),
[02. Motor imagery and ERD](02-motor-imagery-erd.md) (why C3 and C4 specifically) and
[03. Artifacts](03-artifacts.md) (what a bad electrode looks like in the data).

## 1. What you are trying to add and why

Hand motor imagery produces ERD in the mu and beta bands over the hand area of primary
motor cortex, which in the 10-20 system is C3 for the right hand and C4 for the left.
The effect is strongest directly over the source and falls off quickly with distance,
because skull and scalp smear the signal but do not transport it far with any strength.
TP9 and TP10 sit several centimetres away, behind and below, so what reaches them is a
volume-conducted remainder of an effect that is not large to begin with.

Adding an electrode at C3 does not improve the method, the filters or the classifier. It
changes the one thing that no amount of signal processing can fix, which is whether the
information is present in the recording at all. That is why this is worth doing and also
why it is the only hardware change worth considering.

## 2. What the Muse actually offers: one channel, not four

The Muse has an auxiliary input on the micro-USB port at the back right of the headband,
the same port you charge it through. It is a single-ended EEG input, exposed in the data
stream as the fifth EEG value and usually called **AUX Right**.

**In BrainFlow.** The default Muse preset streams four EEG channels. The fifth value has
to be switched on with a board configuration command *before* the stream starts:

* `p21` gives 4 EEG channels plus IMU. This is the default.
* `p20` gives 5 EEG values plus IMU.
* `p51` gives 4 EEG channels plus IMU plus PPG.
* `p50` gives 5 EEG values plus IMU plus PPG.

In this installation, `BoardShim.get_eeg_channels(38)` returns `[1, 2, 3, 4]` and
`BoardShim.get_other_channels(38)` returns `[5]`. Row 5 is where the aux value lands. It
is classified as "other" rather than "eeg" precisely because BrainFlow cannot know
whether anything is plugged in. The exact call is in [SYNTAX.md](../SYNTAX.md#brainflow).

**The physical connector.** Community tutorials for the Muse 2016 solder the electrode
lead to pin 4 of the 5-pin micro-USB B connector, the ID pin. Those tutorials were
written for the 2016 model. The Muse 2 is widely reported to expose the same input on
the same port, and BrainFlow lists the same presets for both, but you have not verified
it on *your* unit, so treat it as an assumption until section 6 confirms it.

**Buy or build.** Interaxon sells a ready-made single-cup auxiliary electrode with a
micro-USB connector for about 50 CAD. The DIY route costs around a third of that in
parts and needs soldering. Buy the finished one. The failure mode of a hand-soldered
electrode is a bad joint that produces plausible-looking noise instead of an obvious
break, and you would spend days trying to work out whether the flat result at C3 is
physiology or a cold joint.

**The hard limit: there is one aux input, not two.** You cannot record C3 and C4 at the
same time. Everything below follows from that.

## 3. What one extra channel buys, and what it does not

| Task with the aux at C3 | Feasible | Why |
|---|---|---|
| Right-hand imagery vs. rest | Yes, and this is the best bet | Contralateral mu ERD at C3 against a baseline. One channel is enough because the contrast is power at one site over time, not a comparison between sites |
| Left vs. right hand imagery | Weakly | ERD at C3 is stronger for the right hand than the left, so the classes differ in degree at a single site rather than in which site responds. Expect a small effect |
| Left vs. right, using C3 and C4 together | No | Requires two aux inputs. The classic lateralisation feature is not available to you |
| Foot or tongue imagery | Not with C3 | Foot imagery localises near Cz, which would need the aux moved and a different protocol |

Two consequences worth being clear about before you spend money.

**The project's headline question does not change.** Research question 1 in
[REPORT.md](../REPORT.md) is about left vs. right hand imagery, and one central
electrode gives you a weak version of that contrast, not a strong one. What the aux
gives you instead is a second, cleaner question that this hardware can actually answer:
does imagery versus rest produce measurable ERD at a real motor site. A rest-versus-
imagery result at C3 is a genuine BCI by Wolpaw's definition, it is a real control
signal, and single-channel ERD switches are a well-established design in the literature.
It is a smaller claim than left versus right, and far more likely to be true.

**CSP becomes slightly less pointless, and still is not the interesting part.** With
five channels CSP has one more dimension to work with, and for the first time one of
those dimensions sits over motor cortex. But for imagery versus rest at a single site,
the classical feature is simply log band power at C3 in 8:13 Hz, relative to a baseline
window. Run that as a baseline feature alongside CSP. If CSP does not beat plain band
power on five channels, say so in the report. That is a result, not a failure.

## 4. The electrical reality

**Reference.** The Muse references everything to FPz, the conductive strip in the middle
of the forehead. So the aux does not record "C3", it records the potential difference
C3 minus FPz. The distance is large, which is fine and in fact gives you more amplitude,
but it also means frontal activity, including every blink and every eye movement, enters
the aux channel with the same polarity logic as it enters the four built-in channels.
An eye artifact at C3 does not mean the electrode is misplaced.

**Impedance and hair.** TP9, AF7, AF8 and TP10 are all on bare skin. C3 and C4 are under
hair, and this is the single biggest practical obstacle. Dry electrodes rely on direct
skin contact, and hair prevents it, which is why "long hair" spike electrodes exist and
why they still perform worse than a wet contact. A silver/silver-chloride cup electrode
with conductive paste, pressed onto skin exposed by parting the hair, is the reliable
option. The Muse has nothing to hold it in place, so plan the mechanical fixing before
you order anything: an elastic band, a bandana or a swim cap over the top.

**You have no impedance readout.** Research amplifiers measure electrode impedance and
tell you when a contact is bad. The Muse tells you nothing about the aux, because the
firmware does not know it exists. Your only measure of contact quality is the signal
itself, which is why the acceptance tests in section 6 are not optional.

**The lead is an antenna.** An unshielded wire running from the headband to C3 picks up
mains hum capacitively, and any movement of the wire changes its capacitance to
everything around it, producing low-frequency swings that look nothing like EEG. Two
testable predictions follow, and both belong in the report as measured numbers:

* the 50 Hz peak on the aux channel will be larger than on the four built-in channels,
  partly from the lead and partly because a mismatch in impedance between the aux and
  the reference degrades common-mode rejection specifically on that channel
* the aux channel will show more low-frequency drift, and it will correlate with head
  movement rather than with anything physiological

Tape the lead down along its whole length. Most of the "the extra electrode is unusable"
reports come from a swinging wire.

## 5. Safety and the things that go wrong physically

* **Never charge the headband while wearing it, and never with the electrode attached.**
  The aux input and the charging port are the same port. A battery-powered device on your
  head is isolated from the mains; a charging one is not. This is the only genuine safety
  rule in this document and it is not negotiable.
* Do not connect the aux lead to anything other than an electrode on your own skin.
* Only ever use this on yourself. Recording another person turns a hobby project into
  something with consent and ethics requirements you have not set up.
* Opening the headband voids the warranty and is not needed. Everything here uses the
  port as it was designed to be used.
* The port is fragile and was designed for a charging cable that gets plugged in twice a
  week, not for a lead under tension. Strain-relieve it.

## 6. Acceptance tests, in order, before you trust a single number

Each test tells you something the previous one could not, so run them in sequence and
stop at the first failure. Record and keep the data for every one of them; several
become figures.

1. **Does the channel exist?** Configure `p20` or `p50`, start the stream with nothing
   plugged into the port, and look at row 5 for 30 seconds. Whatever a disconnected
   input looks like on your unit, that is your "not connected" reference picture.
2. **Is the signal path real?** Attach the electrode and place it on the mastoid right
   next to TP9, a few centimetres away. Record 60 seconds, bandpass both channels at
   1:40 Hz, and correlate row 5 against the TP9 row. Two electrodes that close should
   agree strongly. If they do not correlate, the connector, the pin or the electrode is
   wrong, and nothing after this point is interpretable.
3. **Does it record brain?** Move the electrode to Oz, at the back of the head just
   above the bump of the occipital bone. Record 60 seconds with eyes open and 60 with
   eyes closed, then plot the power spectrum of each. Eyes closed must produce an
   obvious peak in 8:13 Hz, several times larger than eyes open. Occipital alpha is the
   largest, most reliable effect in scalp EEG, and if you cannot see it here, your
   electrode is not recording cortex and no motor imagery result from it will mean
   anything. **This is the test that decides whether the whole modification worked.**
4. **What did it cost in noise?** With the electrode finally at C3, compare the aux
   channel against TP9: height of the 50 Hz peak, broadband amplitude, drift. Write the
   ratios into the report. This is what quantifies "consumer headband plus a hand-placed
   electrode" against its own built-in channels.
5. **Does it see movement?** Motor *execution* first, never imagery. Right fist clench
   versus rest, 30 trials each, and look for mu power dropping during the clench in a
   time-frequency plot. Execution produces the largest ERD you will ever record. If test
   3 passed and this fails, suspect the electrode position before you suspect the brain,
   then move on to imagery knowing the chain works.

## 7. Can the one input be made into two?

The obvious next thought is to get more out of the port: two electrodes on the one pin,
or a switch that alternates between them. Both are buildable. Neither gives you what the
left-versus-right task needs, and it is worth understanding exactly why before spending
an evening with a soldering iron.

### The ceiling is the firmware, not the pins

The Muse transmits a fixed BLE packet, and the largest EEG payload its firmware offers
is five values, which is what `p20` and `p50` select. BrainFlow decodes that packet; it
cannot decode a sixth number that was never sent. So no amount of soldering, and no
spare input that may or may not exist inside the headband, can raise the channel count.
The limit is a closed packet format, and it is not reachable from outside the device.
That single fact ends the "modify the Muse further" line of thought.

### Two electrodes wired to the same input

Physically straightforward: both leads go to the same pin. What the amplifier then sees
is one node whose potential is a weighted average of the two sites, weighted by the two
contact impedances, which you do not know, cannot set and which drift as the paste
dries. Three consequences:

* For **imagery versus rest** this is defensible. Mu ERD is bilateral with a
  contralateral bias, so an average of C3 and C4 still drops during imagery, and the
  pair may pick up slightly more of it than one electrode alone.
* For **left versus right** it is worse than useless. Averaging the two hemispheres
  sums the desynchronised side with the synchronised one, so the two classes converge on
  the same value. You would be building hardware whose specific purpose is to cancel your
  own contrast.
* Wiring two scalp sites together also shunts current between them through the wire,
  which attenuates the hemispheric asymmetry itself rather than merely failing to
  measure it. This is the standard objection to physically linked-ear references, and it
  bites harder here because C3 and C4 sit directly over the sources you care about.

Worth one recording as a variant of the rest-versus-imagery task. Never for left versus
right.

### Switching the input between C3 and C4

An analog switch alternating the aux between two electrodes is buildable, and it fails
for three separate reasons. Every switch produces a DC step from the difference in
electrode half-cell potentials, which takes a good fraction of a second to settle
through the amplifier's high-pass, so much of the recording is transient rather than
signal. You halve the samples per site. And decisively, CSP and every other spatial
filter work on the covariance *between channels at the same instant*, which
time-multiplexed data does not contain. You would have built an elaborate way to record
two sites that can never be combined spatially, which is the only thing you wanted them
for.

### Move the electrodes you already have, first, for nothing

There is one experiment here that costs zero and involves no hardware at all: wear the
headband in a different position and measure what changes. Sliding it up and back moves
TP9 and TP10 toward T7 and T8, which are closer to the hand area, and moves the FPz
reference with them. Contact will get worse as the electrodes meet hair and the
reference may fail outright, which is precisely why this is an experiment and not a
plan. One session, the Phase 1 signal-quality check and the alpha test from section 6
tell you whether the new position is usable at all. If a two-centimetre shift is worth
two points of accuracy, that is a result, and if it destroys the signal that is also a
result. Do this before ordering anything.

## 8. The real answer: a second device

If you want C3 and C4 at the same time, you need an amplifier that has two inputs to
give. BrainFlow already supports a long list of them with the same API you are writing
against, so the acquisition code you write in Phase 1 mostly survives the switch:

| Board | `BoardIds` | Channels | Rate |
|---|---|---|---|
| Ganglion | `GANGLION_BOARD` = 1 | 4 | 200 Hz |
| Cyton | `CYTON_BOARD` = 0 | 8 | 250 Hz |
| Cyton + Daisy | `CYTON_DAISY_BOARD` = 2 | 16 | 125 Hz |
| PiEEG (Raspberry Pi shield) | `PIEEG_BOARD` = 56 | 8 | 250 Hz |
| FreeEEG32 | `FREEEEG32_BOARD` = 17 | 32 | 512 Hz |

Verify the channel counts yourself with `BoardShim.get_eeg_channels(board_id)` rather
than trusting this table, and check current prices and availability before planning
around any of them. The ADS1299-based hobby boards in particular come and go.

**You do not have to synchronise it with the Muse.** The instinct is to run both devices
together and merge the streams, which means two clocks, two Bluetooth stacks and drift
you would have to correct for. Decline the problem: a second device *replaces* the Muse
for the motor imagery condition, while the Muse keeps doing the eye/jaw condition and
the game. Two separate recordings, one pipeline, two rows of the ladder.

**Safety, and it is a different rule from the Muse's.** The Muse is a sealed,
battery-powered device. A bare amplifier board wired to a laptop that is plugged into
the mains puts you on the same electrical circuit as the building, through electrodes on
your scalp. Run the laptop on battery, or use a device with galvanic isolation, and read
the manufacturer's safety documentation properly before the first recording. This is the
point at which a hobby project acquires a real hazard.

Recommendation, in order: try the free repositioning experiment, spend the 50 CAD on the
aux electrode and run Phase 4B, and let those measurements decide whether the project has
earned a real amplifier. If the alpha test in section 6 passes and C3 shows execution
ERD, you will have the argument for better hardware backed by your own data rather than
by hope.

## Common mistakes

* **Placing C3 by eye.** C3 is not "somewhere on the left side". Measure it: halfway
  between nasion and inion for the coronal line, then 10, 20, 20 per cent across from
  the left preauricular point. A few centimetres of error puts you over a different
  functional area, and this is a project where a few centimetres is the entire point.
* **Trusting the aux because it produces a plausible waveform.** A floating, unconnected
  EEG input produces something that looks like noisy EEG to the untrained eye. Test 3
  exists because plausibility is not evidence.
* **Pooling four-channel and five-channel recordings into one training set.** They have
  different feature dimensionality and come from different sessions with different
  electrode placement. They are different datasets and belong to different rows of the
  ladder.
* **Comparing the five-channel result against the four-channel result from another day.**
  Session variability alone is worth several points of accuracy. To attribute a gain to
  the extra electrode, you need both conditions from the same session, which you can do
  by recording with the aux attached and then simply dropping row 5 in the analysis.
  That comparison is within-session, within-subject and within-pipeline, and it is the
  cleanest experiment in the entire project.
* **Assuming more electrodes means more accuracy.** A fifth channel with worse contact
  than the other four can lower accuracy by adding a noisy dimension for CSP to overfit
  to. This is one of the things you are measuring, not something you can assume away.
* **Buying a different amplifier before finishing Phase 3.** If the pipeline is not
  validated on public data, better hardware will only produce a better-looking wrong
  answer.

## Questions

**Q1.** Why does adding an electrode at C3 address a problem that no filter, feature or
classifier can address?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q2.** You have one aux input. Explain why that rules out the standard left-versus-right
lateralisation feature, and what task you can run instead.
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q3.** The aux is referenced to FPz like every other Muse channel. What does that mean
for eye artifacts appearing on the C3 channel, and why is that not evidence of bad
placement?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q4.** Why is the eyes-closed alpha test at Oz the test that decides everything, and
what exactly would you plot to pass or fail it?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q5.** You record with five channels on Monday and compare against a four-channel
recording from the previous Thursday. Name two reasons the difference cannot be
attributed to the extra electrode, and describe the design that fixes it.
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q6.** Give two independent reasons the aux channel is expected to show a larger 50 Hz
peak than TP9.
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q7.** Why must the headband never be charging while an aux electrode is on your head?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q8.** Which BrainFlow preset codes give five EEG values on a Muse 2, what else does
each of them switch on, and at what point in the session must the call be made?
*Source: BrainFlow, "Supported Boards", Muse section. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q9.** BrainFlow reports the fifth value through `get_other_channels()` rather than
`get_eeg_channels()`. What does that tell you about what the library knows, and what
does it mean for how your loading code should treat row 5?
*Source: this document plus BrainFlow, "Supported Boards". `reviewed: no`*

> **Answer:** _(unanswered)_

**Q10.** Krigolson et al. validated the stock Muse against research equipment. What did
they successfully measure with it, what did they have to do to their protocol to get
there, and what does their result say about the baseline your modified headband has to
beat before the extra electrode counts as an improvement?
*Source: Krigolson et al. (2017). `reviewed: no`*

> **Answer:** _(unanswered)_

**Q11.** MNE distinguishes a montage from a layout. State the difference, and say which
one you would need in order to plot a five-channel Muse CSP pattern as a topography, and
whether that plot would be worth making.
*Source: MNE, "EEG sensor locations". `reviewed: no`*

> **Answer:** _(unanswered)_

**Q12.** You solder a second electrode onto the same aux pin, at C4, and record left
versus right hand imagery. Explain what the amplifier now measures, and why this
particular combination of hardware and task is self-defeating.
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q13.** Someone suggests an analog switch that alternates the aux input between C3 and
C4 every 100 ms, so you get both sites. Give the reason this fails that has nothing to
do with the switching transients or the halved sample rate.
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q14.** Kappenman and Luck found that high electrode impedance harms data quality in a
frequency-dependent way rather than uniformly. Which end of the spectrum suffers most,
and why does that matter more for the drift you will see on a hand-placed C3 electrode
than for the mu band you are trying to measure?
*Source: Kappenman & Luck (2010). `reviewed: no`*

> **Answer:** _(unanswered)_

## Sources

This whole topic is optional (Phase 4B is gated behind Phase 4), so treat even Tier 1 as
"read before you spend money", not "read before you continue". The Krigolson paper is in
Tier 1 rather than Tier 2 for one reason: it tells you what the unmodified headband can
already do, which is the number any modification has to beat to have been worth it.

### Tier 1

* **BrainFlow, "Supported Boards"**,
  https://brainflow.readthedocs.io/en/stable/SupportedBoards.html
  The Muse section lists the `p20` / `p21` / `p50` / `p51` presets and the channel
  layout. Short, and the authoritative answer to what the library will give you.
* **Krigolson, O. E., Williams, C. C., Norton, A., Hassall, C. D., & Colino, F. L. (2017).
  "Choosing MUSE: Validation of a low-cost, portable EEG system for ERP research."**
  *Frontiers in Neuroscience*, 11, 109. What the stock Muse has been shown to measure
  reliably, and just as usefully, how much averaging they needed to get there. Open
  access, and it is the honest baseline for judging both the aux electrode and the
  project as a whole.

### Tier 2

* **Muse EEG Headset: Making Extra Electrode**, Hackaday,
  https://hackaday.io/project/162169-muse-eeg-headset-making-extra-electrode
  The DIY build, with parts list and photos. Written for the Muse 2016. Worth ten minutes
  even if you buy the finished electrode, because it shows you what the connection
  physically is and therefore what can go wrong with it.
* **Auxiliary Electrode, Micro USB single cup**, Interaxon,
  https://ca.choosemuse.com/products/auxiliary-electrode-single-cup
  The ready-made accessory, and the recommended purchase. Check the connector against
  your headband before ordering, since Interaxon also sells a USB-C version for newer
  models.
* **MNE-Python, "EEG sensor locations"**,
  https://mne.tools/stable/auto_tutorials/intro/40_sensor_locations.html
  Montages, layouts and the `standard_1020` positions. Open it when you need to know
  where C3 is in a coordinate system rather than on a diagram, which is the day you try
  to plot a five-channel montage.
* **Kappenman, E. S., & Luck, S. J. (2010). "The effects of electrode impedance on data
  quality and statistical significance in ERP recordings."** *Psychophysiology*, 47(5),
  888-904. The measured answer to "how good does electrode contact actually have to be",
  and the reason skin preparation at C3 is worth the trouble. Read it if the aux channel
  turns out noisy and you need to decide whether that is fixable.
