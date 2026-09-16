# aetron-engine-wheels

Prebuilt `llama-cpp-python` wheels for the AETRON miner fleet.

## Why this exists

`llama-cpp-python` is published on PyPI **as a source distribution only**, so `env install`
compiles the engine on every miner's machine. Two consequences, both of which have already cost
us production incidents:

1. **The backend is decided by the miner's machine, not by us.** An sdist that finds no CUDA
   toolkit builds a CPU-only engine and says nothing about it — the whole Linux fleet ran a 27B
   model on the processor for weeks (2026-09-05), and the same happened silently on MI300X.
2. **The engine revision is whatever the wrapper happens to vendor.** `llama-cpp-python 0.3.35`
   ships llama.cpp `4df29be`; the revision the fleet is calibrated against is a separate
   decision that PyPI cannot express.

The kernels decide the logits, and the logits decide whether an honest miner is judged honest.
So the engine must be a **fixed artifact we build**, not a build step on unknown hardware.

## What is here

Wheels are published as **release assets**, one release per engine revision. Each wheel carries a
local version label — `0.3.35+aetron.v0.4.0` — so `pip` can never satisfy the requirement with
the same-numbered package from PyPI, which contains different kernels.

Wheels are built by `scripts/build-llamacpp.sh` in the `aetron-miner` repository. The build prints
a fingerprint (llama.cpp commit, backend, CMake flags, sha256 of the shared library); those
fingerprints live in `release/engine-builds.json` there and end up in the runner's env gate, which
refuses to start on a library it does not recognise.

## Backends

One backend needs more than one artifact where a single toolchain cannot cover the hardware:
`nvcc` 12.4 cannot target `compute_120`, so Ampere/Ada and Blackwell are separate wheels.

| backend | target | toolkit | asset name |
|---|---|---|---|
| metal | apple-arm64 | Xcode (no Metal toolchain component required) | `…-py3-none-macosx_26_0_arm64.whl` |
| cuda | sm86_89 | CUDA 12.4+ | `…-py3-none-linux_x86_64.whl` |
| cuda | sm120 | CUDA 12.8+ | `…-120-py3-none-linux_x86_64.whl` |
| hip | gfx942 | ROCm 6.4 (built with 6.4.3; ggml needs a 6.x runtime) | `…-0gfx942-py3-none-linux_x86_64.whl` |
| cpu | x86_64 | — (GGML_RPC on, no accelerator kernels) | `…-0cpu-py3-none-linux_x86_64.whl` |

Linux wheels for different hardware share the platform tag, so they are told apart by a PEP 427
build tag (`120`, `0gfx942`, `0cpu`) instead of by name: installing the plain-named linux wheel on
an AMD or CPU-only machine would silently give it the sm_86/89 CUDA kernels.

## Licence

The wheels package [llama.cpp](https://github.com/ggml-org/llama.cpp) (MIT) and
[llama-cpp-python](https://github.com/abetlen/llama-cpp-python) (MIT), with a one-field patch to
the wrapper's ctypes `llama_model_params` so it matches the C ABI of the vendored engine. The
patch is in `scripts/patches/` of the `aetron-miner` repository.
