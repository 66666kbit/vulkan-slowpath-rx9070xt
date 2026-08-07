## Diagnostic follow-up — code-side findings, did not fully reproduce the 5–7× regression on b10298

Tested on the same hardware as the OP: **RX 9070 XT (gfx1201), Ryzen 7 7800X3D, Windows 11, AMD Adrenalin 2026-08**, `llama.cpp` `b10298` (commit `15586e2d7`), both Vulkan and HIP backends built from the same source.

### Headline

I did **not** reproduce the 5–7× slowdown you reported. On `b10298`, the **Vulkan** backend is in fact **1.15×–1.96× faster than HIP** on token generation, across 0.6B / 1.7B / 4B / 8B models. Prefill (PP) is roughly even at small batch, HIP pulls ahead 1.3–1.5× at `pp512`. The pattern on the same hardware **does not match the 5–7× reported in the OP**.

That said, the source has a **real, independent code bug** for RDNA4 (`gfx1201`) that should be fixed.

### Reproduction (5 Qwen3 Q4_K_M, `llama-bench -ngl 99 -mg 0 -r 2 -o json`)

| model | hidden | backend | pp128 t/s | pp512 t/s | tg128 t/s |
|---|---|---|---:|---:|---:|
| 0.6B | 1024  | vulkan | 18116 | —    | 571.9 |
| 0.6B | 1024  | hip    | 10831 | —    | 292.2 |
| 1.7B | 2048  | vulkan |  9735 | —    | 321.8 |
| 1.7B | 2048  | hip    |  7193 | —    | 221.7 |
| 4B   | 2560  | vulkan |  4113 | 5223 | 174.5 |
| 4B   | 2560  | hip    |  4072 | 6671 | 129.2 |
| 8B   | 4096  | vulkan |  2719 | 2866 | 104.3 |
| 8B   | 4096  | hip    |  2908 | 4199 |  91.1 |
| 14B  | 5120  | (download pending on WARP egress) | | | |

For context, your OP reports `4B pp128 ≈ 3923` (I see **4113**) and `8B pp128 ≈ 791` (I see **2719**). I am 3.4× faster than your 8B pp128 number, on the same model and same backend.

### Per-op profile (8B Vulkan, `GGML_VK_PERF_LOGGER=1`)

```
MUL_MAT q4_K m=1024 n=128 k=4096:    54 × 40.5 µs = 2185 µs  ( 26.5 TFLOPS)
MUL_MAT q4_K m=12288 n=128 k=4096:   70 × 293.6 µs = 20550 µs ( 43.9 TFLOPS)
MUL_MAT q4_K m=4096 n=128 k=12288:   18 × 317.2 µs = 5709 µs  ( 40.6 TFLOPS)
MUL_MAT q4_K m=4096 n=128 k=4096:    72 × 100.2 µs = 7217 µs  ( 42.8 TFLOPS)
MUL_MAT_VEC q4_K m=12288 n=1 k=4096:  2 × 857.5 µs = 1715 µs  (   0.117 TFLOPS)  ← decode
MUL_MAT_VEC q4_K m=4096 n=1 k=4096:  37 ×  28.2 µs = 1043 µs  (   1.2 TFLOPS)
FLASH_ATTN_EXT (128,32,128,1): 36 × 76.1 µs                                (  7.05 TFLOPS)
```

MUL_MAT-VEC at n=1 is 200–375× slower than the equivalent MUL_MAT at n=128 — this is structural (no tile-load amortization when the M dimension is 1), not backend-specific.

### Real code bug I did find — RDNA4 misdetection

In `ggml/src/ggml-vulkan/ggml-vulkan.cpp`:

1. `get_device_architecture()` (L439–L451) detects AMD_GCN, AMD_RDNA1, AMD_RDNA3, then **falls through to AMD_RDNA2** at L450. **There is no RDNA4 branch.** RDNA4 (gfx1201) matches `maxSubgroupSize=64, minSubgroupSize=32` so it goes to the RDNA ladder, but its `integerDotProduct4x8BitPackedMixedSignednessAccelerated` is not set, so L447 is false and it falls to L450.

