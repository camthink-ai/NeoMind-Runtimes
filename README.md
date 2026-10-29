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

## `runtimes.json` — the runtime update channel

`runtimes.json` at the repo root is the **update channel** the running
NeoMind platform polls (raw.githubusercontent URL; overridable via
`NEOMIND_RUNTIME_CHANNEL_URL` for mirrors/air-gapped installs) to learn
which llama.cpp runtime to recommend for the standalone, in-app runtime
upgrade (内置模型管理 UI → 运行时行 → 升级):

```json
{
  "schema": 1,
  "recommended": "b11609",
  "variants": {
    "jetson":     { "tag": "…", "asset": "…", "sha256": "…" },
    "linux-cuda": { "tag": "…", "asset": "…", "sha256": "…" }
  },
  "notes": { "zh": "…", "en": "…" },
  "min_platform": "0.0.1"
}
```

- `recommended` — the tag for official-asset variants (cpu / vulkan /
  macOS / windows). Only recommend tags whose official ubuntu builds still
  run on the Ubuntu-22.04 / glibc-2.35 / gcc-11 floor (exec-check them).
- `recommended_by_platform` — per-`"<os>/<arch>"` override of
  `recommended` (e.g. `{"linux/aarch64": "b11609"}`). Upstream floors move
  PER-ARCH: the 0.8.x (b11960) official ubuntu-arm64 builds jumped to
  glibc 2.38 (Ubuntu 24.04 base — verified by symbol scan 2026-10-29)
  while ubuntu-x64 stayed at 2.34 — arm64 must hold back while x64 moves
  forward, and a single global `recommended` cannot express that.
- `variants` — per-self-hosted-variant entries. **`sha256` is mandatory**
  there (executable downloads are hard-pinned); entries may lag
  `recommended` until the Runtimes CI builds catch up. Optional `url`
  overrides the download location (mirror); optional `official_base`
  (top level) redirects official-asset downloads.
- `min_platform` — refuse to recommend a runtime to platforms older than
  this semver.

### Bump runbook (keep everything in sync)

1. Bump `LLAMA_CPP_VERSION` in `crates/neomind-core/src/builtin_llm/runtime.rs`
   (NeoMind) — CI's `llama-pin-sync` gate also requires bumping
   `scripts/build-llama-server.sh` + `scripts/install.sh` defaults.
2. Build + publish the `runtime-<tag>` release assets here (Jetson via the
   Runtimes CI workflow with `-DGGML_NATIVE=OFF -DGGML_CPU_ARM_ARCH=armv8.2-a`;
   linux-cuda via the on-device T4 recipe).
3. Backfill the two compiled-in SHAs in the same core file.
4. Update `runtimes.json` (recommended + variant entries + notes) — **after**
   the assets exist, since self-hosted entries are SHA-pinned.
5. Release NeoMind.

Adding a **new variant** (e.g. RKNN — see `runtimes/rknn/`): the
SERVER-BINARY side is cheap — enum variant + asset mapping in the core
file, a channel entry here, a build pipeline; the upgrade/revert/resolution
flow is variant-agnostic (rkllm3-server is OpenAI-compatible). The MODEL
side is not: RKNN models are chip-specific multi-file packages, not GGUF —
`catalog.json` needs `format`/`target_hardware` fields, NeoMind's
`manifest.rs` moves in lockstep, and the GGUF-centric download/import/
spawn-arg paths need per-format arms. Close the redistribution gate in
`runtimes/rknn/README.md` before any of that starts.

## Model catalog contract — `format` & `target_hardware`

Catalog entries may carry two optional fields that vendor model formats
(rkllm today, further hardware vendors later) require. **Start adding them
to new entries now** — consumers parse and expose them already:

```json
{
  "id": "minicpm5-2b-rk3588",
  "format": "rkllm",
  "target_hardware": ["rk3588", "rk3572"],
  "…": "the existing gguf-shaped fields"
}
```

- `format` — OPEN STRING, not an enum: lowercase ascii/digits/hyphen, 2-16
  chars (`gguf` | `rkllm` | a future `dxnn` | …). Absent = `gguf`. A vendor
  that breaks package compatibility versions its string (`rkllm` →
  `rkllm2`) rather than redefining it. The platform treats
  unknown-but-wellformed formats as forward-compatible "unsupported here"
  entries; only malformed strings drop (that entry alone, with a warn).
- `min_runtime` — minimum llama.cpp runtime tag (e.g. `"b11960"`): while
  the active runtime is older, the platform refuses the DOWNLOAD with
  guidance pointing at the runtime upgrade dialog (LFM3 needs >= b11960 —
  older runtimes fail with cryptic arch errors otherwise). Compared with
  numeric b-tag ordering; absent = no floor.
- `target_hardware` — list of SoC/board ids the package runs on (matches
  `runtimes/rknn/README.md`'s chip-specific multi-file packages:
  `llm.rkllm` + `vision.rknn` + `embed.bin` + tokenizer). Absent/empty =
  runs anywhere the format is supported. Host-side eligibility filtering
  ships with the first hardware-specific model.
- Multi-file packages will additionally carry a `files:
  [{name, sha256}]` list — contract slot reserved, do not invent
  single-sha workarounds for such entries.
- **Hosting**: runtime binaries (tens of MB) live on this repo's Releases,
  but model packages do NOT — GitHub caps release assets at 2 GiB and
  catalog GGUFs already run 1.5–4.8 GB. Large packages go to CamThink's own
  CDN (resources.camthink.ai or an S3/R2 bucket); catalog `files` entries
  carry full URLs (the platform's mirror layer already honors them).
- **RK boards may be rkllm-only**: whether the official llama.cpp
  ubuntu-arm64 CPU build runs there depends on the image's libstdc++
  (the Jetson note: official arm64 assets need gcc-13 GLIBCXX_3.4.32).
  Never ASSUME a gguf CPU fallback on a board — `host_formats()` must
  derive from what the runtime layer can actually resolve on that image
  (verify on-device before enabling gguf entries in its filtered list).
- **Converted models ship only through the output-diff gate** (compare
  against the base model's outputs — the Qwen3.5-VL w4a16 export was
  numerically corrupt with immediate EOS; see runtimes/rknn/README.md),
  and expect a model x SoC packaging matrix (per-chip NPU targets), not
  one artifact per model.

NeoMind's `manifest.rs` writes `format` at install time (serde default
`gguf` keeps every pre-format deployed manifest parsing unchanged).
