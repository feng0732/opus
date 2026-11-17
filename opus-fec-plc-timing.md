# Opus 解码器中 FEC 与 PLC 对解码时间的推进

本报告分析本仓库（libopus）中前向纠错（FEC）与丢包隐藏（PLC）如何推进 Opus 解码时间：
丢包或延迟恢复时解码器产出多少采样、下一包的时间戳如何对齐、何时应使用 FEC，
以及连续丢包如何改变解码器内部状态。所有结论均引用仓库源码与 API 测试证据，不涉及音质评估。

---

## 1. 丢包与延迟恢复对采样数及下一包时间戳的影响

### 1.1 Opus 解码器不维护时间戳，采样数即时间

Opus 解码器本身没有任何 RTP 时间戳或播放时钟概念；**时间完全由"解码出的采样数"推进**。
应用层每调用一次解码，就把播放时间戳向前推进返回值的采样数。因此丢包时必须用 PLC
"补出"与丢失时长**完全相等**的采样数，否则之后所有音频的时间戳都会整体漂移。

API 文档对此有明确要求（`include/opus.h:506-511`）：

> In the case of PLC (data==NULL) or FEC (decode_fec=1), then frame_size needs to be
> **exactly the duration of audio that is missing**, otherwise the decoder will not be in
> the optimal state to decode the next incoming packet. For the PLC and FEC cases,
> frame_size **must** be a multiple of 2.5 ms.

该约束在代码中强制执行——PLC/FEC 时 `frame_size` 必须是 2.5 ms 的整数倍，否则返回
`OPUS_BAD_ARG`（`src/opus_decoder.c:745-747`）：

```c
/* For FEC/PLC, frame_size has to be to have a multiple of 2.5 ms */
if ((decode_fec || len==0 || data==NULL) && frame_size%(st->Fs/400)!=0)
   return OPUS_BAD_ARG;
```

### 1.2 PLC 路径：产出恰好 frame_size 个采样

当 `data==NULL` 或 `len==0` 时，`opus_decode_native()` 进入 PLC 循环
（`src/opus_decoder.c:778-792`）：反复调用 `opus_decode_frame(st, NULL, 0, ...)` 直到
填满 `frame_size`，并把 `st->last_packet_duration` 设为该值后返回：

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

注意正常解码路径中 `frame_size` 只是缓冲区上限（`opus_decode()` 会用
`opus_decoder_get_nb_samples()` 把它收紧到包的实际时长，`src/opus_decoder.c:924-931`），
而 **PLC/FEC 路径中 `frame_size` 是要生成的确切采样数**——这正是"采样数推进解码时间"
的直接体现。

### 1.3 丢包时长的来源

应用层需要知道丢了多少采样才能正确调用 PLC：

- `opus_decoder_get_nb_samples()`（`src/opus_decoder.c:1353-1357`）解析包头得到包的采样数；
- `OPUS_GET_LAST_PACKET_DURATION`（`src/opus_decoder.c:1193`）返回上一次解码的时长，
  对同码流连续丢包可沿用该值。

### 1.4 SILK 侧的最小时间粒度

SILK 的 PLC 单次不能产生小于 10 ms 的帧，`opus_decode_frame()` 把 payload 尺寸钳到至少
10 ms（`src/opus_decoder.c:421-422`）：

```c
/* The SILK PLC cannot produce frames of less than 10 ms */
st->DecControl.payloadSize_ms = IMAX(10, 1000 * audiosize / st->Fs);
```

并通过 `do { silk_Decode(...); } while( decoded_samples < frame_size )` 循环
（`src/opus_decoder.c:471-494`）累加多个 SILK 帧直到凑满 `frame_size`。SILK 内部按
`payloadSize_ms` 选择每包帧数与子帧数（10/20/40/60 ms → `nFramesPerPacket` 1/1/2/3，
`silk/dec_API.c:200-220`）。

### 1.5 延迟到达的包与"补洞"语义

抖动缓冲中常见的时序问题是：包 *n* 超时未到，已对它做了 PLC（播放时钟已前进
`frame_size` 个采样），随后包 *n+1* 到达。此时有两种选择：

1. **用 FEC 补洞**：以 `decode_fec=1` 解码包 *n+1*，恢复的是**包 *n* 的音频**，
   填入之前 PLC 占掉的时间槽。播放时间戳不再额外前进——FEC 只是"替换"已预留的时间，
   下一包 *n+1* 自己的时间戳位置不变。
2. 若包 *n* 本身迟到才到达：它的音频时间槽已被 PLC 占用，通常直接丢弃；若再正常解码它，
   会额外推进一次解码器状态和输出时长，造成时间戳重复/错位。

