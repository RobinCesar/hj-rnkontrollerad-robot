# 12. Game design for BCI

**Needed in:** Phase 6
**In one sentence:** Designing for an input that is slow, binary, roughly 70 % accurate
and occasionally unavailable is a genuine design constraint, and the games that work are
the ones built around those properties rather than in spite of them.

## Why this matters here

The instinct is to take a game you like and bolt EEG onto it. That produces a
frustrating experience, because almost every existing game assumes an input that is
instant, precise and reliable. Your input is none of those. Designing for what the
signal actually is turns the limitation into the interesting part.

## What your input actually is

Be honest about the specification before designing anything:

| Property | Reality |
|---|---|
| Dimensionality | 1 binary axis (left/right), possibly with a neutral state |
| Update rate | ~4 decisions per second, heavily correlated |
| Latency | 1.5 to 3 s from intention to response ([11](11-realtime.md)) |
| Accuracy | 60 to 75 % if things go well |
| Availability | Degrades with fatigue; needs recalibration between sessions |
| Effort | High. Sustained motor imagery is genuinely tiring |

Compare that with a keyboard: 2+ axes, instant, 100 % accurate, effortless. This is a
fundamentally different input device, and it needs a different game.

## Design principles

### 1. Never require speed

Anything reflex-based is impossible. The player's intention takes seconds to register.
Design for deliberate, planned actions with generous time windows.

### 2. Make errors recoverable

At 70 % accuracy roughly one action in three is wrong. If a single mistake ends the run,
the game is unplayable and the player learns nothing about their own control.

* Prefer gradual movement over discrete commitment. A ball nudged left over several
  hundred milliseconds is forgiving; an irreversible branch choice is not.
* Avoid instant-death mechanics entirely.
* Let the player correct: if the character drifts the wrong way, sustained imagery
  should bring it back.

### 3. Movement should be continuous, not triggered

Map the classifier's smoothed **probability** to a continuous quantity (velocity,
acceleration, tilt) rather than firing a discrete action on threshold crossing.
Continuous mapping means a marginal 0.55 probability produces a small nudge rather than
a full command, so uncertainty degrades gracefully instead of causing a hard error.

### 4. Show the player what the system sees

Non-negotiable for a BCI. Display the smoothed confidence as a bar or a colour. Without
it the player has no idea whether a failure was their imagery or the classifier, and
they cannot learn to control the system.

This is not just a debug feature. BCI control is a **skill the user learns**, and they
can only learn it with feedback. Vidaurre et al. (2011) describe this coadaptation: the
classifier adapts to the user and the user adapts to the classifier. Hiding the
confidence prevents the second half.

### 5. Support an explicit neutral state

"I am not trying to do anything" is a legitimate and important input. Without it the
game is always moving, which is exhausting and makes rest impossible. Implement it as a
confidence threshold: below it, nothing happens.

### 6. Build in calibration

The session must begin with a calibration phase that records training data (essentially
the Phase 4 protocol, embedded in the game). Make it as short as tolerable, and consider
framing it as a tutorial so it does not feel like homework.

## Game candidates

Reasonable choices for this input, in roughly increasing complexity:

| Game | Why it works | Watch out for |
|---|---|---|
| **1D cursor-to-target** | The classic BCI task. Cursor on a line, target at one end, steer to it. Directly comparable to published BCI results | Barely a game. Best as your debug view and baseline metric |
| **Catch falling objects** | Continuous horizontal control, forgiving, difficulty tunable by fall speed | Keep fall speed slow enough that latency is not fatal |
| **Auto-forward maze / runner** | Forward motion is automatic, player only steers left/right. Natural fit for one binary axis | Junction spacing must exceed your total latency |
| **Slow Pong** | Familiar, self-paced, errors cost a point rather than the run | Ball speed must be well below reaction requirements |
| **Balance / tilt** | Keep a beam level against drift. Uses sustained imagery naturally | Can be tiring, since it never lets the player rest |

The auto-forward maze is probably the best fit: the game handles everything except the
one decision your BCI can make, difficulty scales cleanly by adjusting junction
spacing, and it is visually satisfying.

## Architecture: keep the game ignorant of EEG

The most important structural decision, and it is worth stating explicitly because it
determines how debuggable the project is.

Define one interface, for example a function returning a value in [-1, 1] plus a
confidence, and provide two implementations:

* A **keyboard** source
* A **classifier** source

The game only knows about the interface. It never imports BrainFlow or sklearn.

Benefits:

* **Build and test the game without the headset on.** Enormous practical win, since
  putting on the Muse, checking impedance and calibrating takes ten minutes each time.
* When something misbehaves you can immediately determine whether it is the game or the
  BCI, by switching sources.
* You can add a **replay** source that feeds recorded data through the pipeline, giving
  you a fully deterministic, repeatable test of the entire live chain with no hardware.
  This is the single most useful debugging tool you can build in Phase 6.

**Build the game with keyboard control first and make it fun.** If it is not fun with a
perfect input, it will not be fun with a bad one. This also cleanly separates "learning
pygame" from "debugging real-time EEG", which are two hard things that should not be
attempted simultaneously.

## Measuring whether it works

Beyond classification accuracy, the game gives you behavioural metrics worth reporting:

* **Time to target**, i.e. how long to reach a goal
* **Path efficiency**, actual path length divided by optimal
* **False activations per minute during intentional rest**
* **Information transfer rate** (bits/minute), the standard BCI metric that combines
  accuracy and speed and makes your system comparable to published work
