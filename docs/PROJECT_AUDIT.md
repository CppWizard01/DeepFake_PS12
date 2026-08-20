# VoiceLab Project Audit

Status: Phase 1 evidence audit completed before deployment.

## Source Locations Audited

- Submitted GitHub working copy: `C:\Users\EKANSH SINGAL\Downloads\Compressed\voice lab\DeepFake_PS12`
- Original remote-server copy: `C:\Users\EKANSH SINGAL\Downloads\Compressed\ekansh_60gb\ekansh`

## High-Level Project Goal

VoiceLab is a deepfake-audio generation and detection project. The intended system combines:

- A FastAPI web application for audio upload, text input, voice cloning, spoof classification, result history, health checks, and file cleanup.
- A voice-cloning workflow based on XTTS, using uploaded speaker reference audio and target text.
- An AASIST-style deepfake detector using 16 kHz mono waveform preprocessing, dual checkpoints, fixed thresholds, and classification history.
- Research/training workflows for ASVspoof 2019 LA/PA anti-spoofing, XTTS generation, speaker similarity analysis, noisy stress testing, and explainability.

## Repository State Summary

The submitted repository is incomplete relative to the server copy.

- Submitted repo: 149 files, about 0.07 GB.
- Server copy: 425,519 files, about 59.76 GB.
- The submitted repo contains the FastAPI app, AASIST classifier wrapper, notebooks, reports, manifests, RawNet/SpecRNet training code, and frontend assets.
- The server copy contains additional training modules, datasets, generated artifacts, model checkpoints, XTTS fine-tuning experiments, explainability outputs, and report CSV/JSON outputs.

## Architecture Observed

### Web Application

`webapp/main.py` defines a FastAPI app with these major endpoints:

- `GET /`: serves the static frontend.
- `POST /upload-voice`: accepts `.wav`/`.mp3`, converts to 16 kHz mono WAV, and stores metadata in memory.
- `POST /upload-text`: accepts UTF-8 `.txt`, stores text metadata in memory.
- `POST /generate`: invokes `TTSEngine` for XTTS voice cloning and optionally a second generated variant.
- `POST /classify`: runs both AASIST detector checkpoints on normalized 16 kHz audio.
- `GET /classify-history`: returns last 50 classification records.
- `GET /audio/{file_id}` and `GET /download/{file_id}`: stream/download generated files.
- `GET /jobs`: returns last 50 generation jobs.
- `DELETE /cleanup`: deletes files older than 24 hours.
- `GET /health`: reports model/classifier load status and GPU memory.

Runtime state is currently in process memory, so job history, upload metadata, text metadata, and classification history disappear on restart.

### Frontend

The frontend is a static HTML/CSS/JS app under `webapp/static/`.

Main views:

- New Generation
- Classifier
- History
- Server Status

It supports drag/drop upload, text input or text-file upload, generation options, waveform display, result playback/download, classifier history, and health polling.

### Deepfake Generation Pipeline

`webapp/tts_engine.py` wraps Coqui XTTS:

- Loads XTTS config, checkpoint, and vocab from either a model directory or explicit paths.
- Selects CUDA if available, otherwise CPU.
- Uses reference audio to compute XTTS conditioning latents and speaker embeddings.
- Synthesizes English speech from text.
- Saves normalized PCM-16 WAV output.

Server-side research scripts add a larger generation pipeline:

- `src/improved/clone_dl1_batch.py`: batch XTTS cloning for 50 DL1 utterances.
- `src/improved/finetune_xtts_dl1_pipeline.py`: builds speaker datasets, fine-tunes XTTS for two speakers, generates outputs, writes manifests, and runs post-processing.
- `src/improved/tts_postprocess_pipeline.py`: stage pipeline for TTS ingest/generation, RIR convolution, DAC codec wash, band-limited noise, resampling, and MP3 passthrough.
- `src/improved/localhost_tts_web.py`: separate local web wrapper around a command-template TTS pipeline and post-processing chain.

### Deepfake Detection Pipeline

The production web detector is in `webapp/aasist_classifier.py`.

Inference preprocessing:

- Load audio with `torchaudio`.
- Convert multi-channel audio to mono.
- Resample to 16 kHz.
- Apply pre-emphasis.
- Apply energy-based VAD.
- Apply RMS normalization to about -23 dB.
- Pad/crop to 64,600 samples.

