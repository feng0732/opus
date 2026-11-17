# Opus 定点与浮点路径的质量兼容边界分析

本报告基于本仓库（libopus 源码树，分支 `task/28`）的实现，分析定点（`FIXED_POINT`）
与浮点两条数值路径在舍入、削波、状态推进上的差异，明确哪些 API 结果必须保持一致，
并给出在安静输入、接近边界输入与模式切换时可观察的差异及其外部接口兼容边界。
所有结论均附实现出处（`文件:行号`）。

## 0. 两条路径的构成

定点/浮点是**编译期**选择（`FIXED_POINT` 宏），不是运行期开关。`celt/arch.h:113-118`
的注释定义了四层信号域，是理解全部差异的基础：

- **FLOAT**：浮点 API 域，归一化到 ±1；
- **INT16**：16 位 API 域，归一化到 ±32768；
- **RES**：Opus 内部分辨率——浮点构建下为 float（±1），定点构建下为 16 位 int
  （启用 `ENABLE_RES24` 时为 24 位，见 `celt/arch.h:149-178`）；
- **SIG**：CELT 内部分辨率——浮点构建下为 ±32768 的 float，定点构建下为 Q27
  （int16 左移 `SIG_SHIFT=12`，`celt/arch.h:208`）。

两条路径在各子系统中的实际分布并不对称，这是兼容边界的关键事实：

| 子系统 | 定点构建 | 浮点构建 | 出处 |
|---|---|---|---|
| SILK 编码器分析（LPC/基音/噪声整形） | `silk/fixed/*_FIX.c` | `silk/float/*_FLP.c` | `silk/enc_API.c:38-42` |
| SILK 编码器输入重采样 | 定点 `silk_resampler`，输入先 `RES2INT16` 量化到 16 位 | **同样**定点、同样先量化到 16 位 | `silk/enc_API.c:294-296,343`；`silk_sources.mk:63-70`（resampler 无 FIX/FLP 之分） |
| SILK 解码器（含 PLC、CNG、重采样） | 定点，内部 `opus_int16` 样本 | **完全相同**的定点代码，输出经 `INT16TORES` 转 float | `silk/dec_API.c:151-161,440-444` |
| CELT 编/解码器（MDCT、带域、PVQ） | Q27 定点 | float | `celt/arch.h:140-147,259-266` |
| 熵编码器（range coder） | 整数，两构建**逐位一致** | 同左 | `celt/entenc.c`/`entdec.c`（无 `FIXED_POINT` 分支） |

即：**SILK 的解码侧与重采样器在两种构建下是同一份定点实现**；浮点构建只把
SILK 编码器的分析前端和整个 CELT 换成了浮点。这决定了后文哪些差异存在、哪些不存在。

## 1. 舍入、削波与状态推进的差异

### 1.1 舍入

**定点路径**的舍入规则分散在各算子宏中，且并不统一：

- `PSHR32(a,shift)`：加半个 LSB 后算术右移，即"round-half-up（向正无穷方向入）"
  （`celt/fixed_generic.h:123`）；`ROUND16`/`SROUND16` 在其上再加饱和
  （`celt/fixed_generic.h:139-141`）。
- `MULT16_16_Q15`、`MULT16_32_Q15` 等 Q 系列乘法：直接截断（右移不加舍入，
  `celt/fixed_generic.h:53-58,189-193`）；对应的 `P` 系列（`MULT16_16_P15` 等）
  才加 0.5 舍入（`celt/fixed_generic.h:195-197`）。同一滤波器不同抽头用不同
  舍入是刻意为之的精度/溢出折中。
- 常量编译期量化：`QCONST16(x,bits) = (opus_val16)(.5 + x*2^bits)`
  （`celt/fixed_generic.h:92`），滤波器系数本身就有 ±0.5 LSB 的量化误差。
- SILK 侧同样混用：`silk_RSHIFT_ROUND`（舍入）与 `silk_RSHIFT`（截断），例如
  编码器高通/双二阶输出 `silk_SAT16( silk_RSHIFT( out32_Q14 + (1<<14) - 1, 14 ) )`
  （`src/opus_encoder.c:398`）是"加 (1<<14)-1 后截断"，对负数相当于向零舍入，
  与 `PSHR32` 的 half-up 在负半轴上不对称。