* **Learning curve across sessions**: does the player improve? This is a genuinely
  interesting result, because improvement means the user is learning to modulate their
  own mu rhythm

That last one is the most compelling thing this project could show. It is also cheap to
collect: log every session.

## pygame specifics

* **Game loop:** poll events, update state, draw, flip. Every frame.
* **`clock.tick(60)`** caps the frame rate and returns elapsed milliseconds. Use that
  delta to make movement frame-rate independent (`x += velocity * dt`) rather than
  adding a fixed amount per frame.
* **Do not block the game loop.** Classification runs in another thread
  ([11](11-realtime.md)); the loop just reads the latest decision. If the game stutters
  when a classification runs, you have them on the same thread.
* **`pygame.event.get()` must be called every frame**, even if you ignore the events, or
  the OS will consider the window unresponsive.
* pygame is simple enough that you can learn what you need in an afternoon. Do not
  over-engineer the game; the interesting part of this project is upstream of it.

## Common mistakes

* Designing a game that needs fast or precise input.
* Punishing single errors harshly.
* Triggering discrete actions on threshold crossings instead of continuous mapping.
* Hiding the classifier's confidence from the player.
* No neutral state, so the player can never rest.
* Coupling the game directly to BrainFlow, so it cannot run or be tested without hardware.
* Building the game and the real-time pipeline at the same time, so you cannot tell
  which one is broken.
* Spending more time on graphics than on the BCI, which is the part that is actually
  hard and actually interesting to an employer.

## Questions

Write your answers in the boxes. See
[the convention](README.md#answering-the-questions).

**Q1.** Given 2.5 s latency and 70 % accuracy, why is a reflex-based game impossible?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q2.** Why map probability to velocity rather than triggering a move at threshold?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q3.** Why must the game be playable with a keyboard?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q4.** What is a replay input source and why is it the most useful thing to build in
Phase 6?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q5.** Why does showing confidence to the player affect their actual performance, not
just their understanding?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q6.** Which metric would let you compare your game against published BCI systems?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q7.** Why does the auto-forward maze need a minimum junction spacing, and how would
you compute that spacing from your measured latency?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q8.** What are the four steps of a minimal pygame program's main loop? What does
`pygame.display.flip()` do, and when would you use `update()` instead?
*Source: pygame documentation and the introductory tutorial. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q9.** What does `clock.tick(60)` return, and how do you use that return value to make
movement independent of frame rate? Write the one-line update expression.
*Source: pygame documentation. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q10.** What does a `pygame.sprite.Group` do for you, and what does a sprite's
`update()` method conventionally contain? Why is `Group.draw()` easier than blitting
each object yourself?
*Source: pygame, "Sprite module introduction". `reviewed: no`*

> **Answer:** _(unanswered)_

**Q11.** Find a consumer-EEG game project through NeuroTechX or the OpenBCI community.
What control signal did it actually use: motor imagery, alpha power, blinks, or jaw
clenches? What does the answer suggest about which signals are genuinely reliable on
four channels?
*Source: NeuroTechX, OpenBCI community. `reviewed: no`*

> **Answer:** _(unanswered)_

## Sources

### Start here

* **pygame documentation**, https://www.pygame.org/docs/
  and the introductory tutorial at https://www.pygame.org/docs/tut/PygameIntro.html
  Enough to build everything you need here.
* **pygame, "Sprite module introduction"**, https://www.pygame.org/docs/tut/SpriteIntro.html
  Useful once you have more than a couple of moving objects.

### Go deeper

* **NeuroTechX**, https://neurotechx.com/
  Community projects, many of them consumer-EEG games. Useful for seeing what people
  actually manage with 4-channel headsets, and for calibrating ambition.
* **OpenBCI community**, https://openbci.com/community/
  Similar, with more hardware-oriented discussion.

### Papers

* Vidaurre, C., Sannelli, C., Müller, K.-R., & Blankertz, B. (2011).
  "Machine-learning-based coadaptive calibration for brain-computer interfaces."
  *Neural Computation*, 23(3), 791-816. On the user and the classifier adapting to each
  other, which is the theoretical basis for showing feedback.
* Lotte, F., Larrue, F., & Mühl, C. (2013). "Flaws in current human training protocols
  for spontaneous brain-computer interfaces: lessons learned from instructional design."
  *Frontiers in Human Neuroscience*, 7, 568. Argues that standard BCI training and
  feedback design is poor, and applies instructional-design principles to fix it.
  Directly relevant to designing your feedback and calibration phases.
* Wolpaw, J. R., Birbaumer, N., McFarland, D. J., Pfurtscheller, G., & Vaughan, T. M.
  (2002). "Brain-computer interfaces for communication and control." *Clinical
  Neurophysiology*, 113(6), 767-791. The field-defining review, and the standard source
  for the information transfer rate definition.
* Kübler, A., Holz, E. M., Riccio, A., et al. (2014). "The user-centered design as novel
  perspective for evaluating the usability of BCI-controlled applications." *PLOS ONE*,
  9(12), e112392. On evaluating a BCI application by usability rather than accuracy
  alone.

### Video

* **Controllerless Basketball Game using EEG-Based Brain Computer Interface**,
  https://www.youtube.com/watch?v=NmX_fQkcEic
  A finished BCI game, which is the useful part: it shows how a slow and unreliable
  control signal has to shape the game design rather than the other way round.
