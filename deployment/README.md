# VoiceLab Deployment Guide

This guide documents the production-ready path chosen after the audit: deploy detection first on CPU, then add XTTS generation as a GPU-backed service when budget allows.

## Recommended First Deployment

Run the FastAPI app in detection-only mode:

```bash
cp .env.example .env
docker compose up --build
```

Open:

```text
http://127.0.0.1:7860
```

The `/classify` endpoint should be available when both AASIST checkpoints load. The `/generate` endpoint remains disabled unless XTTS is configured.

## Required Environment

- `VOICELAB_ENABLE_TTS=false` for CPU detection-only deployment.
- `AASIST_CHECKPOINT_PATH=models/checkpoints/ModelA_LA_bestnew.pt`
- `AASIST_CHECKPOINT_B_PATH=models/checkpoints/ModelB_PA_bestnew.pt`
- `AASIST_THRESHOLD=0.420`
- `AASIST_THRESHOLD_B=0.118`

## Optional XTTS Generation

XTTS generation should not be hosted on a small CPU instance. To enable it:

```bash
VOICELAB_ENABLE_TTS=true
XTTS_MODEL_PATH=/models/xtts-speaker2
```

The model directory must contain:

- `best_model.pth` or `model.pth`
- `config.json`
- `vocab.json`

For production, store these large model assets in external storage such as Hugging Face model repositories, S3, GCS, or a release artifact. Avoid committing XTTS checkpoints to Git.

## Provider Recommendation

- Cheapest practical demo: CPU container on Render, Railway, Hugging Face Spaces CPU, or a VPS with detection-only mode.
- Full interactive generation demo: Hugging Face Spaces GPU, or a small cloud GPU instance with a queue-backed worker.
- Long-term production: CPU API service plus GPU worker, object storage, persistent job store, and rate limiting.

## Health Check

```bash
curl http://127.0.0.1:7860/health
```

Expected detection-only shape:

```json
{
  "status": "ok",
  "model_loaded": false,
  "classifier_loaded": true
}
```

`model_loaded=false` is expected when XTTS is disabled.

