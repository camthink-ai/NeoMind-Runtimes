# Jetson runtime (linux-aarch64, sm_87, CUDA)

Our gcc-11 CUDA build of llama-server for Orin-family Jetsons. Official
llama.cpp has no linux-aarch64 CUDA asset, and its `ubuntu-arm64` CPU build
requires gcc-13 libstdc++ (GLIBCXX_3.4.32) — JetPack 6 ships gcc-11 and
fails to exec. This build closes that gap.

Release asset (flat, per llama.cpp version, under tag `runtime-<tag>`):
`llama-server-<tag>-linux-aarch64-jetson.tar.gz`

## b10545 — on-device built (verified 2026-08-24)

- Board: Orin Nano 8GB (JP 6.2.1, CUDA 12.6.68, gcc 11.4, MAXN_SUPER)
- Config: `-DGGML_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES=87
  -DCMAKE_BUILD_WITH_INSTALL_RPATH=ON -DCMAKE_INSTALL_RPATH='$ORIGIN'`
- Built `llama-server` only, `-j4`, 22 min.
- SHA-256: `4ddbb55d0ebf4a12dfe16627381ad772626f120d85ece915d5c2054ded9ef8ef`
- Smoke: LFM2.5 QAD 36.5 tok/s gen / 1262 tok/s ingest; Qwen3.5-4B @32K
  17.8/538; Gemma4-E2B QAT 34.8/1197. GPU-bound, bandwidth-limited.
- Whole `bin/` tree must ship together (thin wrapper + sibling `.so`s);
  `$ORIGIN` RUNPATH keeps it working from any working directory.

## Verify before promoting a new build

Run on a real Jetson: extract → `./llama-server --version` → health →
one generation. Only then bump the NeoMind `JETSON_RUNTIME_SHA256` pin.
