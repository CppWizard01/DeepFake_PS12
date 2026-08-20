# Model Size and GPU Requirement Report

Status: Phase 3 feasibility analysis.

## Size Inventory

### Repository and Server Copy

- Submitted repo size: about 0.07 GB across 149 files.
- Server copy size: about 59.76 GB across 425,519 files.

### Server Copy Top-Level Size Drivers

| Area | Size |
|---|---:|
| `experiments/` | 24.771 GB |
| `PA/` | 16.551 GB |
| `data/` | 7.737 GB |
| `LA/` | 7.130 GB |
| `processed.zip` | 1.252 GB |
| `processed/` | 1.249 GB |
| `reports/` | 36.87 MB |
| `webapp/` | 24.70 MB |
| `src/` | 0.50 MB |

### Model-Like File Groups

| Group | Count | Size |
|---|---:|---:|
| XTTS experiments | 10 | 24.763 GB |
| Precomputed tensors | 50,224 | 7.597 GB |
| Training models | 2 | 1.43 MB |
| Root/other model files | 3 | 0.89 MB |

### Deployable Checkpoint Candidates

| File | Approx Size | Notes |
|---|---:|---|
| `ModelA_LA_bestnew.pt` | 0.44 MB | AASIST Model A, needed for web classifier |
| `ModelB_PA_bestnew.pt` | 0.44 MB | AASIST Model B, needed for web classifier |
| `models/user_train_gpu32/best.pt` | 0.71 MB | improved CNN-style checkpoint |
| `models/user_train_gpu32_restart/best.pt` | 0.71 MB | improved CNN-style checkpoint |
| XTTS speaker1 `best_model.pth` | 5,348.14 MB | large TTS generation checkpoint |
| XTTS speaker2 `best_model.pth` | 5,348.14 MB | large TTS generation checkpoint |
| XTTS base `model.pth` per speaker copy | 1,781.40 MB | duplicate base XTTS model copy |
| XTTS `dvae.pth` per speaker copy | 200.76 MB | duplicate support weight |

## RAM and Storage Requirements

### Detection-Only

Detection-only deployment is lightweight.

Estimated minimum:

- Storage: 1-2 GB application image is enough if only AASIST checkpoints are bundled.
- RAM: 1-2 GB should run FastAPI plus two AASIST models.
- CPU: practical for occasional inference.
- GPU: not required.

The AASIST checkpoints are tiny and the model has about 104,666 trainable parameters according to the report.

### Full Generation + Detection

Full XTTS generation is heavy.

Estimated minimum:

- Storage: at least 8-12 GB if using one XTTS model directory carefully deduplicated; 25+ GB if copying the current experiment folders as-is.
- RAM: 8-16 GB minimum depending on PyTorch/TTS loading behavior.
- GPU VRAM: 8 GB may be tight; 16 GB is safer; 24 GB was used in the original environment.
- CPU-only generation: possible in principle but likely slow and poor UX.

## GPU Requirement Analysis

### Components That Require or Strongly Prefer GPU

- XTTS voice cloning: strongly prefers GPU.
- XTTS fine-tuning: requires GPU for realistic training time.
- AASIST training: strongly prefers GPU.
- Full ASVspoof evaluation/benchmarking: strongly prefers GPU for speed.
- Explainability scripts with repeated perturbation inference: strongly prefer GPU.

### Components That Can Run on CPU

- FastAPI frontend/backend serving.
- AASIST detection inference for low traffic.
- Audio normalization, VAD, and file conversion.
- Static frontend.
- Report viewing and small batch scripts.

### Expected Latency

Observed report/server evidence:

- AASIST: about 3.85-3.88 ms on RTX A5000 for a 5-second input.
- RawNet: about 2.22 ms on RTX A5000 for a 5-second input.
- SpecRNet: about 1.61 ms on RTX A5000 for a 5-second input.

Estimated CPU latency:

- AASIST detection: likely acceptable for demo traffic, roughly tens to low hundreds of ms per short clip depending on CPU and audio length.
- XTTS generation: likely many seconds to minutes per request on CPU; not recommended for an interactive hosted demo.

## GPU Options

| GPU | Fit |
|---|---|
| T4 16 GB | Good minimum for demo XTTS inference if model loads cleanly; common and relatively cheap. |
| L4 24 GB | Recommended practical GPU for production-style demo; better memory and newer inference performance. |
| A10/A10G 24 GB | Strong choice for full generation plus detection; usually more expensive than T4/L4. |
| A100 40/80 GB | Overkill for this project unless doing training or high-concurrency inference. |

## Cost Estimates

Checked on 2026-08-20. Prices can change, so re-check before launch.

| Option | Estimated Cost | Fit |
|---|---:|---|
| Hugging Face Spaces CPU Basic | Free hardware, paid account/plan may be needed for compute Spaces | Detection-only demo. |
| Hugging Face Spaces CPU Upgrade | $0.03/hr, about $22/month if always running | Better CPU detection demo. |
| Render Hobby web service | Free tier available, sleeps after inactivity | Low-traffic detection-only demo. |
| Render always-on small stack example | About $13/month for a Starter web service plus Basic Postgres, before bandwidth/storage growth | Detection plus persistent history. |
| Hugging Face Spaces T4 small | $0.40/hr, about $292/month if always running | Minimum GPU generation demo. |
| Hugging Face Spaces L4 | $0.80/hr, about $584/month if always running | Recommended XTTS demo GPU. |
| Hugging Face Spaces A10G small | $1.00/hr, about $730/month if always running | Strong XTTS demo GPU. |
| RunPod L4 | From $0.39/hr, about $285/month if always running | Lower-cost GPU worker option. |
| AWS g4dn.xlarge T4 | About $0.526/hr, about $384/month in us-east-1 | Cloud GPU if AWS is required. |
| AWS g6.xlarge L4 | About $0.805/hr, about $588/month in us-east-1 | Cloud GPU with L4 class hardware. |

## Feasibility Conclusion

The cheapest production-ready route is not full XTTS generation on a general web host. Recommended staging:

1. Deploy detection-only first on CPU.
2. Add optional async GPU generation service later.
3. Store XTTS checkpoints externally, not in Git.
4. Use a queue for generation to avoid request timeouts and GPU contention.

## Pricing Sources

- Hugging Face Spaces hardware pricing: https://huggingface.co/docs/hub/en/spaces-gpus
- Render pricing overview and July 2026 small-stack example: https://render.com/articles/how-much-does-cloud-application-hosting-cost-for-small-businesses
- Render 2026 workspace pricing update: https://render.com/docs/new-workspace-plans
- RunPod L4 pricing: https://www.runpod.io/gpu-models/l4
- AWS EC2 price list docs: https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/using-the-aws-price-list-bulk-api-fetching-price-list-files-manually.html
- AWS On-Demand billing model: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-on-demand-instances.html
- AWS g4dn.xlarge and g6.xlarge current price aggregators used for quick us-east-1 estimates:
  - https://www.devzero.io/instances/aws/g4dn.xlarge
  - https://www.devzero.io/instances/aws/g6.xlarge
