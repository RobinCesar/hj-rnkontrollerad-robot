# Syntax reference

Building blocks and API signatures, not finished solutions. Assemble them yourself.

Everything here was checked against the versions in `requirements.txt`. For the theory
behind each call, see [topics/](topics/). For when to use it, see
[LEARNING_PLAN.md](LEARNING_PLAN.md).

## Contents

* [BrainFlow](#brainflow)
* [MNE](#mne)
* [scikit-learn](#scikit-learn)
* [scipy.signal](#scipysignal)
* [pygame](#pygame)
* [numpy](#numpy-what-you-will-use-most)
* [Threading](#threading-phase-5)
* [Unit and shape conventions](#unit-and-shape-conventions)

## BrainFlow

Theory: [topics/04-data-acquisition.md](topics/04-data-acquisition.md)

```python
from brainflow.board_shim import BoardShim, BrainFlowInputParams, BoardIds
from brainflow.data_filter import DataFilter, FilterTypes, NoiseTypes, DetrendOperations

# --- Connection ---
params = BrainFlowInputParams()
params.mac_address = ""        # or params.serial_number = "MuseS-XXXX"
# params.serial_port = "/dev/ttyACM0"   # only if you use a BLED112 dongle

board_id = BoardIds.MUSE_2_BOARD    # 38. Others: MUSE_S_BOARD=39,
                                    # MUSE_2016_BOARD=41, MUSE_2_BLED_BOARD=22
board = BoardShim(board_id, params)

# --- Metadata (static methods, no connection needed) ---
BoardShim.get_sampling_rate(board_id)     # 256
BoardShim.get_eeg_channels(board_id)      # [1, 2, 3, 4]  -> row indices in the matrix
BoardShim.get_eeg_names(board_id)         # ['TP9', 'AF7', 'AF8', 'TP10']
BoardShim.get_timestamp_channel(board_id) # 6
BoardShim.get_marker_channel(board_id)    # 7
BoardShim.get_other_channels(board_id)    # [5]  -> the aux EEG input, see below

# --- Session ---
board.prepare_session()

# Optional: enable the 5th EEG value (the aux electrode on the micro-USB port).
# Must be called AFTER prepare_session and BEFORE start_stream.
#   p21 = 4 EEG + IMU (default)      p20 = 5 EEG + IMU
#   p51 = 4 EEG + IMU + PPG          p50 = 5 EEG + IMU + PPG
board.config_board("p20")                 # theory: topics/14-electrode-hardware.md
# The aux value then arrives in row 5, i.e. get_other_channels(), not get_eeg_channels().

board.start_stream(45000)                 # ring buffer size in samples
board.insert_marker(1.0)                  # label in the stream, inserted at the cue

data = board.get_current_board_data(512)  # last 512 samples, does NOT empty the buffer
data = board.get_board_data()             # everything since last call, EMPTIES buffer
# data.shape == (n_rows, n_samples)

board.stop_stream()
board.release_session()

BoardShim.enable_dev_board_logger()       # debugging connection problems
```

```python
# --- Filters (operate IN-PLACE on a 1D float64 array, one channel at a time) ---
DataFilter.perform_bandpass(chan, sr, start_freq, stop_freq, order, FilterTypes.BUTTERWORTH, 0)
DataFilter.perform_bandstop(chan, sr, 48.0, 52.0, 4, FilterTypes.BUTTERWORTH, 0)
DataFilter.remove_environmental_noise(chan, sr, NoiseTypes.FIFTY)
DataFilter.detrend(chan, DetrendOperations.LINEAR)

DataFilter.get_psd_welch(chan, nfft, overlap, sr, window)   # -> (amplitudes, freqs)

DataFilter.write_file(data, "data/raw/sess1.csv", "w")
data = DataFilter.read_file("data/raw/sess1.csv")
```

> In-place means the array is modified. To keep the original: `c = data[ch].copy()`.
> Requires `float64` and a contiguous array: `np.ascontiguousarray(x, dtype=np.float64)`.

## MNE

Theory: [topics/06-epoching.md](topics/06-epoching.md), [topics/07-csp.md](topics/07-csp.md)

```python
import mne

# --- Build a Raw object from your own numpy data ---
info = mne.create_info(ch_names=['TP9','AF7','AF8','TP10'], sfreq=256., ch_types='eeg')
raw = mne.io.RawArray(eeg_data, info)          # eeg_data: (n_channels, n_samples), in VOLTS
raw.set_montage('standard_1020')               # gives electrode positions -> topographies

# --- Filtering ---
raw.filter(l_freq=8., h_freq=30., method='iir')
raw.notch_filter(freqs=50.)
raw.resample(128.)

# --- Events and epoching ---
# events: (n_events, 3) -> [sample_index, 0, event_id]
event_id = {'left': 1, 'right': 2}
epochs = mne.Epochs(raw, events, event_id, tmin=-1.0, tmax=4.0,
                    baseline=(-1.0, 0.0), preload=True)

X = epochs.get_data()          # (n_trials, n_channels, n_samples)
y = epochs.events[:, -1]       # labels
epochs_cropped = epochs.copy().crop(tmin=0.5, tmax=3.5)

# --- Inspection ---
raw.plot(); epochs.plot(); epochs.average().plot_topomap()
epochs.compute_psd(fmin=1, fmax=40).plot()

# --- Public dataset (Phase 3) ---
from mne.datasets import eegbci
files = eegbci.load_data(subject=1, runs=[4, 8, 12])   # imagined left/right hand
raw = mne.io.concatenate_raws([mne.io.read_raw_edf(f, preload=True) for f in files])
eegbci.standardize(raw)
events, _ = mne.events_from_annotations(raw)

# --- CSP ---
from mne.decoding import CSP
csp = CSP(n_components=4, reg=None, log=True, norm_trace=False)
X_feat = csp.fit_transform(X, y)      # (n_trials, n_components)
csp.plot_patterns(epochs.info, ch_type='eeg')   # <- figure for the report
```

> MNE works in **volts**, BrainFlow gives you **microvolts**. Multiply by `1e-6` when
> going from BrainFlow to MNE, otherwise every plot looks absurd.

## scikit-learn

Theory: [topics/08-classification.md](topics/08-classification.md),
[topics/09-validation.md](topics/09-validation.md)

```python
from sklearn.pipeline import Pipeline
from sklearn.discriminant_analysis import LinearDiscriminantAnalysis as LDA
from sklearn.svm import SVC
from sklearn.preprocessing import StandardScaler
from sklearn.model_selection import cross_val_score, StratifiedKFold, permutation_test_score
from sklearn.metrics import accuracy_score, confusion_matrix, ConfusionMatrixDisplay

# A Pipeline holds everything that learns something from the data, in one object.
# This is how you avoid leakage: every step is refitted per fold.
pipe = Pipeline([
    ('csp', CSP(n_components=4, log=True)),
    ('clf', LDA(solver='lsqr', shrinkage='auto')),   # shrinkage helps when trials are few
])

cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
scores = cross_val_score(pipe, X, y, cv=cv, scoring='accuracy')
print(scores.mean(), scores.std())

# Permutation test: is the result better than chance?
score, perm_scores, pvalue = permutation_test_score(pipe, X, y, cv=cv, n_permutations=1000)

# Fit the final model and use it
pipe.fit(X, y)
proba = pipe.predict_proba(X_new)     # confidence, used in real time

# SVM for comparison (scale the features first)
pipe_svm = Pipeline([('csp', CSP(4, log=True)), ('sc', StandardScaler()),
                     ('clf', SVC(kernel='linear', C=1.0, probability=True))])
```

```python
import joblib
joblib.dump(pipe, 'models/muse_lr.joblib')
pipe = joblib.load('models/muse_lr.joblib')
```

## scipy.signal

Theory: [topics/05-digital-filtering.md](topics/05-digital-filtering.md)

Use this when you want full control over the filter, especially in real time.

```python
from scipy.signal import butter, sosfilt, sosfiltfilt, sosfilt_zi, iirnotch, welch, spectrogram

sos = butter(4, [8, 30], btype='bandpass', fs=256, output='sos')
x_off = sosfiltfilt(sos, x, axis=-1)     # zero-phase, OFFLINE only
x_on  = sosfilt(sos, x, axis=-1)         # causal, for real time

# Carry the filter state between blocks in real time, otherwise every block
# starts with a transient:
zi = sosfilt_zi(sos) * x[0]
y, zi = sosfilt(sos, block, zi=zi)

f, Pxx = welch(x, fs=256, nperseg=256)   # power spectrum
```

> Use `output='sos'` rather than `b, a`: it is numerically far more stable at higher
> filter orders.

## pygame

Theory: [topics/12-game-design.md](topics/12-game-design.md)

```python
import pygame

pygame.init()
screen = pygame.display.set_mode((800, 600))
pygame.display.set_caption("Brain-Controlled Game")
clock = pygame.time.Clock()

running = True
while running:
    dt = clock.tick(60) / 1000.0            # seconds since the last frame
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False
        elif event.type == pygame.KEYDOWN and event.key == pygame.K_LEFT:
            ...

    keys = pygame.key.get_pressed()         # continuous input
    if keys[pygame.K_LEFT]:
        ...

    screen.fill((20, 20, 30))
    pygame.draw.circle(screen, (200, 80, 80), (x, y), 20)
    pygame.draw.rect(screen, (80, 200, 120), pygame.Rect(x, y, w, h))
    font = pygame.font.SysFont(None, 32)
    screen.blit(font.render(f"Confidence: {p:.2f}", True, (255, 255, 255)), (10, 10))

    pygame.display.flip()

pygame.quit()
```

## numpy: what you will use most

```python
X.shape                       # (trials, channels, samples), ALWAYS check this
np.var(X, axis=-1)            # variance over time, per trial and channel -> CSP feature
np.log(np.var(X, axis=-1))
X.mean(axis=2, keepdims=True) # baseline per channel, keeps the dimension

np.stack(list_of_epochs)      # -> (n_trials, ...)
X[:, [0, 3], :]               # pick channels 0 and 3
np.abs(X).max(axis=(1, 2)) > 100      # artifact mask per trial
X_clean = X[~mask]

np.ascontiguousarray(x, dtype=np.float64)   # required by BrainFlow's filters
np.searchsorted(timestamps, cue_time)       # find the sample index for an event
```

## Threading (Phase 5)

Theory: [topics/11-realtime.md](topics/11-realtime.md)

```python
import threading, queue

q = queue.Queue()
stop = threading.Event()

def acquire():
    while not stop.is_set():
        q.put(board.get_board_data())

t = threading.Thread(target=acquire, daemon=True)
t.start()
...
stop.set(); t.join()
```

## Unit and shape conventions

Pin these down once and every bug gets easier to find.

| Thing | Convention |
|---|---|
| BrainFlow output | microvolts, shape `(n_rows, n_samples)` |
| MNE input | volts, shape `(n_channels, n_samples)` |
| Epoched data | `(n_trials, n_channels, n_samples)` |
| Feature matrix `X` for sklearn | `(n_samples, n_features)`, i.e. one row per trial |
| Labels `y` | `(n_trials,)`, integers |
| CSP input | `(n_trials, n_channels, n_samples)`, bandpass filtered first |
| Time axis | always last (`axis=-1`) |
