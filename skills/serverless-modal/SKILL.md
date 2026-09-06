---
name: serverless-modal
description: "Prepare and run GPU workloads on Modal when Modal is the selected compute backend, including training, inference, and batch jobs. Estimate current costs and use the authorized resource budget."
argument-hint: "[task-description]"
allowed-tools: Bash(*), Read, Grep, Glob, Edit, Write
---

# Modal Cloud GPU — Training & Inference

Apply [ARIS task scope and run limits](../shared-references/effort-contract.md#task-scope-and-run-limits) when interpreting defaults, checkpoints, and downstream calls.

Task: $ARGUMENTS

## Overview

**Modal** is a serverless GPU cloud. Key advantages over SSH-based platforms (vast.ai, remote servers):
- **Python-defined runtime**: declare the container image, resources, and dependencies without managing an SSH server.
- **Scale-to-zero**: containers can scale down after work ends. Include startup, configured minimum containers, idle/scaledown time, storage, and other billed resources in estimates.
- **Local authoring, remote execution**: `modal run` packages selected source and inputs for Modal. Results return through function values, logs, or explicit volume downloads; local data does not automatically remain local.
- **Reproducible environments**: dependencies declared in code via `modal.Image`, not system-level packages.
  Treat the `modal.Image` chain as the RENDERED form of the declarative env spec in
  `../shared-references/compute-env-contract.md` — same spec fields (base, ordered
  pip phases, env vars, smoke probes), same `env:<name>@<specHash>` ledger entry in
  `.aris/compute/modal.md`, same three-tier validation before a long run.

## Setup and current cost estimate

Use the installed Modal SDK and account when available. If setup is requested, install Modal in the project's environment and run `modal setup` through its supported sign-in flow. A local package check needs no cloud job:

```bash
python3 -c 'import modal; print(modal.__version__)'
```

Read current [Modal pricing](https://modal.com/pricing) and the account's actual credits/budget before estimating a new workload. Do not assume a particular free-credit amount, GPU price, or speed from this skill. Never add a payment method as an automatic setup step.

Before a new paid run, prepare the launcher and estimate GPU count × runtime × current rate, plus relevant CPU/RAM, storage, transfer, and idle costs. State the runtime and cost limit. Continue within an already authorized budget; request the concrete spending decision only when it is unresolved or the estimate would exceed that budget. “Need a GPU” alone does not select Modal or authorize a rental.

For memory sizing, estimate weights, activations, optimizer state, KV cache, and batch/sequence dimensions for the actual workload. Benchmark a representative small sample only within the authorized budget; measured throughput is preferable to a static GPU speed table.

```text
Cost estimate (Modal):
  Workload and model: ...
  GPU type/count and expected peak VRAM: ...
  Current resource rate and source date: ...
  Expected runtime / enforced timeout: ...
  Estimated total / authorized remaining budget: ...
```

## Workflow

### Step 1: Analyze Task → Estimate Cost → Choose GPU

Determine workload-specific VRAM, choose a suitable available GPU, and calculate the estimate above. Treat the table below as a rough inference-memory starting point; training and long contexts can require substantially more memory.

**VRAM Rules of Thumb:**
| Model Size | FP16 VRAM | Recommended GPU |
|---|---|---|
| ≤3B | ~8GB | T4, L4 |
| 7-8B | ~22GB | L4, A10, A100-40GB |
| 13B | ~30GB | L40S, A100-40GB |
| 30B | ~65GB | A100-80GB, H100 |
| 70B | ~140GB | H100:2, H200 |

### Step 2: Generate Modal Launcher

Choose only the requested pattern. Patterns B and C expose a service and require authorization for that deployment and its access model; a one-shot experiment does not require an endpoint. Include only the reviewed source/data needed for the run. The examples use the current image-based file inclusion API from the [Modal 1.0 migration guide](https://modal.com/docs/guide/modal-1-0-migration). Adjust the `src` directory to the actual project and preserve the project's dependency constraints.

#### Pattern A: One-Shot GPU Function (training, evaluation, benchmark)

The most common pattern for `run-experiment` integration. Wraps an existing training script:

```python
import modal

app = modal.App("experiment-name")
# One .pip_install() call per SPEC PHASE (chained calls install in order, so a
# pinned torch in the first call can't be dragged by packages in the second —
# the rendered form of compute-env-contract.md's ordered pip_phases):
image = (
    modal.Image.debian_slim(python_version="3.11")
    .pip_install("torch")                                        # phase 1: pins
    .pip_install("transformers", "accelerate", "datasets", "wandb")  # phase 2
    .add_local_dir("src", remote_path="/workspace")  # reviewed project source only
)

# Persistent volume for checkpoints and results
volume = modal.Volume.from_name("experiment-results", create_if_missing=True)

@app.function(
    image=image,
    gpu="A100-80GB",          # Chosen based on Step 1 analysis
    volumes={"/results": volume},
    timeout=3600 * 6,         # 6 hours max
    secrets=[modal.Secret.from_name("wandb-secret")],  # Optional
)
def train():
    import subprocess
    subprocess.run(
        ["python", "train.py", "--output_dir", "/results/run_001"],
        cwd="/workspace",
        check=True,
    )
    volume.commit()  # Persist results to volume

@app.local_entrypoint()
def main():
    train.remote()
    print("Training complete. Results saved to Modal volume 'experiment-results'.")
```

Run: `modal run launcher.py`

#### Pattern B: Web API (persistent inference service)

```python
import modal

app = modal.App("inference-api")
image = (
    modal.Image.debian_slim(python_version="3.11")
    .pip_install("torch")                          # phase 1: pins
    .pip_install("transformers", "accelerate", "fastapi[standard]")  # phase 2
)

@app.cls(image=image, gpu="L40S")
@modal.concurrent(max_inputs=10)
class InferenceAPI:
    @modal.enter()
    def load_model(self):
        from transformers import AutoModelForCausalLM, AutoTokenizer
        self.tokenizer = AutoTokenizer.from_pretrained("meta-llama/Llama-3.2-1B")
        self.model = AutoModelForCausalLM.from_pretrained(
            "meta-llama/Llama-3.2-1B", device_map="auto"
        )

    @modal.fastapi_endpoint(method="POST")
    def generate(self, request: dict):
        inputs = self.tokenizer(request.get("prompt", ""), return_tensors="pt").to("cuda")
        outputs = self.model.generate(**inputs, max_new_tokens=256)
        return {"text": self.tokenizer.decode(outputs[0], skip_special_tokens=True)}
```

Deploy: `modal deploy app.py`

#### Pattern C: vLLM High-Performance Inference

```python
import modal, subprocess

app = modal.App("vllm-server")
image = modal.Image.debian_slim(python_version="3.11").pip_install("vllm")
VOLUME = modal.Volume.from_name("model-cache", create_if_missing=True)
MODEL = "Qwen/Qwen3-4B"

@app.function(image=image, gpu="H100", volumes={"/models": VOLUME}, timeout=3600)
@modal.concurrent(max_inputs=100)
@modal.web_server(port=8000)
def serve():
    subprocess.Popen(["python", "-m", "vllm.entrypoints.openai.api_server",
                      "--model", MODEL, "--download-dir", "/models", "--port", "8000"])
```

#### Pattern D: Batch Parallel (map over dataset)

```python
@app.function(image=image, gpu="T4", timeout=600)
def process_item(item: dict) -> dict:
    # ... process one item ...
    return {"result": "processed"}

@app.local_entrypoint()
def main():
    results = list(process_item.map([{"id": i} for i in range(1000)]))
```

#### Pattern E: LoRA Fine-Tuning

```python
@app.function(
    image=image, gpu="A100-80GB", volumes={"/output": volume},
    timeout=3600 * 6, secrets=[modal.Secret.from_name("huggingface-secret")],
)
def train():
    # ... transformers + peft + trl training code ...
    trainer.save_model("/output/final")
    volume.commit()
```

#### Pattern F: Multi-GPU Distributed Training

```python
@app.function(image=image, gpu="H100:4", volumes={"/output": volume}, timeout=3600 * 12)
def train_distributed():
    import subprocess
    subprocess.run(["accelerate", "launch", "--num_processes", "4",
                    "--mixed_precision", "bf16", "train.py"], check=True)
```

### Step 3: Run

```bash
modal run launcher.py     # One-shot execution (most common for experiments)
modal deploy app.py       # Persistent service deployment
```

### Step 4: Verify & Monitor

```bash
modal app list            # List running apps
modal app logs <app-name> # Stream logs
```

### Step 5: Collect Results

Results collection depends on the pattern used:

**Volume-based** (recommended for training):
```python
# Download results from volume after run completes
# Download explicit result paths after the run
modal volume ls experiment-results
modal volume get experiment-results /run_001/results.json ./results/
```

**Stdout/return-based** (for evaluation/benchmarks):
Results are printed to terminal or returned from the function — already local.

### Step 6: Cleanup

Verify that the launched run/service ended as intended and collect the requested results. Stop a persistent service only when that cleanup is authorized for the identified app. Preserve volumes and checkpoints; deletion is a separate destructive operation and is not implied by a successful run.

## CLI Reference

```bash
modal run app.py          # Run once
modal deploy app.py       # Deploy persistent service
modal app logs <app>      # View logs
modal app list            # List apps
modal app stop <app>      # Stop
modal volume ls           # List volumes
modal volume get <vol> <remote> <local>  # Download from volume
modal secret create NAME KEY=VALUE       # Create secret
```

## Key Tips

- GPU fallback: `gpu=["H100", "A100-80GB", "L40S"]` — Modal tries each in order
- Multi-GPU: `gpu="H100:4"` (up to 8 GPUs, cost scales linearly)
- Volume: `modal.Volume.from_name("x", create_if_missing=True)` for persistent storage
- `@modal.enter()` loads model once per container | `@modal.concurrent()` for concurrent requests
- Long training: set `timeout=3600 * N` (default is 5 min)
- Local code: `image.add_local_dir("src", remote_path="/workspace")`; inspect included paths before uploading
- W&B integration: `secrets=[modal.Secret.from_name("wandb-secret")]` + `wandb.init()` in your script

## Composing with Other Skills

```
/run-experiment "train model"       <- detects gpu: modal, calls /serverless-modal
  -> /serverless-modal              <- analyzes task, generates launcher, runs
  -> Results returned locally or to Modal Volume
  -> No destroy step needed (auto scale-to-zero)

/serverless-modal                   <- standalone: any Modal GPU workload
/serverless-modal "deploy vLLM"     <- inference service deployment
```

## CLAUDE.md Example

```markdown
## Modal
- gpu: modal                 # tells run-experiment to use Modal serverless
- modal_gpu: A100-80GB       # optional: override GPU selection (default: auto-select)
- modal_timeout: 21600       # optional: max seconds (default: 6 hours)
- modal_volume: my-results   # optional: named volume for results persistence
```

The local Modal SDK and an authenticated account are required. Use the existing account and runtime configuration.


## Documentation

- Docs: https://modal.com/docs/guide
- GPU: https://modal.com/docs/guide/gpu
- Pricing: https://modal.com/pricing
- Examples: https://modal.com/docs/examples