2. `ggml_vk_khr_cooperative_matrix_support()` at L18794–L18799:
   ```cpp
   case VK_VENDOR_ID_AMD:
       if (driver_props.driverID == vk::DriverId::eAmdProprietary || ...) {
           // Workaround for AMD proprietary driver reporting support on all GPUs
           return arch == vk_device_architecture::AMD_RDNA3;   // ← only RDNA3 gets coopmat
       }
   ```
   Comment says "AMD driver reports support on all GPUs" — so the code whitelists RDNA3 only. **RDNA4 (mis-detected as RDNA2) is therefore disabled from the KHR_cooperative_matrix path entirely.**

3. `gpu_pipeline_configs` (L4076–L4091) only contains RDNA1 and RDNA2 entries. `get_subgroup_size()` (L4093–L4112) returns 0 for any arch not in the table; the call site (L4388) then leaves `required_subgroup_size=0`, and `L7252–L7253` falls back to `subgroup_props.subgroupSize` (the driver-reported value), which is **64 on RDNA4** even though the hardware native SIMD width is 32. This is consistent with the `warp size: 64` you can see in the Vulkan device init log.

The compound effect: on RDNA4, ggml-vulkan runs **without KHR_coopmat** and **with subgroup size 64** (instead of the hardware-native 32). This is a real bug worth fixing regardless of whether it explains your 5–7× number.

### Suggested minimal fix

1. Add `vk_device_architecture::AMD_RDNA4` to the enum.
2. In `get_device_architecture()`, detect RDNA4 (e.g. by `props.deviceID == 0x1201` or the RDNA4-specific shader-core / dot-product property) and return it before the L450 fallthrough.
3. Add an RDNA4 entry to `gpu_pipeline_configs` with `RDNA_DEFAULT_SUBGROUP_SIZE = 32` (same as RDNA1/RDNA2).
4. Change L18797 from `arch == AMD_RDNA3` to `arch >= AMD_RDNA3` (or include RDNA4 explicitly).

### Why my numbers don't match yours

Best guesses, in order of likelihood:

- **Commit difference.** You reported the issue on 2026-08-06; I tested `b10298` (2026-08-07). If you tested an earlier commit (e.g. b10100–b10200) and the regression was already partially fixed, that could close most of the 5–7× gap.
- **Driver version.** If you tested on a pre-Aug Adrenalin and I tested on the current one, the underlying shader compile / driver-side coopmat behavior could differ.
- **Test configuration.** Different `n-prompt` / `n-gen` / model size with `hidden ≥ 4096` may exhibit a different bottleneck.
- **Warp count / power-state.** RDNA4 has variable clocks; if your run was throttled (power limit, sustained load), Vulkan would be hit harder than HIP since HIP is more compute-bound and not memory-bandwidth-bound for these shapes.

I would love to see a `GGML_VK_PERF_LOGGER=1` trace from your run on the original commit you tested, plus the Adrenalin version (`RadeonSoftwareVersion` from registry or AMD Adrenalin UI). If RDNA4 was indeed falling off the KHR_coopmat path on that commit too, the per-op profile should show very low `MUL_MAT` TFLOPS in the 8B trace (sub-20 TFLOPS would be a smoking gun).

### Files

- `data/vulkan_vs_hip.csv` — raw `vulkan_vs_hip` run (4 models, 14B pending)
- `data/vulkan_vs_hip_repro.csv` — extended (pp128 + pp512 + PERF env row)
- `logs/perf_4b_vklogger_stderr.log` — 4B per-op profile
- `logs/perf_8b_vklogger_stderr.log` — 8B per-op profile
- `ROOT_CAUSE.md` — full code analysis (also in this PR-ready bundle)

Happy to file a PR with the RDNA4 detection fix once the maintainers confirm the direction, or if you'd like me to also include a quick `ggml_vk_khr_cooperative_matrix_support` adjustment to whitelist RDNA4. Just let me know.
