# Manual Post Instructions 閳?issue #26663 comment

> Mavis 閼奉亙瀵屾禒璇插閸?**token 閺夊啴妾洪梽鎰煑閺冪姵纭堕懛顏勫З POST**, 鐠囬鏁ら幋閿嬪閸斻劏鍒涢妴?> 閸忋劍鏋冨?ready, 婢跺秴鍩楃划妯垮垱閸楀啿褰查妴?
## 闁插秷顩﹂崢鐔锋礈鐠囧瓨妲?
- 閹存垹鏁ら惃?fine-grained PAT (`a fine-grained personal access token`) 閸?`66666kbit/*` 娴犳挻妲?admin, 娴?*鐎电懓顦婚柈銊ょ波 `ggml-org/llama.cpp` 閸欘亝婀?`pull` 閺夊啴妾?* (verified: `GET /repos/ggml-org/llama.cpp` 鏉╂柨娲?`permissions: {pull: true, push: false, triage: false, maintain: false, admin: false}`)閵?- 鐠?repo write 闂団偓鐟?explicit grant, fine-grained PAT 姒涙顓绘稉宥囩舶閵?- 闁偓閸?plan B (閹?PROMPT 闁偓鐠?: 閸愭瑥銈介張顒€婀?+ 閻?manual post 閹稿洤宕￠妴?
## 閻劍鍩涢幙宥勭稊 (3 min)

### Step 1: 閹垫挸绱?issue
濞村繗顫嶉崳銊ョ磻: `https://github.com/ggml-org/llama.cpp/issues/26663`

### Step 2: 婢跺秴鍩?comment body
閹垫挸绱戦張顒€婀撮弬鍥︽: `vulkan-slowpath/docs\issue_comment_body.md`

**閸忋劑鈧?(Ctrl+A) 閳?婢跺秴鍩?(Ctrl+C)**

### Step 3: 缁鍒涢獮鑸靛絹娴?- 閸?issue 妞ょ敻娼版惔鏇㈠劥 "Add a comment" 濡?- **缁鍒?(Ctrl+V)**
- 閻?"Comment" 閹稿鎸?
### 鐎瑰本鍨?comment URL 娴兼碍妲?`https://github.com/ggml-org/llama.cpp/issues/26663#issuecomment-<ID>`, 閻劍鍩涚拹鏉戠暚閹?ID 閸欐垵娲栫紒?Mavis, Mavis 閹跺﹤鐣犳繅顐㈠煂:
- `vulkan-slowpath/FINAL.md` (Issue 闁剧偓甯存稉鈧懞?
- `vulkan-slowpath/docs\issue_comment_url.txt` (鏉╂瑤閲滈弬鍥︽閻╊喖澧犳稉宥呯摠閸? 閸掓稑缂撻獮璺猴綖閸?

## 婢跺洨鏁? 閸︺劏鍤滃鍙樼波瀵偓 issue 瀵洜鏁?(婵″倹鐏夋稉宥嗗厒鐠?#26663)

娑旂喎褰叉禒銉ユ躬 `66666kbit/rx9070xt-llm-bench` 娴犳挸绱戞稉鈧稉顏呮煀 issue, 瀵洜鏁?#26663, 闂?ROOT_CAUSE.md 闁剧偓甯撮妴?*鏉╂瑦鐗辨稉宥夋付鐟曚礁顦婚柈銊ょ波閺夊啴妾?*閵嗗倷绲剧粈鎯у隘閸欘垵顫嗛幀褌缍? 娑撳秵甯归懡鎰┾偓?
## 婢跺洨鏁? 缂?token 閸旂姵娼堥梽?(5 min)

婵″倹鐏夐悽銊﹀煕閹版寧鍓?

1. 濞村繗顫嶉崳銊ョ磻 `https://github.com/settings/tokens?type=beta`
2. 閹垫儳鍩岀€电懓绨?token (閸氬秴鐡ч崠鍛儓 `Mavis` / `RDNA4` / `vulkan-slowpath` 缁?
3. Repository access 閳?闁?"Only select repositories"
4. **濞ｈ濮?`ggml-org/llama.cpp` (read + write)**
5. Permissions 閳?Issues: Write
6. 娣囨繂鐡?
**濞夈劍鍓?*: 鏉╂瑤绱扮拋?agent 閼宠棄绶氭禒璁崇秿閻劍鍩涢幒鍫熸綀閻ㄥ嫪绮ㄩ崘? 鐎瑰鍙忛幀褏鏁ら幋鐤殰瀹歌鲸娼堢悰掳鈧?
---

## Comment 閸愬懎顔?(preview)

瀵偓婢?
> ## Diagnostic follow-up 閳?code-side findings, did not fully reproduce the 5閳?鑴?regression on b10298
>
> Tested on the same hardware as the OP...

娑擃厽顔岄張澶婄暚閺?reproduction table (5 濡€崇€?鑴?pp128/pp512/tg128), per-op profile, RDNA4 misdetection code analysis, 閹绘劘顔?fix閵?
缂佹挸鐔?
> Happy to file a PR with the RDNA4 detection fix once the maintainers confirm the direction, or if you'd like me to also include a quick `ggml_vk_khr_cooperative_matrix_support` adjustment to whitelist RDNA4. Just let me know.