Model architecture:

- SincConv frontend.
- Residual 1D convolutional encoder.
- Graph attention layer/HSGAL-like pooling.
- Fully connected 2-class output.

Web app loads two checkpoints by default:

- `ModelA_LA_bestnew.pt`
- `ModelB_PA_bestnew.pt`

Default thresholds:

- Model A: `0.420`
- Model B: `0.118`

These files are missing from the submitted repo but present in the server copy.

### Training Workflows

Submitted training CLI:

- `src/train.py`: trains anti-spoof models, saves `last.pt`, `best.pt`, history, plots, and summaries.
- `src/evaluate.py`: evaluates checkpoints and writes metrics, ROC, and DET plots.
- `src/benchmark.py`: estimates MACs, parameters, latency, and checkpoint size.

However, submitted training code is not runnable as-is because it imports missing modules (`src.data.*`, `src.models.cnn_baseline`, optionally `src.models.audio_mamba`).

Server copy contains the missing or older training pieces:

- `src/data/dataset.py`
- `src/data/make_manifests.py`
- `src/data/augmentation.py`
- `src/models/cnn_baseline.py`
- `src/models/cnn_improved.py`
- `src/improved/train_cnn.py`
- `src/improved/evaluate_cnn.py`
- `src/improved/losses.py`

The server copy appears older in some places than the submitted repo. For example, server `src/train.py` only supports `cnn`, `crnn`, and `specrnet`, while the submitted `src/train.py` also refers to `rawnet` and `audiomamba`.

## Report and Notebook Findings

`Report_G27.pdf` describes the intended challenge solution:

- Task 1: binary anti-spoof classifier on ASVspoof 2019 LA/PA.
- Task 2: XTTS voice cloning and ECAPA-TDNN similarity analysis.
- Task 3: stress testing with AWGN and babble noise at 20 dB, 10 dB, and 5 dB SNR.
- Task 4: interactive demo integration.

Key reported AASIST results:

- Model A / Run 1 in-domain EER: 13.89%.
- Model A / Run 1 cross-domain EER: 31.91%.
- Model B / Run 2 in-domain EER: 25.77%.
- Model B / Run 2 cross-domain EER: 36.44%.
- AASIST trainable parameters: 104,666.
- AASIST saved model size: about 0.44 MB.
- AASIST mean latency on RTX A5000: about 3.85-3.88 ms for a 5-second input.

Task 2 report:

- XTTS used for voice cloning.
- 2 speakers, 25 sentences each.
- 50 real + 50 synthetic samples.
- ECAPA-TDNN mean cosine similarity:
  - Speaker 1: 0.4779
  - Speaker 2: 0.5401
  - Overall: 0.5090
- Attack success:
  - Model A: EER 20%, attack success 40%.
  - Model B: EER 52%, attack success 64%.

Task 3 noisy stress testing:

- Model A was robust to AWGN in this small test set but degraded under babble.
- Model B degraded significantly under both AWGN and babble.

## Existing Evaluation Artifacts

Submitted repo includes RawNet/SpecRNet reports. Example benchmark evidence:

- RawNet Run 2:
  - 1,138,666 parameters
  - 8.20B MACs
  - 13.13 MB checkpoint
  - 2.22 ms mean latency on RTX A5000
- SpecRNet Run 2:
  - 804,771 parameters
  - 536.37M MACs
  - 3.11 MB checkpoint
  - 1.61 ms mean latency on RTX A5000

Server reports include:

- `reports/question2_part2_attack_metrics_summary.csv`
- `reports/question3_noisy_metrics.csv`
- `reports/aasist_actual_test_dual_model_summary.json`
- `reports/aasist_dual_model_eval_summary.json`

## Major Audit Conclusion

The project should not be deployed directly from the submitted repository. It needs a controlled reconstruction from both local sources:

1. Recover missing source modules from the server copy.
2. Decide which model assets are deployable.
3. Separate production inference code from research/training artifacts.
4. Add environment configuration, Docker, and deployment docs.
5. Avoid bundling datasets, tensor caches, generated outputs, and duplicate XTTS weights into the application image.

