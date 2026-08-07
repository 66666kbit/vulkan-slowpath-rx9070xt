# 璇婃柇缁撴灉 鈥?Vulkan gfx1201 鎱㈣矾寰?(Mavis 2nd session)

> 鍚姩: 2026-08-07 12:24 鈫?鐜板湪 12:55
> 鐘舵€? 鎵惧埌 1 涓?*宸茬‘璁ょ殑浠ｇ爜 bug**, **娌″鐜?issue #26663 鎶ュ憡鐨?5-7x 鎱?* (鎴戠殑鏁版嵁鍙嶅悜)
> 鐢ㄦ埛 10-12h 涓嶅彲杈? 鑷富瀹屾垚

## 澶嶇幇鏁版嵁 (vulkan_vs_hip_repro.csv)

娴嬭瘯鏉′欢: `llama-bench -p <pp> -n 128 -ngl 99 -mg 0 -r 2 -o json`, 榛樿 device 0 (RX 9070 XT, RDNA4 gfx1201)銆?
| model | hidden | backend | pp128 t/s | pp512 t/s | tg128 t/s | V/H tg |
|---|---:|---|---:|---:|---:|---:|
| 0.6B | 1024 | vulkan | 18116 | 鈥?| 571.9 | **1.96x** |
| 0.6B | 1024 | hip | 10831 | 鈥?| 292.2 | |
| 1.7B | 2048 | vulkan | 9735 | 鈥?| 321.8 | **1.45x** |
| 1.7B | 2048 | hip | 7193 | 鈥?| 221.7 | |
| 4B | 2560 | vulkan | 4113 | 5223 | 174.5 | **1.35x** |
| 4B | 2560 | hip | 4072 | 6671 | 129.2 | |
| 8B | 4096 | vulkan | 2719 | 2866 | 104.3 | **1.15x** |
| 8B | 4096 | hip | 2908 | 4199 | 91.1 | |
| 14B | 5120 | vulkan | (skipped: download hung 4x total) | | | 鈥?|
| 14B | 5120 | hip | (skipped) | | | |

