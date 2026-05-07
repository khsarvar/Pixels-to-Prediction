# DL Spring 2026 Final — ScienceQA Visual MCQ Reproducibility Repo

This repo is the cleaned final-submission version of our ScienceQA visual multiple-choice project.

The goal is to avoid the issues from the midterm grading:
- report every LoRA hyperparameter, including target modules and dropout
- avoid hardcoded Kaggle username paths
- provide reproducible training and inference steps
- load model weights offline for Kaggle submission
- include a local validation metric and breakdowns
- separate clean validation ablations from final train+val leaderboard training

---

## Start Here

Final files:

```text
training/final_training_reproducible.ipynb
inference/final_inference_offline.ipynb
requirements.txt
README.md
```

For final Kaggle submission, run:

```text
inference/final_inference_offline.ipynb
```

Expected output:

```text
/kaggle/working/submission.csv
/kaggle/working/inference_manifest.json
```

---

## Offline Kaggle Setup

The final inference notebook must run with Kaggle internet disabled.

Attach three Kaggle datasets:

1. **Competition dataset**
   - must contain `test.csv`
   - should also contain `train.csv`, `val.csv`, and images if available

2. **Offline base model dataset**
   - full Hugging Face snapshot of:
     ```text
     HuggingFaceTB/SmolVLM-500M-Instruct
     ```
   - the folder must contain files like:
     ```text
     config.json
     model.safetensors or model.safetensors.index.json
     preprocessor_config.json
     tokenizer files
     ```

3. **Final adapter dataset**
   - output from the training notebook
   - must contain:
     ```text
     adapter_config.json
     adapter_model.safetensors
     train_manifest.json
     tokenizer/processor files
     ```

The notebook auto-detects these paths. If auto-detection fails, set environment variables:

```bash
DATA_DIR=/kaggle/input/<competition-dataset-folder>
BASE_MODEL_DIR=/kaggle/input/<base-model-dataset-folder>/<snapshot-folder>
ADAPTER_DIR=/kaggle/input/<adapter-dataset-folder>/final_adapter
```

Do not use a path like:

```text
/kaggle/input/datasets/<username>/<dataset-name>/...
```

That was one of the portability problems from the previous submission.

---

## How to Create the Offline Base Model Dataset

Run this once locally or in a temporary online environment:

```bash
pip install huggingface_hub
huggingface-cli download HuggingFaceTB/SmolVLM-500M-Instruct \
  --local-dir smolvlm-500m-instruct \
  --local-dir-use-symlinks False
```

Then upload `smolvlm-500m-instruct/` as a Kaggle dataset.

This step is necessary because `AutoModel.from_pretrained("HuggingFaceTB/SmolVLM-500M-Instruct")` tries to resolve files from Hugging Face unless the model is already local. For the final offline Kaggle notebook, the base model must be attached as a dataset.

---

## Final Best-Run Configuration

| Setting | Value |
|---|---|
| Base model | `HuggingFaceTB/SmolVLM-500M-Instruct` |
| Fine-tuning | QLoRA, 4-bit NF4 |
| LoRA rank | `r=8` |
| LoRA alpha | `32` |
| LoRA scale | `alpha / r = 4` |
| LoRA dropout | `0.05` |
| Target modules | `q_proj`, `k_proj`, `v_proj`, `o_proj`, `gate_proj`, `up_proj`, `down_proj` |
| Learning rate | `2e-4` |
| Optimizer | AdamW |
| Weight decay | `0.01` |
| Epochs | `3` |
| Batch size | `1` |
| Gradient accumulation | `8` |
| Effective batch size | `8` |
| Image size | `224 x 224` |
| Text cap | `400` chars each for lecture and hint |
| Loss | final answer-letter token only |
| Seed | `42` |
| Final training data | `train + val` |
| Clean ablation mode | train only, evaluate on val |

The notebooks print and save this config to JSON so the report can cite the exact settings.

---

## Training

For the final leaderboard model:

```bash
TRAIN_MODE=final_train_val
```

Run:

```text
training/final_training_reproducible.ipynb
```

Outputs:

```text
/kaggle/working/final_adapter/
/kaggle/working/final_adapter.zip
/kaggle/working/submission.csv
```

For clean validation ablations:

```bash
TRAIN_MODE=train_only_eval
```

This trains only on `train.csv` and evaluates on `val.csv`.

Important reporting note: the final best model may train on `train + val`, but validation accuracy from that model should be labeled as a diagnostic only, not a clean ablation.

---

## Inference

Run:

```text
inference/final_inference_offline.ipynb
```

It will:

1. find `test.csv`
2. find the offline base model snapshot
3. find the LoRA adapter
4. load everything with `local_files_only=True`
5. run a small validation sanity check if `val.csv` is available
6. write `submission.csv`

To run full validation in the inference notebook:

```bash
RUN_FULL_VAL=1
```

---

## Repository Structure

```text
.
├── README.md
├── requirements.txt
├── .gitignore
├── training/
│   └── final_training_reproducible.ipynb
└── inference/
    └── final_inference_offline.ipynb
```

Do not commit raw Kaggle data, model weights, zip outputs, or notebook checkpoints.

---

## Reproducibility Checklist for Final Report

Include these in the ACL report:

- exact base model name and parameter size
- LoRA rank, alpha, dropout, target modules
- trainable parameter count from `model.print_trainable_parameters()`
- optimizer, LR, epochs, batch size, gradient accumulation, image size
- whether the run used train-only or train+val
- validation accuracy and breakdowns from clean train-only runs
- public leaderboard score for the final train+val model
- limitations: final train+val validation is not a clean metric
- GitHub link
- model weights link
- AI tooling disclosure
