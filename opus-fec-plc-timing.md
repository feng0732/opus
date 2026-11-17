# Opus 解码器中 FEC 与 PLC 如何推进解码时间

本文分析本仓库（libopus）中丢包隐藏（PLC）与带内前向纠错（FEC）在**解码时间轴**上的行为：每次调用产生多少采样、解码器内部状态如何随丢包演化、以及应用层的"下一包时间戳"应如何推进。所有结论均附源码引用，并附 API 测试与实测程序证据（不涉及音质跑分）。

---

## 1. 丢包与延迟恢复对采样数及下一包时间戳的影响

### 1.1 基本契约：丢失多少时间，就必须请求多少采样

Opus 解码器**不维护任何内部时间戳**。时间轴完全由应用层（抖动缓冲区）管理：每次 `opus_decode*()` 返回的采样数就是调用者应当向前推进的播放时间。因此丢包时 API 的契约是"丢失多少音频就补多少采样"：

> "In the case of PLC (data==NULL) or FEC (decode_fec=1), then frame_size needs to be **exactly the duration of audio that is missing**, otherwise the decoder will not be in the optimal state to decode the next incoming packet. For the PLC and FEC cases, frame_size **must** be a multiple of 2.5 ms."
> — `include/opus.h:508-511`（`opus_decode` 文档，另两处相同文案见 `:533-536`、`:558-561`）

实现上的强制检查在 `src/opus_decoder.c:745-747`：

```c
/* For FEC/PLC, frame_size has to be to have a multiple of 2.5 ms */
if ((decode_fec || len==0 || data==NULL) && frame_size%(st->Fs/400)!=0)
   return OPUS_BAD_ARG;
```

### 1.2 PLC（`data == NULL` 或 `len == 0`）：输出恰好 `frame_size` 个采样

`opus_decode_native()` 对空包进入一个循环，反复调用 `opus_decode_frame(st, NULL, 0, ...)` 直到填满 `frame_size`，并把结果记入 `st->last_packet_duration`（`src/opus_decoder.c:778-792`）：

```c
if (len==0 || data==NULL)
{
   int pcm_count=0;
   do {
      ret = opus_decode_frame(st, NULL, 0, pcm+pcm_count*st->channels, frame_size-pcm_count, 0 ...);
      ...
      pcm_count += ret;
   } while (pcm_count < frame_size);
   st->last_packet_duration = pcm_count;
   return pcm_count;
}
```

要点：

- **返回采样数恒等于请求的 `frame_size`**，与丢失包的原始时长无关——应用层说丢了多少就补多少，下一包的时间戳 = 当前时间戳 + `frame_size`。
- 单帧 PLC 只支持 2.5/5/10/20 ms 粒度，更大的 `frame_size` 会被拆成 ≤20 ms 的多次 PLC 调用（`src/opus_decoder.c:343-365`），所以一次调用掩盖 40 ms、60 ms 等任意 2.5 ms 倍数时长都可以。
- `len<=1`（含 ToC 共 ≤2 字节）的包也触发 PLC/DTX，且此时掩盖时长被钳制到 ToC 声明的帧长："don't conceal more than what the ToC says"（`src/opus_decoder.c:315-321`）。
- 全新解码器（还没有任何模式历史）上做 PLC 直接输出零（`src/opus_decoder.c:334-341`："If we haven't got any packet yet, all we can do is return zeros"）。
- `OPUS_GET_LAST_PACKET_DURATION` 可取回上次调用产生的采样数（`src/opus_decoder.c:791`），供应用层校准时间轴。

### 1.3 FEC（`decode_fec = 1`）：用"下一包"恢复"丢失包"，时间戳不漂移

带内 FEC 的工作方式是**延迟一个包恢复**：包 N 丢失，等包 N+1 到达后，用 `decode_fec=1` 解码包 N+1，**输出的是包 N 的音频**。其实现分两段（`src/opus_decoder.c:813-847`）：