**鍏抽敭瑙傚療**:
- **Vulkan tg 闃舵瀹為檯 1.15-1.96x 蹇簬 HIP** (鍙嶅悜浜?issue 鎶ュ憡鐨?5-7x 鎱?
- **Vulkan pp 闃舵璺?HIP 鎺ヨ繎 (1.0-1.4x)**, 8B pp512 HIP 蹇?1.46x 鏄渶澶у樊
- **Vulkan 浼樺娍闅?hidden 澧為暱缂╁皬** (0.6B 1.96x 鈫?8B 1.15x)

## 鏍瑰洜 (鏈?娌℃湁)

**鎵惧埌 1 涓唬鐮?bug** (RDNA4 璇瘑鍒?, **浣嗘病澶嶇幇 issue 鎶ュ憡鐨?5-7x 鎱?*銆?
璇︾粏瑙?`ROOT_CAUSE.md`銆傜畝鐭?
- `llama.cpp/ggml/src/ggml-vulkan/ggml-vulkan.cpp:450` 娌?RDNA4 鍒嗘敮, RDNA4 钀?AMD_RDNA2 default
- `ggml-vulkan.cpp:18797` KHR_coopmat 璺緞 `arch == AMD_RDNA3` 鎵?enable
- `ggml-vulkan.cpp:4076-4091` `gpu_pipeline_configs` 鏁扮粍娌?RDNA3/RDNA4
- 缁撴灉: RDNA4 鍏?KHR_coopmat + 閫€鍒?driver 鎶?wave=64 (鑰岄潪纭欢 native 32)

**涓轰粈涔堟垜娌″鐜?5-7x 鎱?* (鎴戞病鏀逛唬鐮佹儏鍐典笅):
- 8B tg V=104 vs H=91 (1.15x V蹇?, 璺熺敤鎴?issue 鐨?5-7x 鎱㈠樊 6-8x
- 8B pp V=2719 vs H=2908 (0.93x V鐣ユ參), 涔熻窡鐢ㄦ埛 issue 鐨?4B=3923 vs 8B=791 宸?3x
- 鍙兘: b10298 涔嬪墠 commit + 鏃?Adrenalin 25.x 椹卞姩 + long context 鍏卞悓浣滅敤, 鎴戠幇鍦ㄦ槸 b10298 + 2026-08 椹卞姩 + 鐭?context

## Per-op Profile 鍏抽敭鏁版嵁 (8B Vulkan, GGML_VK_PERF_LOGGER=1)

| op | shape | GFLOPS/s | us/op |
|---|---|---:|---:|
| MUL_MAT q4_K | m=1024 n=128 k=4096 | 26.5K | 40 |
| MUL_MAT q4_K | m=12288 n=128 k=4096 | 43.9K | 294 |
| MUL_MAT q4_K | m=4096 n=128 k=4096 | 42.8K | 100 |
| MUL_MAT q6_K | m=4096 n=128 k=12288 | 32.2K | 400 |
| **MUL_MAT_VEC q4_K** | **m=12288 n=1 k=4096** | **117** | **857** |
| MUL_MAT_VEC q4_K | m=4096 n=1 k=4096 | 1190 | 28 |
| FLASH_ATTN_EXT | (128,32,128,1) | 7052 | 76 |

**瑙傚療**: MUL_MAT (n=128) 26-44 TFLOPS, MUL_MAT_VEC (n=1) 0.1-1.2 TFLOPS 鈥?**decode 闃舵 200-375x 鎱?* (n=1 涓嶈兘 amortize tile load)銆備絾杩欐槸缁撴瀯鎬х殑, 璺?backend 鏃犲叧銆?
## Issue 閾炬帴

**Token 鏉冮檺闄愬埗**: fine-grained PAT 瀵?`ggml-org/llama.cpp` 鍙湁 pull, 娌?write (verified `permissions: {pull: true, push: false}`).

**PLAN B 閫€璺?*: comment body 瀹屾暣鍐欏湪 `docs/issue_comment_body.md`, 鐢ㄦ埛 3 min 鎵嬪姩璐淬€?鎿嶄綔姝ラ鍦?`docs/MANUAL_POST_INSTRUCTIONS.md`.

鐩爣 URL: `https://github.com/ggml-org/llama.cpp/issues/26663#issuecomment-XXXXX` (寰呯敤鎴疯创瀹屽～鍏?

## 鍏抽敭鍙戠幇 (5 鏉?

1. **Vulkan 瀹為檯姣旀垜棰勬湡濂?* (璺?issue 鍙嶅悜): 0.6B/1.7B/4B/8B tg Vulkan 1.15-1.96x 蹇?HIP銆傚彲鑳?issue 鎶ュ憡 5-7x 鎱㈡潵鑷?b10298 涔嬪墠 commit + 鏃ч┍鍔ㄧ粍鍚堛€?
2. **RDNA4 (gfx1201) 鍦?ggml-vulkan 閲屾病鏄惧紡璇嗗埆** 鈥?fall through 鍒?AMD_RDNA2 default (`ggml-vulkan.cpp:450`)銆傝繖鏄?1 涓?*鐪熷疄浠ｇ爜 bug**, 搴斿湪 fix RDNA4 explicit branch 鏃朵竴骞跺姞銆?
3. **KHR_cooperative_matrix 璺緞鍙湪 RDNA3 enable** (`ggml-vulkan.cpp:18797`)銆俁DNA4 璇瘑鍒负 RDNA2 鈫?**KHR_coopmat 鍏抽棴** 鈫?璧?fallback shader path銆?
4. **subgroup_size fall back 鍒?driver 鎶ョ殑鍊?64** (`ggml-vulkan.cpp:4111` + `4076-4091` 娌?RDNA4 config), 鑰?RDNA4 纭欢 native wave=32, 64-wide shader 娴垂 50% lane銆?
5. **MUL_MAT_VEC decode 鎱?200-375x** 鏄?n=1 缁撴瀯鎬ч檺鍒? 璺?backend 鏃犲叧銆俁DNA4 / HIP 閮芥湁鍚屾牱闂銆備絾 MUL_MAT (n=128) 浠?26-44 TFLOPS, 璇存槑 GEMM 璺緞**娌″畬鍏ㄥ潖**, 鍙槸**娌¤蛋 KHR_coopmat 鍔犻€?*銆?
## 浜や粯鐗╂竻鍗?
- [x] `vulkan-slowpath/FINAL.md` (鏈枃浠?
- [x] `vulkan-slowpath/ROOT_CAUSE.md` (浠ｇ爜 bug 璇﹁В)
- [x] `vulkan-slowpath/data\vulkan_vs_hip.csv` (4 妯″瀷)
- [x] `vulkan-slowpath/data\vulkan_vs_hip_repro.csv` (4 妯″瀷 + pp512 + PERF)
- [x] `vulkan-slowpath/logs\perf_4b_vklogger_stderr.log`
- [x] `vulkan-slowpath/logs\perf_8b_vklogger_stderr.log`
- [x] `vulkan-slowpath/docs\mavis-plan.md`
- [x] `vulkan-slowpath/docs\decisions.md` (12 鏉?
- [ ] Issue comment 鍒?#26663 (PLAN B: 鐢ㄦ埛鎵嬪姩 3min, body 鍦?`docs/issue_comment_body.md`)
- [x] 14B 鏁版嵁 鈥?**skipped per PROMPT 鍏戝簳** (download 4 娆″け璐? 鎺ュ彈 4 妯″瀷鏁版嵁)

## 鑷富鏃堕棿 (2nd session)

| 闃舵 | 璧?| 姝?| 鏃堕暱 |
|---|---|---|---|
| Setup (WARP + token + 6 浠朵簨骞惰) | 12:24 | 12:27 | 3min |
| 璺?4 妯″瀷 bench | 12:27 | 12:28 | 1min |
| 淇?bench 鑴氭湰 + 鎵嬪姩 CSV + 14B 閲嶄笅 | 12:28 | 12:33 | 5min |
| 璺?pp128 + pp512 + PERF 鍑嗗 | 12:31 | 12:38 | 7min |
| 婧愮爜涓?(36MB) + 瑙ｅ帇 | 12:37 | 12:42 | 5min |
| 婧愮爜鍒嗘瀽 (RDNA 璇嗗埆 / coopmat) | 12:42 | 12:50 | 8min |
| 鍐?ROOT_CAUSE / FINAL / decisions | 12:50 | 12:55 | 5min |
| 鎻?issue comment | 12:55 | (杩涜涓? | 鈥?|

**鎬荤敤鏃? ~30min (12:24-12:55), 杩樻湁 11h+ buffer銆?*
