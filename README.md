# NeoMind Runtimes

Binary runtime assets + the central model catalog for [NeoMind](https://github.com/camthink-ai/NeoMind).

This repo exists because some runtimes must ship as **our own builds** (the
official llama.cpp prebuilts don't cover them), and because the model catalog
deserves a versioned home that isn't the product repo.

## Layout

```
├── llama-server-<tag>-linux-aarch64-jetson.tar.gz   ← release asset (see below)
├── models/catalog.json                              ← model directory manifest
└── .github/workflows/build-jetson-runtime.yml       ← rebuild pipeline
```

## Releases — one tag per llama.cpp version

Each release tag is named `runtime-<llama.cpp-tag>` and carries the runtime
binaries for that version:

| Asset | Built with | Notes |
|---|---|---|
| `llama-server-<tag>-linux-aarch64-jetson.tar.gz` | gcc-11 + CUDA 12.6, `sm_87` | Orin-family. Official llama.cpp has **no** linux-aarch64 CUDA asset, and its `ubuntu-arm64` CPU build needs gcc-13 libstdc++ (GLIBCXX_3.4.32) — JetPack 6 ships gcc-11 and fails to exec. This is our CUDA build for that gap. |

### Building (CI)

`.github/workflows/build-jetson-runtime.yml` is a `workflow_dispatch` with a
`LLAMA_CPP_TAG` input: on `ubuntu-22.04-arm`, installs the CUDA 12.6 aarch64
toolkit, configures with `-DGGML_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES=87
-DCMAKE_BUILD_WITH_INSTALL_RPATH=ON -DCMAKE_INSTALL_RPATH=$ORIGIN`, builds only
`llama-server`, then packages the whole `bin/` tree (the server is a thin
wrapper that dlopens sibling `.so`s — a single-file extract dies on exec).

The first release (b10545) was **built on-device** on a real Orin Nano 8GB
(JP 6.2.1, CUDA 12.6) — best provenance; the CI path reproduces the same
recipe for future bumps. The repo's release quality gate is a real-Jetson
smoke (run llama-server + a generation) before promoting.

### Adding a new version

1. Run the workflow with the new `LLAMA_CPP_TAG`, or build on-device.
2. Smoke the artifact on a Jetson.
3. `gh release create runtime-<tag> llama-server-<tag>-linux-aarch64-jetson.tar.gz --repo camthink-ai/NeoMind-Runtimes`
4. In the NeoMind repo: bump `LLAMA_CPP_VERSION` + `JETSON_RUNTIME_SHA256`.

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