```c
if (decode_fec)
{
   /* If no FEC can be present, run the PLC (recursive call) */
   if (frame_size < packet_frame_size || packet_mode == MODE_CELT_ONLY || st->mode == MODE_CELT_ONLY)
      return opus_decode_native(st, NULL, 0, pcm, frame_size, 0, ...);   /* 退化为 PLC */
   /* Otherwise, run the PLC on everything except the size for which we might have FEC */
   if (frame_size-packet_frame_size!=0)
      ret = opus_decode_native(st, NULL, 0, pcm, frame_size-packet_frame_size, 0, ...);  /* 前段 PLC */
   /* Complete with FEC */
   ...
   ret = opus_decode_frame(st, data, size[0], pcm+st->channels*(frame_size-packet_frame_size),
         packet_frame_size, 1 ...);                                       /* 后段 FEC */
   st->last_packet_duration = frame_size;
   return frame_size;
}
```

对时间轴的含义：

- 本次调用总共仍输出 **`frame_size` 个采样**（= 丢失的时长）：前 `frame_size - packet_frame_size` 用 PLC 填，最后 `packet_frame_size`（当前包一帧的时长）用 FEC 数据填。`frame_size` 只覆盖**第一个**丢失帧（`size[0]`），多帧丢失的其余部分仍靠 PLC。
- 包 N+1 的载荷在这次调用中**只被消费了 FEC 部分**；包 N+1 自身的音频要在**下一次**用 `decode_fec=0` 正常解码同一个包得到。因此应用层的时间戳序列是：丢包槽位 → FEC 调用补 `frame_size`；下一槽位 → 正常解码包 N+1。时间轴严格按采样数推进，**不丢不错位**。
- 若 `frame_size` 与真实丢失时长不符，解码器内部预测状态（SILK 的 LTP/LPC 历史、CELT 的 MDCT 重叠缓冲）推进的量就与码流错位，下一包解码质量受损——这就是 opus.h 文档警告 "not be in the optimal state" 的原因。

---

## 2. 何时使用 FEC，以及连续丢包如何影响解码器状态

### 2.1 FEC 的适用条件

- **只有 SILK 和混合（Hybrid）模式携带带内 FEC**（SILK LBRR，低码率冗余）。编码端由 `OPUS_SET_INBAND_FEC` 开启，并按 `OPUS_SET_PACKET_LOSS_PERC` 告知预期丢包率（`include/opus.h:185` 及 opus_defines.h 中对应 CTL）。
- 解码端在以下任一情况下 `decode_fec=1` **退化为纯 PLC**（`src/opus_decoder.c:818-819`）：
  1. 当前包是 CELT-only 模式（`packet_mode == MODE_CELT_ONLY`）；
  2. 解码器上一模式是 CELT-only（`st->mode == MODE_CELT_ONLY`）；
  3. 请求的 `frame_size` 小于当前包一帧的采样数。
- SILK 侧通过包头的 LBRR 标志判断 FEC 数据是否真的存在：`silk_Decode()` 先解 `LBRR_flag`/`LBRR_flags[]`（`silk/dec_API.c:245-267`）；只有当 `lostFlag == FLAG_DECODE_LBRR` 且对应帧的 `LBRR_flags[i] == 1` 时才真正走 FEC 解码，否则仍落入 PLC（`silk/decode_frame.c:70-71` 与 `:133`）。
- Opus 层用 `lost_flag` 区分三种情形（`src/opus_decoder.c:469`）：

```c
lost_flag = data == NULL ? 1 : 2 * !!decode_fec;   /* 0=正常, 1=PLC, 2=FEC */
```

### 2.2 连续丢包对 SILK 解码器状态的影响

每丢一帧，`silk_PLC()` 走 `silk_PLC_conceal()` 并递增丢失计数（`silk/PLC.c:89-99`）：

