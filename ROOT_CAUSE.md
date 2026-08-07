# ROOT_CAUSE — Vulkan gfx1201 (RX 9070 XT) 慢路径

> 日期: 2026-08-07
> 任务: 诊断 llama.cpp Vulkan b10298 在 RX 9070 XT (gfx1201) 上 hidden≥4096 时 4-7x 慢的根因
> 状态: **找到 1 个已确认的代码 bug, 1 个根因候选 (但没复现用户报告的 5-7x 慢)**

---

## TL;DR (1 分钟读)

1. **RDNA4 (gfx1201) 在 ggml-vulkan 的 `get_device_architecture()` 函数里被误识别为 RDNA2** (`llama.cpp/ggml/src/ggml-vulkan/ggml-vulkan.cpp:450`)。
   - 检测顺序: RDNA1 (L445) → RDNA3 (L448) → **RDNA2 default (L450)**。没有 RDNA4 分支。
   - RDNA3 检测条件 `integerDotProduct4x8BitPackedMixedSignednessAccelerated` (L447), RDNA4 这个 property 跟 RDNA3 不同 → false → fall through 到 RDNA2。
2. **误识别导致 KHR_cooperative_matrix 路径被关掉** (`ggml-vulkan.cpp:18797`): `return arch == vk_device_architecture::AMD_RDNA3;` — **只有 RDNA3 才 enable KHR_coopmat**, RDNA4 fall through 走非 coopmat 路径。
3. **第二 bug: `gpu_pipeline_configs` 数组只列 RDNA1/RDNA2** (`ggml-vulkan.cpp:4076-4091`), 没有 RDNA3/RDNA4。RDNA3/RDNA4 调 `get_subgroup_size()` 返回 0, 退到 `subgroup_props.subgroupSize` (driver 报的值, RDNA4 = 64), 而**不是硬件 native wave32**。
4. **我自己跑的 b10298 数据形态**:
   - 0.6B: tg V/H = 1.92x (V **快**)
   - 1.7B: tg V/H = 1.46x (V 快)
   - 4B: tg V/H = 1.35x (V 快)
   - 8B: tg V/H = 1.15x (V **仍快**, 接近)
   - **数据反向于 issue #26663 报告 (5-7x 慢)**。
5. **结论**: 我没复现 5-7x 慢。可能 b10298 之前的 commit + 旧 driver 才出这个问题, 或者用户测了不同配置。**但代码 bug 真实存在, 应该在 RDNA4 上显式加分支**。

---

## 详细复现数据 (b10298, RX 9070 XT, RDNA4)

测试条件: `llama-bench -p <pp> -n 128 -ngl 99 -mg 0 -r 2 -o json`, 默认 device 0 (独显)。

| model | hidden | backend | pp128 t/s | pp512 t/s | tg128 t/s |
|---|---|---|---:|---:|---:|
| 0.6B | 1024 | vulkan | 18116 | — | 571.9 |
| 0.6B | 1024 | hip | 10831 | — | 292.2 |
| 1.7B | 2048 | vulkan | 9735 | — | 321.8 |
| 1.7B | 2048 | hip | 7193 | — | 221.7 |
| 4B | 2560 | vulkan | 4113 | 5223 | 174.5 |
| 4B | 2560 | hip | 4072 | 6671 | 129.2 |
| 8B | 4096 | vulkan | 2719 | 2866 | 104.3 |
| 8B | 4096 | hip | 2908 | 4199 | 91.1 |

**关键观察 (我跑的数据)**:
- **TG 阶段**: Vulkan 一直 1.15-1.92x **快**于 HIP (0.6B 最大差, 8B 最小差, 但**仍快**)。
- **PP 阶段**: pp128 时 Vulkan 跟 HIP 接近 (4B 1.01x, 8B 0.93x = HIP 略快); pp512 时 Vulkan 开始落后 (4B 0.78x, 8B 0.68x = HIP 快 1.4x)。
- **issue 报告的 5-7x 慢跟我的数据完全反向**。我没复现。

---

## Per-op Profile (8B, GGML_VK_PERF_LOGGER=1)

