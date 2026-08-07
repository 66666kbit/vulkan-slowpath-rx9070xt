# Mavis 鑷富璇婃柇 plan 鈥?Vulkan gfx1201 鎱㈣矾寰?
> 鍚姩鏃堕棿: 2026-08-07 12:25 (2nd session)
> 鐢ㄦ埛涓嶅彲杈?10-12h
> 鎺ョ画鑷?1st session (2026-08-07 11:35-11:52)

## 鍏抽敭鍙戠幇 (1st session 楠岃瘉鍚?

- **鍙屽悗绔兘瑁呭ソ**: `llama-vulkan\` (Vulkan) + `llama-hip\` (HIP), 鍚屼竴 b10298 commit
- **纭欢宸紓** (杩欎釜寰堝彲鑳芥槸鏍瑰洜):
  - **Vulkan 鎶ュ憡**: device 0 = RX 9070 XT, **warp size: 64**, matrix cores: KHR_coopmat
  - **HIP 鎶ュ憡**: device 0 = RX 9070 XT, gfx1201, **Wave Size: 32**
  - RDNA4 (gfx1201) **纭欢** wave size = 32 (璺?Navi31/32 涓€鑷? RDNA 鍏ㄤ唬 32-wide)
  - **Vulkan driver 鎶?64 = 鍙嶅父**銆傚彲鑳芥槸 AMDVLK/Mesa driver reporting bug, 涔熷彲鑳芥槸 ggml-vulkan 鏄惧紡鍋囧畾 64
- **14B 鏈笅瀹?* (閿?.stale, 娈嬫枃浠?80MB), 闇€琛?- **0.6B/1.7B/4B/8B 閮戒笅瀹?* (4 涓?~8.5GB)

## 鍋囪 (priority 鎺掑簭)

### H1 (鏈€寮?: Vulkan 鍦?RDNA4 涓婄敤 wave=64 璋冨害, 娴垂涓€鍗?lane
- 瑙﹀彂鏉′欢: hidden鈮?096 (8B) 鏃?4-7x 鎱? 灏忔ā鍨?(0.6B) 姝ｅ父
- 瑙ｉ噴: 褰?GEMM M 缁村害 (hidden) 杈冨ぇ鏃? 64-wide wave 浼氳鍗?lane idle, 瀹為檯鍚炲悙鍑忓崐; 浣?4-7x 鎱㈣鏄庤繕鏈夊埆鐨?- 楠岃瘉鏂瑰紡: `GGML_VULKAN_PERF=1` 璺?4B/8B, 鐪?per-op 鏃堕暱; 鍚屾椂璺?0.6B/1.7B 瀵规瘮

### H2: ggml-vulkan shader compile 鏃堕€夐敊 sub-group size
- KHR_coopmat 鏀寔 dynamic sub-group, 浣?ggml 鍙兘 hard-code sub_group_size=64
- 楠岃瘉: SPIR-V dump (闇€ Vulkan SDK), 鎵?sub_group_size hint

### H3: Vulkan 鍚庣璧伴潪鏈€浼?GEMM kernel (娌″垏鍒?coopmat path)
- `matrix cores: KHR_coopmat` 鎶ヤ笂浜? 浣?ggml-vulkan 鍙兘娌″惎鐢?(榛樿璺緞杩樻槸鏅€?GEMM)
- 楠岃瘉: `GGML_VULKAN_DEBUG=1` 鐪?dispatch log, 鏈夋病鏈?coopmat shader dispatch

### H4: 鏄惧瓨璋冨害宸紓 (HIP MMA via rocblas vs Vulkan software path)
- HIP 璧?rocblas/hipblaslt, 楂樺害浼樺寲
- Vulkan 璧拌嚜宸辩殑 shader pipeline
- 楠岃瘉: per-op profile, 鐪嬪摢涓?op 宸窛鏈€澶?
## 璁″垝 (鎸?PROMPT 鏃堕棿棰勭畻)

| 闃舵 | 鐩爣 | 鏃堕棿 | 鐘舵€?|
|---|---|---|---|
| 0. Setup | WARP+token+script+14B download 鍚姩 | 30min | doing |
| 1. 璺?4 妯″瀷 bench (鏃?14B) | 鎷?vulkan_vs_hip.csv 鍓?4 琛?| 1h | next |
| 2. per-op profile (4B vs 8B) | GGML_VULKAN_PERF=1, 鎵炬參 op | 2h | pending |
| 3. SPIR-V 鍙嶆眹缂?(鎸夐渶) | Vulkan SDK 瑁?+ spirv-dis | 2h | pending |
| 4. 鏀逛唬鐮?/ 鍐?ROOT_CAUSE.md | 鎻?PR 鎴栧啓 issue comment | 2h | pending |
| 5. 鎻?comment 鍒?#26663 | 鏍瑰洜鎴栧亣璁?| 30min | pending |

**鎬?7.5h + buffer**銆備换浣曢樁娈?> 50% 瓒呴绠楃珛鍗冲仠銆?
## 鍏抽敭 env (1st session 宸叉湁, 2nd 娌跨敤)

- WARP: connected (12:24 楠岃瘉)
- `HF_ENDPOINT=https://huggingface.co` (鑴氭湰鍐呭凡璁?
- `GITHUB_TOKEN=a fine-grained personal access token` (User-level, 楠岃瘉 OK)
- `py` (Windows 涓嶇敤 `python`)

## 鍙嶅悜瑙勫垯 (鍐嶈儗涓€娆?

- 涓嶉棶鐢ㄦ埛, 涓嶈皟鏃堕棿, 涓嶈皟鐩爣 issue, 涓嶆鏌?WORKLOG 鍐欓敊娌?- 60% 瀹屾垚鐨勬姤鍛?> 0% 瀹岀編闂彞
- 澶辫触鍏滃簳: 鎻?comment 鍚?per-op profile + 鍋囪, **涓嶆槸澶辫触**