```c
if( lost ) {
    silk_PLC_conceal( psDec, psDecCtrl, frame, ... );
    psDec->lossCnt++;
} else {
    silk_PLC_update( psDec, psDecCtrl );   /* 正常帧：更新 PLC 参考状态 */
}
```

`lossCnt` 直接索引衰减表（`silk/PLC.c:275-279`，表定义在 `:40-43`）：

```c
#define NB_ATT 2
static const opus_int16 HARM_ATT_Q15[NB_ATT]              = { 32440, 31130 }; /* 0.99, 0.95 */
static const opus_int16 PLC_RAND_ATTENUATE_V_Q15[NB_ATT]  = { 31130, 26214 }; /* 0.95, 0.8  */
static const opus_int16 PLC_RAND_ATTENUATE_UV_Q15[NB_ATT] = { 32440, 29491 }; /* 0.99, 0.9  */
...
harm_Gain_Q15 = HARM_ATT_Q15[ silk_min_int( NB_ATT - 1, psDec->lossCnt ) ];
```

即第 1 个丢失帧按 0.99/0.95 衰减，**从第 2 个连续丢失帧起**改用更强的 0.95/0.8（浊音噪声分量）衰减，激励能量逐帧指数式衰减，避免长时丢包产生持续伪音。此外丢包时解除增益钳制，防止能量"反弹"（`silk/dec_API.c:478-482`）：

```c
if( lostFlag == FLAG_PACKET_LOST ) {
   /* On packet loss, remove the gain clamping to prevent having the energy "bounce back" ... */
   for ( i = 0; i < psDec->nChannelsInternal; i++ )
      psDec->channel_state[ i ].LastGainIndex = 10;
}
```

### 2.3 连续丢包对 CELT 解码器状态的影响

CELT 侧每次丢失调用 `celt_decode_lost()`，帧末累积丢失时长（`celt/celt_decoder.c:1186-1194`）：

```c
st->loss_duration = IMIN(10000, loss_duration+(1<<LM));
st->plc_duration  = IMIN(10000, st->plc_duration+(1<<LM));
st->last_frame_type = curr_frame_type;
```

帧类型随连续丢失演化（`celt/celt_decoder.c:747-756`）：

- 初始为**基音外推型 PLC**（`FRAME_PLC_PERIODIC`）；
- `plc_duration >= 40`（约 40 个 2.5 ms 单位，即持续丢包）或处于混合模式 SILK 段（`start != 0`）或 `skip_plc` 置位时，切换为**噪声型 PLC/CNG**（`FRAME_PLC_NOISE`），并按丢失时长加大能量衰减（`:786-793`：`decay = (1<<LM)*0.0625`，首帧额外加上 `min(160, loss_duration)*0.0625`）；
- 启用 Deep PLC 且复杂度 ≥5 时，前 80 个单位内可用神经 PLC（`FRAME_PLC_NEURAL`，`:752-756`）。

噪声型 PLC 会置 `st->skip_plc = 1`（`:834-835`），此后**必须连续收到两个正常包**才重新允许基音型 PLC（`:1402-1404`：`if (st->loss_duration == 0) st->skip_plc = 0;`——`loss_duration` 在正常解码成功时清零，`:1706-1708`）。

丢包后的**第一个正常包**还有专门的保护措施：若该包使用能量预测（非 intra），解码器会把预测参考能量向下修正，修正量随 `loss_duration` 增大（最多计 10 帧），防止丢失期间状态外推误差引发爆音（`celt/celt_decoder.c:1467-1497`："make sure the energy prediction safe to reduce the risk of getting loud artifacts"）。

### 2.4 模式切换与丢包的交互

丢包还会推迟模式切换的过渡处理：当新包模式与 `prev_mode` 不同（SILK↔CELT），解码器会递归调用一次 5 ms PLC 生成过渡音频做 `smooth_fade` 交叉淡化（`src/opus_decoder.c:374-391`、`:550-553`、`:671-690`）；若过渡中的冗余帧（5 ms redundancy）因丢包缺失，代码注释明确说明 "the first redundancy frame in a transition from SILK may have been lost"，此时冗余音频被解码但弃用（`:588-600`、`:658-670`）。

