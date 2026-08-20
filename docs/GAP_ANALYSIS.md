# VoiceLab Gap Analysis

Status: Phase 2 gap analysis based on submitted repo vs server copy.

## Critical Missing Components in Submitted Repo

### Missing Source Files

The submitted repo imports files that are not present:

- `src/data/__init__.py`
- `src/data/dataset.py`
- `src/data/make_manifests.py`
- `src/data/augmentation.py`
- `src/models/cnn_baseline.py`
- `src/models/audio_mamba.py`

The server copy contains `src/data/*` and `src/models/cnn_baseline.py`, but no `src/models/audio_mamba.py` was found during the source inventory. Any `audiomamba` option must either be removed, recovered from another source, or implemented properly.

### Missing Production Model Assets

The submitted repo does not include the AASIST checkpoints required by `webapp/main.py`:

- `ModelA_LA_bestnew.pt`
- `ModelB_PA_bestnew.pt`

Both are present in the server copy:

- `C:\Users\EKANSH SINGAL\Downloads\Compressed\ekansh_60gb\ekansh\ModelA_LA_bestnew.pt`
- `C:\Users\EKANSH SINGAL\Downloads\Compressed\ekansh_60gb\ekansh\ModelB_PA_bestnew.pt`

The submitted repo also lacks XTTS model assets required for generation:

- XTTS checkpoint (`best_model.pth`, `model.pth`, or equivalent)
- XTTS `config.json`
- XTTS `vocab.json`

These are present in the server copy under:

- `experiments/dl1_xtts_ft/speaker1/...`
- `experiments/dl1_xtts_ft/speaker2/...`

### Missing Data for Training/Reproduction

The submitted repo does not include the full ASVspoof 2019 LA/PA datasets.

Present only in server copy:

- `LA/LA/...`
- `PA/PA/...`
- `data/precomputed_tensors/...`
- DL1 speaker recordings and generated outputs
- processed red-team pipeline outputs

This is acceptable for GitHub if documented, but training and reproduction require download/setup instructions.

### Missing Configuration and Deployment Files

The submitted repo currently lacks production deployment basics:

- `.env.example`
- Dockerfile
- docker-compose file
- production start command
- healthcheck definition for deployment
- model download/setup script
- deployment-specific README
- CI/CD workflow
- `.gitignore` rules for models, datasets, uploads, outputs, caches, and checkpoints
- tests

### Broken or Risky Code Paths

1. `src/train.py` in the submitted repo imports `src.data.dataset` and `balanced_sampler_from_manifest`, but `src/data` is missing.
2. `src/models/factory.py` imports `src.models.cnn_baseline`, but `cnn_baseline.py` is missing.
3. `src/models/factory.py` refers to `src.models.audio_mamba`, but no `audio_mamba.py` was found.
4. The README claims `webapp/HOSTING.md` exists, but no such file is present in the submitted repo.
5. The web app defaults to XTTS model paths under `experiments/...`, but that folder is not present in the submitted repo.
6. The web app defaults to AASIST checkpoint files in the repo root, but those files are not present in the submitted repo.
7. Uploaded/generated/job/classification state is stored in memory only.
8. `webapp/static/index.html` loads Lucide from a CDN; fully offline deployment would fail icons unless bundled.
9. `webapp/requirements.txt` pins `numpy==2.4.4`, which may not be compatible with all audio/ML dependencies.
10. The app has no authentication, rate limiting, abuse controls, persistent storage, queueing, or background job tracking.
11. The classification endpoint requires both AASIST models to load; a missing second checkpoint makes detection unavailable.
12. Generated audio is stored locally and cleaned by age only; this is fragile on ephemeral hosting.

### Server Copy Quality Issues

The server copy includes many non-repository artifacts that should not be committed:

- Python/pip caches: `.pip-cache`, `.pip-tmp`, `.tmp`
- rclone binaries and zip files
- duplicate XTTS checkpoints
- ASVspoof datasets
- precomputed tensor cache
- generated/processed audio
- `__pycache__`
- zero-byte dependency-named files such as `=0.12`, `=1.24`, etc.

### Notebook/Report Drift

The report and notebooks use hardcoded Linux server paths such as:

- `/DATA/Trashaimpms/ekansh`
- `/DATA/llop/LA`
- `/DATA/llop/PA`

These need to be converted into documented config paths or CLI arguments before reproduction.

### Results/Threshold Drift

Different artifacts mention different threshold/checkpoint names:

- Web app defaults: Model A threshold `0.420`, Model B threshold `0.118`.
- Some server reports reference thresholds `0.626` and `0.500`.
- Report table states thresholds are validation-derived at EER operating points.

Production docs must clearly specify which checkpoints and thresholds are the production pair.

## Assets to Recover from Server Copy

Required for detection-only production:

- `ModelA_LA_bestnew.pt`
- `ModelB_PA_bestnew.pt`
- `webapp/aasist_classifier.py`
- `webapp/main.py`
- `webapp/static/*`

Required for generation production:

- One selected XTTS model directory containing:
  - `best_model.pth`
  - `config.json`
  - `vocab.json`
- `webapp/tts_engine.py`

Required for training/reproducibility:

- `src/data/*`
- `src/models/cnn_baseline.py`
- selected training/evaluation scripts
- manifests
- README dataset setup instructions

Optional research artifacts:

- `src/improved/*`
- `reports/*.csv`
- `reports/*.json`
- `explainability/*` summaries and selected generated figures

## Recommended Resolution

Do not copy the full 59.76 GB server folder into GitHub. Reconstruct a clean repo:

- Keep source, configs, docs, tests, Docker, and small reports.
- Store large model files outside Git, preferably via Git LFS, Hugging Face model repo, S3, GCS, or a release artifact.
- Exclude datasets, generated audio, caches, duplicate checkpoints, and local server tooling.
- Add scripts that download or place required model files into `models/checkpoints/`.

