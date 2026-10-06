# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` � host `Windows-AMD64` � llama.cpp `b10488`
CPU: **8 physical � 16 logical** cores � `ngl=99` � metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 35.4 | 95% |
| 4 | 34.3 | 92% |
| 8 | 36.5 | 98% |
| 16 | 37.1 | 100% |
| 32 | 37.1 | 100% |

**Best**: `-t 16` at 37.1 tok/s
**Slowest tested**: `-t 4` at 34.3 tok/s (1.08x spread)
**Against the physical-core default** (`-t 8`, 36.5 tok/s): 1.02x

Use this in your run:

```bash
LAB_N_THREADS=16 make bench
```

## Your explanation

_Where is the knee, and why there? If the peak sits at your physical core count
and drops above it, say what the extra threads are competing for. If your curve
does something else -- flat, or still climbing at 2x logical cores -- say that
instead and reason about why. A result that contradicts the expected shape is
worth more than one that matches it, as long as you explain it._

D&#432;&#7899;i &#273;�y l� n&#7897;i dung m&#7851;u &#273;&#7875; b&#7841;n &#273;i&#7873;n v�o m&#7909;c **`## Your explanation`**:

### Knee n&#7857;m &#7903; &#273;�u v� v� sao &#7903; &#273;�?

* **V&#7883; tr� c&#7911;a "knee" (&#273;i&#7875;m b�o h�a / u&#7889;n cong):** N&#7857;m t&#7841;i **`-t 8`** (s&#7889; nh�n v&#7853;t l� - physical cores) ho&#7863;c k�o d�i &#273;&#7871;n t&#7889;i &#273;a **`-t 16`** (s&#7889; lu&#7891;ng logic - logical cores). Tuy nhi�n, tr�n th&#7921;c t&#7871; to�n b&#7897; &#273;&#432;&#7901;ng cong hi&#7879;u n&#259;ng g&#7847;n nh&#432; **ph&#7859;ng l� (flat)** tr�n to�n d&#7843;i t&#7915; 1 &#273;&#7871;n 32 threads (ch�nh l&#7879;ch gi&#7919;a &#273;i&#7875;m th&#7845;p nh&#7845;t v� cao nh&#7845;t ch&#7881; v&#7887;n v&#7865;n ~1.08x, t&#7915; 34.3 l�n 37.1 tok/s).
* **V� sao l&#7841;i nh&#432; v&#7853;y?** Do c&#7901; **`ngl=99`** &#273;� offload to�n b&#7897; c�c layer t�nh to�n sang **GPU**. Khi to�n b&#7897; ma tr&#7853;n &#273;&#432;&#7907;c GPU x&#7917; l�, vai tr� c&#7911;a CPU ch&#7881; gi&#7899;i h&#7841;n &#7903; vi&#7879;c &#273;i&#7873;u ph&#7889;i, dispatch kernel v� x&#7917; l� sampling nh&#7865;; CPU kh�ng c�n l� n�t th&#7855;t c&#7893; chai (bottleneck) t�nh to�n. Do &#273;�, vi&#7879;c t&#259;ng s&#7889; l&#432;&#7907;ng CPU threads g&#7847;n nh&#432; kh�ng mang l&#7841;i c&#7843;i thi&#7879;n &#273;�ng k&#7875; v&#7873; t&#7889;c &#273;&#7897; sinh t&#7915; (`tg128`).

### C�c thread th&#7915;a tranh ch&#7845;p t�i nguy�n g�?

Khi s&#7889; thread v&#432;&#7907;t qu� s&#7889; core v&#7853;t l� (t&#7915; 16 &#273;&#7871;n 32 threads), c�c thread th&#7915;a kh�ng t&#259;ng th�m th�ng l&#432;&#7907;ng m� ph&#7843;i c&#7841;nh tranh gay g&#7855;t v&#7873; **b&#259;ng th�ng b&#7897; nh&#7899; L3 cache, c�c &#273;&#417;n v&#7883; th&#7921;c thi chia s&#7867; tr�n c�ng m&#7897;t core v&#7853;t l� (Hyper-Threading / SMT)**, &#273;&#7891;ng th&#7901;i ch&#7883;u t&#7893;n th&#7845;t do **overhead chuy&#7875;n ng&#7919; c&#7843;nh (context switching) v� tranh ch&#7845;p kh�a &#273;i&#7873;u ph&#7889;i (thread synchronization/lock contention)** c&#7911;a h&#7879; &#273;i&#7873;u h�nh.
