# 10. Experimental design

**Needed in:** Phase 4
**In one sentence:** The quality of your recording protocol sets a hard ceiling on
everything that follows, and unlike your code, a badly designed session cannot be fixed
afterwards.

## Why this matters here

You can rewrite a filter in ten minutes. You cannot re-run a session in which the cue
arrow told the subject which way to look, or in which fatigue made the last third of
the trials useless. Design the protocol on paper, write it into
[REPORT.md](../REPORT.md) section C.2, and only then record.

## The Graz paradigm

The standard motor imagery protocol, from the Graz BCI group. One trial:

```
  t = -2s     fixation cross appears, subject relaxes and fixates
  t =  0s     cue (arrow left or right) appears for ~1.25 s
  t =  0.5s   subject begins kinaesthetic imagery
  t =  4s     cue period ends, imagery stops
  t =  4-8s   rest, variable duration
```

Then repeat, 40 to 60 times per class, split into blocks with breaks.

Every element of this is there for a reason:

* **The fixation cross** gives a defined baseline period and stops the eyes wandering.
  Without it, your baseline is contaminated and eye position varies between trials.
* **The cue is brief.** It tells the subject what to do without remaining on screen as
  a continuously varying visual stimulus that differs between classes.
* **Imagery starts after the cue, not at it.** This is why the analysis window in
  [06](06-epoching.md) typically starts around 0.5 s: before that, you are recording the
  visual response to the arrow and the subject's reaction time, not motor imagery.
* **The rest period is variable in length.** If rest is always exactly 4 s, the subject
  learns the rhythm, starts anticipating the next cue, and produces preparatory motor
  activity *before* the cue. That anticipation contaminates your baseline.
* **Rest must be long enough** for the beta rebound ([02](02-motor-imagery-erd.md)) to
  finish. Two seconds is not enough; the rebound is still going. Three to five seconds
  is safer.

## The cue itself is a confound

This deserves its own section because it is the design flaw most likely to invalidate
your result.

If your left cue is an arrow on the left of the screen and your right cue is an arrow
on the right, then:

* The subject's eyes move left for one class and right for the other.
* Horizontal eye movement is a large, lateralised, low-frequency signal that AF7 and
  AF8 pick up beautifully ([03](03-artifacts.md)).
* Your classifier now has a trivially easy, completely non-neural feature to learn.

You would get a good accuracy and it would mean nothing.

**Mitigations, in order of preference:**

1. **Present the cue at fixation.** An arrow drawn at the centre of the screen, small,
   with the subject fixating on it. No eye movement required to perceive it.
2. **Use auditory cues.** A spoken "left"/"right" or two distinct tones. Removes the
   visual confound entirely. The best option if you can do it.
3. **Keep the cue symmetric.** Same visual energy, same position, differing only in a
   minimal feature.
4. **Analyse from after the cue disappears**, so the visual response is outside your
   window.

Whatever you choose, **state it in the method section and justify it**. Noticing this
confound is exactly the kind of thing that shows you understand experimental design.

## Trial counts

| Trials per class | Verdict |
|---|---|
| Under 20 | Unusable. Chance can produce 65 % accuracy |
| 20 to 30 | Weak. Confidence intervals will swamp any effect |
| **40 to 60** | **Workable minimum for this project** |
| 80 to 120 | Comfortable, standard in published studies |

Balance the classes and randomise the order (with a constraint against long runs of the
same class, which lets the subject relax and drift).

More trials means a longer session, which means fatigue. This is a real trade-off, not
a free parameter. Which brings us to:

## Fatigue and session structure

Motor imagery is genuinely tiring. Concentration degrades, mu rhythm changes, and the
last block of a long session is often noticeably worse than the first.

* **Break the session into blocks** of 20 to 30 trials with a minute or two of rest.
* **Keep total recording under about 40 minutes**, including breaks.
* **Note in your log which block is which**, so you can test whether accuracy degrades
  over the session. That is a genuine result worth reporting, and it costs nothing to
  collect.

## Instructing the subject

Underrated. The instruction is part of your method, and it should be written down and
delivered identically every session.

* Specify **kinaesthetic** imagery: "feel the sensation of squeezing your hand, the
  tension in the muscles, without seeing it and without actually moving".
* Specify a **consistent** movement. "Imagine your hand" is too vague; the subject will
  do something different each trial. "Imagine repeatedly squeezing a ball" is concrete
  and repeatable.
* Say explicitly **not to move**, not to tense the jaw, and not to hold their breath.
  People do all three unconsciously when concentrating.
* Designate **when to blink**, i.e. during rest, not during imagery.

## Sessions and the recalibration problem

