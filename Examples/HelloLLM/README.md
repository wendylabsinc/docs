# HelloLLM

Run a small language model on your Jetson's GPU and serve it as a web app — no cloud API needed.

The example deploys [DistilGPT2](https://huggingface.co/distilbert/distilgpt2) (82M parameters) using HuggingFace Transformers with Jetson-optimised PyTorch wheels. It serves a web UI at port 8080 where you can enter a prompt and get completions in real time.

## Prerequisites

- Jetson device running WendyOS
- `wendy` CLI installed and device discovered

## Get the code

```sh
git clone https://github.com/wendylabsinc/wendy
cd wendy/Examples/HelloLLM
```

## Deploy

```sh
wendy run
```

`wendy run` detects the `Dockerfile`, builds a container image for `linux/arm64`, pushes it, and starts the app. The browser opens automatically at `http://<device>.local:8080` when the model is ready.

> **First run takes a few minutes.** The Dockerfile installs PyTorch from NVIDIA's Jetson-optimised wheel index and the model weights are downloaded from HuggingFace on startup (~350 MB). Subsequent deploys are fast.

## How it works

```
wendy.json
  └── entitlements: [gpu, network/host]

Dockerfile
  └── installs pytorch==2.8.0 from pypi.jetson-ai-lab.io/jp6/cu126/
      (pre-built for Jetson ARM64 + CUDA 12.6)

app.py
  └── loads distilbert/distilgpt2 onto CUDA
  └── runs a demo (3 sample completions + benchmark)
  └── starts a Flask web server on 0.0.0.0:8080
```

The `gpu` entitlement in `wendy.json` grants the container access to the Jetson's GPU and CDI-mapped CUDA libraries. Without it, `torch.cuda.is_available()` returns `False`.

## Modifying the model

To swap in a different model, change the `model_name` argument in `load_model()`:

```python
model, tokenizer = load_model("distilbert/distilgpt2")   # 82M params, fast
# model, tokenizer = load_model("gpt2")                  # 117M params
# model, tokenizer = load_model("gpt2-medium")           # 345M params, slower
```

Larger models need more GPU memory. The Jetson AGX Orin has 64 GB unified memory; the Jetson Orin Nano has 8 GB.

## Iterating

```sh
wendy watch
```

`wendy watch` rebuilds and redeploys whenever you save a file. Changes to `app.py` that don't affect the Dockerfile layer (e.g. prompt tweaks, new routes) redeploy in under 30 seconds.

## See also

- [GPU entitlement](../../apps/wendy.json.md#gpu) — `wendy.json` reference
- [HelloGPU](../HelloGPU/README.md) — verifies CUDA access with PyTorch matrix multiplication
- [Troubleshooting: GPU not available](../../debugging/README.md#gpu-not-available-inside-the-container)
