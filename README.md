# GNSS Mahalanobis Anomaly Detector

A robust, nominal-only anomaly detector for GNSS I/Q data using interpretable RF features and Mahalanobis distance.

DATASET https://zenodo.org/records/13846381
## 🚀 Key Features

- **Nominal-Only Training**: Trains on clean signals (`S*.mat`), detects anomalies in spoofed signals (`SS*.mat`).
- **Robust Pipeline**:
  - Flexible MAT file parsing (auto-detects I/Q keys).
  - Feature extraction: Spectral entropy, flatness, power, kurtosis, etc.
  - **Ledoit-Wolf** covariance estimation for stability with limited data.
  - Per-file error handling and nan/inf sanitation.
- **Configurable**: Adjustable window specs, downsampling, and detection thresholds.
- **Visual Outputs**: Generates score distributions, time-series plots, and threshold sensitivity curves.

## 📦 Prerequisites

Requires Python 3.8+ and the following libraries:

```bash
pip install numpy scipy scikit-learn matplotlib
```

## 📂 Data Structure

The script expects a root directory with .mat files recursively organized. It distinguishes files by filename prefix:

- **Clean/Nominal**: `S*.mat` (e.g., `S1_50.mat`, `S_clean.mat`)
- **Spoof/Anomaly**: `SS*.mat` (e.g., `SS1_50.mat`, `SS_attack.mat`)

**Example Layout:**
```text
data/
├── oct_18/
│   ├── S1_50.mat
│   └── SS1_50.mat
└── oct_19/
    ├── S2_100.mat
    └── SS2_100.mat
```

## 🛠 Usage

Run the script pointing to your data directory:

```bash
python mahal_poc_robust.py --data_root ./data/oct_18
```

### Common Arguments

| Argument | Default | Description |
|----------|---------|-------------|
| `--data_root` | - | Root folder for single-directory mode |
| `--train_dir` | - | Training directory (cross-day mode) |
| `--test_dir` | - | Test directory (cross-day mode) |
| `--categories` | All | Filter to categories 1-4 |
| `--auto_fs` | Off | Auto-detect fs from category |
| `--fs_hz` | 25MHz | Default sample rate |
| `--target_fs_hz` | 5MHz | Downsample target |
| `--win_ms` | 1.0 | Window length (ms) |
| `--hop_ms` | 1.0 | Window hop (ms) |
| `--threshold_quantile` | 0.995 | Detection threshold |
| `--outdir` | `outputs_mahalanobis` | Output directory |

### Example Command

Train on clean data, test on spoof, use overlapping windows (50%), and set a stricter threshold:

```bash
python mahal_poc_robust.py \
  --data_root ./gnss_data \
  --win_ms 2.0 \
  --hop_ms 1.0 \
  --threshold_quantile 0.99 \
  --outdir ./results
```

### Cross-Day Validation (Recommended)

Train on one day, test on another for robust generalization evaluation:

```bash
# Train on Oct 18, test on Oct 19
python mahal_poc_robust.py \
  --train_dir ./data/oct_18 \
  --test_dir ./data/oct_19 \
  --auto_fs \
  --outdir ./results_cross_day
```

### Per-Category Training (Most Rigorous)

Train a separate detector for each category. This is the **scientifically honest** approach since each category has different RF statistics:

```bash
# Category 1: Baseline (25 MHz, 16-bit)
python mahal_poc_robust.py --data_root ./data --categories 1 --auto_fs --outdir ./results_cat1

# Category 2: Multipath (25 MHz, 16-bit)  
python mahal_poc_robust.py --data_root ./data --categories 2 --auto_fs --outdir ./results_cat2

# Category 3: High Sampling Rate (50 MHz, 16-bit)
python mahal_poc_robust.py --data_root ./data --categories 3 --auto_fs --outdir ./results_cat3

# Category 4: Low Quantization (25 MHz, 8-bit)
python mahal_poc_robust.py --data_root ./data --categories 4 --auto_fs --outdir ./results_cat4
```

### Categories Reference

| Cat | Description | Sample Rate | Quantization |
|-----|-------------|-------------|--------------|
| 1 | Baseline | 25 MHz | 16-bit |
| 2 | Multipath | 25 MHz | 16-bit |
| 3 | High Fs | 50 MHz | 16-bit |
| 4 | Low Quant | 25 MHz | 8-bit |

## 📊 Outputs

The script generates the following in `outdir`:

1.  **`score_distributions.png`**: Histogram of Mahalanobis distances (D²) for Nominal vs. Spoof sets.
2.  **`threshold_sweep.png`**: False Alarm Rate (FAR) vs. Detection Rate for various percentiles.
3.  **`score_timeseries.png`**: Temporal view of scores for one example clean file vs. one spoof file.
4.  **`scores_nominal.npy` / `scores_spoof.npy`**: Raw score arrays for further analysis.

## 🔬 Technical Details

1.  **Loading**: Reads I/Q data from `.mat` files, auto-selecting the largest complex array.
2.  **Preprocessing**: Optional downsampling using polyphase resampling (`resample_poly`).
3.  **Feature Extraction** (per window):
    *   Total Power (dB)
    *   Noise Floor (dB)
    *   Spectral Entropy
    *   Spectral Flatness
    *   PSD Variance
    *   Amplitude Kurtosis
    *   Amplitude Std Dev
4.  **Modeling**:
    *   StandardScaler for feature normalization.
    *   **Ledoit-Wolf** estimator for robust covariance matrix estimation.
5.  **Scoring**:
    *   Computes Squared Mahalanobis Distance ($D^2$) for all test windows.
    *   Anomalies are flagged if $D^2 > Threshold$.