EEG features drift. Between two sessions the headset sits slightly differently,
impedance changes, and the subject's state differs. A classifier trained on Monday
often works noticeably worse on Tuesday.

This matters directly for your game: nobody wants to record 80 calibration trials each
time they play.

**Record at least two sessions on different days** and report:

* Within-session cross-validation accuracy (the optimistic number)
* Cross-session accuracy, training on day 1 and testing on day 2 (the realistic number)

The gap between them is one of the more interesting results you can produce, and it is
frequently omitted from student projects. It also motivates the practical design of the
game's calibration phase.

## Ethics and safety

Low stakes here, since you are recording yourself with a consumer device, but worth a
paragraph in the report:

* Consumer EEG is non-invasive and passive. No current is applied. There is no
  meaningful physical risk beyond skin irritation from prolonged wear.
* If you ever record another person, get informed consent, explain what the data is,
  and let them stop at any time.
* EEG data is personal data. Do not commit recordings to a public GitHub repository
  without thinking about it, which is a good reason to keep `data/` out of version
  control.

## Common mistakes

* Designing the protocol while writing the recording script, rather than beforehand.
* Lateralised cues that induce lateralised eye movements.
* Fixed-duration rest periods that let the subject anticipate.
* Rest too short, so the beta rebound bleeds into the next trial's baseline.
* Too few trials to say anything statistically.
* No breaks, so the second half of the data is fatigue-contaminated.
* Vague instructions that change between sessions.
* Only ever recording one session, so cross-session performance is unknown.
* Not writing down what happened during the session. "TP10 was loose for the first
  block" is invaluable three weeks later and impossible to reconstruct.

## Questions

Write your answers in the boxes. See
[the convention](README.md#answering-the-questions).

**Q1.** Why does imagery start 0.5 s after the cue rather than at it?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q2.** Why must the rest period vary in length?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q3.** Your cue is an arrow at the left or right edge of the screen. What is the
confound, and what are two fixes?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q4.** Why is 20 trials per class not enough? (See [09](09-validation.md).)
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q5.** Why record two sessions on different days rather than one long session?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q6.** What should you write down during a recording session that is not in the data
file?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q7.** Why must the rest period be at least three seconds rather than two? Name the
specific physiological effect that sets the lower bound.
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q8.** Open the BCI Competition IV dataset 2a description and write out the exact Graz
trial timing it specifies: when the fixation cross appears, when the cue appears and for
how long, when imagery ends, and how long the break is. How many trials per class per
session, and how many sessions per subject?
*Source: BCI Competition IV. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q9.** Dataset 2a recorded 22 EEG channels and 3 additional EOG channels. Why did they
bother recording EOG separately, and what does that tell you about a limitation of your
own protocol?
*Source: BCI Competition IV. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q10.** The PhysioNet EEG Motor Movement/Imagery protocol alternates task and rest
differently from the Graz paradigm. Describe its run structure, and name one thing it
does that you would *not* want to copy for a left/right classification study.
*Source: PhysioNet, EEG Motor Movement/Imagery Dataset. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q11.** Shenoy et al. quantify how much a classifier degrades between the session it was
calibrated on and a later one. Roughly how large is the drop they report, what do they
identify as the cause, and what does that predict for the calibration phase of your game?
*Source: Shenoy et al. (2006). `reviewed: no`*

> **Answer:** _(unanswered)_

## Sources

The point of this topic is to copy a protocol that already works rather than invent one,
so **Tier 1** is two dataset descriptions rather than anything theoretical. Read them
with a pen, and write your own protocol into [REPORT.md](../REPORT.md) section C.2 as you
go.

### Tier 1

* **BCI Competition IV**, https://www.bbci.de/competition/iv/
  Dataset 2a is the standard motor imagery dataset. The description PDF contains a clear
  diagram of the Graz paradigm timing, which is the canonical reference for your Phase 4
  protocol. This is the single most directly copyable thing in the folder.
* **PhysioNet, EEG Motor Movement/Imagery Dataset**,
  https://physionet.org/content/eegmmidb/1.0.0/
  A second complete protocol, structured differently: cue presentation, timings, run
  structure, task definitions. Reading it next to the Graz one shows you which parts of a
  protocol are conventions and which parts are load-bearing.

### Tier 2

* **Shenoy, P., Krauledat, M., Blankertz, B., Rao, R. P., & Müller, K.-R. (2006).
  "Towards adaptive classification for BCI."** *Journal of Neural Engineering*, 3(1),
  R13-R23. Session-to-session non-stationarity, which is the problem behind your
  cross-session experiment and behind the game needing to recalibrate. Read it once you
  have two sessions recorded and a gap between them to explain.
