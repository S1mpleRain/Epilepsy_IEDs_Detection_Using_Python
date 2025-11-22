# Automated Interictal Epileptiform Discharge (IED) Detection System

## Overview

This project implements an automated IED detection system for EEG signal analysis based on International Federation of Clinical Neurophysiology (IFCN) standards. The system uses multi-criteria pattern recognition to identify epileptiform discharges with configurable sensitivity and specificity.

## Code Functionality

This is an automated epileptic spike detection pipeline that processes clinical EEG recordings:

1. **Loads and preprocesses** EEG data in EDF format
2. **Applies filtering** (60Hz notch filter + 1-70Hz bandpass)
3. **Removes artifacts** using Independent Component Analysis (ICA)
4. **Detects spike candidates** using amplitude and sharpness thresholds
5. **Validates detections** against IFCN morphological criteria
6. **Merges overlapping events** across channels
7. **Generates reports** with detection statistics and ground truth comparison

## Key Features

### Multi-Criteria Detection Algorithm

The system employs a **weighted scoring system** based on four IFCN criteria:

| Criterion | Description | Weight |
|-----------|-------------|--------|
| **1. Sharp Transient** | Rising edge derivative threshold | +1 |
| **2. Duration Constraint** | Full-width half-maximum: 20-70ms (spikes only) | +1 |
| **3. After-wave Presence** | Slow-wave detection with baseline correction | +1 |
| **4. Spatial Consistency** | Multiple channels within 20ms | +2 |

**Classification Threshold**: Score ≥3 required for IED confirmation

### Signal Processing Pipeline

```
RAW EEG DATA (EDF Format)
    ↓
[Stage 1: Preprocessing]
├─ 60Hz Notch Filter (power line interference removal)
├─ 1-70 Hz Bandpass Filter
├─ Channel Standardization (10-20 system)
└─ Channel Type Classification (EEG/EOG/ECG/EMG/MISC)
    ↓
[Stage 2: Artifact Removal]
├─ ICA Decomposition (up to 40 components)
├─ Automatic EOG Component Removal
├─ Automatic ECG Component Removal
└─ Average EEG Reference
    ↓
[Stage 3: Spike Detection]
├─ 0.5-60 Hz Bandpass (optimal for spikes & slow waves)
├─ Amplitude Threshold: Configurable (default: 1.8σ)
├─ Sharpness Threshold: Configurable (default: 0.5σ)
└─ Peak Detection with Minimum Separation (150ms)
    ↓
[Stage 4: IFCN Validation]
├─ Criterion 1: Sharp transient check
├─ Criterion 2: Duration validation (20-70ms, spikes only)
├─ Criterion 3: Slow after-wave detection with baseline correction
└─ Criterion 4: Spatial consistency (≥4 channels within 20ms)
    ↓
[Stage 5: Event Merging]
├─ Temporal clustering (500ms window)
├─ Cross-channel aggregation
└─ Final IED event list
    ↓
DETECTION REPORT
├─ Total IED count
├─ IED rate (events/minute)
├─ Average duration per event
├─ Spatial distribution statistics
└─ Ground truth comparison (if available)
```

## Technical Requirements

### Software Dependencies

```python
Python 3.8+
mne >= 1.0          # Clinical neurophysiology toolkit
numpy >= 1.20       # Numerical computing
scipy >= 1.7        # Signal processing (peak detection, filtering)
pandas >= 1.3       # Data manipulation
matplotlib >= 3.0   # Visualization
```

### Installation

```bash
pip install mne numpy scipy pandas matplotlib
```

### Hardware Recommendations

- **Minimum**: 4GB RAM, dual-core CPU
- **Recommended**: 8GB+ RAM, quad-core CPU for faster ICA and large candidate sets
- **Note**: Low threshold settings may require significant memory (see Limitations)

## Data Format

### Input Requirements

**File Format**: EDF (European Data Format)
- Standard format for clinical EEG recordings
- Multi-channel time-series data with metadata

