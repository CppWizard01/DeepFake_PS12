# Audio Anti-Spoofing — DeepFake Detection

A binary audio anti-spoofing system that classifies utterances as bonafide or spoofed. The project supports five model architectures, trains and evaluates on the ASVspoof 2019 Logical Access (LA) and Physical Access (PA) scenarios, and ships a FastAPI web interface for live inference.

---

## Repository layout

```text
.
├── src/
│   ├── __init__.py
│   ├── train.py               # Training loop with checkpoint saving
│   ├── evaluate.py            # Checkpoint evaluation with ROC/DET plots
│   ├── benchmark.py           # MACs and latency profiling
│   ├── metrics.py             # EER and binary classification metrics
│   ├── utils.py               # Seeding, JSON helpers, logit extraction
│   ├── models/
│   │   ├── __init__.py
│   │   ├── factory.py         # build_model() dispatcher
│   │   ├── rawnet.py          # RawNet (1-D waveform residual network)
│   │   └── spec_rnet.py       # SpecRNet with LFCC frontend and FocalLoss
│   └── data/
│       ├── dataset.py         # CMManifestDataset, sampler utilities
│       └── make_manifests.py  # Builds CSV manifests from ASVspoof protocols
├── scripts/
│   ├── run_task1.sh           # Full CNN/CRNN pipeline
│   ├── run_task1_rawnet.sh    # RawNet pipeline with recommended flags
│   └── run_task1_specrnet.sh  # SpecRNet pipeline
├── manifests/                 # Pre-built CSV manifests (la_*, pa_*)
├── reports/                   # Per-run metrics JSON, ROC/DET PNG outputs
├── webapp/                    # FastAPI inference server
│   ├── main.py
│   ├── tts_engine.py
│   ├── run_webapp.sh
│   ├── requirements.txt
│   └── HOSTING.md
├── AASIST_FINAL.ipynb         # AASIST experiment notebook
└── README.md
```

---

## Requirements

Python 3.10 or later is recommended.

### Core dependencies

```text
torch>=2.1.0
torchaudio>=2.1.0
numpy
matplotlib
tqdm
scikit-learn
soundfile
pandas
thop
```

### Web application dependencies

```text
fastapi
uvicorn[standard]
python-multipart
TTS
```

Install all dependencies for the training pipeline:

```bash
pip install torch torchaudio numpy matplotlib tqdm scikit-learn soundfile pandas thop
```

Install CPU-only PyTorch wheels (recommended for development machines without a GPU):

```bash
pip install --index-url https://download.pytorch.org/whl/cpu torch==2.5.1+cpu torchaudio==2.5.1+cpu
```

Install webapp dependencies:

```bash
pip install -r webapp/requirements.txt
```

---

## Setup

### 1. Clone and create a virtual environment

```bash
git clone <repository-url>
cd DeepFake_PS12
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
```

### 2. Obtain the ASVspoof 2019 dataset

Download the LA and PA partitions from the official ASVspoof website and place them under the repository root so the layout matches:

```text
LA/LA/ASVspoof2019_LA_cm_protocols/
LA/LA/ASVspoof2019_LA_train/flac/
LA/LA/ASVspoof2019_LA_dev/flac/
LA/LA/ASVspoof2019_LA_eval/flac/
PA/PA/ASVspoof2019_PA_cm_protocols/
PA/PA/ASVspoof2019_PA_train/flac/
PA/PA/ASVspoof2019_PA_dev/flac/
PA/PA/ASVspoof2019_PA_eval/flac/
```

### 3. Build CSV manifests

```bash
python -m src.data.make_manifests --data-root . --out-dir data/manifests --verify-exists
```

Pre-built manifests are also committed under `manifests/` and contain the expected row counts (LA: 25 380 train / 24 844 dev / 71 237 eval; PA: 54 000 / 29 700 / 134 730).

---

## Models

| Key | Architecture | Input | Loss |
|---|---|---|---|
| `cnn` | CNN baseline | Mel-spectrogram | BCEWithLogitsLoss |
| `crnn` | CNN + GRU baseline | Mel-spectrogram | BCEWithLogitsLoss |
| `rawnet` | 1-D residual network with attentive stats pooling | Raw waveform | Weighted BCE |
| `specrnet` | SpecRNet with linear-filterbank LFCC frontend | LFCC features | FocalLoss (γ=2) |
| `audiomamba` | AudioMamba | Raw waveform | Weighted BCE |

---

## Training