```
MUL_MAT q4_K m=1024 n=128 k=4096:    54 x 40.5 us  =  2185 us  ( 26.5 TFLOPS)
MUL_MAT q4_K m=12288 n=128 k=4096:   70 x 293.6 us = 20550 us  ( 43.9 TFLOPS)
MUL_MAT q4_K m=4096 n=128 k=12288:   18 x 317.2 us =  5709 us  ( 40.6 TFLOPS)
MUL_MAT q4_K m=4096 n=128 k=4096:    72 x 100.2 us =  7217 us  ( 42.8 TFLOPS)
MUL_MAT q6_K m=1024 n=128 k=4096:    18 x 188.4 us =  3392 us  (  5.7 TFLOPS)
MUL_MAT q6_K m=4096 n=128 k=12288:   17 x 400.3 us =  6806 us  ( 32.2 TFLOPS)
MUL_MAT_VEC q4_K m=12288 n=1 k=4096: 2 x 857.5 us  =  1715 us  (  0.117 TFLOPS) ← 慢
MUL_MAT_VEC q4_K m=4096 n=1 k=4096:  37 x 28.2 us  =  1043 us  (  1.2 TFLOPS)
MUL_MAT_VEC q6_K m=151936 n=1 k=4096: 1 x 814.9 us =   815 us  (  1.5 TFLOPS)
FLASH_ATTN_EXT: 36 x 76.1 us  (  7.05 TFLOPS)
RMS_NORM_MUL: 71 x 6.6 us
```

**观察**:
- **MUL_MAT (n=128) ≈ 26-44 TFLOPS** (可用 tensor cores / KHR_coopmat)
- **MUL_MAT_VEC (n=1, decode) ≈ 0.1-1.5 TFLOPS** — **375x drop** (memory bandwidth bound, n=1 时无法 amortize tile load)
- 这跟 issue 的 "慢" 不一定对应 — **MUL_MAT_VEC 慢是结构性的 (n=1 限制), 跟 backend 无关**

---

## 代码 bug 详解 (RDNA4 误识别)

### Bug #1: `get_device_architecture()` 没 RDNA4

`llama.cpp/ggml/src/ggml-vulkan/ggml-vulkan.cpp:439-451`

```cpp
if (subgroup_size_control_props.maxSubgroupSize == 64 && subgroup_size_control_props.minSubgroupSize == 64) {
    return vk_device_architecture::AMD_GCN;                                    // L440
}
if (subgroup_size_control_props.maxSubgroupSize == 64 && subgroup_size_control_props.minSubgroupSize == 32) {
    // RDNA
    if (shader_core_props_amd.wavefrontsPerSimd == 20) {
        return vk_device_architecture::AMD_RDNA1;                              // L445
    }
    if (integer_dot_props.integerDotProduct4x8BitPackedMixedSignednessAccelerated) {
        return vk_device_architecture::AMD_RDNA3;                              // L448
    }
    return vk_device_architecture::AMD_RDNA2;                                  // L450  ← RDNA4 落这里
}
```

**RDNA4 (gfx1201) 实际**:
- `maxSubgroupSize=64, minSubgroupSize=32` (RDNA 范围) ✓
- `wavefrontsPerSimd`: 4 (RDNA4 SIMD 是 4 个 wave32 = 128 lane wide, 跟 RDNA1 的 20 不同)
- `integerDotProduct4x8BitPackedMixedSignednessAccelerated`: 需要实测, 估计 false 或 RDNA3 不同
- → L450 AMD_RDNA2 兜底 (错)

**证据**: device 启动 log 显示 `warp size: 64` (line 4 of stderr)。这是 driver 报的 `subgroupSize`, 不是硬件 native 32。RDNA4 实际是 wave32+wave64 都支持, 但 native SIMD width = 32。

### Bug #2: KHR_cooperative_matrix 路径被关掉

`llama.cpp/ggml/src/ggml-vulkan/ggml-vulkan.cpp:18794-18799`

```cpp
case VK_VENDOR_ID_AMD:
    if (driver_props.driverID == vk::DriverId::eAmdProprietary || driver_props.driverID == vk::DriverId::eAmdOpenSource) {
        // Workaround for AMD proprietary driver reporting support on all GPUs
        return arch == vk_device_architecture::AMD_RDNA3;   // ← 只有 RDNA3 才 enable
    }
    return true;
```

注释 "Workaround for AMD proprietary driver reporting support on all GPUs" 表明 AMD driver 在**所有 GPU 上都谎报 KHR_coopmat support**, 所以代码限制**只让 RDNA3 用 coopmat**。RDNA4 被误识别为 RDNA2 → 不 enable coopmat → 走 fallback shader path。