**Channel Configuration**:
- **EEG Channels**: Following International 10-20 System
  - Frontal: Fp1, Fp2, F3, F4, F7, F8, Fz
  - Central: C3, C4, Cz
  - Temporal: T7, T8, P7, P8
  - Parietal: P3, P4, Pz
  - Occipital: O1, O2

- **Auxiliary Channels** (optional):
  - EOG: Eye movement monitoring (LOC1, LOC2 or Fp1-Fp2 bipolar)
  - ECG: Cardiac artifact detection (EKGL, EKGR)
  - EMG: Muscle artifact detection

**Sampling Rate**: 200-512 Hz (typically 250 Hz)

**Recording Duration**: Optimized for 10-60 minute segments

### Ground Truth Format

Optional `.rec` files for validation:
```
channel_id,start_time,end_time,event_type
```
- Event types: 1=SPSW, 2=PLED, 3=GPD, 5=Artifact, 6=Other

## Usage

### Running the Detection

Open and run the Jupyter notebook:
```bash
jupyter notebook For_TUM_data.ipynb
```

Or use the Python script equivalents in the `Agashe/` directory.

### Detection Parameters

Key parameters can be adjusted:

```python
# Detection thresholds
AMPLITUDE_SD_THRESH = 1.8        # Standard deviations above mean
SHARPNESS_SD_THRESH = 0.5        # Derivative threshold

# IFCN morphology criteria
SPIKE_DURATION_MIN_MS = 20       # 20 ms minimum
SPIKE_DURATION_MAX_MS = 70       # 70 ms maximum (spikes only)
AFTERWAVE_MIN_AMPLITUDE = 0.25   # 25% of spike amplitude

# Slow after-wave detection
SLOW_MAX_HZ = 7.0                # Low-pass filter for slow waves
BASELINE_WINDOW_S = 0.2          # Baseline estimation window
MIN_AFTERWAVE_DURATION_MS = 20   # Minimum after-wave duration
MAX_AFTERWAVE_DURATION_MS = 400  # Maximum after-wave duration

# Spatial consistency
MIN_CONCURRENT_CHANNELS = 4      # Channels required for validation
SPATIAL_CONSISTENCY_WINDOW_S = 0.020  # 20 ms time window

# Event merging
MERGE_WINDOW_S = 0.5             # 500 ms merging window
MIN_EVENT_SEPARATION_S = 0.15    # Minimum separation between events
```

### Performance Tuning

**For faster processing (reduced sensitivity):**
```python
AMPLITUDE_SD_THRESH = 4.0
SHARPNESS_SD_THRESH = 2.5
```
- Fewer candidates (~100-500)
- Processing time: <2 min per 10-min recording

**For higher sensitivity (slower processing):**
```python
AMPLITUDE_SD_THRESH = 1.8
SHARPNESS_SD_THRESH = 0.5
```
- More candidates (~10,000-100,000)
- Processing time: 5-15 min per 10-min recording

## Output Format

### Console Output Example

```
================================================================================
DETECTION PARAMETERS:
================================================================================
  Amplitude threshold:      1.8σ
  Sharpness threshold:      0.5σ
  Score threshold:          3
  Spatial time window:      20ms
  Min concurrent channels:  4
  Duration range:           20-70ms (spikes only)
  After-wave amplitude:     ≥25% of spike amplitude
================================================================================

[IED Detection] Found 74225 candidate events across all channels.
[IED Detection] Found 892 high-confidence events (score >= 3) before merging.
[IED Merging] Merged into 156 unique IED events.
[Statistics] Average duration: 42.3 ms
[Statistics] Average concurrent channels: 8.7
[Statistics] IED rate: 7.92 events/minute
```

### Detection Report

The system generates a comprehensive report including:

1. **Detection Summary**:
   - Total events detected
   - Detection rate (events/minute)
   - Duration statistics (mean, SD, range)
   - Concurrent channel statistics

2. **Detected Events Table**:
   - Event number
   - Time (seconds)
   - Duration (milliseconds)
   - Detection score
   - Number of concurrent channels

3. **Ground Truth Comparison** (if .rec file available):
   - Confusion matrix (TP, FP, FN)
   - Sensitivity (Recall)
   - Precision (PPV)
   - F1 Score
   - Matched events
   - False positives analysis
   - False negatives analysis