---

## 3. 三种丢包场景：采样数与状态变化（含实测证据）

测试方法：用本仓库构建的 libopus（cmake Release 构建，`test_opus_decode` 全部通过），编码 8 个 20 ms SILK 包（`OPUS_SET_INBAND_FEC(1)`、`OPUS_SET_PACKET_LOSS_PERC(20)`、强制 SILK 模式、24 kb/s、48 kHz 单声道、440 Hz 正弦），分别演示三种场景。48 kHz 下 20 ms = **960 采样**。

### 3.1 场景一：孤立丢包（纯 PLC）

序列：解码 pkt0、pkt1 → pkt2 丢失，`opus_decode(dec, NULL, 0, out, 960, 0)` → 正常解码 pkt3。

实测输出：

```
A: decode pkt0          -> 960 samples
A: decode pkt1          -> 960 samples
A: PLC for lost pkt2    -> 960 samples (last_packet_duration=960, rms=5981.3)
A: decode pkt3 (normal) -> 960 samples
```

- 采样数：PLC 调用返回**恰好 960**（= 请求丢失的 20 ms），`last_packet_duration` 同步为 960。下一包 pkt3 的时间戳 = pkt1 时间戳 + 2×960，时间轴无缺口。
- 状态变化：SILK `lossCnt` 0→1（`silk/PLC.c:99`），按第一档衰减（0.99/0.95）外推一帧；pkt3 到达后 `silk_PLC_update()` 恢复正常参考状态。
- API 测试佐证：`tests/test_opus_decode.c:111-115` 验证全新解码器上 PLC 返回 `120/factor`（2.5 ms）采样且 `OPUS_GET_LAST_PACKET_DURATION` 一致；`:117-119` 验证非 2.5 ms 倍数的 `frame_size` 返回 `OPUS_BAD_ARG`。

### 3.2 场景二：孤立丢包，FEC 可恢复

序列：解码 pkt0、pkt1 → pkt2 丢失 → pkt3 到达后以 `decode_fec=1` 解码（产出 **pkt2** 的音频）→ 再次以 `decode_fec=0` 解码 pkt3（产出 pkt3 自身）。

实测输出：

```
B: FEC decode of pkt3   -> 960 samples (recovers lost pkt2, last_packet_duration=960, rms=6999.7)
B: decode pkt3 (normal) -> 960 samples
```

- 采样数：FEC 调用返回 960（= 丢失帧时长；若 `frame_size > packet_frame_size`，超出部分由 PLC 补齐，见 `src/opus_decoder.c:822-831`）。
- 状态变化：`lost_flag=2`（`src/opus_decoder.c:469`），SILK 走 `FLAG_DECODE_LBRR` 路径，用 pkt3 中的 LBRR 数据重建 pkt2——`lossCnt` **不递增**，解码器 LTP/LPC 状态由真实（低码率）参数更新，比 PLC 的外推状态更接近编码端。作为参照：真实信号 RMS≈7071，FEC 恢复帧 RMS=6999.7，而同位置纯 PLC 帧 RMS=5981.3（仅作状态一致性佐证，非音质评价）。
- 注意 pkt3 被解码**两次**：第一次只取 FEC，第二次才取主码流。应用层时间戳仍按每次返回值顺序推进。
- API 测试佐证：`tests/test_opus_decode.c:105-166` 对 `fec=0/1` 两种标志遍历全部边界行为（含 `:160-162` "Crazy FEC values"：`decode_fec=-1` 或 `2` 必须返回错误码，对应 `src/opus_decoder.c:743-744` 的 `decode_fec<0 || decode_fec>1` 检查）。

### 3.3 场景三：连续丢包

序列：解码 pkt0、pkt1 → 连续 3 次 PLC（各 20 ms）→ 再一次 40 ms 的 PLC → 正常解码 pkt6。