换言之：**PLC 和 FEC 都推进（或填充）固定的采样数，时间的连续性由应用层保证
"每丢一包恰好补一包时长"来维持**。补多了或补少了，后续所有包的播放时间戳都会平移，
且解码器内部状态（SILK 帧边界、CELT MDCT 重叠）也会与码流错位——这正是 opus.h 中
"not be in the optimal state to decode the next incoming packet" 的含义。

---

## 2. 何时使用 FEC，以及连续丢包如何影响解码器状态

### 2.1 FEC 的使用时机与回退逻辑

带内 FEC 的使用模式是：**包 *n* 丢失 → 收到包 *n+1* 时，用 `decode_fec=1` 调
`opus_decode()` 恢复包 *n***。`decode_fec` 的语义（`include/opus.h:512-513`）：

> Flag (0 or 1) to request that any in-band forward error correction data be decoded.
> **If no such data is available, the frame is decoded as if it were lost.**

`opus_decode_native()` 的 FEC 分支（`src/opus_decoder.c:813-847`）实现了完整的判定与回退：

```c
if (decode_fec)
{
   /* If no FEC can be present, run the PLC (recursive call) */
   if (frame_size < packet_frame_size || packet_mode == MODE_CELT_ONLY || st->mode == MODE_CELT_ONLY)
      return opus_decode_native(st, NULL, 0, pcm, frame_size, 0, ...);   /* 退化为 PLC */
   /* Otherwise, run the PLC on everything except the size for which we might have FEC */
   if (frame_size-packet_frame_size!=0)
      ret = opus_decode_native(st, NULL, 0, pcm, frame_size-packet_frame_size, 0, ...);  /* 先 PLC 多余部分 */
   ...
   ret = opus_decode_frame(st, data, size[0], pcm+st->channels*(frame_size-packet_frame_size),
         packet_frame_size, 1 ...);   /* 再用 FEC 解一帧 */
   st->last_packet_duration = frame_size;
   return frame_size;
}
```

要点：

- **CELT-only 包不含带内 FEC**，直接退化为 PLC（`src/opus_decoder.c:818-819`）；FEC 只存在于
  SILK 与 hybrid 模式（SILK 的 LBRR，以及 hybrid 的 0–8 kHz 冗余带，`src/opus_decoder.c:510-538`）。
- 若丢失时长（`frame_size`）大于本包一帧的时长（`packet_frame_size`），**先对差额做 PLC，
  再用 FEC 解最后一帧**，总返回采样数仍恰为 `frame_size`。
- SILK 侧通过 `lost_flag = 2 * !!decode_fec`（即 `FLAG_DECODE_LBRR`，`src/opus_decoder.c:469`）
  告知 SILK 解码 LBRR 数据；仅当该包的 LBRR 标志置位时才真正解码冗余信息
  （`silk/decode_frame.c:70-71`），否则同样落入 PLC 分支（`silk/decode_frame.c:131-137`）。
- FEC 解码只消费包 *n+1* 中携带的冗余数据；包 *n+1* 自己的音频仍需随后以
  `decode_fec=0` 再解一次。

### 2.2 连续丢包对 SILK 状态的影响

SILK 用 `lossCnt` 记录**连续**丢包数，每丢一帧加一（`silk/PLC.c:99`），成功解码（含 FEC
成功）时清零（`silk/decode_frame.c:125`）。`lossCnt` 直接索引 PLC 衰减表
（`silk/PLC.c:40-43, 275-279`）：

```c
#define NB_ATT 2
static const opus_int16 HARM_ATT_Q15[NB_ATT]              = { 32440, 31130 }; /* 0.99, 0.95 */
static const opus_int16 PLC_RAND_ATTENUATE_V_Q15[NB_ATT]  = { 31130, 26214 }; /* 0.95, 0.8  */
static const opus_int16 PLC_RAND_ATTENUATE_UV_Q15[NB_ATT] = { 32440, 29491 }; /* 0.99, 0.9  */
...
harm_Gain_Q15 = HARM_ATT_Q15[ silk_min_int( NB_ATT - 1, psDec->lossCnt ) ];
```

即第 1 个连续丢包用第一档（谐波增益 0.99、浊音随机激励 0.95），**从第 2 个连续丢包起
衰减加剧**（0.95 / 0.8），之后保持该档。此外：

- 丢包时把 `LastGainIndex` 重置为 10，防止能量在丢包后"反弹"
  （`silk/dec_API.c:478-482`）；
- 丢包期间参数解码走条件编码分支（`silk/decode_parameters.c:81`），CNG 在丢包时更新
  舒适噪声（`silk/CNG.c:129`）；
- PLC 的基音周期、LTP 增益等外推种子来自最近成功解码帧（`silk/PLC.c:135-189`），
  连续丢包意味着外推越来越依赖自己生成的信号，能量按上表逐帧衰减。

### 2.3 连续丢包对 CELT 状态的影响