## Limitations

### Clinical Limitations
- **Not a diagnostic tool**: Requires clinical correlation by qualified neurologists
- **Optimal for**: Routine scalp EEG (not invasive electrodes)
- **Sensitivity limitations**: May miss very small or deeply-located spikes
- **Specificity challenges**: Highly rhythmic artifacts may occasionally pass filters

### Technical Limitations

- **Physical artifact rejection**: Currently lacks effective removal of non-physiological noise from physical environmental changes (e.g., electrode friction, cable movement, patient motion artifacts). Future versions will incorporate enhanced artifact detection algorithms

- **Computational complexity**:
  - Low threshold settings (e.g., 1.8σ amplitude) can generate 10,000-100,000 candidate events, requiring significant processing time and memory
  - Long-duration recordings (>1 hour) may require substantial RAM (8GB+ recommended) for processing
  - O(N²) spatial consistency checking becomes a bottleneck with large candidate sets
  - Future optimization planned: sliding window approach and vectorized operations to reduce memory footprint

- **Processing scalability**: Current implementation is optimized for 10-60 minute recordings; whole-day recordings may experience performance degradation

## Scientific Background

### IFCN Morphological Criteria

According to Kane et al. (2017), IEDs are characterized by:

- **Morphology**: Di- or tri-phasic waves with pointed peak
- **Duration**: 20-70 milliseconds for spikes (this system focuses on spikes)
- **Asymmetry**: Sharply rising phase and slowly decaying phase
- **After-wave**: Associated slow wave of opposite polarity
- **Background Disruption**: Distinguished from background activity
- **Spatial Distribution**: Multi-channel presence indicates brain origin

### Evidence Base

This implementation incorporates findings from peer-reviewed literature:

1. **Kane et al. (2017)** - IFCN glossary defining IED morphology standards
2. **Kural et al. (2020)** - Multi-criteria approach for improved specificity
3. **Reus et al. (2022)** - Spatial consistency for false positive reduction
4. **Tyvaert et al. (2017)** - IFCN best practices for EEG filtering

## Validation Dataset

The system has been tested on the TUM SPSW dataset:

- **Total Files**: 40 SPSW (Spike-and-Slow-Wave) EEG recordings
- **Format**: EDF with corresponding .rec annotation files
- **Dataset IDs**: 001, 002, 005, 008, 010, 012, 015, 017, 018, 020, 022, 023, 024, 032, 035, 037, 038, 040, 041, 043, 046 (×2), 047, 048, 049, 054, 058, 064, 067, 070, 072, 074, 077, 088, 089, 091, 093, 098 (×2), 099
- **Annotation Types**: SPSW, PLED, GPD, Artifact, Other

**Note**: Performance varies by file and parameter settings. Tested files include 005, 037, 038, 041, 070 with good results.

## Future Development

### Planned Enhancements

- **Performance optimization**: Sliding window approach for O(N) spatial consistency
- **Enhanced artifact rejection**: Physical noise detection algorithms
- **Real-time processing**: Streaming analysis capability
- **Parameter auto-tuning**: Adaptive thresholds based on recording quality
- **Deep learning integration**: CNN/RNN for pattern recognition

## License

This project is intended for **academic research and educational purposes only**.

**Medical Disclaimer**: This software is NOT approved for clinical diagnosis or treatment decisions. All results must be reviewed by qualified healthcare professionals. The authors assume no liability for clinical use of this software.

## Author & Contact

**Developer**: Boyu Qian (Neural Engineering, Duke University)

**Laboratory**: Dr. Shruti Agashe's Laboratory, Department of Biomedical Engineering

For questions or collaboration:
- GitHub Issues: Submit to project repository
- Email: boyu.qian@duke.edu

## Acknowledgments

- International Federation of Clinical Neurophysiology (IFCN) for standardization
- MNE-Python development team
- Clinical neurophysiology community for open science practices

---

**Version**: 2.0  
**Last Updated**: November 2024  
**Tested On**: Python 3.8-3.12, MNE 1.0+
