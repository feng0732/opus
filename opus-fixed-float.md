# Opus 定点与浮点路径的质量兼容边界

本报告基于本仓库（libopus 源码树）的实现，分析定点（`FIXED_POINT`）与浮点两条数值路径在舍入、削波与状态推进上的差异，归纳外部音频接口必须满足的兼容边界。所有结论均附实现出处（`文件:行号`）。

## 0. 两条路径如何分叉

整个数值表示由编译宏 `FIXED_POINT` 切换（`celt/arch.h:130`）。代码中统一使用四层信号表示（`celt/arch.h:113-118` 的注释）：

| 层 | 浮点构建 | 定点构建 |
|---|---|---|
| FLOAT（float API） | ±1.0 | ±1.0（仅 API 边界处） |
| INT16（16-bit API） | ±32768 | ±32768 |
| RES（Opus 内部分辨率） | float，±1.0 | 16-bit int（`ENABLE_RES24` 时为 24-bit，`celt/arch.h:149-178`） |
| SIG（CELT 内部分辨率） | float，±32768 | Q27（int16 左移 `SIG_SHIFT=12`，`celt/arch.h:208`） |

关键事实：**并非所有模块都有两套实现**。从 `silk_sources.mk` 可见：

- **两种构建完全共享同一份整数代码**：SILK 解码器全部（`decode_core.c`、`PLC.c`、`NSQ.c`、重采样器 `resampler*.c` 等）、SILK 编码器的量化与熵编码核心（NSQ、增益量化、shell coder）、范围编码器（`celt/entenc.c`/`entdec.c`）、包解析（`src/opus.c`）。
- **只有 SILK 编码器的分析部分**有 `silk/fixed/` 与 `silk/float/` 两套（LPC/LTP/基音搜索、噪声整形分析）。
- **CELT 的编码器与解码器**通过 `arch.h`/`fixed_generic.h` 的宏把同一套算法映射到两种算术。

由此产生一个重要的结构性结论：**SILK-only 帧的解码输出在两种构建间逐比特一致**（SILK 解码器输出先以 16-bit 整数生成，再经 `INT16TORES` 转换，`silk/dec_API.c:440-465`）；跨构建差异只来自 CELT 部分与编码器分析决策。

## 1. 舍入、削波与状态推进的差异

### 1.1 舍入

**浮点→整数转换（API 边界）**：`float2int()` 在各平台使用"最近偶"舍入——SSE `cvtss2si`、aarch64 `vcvtns_s32_f32`、C99 `lrintf`；无这些设施时退化为 `(int)floor(.5+x)`（`celt/float_cast.h:64-147`）。`FLOAT2INT16` 先乘 32768、钳制到 [-32768, 32767] 再舍入（`celt/float_cast.h:150-156`）。

**定点内部舍入**：使用显式的 rounding-shift 宏，且不同宏的半值处理不同，例如 `silk_RSHIFT_ROUND` 在移位为 1 时是 `(a>>1)+(a&1)`，否则是"加半再移"（`silk/SigProc_FIX.h:527-528`）；常量量化 `QCONST16(x,bits)` 用 `.5+x*2^bits` 截断（`celt/fixed_debug.h` 与 `fixed_generic.h`）。浮点构建中这些宏全部退化为恒等或普通浮点乘加（`celt/arch.h:303-389`），不存在任何中间量化。

**等效滤波器的舍入路径不同**，是两条路径逐样本差异的主要来源：

- 高通双二阶 `hp_cutoff`：定点（非 RES24）走 `silk_biquad_alt`（Q28 系数、Q14 中间量、`SAT16` 输出），浮点/RES24 走 `silk_biquad_res`；后者的浮点版把 Q28 系数转回 float 直接计算（`src/opus_encoder.c:362-476`）。
- 直流抑制 `dc_reject`：定点用 `shift = ilog2(Fs/(4*cutoff))` 的移位近似一阶 IIR，并对输入饱和到 +6 dBFS 防回绕；浮点用精确系数 `coef = 6.3*cutoff_Hz/Fs`（`src/opus_encoder.c:478-546`）。
- SILK 浮点编码器在分析完成后，把浮点结果**量化回定点 Q 格式**（`silk_float2int`）再送入共享的 NSQ/熵编码核心（`silk/float/wrappers_FLP.c:123-159`）——因此浮点 SILK 编码的"量化噪声注入点"在分析输出处，而定点构建的量化噪声遍布整个分析过程，二者决策可能不同。
- SILK 编码器输入在两种构建中都先被压成 16-bit（`RES2INT16`，`silk/enc_API.c:294-338`）：SILK 路径的有效输入分辨率边界就是 16 bit，浮点构建也不例外。