**浮点路径**中这些宏全部退化为恒等或普通浮点乘加（`celt/arch.h:312-369`），
唯一的舍入发生在 API 边界的 float→int 转换：

- `float2int()` 在 SSE 上为 `_mm_cvt_ss2si`、aarch64 上为 `vcvtns_s32_f32`、
  否则为 `lrintf`——均为 **round-to-nearest-even（银行家舍入）**
  （`celt/float_cast.h:64-136`）；最后的兜底 `(int)floor(.5+x)` 是 half-up。
- `FLOAT2INT16` 先乘 32768、钳到 [-32768, 32767] 再转换（`celt/float_cast.h:150-156`）。

**可观察后果**：同一个小数恰好在 .5 上的样本，定点 `PSHR32` 入、浮点
`lrintf` 按偶数舍，两者可差 1 个 LSB；这种差是确定性的、逐样本可复现的，
不构成兼容性违例（见第 4 节）。

### 1.2 削波（饱和）

定点路径在**内部多级**饱和，浮点路径内部基本不饱和、只在 API 出口处理：

定点构建：

- SIG 域饱和门限 `SIG_SAT = 2^29-1`，约 +6 dB 余量，专为 ARM 汇编 comb filter
  的左移加法和 MDCT 前级不溢出而设（`celt/arch.h:209-216`）；CELT 解码器在去加重
  环路内每样本 `SATURATE(x[j] + VERY_SMALL + m, SIG_SAT)`
  （`celt/celt_decoder.c:294,320-321`），MDCT 输出合成处同样饱和
  （`celt/celt_decoder.c:530,1044,1110-1115`）。
- SIG→INT16 出口 `SIG2WORD16`：`PSHR32` 舍入后硬钳到 [-32768, 32767]
  （`celt/fixed_generic.h:209-216`）。
- 编码器前端 `dc_reject` 先把输入饱和到 +6 dBFS 防回绕
  （`src/opus_encoder.c:492,500`）。
- `decode_gain` 应用后 `SATURATE(x, 32767)`（`src/opus_decoder.c:704-705`）。

浮点构建：

- `SATURATE`/`SATURATE16` 是恒等宏（`celt/arch.h:324-325`），内部信号可以超过
  ±1（SIG 域 ±32768）一路传播。
- 唯一的有损限幅是 **soft clip**：`opus_decode`（int16 API）在浮点构建下默认启用
  （`OPTIONAL_CLIP=1`，`src/opus_decoder.c:896-900,935`），
  `opus_pcm_soft_clip_impl` 用二次非线性 `x + a*x^2` 把越限峰压回 ±1，并跨帧保存
  `declip_mem` 避免不连续（`src/opus.c:39-161`）。它是**浮点构建独有**的；
  定点构建靠 `SAT16` 硬削波达到同样的"输出不越界"效果，但波形形状不同
  （软拐点 vs 平顶）。
- `opus_decode_float` 不做任何限幅（`src/opus_decoder.c:999-1006` 直接走
  `opus_decode_native`，soft_clip=0），输出可以合法地超过 ±1。
- 编码器侧，浮点输入在 `celt_preemphasis` 里被钳到 ±2 FS（SIG 域 ±65536），
  注释明确写"Clip input to avoid encoding non-portable files"
  （`celt/celt_encoder.c:591-597`）——即限制编码器产生定点解码器无法在 SIG_SAT
  余量内复现的信号，这是**编码器为跨路径兼容主动让渡**的边界。定点构建的
  `FLOAT2SIG` 在 float API 入口做同样的 ±2 FS 钳制（`celt/float_cast.h:165-172`）。

### 1.3 状态推进

两条路径的滤波器/预测器**状态变量精度不同，因此长时输出必然发散**，但发散被
各自限制在有界范围内：