CELT 侧的状态变量在 `celt_decoder.c` 中：

- `st->loss_duration` / `st->plc_duration`：每隐藏一帧累加 `1<<LM`（即以 2.5 ms 为单位
  的帧长），上限 10000（`celt/celt_decoder.c:1186-1187`）；正常解码成功后清零
  （`celt/celt_decoder.c:1706-1708`）。
- **PLC 模式随连续丢包时长切换**（`celt/celt_decoder.c:749-756`）：
  - 默认 `FRAME_PLC_PERIODIC`（基于基音的外推）；
  - `plc_duration >= 40`（即连续隐藏 ≥ 100 ms）或 `skip_plc` 置位时退化为
    `FRAME_PLC_NOISE`（噪声/舒适噪声隐藏）；
  - 启用 Deep PLC 且复杂度 ≥ 5 时，`plc_duration < 80`（< 200 ms）内可用神经
    `FRAME_PLC_NEURAL`。
- 噪声 PLC 的衰减随 `loss_duration` 增长：`decay += MAX(1, min(160, loss_duration)*0.0625)`
  （`celt/celt_decoder.c:787-788`），丢得越久衰减越快。
- **丢包后的恢复帧也受影响**：正常解码时若 `loss_duration != 0`，会对带能量施加额外
  衰减并限制能量增长（`celt/celt_decoder.c:1469-1473, 1683`），使恢复帧与隐藏段平滑衔接。
- `skip_plc`：噪声型 PLC 之后置 1（`celt/celt_decoder.c:834-835`），注释明确
  "Skip regular PLC until we get two consecutive packets"——即**需要连续两个成功包**，
  才在 `loss_duration == 0` 处清除（`celt/celt_decoder.c:1404`），在此之前若再丢包直接
  走噪声 PLC 而不再做基音外推。

### 2.4 连续丢包下 FEC 的局限

FEC 只能恢复**上一包**：`decode_fec=1` 时只对 `packet_frame_size` 一帧做 FEC，超出部分
一律 PLC（`src/opus_decoder.c:822-831`）。连续丢 ≥ 2 包时，中间的包没有冗余载体，只能
PLC；且 SILK LBRR 是否存在于包中由编码端 `OPUS_SET_INBAND_FEC` 及丢包率预测决定
（编码器 CTL 测试见 `tests/test_opus_api.c:1310-1316`）。因此连续丢包场景下解码时间的
推进完全依靠 PLC 逐帧补齐，状态按 §2.2/§2.3 所述逐帧退化。

（仓库还包含实验性的 DRED/Deep PLC 扩展：`ENABLE_DRED` 下 `opus_decoder_dred_decode()`
可从深度冗余中恢复更长的丢失段，见 `src/opus_decoder.c:748-773` 与
`include/opus.h:668-697`，本文不展开。）

---

## 3. 三种丢包场景：采样数与状态变化

### 3.1 孤立丢包（单包丢失，纯 PLC）

**调用**：`opus_decode(dec, NULL, 0, pcm, frame_size, 0)`，`frame_size` = 丢失包的时长
（2.5 ms 的整数倍）。

**采样数**：返回值恰为 `frame_size`（PLC 循环保证，`src/opus_decoder.c:778-792`）。
播放时间戳前进 `frame_size` 个采样，与未丢包时完全一致。

**状态变化**：

- SILK：`lossCnt` 0 → 1，使用第一档衰减（谐波 0.99 / 浊音随机 0.95）；`LastGainIndex`=10；
  下一成功帧 `lossCnt` 清零（`silk/decode_frame.c:125`）。
- CELT：`loss_duration` 0 → 帧长（2.5 ms 单位），走 `FRAME_PLC_PERIODIC` 基音外推；
  下一成功帧 `loss_duration`/`plc_duration` 清零并置 `FRAME_NORMAL`
  （`celt/celt_decoder.c:1706-1708`），恢复帧带能量按 `loss_duration` 做衔接衰减。

**测试证据**：`tests/test_opus_decode.c:134-135` 对每种采样率/帧长配置验证零长包
（PLC）返回恰好请求的采样数：

```c
out_samples = opus_decode(dec[t], packet, 0, outbuf, 120/factor, fec);
if(out_samples!=120/factor)test_failed();
```

`tests/test_opus_encode.c:589-591` 的 LBRR 测试中，随机丢包时以 `len=0` 解码，
同样断言输出采样数等于请求帧长：

```c
loss=(fast_rand()&63)==0;
out_samples = opus_multistream_decode(MSdec_err, packet, loss?0:len, out2buf, frame_size*6, (fast_rand()&3)!=0);
if(out_samples!=(frame_size*6))test_failed();
```

### 3.2 FEC 可恢复的单包丢失

