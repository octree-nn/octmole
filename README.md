# OctMole: Multiscale 3D Molecular Generation with Hierarchical Spatial Flows

A 3D molecule generative model jointly conducted by researchers from Peking
University and BAAI; the paper is available [here](https://arxiv.org/abs/2610.16586).

OctMole is a two-stage generative model for 3D molecules. The first stage
generates a sparse hierarchical occupancy octree. The second stage predicts an
atom type and a continuous within-cell coordinate for every occupied leaf.

The repository contains the QM9 and GEOM-DRUG training and end-to-end sampling
configurations used by the project. Dense voxel models remain in the source as
a reference implementation, but the supported end-to-end pipeline uses the
octree representation.

The code is partially based on Tianhong Li's [Just image Transformer (JiT) for Pixel-space Diffusion](https://github.com/LTH14/JiT).

## Installation

Please run all commands from the repository root.

OctMole requires Python 3.10 and a CUDA environment compatible with PyTorch
2.11. Dependencies are managed with [uv](https://docs.astral.sh/uv/):

```bash
uv sync
```

## Training from scratch

We train separate models on the QM9 and GEOM-DRUG datasets. Download
`qm9_data.tar.gz` and `drug_data.tar.gz` from the
[Uni-3DAR Hugging Face repository](https://huggingface.co/dptech/Uni-3DAR/tree/main),
extract them, and organize each dataset using the following layout:

```text
DATA_PATH/
├── train.lmdb
└── valid.lmdb
```

Train the occupancy and atom models for QM9:

```bash
DATA_PATH=xxx bash scripts/run_qm9_oct64.sh   # default bsz = 256 * 4 GPUs = 1024
DATA_PATH=xxx bash scripts/run_qm9_feat64.sh  # default bsz = 256 * 4 GPUs = 1024
```

Train the occupancy and atom models for GEOM-DRUG:

```bash
DATA_PATH=xxx bash scripts/run_drug_oct64.sh   # default bsz = 128 * 8 GPUs = 1024
DATA_PATH=xxx bash scripts/run_drug_feat64.sh  # default bsz = 128 * 8 GPUs = 1024
```

You can override the number of visible gpus based on your devices.
The default scripts use a finest octree depth of 6, corresponding to a
`64^3` spatial resolution.

## Inference with pretrained checkpoints

Pretrained checkpoints for QM9 and GEOM-DRUG are available from the
[OctMole checkpoint repository](https://huggingface.co/Disembo/octmole):

| Dataset | Occupancy model (Stage 1) | Atom model (Stage 2) |
|---|---|---|
| QM9 | `qm9-stage1.pth` | `qm9-stage2.pth` |
| GEOM-DRUG | `drug-stage1.pth` | `drug-stage2.pth` |

Generate 10,000 reranked final molecules with the two pretrained checkpoints:

```bash
STAGE1_MODEL=/path/to/qm9-stage1.pth STAGE2_MODEL=/path/to/qm9-stage2.pth bash scripts/sample_qm9.sh
STAGE1_MODEL=/path/to/drug-stage1.pth STAGE2_MODEL=/path/to/drug-stage2.pth bash scripts/sample_drug.sh
```

You can override the number of visible gpus based on your devices.
A sampling output has the following structure:

```text
OUT_DIR/
├── samples_all.xyz
├── samples.xyz
└── metrics.json
```

`samples_all.xyz` contains the candidate pool and `samples.xyz` contains the
reranked final samples.

## Open Babel evaluation

Evaluate the final XYZ file with the default 0.8 Å clash filter:

```bash
uv run python tools/evaluate_openbabel.py \
  --xyz output/qm9_oct64/samples.xyz \
  --reference-stats /path/to/qm9_data/valid_openbabel_stats.json \
  --overwrite
```

The evaluator writes results to `OUT_DIR/openbabel/`. Reference statistics of
the dataset can be built once from a validation LMDB:

```bash
uv run python tools/build_openbabel_stats.py --help
```
