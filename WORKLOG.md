# Vulkan gfx1201 閹便垼鐭惧鍕槚閺?閳?瀹搞儰缍旈弮銉ョ箶 (Mavis 1st session 閻ｆ瑤绗呴惃?

> 閸氼垰濮╅弮鍫曟？: 2026-08-07 11:35
> 1st session 瀹稿弶娈忛崑?(娑撳﹣绗呴弬鍥╂暏鐏?, 閺傞绱扮拠婵囧复缂?> 閻╊喖缍? vulkan-slowpath/

## 瑜版挸澧犻悩鑸碘偓?(閺傞绱扮拠婵囧复缂侇厽妞? 瀹稿弶婀佺挧鍕爱)

### 瀹歌弓绗呯€瑰瞼娈戝Ο鈥崇€?(4/5)
- `Qwen3-0.6B-GGUF/Qwen3-0.6B-Q4_K_M.gguf` (378 MB) 閴?- `Qwen3-1.7B-GGUF/Qwen3-1.7B-Q4_K_M.gguf` (1.0 GB) 閴?- `Qwen3-4B-GGUF/Qwen3-4B-Q4_K_M.gguf` (2.4 GB) 閴?- `Qwen3-8B-GGUF/Qwen3-8B-Q4_K_M.gguf` (4.7 GB) 閴?- `Qwen3-14B-GGUF/Qwen3-14B-Q4_K_M.gguf` (8.4 GB) 閴?**闂団偓闁插秳绗?* (lock + incomplete 瀹歌尪顫?mark 娑?.stale, 閻╁瓨甯撮柌宥堢獓 download_models.py 閸楀啿褰?

### 瀹歌尪顥婃總鐣屾畱 llama.cpp (閸欏苯鎮楃粩?
- `llama-vulkan/llama-bench.exe` (b10298, Vulkan build, 32 MB)
- `llama-hip/llama-bench.exe` (b10298, HIP/ROCm build, 310 MB)
- 娑撱倓閲滈柈钘夊嚒鐟欙絽甯? 閸欏苯鍤崣顖濈獓
- 妤犲矁鐦夋潻?
  - Vulkan: 閹垫儳鍩?2 device, RX 9070 XT (gfx1201, KHR_coopmat), iGPU
  - HIP: 閹垫儳鍩?2 device, RX 9070 XT (gfx1201, 16GB VRAM, Wave32), iGPU
  - **闁插秷顩﹀顔肩磽**: Vulkan wave=64, HIP wave=32 (娑撳秴鎮撻幍褑顢戝Ο鈥崇础)

### 瀹告彃鍟撴總鐣屾畱閼存碍婀?- `scripts/download_models.py` (4/5 鐎瑰本鍨? 闁插秷绐囨导?skip 瀹歌弓绗? 閼奉亜濮╃悰?14B)
- `scripts/download-llama.ps1` (瀹歌尪绐囩€? 娑撳秳绱伴崘宥埿曢崣?
- `scripts/bench_vulkan_vs_hip.py` (閺堫亣绐? **閺傞绱扮拠婵堫儑娑撯偓娑擃亣绐囨潻娆庨嚋**)

### 閺傚洣娆㈡径鐟扮鐏炩偓
```
vulkan-slowpath/
閳规壕鏀㈤埞鈧?llama-vulkan\          # b10298 Vulkan build
閳规壕鏀㈤埞鈧?llama-hip\             # b10298 HIP build
閳规壕鏀㈤埞鈧?models\
閳?  閳规壕鏀㈤埞鈧?Qwen3-0.6B-GGUF\   (1 GGUF)
閳?  閳规壕鏀㈤埞鈧?Qwen3-1.7B-GGUF\   (1 GGUF)
閳?  閳规壕鏀㈤埞鈧?Qwen3-4B-GGUF\     (1 GGUF)
閳?  閳规壕鏀㈤埞鈧?Qwen3-8B-GGUF\     (1 GGUF)
閳?  閳规柡鏀㈤埞鈧?Qwen3-14B-GGUF\    (瀵板懘鍣告稉?
閳规壕鏀㈤埞鈧?scripts\
閳?  閳规壕鏀㈤埞鈧?download_models.py
閳?  閳规壕鏀㈤埞鈧?download-llama.ps1
閳?  閳规柡鏀㈤埞鈧?bench_vulkan_vs_hip.py
閳规壕鏀㈤埞鈧?data\                  # 鐠烘垵鐣崥搴㈡杹 CSV
閳规壕鏀㈤埞鈧?logs\                  # 閹碘偓閺堝妫╄箛?閳规壕鏀㈤埞鈧?WORKLOG.md             # 閺堫剚鏋冩禒?閳规柡鏀㈤埞鈧?PROMPT-*.md            # 閺傞绱扮拠?prompt (1st session 閸愭瑧娈?
```

## 閼奉亙瀵岄弮鍫曟？妫板嫮鐣?(1st session 鐎规氨娈? 娴犲懍绶甸崣鍌濃偓?

| 闂冭埖顔?| 妫板嫯顓?| 鐟欙箑褰?閸?閻ㄥ嫭娼禒?|
|---|---|---|
| A. Setup + 娑撳娴囧Ο鈥崇€?| 1.5h (瀹告彃銇囬柈銊ュ瀻鐎瑰本鍨? | HIP 鐟佸懍绗夋稉濠傛皑閸?|
| B. 婢跺秶骞?Vulkan vs HIP | 2h | 閺佺増宓佺捄鐔烘暏閹撮鐟拋鏉款嚠娑撳秳绗傜亸鍗炰粻 |
| C. per-op profile | 3h | `GGML_VULKAN_PERF` 濞屸€蹭繆閹垰姘ㄩ崑?|
| D. SPIR-V (閹稿娓? | 2h | Vulkan SDK 鐟佸懍绗夋稉濠傛皑閸?|
| E. 閹绘劘顔呮穱顔碱槻 | 3h | 閹靛彞绗夐崚鐗堢壌閸ョ姴姘ㄦ潪?issue comment |
| F. 閹绘劒姘?| 1h | API 閹恒劋绗夋稉濠傛皑閸?|

**閹?13h + 2h buffer**閵嗗倷鑵戦梻缈犳崲娴ｆ洑绔撮梼鑸殿唽閸椻剝顒寸搾鍛扮箖妫板嫮鐣荤亸鍗炰粻娑? 閹跺﹤缍嬮崜宥呭絺閻滄澘鍟撻幋?issue comment 閹绘劒姘? 鏉╂瑤绡冮弰顖涙箒閺佸牅姘︽禒妯糕偓?
## 閸愬磭鐡ョ拋鏉跨秿

### #1 - 閸氼垰濮?(2026-08-07 11:35)
**閸樼喎娲?*: 閻劍鍩涢幒鍫熸綀閼奉亙瀵岄幍褑顢? 缂佹瑤绨?Vulkan slowpath 娴犺濮?**閸愬啿鐣?*: 閼奉亜绻侀獮? 閺冨爼妫挎０鍕暬閼奉亜绻佺€? Vulkan SDK 娑撳秹顣╅崗鍫ｎ棅

### #2 - 瀹搞儰缍旈惄顔肩秿 (2026-08-07 11:36)
**閸愬啿鐣?*: vulkan-slowpath/ 闂呮梻顬? 鐠?1st 娴犺濮熼惃?quant bench 閸掑棗绱?
### #3 - cron 鐠佸墽鐤?(2026-08-07 11:36)
**閸愬啿鐣?*: cron "vulkan-slowpath-check" 濮?30min 閻╂垶甯?heartbeat (閺傞绱扮拠婵嗗讲娴犮儰绻氶幐浣瑰灗闁插秵鏌婄拋?

### #4 - 1st session 閺嗗倸浠?(2026-08-07 11:52)
**閸樼喎娲?*: 閻劍鍩涚憰浣圭湴閸嬫粈绗呴弶? 娑撳﹣绗呴弬鍥у讲閼虫垝绗夋径? 閺傞绱扮拠婵囧复缂?**閻樿埖鈧?*: 4/5 濡€崇€锋稉瀣暚, 2 娑?build 闁€燁棅婵? bench 閼存碍婀伴崘娆忋偨娴ｅ棙婀捄?**閺傞绱扮拠婵囧复缂侇厾鍋?*: 闁插秷绐?`download_models.py` 鐞?14B 閳?鐠?`bench_vulkan_vs_hip.py` 閹锋寧鏆熼幑?
---

## 閸忔娊鏁崣鎴犲箛 (瀵板懎锝? 缂佹瑦鏌婃导姘崇樈)

(濮ｅ繐褰傞悳棰佺閺夆剝鐗撮崶鐘垫祲閸忓磭娈戞禍瀣杽鐏忚精鎷烽崝?

---

## 婢惰精瑙﹂崗婊冪俺 (缂佹瑦鏌婃导姘崇樈)

- Vulkan SDK 鐟佸懍绗夋稉?閳?鐠哄疇绻?SPIR-V, 閹?issue comment
- 閹靛彞绗夐崚鐗堢壌閸?閳?閹?issue comment 閸?per-op profile + 閸嬪洩顔?- API 閹恒劋绗夋稉?閳?閻ｆ瑧绮伴悽銊﹀煕閹靛濮╅幒?- 娑撳﹣绗呴弬鍥偓妤€鏁?閳?閸愭瑦婀伴弬鍥︽閻樿埖鈧? 閸氼垱鏌婃导姘崇樈

## 瀹歌尙鐓￠崣鍌濃偓?(閻劍鍩涚粭鏃囶唶閸?GitHub)

- `ISSUE_DRAFT_vulkan_gfx1201_slowpath.md` (issue #26663 閼藉顭?
- `LLM_BENCH.md` (Vulkan vs HIP 鐎佃鐦崺铏瑰殠)
- `ISSUE_TRACKING.md` (鐠虹喕绻樼悰?
- `TRITON_UNLOCK_ADDENDUM.md`, `DGX_SPARK_ECOSYSTEM.md` (閸忔湹绮粭鏃囶唶, 闂傚瓨甯撮崣鍌濃偓?

GitHub 娴? github.com/66666kbit/rx9070xt-llm-bench

---

## 2nd session 缂侇厼浠?(2026-08-07 12:24 閳?12:48, 鏉╂稖顢戞稉?

### 閻樿埖鈧礁褰夐崠?
- 閹恒儳鐢?1st session 閻樿埖鈧? 2nd session 娑撳﹣绗呴弬?*閺?* (娴?Mavis memory 瀹稿弶婀侀梹璺ㄢ柤閸楀繗顔?
- 1st session 閻ｆ瑧娈?4/5 濡€崇€?+ 閸?build + bench 閼存碍婀? 閸?valid, 濞岃法鏁?
### 閸忔娊鏁崣鎴犲箛 (2nd session 閼奉亙瀵?

#### 閸欐垹骞?#1: Vulkan 鐎圭偤妾В?HIP 韫?1.15-1.96x (閸欏秴鎮滄禍?issue 閹躲儱鎲?

| model | hidden | vulkan tg | hip tg | V/H |
|---|---|---:|---:|---:|
| 0.6B | 1024 | 571.9 | 292.2 | **1.96x** |
| 1.7B | 2048 | 321.8 | 221.7 | **1.45x** |
| 4B | 2560 | 174.5 | 129.2 | **1.35x** |
| 8B | 4096 | 104.3 | 91.1 | **1.15x** |

Vulkan tg 闂冭埖顔屾稉鈧懛鏉戞勾**韫囶偂绨?* HIP, 鐠?issue #26663 閹躲儱鎲￠惃?5-7x 閹?鐎瑰苯鍙忛崣宥呮倻閵?閸欘垵鍏橀崢鐔锋礈: b10298 娑斿澧?commit + 閺?Adrenalin 25.x 妞瑰崬濮╃紒鍕値閹靛秵婀?5-7x 閹? b10298 娣囶喕绨℃径褔鍎撮崚鍡愨偓?
#### 閸欐垹骞?#2: Vulkan pp 闂冭埖顔岄崷?8B / pp512 閺冭泛绱戞慨瀣儰閸?HIP (1.46x)

| model | backend | pp128 | pp512 |
|---|---|---:|---:|
| 4B | vulkan | 4113 | 5223 |
| 4B | hip | 4072 | 6671 |
| 8B | vulkan | 2719 | 2866 |
| 8B | hip | 2908 | 4199 |

**瑜般垺鈧?*: 8B pp512 V/H = 0.68 (V 閹?1.46x)閵?*鏉╂瑨绐￠悽銊﹀煕閻?"hidden閳?096 閹? 瑜般垺鈧椒绔撮懛?*, 娴ｅ棙鍙冮惃鍕偓宥嗘殶閺?1.46x 娑撳秵妲?5-7x閵?
#### 閸欐垹骞?#3: MUL_MAT_VEC (n=1, decode) 閹?200-375x (缂佹挻鐎幀? 鐠?backend 閺冪姴鍙?

8B PERF (GGML_VK_PERF_LOGGER=1):
- MUL_MAT q4_K m=4096 n=128 k=4096: 42.8 TFLOPS
- MUL_MAT_VEC q4_K m=4096 n=1 k=4096: 1.2 TFLOPS (**37x drop**)
- MUL_MAT_VEC q4_K m=12288 n=1 k=4096: 0.117 TFLOPS (**375x drop**)

n=1 娑撳秷鍏?amortize tile load, 閸欘亣鍏樼挧浼粹偓姘辨暏 shader閵?*鏉╂瑦妲哥紒鎾寸€梽鎰煑, 鐠?backend 閺冪姴鍙?* (HIP 娑旂喐婀? 娴?hipblaslt 娴兼ê瀵查弴鏉戙偨)閵?
#### 閸欐垹骞?#4: 娴狅絿鐖?bug - RDNA4 鐠囶垵鐦戦崚?(CONFIRMED, 閻喎鐡ㄩ崷?

`llama.cpp/ggml/src/ggml-vulkan/ggml-vulkan.cpp`:

1. **L450**: `get_device_architecture()` 濞?RDNA4 閸掑棙鏁? RDNA4 (gfx1201) fall through 閸?`AMD_RDNA2` default
2. **L18797**: KHR_coopmat 鐠侯垰绶?`arch == AMD_RDNA3` 閹?enable (workaround for AMD driver false-positive), RDNA4 鐞氼偄鍙?3. **L4076-4091**: `gpu_pipeline_configs` 閺佹壆绮嶅▽?RDNA3/RDNA4, `get_subgroup_size()` 鏉╂柨娲?0, 闁偓閸?driver 閹躲儳娈?`subgroupSize=64` (闂?RDNA4 native 32)

**婢跺秴鎮庨弫鍫濈安**: RDNA4 閸?KHR_coopmat + 閻?wave64 (閼板矂娼?wave32 native) 閳?濞搭亣鍨傞幀褑鍏橀妴?
#### 閹绘劘顔?fix (閺堫亝鏁? 閻ｆ瑥鍩?PR 闂冭埖顔?

1. enum 閸?`AMD_RDNA4`
2. `get_device_architecture()` 閸?RDNA4 閸掑棙鏁?(e.g. `props.deviceID == 0x1201` 閹?RDNA4-specific dot-product property)
3. `gpu_pipeline_configs` 閸?RDNA4 entry, 閻?`RDNA_DEFAULT_SUBGROUP_SIZE = 32`
4. L18797 `arch == RDNA3` 閺€?`>= RDNA3` (or include RDNA4)

妫板嫯顓?5-10 鐞涘奔鍞惍? 1st build ~10 min, 妤犲矁鐦?~30 min閵?
### 娴溿倓绮悧鈺冨Ц閹?
- [x] `data/vulkan_vs_hip.csv` (4 濡€崇€? 14B pending)
- [x] `data/vulkan_vs_hip_repro.csv` (4 濡€崇€?+ pp128/pp512/PERF env)
- [x] `logs/perf_4b_vklogger_stderr.log` (4B PERF)
- [x] `logs/perf_8b_vklogger_stderr.log` (8B PERF)
- [x] `docs/mavis-plan.md` (2nd session 閼奉亙瀵岀憴鍕灊)
- [x] `docs/decisions.md` (12 閺夆€冲枀缁?
- [x] `docs/issue_comment_body.md` (鐎瑰本鏆?comment 閺傚洦婀? ready to post)
- [x] `docs/MANUAL_POST_INSTRUCTIONS.md` (閻劍鍩涢幍瀣З鐠愬瓨瀵氶崡?
- [x] `ROOT_CAUSE.md` (娴狅絿鐖?bug 鐠囷箒袙)
- [x] `FINAL.md` (鐠囧﹥鏌囩紒鎾寸亯濮瑰洦鈧?
- [ ] Issue comment POST (token 閺夊啴妾洪梽鎰煑, 闁偓鐠?PLAN B)
- [ ] 14B 閺佺増宓?(download 鏉╂稖顢戞稉? ~5 min)

### Token 閺夊啴妾洪梻顕€顣?
- fine-grained PAT `a fine-grained personal access token` 閸?`66666kbit/*` 閺?admin
- 鐎?`ggml-org/llama.cpp` 閸欘亝婀?`pull` (verified via `GET /repos/...` `permissions: {pull: true, push: false}`)
- 鐠?repo write 闂団偓鐟?explicit grant (PROMPT 閸愭瑩鏁? "admin 閺夊啴妾?閺勵垳鏁ら幋鐤殰瀹稿彉绮ㄩ惃?admin)
- 闁偓鐠? 閻?local copy + `MANUAL_POST_INSTRUCTIONS.md` 3 min 閹靛濮╃拹?
### 閸愬磭鐡?(閹芥顩?

12 閺夆€冲枀缁涙牕鍙忛崷?`docs/decisions.md`, 閸忔娊鏁?
- D-6: 閺佺増宓侀崣宥呮倻 issue 閹躲儱鎲? 娑撳秴宸辩悰灞筋殰, 閹稿鏆熼幑顔煎晸
- D-7: 娣?bench 閼存碍婀?None.get 瀹曗晜绨?- D-8: 14B download 閸椻剝顒存径姘偧, inline 闁插秴鎯?- D-9: PERF env 閺€鍦暏 `GGML_VK_PERF_LOGGER` (娑撳秵妲?`GGML_VULKAN_PERF`)
- D-10: clone llama.cpp 濠ф劗鐖滈惇?RDNA 鐠囧棗鍩?- D-11: 閹垫儳鍩岄弽鐟版礈娴狅絿鐖?bug, **濞屸剝鏁兼禒锝囩垳**, 缁?issue 鐠囧嫯顔戦崣宥夘洯
- D-12: 14B 缁涘绶? 閹?comment 閺冨墎鏁?4 濡€崇€烽弫鐗堝祦

### 閼奉亙瀵岄弮鍫曟？ (2nd session, 閹搭亣鍤?12:48)

- Setup: 3 min
- 4 濡€崇€?bench: 1 min (30 sec 鐠烘垵鐣?
- bench 閼存碍婀版穱?+ 14B 闁插秳绗? 5 min
- pp128/pp512 + PERF 閸戝棗顦? 7 min
- 濠ф劗鐖滄稉?(36MB) + 鐟欙絽甯? 5 min
- 濠ф劗鐖滈崚鍡樼€?(RDNA 鐠囧棗鍩?/ coopmat): 8 min
- 閸?ROOT_CAUSE / FINAL / decisions / comment body: 5 min
- Token 403 閹烘帗鐓?+ PLAN B 閸? 5 min
- **閹? ~39 min (12:24-13:03)** 閳?鏉╂ɑ婀?9h+ buffer, 14B 閸︺劏绐? 閺€璺虹啲娑?