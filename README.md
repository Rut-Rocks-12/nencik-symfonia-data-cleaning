# Predicting single-trial reaction time from EEG (MSIT task)

Course project for **MSB1015 Scientific Programming**, Maastricht University.
Author: Rutvik Karupothula

A reproducible Python / Jupyter pipeline that turns raw 128-channel EEG into single-trial features
(P300 amplitude and frontal theta power) and links each trial to its reaction time (RT).

## Research question

> Can single-trial EEG/ERP features (P300 amplitude, frontal theta power) predict the reaction time of that trial?

The Multi-Source Interference Task (MSIT) creates conflict between what is seen and the response required.
Harder conditions give slower reactions, and both the P300 and frontal theta are known markers of
stimulus evaluation and cognitive control, so they are plausible predictors of RT.

## Dataset

**Nencki-Symfonia EEG/ERP dataset** ([OpenNeuro ds004621](https://openneuro.org/datasets/ds004621), NEMAR `on004621`).

- 42 participants, 128 EEG channels, recorded at 1000 Hz (BrainVision format).
- This project uses the **MSIT task** only: four conditions, about 390 trials per participant.
  - `00` no conflict, `S0` Simon conflict, `F0` flanker conflict, `FS` both conflicts.
  - In the raw event files the conditions appear as trigger codes (`S  7` = 00, `S  8` = S0, `S  5` = F0, `S  6` = FS),
    taken from the dataset's event description (`task-msit_events.json`).
- Reaction time = time from the stimulus to the first button press, within 2 s and before the next stimulus.

Mean RT per condition (all trials, from the exploratory analysis):

| Condition | Trials | Mean RT (ms) | SD (ms) |
|---|---|---|---|
| 00 | 4118 | 536 | 120 |
| S0 | 4275 | 590 | 125 |
| F0 | 3777 | 672 | 146 |
| FS | 3943 | 741 | 175 |

RT rises as conflict is added, which is the expected interference effect.
<!-- TODO: re-run section 4 with all 42 subjects (the RT table currently excludes sub-29) and update this table -->

## Pipeline

All steps are in `SP_Project_cleaned.ipynb`. Each step saves its result per subject, so a re-run skips finished subjects.

| # | Step | Parameters | Why |
|---|---|---|---|
| 1 | Download | NEMAR, MSIT only | The full dataset is far larger than the task needs |
| 2 | Band-pass filter | 0.1-30 Hz, FIR (firwin), zero-phase | Removes slow drift and high-frequency noise, keeps theta and P300 |
| 3 | Crop dead time | first to last task event, 1 s buffer | Filter first on the whole recording, so filter edge effects fall in the removed part |
| 4 | Downsample | 1000 to 250 Hz | 4x less data. Safe because the signal was low-passed at 30 Hz before (checked: r = 0.998 at Pz for sub-01) |
| 5 | ICA blink removal | 20 components, fitted on a 1 Hz high-passed copy (cleaning is applied to the 250 Hz, 0.1-30 Hz data), blink found automatically using Fp1 as EOG proxy | The dataset has no EOG channel. Blinks are the largest artifact. Slow drift stops ICA from converging, hence the 1 Hz copy |
| 6 | Re-reference | average reference, FCz re-created | FCz was the online reference and is not recorded |
| 7 | Epoching + RT matching | -0.8 to +1.2 s around each stimulus, baseline -0.2 to 0 s, reject epochs above 150 uV peak-to-peak | Each epoch is one trial. Stimuli are at least 1.73 s apart, so epochs do not overlap |
| 8 | Wavelet transform | Morlet, 3-30 Hz, dB versus -0.5 to -0.2 s | Shows when theta changes, which an ERP cannot |
| 9 | Channel selection | top 5 channels by group-mean P300 and theta | RT is never used, so the choice is not circular |
| 10 | Feature extraction | early window P300 0.3-0.4 s, theta 0.2-0.4 s (also literature windows for comparison) | See the window choice below |

### The window choice

Literature P300 windows are about 0.3-0.6 s. In this data the median RT is about 604 ms, so in about half of the
trials the response is already made before 0.6 s and the feature would partly measure the movement.
The main features therefore use an **early window** that ends before the response in 97.6% of trials.
The literature windows are kept as a comparison.

## Results so far

- **41 of 42 subjects** had usable epochs for channel selection (sub-16 had no usable epochs and was skipped).
- **P300 channels (parieto-occipital):** PPO1h, PPO2h, P2, Pz, PO3 (about 4.2-4.7 uV).
- **Theta channels (frontocentral):** FCz, FCC1h, C1, Cz, FCC2h (about +1.1 to +1.2 dB versus baseline).
- Single-trial features are saved in `data/features/msit_features.csv` (one row per trial with RT, condition and four features).
- Figures are in `output/`.

The **prediction model is not part of this version.** A first exploratory run showed that condition explains most of what
is predictable, and that the early EEG features added little. The features are being improved before the model is reported.
<!-- TODO: add model results here once re-run on all subjects -->

## Repository structure

```
.
├── SP_Project_cleaned.ipynb   # the full pipeline
├── README.md
├── requirements.txt
├── .gitignore
├── output/                    # figures and small result tables
└── data/
    └── features/              # msit_features.csv, selected_channels.json (small, committed)
```

Not in the repository (created when you run the notebook): `ds004621-main/` (metadata), and in `data/`:
`on004621/` (raw EEG), `downsampled/`, `ica_cleaned/`, `epochs/`, `channel_scores/`.
The notebook creates all of these folders itself, so nothing has to be set up by hand.

## Getting started

1. **Python 3.10 or newer.** Create an environment and install the requirements:
   ```bash
   python -m venv .venv
   source .venv/bin/activate        # Windows: .venv\Scripts\activate
   pip install -r requirements.txt
   ```
2. **Open `SP_Project_cleaned.ipynb` and run it from top to bottom.** No settings files or passwords are needed,
   because the dataset is public and the notebook downloads it. All settings are in the config cell at the top:
   - **Quick test:** set `FIRST_SUBJECT, LAST_SUBJECT = 1, 3` (default is all subjects, `1, 42`).
   - **Full run:** the raw MSIT EEG is about 27 GB, and ICA is the slowest step.
   - **Out of memory?** Lower `N_JOBS` and `N_JOBS_HEAVY` (number of subjects processed in parallel).

## Reproducibility

- **Per-subject checkpoints:** each stage saves one file per subject and skips it on the next run.
- **Parallel:** subjects are independent, so they run in parallel with `joblib` (2 workers for filtering and ICA, 3 for the rest).
- **Seed:** ICA uses a fixed random seed (`rng=42`).
- **Explicit parameters:** every parameter that matters (windows, thresholds, frequency range, ICA settings) is a named constant in the config cell or next to the function that uses it.

## Limitations

- Fp1 is used as a stand-in for an EOG channel, and only one blink component is removed. Muscle and horizontal eye movements can remain.
- FCz (the online reference) is re-created, so it is an estimate, not a recording.
- The early P300 window covers only the first part of the P300 wave and ignores its latency.
- Wavelet power at 4-5 Hz can include some of the slow evoked response, so theta is not a perfectly clean oscillation measure.
- Channels are chosen with all subjects (RT is never used), and top 5 is an arbitrary cut-off.
- The 150 uV rejection threshold is strict across about 120 channels. One subject (sub-16) had no usable epochs.
- Error trials are not excluded, and RT uses the first button press only.
- Epoch times are rounded to the 4 ms sampling grid (up to 2 ms jitter).

## Next steps

- Run and report all subjects end to end.
- More features: P300 latency and pre-stimulus alpha, with per-subject normalisation.
- Statistics for the condition effects (repeated-measures ANOVA or paired t-tests).
- Prediction model (ridge regression, leave-one-subject-out) compared with a condition-only baseline.
- Check how sensitive the results are to the main thresholds (150 uV, number of ICA components, baseline window).

## Data and citation

Data: Nencki-Symfonia EEG/ERP dataset, OpenNeuro ds004621. Please check the dataset page for its license and for the
recommended citation (Dzianok et al., *GigaScience*, 2022).
<!-- TODO: verify the citation and license on the OpenNeuro page, and add a LICENSE file for the code (for example MIT) -->

Main libraries: [MNE-Python](https://mne.tools), NumPy, pandas, SciPy, scikit-learn, joblib.
