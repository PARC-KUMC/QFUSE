# QFUSE-ECG

QFUSE-ECG is an open-source ECG signal delineation tool for identifying six major fiducial boundaries: **P-wave onset, P-wave offset, QRS onset, QRS offset, T-wave onset, and T-wave offset**. It is designed as a transparent signal-processing approach that can be inspected, modified, and adapted for research workflows without requiring a trained machine-learning model.

QFUSE works by first detecting representative cardiac beats and processing each ECG lead independently. It uses filtered waveform information and **derivative energy** to identify regions of electrical activity, together with robust baseline estimates based on the median and median absolute deviation. Separate physiologic search regions and decision rules are used for the P wave, QRS complex, and T wave so that each waveform is delineated according to its characteristic morphology rather than with a single generic threshold.

The defining feature of QFUSE is its **multilead quantile-fusion strategy**. Fiducial locations are estimated separately in each available lead and then combined across leads using boundary-specific robust quantiles. Earlier quantiles are used for boundaries where the first reliable evidence of activity is important, while later quantiles are used for terminal boundaries such as T-wave offset. This allows QFUSE to use information distributed across the ECG instead of relying on one lead or a simple median across leads.

QFUSE-ECG is intended for research use and can serve as an interpretable foundation for ECG interval measurement, morphology analysis, and large-scale cardiovascular signal processing.

## License

QFUSE-ECG is open-source software distributed under the **GNU General Public License v3.0 (GPL-3.0)**.