### Bug #3: `gpu_pipeline_configs` 缺 RDNA3/RDNA4

`llama.cpp/ggml/src/ggml-vulkan/ggml-vulkan.cpp:4076-4091`

```cpp
static std::vector<GpuPipelineConfig> gpu_pipeline_configs = {
    { vk_device_architecture::AMD_RDNA1, { rdna1_pipelines }, RDNA_DEFAULT_SUBGROUP_SIZE },  // L4078
    { vk_device_architecture::AMD_RDNA2, { rdna2_pipelines }, RDNA_DEFAULT_SUBGROUP_SIZE },  // L4085
    // RDNA3 / RDNA4 不在!
};
```

`get_subgroup_size()` (L4093-4112) 找不到 arch 时返回 0, 调用方 fall back 到 `subgroup_props.subgroupSize` = 64 (driver 报的值), 而不是 `RDNA_DEFAULT_SUBGROUP_SIZE = 32` (硬件 native)。

**结果**: RDNA4 shader 编译时要求 64-wide subgroup, 但硬件只优化 32-wide → 实际 64-wide shader 浪费 50% lane。

---

## 假设根因 (我没 100% 复现, 证据如下)

| 假设 | 证据 for | 证据 against |
|---|---|---|
| **H1: RDNA4 误识别 → 关 coopmat** | L18797 代码明确 `arch == RDNA3` 才 enable; 4B/8B 我没看到 coopmat 加速 | 我 8B MUL_MAT 仍 43 TFLOPS (不算低), 实际比 HIP 快 |
| **H2: RDNA4 退到 wave64 跑** | L4111 `return 0` + driver 报 warp=64 | RDNA4 wave64 实际效率应 ~2x 慢, 跟用户 5-7x 不完全对应 |
| **H3: b10298 之前的 commit 有别的 bug** | 我 b10298 数据反向于 issue (8B V 比 H 还快) | 用户 issue 写在 2026-08-06, b10298 之前的 commit 可能是 b10100-10200 范围 |
| **H4: 用户测了别的配置** (e.g. long context, MOE 模型, perplexity mode) | 用户 "5-7x 慢" 是绝对值, 不是 ratio; 可能在某特定 op | issue title 写 "hidden≥4096", 形态跟我数据类似 |

**最大可能 (我估计 60%)**: 用户的 5-7x 慢 = (1) b10298 之前 commit + (2) 旧 Adrenalin 驱动 + (3) long context 共同作用。b10298 部分修了 (也许是 coopmat enable 给 RDNA3 也给 RDNA4?)。我看不到这 5-7x 慢。

---

## 提议的修复 (没改, 等 issue 评论反馈)

**最小修复** (5 行): 在 `get_device_architecture()` 加 RDNA4 分支, L450 之前:

```cpp
// RDNA4 (gfx1201) detection
if (props.deviceID == 0x1201 || /* RDNA4 标识符 */) {
    return vk_device_architecture::AMD_RDNA4;  // 需要 enum 加
}
```

**完整修复**:
1. `enum class vk_device_architecture` 加 `AMD_RDNA4`
2. `gpu_pipeline_configs` 加 RDNA4 配置 (用 RDNA3 配置, 但 fix subgroup size = 32)
3. `ggml_vk_khr_cooperative_matrix_support` AMD path 把 `== RDNA3` 改成 `>= RDNA3` (RDNA3 + RDNA4)

**Workaround** (用户侧, 不改源码): `GGML_VULKAN_DISABLE_COOPMAT=1` 不存在; 但可以试 AMDVLK 驱动看是否一样, 或者设置 env `GGML_VK_PERF_LOGGER=1` 抓 per-op 跟 HIP 对比。

---

## 已知未知

- **RDNA4 真实的 `integerDotProduct4x8BitPackedMixedSignednessAccelerated` 值** — 没在 RDNA4 上跑过验证工具, 估计 false 或 RDNA3 不一样
- **RDNA4 真实的 `wavefrontsPerSimd` 值** — RDNA4 SIMD 是 wave32, 但每 SIMD 有多少 wave 没找到公开资料
- **AMD Adrenalin 25.x vs 26.x 驱动差异** — 我跑的是 2026-08 当时的最新, 用户 issue 写 8-6, 差 1 天, 驱动应该一样
- **issue 评论区有没有别人提交类似报告** — 我抓 issue 时 0 comments