### 1.2 削波 / 饱和

**定点：逐级硬饱和。**

- 16-bit 饱和 `SAT16`（`celt/arch.h:231-233`）、SILK 的 `silk_SAT16`/`silk_ADD_SAT32` 等（`silk/SigProc_FIX.h:470`、`silk/macros.h:99-105`）。
- CELT 内部信号饱和上限 `SIG_SAT = 2^29-1`，其取值由 ARM 汇编 comb filter 的"左移 1 位再相加"约束推算而来（`celt/arch.h:209-216` 注释）。
- `dc_reject` 输入饱和到 +6 dBFS 防 wrap-around（`src/opus_encoder.c:491-500`）。
- 解码增益 `decode_gain` 作用后饱和到 32767（RES24 时饱和到对应余量，`src/opus_decoder.c:692-708`）。

**浮点：内部不饱和，出口软削波。**

- 浮点构建中 `SATURATE`/`SATURATE16` 是恒等宏（`celt/arch.h:324-325`），内部允许瞬时超过 ±1。
- 解码输出经 `opus_pcm_soft_clip_impl` 做软削波：非线性域限制在 [-2, 2]，系数 `a` 额外加 `2.4e-7`（约 2^-22）余量，专门保证即使 `-ffast-math` 下输出也不超 ±1（`src/opus.c:39-166`，余量见 `src/opus.c:131-134`）。软削波只在浮点构建的解码路径启用（`src/opus_decoder.c:887-892` 与 `OPTIONAL_CLIP` 宏，`src/opus_decoder.c:896-900`）。
- int16 输出接口处，浮点构建用 `celt_float2int16`→`FLOAT2INT16` 做硬钳制（`celt/mathops.c:307`、`src/opus_decoder.c:943`）。

**异常输入处理不对称**：浮点编码器对 NaN/巨值输入有显式防御——帧能量 `!(sum < 1e9)` 或 NaN 时清零该帧并重置高通记忆（`src/opus_encoder.c:2030-2042`）；定点构建无此检查（整数不会 NaN，且输入转换时已被 `FLOAT2INT16` 钳制）。浮点构建还禁止直接用 `-ffast-math` 编译（除非定义 `FLOAT_APPROX`），否则 `#error`（`celt/arch.h:281-283`）——这是为了保证 NaN 检测与上述防御可靠。

### 1.3 状态推进

**编码器侧（决策性状态，两路径可分叉）：**

- 音调/语音分析（tonality analysis）的启用阈值不同：定点要求 complexity ≥ 10，浮点要求 ≥ 7（`src/opus_encoder.c:1256-1260`）。
- 数字静音判定不同：定点要求 `sample_max == 0`（严格全零）；浮点允许 `sample_max <= 1/2^lsb_depth`（`src/opus_encoder.c:1067-1084`）。
- 帧能量计算、VAD/DTX、模式/带宽/比特分配全部由各自算术路径推出，因此**同一输入下两种构建的编码器状态机（模式、带宽、DTX、立体声宽度等）允许分叉**——编码比特流不要求跨构建一致（详见第 2、3 节）。
- 浮点滤波器中散布 `+ VERY_SMALL` 项（如 `dc_reject` 的 `m0 = coef*x + VERY_SMALL + coef2*m0`，`src/opus_encoder.c:525-526`），用于把状态推离次正规（denormal）区避免 CPU 性能塌陷；定点无此概念。

**解码器侧（必须可从比特流重建的状态）：**

