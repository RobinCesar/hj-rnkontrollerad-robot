# 04. Data acquisition

**Needed in:** Phase 1
**In one sentence:** Getting samples off the headset is the easy part; knowing exactly
*when* each sample happened, relative to what you showed on screen, is the part that
quietly ruins projects.

## Why this matters here

Your labels come from timing. A trial is "the 3 seconds after the cue appeared", and if
your notion of when the cue appeared is off by 300 ms, every epoch is misaligned, ERD
falls outside your analysis window, and no amount of clever classification will recover
it. Timing errors do not announce themselves; they just look like the effect not being
there.

## The acquisition chain

```
scalp -> electrode -> amplifier -> ADC -> Bluetooth -> OS BLE stack -> BrainFlow buffer -> your code
```

Each stage adds delay, and only some of it is constant. The Bluetooth and OS stages in
particular introduce **variable** latency (jitter), typically tens of milliseconds,
occasionally much worse if packets are retransmitted.

## Sampling

* The Muse samples at **256 Hz**, so one sample every 3.9 ms.
* **Nyquist:** you can only represent frequencies up to half the sampling rate, 128 Hz.
  Anything above that gets **aliased**, folded down and disguised as a lower frequency.
  The hardware applies an anti-aliasing filter before digitising, so you do not have to
  worry about this at acquisition time, but you must worry about it if you
  **downsample** later. See [05](05-digital-filtering.md).
* 256 Hz is comfortably enough for an 8 to 30 Hz analysis.

## Buffering: the concept that matters most

BrainFlow runs a background thread that continuously writes incoming samples into a
fixed-size **ring buffer**. Your code reads from it. Two consequences:

1. **You must read often enough.** If the buffer holds 45000 samples at 256 Hz, that is
   about 175 seconds of data. Read less often than that and old samples are silently
   overwritten. You get no error; you just lose data.
2. **There are two different read semantics**, and confusing them is a classic bug:

| Call | Returns | Buffer after |
|---|---|---|
| `get_board_data()` | everything accumulated since the last call | emptied |
| `get_current_board_data(n)` | the most recent `n` samples | unchanged |

Use `get_board_data()` when **recording** (you want every sample, exactly once).
Use `get_current_board_data(n)` when **classifying in real time** (you want the latest
window, and overlapping windows are the whole point). Using the emptying version in a
real-time loop gives you variable-length, non-overlapping chunks, which is not what
your sliding window needs.

## The data matrix

BrainFlow returns a single 2D array of shape `(n_rows, n_samples)` where different rows
mean different things. **The row layout depends on the board.** Never hardcode indices;
always ask:

```python
BoardShim.get_eeg_channels(board_id)       # [1, 2, 3, 4] for Muse 2
BoardShim.get_timestamp_channel(board_id)  # 6
BoardShim.get_marker_channel(board_id)     # 7
```

Note that time runs along the **second** axis. This matches the convention used
everywhere else in the project (`axis=-1` is always time).

## Markers: how labels get into the data

`board.insert_marker(value)` writes a number into the marker row at the current
position in the stream. Everywhere else that row is zero. Afterwards, finding your
events is just finding the non-zero entries:

```python
marker_row = data[BoardShim.get_marker_channel(board_id)]
event_samples = np.flatnonzero(marker_row)
event_values  = marker_row[event_samples]
```

Use distinct values per class (e.g. 1 for left, 2 for right), and consider extra values
for session start, rest blocks, and rejected trials. It costs nothing now and saves you
from re-recording later.

**The marker is inserted when your code calls it, not when the subject saw the cue.**
The gap between those two moments is your timing error. Which brings us to:

## Timing and synchronisation

Sources of delay between "the subject perceived the cue" and "the marker lands in the
data":

| Source | Typical size | Constant? |
|---|---|---|
| Bluetooth transmission and OS BLE stack | 20 to 100 ms | No, jittery |
| Display latency (screen buffer, vsync) | 8 to 30 ms | Roughly |
| Your code's scheduling | 0 to 20 ms | No |
| Subject reaction/attention | 100 to 300 ms | No |

For motor imagery you are analysing a multi-second window, so **you do not need
millisecond precision**. An error of 50 ms is harmless. An error of 500 ms is not,
because it can push your analysis window past the ERD onset. The goal is to know the
delay to within about 100 ms, and to know that it is stable.

**How to measure it empirically.** You need one event that is visible in both the EEG
and in your program's timeline. The cheapest method:

1. Start recording.
2. Have the subject blink hard, deliberately, at the exact moment your program inserts
   a marker (e.g. on a keypress that both inserts the marker and is the cue to blink).
3. Repeat 20 times.
4. In the recorded data, find the blink artifact (huge, obvious, frontal) and measure
   the offset between the blink peak and the marker sample.

The mean offset is your constant delay, and you can compensate for it. The standard
deviation is your jitter, and you cannot compensate for it, so report it. This is a
genuinely good experiment to include in the report; most student BCI projects skip it.

**Alternative:** BrainFlow provides a timestamp row. Compare `board.get_board_data()`
timestamps against `time.time()` in your cue code. This catches gross errors but not
the transmission delay itself, because the timestamp is applied on arrival.

**Why an arrival timestamp cannot fix this, in one paragraph.** The number you want is
when the sample was *taken at the scalp*. The number you get is when the packet reached
your process, after the amplifier buffered it, the headset's radio waited for its next
BLE connection interval, the OS Bluetooth stack queued it, and Python got scheduled. Each
of those adds delay, and only the first is constant. Worse, samples arrive in **chunks**:
a packet carrying twelve samples gets one arrival time, so eleven of the twelve
timestamps are reconstructed by assuming a perfectly regular sample interval, which the
headset's clock does not exactly honour. So an arrival timestamp gives you a good
estimate of the *average* delay and tells you nothing about the jitter around it.

