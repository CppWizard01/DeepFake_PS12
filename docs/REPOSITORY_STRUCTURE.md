# Refactored Repository Structure

This structure separates production inference, training/research code, deployment assets, documentation, and runtime data.

```text
VoiceLab/
|-- .github/workflows/
|   `-- ci.yml
|-- configs/
|   `-- voicelab.example.yaml
|-- deployment/
|   `-- README.md
|-- docker/
|   `-- Dockerfile
|-- docs/
|   |-- PROJECT_AUDIT.md
|   |-- GAP_ANALYSIS.md
|   |-- MODEL_AND_GPU_REPORT.md
|   |-- DEPLOYMENT_STRATEGY.md
|   |-- REPOSITORY_STRUCTURE.md
|   `-- IMPROVEMENTS_ROI.md
|-- manifests/
|-- models/
|   `-- checkpoints/
|       |-- ModelA_LA_bestnew.pt
|       `-- ModelB_PA_bestnew.pt
|-- reports/
|-- scripts/
|-- src/
|   |-- data/
|   |-- improved/
|   |-- models/
|   |-- train.py
|   |-- evaluate.py
|   `-- benchmark.py
|-- tests/
|-- webapp/
|   |-- static/
|   |-- main.py
|   |-- aasist_classifier.py
|   |-- tts_engine.py
|   |-- requirements.txt
|   `-- requirements-tts.txt
|-- .env.example
|-- .gitignore
|-- docker-compose.yml
`-- README.md
```

## What Moved Conceptually

- `webapp/` is the production API and UI surface.
- `models/checkpoints/` contains only small production AASIST checkpoints.
- `src/` contains training, evaluation, recovered data modules, model definitions, and research scripts.
- `docs/` contains audit and decision records.
- `deployment/` and `docker/` contain production packaging.
- Runtime files stay under `webapp/uploads`, `webapp/outputs`, and `webapp/temp`, but are ignored by Git.

## What Stays Out of Git

- Full ASVspoof datasets.
- Tensor caches.
- XTTS checkpoints and experiment folders.
- Generated audio.
- Local pip caches and binaries.
- Python bytecode.