```bash
python -m src.train \
  --data-root . \
  --train-manifest data/manifests/la_train.csv \
  --val-manifest data/manifests/la_dev.csv \
  --output-dir models/run1/rawnet \
  --model rawnet \
  --epochs 20 \
  --batch-size 64 \
  --lr 1e-3 \
  --weight-decay 1e-4 \
  --trim-silence \
  --pre-emphasis \
  --augment \
  --balance-data \
  --amp
```

Key flags:

- `--model` — one of `cnn`, `crnn`, `rawnet`, `specrnet`, `audiomamba`
- `--augment` — enables random gain, shift, noise injection, and segment attenuation (on by default for `rawnet` and `audiomamba`)
- `--balance-data` — uses `WeightedRandomSampler` and positive-class weighting
- `--amp` — mixed-precision training (CUDA only)
- `--resume-from <path>` — resume from a saved checkpoint

Checkpoints and training curves are written to `--output-dir`. The best checkpoint by validation EER is saved as `best.pt`; the most recent epoch as `last.pt`.

### Pre-configured scripts

```bash
# CNN and CRNN, two runs
bash scripts/run_task1.sh

# RawNet, two runs with recommended preprocessing
bash scripts/run_task1_rawnet.sh

# SpecRNet, two runs with LFCC settings
bash scripts/run_task1_specrnet.sh
```

Each script follows the two-run protocol defined in `manifests/runs.yaml`: Run 1 trains on LA and tests on LA (in-domain) and PA (cross-domain); Run 2 trains on PA and tests on PA (in-domain) and LA (cross-domain).

---

## Evaluation

```bash
python -m src.evaluate \
  --data-root . \
  --manifest data/manifests/la_eval.csv \
  --checkpoint models/run1/rawnet/best.pt \
  --output-dir reports/run1/rawnet/in_domain
```

Outputs written to `--output-dir`:

- `metrics.json` — EER, accuracy, precision, recall, F1 per class, and confusion matrix
- `roc.png` — ROC curve
- `det.png` — DET curve

The threshold stored in the checkpoint is used by default. Pass `--threshold <float>` to override it.

---

## Benchmarking

Measures multiply-accumulate operations (MACs) and mean inference latency over a 5-second clip:

```bash
python -m src.benchmark \
  --data-root . \
  --manifest data/manifests/la_eval.csv \
  --checkpoint models/run1/rawnet/best.pt \
  --output-json reports/run1/rawnet/benchmark.json \
  --iters 100
```

The output JSON records MACs, parameter count, latency in milliseconds, checkpoint size, and hardware identifiers.

---

## Web application

The webapp is a FastAPI server that exposes an audio upload endpoint for live spoof detection.

### Start the server

Run from the repository root after activating the virtual environment and installing `webapp/requirements.txt`:

```bash
./webapp/run_webapp.sh
```

The server binds to `0.0.0.0:7860` and is accessible locally at:

```text
http://127.0.0.1:7860
```

### Custom Python interpreter

`PYTHON_BIN` is optional. Set it when you need the script to use a specific interpreter:

```bash
# Default: uses the active environment
./webapp/run_webapp.sh

# Explicit interpreter
PYTHON_BIN=/path/to/python ./webapp/run_webapp.sh
```

### Model path overrides

The webapp resolves model weights from the repository by default. Override with environment variables:

```bash
MODEL_PATH=/path/to/model \
XTTS_CHECKPOINT=/path/to/checkpoint \
XTTS_CONFIG=/path/to/config.json \
XTTS_VOCAB=/path/to/vocab.json \
./webapp/run_webapp.sh
```

### Hosting notes

See `webapp/HOSTING.md` for deployment instructions, reverse proxy configuration, and production considerations.

---

## Experiment notebook

`AASIST_FINAL.ipynb` contains the AASIST model experiments. Open with Jupyter:

```bash
pip install jupyter
jupyter notebook AASIST_FINAL.ipynb
```

---

## Output structure

```text
models/
└── run{1,2}/
    └── {model}/
        ├── best.pt          # Best checkpoint by validation EER
        ├── last.pt          # Latest epoch checkpoint
        ├── history.json     # Per-epoch loss, accuracy, EER
        ├── best_summary.json
        ├── data_summary.json
        ├── loss_curve.png
        ├── val_eer_curve.png
        └── accuracy_curve.png

reports/
└── run{1,2}/
    └── {model}/
        ├── in_domain/
        │   ├── metrics.json
        │   ├── roc.png
        │   └── det.png
        ├── cross_domain/
        │   ├── metrics.json
        │   ├── roc.png
        │   └── det.png
        └── benchmark.json
```
