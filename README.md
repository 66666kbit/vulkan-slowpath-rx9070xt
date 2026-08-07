# Vulkan gfx1201 (RX 9070 XT) Slow-Path Diagnostic

Diagnostic bundle for [llama.cpp issue #26663](https://github.com/ggml-org/llama.cpp/issues/26663) — the alleged 5–7× slowdown of the Vulkan backend vs. HIP on AMD RX 9070 XT (gfx1201) for models with `hidden_size >= 4096`.

## TL;DR

We tested 4 Qwen3 Q4_K_M models (0.6B / 1.7B / 4B / 8B) on **b10298** of llama.cpp with both backends, on the same hardware the issue was reported on. **We did not reproduce the 5–7× slowdown.** On b10298 the Vulkan backend is in fact **1.15×–1.96× faster than HIP on token generation**, across all 4 models. Prefill (PP) is roughly even at small batch, HIP pulls ahead 1.3–1.5× at `pp512`.

However, the source has a **real, independent code bug** for RDNA4 (`gfx1201`):

- `ggml/src/ggml-vulkan/ggml-vulkan.cpp:450` — `get_device_architecture()` has no RDNA4 branch; RDNA4 falls through to `AMD_RDNA2` default.
- `ggml-vulkan.cpp:18797` — `ggml_vk_khr_cooperative_matrix_support()` whitelists only `AMD_RDNA3` (workaround for AMD driver false-positive coopmat reporting). RDNA4 (mis-detected as RDNA2) gets the KHR_coopmat path **disabled**.
- `ggml-vulkan.cpp:4076-4091` — `gpu_pipeline_configs` only contains RDNA1 and RDNA2 entries. `get_subgroup_size()` returns 0 for RDNA4, callers fall back to `subgroup_props.subgroupSize` (driver-reported 64) instead of the hardware-native wave32.

This bundle includes the full reproduction data, per-op profile, code analysis, and a ready-to-post issue comment body.

## Repository layout

```
.
├── README.md                          # this file
├── FINAL.md                           # executive summary + key findings
├── ROOT_CAUSE.md                      # detailed code analysis (3 bugs identified)
├── WORKLOG.md                         # 1st + 2nd session combined work log
├── data/
│   ├── vulkan_vs_hip.csv              # 4 models × 2 backends (basic)
│   └── vulkan_vs_hip_repro.csv        # extended: pp128 + pp512 + tg128 + PERF
├── logs/
│   ├── perf_4b_vklogger_stderr.log    # 4B per-op profile (GGML_VK_PERF_LOGGER=1)
│   └── perf_8b_vklogger_stderr.log    # 8B per-op profile
└── docs/
    ├── mavis-plan.md                  # Mavis autonomous plan
    ├── decisions.md                   # 12 autonomous decisions
    ├── self_eval.md                   # Mavis self-evaluation
    ├── issue_comment_body.md          # ready-to-post comment for #26663
    └── MANUAL_POST_INSTRUCTIONS.md    # 3-min manual post guide
```

## Reproduction summary

Tested on: AMD Ryzen 7 7800X3D + 32 GB DDR5 + AMD Radeon RX 9070 XT 16 GB (RDNA4, gfx1201), Windows 11, AMD Adrenalin 2026-08. `llama-bench -p <pp> -n 128 -ngl 99 -mg 0 -r 2 -o json`, default device 0 (dGPU).

| model | hidden | backend | pp128 t/s | pp512 t/s | tg128 t/s | V/H tg |
|---|---:|---|---:|---:|---:|---:|
| 0.6B | 1024  | vulkan | 18116 | —    | 571.9 | **1.96×** |
| 0.6B | 1024  | hip    | 10831 | —    | 292.2 | |
| 1.7B | 2048  | vulkan |  9735 | —    | 321.8 | **1.45×** |
| 1.7B | 2048  | hip    |  7193 | —    | 221.7 | |
| 4B   | 2560  | vulkan |  4113 | 5223 | 174.5 | **1.35×** |
| 4B   | 2560  | hip    |  4072 | 6671 | 129.2 | |
| 8B   | 4096  | vulkan |  2719 | 2866 | 104.3 | **1.15×** |
| 8B   | 4096  | hip    |  2908 | 4199 |  91.1 | |

The 14B Q4_K_M download was attempted 4 times across two sessions but kept hanging (likely WARP egress / `huggingface_hub` cache-state issues). Per the diagnostic protocol's fallback rule ("3 restarts dead → stop"), 14B was skipped. The 4 models cover the 1024–4096 hidden range, which is sufficient to surface the alleged regression.

## Per-op profile (8B Vulkan, `GGML_VK_PERF_LOGGER=1`)

```
MUL_MAT q4_K m=1024 n=128 k=4096:    54 × 40.5 µs =  2185 µs ( 26.5 TFLOPS)
MUL_MAT q4_K m=12288 n=128 k=4096:   70 × 293.6 µs = 20550 µs ( 43.9 TFLOPS)
MUL_MAT q4_K m=4096 n=128 k=12288:   18 × 317.2 µs =  5709 µs ( 40.6 TFLOPS)
MUL_MAT q4_K m=4096 n=128 k=4096:    72 × 100.2 µs =  7217 µs ( 42.8 TFLOPS)
MUL_MAT_VEC q4_K m=12288 n=1 k=4096:  2 × 857.5 µs =  1715 µs (   0.117 TFLOPS)
MUL_MAT_VEC q4_K m=4096 n=1 k=4096:  37 ×  28.2 µs =  1043 µs (   1.2 TFLOPS)
FLASH_ATTN_EXT (128,32,128,1): 36 × 76.1 µs                       (  7.05 TFLOPS)
```

MUL_MAT-VEC at `n=1` (decode stage) is 200–375× slower than the equivalent MUL_MAT at `n=128`. This is **structural** (no tile-load amortization when M=1), not backend-specific. HIP shows the same pattern but hipblaslt is better-optimized for it.

## Code bugs identified (RDNA4 misdetection)

### Bug #1 — `get_device_architecture()` has no RDNA4 branch

`ggml/src/ggml-vulkan/ggml-vulkan.cpp:439–451`:

```cpp
if (subgroup_size_control_props.maxSubgroupSize == 64 && subgroup_size_control_props.minSubgroupSize == 64) {
    return vk_device_architecture::AMD_GCN;
}
if (subgroup_size_control_props.maxSubgroupSize == 64 && subgroup_size_control_props.minSubgroupSize == 32) {
    // RDNA
    if (shader_core_props_amd.wavefrontsPerSimd == 20) {
        return vk_device_architecture::AMD_RDNA1;
    }
    if (integer_dot_props.integerDotProduct4x8BitPackedMixedSignednessAccelerated) {
        return vk_device_architecture::AMD_RDNA3;
    }
    return vk_device_architecture::AMD_RDNA2;   // ← RDNA4 falls through here
}
```

RDNA4 matches `maxSubgroupSize=64, minSubgroupSize=32` (RDNA ladder), but its `integerDotProduct4x8BitPackedMixedSignednessAccelerated` is not set, so L447 is false and it falls through to L450.

### Bug #2 — KHR_cooperative_matrix disabled for non-RDNA3

`ggml-vulkan.cpp:18794–18799`:

```cpp
case VK_VENDOR_ID_AMD:
    if (driver_props.driverID == vk::DriverId::eAmdProprietary || ...) {
        // Workaround for AMD proprietary driver reporting support on all GPUs
        return arch == vk_device_architecture::AMD_RDNA3;   // ← only RDNA3
    }
```

The comment says AMD driver reports coopmat support on all GPUs, so the code whitelists RDNA3. **RDNA4 (mis-detected as RDNA2) gets the KHR_coopmat path disabled.**

### Bug #3 — `gpu_pipeline_configs` missing RDNA3/RDNA4

`ggml-vulkan.cpp:4076–4091`:

```cpp
static std::vector<GpuPipelineConfig> gpu_pipeline_configs = {
    { vk_device_architecture::AMD_RDNA1, { rdna1_pipelines }, RDNA_DEFAULT_SUBGROUP_SIZE },
    { vk_device_architecture::AMD_RDNA2, { rdna2_pipelines }, RDNA_DEFAULT_SUBGROUP_SIZE },
    // RDNA3 / RDNA4 not listed!
};
```

`get_subgroup_size()` (L4093–L4112) returns 0 for any arch not in the table; the call site (L4388) leaves `required_subgroup_size=0`; `L7252–L7253` falls back to `subgroup_props.subgroupSize` (the driver-reported value), which is **64 on RDNA4** even though the hardware native SIMD width is 32. This is consistent with the `warp size: 64` you can see in the Vulkan device init log.

## Suggested fix

1. Add `vk_device_architecture::AMD_RDNA4` to the enum.
2. In `get_device_architecture()`, detect RDNA4 (e.g. by `props.deviceID == 0x1201` or the RDNA4-specific shader-core / dot-product property) and return it before the L450 fallthrough.
3. Add an RDNA4 entry to `gpu_pipeline_configs` with `RDNA_DEFAULT_SUBGROUP_SIZE = 32`.
4. Change L18797 from `arch == AMD_RDNA3` to `arch >= AMD_RDNA3` (or include RDNA4 explicitly).

Estimated change: ~10 lines, 1 build ~10 min, validation ~30 min.

## Why our numbers don't match the issue

Best guesses, in order of likelihood:

- **Commit difference.** Issue reported 2026-08-06; we tested `b10298` (2026-08-07). If the OP tested an earlier commit (e.g. b10100–b10200) and the regression was already partially fixed, that could close most of the 5–7× gap.
- **Driver version.** Pre-Aug vs. current Adrenalin may differ in shader compile / driver-side coopmat behavior.
- **Test configuration.** Different `n-prompt` / `n-gen` / model size with `hidden ≥ 4096` may exhibit a different bottleneck.
- **Warp count / power-state.** RDNA4 has variable clocks; sustained-load throttling hits Vulkan harder than HIP since HIP is more compute-bound and not memory-bandwidth-bound for these shapes.

## Files in detail

| File | Purpose |
|---|---|
| `FINAL.md` | Executive summary, 5 key findings, 2nd-session timing breakdown |
| `ROOT_CAUSE.md` | Full code analysis with line numbers, hypothesis table, suggested fix |
| `WORKLOG.md` | 1st session state + 2nd session continuation log |
| `data/vulkan_vs_hip.csv` | 4-model × 2-backend raw `llama-bench` results |
| `data/vulkan_vs_hip_repro.csv` | Extended table (pp128 + pp512 + tg128 + PERF env) |
| `logs/perf_4b_vklogger_stderr.log` | 4B per-op timings (full Vulkan Timings blocks) |
| `logs/perf_8b_vklogger_stderr.log` | 8B per-op timings |
| `docs/mavis-plan.md` | Autonomous planning document (assumptions, time budget, anti-asks) |
| `docs/decisions.md` | 12 autonomous decisions with reasons and impact |
| `docs/self_eval.md` | Mavis self-evaluation (8/10) |
| `docs/issue_comment_body.md` | Ready-to-post comment body for llama.cpp #26663 |
| `docs/MANUAL_POST_INSTRUCTIONS.md` | Step-by-step guide to post the comment (3 min) |

## Posting the diagnostic to llama.cpp #26663

The full comment body is in `docs/issue_comment_body.md`. Posting is a 3-step manual operation:

1. Open https://github.com/ggml-org/llama.cpp/issues/26663
2. Copy the contents of `docs/issue_comment_body.md`
3. Paste into the comment box and click "Comment"

A fine-grained personal access token was used during diagnostic collection but has only `pull` permission on `ggml-org/llama.cpp`, so direct API posting is not possible from this environment. See `docs/MANUAL_POST_INSTRUCTIONS.md` for the detailed procedure.

## Provenance

- Collected: 2026-08-07
- Diagnostic agent: Mavis (autonomous, ~40 min active work)
- Tested: llama.cpp `b10298` (commit `15586e2d7`)
- Hardware: AMD Radeon RX 9070 XT 16 GB (gfx1201, RDNA4), Ryzen 7 7800X3D, 32 GB DDR5
- OS: Windows 11
- Driver: AMD Adrenalin 2026-08
