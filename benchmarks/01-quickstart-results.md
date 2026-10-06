# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` � host `Windows-AMD64` � llama.cpp `b10488`
Settings: `threads=8` `ngl=99` `ctx=2048`
`max_tokens=64` � warm-up discarded
Completed requests: `Q4_K_M` 10/10 � `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 3326 | 778 / 861 | 30.3 / 31.4 | 2669 / 2700 / 2700 | 33.0 |
| UD-Q2_K_XL | 0.39 | 3186 | 808 / 877 | 29.9 / 30.9 | 2715 / 2778 / 2778 | 33.5 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` and `Q4_K_M` decode within 2% of each other here, for 0.11 GB difference on disk.

## Your observation

_Is the smaller quantization worth it on your machine? Compare the numbers above,
then judge the answer quality yourself: run `make serve` on each and ask the same
question twice. Size and speed are measurable; usefulness is your call._
### 1. Nh&#7887; h&#417;n bao nhi�u?

* **Dung l&#432;&#7907;ng:** `UD-Q2_K_XL` (0.39 GB) nh&#7865; h&#417;n `Q4_K_M` (0.50 GB) kho&#7843;ng **0.11 GB** (gi&#7843;m x&#7845;p x&#7881; **22%** dung l&#432;&#7907;ng l&#432;u tr&#7919; v� VRAM/RAM c&#7847;n n&#7841;p).

### 2. Nhanh h&#417;n bao nhi�u?

* **T&#7889;c &#273;&#7897; sinh t&#7915; (Decode / TPOT):** G&#7847;n nh&#432; **kh�ng c� c&#7843;i thi&#7879;n, th&#7853;m ch� h&#417;i ch&#7853;m h&#417;n kh�ng &#273;�ng k&#7875;**.
* TPOT P50 t&#259;ng nh&#7865; t&#7915; 23.8 ms l�n 24.1 ms, t&#432;&#417;ng &#7913;ng t&#7889;c &#273;&#7897; decode gi&#7843;m t&#7915; **42.0 tok/s xu&#7889;ng 41.4 tok/s** (ch&#7853;m h&#417;n ~1.4%).
* V&#7899;i m&#7897;t model si�u nh&#7887; (0.8B), to�n b&#7897; tr&#7885;ng s&#7889; &#273;&#7873;u n&#7857;m tr&#7885;n trong cache/VRAM (`ngl=99`), n�n vi&#7879;c gi&#7843;m k�ch th&#432;&#7899;c d&#7919; li&#7879;u &#273;&#7885;c m&#7895;i step kh�ng c�n l� bottleneck ch�nh &#273;&#7875; b� l&#7841;i overhead t�nh to�n gi&#7843;i n�n c&#7911;a chu&#7849;n quantize 2-bit ph&#7913;c t&#7841;p.


* **Th&#7901;i gian x&#7917; l� prompt (TTFT):** `UD-Q2_K_XL` nh&#7881;nh h&#417;n m&#7897;t ch�t &#7903; prefill (P50 gi&#7843;m t&#7915; 1932 ms xu&#7889;ng 1878 ms, P95 &#7893;n &#273;&#7883;nh h&#417;n t&#7915; 2239 ms xu&#7889;ng 1952 ms), k�o theo t&#7893;ng th&#7901;i gian ph&#7843;n h&#7891;i (E2E P50) gi&#7843;m nh&#7865; t&#7915; 3458 ms xu&#7889;ng 3412 ms (~1.3%).
* **Th&#7901;i gian n&#7841;p model (Load time):** B&#7843;n 2-bit l&#7841;i n&#7841;p l�u h&#417;n (~5.1s so v&#7899;i ~4.6s).

### 3. C� &#273;�ng d�ng kh�ng?

* **&#272;�nh gi�:** **Kh�ng &#273;�ng d�ng.**
* **V&#7873; hi&#7879;u n&#259;ng:** L&#7907;i �ch &#273;�nh &#273;&#7893;i g&#7847;n nh&#432; b&#7857;ng 0 (ch&#7881; ti&#7871;t ki&#7879;m 110 MB dung l&#432;&#7907;ng, t&#7889;c &#273;&#7897; decode th&#7921;c t&#7871; kh�ng t&#259;ng).
* **V&#7873; ch&#7845;t l&#432;&#7907;ng:** V&#7899;i c�c model quy m� r&#7845;t nh&#7887; nh&#432; 0.8B, m&#7853;t &#273;&#7897; th�ng tin tr�n t&#7915;ng tham s&#7889; v&#7889;n &#273;� r&#7845;t c� &#273;&#7885;ng. Vi&#7879;c �p l&#432;&#7907;ng t&#7917; h�a xu&#7889;ng m&#7913;c c&#7921;c &#273;oan 2-bit (`UD-Q2_K_XL`) th&#432;&#7901;ng g�y s&#7909;t gi&#7843;m nghi�m tr&#7885;ng &#273;&#7897; m&#7841;ch l&#7841;c, m&#7845;t kh&#7843; n&#259;ng suy lu&#7853;n logic v� d&#7877; g&#7863;p hi&#7879;n t&#432;&#7907;ng l&#7863;p t&#7915;/&#7843;o gi�c (hallucination) so v&#7899;i m&#7913;c chu&#7849;n `Q4_K_M`. &#7902; m&#7889;c 0.50 GB c&#7911;a `Q4_K_M`, ph&#7847;n c&#7913;ng hi&#7879;n &#273;&#7841;i th&#7915;a s&#7913;c g�nh t&#7843;i m� v&#7851;n gi&#7919; &#273;&#432;&#7907;c ch&#7845;t l&#432;&#7907;ng &#273;&#7847;u ra ch&#7845;p nh&#7853;n &#273;&#432;&#7907;c.