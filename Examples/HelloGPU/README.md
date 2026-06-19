# HelloGPU

Verifies that the Jetson's GPU is accessible and working from inside a container.

Runs a CUDA availability check, a matrix multiplication benchmark (2048×2048), and a neural network forward pass, then exits with status 0 on success. Useful for confirming the `gpu` entitlement is correctly set up before writing your own GPU code.

## Deploy

```sh
git clone https://github.com/wendylabsinc/wendy
cd wendy/Examples/HelloGPU
wendy run
```

## Expected output

```
GPU Availability Check
======================
CUDA Available: True
Number of GPUs: 1
GPU 0:
  Name: Orin
  Memory Total: 62.72 GB
  Compute Capability: 8.7

Matrix Computation Test (Size: 2048x2048)
  Average Time per Iteration: 0.0021 seconds
  ✓ Computation verified correct!

Neural Network Test
  ✓ Neural network test passed!

✓ All GPU tests passed successfully!
```

If you see `CUDA Available: False`, the `gpu` entitlement is missing from `wendy.json`. See [Troubleshooting](../../debugging/README.md#gpu-not-available-inside-the-container).

## See also

- [HelloLLM](../HelloLLM/README.md) — runs a language model with a web UI
- [GPU entitlement](../../apps/wendy.json.md#gpu)