- 编码器高通滤波器：定点用 `silk_biquad_alt`（Q12 状态、每步 `silk_RSHIFT_ROUND`
  舍入，`src/opus_encoder.c:374-399`）；浮点用直 II 型转置、float 状态，并往状态
  里加 `VERY_SMALL`（1e-30）防止 denormal 拖慢 FPU（`src/opus_encoder.c:426-437`，
  `celt/arch.h:294`）。定点对应物 `VERY_SMALL=0`（`celt/arch.h:223`）。
  `dc_reject` 同样：定点用移位近似系数 `shift=ilog2(Fs/(4*cutoff))`
  （`src/opus_encoder.c:485`），浮点用精确系数 `6.3*cutoff/Fs`
  （`src/opus_encoder.c:511`）——**截止频率本身就不完全相等**。
- SILK 重采样器状态（`silk_resampler_state_struct`）在两种构建下都是同一份定点
  状态，推进逐位一致（`silk/resampler*.c` 无浮点版本）；立体声拆/并时通过复制
  或平均两个声道的重采样器状态保持同步（`silk/enc_API.c:298-300,324-335`，
  最近一次提交 0145a62 修正了"仅当两个重采样器同速率时才混合"的边界条件）。
- CELT 去加重记忆 `preemph_memD`：定点是 Q27 且每步饱和，浮点是 float；两者从
  同一比特流出发会在若干样本后产生亚 LSB 级差异并持续携带。
- **熵编码器状态是唯一要求逐位一致的状态**：range coder 全整数实现，
  `OPUS_GET_FINAL_RANGE` 返回的 `rangeFinal` 对同一码流在两种构建下必须相等
  （`src/opus_decoder.c:622,713`；测试在 `tests/test_opus_decode.c:198-200`
  强制多解码器实例 final_range 相等）。它是"解码器确实消费了与编码器相同的
  比特"的密码学式校验，也是 RFC 6716 一致性测试的核心。

## 2. 必须保持一致的 API 结果

以下结果属于**控制面与比特流面**，与数值路径无关，两种构建必须逐位一致：

1. **比特流合法性互认**：任一构建编码器产出的码流必须能被任一构建的解码器
   解码。这是由"码流格式规范、解码过程规范"分层保证的——定点解码器是 RFC 6716
   的规范性参考，浮点解码器是允许有数值偏差的实现（见第 4 节的验收标准）。
2. **包解析族**：`opus_packet_get_samples_per_frame`/`opus_packet_get_nb_frames`/
   `opus_packet_get_nb_samples`/`opus_decoder_get_nb_samples`、repacketizer、
   `opus_packet_pad/unpad`——纯比特级解析，无数值路径（`src/opus.c`、`src/repacketizer.c`）。
3. **`OPUS_GET_FINAL_RANGE`**：同一码流、同一解码序列下两构建必须相等
   （熵编码器为纯整数，见 1.3）。
4. **CTL 控制面**：`opus_encoder_ctl`/`opus_decoder_ctl` 的 setter/getter
   （码率、带宽、VBR、DTX、增益、相位反转等）及其错误码；`OPUS_GET_SAMPLE_RATE`、
   `OPUS_GET_LAST_PACKET_DURATION` 等查询结果为整数语义，与数值路径无关。
5. **帧结构语义**：返回值（采样数/字节数）、`OPUS_INVALID_PACKET` 等错误判定、
   FEC/PLC 的触发条件（由包丢失与 TOC 决定，不由样本值决定）。
6. **输出幅度边界**：`opus_decode`（int16）输出必须落在 [-32768, 32767]——定点
   靠 `SAT16`，浮点靠 soft clip + `FLOAT2INT16` 钳制（`celt/float_cast.h:153-154`），
   机制不同但边界一致。

**不要求一致**的：编码器产出的具体比特流（定点与浮点编码器对同一输入可以、
也通常会产出不同码流——编码器不在规范约束内）；解码器输出的逐样本值。

## 3. 可观察差异：安静输入、边界输入、模式切换

### 3.1 安静输入

- **静音判定阈值不同**：定点构建 `is_digital_silence` 要求 `sample_max == 0`
  （严格数字静音）；浮点构建是 `sample_max <= 1/2^lsb_depth`
  （`src/opus_encoder.c:1067-1084`）。后果：一个幅度在 1 LSB 附近的近静音输入，
  浮点构建可能判为静音而走静音快捷路径（影响 DTX 计时与 `voice_ratio` 保持逻辑，
  `src/opus_encoder.c:1280-1283`），定点构建则当作正常信号编码——**同一输入
  可能产生不同的 DTX 行为和码流长度**。
