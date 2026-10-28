# MoFRR Audited Reproduction

> **Unofficial reproduction / 非官方复现.** This repository is an independent,
> community-maintained reproduction of MoFRR. It is not authored, endorsed, or
> maintained by the original paper authors or their institutions. Results here
> describe our available-assets Tencent experiments, not the authors' official
> benchmark numbers.

This repository reproduces the available MoFRR pipeline with four frozen
specialists—Whitening, Smoothing, Face Lifting and Eye Enlarging—and a
topology-aware Multi router/combiner. Results below use identity-disjoint
Tencent pairs; they are not the paper authors' private RetouchingFFHQ++ test
set. Data, API responses and model checkpoints are intentionally excluded.

## Install and verify

Tested with Python 3.8.20, PyTorch 2.4.1+cu118 and torchvision 0.19.1. Install
the PyTorch build matching your CUDA driver first, then:

```bash
cd Comparsion/MoFRR/MoFRR-master
python -m pip install -r requirements-repro.txt
python -m pytest -q mofrr_repro/tests reproduction_tools/tests
python -m mofrr_repro smoke --config configs/reproduction_smoke.yaml
```

Training needs CUDA. Evidence inspection and most unit tests run on CPU.

## Local paths and external assets

Copy the required YAML before editing it. Set these fields to your own files:

- `manifest`, `output_root`, and optional `face_manifest`;
- `base_config` and each Router/degree/expert/Combine checkpoint path;
- every adjacent `*_sha256` after replacing an artifact;
- `device`, batch size, AMP and GPU-memory limits if needed.

Paths are resolved relative to `MoFRR-master/`, except `base_config`, which is
resolved relative to the YAML file. A paired JSONL row must contain `id`,
`source_id`, `split`, `operations` (W/S/F/E bits), `strength`, `input`,
`target`, `input_sha256`, `target_sha256`, and `target_semantics` set to
`paired_reference_rgb8`. Keep source identities disjoint across
`train/val/test`. No API credentials are needed for cached-pair training.

## Single-specialist training

All runners freeze selection on validation before opening test data:

```bash
# Whitening or Smoothing
python -m reproduction_tools.whitening_experiment prepare --config <yaml>
python -m reproduction_tools.whitening_experiment train-degree --config <yaml>
python -m reproduction_tools.whitening_experiment train-expert --config <yaml>
python -m reproduction_tools.whitening_experiment select --config <yaml>
python -m reproduction_tools.whitening_experiment test --config <yaml>

# Replace whitening_experiment with smoothing_experiment for Smoothing.

# Face
python -m reproduction_tools.face_reproduction prepare --config <yaml>
python -m reproduction_tools.face_reproduction train-degree --config <yaml>
python -m reproduction_tools.face_reproduction train-expert --config <yaml>
python -m reproduction_tools.face_reproduction calibrate --config <yaml>
python -m reproduction_tools.face_reproduction test --config <yaml>

# Eye: train on 30/60/90, then evaluate in-domain and held-out strength 100.
python -m reproduction_tools.eye_degree_generalization prepare --config <yaml>
python -m reproduction_tools.eye_degree_generalization train-eye --config <yaml>
python -m reproduction_tools.eye_degree_generalization calibrate \
  --config <yaml> --checkpoint <eye.pt>
python -m reproduction_tools.eye_degree_generalization evaluate-in-domain \
  --config <yaml> --checkpoint <eye.pt>
python -m reproduction_tools.eye_degree_generalization evaluate-cross-degree \
  --config <yaml> --checkpoint <eye.pt>
```

## Multi training and frozen evaluation

`configs/reproduction_multi_tencent_v1.yaml` binds all external artifacts and
their hashes. The core sequence is:

```bash
python -m reproduction_tools.multi_expert_experiment prepare --config <multi.yaml>
python -m reproduction_tools.multi_expert_experiment cache \
  --config <multi.yaml> --split train
python -m reproduction_tools.multi_expert_experiment cache \
  --config <multi.yaml> --split val
python -m reproduction_tools.multi_expert_experiment train-combine \
  --config <multi.yaml> --train-cache <train-cache> --val-cache <val-cache>

python -m reproduction_tools.multi_embedding_topology_router \
  --config <multi.yaml> --router-audit <router-audit> --cache <val-cache> \
  --checkpoint <combine.pt> --output <topology-dir> \
  --epochs 80 --batch-size 256 --embedding-batch-size 1

python -m reproduction_tools.multi_deployment_selection \
  --config <multi.yaml> --topology-summary <topology-summary.json> \
  --causal-summary <causal-summary.json> --degree-summary <degree-summary.json> \
  --output <deployment.json>

# Opens the sealed test set once; do not use it for tuning.
python -m reproduction_tools.multi_final_evaluation \
  --config <multi.yaml> --deployment <deployment.json> --output <final-dir>
python -m reproduction_tools.multi_final_visualization \
  --config <multi.yaml> --deployment <deployment.json> \
  --final-root <final-dir> --output <gallery-dir>
```

## Reproduced results

Forced-specialist PSNR gain by strength:

| Expert | 30 | 60 | 90 | 100 unseen |
| --- | ---: | ---: | ---: | ---: |
| Whitening | +2.215 | +3.548 | +1.854 | +1.520 |
| Smoothing | +0.353 | +4.644 | +4.897 | +4.729 |
| Face | +0.380 | +1.713 | +1.639 | +1.879 |
| Eye | -0.074 | +0.005 | +0.015 | +0.034 |

Eye is deliberately retained as a negative result. In the frozen 206-image
Multi test, input PSNR/SSIM is 25.7405/.902293; the embedding-topology route
reaches **31.0454/.951696** (**+5.3049 dB**), with 197/206 PSNR wins. Gain is
+3.6687 dB for mask `0011` and +6.9411 dB for `1111`. Router retouched exact
match is 91.21%; identity false activation is 9.45%. Supported topologies are
only `0000/0001/0011/1111`, not all 16 W/S/F/E combinations.

![Aggregate Multi metrics](Comparsion/MoFRR/evidence/public_results/metrics_overview.png)

Machine-readable aggregates are in `Comparsion/MoFRR/evidence/public_results/`;
the detailed Multi protocol is in
`Comparsion/MoFRR/docs/MULTI_EXPERT_REPRODUCTION_V1.md`. Human-face panels
are excluded until dataset and API-output redistribution rights are confirmed;
the visualization command regenerates them locally from authorized data.

## Publication boundary

Never commit datasets, API caches, raw manifests, request IDs, credentials,
checkpoints, local absolute paths or face panels without explicit rights. The
upstream source snapshot has no root license file; confirm redistribution
permission before publishing that third-party source. Cite the ICCV 2025
paper, “MoFRR: Mixture of Diffusion Models for Face Retouching Restoration.”