- SILK 解码器状态：两种构建同一整数代码，逐比特一致。
- CELT 解码器状态（MDCT 重叠缓存、去加重记忆、PLC 基音状态、anti-collapse 能量）：定点是确定的整数序列，浮点是浮点序列，**同一比特流下二者会有微小差异并随帧累积**，但都被测试向量的加权误差门限约束（见第 3 节）。
- 模式切换时的状态操作两条路径机制相同：编码器在模式变化时 `OPUS_RESET_STATE` + 2.5 ms 预填充（`src/opus_encoder.c:2499-2507`）；解码器在 `mode != prev_mode` 时同样重置 CELT 状态（`src/opus_decoder.c:612-614`）；hybrid→SILK 切换时解码一帧 CELT 静音让 MDCT 自然淡出（`src/opus_decoder.c:630-636`）；SILK↔CELT 切换用 5 ms 冗余帧 + `smooth_fade` 交叉淡化（`src/opus_decoder.c:647-690`）。
- **范围编码器状态 `final_range` 是纯整数，两条路径完全一致**（见 1.4）。

### 1.4 必须保持一致的 API 结果

以下结果与数值路径无关，必须逐比特/逐值一致：

1. **`OPUS_GET_FINAL_RANGE`**：范围编码器（`celt/entenc.c`/`entdec.c`）是纯整数进位传播实现，同一比特流下编码器与解码器的最终状态必须**精确相等**——包括跨构建（浮点编码 → 定点解码，反之亦然）。实现要点：
   - 编码端 `st->rangeFinal ^= redundant_rng`（`src/opus_encoder.c:2530-2533, 2574`），解码端同样异或冗余帧 rng（`src/opus_decoder.c:622, 638, 710-713`）；
   - 冗余帧即使音频被丢弃也必须解码，"the final range is still needed (for testing)"（`src/opus_decoder.c:589-600` 注释）；
   - 丢包隐藏（`data == NULL`）时 `rangeFinal = 0`，`len <= 1` 的帧也清零（`src/opus_decoder.c:638, 710-711`）；
   - 一致性由 `tests/test_opus_encode.c:501, 628`（`dec_final_range != enc_final_range` 即 `test_failed()`）和 `src/opus_demo.c:1284-1294` 强制检查。
2. **返回值与错误码**：`opus_encode*` 返回字节数、`opus_decode*` 返回采样数、`opus_packet_parse`/`opus_packet_get_*` 等纯整数包解析（`src/opus.c:170-398`），对同一输入必须一致。
3. **SILK-only 帧的解码 PCM**：SILK 解码器是共享整数代码，跨构建逐样本一致（`silk_sources.mk` 的共享 `SILK_SOURCES`；输出经 `INT16TORES`，`silk/dec_API.c:440-465`）。
4. **DTX/包长信令的可解码性**：TOC、帧数、帧长、padding 的编码是纯整数（`src/opus_encoder.c:gen_toc`、`src/opus.c:encode_size`），任何构建产生的包都必须能被任何构建解析出相同的帧边界。

## 2. 可观察差异的场景

### 2.1 安静输入

- **严格全零**：两路径一致——`is_digital_silence` 都判真，编码器走静音/DTX 分支（`src/opus_encoder.c:1253, 1467-1470`），产生相同的 1 字节静音帧或 DTX 跳帧。
- **接近零但非零**：分叉点。定点要求采样**严格等于 0** 才判静音；浮点允许 `≤ 2^-lsb_depth`（`src/opus_encoder.c:1076-1081`）。一个浮点构建下被判为静音（触发 DTX、停止发包）的微弱抖动信号，在定点构建中可能被判为活动帧（正常编码）——**可观察差异是包是否发出、以及 `OPUS_GET_DTX` 状态**。反之，定点构建经 `opus_encode_float` 输入的次 LSB 信号会在 `FLOAT2RES=FLOAT2INT16` 处被舍入为 0，反而更"安静"。
- **解码侧**：静音包/丢包走 PLC。SILK PLC 跨构建一致；CELT PLC（基音外推 + 能量衰减）随各自算术路径漂移；若启用 Deep PLC/DRED，则只有浮点构建可用（`configure.ac:897-898` 禁止 `--enable-fixed-point` 与 `--enable-deep-plc/--enable-dred/--enable-osce` 组合）——**丢包隐藏的输出属于格式未规定行为，是兼容边界之外的部分**。
- 浮点路径的 `VERY_SMALL` 防次正规项（`src/opus_encoder.c:525` 等）意味着浮点构建的"静音"滤波器状态并非精确 0，而是 1e-30 量级——对输出无影响，但说明两路径的内部"零状态"定义不同。