- **极小信号的能量估计**：定点 `compute_frame_energy` 用动态移位防溢出
  （`src/opus_encoder.c:1087-1112`），对极小信号先归一化再平方，分辨率受限；
  浮点直接 `celt_inner_prod/len`。这影响 `peak_signal_energy` 跟踪，进而影响
  基于能量门限的编码器决策（如 DTX 挂起）。
- **滤波器零输入响应**：浮点路径向滤波器状态注入 `VERY_SMALL` 抗 denormal
  （`src/opus_encoder.c:433,525,540`），定点无此概念；静音段后的起始若干样本
  两路径的滤波器"尾巴"不同。
- **解码侧**：对静音帧（DTX 下的 1 字节帧或 PLC），SILK 的 CNG/PLC 在两构建下
  是同一份定点代码，输出一致；CELT 的 PLC 在浮点构建下是浮点激励外推，与定点
  的 Q27 外推在噪声底上有亚 LSB 差异。

### 3.2 接近边界（满幅/越限）输入

- **编码器入口**：int16 API 下两构建输入域相同；float API 下，浮点构建把 ±1 之外
  的信号带进内部（预加重前才钳到 ±2 FS，`celt/celt_encoder.c:591-597`），定点
  构建在 `FLOAT2RES`/`FLOAT2SIG` 处就钳掉（`celt/float_cast.h:150-172`）。
  对 [-2, -1]∪[1, 2] 区间的 float 输入，两构建的有效输入信号不同。
- **解码器出口**：满幅 CELT 解码时，定点在去加重环路内按 `SIG_SAT`（约 +6 dB 余量）
  逐级饱和、出口 `SIG2WORD16` 硬削波；浮点不饱和，int16 API 由 soft clip 把峰
  "揉"回 ±1（`src/opus.c:78-139`），float API 直接输出 >±1 的样本。
  **同一满幅码流：`opus_decode` 下定点输出平顶、浮点输出软拐点波形；
  `opus_decode_float` 下浮点构建可输出 |x|>1，定点构建经 `RES2FLOAT` 永远不会
  超过 ±1**（`src/opus_decoder.c:1035-1036` 的 `RES2FLOAT` 作用在已饱和的 RES
  域上）。
- **舍入边界**：恰落在 .5 的样本，定点 half-up 与浮点 round-half-even 差 1 LSB
  （见 1.1）；`FLOAT2INT24` 允许 2× 满幅（钳到 ±16777216，`celt/float_cast.h:158-164`），
  与 `ENABLE_RES24` 定点构建的 `INT24TORES` 饱和点一致（`celt/arch.h:158`），
  这是 24 位路径上刻意对齐的兼容边界。
- **`decode_gain`**：定点在增益应用后饱和到 ±32767，浮点不饱和
  （`src/opus_decoder.c:699-706` 的 `SATURATE` 在浮点构建为恒等）；且增益本身由
  `celt_exp2` 计算，定点是整数 exp2 近似、浮点是精确指数，增益值可有微小差别。

### 3.3 模式切换（SILK↔CELT↔Hybrid、带宽/声道切换）

- **切换决策可能不同**：模式/带宽选择由码率门限（整数逻辑，两构建一致）加上
  浮点分析结果（`run_analysis` 的 tonality/`music_prob`/`detected_bandwidth`）
  共同决定。注意分析器的启用门槛本身就不一样：定点构建要求
  `complexity >= 10`，浮点构建 `complexity >= 7`
  （`src/opus_encoder.c:1256-1260`）。在中等复杂度、码率接近 SILK/CELT 分界时，
  两构建可以对同一输入选出**不同的模式或带宽**，进而产生完全不同的码流结构
  （TOC、帧长组合、是否含冗余帧）。这是合法的——编码器决策自由——但意味着
  "同一输入同一码率"不保证同一模式序列。
- **编码器切换机制相同、状态不同**：CELT→SILK 用冗余帧 + SILK prefill（重置
  SILK 状态但保留可变 LP 状态，`silk/enc_API.c:215-244`；
  `src/opus_encoder.c:2211-2226` 还对 prefill 输入做增益渐隐）；SILK→CELT 用
  10ms 冗余 + 交叉淡化。这些机制的**结构**两构建一致，但参与计算的样本值
  按各自路径的精度推进。