This is the problem that Lab Streaming Layer exists to solve, by timestamping at the
source and continuously estimating the offset between the two machines' clocks. You do
not need it here, because motor imagery is analysed over multi-second windows and 50 ms
of jitter is irrelevant at that scale. But know the name and know why the problem is
real, because if you ever move to an ERP paradigm the same shortcut becomes fatal.

## Practical Muse notes

* **Board ID depends on connection method.** Native Bluetooth LE and the BLED112 USB
  dongle are *different* board IDs for the same physical headset
  (`MUSE_2_BOARD` = 38 vs `MUSE_2_BLED_BOARD` = 22). Getting this wrong gives a
  connection failure that looks like a hardware problem.
* **Discard the first few seconds.** Dry electrodes need time to settle, and the
  initial samples are usually garbage.
* **Check signal quality before every session**, not after. Standard deviation per
  channel is a good enough proxy: a channel that is flat or wildly variable is not
  making contact. Hair under TP9/TP10 is the usual cause.
* **`BoardShim.enable_dev_board_logger()`** turns on verbose logging. Use it the first
  time you try to connect.
* Always call `release_session()`, including on error. An unreleased session can leave
  the headset unable to reconnect until it is power-cycled. A `try/finally` block is
  the right structure here.

## Storage

Decide the format in Phase 1 and never change it.

* **Always keep the raw recording**, unfiltered and unprocessed, in `data/raw/`. Disk
  is free; re-recording a session is not. Every processing decision you make later,
  you will want to revisit.
* **Save metadata with the data**: date, subject, board ID, sampling rate, channel
  names, protocol parameters, and any notes ("electrode TP10 was loose"). A small JSON
  file next to each recording is enough. Six months later, an unlabelled `.csv` is
  worthless.
* `DataFilter.write_file()` and `read_file()` handle CSV round-tripping. It is
  inefficient but human-inspectable, which is worth a lot early on. Alternatives are
  `.npy` (fast, numpy-native) and MNE's `.fif` (stores channel metadata alongside the
  data, which is why it is worth graduating to).

## Common mistakes

* Hardcoding row indices instead of querying them.
* Using `get_board_data()` in a real-time loop.
* Reading from the buffer too rarely and losing data with no warning.
* Never measuring the actual cue-to-marker delay, then wondering why ERD is absent.
* Storing only processed data and losing the ability to reprocess.
* Forgetting `release_session()`.

## Questions

Write your answers in the boxes. See
[the convention](README.md#answering-the-questions).

**Q1.** Your ring buffer is 45000 samples and you sample at 256 Hz. How long can you go
between reads?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q2.** Why is `get_current_board_data()` the right call for the real-time loop?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q3.** How would you find the sample index of the third "left" cue in a recording?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q4.** Design a procedure to measure your cue-to-marker latency to within 50 ms.
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q5.** Why would a 400 ms timing error be much worse than a 40 ms one, given a 3 s
analysis window?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q6.** Of the delay sources listed in the timing table, which can you compensate for
after the fact and which can you not? What do you do with the ones you cannot?
*Source: this document. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q7.** What do `prepare_session()`, `start_stream()` and `release_session()` each do,
and in what order must they be called? What are the two arguments to `start_stream()`
for?
*Source: BrainFlow, User API. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q8.** `BoardShim` has a family of static query methods. Name four of them besides
`get_eeg_channels`, and say what each returns. Why are they static rather than instance
methods?
*Source: BrainFlow, User API. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q9.** Find the Muse 2 entry on the Supported Boards page. What is the board ID for
native BLE, what is required to make native BLE work on Linux, and what does the page
say you must do to get PPG or accelerometer data?
*Source: BrainFlow, "Supported Boards". `reviewed: no`*

> **Answer:** _(unanswered)_

**Q10.** In BrainFlow's code samples, which calls write data to a file and read it back,
and what does the written file actually contain compared to the array you had in memory?
*Source: BrainFlow, Code Samples. `reviewed: no`*

> **Answer:** _(unanswered)_

**Q11.** LSL exists to solve the timing problem properly. What does it do that a
timestamp applied when the packet arrives at your computer cannot do, and what would you
need in your setup to benefit from it?
*Source: Lab Streaming Layer documentation. `reviewed: no`*

> **Answer:** _(unanswered)_

## Sources

This is the most purely practical topic in the folder, and it shows: **Tier 1** is one
documentation site, and there are no papers worth your time. Everything here is learned
by connecting to the headset and printing array shapes.

### Tier 1

* **BrainFlow, User API**, https://brainflow.readthedocs.io/en/stable/UserAPI.html
  The authoritative reference for every call. Read the `BoardShim` and `DataFilter`
  sections properly, once, rather than looking up calls one at a time forever.
* **BrainFlow, Supported Boards**,
  https://brainflow.readthedocs.io/en/stable/SupportedBoards.html
  Board IDs, channel layouts, sampling rates, and the connection requirements per
  device. Find the Muse section and read it before your first connection attempt; it
  will save you an evening of debugging a "hardware fault" that is a wrong board ID.
* **BrainFlow, Code Samples**, https://brainflow.readthedocs.io/en/stable/Examples.html
  Short, complete, runnable examples for streaming and filtering. Start from these
  rather than from a blank file.

### Tier 2

* **Lab Streaming Layer (LSL)**, https://labstreaminglayer.readthedocs.io/
  The standard system for synchronising multiple data streams with a shared clock, and
  the proper solution to the timing problem described above. Genuinely overkill for this
  project, so this is here for one reason: read enough of it to be able to say in the
  report what you did *not* do about timing and why that was acceptable for a
  multi-second paradigm. BrainFlow can output to LSL if you ever need it.