### 2.2 接近边界的输入

- **float API 输入超过 ±1.0**：浮点构建内部不饱和，允许过幅信号贯穿编码器；定点构建在输入转换处即被 `FLOAT2INT16` 钳到 ±32767/-32768（`celt/float_cast.h:150-156`，`FLOAT2RES` 即 `FLOAT2INT16`，`celt/arch.h:174`）。同一过幅输入下，两种构建的编码输入已经不同。
- **NaN/Inf**：浮点构建检测并将整帧清零、重置高通状态（`src/opus_encoder.c:2030-2042`）；定点构建中 NaN 经 `float2int` 转换的结果是平台相关的整数（SSE 上为 `INT_MIN`），随后被钳制——行为未定义但无害。这是浮点构建明确承诺、定点构建不承诺的边界。
- **满幅解码输出**：同一比特流在接近 ±1.0 处，定点输出是 `SAT16` 硬削波，浮点输出是软削波（`src/opus.c:39-166`）——波形在峰值附近可观察地不同，但两者都保证 int16 API 输出落在 [-32768, 32767]。
- **int24 API**：`ENABLE_RES24` 定点构建保留 24-bit 内部分辨率（`MAX_ENCODING_DEPTH=24`，`celt/arch.h:163`）；普通定点构建把 int24 输入饱和到 16-bit（`INT24TORES = SAT16(PSHR32(a,8))`，`celt/arch.h:172`），`MAX_ENCODING_DEPTH=16`（`celt/arch.h:177`）；浮点构建 24-bit 无损缩放（`celt/arch.h:379, 389`）。即"24-bit 编码深度"这一能力本身依赖构建配置。
- **内部饱和点**：定点 CELT 的 `SIG_SAT=2^29-1`（`celt/arch.h:216`）在极端满幅高频音下会触发内部饱和；浮点无对应机制。此类输入下两路径的内部表示差异最大，但都被出口削波约束回合法范围。

### 2.3 模式切换

- **切换时机可分叉**：SILK/hybrid/CELT 的模式与带宽决策来自各自路径的分析结果（`src/opus_encoder.c` 中 `equiv_rate`、音调度、立体声宽度等），数值差异可使切换点相差若干帧——产生的比特流不同，但都是合法流。
- **切换机制一致**：无论哪条路径做出的切换决策，其比特流表达（TOC 模式位、冗余标志、5 ms 冗余帧）与解码侧状态操作（CELT 状态重置 + 预填充、交叉淡化、hybrid→SILK 静音淡出）完全相同（`src/opus_encoder.c:2499-2507, 2535-2566`；`src/opus_decoder.c:589-690`）。因此**切换点分叉不影响跨构建可解码性与 `final_range` 同步**。
- **状态连续性边界**：规范层面承认切换处的状态处理属于兼容边界——opus-update 草案记录了 SILK 立体声状态在模式切换时未重置会在解码输出产生单样本脉冲的已知问题及其修法（`doc/draft-ietf-codec-opus-update.xml:96-106`），说明"切换瞬间的单样本级差异"被显式纳入规范容忍范围。
- **冗余帧的 `final_range` 义务**：CELT→SILK 冗余帧即使因前一帧丢失而无法用于音频，也必须被解码以维持范围编码器状态同步（`src/opus_decoder.c:589-600`）——这是模式切换场景下跨构建必须维持的最强不变量。

## 3. 外部音频接口必须满足的兼容边界

综合上述实现依据，当两条数值路径产生不同内部表示时，外部接口的兼容边界如下：

