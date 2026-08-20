# VoiceLab - Deepfake Audio Generation and Detection

VoiceLab is a FastAPI web application and research codebase for deepfake audio detection, voice cloning experiments, and anti-spoofing evaluation. The project was reconstructed from the submitted GitHub repository and the original remote-server working folder.

The production recommendation from the audit is detection-first deployment on CPU, with XTTS voice generation enabled later through a GPU-backed model service.

## What Works Now

- FastAPI app with static frontend.
- Audio upload and 16 kHz mono normalization.
- Dual-checkpoint AASIST spoof detection.
- Detection history and health endpoint.
- Optional XTTS voice generation when large external model assets are configured.
- Training/evaluation utilities for ASVspoof LA/PA experiments.
- Audit, gap, model-size, GPU, and deployment strategy documentation.

## Repository Layout

```text
.
|-- .github/workflows/       # CI smoke checks
|-- configs/                 # Runtime/training config files
|-- deployment/              # Deployment guide and hosting notes
|-- docker/                  # Container build files
|-- docs/                    # Audit, gap analysis, GPU/model reports
|-- manifests/               # ASVspoof CSV manifests
|-- models/checkpoints/      # Small production AASIST checkpoints
|-- reports/                 # Selected experiment metrics and plots
|-- scripts/                 # Training/evaluation shell entrypoints
|-- src/                     # Training, data, model, explainability code
|-- tests/                   # Lightweight repository contract tests
`-- webapp/                  # FastAPI app and frontend
```

Large datasets, generated audio, tensor caches, and XTTS checkpoints are intentionally excluded from Git.

## Quick Start: Detection-Only

Create a virtual environment and install the CPU runtime:

```bash
python -m venv .venv
.venv\Scripts\activate
pip install -r webapp/requirements.txt
```

Start the app:

```bash
set VOICELAB_ENABLE_TTS=false
python webapp/main.py --host 0.0.0.0 --port 7860
```

Open:

```text
http://127.0.0.1:7860
```

Health check:

```bash
curl http://127.0.0.1:7860/health
```

In detection-only mode, `model_loaded` is expected to be `false` because XTTS is disabled. `classifier_loaded` should be `true` when both AASIST checkpoints load.

## Docker

```bash
copy .env.example .env
docker compose up --build
```

The Docker image is CPU-oriented and intended for the cheapest practical deployment path.

## Optional XTTS Generation

XTTS generation needs large model files and is not bundled in Git. To enable it, install optional dependencies and set the model path:

```bash
pip install -r webapp/requirements-tts.txt
set VOICELAB_ENABLE_TTS=true
set XTTS_MODEL_PATH=C:\path\to\xtts-model-dir
python webapp/main.py --host 0.0.0.0 --port 7860
```

The XTTS model directory must contain:

- `best_model.pth` or `model.pth`
- `config.json`
- `vocab.json`

For live public hosting, use a GPU worker or a GPU ML platform instead of a small CPU web host.

## Training

Download ASVspoof 2019 LA/PA and place it under:

```text
LA/LA/...
PA/PA/...
```

Build manifests:

```bash
python -m src.data.make_manifests --data-root . --out-dir data/manifests --verify-exists
```

Train a detector:

```bash
python -m src.train ^
  --data-root . ^
  --train-manifest data/manifests/la_train.csv ^
  --val-manifest data/manifests/la_dev.csv ^
  --output-dir models/run1/rawnet ^
  --model rawnet ^
  --epochs 20 ^
  --batch-size 64 ^
  --trim-silence ^
  --pre-emphasis ^
  --augment ^
  --balance-data
```

Available training model keys:

- `cnn`
- `crnn`
- `rawnet`
- `specrnet`

AudioMamba was described in experiments but its implementation was not present in the recovered artifacts, so it is not exposed as a runnable training option.

## Key Documentation

- [Project audit](docs/PROJECT_AUDIT.md)
- [Gap analysis](docs/GAP_ANALYSIS.md)
- [Model and GPU report](docs/MODEL_AND_GPU_REPORT.md)
- [Deployment strategy](docs/DEPLOYMENT_STRATEGY.md)
- [Deployment guide](deployment/README.md)

## Production Notes

Recommended first deployment:

- CPU container.
- Detection-only mode.
- Bundled AASIST checkpoints.
- No ASVspoof datasets, runtime uploads, generated audio, or XTTS weights in Git.

Recommended full deployment:

- CPU API/frontend service.
- GPU XTTS worker.
- External object storage for uploads and outputs.
- Persistent job store and rate limiting.
