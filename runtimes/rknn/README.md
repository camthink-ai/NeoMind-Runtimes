# RKNN runtime (Rockchip RKNPU3 / RK182x) — shelved

Status 2026-10: investigated, then shelved pending a product decision on the
board's distribution form factor. No asset ships here. This note records what
the investigation established so a revival doesn't re-derive it — including
that the ORIGINAL placeholder plan ("a future llama.cpp build targeting
Rockchip NPUs") was wrong.

## Why there is no "RK llama.cpp" to fetch or build

- Upstream llama.cpp has no RKNN backend — ggml/src ships cuda, metal,
  vulkan, hip, sycl, cann (Ascend), hexagon (Snapdragon), musa, … no Rockchip.
- airockchip maintains no llama.cpp fork; their LLM path is the RKLLM stack.
- Community `invisiofficial/rk-llama.cpp` (MIT) wraps RKLLM as a ggml backend
  on its `rknpu2` branch only (gen-2 chips, no prebuilt releases) and still
  needs .rkllm models underneath. Reference material, not a product base.

## The viable path, already validated on the eval board

Board: RK3572 (Buildroot, BusyBox, kernel 6.12). Eval notes live in the
sibling repo `CamThink Project/laya-eval/` (AGENT_CAPABILITY_PLAN.md,
MODEL_SELECTION_AND_TRAINING_PLAN.md).

- Engine: **rkllm3-server** — OpenAI-compatible, runs on-board
  (MiniCPM-V-4.6 24.8 tok/s). Tool-call format shim exists, dev-grade:
  `NeoMind-Extensions/extensions/video-vlm/tools/rkllm_shim.py`.
- Conversion: airockchip `rknn3-toolkit` 1.1.0 + `rknn3-model-zoo`
  (Apache-2.0; zoo covers LFM2_5, MiniCPM_V_4, Qwen2_5_VL, Nanbeige4_2 …
  — aligned with our catalog picks).
- Known toolchain bug: Qwen3.5-VL 0.8B w4a16 export is numerically corrupt
  (immediate EOS; evidence reported to Rockchip). Never promote a converted
  model without an output-diff gate against the base model.
- Models are chip-specific multi-file packages (llm.rkllm + vision.rknn +
  embed.bin + tokenizer): catalog.json needs format/target_hardware fields
  and NeoMind's `manifest.rs` schema must move in lockstep.

## Gates to close before any build work here

1. **Redistribution**: the RKNN SDK License (rknn3-toolkit, rkllm runtime)
   grants redistributing only YOUR modifications/derivatives, for dev and
   test. Hosting Rockchip runtime binaries in our public Releases needs
   written permission from Rockchip. The zoo's Apache-2.0 covers scripts,
   not runtime binaries.
2. **Form factor**: Buildroot vendor images (-dirty kernels, tight
   rkllm ↔ NPU-driver version coupling) mean a self-hosted tarball must be
   built against one pinned image's sysroot. Only sane for an appliance we
   ship end-to-end; "any RK board, one binary" (the Jetson property) is not
   achievable on this stack.

Cost anchor from the 2026-10 scoping: near-zero-code restart via
custom-backend config + catalog entries; full auto-detect / auto-download /
shim productization was scoped at ~3–4 engineer-weeks in NeoMind core.