1. **比特流互解（最强边界）**：任何构建的编码器产生的比特流，必须能被任何构建的解码器正确解码。依据：范围编码器与包封装是纯整数（`celt/entenc.c`、`src/opus.c:170-398`）；SILK 解码与 CELT 解码的算法结构共享，仅算术精度不同。
2. **`final_range` 跨构建相等**：同一比特流，编码端与解码端的 `OPUS_GET_FINAL_RANGE` 必须精确相等，与构建无关。这是唯一被测试强制的逐比特跨组件不变量（`tests/test_opus_encode.c:501, 628`；`src/opus_demo.c:1286-1294`）。
3. **解码输出的符合性判据是加权误差门限，而非逐样本相等**：定点构建的解码器是规范参考（RFC 6716 定点伪代码），浮点构建的 CELT 解码输出允许偏离。测试向量对同一比特流提供两套参考输出（是否应用 CELT 180° 相移，`doc/draft-ietf-codec-opus-update.xml:412-420`），任一构建通过 `opus_compare` 的加权误差门限即合规（`tests/run_vectors.sh:88-99`；门限与质量分计算见 `src/opus_compare.c:369-379`）。SILK-only 内容例外：跨构建逐样本一致（见 1.4-3）。
4. **int16 API 的输出范围与舍入**：两路径都必须把输出约束在 [-32768, 32767]；定点直接产出 int16，浮点经 `FLOAT2INT16` 最近偶舍入 + 钳制（`celt/mathops.c:307`、`celt/float_cast.h:150-156`）。同一比特流下两者允许存在 LSB 级差异（CELT 部分），不允许越界。
5. **float API 的有效分辨率随构建而变**：定点构建的 `opus_decode_float` 输出是 16-bit（RES24 时 24-bit）量化的浮点值（`RES2FLOAT`，`celt/arch.h:155, 170`）；调用方不得假设 float API 必然提供超过内部 RES 的分辨率。同理，定点构建的 `opus_encode_float` 输入在入口处即被量化为 16-bit。
6. **编码比特流不要求跨构建一致**：分析阈值（complexity 7 vs 10，`src/opus_encoder.c:1256-1260`）、静音判定（`src/opus_encoder.c:1076-1081`）、算术路径均不同，同一输入可产生不同但都合法的比特流。兼容义务在"可解码 + 质量门限"，不在"编码输出一致"。
7. **平台确定性不对称**：定点构建在任意平台上的编码/解码结果逐比特确定（整数运算；另有 `FIXED_DEBUG` 基础设施记录全部定点运算用于跨平台核对，`celt/fixed_debug.h`）；浮点构建允许随 FP 环境有微小变化，因此代码显式禁止 `-ffast-math`（除非 `FLOAT_APPROX`，`celt/arch.h:271-285`）并要求确定的 float→int 舍入设施（`celt/float_cast.h:64-147`）。
8. **丢包隐藏与可选增强在边界之外**：PLC（含 Deep PLC）、DRED、OSCE 等行为不由比特流规定；Deep PLC/DRED/OSCE 仅浮点构建可用（`configure.ac:897-898`）。丢包场景下两路径的输出可任意不同，唯一约束是 `rangeFinal=0` 的状态约定（`src/opus_decoder.c:638, 710-711`）与接口不崩溃。
9. **削波语义**：定点路径承诺逐级硬饱和、绝不回绕（`SIG_SAT`、`SAT16`）；浮点路径承诺 int16/float 出口不超满幅（软削波 + 2^-22 余量，`src/opus.c:131-134`）。两条路径对"|输出| ≤ 满幅"这一外部承诺一致，对"峰值附近波形"不承诺一致。

### 结论

Opus 的定点/浮点兼容模型是"**熵编码层逐比特一致、信号层门限内等效**"：范围编码器与 SILK 解码核心作为共享整数代码锚定了跨构建的强不变量（比特流互解、`final_range` 相等、SILK 帧逐样本一致）；CELT 与编码器分析允许数值分叉，其外部可见后果被 `opus_compare` 加权误差门限、int16 输出钳制和软/硬削波的满幅承诺所约束。安静输入的 DTX 判定阈值、过幅输入的钳制点、模式切换的决策时机是两路径可观察分叉的三个主要入口，但它们都不破坏上述接口级边界。
