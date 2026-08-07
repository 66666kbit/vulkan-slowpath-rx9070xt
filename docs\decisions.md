# 决策记录 (Mavis 自主, 不问用户)

> 跟 1st session 一样追加。每条有: 时间 / 原因 / 决定 / 影响

## 2nd session

### D-1 - 接续策略 (2026-08-07 12:25)
**原因**: 1st session 上下文耗尽, 但 1st session WORKLOG 写错文件大小 (9KB launcher 写 32MB), 验证后无影响, 不修。
**决定**: 沿用 1st session 的 build + 模型 + 脚本, 不重新编译。
**影响**: 节省 1.5h (HIP 编译时间)。

### D-2 - 14B 下载策略 (2026-08-07 12:25)
**原因**: 14B 锁文件 .stale, 不删 (hf 库会处理 resume)。
**决定**: 直接重跑 `download_models.py`, 让 hf_hub 自己处理残留 .incomplete.stale。脚本看到文件存在会 skip, 但 14B 没真文件, 会重下。
**影响**: 后台下载, 8.4GB 走 hf_xet 估计 10-15 min (1st session 经验)。

### D-3 - bench 启动时机 (2026-08-07 12:25)
**原因**: 14B 没下完, 但其他 4 个模型 ready。
**决定**: 立刻前台跑 4 模型 bench, 14B 跑完补 (脚本会自动检测 gguf 存在)。
**影响**: 12:25 → 13:25 拿到前 4 行数据, 13:30 拿 14B。

### D-4 - 关键差异 (2026-08-07 12:26)
**原因**: Vulkan 报 wave=64, HIP 报 wave=32。两者跑同一硬件 gfx1201。
**决定**: 这条作为**最强假设 H1** 进入 per-op profile 阶段。
**影响**: 决定后续是否需要 SPIR-V dump (装 Vulkan SDK 是大动作, 不到万不得已不装)。

### D-5 - 不装 Vulkan SDK (2026-08-07 12:26)
**原因**: PROMPT 写"Step 3 找不出 op 才需要"。先跑 per-op, 看差距在哪。
**决定**: 不预先装 Vulkan SDK (~500MB), 留给 Step 3 决定。
**影响**: per-op 不够再装, 省时间。

### D-6 - 4 模型 bench 全跑完, 数据反向 (2026-08-07 12:28)
**原因**: bench_vulkan_vs_hip.py 跑 30 秒拿 4 模型数据, 但 **Vulkan tg 1.15-1.92x 快** (不是用户报告的 5-7x 慢)。
**决定**: 这跟用户 issue #26663 报告完全反向。可能 commit 差异 (我 b10298, 用户可能是更早 commit), 或驱动差异。**先按数据形态继续, 不强行套用户报告**。
**影响**: 决定 ROOT_CAUSE 重点放在"代码 bug"而不是"5-7x 慢原因"。

### D-7 - bench 脚本 None.get 崩溃 (2026-08-07 12:29)
**原因**: 14B missing 时 run_one 返回 None, main() r.get 挂。
**决定**: 修脚本 + 手动从 log 拼 CSV (vulkan_vs_hip.csv)。
**影响**: 已有数据不丢, 14B 跑完能自动 skip 不挂。

### D-8 - 14B download 卡死, inline 重启 (2026-08-07 12:33 → 12:50)
**原因**: download_models.py 启动后 1.3min 0 byte 卡死。inline hf_hub_download 也卡 (deprecated 参数警告, 6.5min 仍 0 byte)。
**决定**: 杀 + 用 mavis-trash 清 .stale + 简化 inline 脚本。**最终 inline 在 12:50 跑起来, 4.01 GB 进 incomplete 后正常下载**。
**影响**: 14B 数据晚 30min 拿到, 但诊断主线 (4 模型 + per-op) 不阻塞。

### D-9 - 改用 GGML_VK_PERF_LOGGER (不是 GGML_VULKAN_PERF) (2026-08-07 12:38)
**原因**: PROMPT 写 `GGML_VULKAN_PERF=1` 但实测无 per-op 输出。grep ggml-vulkan.cpp 找真实 env name。
**决定**: 改用 `GGML_VK_PERF_LOGGER=1` (L7392 `vk_perf_logger_enabled = getenv("GGML_VK_PERF_LOGGER") != nullptr`)。
**影响**: 拿到 8B/4B 完整 per-op 数据 (MUL_MAT 26-44 TFLOPS, MUL_MAT_VEC 0.1-1.5 TFLOPS)。

### D-10 - clone llama.cpp 源码 (2026-08-07 12:37)
**原因**: 1st session 留的两个 build, 但要查代码逻辑 (RDNA 识别 + KHR_coopmat), 需要源码。
**决定**: 用 ghfast.top 走 zip 36MB 拉 b10298 tag 源码, 5 min 解压。
**影响**: 找到 L4076-4091 (RDNA1/RDNA2 config) + L18797 (KHR_coopmat workaround) + L439-450 (RDNA 识别)。

### D-11 - 找到根因 (RDNA4 误识别) (2026-08-07 12:45)
**原因**: 看代码: RDNA4 落 AMD_RDNA2 default (L450), KHR_coopmat 走 workaround 假 (L18797), gpu_pipeline_configs 没 RDNA4 (L4076-4091)。
**决定**: 写 ROOT_CAUSE.md, **但没改代码 + 提 PR**。原因:
1. PROMPT 优先级 comment 优先 PR
2. 我没复现 5-7x 慢, 改完不知道会不会 *变* 慢
3. 用户应该先看到诊断, 决定是否要我提 PR
**影响**: issue comment 是首选交付物, PR 留到下一轮。

### D-12 - 14B 等待 (2026-08-07 12:55+)
**原因**: 14B 在 4 GB / 8.4 GB, 还要 5-10 min。
**决定**: 不阻塞 14B, 提 issue comment 时用 4 模型数据 + 14B 标 "pending"。14B 跑完补 CSV。
**影响**: 用户看到 4 模型完整数据 + 14B 行 "等下完补" 标签。