实测输出：

```
C: PLC loss #1          -> 960 samples, rms=5981.3
C: PLC loss #2          -> 960 samples, rms=2718.0
C: PLC loss #3          -> 960 samples, rms=416.0
C: PLC 40 ms in one call -> 1920 samples (last_packet_duration=1920)
C: decode pkt6 (normal)  -> 960 samples
```

- 采样数：每次 PLC 调用都返回**请求的** `frame_size`（960、960、960、1920），与丢包个数一一对应；一次调用掩盖多个包时长时内部拆成 ≤20 ms 的多次帧级 PLC（`src/opus_decoder.c:778-787` 与 `:345-358`）。应用层为每个丢失包推进 960 采样的时间戳。
- 状态变化：`lossCnt` 逐帧递增（1→2→3…），SILK 从第二帧起切换到强衰减档（`PLC_RAND_ATTENUATE_V_Q15[1]=0.8` 等，`silk/PLC.c:275-279`），输出能量快速收敛（实测 RMS 5981→2718→416）；CELT 模式下则对应 `loss_duration`/`plc_duration` 累积与周期型→噪声型 PLC 的切换（`celt/celt_decoder.c:750-751`、`:1186-1187`）。丢包结束后的首个正常包触发能量预测保护（`celt/celt_decoder.c:1469-1497`）与 SILK 增益钳制解除的恢复。
- API 测试佐证：`tests/test_opus_decode.c:204-216` 对全部 64 种 ToC 模式**连续跑 6 帧 PLC**（"run for 6 frames in order to get better PLC coverage"），每帧都校验返回采样数与 `OPUS_GET_LAST_PACKET_DURATION` 一致；`:217-225` 再补一次 2.5 ms 的 PLC，模拟应用层做微小时钟漂移修正的场景。

### 3.4 附：API 契约边界（实测）

```
D: PLC frame_size=962 (not 2.5ms multiple) -> -1 (OPUS_BAD_ARG=-1)
D: PLC on fresh decoder  -> 960 samples, rms=0.0 (zeros)
D: decode_fec=1 on CELT packet -> 960 samples (PLC fallback)
```

分别对应：`src/opus_decoder.c:746`（2.5 ms 倍数检查）、`:334-341`（无历史状态输出零）、`:818-819`（CELT 包 FEC 退化为 PLC）。

---

## 4. 小结

| 场景 | 调用方式 | 返回采样数 | 时间戳推进 | 解码器状态 |
|---|---|---|---|---|
| 孤立丢包 | `opus_decode(dec, NULL, 0, out, 丢失时长, 0)` | = `frame_size` | +`frame_size` | SILK `lossCnt`+1 / CELT `loss_duration` 累积，外推一帧 |
| FEC 恢复 | 丢包后下一包 `decode_fec=1` | = `frame_size`（丢失时长） | +`frame_size`；该包之后还需正常解码一次 | `lost_flag=2`，用 LBRR 数据更新状态，`lossCnt` 不增 |
| 连续丢包 | 每包一次 PLC（或一次掩盖多包时长） | 每次 = `frame_size` | 每包 +其时长 | 衰减逐帧加深；CELT 周期型→噪声型 PLC；恢复后首包有能量保护 |

核心原则：**解码器按调用者声明的丢失时长产出采样，时间轴由应用层按返回采样数推进；FEC 只是用下一包的冗余把"外推"换成"低码率重建"，不改变采样计数规则。**

---

### 附：复现方式

```sh
cmake -S <repo> -B /tmp/opus-build -DCMAKE_BUILD_TYPE=Release -DOPUS_BUILD_TESTING=ON
cmake --build /tmp/opus-build --target opus test_opus_decode -j
/tmp/opus-build/test_opus_decode        # 官方 API/PLC/FEC 测试，全部通过
# 场景实测程序：/tmp/opus-fec-test/fec_plc_timing.c（链接上述 libopus）
```
