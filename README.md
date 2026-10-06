# QFUSE-ECG

**Open-source, multilead ECG fiducial delineation using derivative-energy detection and quantile fusion.**

QFUSE-ECG is an interpretable signal-processing method for delineating six clinically important ECG fiducials:

- P-wave onset
- P-wave offset
- QRS onset
- QRS offset
- T-wave onset
- T-wave offset

The method delineates each lead independently, then combines lead-level estimates using fiducial-specific robust quantile fusion. It was designed to provide a transparent, modifiable alternative to proprietary ECG delineation systems without requiring a supervised training corpus.

> **Research use only.** The current evaluation was performed on a 100-ECG manually annotated holdout set. QFUSE-ECG has not yet been externally validated across the full range of rhythms, conduction abnormalities, noise levels, and ECG morphologies required for clinical deployment.

---

## Highlights

- **Fully open and interpretable** signal-processing pipeline
- **No supervised training required**
- Multilead delineation rather than reliance on a single lead
- Robust median/MAD-based activity thresholds
- RR-aware physiologic search windows
- Polarity-agnostic peak selection
- QRS-likeness vetoes for P- and T-wave candidate rejection
- Dedicated T-wave onset and offset logic
- Fiducial-specific **quantile fusion** across leads
- Competitive performance against a proprietary clinical algorithm
- Substantially lower error than default NeuroKit2 delineation in the reported test set

---

## Why QFUSE-ECG?

Global ECG boundaries are not necessarily best represented by the median timing across leads.

The earliest reliable expression of an electrical event may appear in only a subset of leads, while the latest repolarization activity may persist in a different subset. QFUSE-ECG therefore:

1. Delineates each lead independently.
2. Rejects implausible or noisy candidates using waveform-specific rules.
3. Collects valid lead-level boundary estimates.
4. Applies robust outlier filtering where appropriate.
5. Uses a different fusion quantile for each fiducial according to its physiologic role.

This allows early events such as onset boundaries to be weighted toward earlier lead evidence, while terminal events such as T-wave offset can be weighted toward later lead evidence.

---

## Method Overview

### 1. Preprocessing

Each lead is processed independently using:

- `float32` conversion
- Fourth-order zero-phase Butterworth bandpass filtering
- 0.5-40 Hz passband
- Median centering

### 2. Representative Beat Construction

R peaks are detected from a multilead composite based on median absolute amplitude across leads.

The R-detection pipeline uses:

- 8-20 Hz QRS-band filtering
- First derivative
- Squaring
- Short-window smoothing
- Robust median/MAD thresholding
- Local amplitude refinement

Approximately 1.2-second beats are extracted around accepted R peaks, spanning:

- 0.6 s before R
- 0.6 s after R

Atypical beats are rejected by template correlation, and retained beats are combined into a median representative beat for each lead.

### 3. Derivative-Energy Boundary Detection

For signal \(x[n]\):

```text
d[n] = x[n] - x[n - 1]
E[n] = moving_average(d[n]^2)
```

Waveform activity thresholds are estimated from quiet baseline regions using:

```text
threshold = median(E_baseline) + k * MAD(E_baseline)
```

Using robust statistics limits the influence of transient artifacts and residual waveform activity.

### 4. QRS Delineation

QRS onset and offset are identified first because they anchor the P- and T-wave search regions.

Core settings:

| Parameter | Value |
|---|---:|
| Search before R | 110 ms |
| Search after R | 175 ms |
| Energy smoothing | 12 ms |
| Threshold | 5.3 × MAD |

The preferred high-energy run is the run containing the lead-specific R location. If none contains R, the closest run is selected.

### 5. P-Wave Delineation

P-wave analysis is restricted to the pre-QRS region.

Core settings:

| Parameter | Value |
|---|---:|
| Search window | 300 ms to 50 ms before R |
| Safety gap before QRS | 35 ms |
| QRS-likeness veto | 1.6 |
| Energy window | 18 ms |
| Threshold | 3.4 × MAD |

Candidate P peaks are ranked using amplitude and prominence and screened for QRS-like spectral content.

P onset and offset are then estimated using derivative-energy return-to-baseline logic.

### 6. T-Wave Delineation

T-wave delineation uses separate rules for onset eligibility, peak selection, onset validation, and offset estimation.

Core settings:

| Parameter | Value |
|---|---:|
| Earliest onset search | 2 ms after QRS offset |
| Earliest peak search | 30 ms after QRS offset |
| Additional peak constraint | at least 50 ms after R |
| Maximum after R | 550 ms |
| Maximum after QRS offset | 500 ms |
| Maximum RR fraction | 50% |
| QRS-likeness veto | 1.0 |
| Energy window | 17 ms |
| Enter threshold | 3.8 × MAD |
| Exit threshold | 1.7 × MAD |

T onset uses an additional early-versus-late validation stage.

T offset is estimated using sustained return to baseline, with:

- Tail-noise estimation
- 3.8 × noise tolerance
- 30 ms hold requirement
- Biologic upper bounds
- Tangent-based fallback
- Bounded-window fallback

### 7. Multilead Quantile Fusion

After lead-level delineation, QFUSE-ECG combines finite estimates using fiducial-specific quantiles.

| Fiducial | Fusion rule |
|---|---:|
| P onset | raw 0.16 quantile |
| P offset | raw 0.54 quantile |
| QRS onset | robust 0.22 quantile |
| QRS offset | robust 0.56 quantile + late-cluster rescue |
| T onset | robust 0.20 quantile |
| T offset | robust 0.90 quantile |

For most boundaries, robust outlier filtering uses:

- Consensus threshold: `4.0`
- Minimum inlier leads: `3`

If too few inliers remain after filtering, the original finite set is retained.

---

## Benchmark Results

QFUSE-ECG was evaluated against manual expert fiducials on a **100-ECG holdout set sampled at 500 Hz**, where one sample corresponds to **2 ms**.

The comparison included:

- QFUSE-ECG
- Philips DXL
- NeuroKit2
- A supervised CNN trained on approximately 1 million Philips-labeled ECGs
- A zero-shot multimodal LLM using standardized ECG plot images

### Mean Absolute Error vs. Manual Annotation

| Method | P onset | P offset | QRS onset | QRS offset | T onset | T offset | Average |
|---|---:|---:|---:|---:|---:|---:|---:|
| **QFUSE-ECG** | 12.71 | 14.83 | 2.74 | **2.62** | 17.23 | 10.75 | **10.147** |
| Philips DXL | 18.35 | 13.20 | **2.13** | 2.75 | 28.04 | 9.60 | 12.345 |
| NeuroKit2 | 30.145 | 34.15 | 16.19 | 34.38 | 45.80 | 27.60 | 31.378 |
| CNN | 12.15 | 12.86 | 5.65 | 8.88 | 29.05 | 14.17 | 13.793 |
| Zero-shot multimodal LLM | **10.60** | **8.22** | 2.49 | 3.67 | **15.77** | **8.79** | **8.257** |

Values are in **samples**.

At 500 Hz, the average errors correspond to approximately:

| Method | Average MAE |
|---|---:|
| Zero-shot multimodal LLM | 16.5 ms |
| **QFUSE-ECG** | **20.3 ms** |
| Philips DXL | 24.7 ms |
| CNN | 27.6 ms |
| NeuroKit2 | 62.8 ms |

QFUSE-ECG achieved the **lowest QRS-offset MAE** in the comparison at **2.62 samples**, approximately **5.2 ms**.

Relative to Philips DXL, QFUSE-ECG reduced average MAE across the six fiducials by approximately **17.8%** in the reported test set.

---

## Installation

The manuscript describes the algorithm but does not specify the final repository package structure or dependency list.

Once the repository layout is finalized, replace this section with the exact installation command, for example:

```bash
git clone https://github.com/<USER_OR_ORG>/QFUSE-ECG.git
cd QFUSE-ECG
pip install -r requirements.txt
```

or, if packaged for pip:

```bash
pip install qfuse-ecg
```

---

## Quick Start

The manuscript defines the algorithmic pipeline but does not specify the final public Python API. The example below is therefore a **README template**, not a claim about the current function names.

```python
# Example API structure only.
# Replace with the actual repository interface.

from qfuse_ecg import delineate_ecg

fiducials = delineate_ecg(
    ecg=ecg_array,
    fs=500,
)

print(fiducials)
```

A natural output structure would be:

```python
{
    "p_onset": ...,
    "p_offset": ...,
    "qrs_onset": ...,
    "qrs_offset": ...,
    "t_onset": ...,
    "t_offset": ...
}
```

---

## Expected Input

The published evaluation used:

- Sampling rate: **500 Hz**
- 16 supplied ECG-like channels:
  - 12 standard leads
  - X
  - Y
  - Z
  - one supplied 3D-like channel

The current manuscript evaluates this specific input configuration. If the released implementation supports other lead sets or sampling rates, document those explicitly here.

---

## Output

QFUSE-ECG returns global fiducial locations for:

```text
P onset
P offset
QRS onset
QRS offset
T onset
T offset
```

For reproducible downstream use, the repository should document whether outputs are expressed as:

- sample indices
- milliseconds
- seconds
- indices within a representative beat
- indices within the original ECG recording

The study reports fiducial error in sample units at 500 Hz.

---

## Interpretability

A central design goal of QFUSE-ECG is auditability.

Each final ECG-level boundary can be traced to:

- lead-level candidate selection
- derivative-energy activity
- robust baseline estimation
- RR-window clipping
- fallback behavior
- cross-lead agreement
- the final wave-specific fusion quantile

This makes failure analysis substantially easier than with a purely black-box delineation model.

---

## Limitations

The current study has several important limitations:

1. The evaluation included only **100 manually annotated ECGs**.
2. External validation across broader rhythm and morphology subgroups has not yet been performed.
3. Inter-reader variability for the manual fiducials was not available.
4. Accepted lead-level estimates currently contribute equally within each fusion rule.
5. T-offset performance can be sensitive to fallback-derived late estimates.
6. The reported evaluation used a specific 16-channel working representation.
7. Some dataset-specific coordinate-frame handling was required during the study and should not be assumed to generalize to other datasets.

Future evaluation should include larger external cohorts, duplicate or adjudicated manual labels, additional error-distribution metrics, and confidence-weighted fusion.

---

## Reproducibility Notes

The manuscript reports the final active QFUSE-ECG parameters used in the evaluation, including:

- preprocessing passband
- QRS, P-wave, and T-wave search windows
- derivative-energy smoothing windows
- median/MAD multipliers
- RR-aware guards
- QRS-likeness vetoes
- onset-validation criteria
- T-offset fallback rules
- cross-lead consensus settings
- fiducial-specific fusion quantiles

These values should be kept version-controlled so that benchmark results can be reproduced exactly.

---

## Comparison With Other Approaches

QFUSE-ECG occupies a middle ground between simple single-lead heuristics and large learned models.

### Compared with NeuroKit2

In the reported test set, QFUSE-ECG had substantially lower MAE for all six fiducials than default NeuroKit2 delineation.

### Compared with Philips DXL

QFUSE-ECG achieved lower average error overall in the reported holdout set while remaining fully inspectable and modifiable.

### Compared with supervised learning

The evaluated CNN was trained on approximately one million ECGs using Philips DXL labels, but QFUSE-ECG remained more accurate for QRS onset, QRS offset, T onset, and T offset against the manual reference labels.

### Compared with multimodal LLM inference

A zero-shot multimodal LLM achieved the lowest overall error in the study, but it is methodologically different from a deterministic raw-signal processing library. QFUSE-ECG remains the strongest non-LLM open-source method in the reported comparison and provides fixed, inspectable sample-level processing rules.

---

## Citation

If you use QFUSE-ECG in academic work, please cite the accompanying manuscript:

```bibtex
@article{harvey_qfuse_ecg,
  title   = {Open-Source Quantile-Fused and Zero Shot LLM Methods Outperform Existing Methods for Delineation of ECG Fiducials},
  author  = {Harvey, Christopher J. and Noheria, Amit},
  note    = {Manuscript}
}
```

Replace this entry with the final journal citation, DOI, year, volume, and page information once available.

---

## License

Add the repository license here.

For a broadly reusable scientific software project, common options include:

- MIT
- BSD-3-Clause
- Apache-2.0

The manuscript does not specify a software license.

---

## Contributing

Contributions are welcome, especially for:

- External validation
- Additional sampling rates
- Alternative lead configurations
- Confidence-weighted fusion
- Improved T-wave offset handling
- Rhythm-specific evaluation
- Unit tests and benchmarking
- Runtime optimization
- Additional public datasets

Please open an issue before making substantial changes to the core delineation logic so that benchmark reproducibility can be preserved.

---

## Authors

**Christopher J. Harvey**  
**Amit Noheria**

---

## Status

QFUSE-ECG should currently be considered a **research implementation under active validation**, not a clinical diagnostic device.

