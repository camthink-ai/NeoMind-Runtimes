# NeoMind Runtimes

Binary runtime assets + the central model catalog for [NeoMind](https://github.com/camthink-ai/NeoMind).

This repo exists because some runtimes must ship as **our own builds** (the
official llama.cpp prebuilts don't cover them), and because the model catalog
deserves a versioned home that isn't the product repo.

## Layout

```
├── llama-server-<tag>-linux-aarch64-jetson.tar.gz   ← release assets (see below)
├── llama-server-<tag>-linux-x64-cuda.tar.gz
├── models/catalog.json                              ← model directory manifest
└── .github/workflows/build-*-runtime.yml            ← rebuild pipelines
```

## Releases — one tag per llama.cpp version

Each release tag is named `runtime-<llama.cpp-tag>` and carries the runtime
binaries for that version:

| Asset | Built with | Notes |
|---|---|---|
| `llama-server-<tag>-linux-aarch64-jetson.tar.gz` | gcc-11 + CUDA 12.6, `sm_87` | Orin-family. Official llama.cpp has **no** linux-aarch64 CUDA asset, and its `ubuntu-arm64` CPU build needs gcc-13 libstdc++ (GLIBCXX_3.4.32) — JetPack 6 ships gcc-11 and fails to exec. This is our CUDA build for that gap. |
| `llama-server-<tag>-linux-x64-cuda.tar.gz` | gcc-11 + CUDA 12.4, `sm_75/80/86/89` | NVIDIA Linux servers (T4, A100, A10, RTX 30xx, L40S, RTX 40xx). llama.cpp's OFFICIAL ubuntu-cuda builds moved to a glibc-2.38 base (Ubuntu 24.04) — unusable on the Ubuntu-22.04 floor NeoMind supports — so we self-host this gcc-11/22.04 build instead. CUDA runtime libs (cudart/cublas/cublasLt) bundled with `$ORIGIN` RUNPATH: hosts need only an NVIDIA driver (≥525), no toolkit. NeoMind **auto-detects** qualifying NVIDIA GPUs (via `nvidia-smi`: compute cap 7.5–8.9, driver ≥525) and uses this build with zero configuration; `NEOMIND_BUILTIN_RUNTIME_VARIANT` forces either way. |

### Building (CI)

`.github/workflows/build-jetson-runtime.yml` is a `workflow_dispatch` with a
`LLAMA_CPP_TAG` input: on `ubuntu-22.04-arm`, installs the CUDA 12.6 aarch64
toolkit, configures with `-DGGML_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES=87
-DCMAKE_BUILD_WITH_INSTALL_RPATH=ON -DCMAKE_INSTALL_RPATH=$ORIGIN
-DGGML_NATIVE=OFF -DGGML_CPU_ARM_ARCH=armv8.2-a`, builds only `llama-server`,
then packages the whole `bin/` tree (the server is a thin wrapper that
dlopens sibling `.so`s — a single-file extract dies on exec). The
`GGML_NATIVE=OFF` pin is mandatory: the arm64 runners are Neoverse (ARMv9,
SVE) and a native CPU backend SIGILLs on every Orin (Cortex-A78AE, ARMv8.2,
no SVE) — the first CI-built b11609 asset shipped exactly that bug and was
rebuilt on-device 2026-10-14.

`.github/workflows/build-linux-cuda-runtime.yml` is the x64 sibling: on
`ubuntu-22.04`, CUDA 12.4 toolkit, `-DCMAKE_CUDA_ARCHITECTURES="75;80;86;89"`,
same `$ORIGIN` packaging **plus** the CUDA runtime libs the binaries link
(ldd-driven, so driver-only hosts work), and it uploads to an existing
`runtime-<tag>` release when one is already there.

The first Jetson release (b10545) was **built on-device** on a real Orin Nano
8GB (JP 6.2.1, CUDA 12.6) — best provenance; the CI path reproduces the same
recipe for future bumps. The first linux-x64 CUDA release (b10545) was
**built on a real Tesla T4 host** (Ubuntu 22.04, CUDA 12.4 toolkit, driver
575) following the same recipe. The repo's release quality gate is a
real-hardware smoke (run llama-server + a generation) before promoting.

### Adding a new version

1. Run the workflow with the new `LLAMA_CPP_TAG`, or build on-device
   (`CUDA=1 scripts/build-llama-server.sh` in the NeoMind repo).
2. Smoke the artifact on a real device of its class (Jetson / NVIDIA server).
3. `gh release create runtime-<tag> <assets…> --repo camthink-ai/NeoMind-Runtimes`
   (or `gh release upload runtime-<tag> <asset> --clobber` if the tag exists).
4. In the NeoMind repo: bump `LLAMA_CPP_VERSION` + the asset's SHA pin
   (`JETSON_RUNTIME_SHA256` / `LINUX_CUDA_RUNTIME_SHA256`).

## Model catalog (`models/catalog.json`)

The central, versioned directory of models NeoMind can offer. NeoMind fetches
this to render the picker; **the model BYTES stay on Hugging Face** (GitHub
Releases caps assets at 2GB — most models exceed it). Add a model = add a
JSON entry, no product release needed.

Schema (mirrors `crates/neomind-core/src/builtin_llm/manifest.rs`):

```jsonc
{
  "id": "qwen3.5-4b",                  // stable id used by NeoMind
  "name": "Qwen3.5-4B",
  "file_name": "qwen3.5-4b-q4_k_m.gguf", // canonical local file
  "sha256": "00fe7986…",                // pin — required
  "quant": "q4_k_m",
  "hf_repo": "unsloth/Qwen3.5-4B-GGUF", // where the bytes live
  "hf_file": "Qwen3.5-4B-Q4_K_M.gguf",  // filename inside hf_repo (may differ from file_name)
  "size_bytes": 2740000000,
  "default_ctx": 32768,
  "default_thinking": false,
  "min_ram_mb": 4096,
  "notes": "…"                          // shown in the picker
}
```

### Catalog versioning

Bump `catalog_version` (top-level field) on each change. NeoMind clients can
pin the version they've validated against; new versions are opt-in upgrades.