**调用**：包 *n* 丢失、包 *n+1* 到达后，`opus_decode(dec, pkt[n+1], len, pcm,
frame_size = 丢失时长, 1)`。

**采样数**：返回恰为 `frame_size`。其中最后 `packet_frame_size` 个采样来自 FEC 解码，
若 `frame_size > packet_frame_size` 则前段由 PLC 补齐（`src/opus_decoder.c:822-838`）。
**时间戳不额外前进**——FEC 填的是丢包时已预留的时间槽，包 *n+1* 随后正常解码时仍落在
其原有时间位置。

**状态变化**：FEC 成功时 SILK 走 `FLAG_DECODE_LBRR` 正常解码分支
（`silk/decode_frame.c:70-71`），`lossCnt` 清零、PLC 状态用真实解码参数更新——
**解码器状态与"没丢包"几乎等价**，这是 FEC 相对 PLC 的核心价值。若包内无 LBRR
（`LBRR_flags==0`）或包为 CELT-only，则自动退化为 §3.1 的 PLC
（`src/opus_decoder.c:818-819`、`silk/decode_frame.c:131-137`、`include/opus.h:513`）。

**测试证据**：`tests/test_opus_encode.c:590` 中 `(fast_rand()&3)!=0` 即以 3/4 概率对
未丢的包置 `decode_fec=1` 做 LBRR 解码并校验采样数；编码器侧 FEC 开关的 CTL 往返测试见
`tests/test_opus_api.c:1310-1316`（`OPUS_SET_INBAND_FEC`/`OPUS_GET_INBAND_FEC`）。
非法 `decode_fec` 值（-1、2）必须返回错误，见 `tests/test_opus_decode.c:160-162`
（对应 `src/opus_decoder.c:743-744` 的检查）。

### 3.3 连续丢包（≥ 2 包）

**调用**：每丢一包调用一次 PLC，`frame_size` 分别为各丢失包的时长（同码流可沿用
`OPUS_GET_LAST_PACKET_DURATION`，`src/opus_decoder.c:1193`）。

**采样数**：每次调用返回各自的 `frame_size`，播放时间戳逐包推进，总补齐采样数 =
各丢失包时长之和。即使第 2 个包之后收到了带 FEC 的包，FEC 也只能恢复**最近一个**
丢失包，更早的洞已由 PLC 永久填充。

**状态变化（随连续丢包数累积）**：

| 连续丢包 | SILK | CELT |
|---|---|---|
| 第 1 包 | `lossCnt`=1，衰减档 0（0.99/0.95） | `loss_duration`=帧长，基音外推 PLC |
| 第 2 包起 | `lossCnt`≥2，衰减档 1（谐波 0.95、浊音随机 0.8、清音 0.9），此后保持 | `loss_duration`/`plc_duration` 继续累加，噪声衰减随 `loss_duration` 增大（`celt_decoder.c:787-788`） |
| ≥ 100 ms | 同上（表已封顶，NB_ATT=2） | `plc_duration ≥ 40` → 切换为噪声型 PLC（`celt_decoder.c:750`） |
| 恢复时 | 首个成功帧 `lossCnt`=0，`LastGainIndex` 解除钳制 | 需**连续两个成功包**才清 `skip_plc`（`celt_decoder.c:834-835, 1404`），恢复帧能量受 `loss_duration` 衰减衔接 |

**测试证据**：`tests/test_opus_encode.c:637-651` 的丢包/模糊测试循环中，随机将
`len` 置 0（丢包）后用两个独立解码器实例解码并断言：丢包时返回采样数非负且不超过
缓冲上限、未丢包时输出恰为 `frame_size` 且两个解码器的 `final_range` 一致——
验证了 PLC 与正常解码交错推进时解码器状态（range）仍保持确定性和一致性。

---

## 4. 结论

1. **解码时间 = 累计采样数**。PLC 与 FEC 调用都按 `frame_size` 精确产出采样
 （`src/opus_decoder.c:778-792, 813-847`），应用层必须保证"每丢一包补恰好一包时长"
 （2.5 ms 整数倍），否则播放时间戳漂移且解码器状态与码流失优（`include/opus.h:506-511`）。
2. **FEC 用于"孤立丢包 + 下一包已到达"的补洞**：它恢复上一包音频、不额外推进时间，
 并使 SILK 状态等同于正常解码；无 FEC 数据（CELT-only、无 LBRR 标志）时自动退化为 PLC。
3. **连续丢包完全由 PLC 推进**：SILK 以 `lossCnt` 递增衰减（两档封顶），CELT 以
 `loss_duration`/`plc_duration` 累积并在约 100 ms 后从基音外推退化为噪声隐藏，
 恢复后还需连续两个成功包才完全回到正常 PLC 策略。FEC 在连续丢包中最多只能救回
 最后一个丢失包。