- **解码器切换处理结构一致**：模式切换时用前一模式做 5 ms PLC 再与新模式输出
  交叉淡化（`src/opus_decoder.c:374-390,671-690` 的 `pcm_transition` 与
  `smooth_fade`）；CELT→SILK 冗余帧直接替换前 2.5 ms
  （`src/opus_decoder.c:661-669`）。差异只在样本精度：SILK 侧过渡音频来自定点
  PLC（两构建一致），CELT 侧过渡音频来自各自精度的 PLC（两构建不同），所以
  **切换点前后约 5 ms 的过渡窗是两路径样本差异最集中的区域**。
- **状态复位语义一致**：`prev_mode`、`prev_redundancy`、`first_frame_after_reset`
  等切换簿记是整数/标志逻辑（`src/opus_decoder.c:715-716`；
  `silk/enc_API.c:172-180,268-272`），两构建一致；单声道↔立体声切换时 SILK
  重采样器状态的复制/平均规则也一致（`silk/enc_API.c:298-300,324-335`）。

## 4. 外部音频接口必须满足的兼容边界

当内部数值表示不同（Q27 vs float SIG、16 位 vs float RES）时，外部接口的兼容
边界可归纳为四层，按严格程度排序：

1. **比特流层（必须逐位兼容）**：码流格式与熵编码状态。任何构建的编码输出必须
   能被任何构建的解码器消费且 `final_range` 校验一致（1.3、第 2 节）。这是
   唯一"逐位"级别的兼容要求，且它约束的是**比特流与整数状态**，不是音频样本。
2. **控制面层（必须语义一致）**：第 2 节列出的包解析、CTL、错误码、帧长/采样数
   返回值。这些接口不经过数值路径，天然一致；任何让数值路径影响这些结果的
   改动都是兼容性 bug。
3. **样本输出层（有界偏差，非逐样本一致）**：
   - `opus_decode`（int16）：输出保证落在 [-32768, 32767]；定点硬削波与浮点
     soft clip 的波形可以不同，但都必须通过 `opus_compare` 对官方测试向量的
     容差判决（`tests/run_vectors.sh:81-104`：浮点构建的解码输出与定点参考
     `.dec` 比较，通过即视为兼容；定点构建则要求与参考逐位一致——脚本中
     `testvector*.dec` 即定点参考输出）。
   - `opus_decode_float`：浮点构建输出可超过 ±1 且不 clip；定点构建输出被
     RES 域精度（16 位，或 RES24 下 24 位）天然限幅。调用方若要求 ±1 保证，
     必须自行限幅或改用 int16 API——**±1 边界只在 int16 API 上是库级保证**。
   - 编码器侧：两构建码流不必相同，但浮点编码器通过 ±2 FS 输入钳制
     （`celt/celt_encoder.c:591-597`）保证其码流不会超出定点解码器的内部
     余量，即"编码器输出必须可被定点参考解码器无溢出复现"。
4. **听感/质量层（统计意义等价）**：`src/opus_compare.c` 的判决逻辑（分带能量
   误差门限，`opus_compare.c:334-344`）定义了"质量兼容"的正式标准：两路径的
   差异允许在每带能量误差的容差内累积，但不允许出现可闻的结构性偏差（如削波
   不连续、过渡窗断裂）。soft clip 的 `a += a*2.4e-7f` 微调（`src/opus.c:131-134`）
   即是为保证即使 `-ffast-math` 下输出也不越 ±1 的边界工程。

**结论**：Opus 定点与浮点路径的兼容边界是"**同一比特流规范 + 同一整数控制面 +
有界样本偏差**"，而不是逐样本等价。定点解码器是规范性参考（测试向量逐位匹配），
浮点解码器只需落在 `opus_compare` 容差内；编码器两侧则通过输入钳制（±2 FS）、
共享的定点 SILK 重采样/解码内核、以及纯整数熵编码，保证任意组合的编码器-解码器
配对都满足上述边界。SILK 解码与重采样在两种构建下共用同一份定点实现这一事实，
把数值发散限制在了 CELT 内核与 SILK 编码器分析前端，是整套兼容策略的结构性基础。
