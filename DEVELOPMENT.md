# 开发笔记与踩坑记录

> 这是给未来的自己看的：**改 bug / 加功能前先读这一份**。
> 整理：WorkBuddy · 2026-07-28 开发当天记录，2026-10-14 仓库化补充。

## 0. 仓库文件与版本关系

| 文件 | 说明 |
| --- | --- |
| `index.html` | **主版本（WorkBuddy 构建）**。单文件、零依赖、纯离线，全部功能都在这个文件里。**以后的改动请以它为准** |
| `audio-quality-analyzer-TRAE.html` | TRAE 构建的独立变体（备用）。与主版本是两套独立实现，不必同步修 bug |
| `DEVELOPMENT.md` / `CHANGELOG.md` | 本文件与更新记录 |

**在线版**：https://gooooddayy.github.io/Audio-Analyzer/ —— 由仓库根目录的 `index.html` 通过 GitHub Pages 自动构建。**改动 `index.html` 并提交后，线上版会自动跟着更新。**

## 1. 总体思路（为什么是现在这个形态）

两条硬约束决定了全部技术选型：

1. **纯本地、不联网** —— 隐私（音频不出浏览器）+ 离线可用
2. **单文件 HTML** —— 双击就能用，不需要任何环境

由此确定：

- **解码**：浏览器原生 `AudioContext.decodeAudioData()`（零依赖）
- **元数据**：自己读文件头（因为 decodeAudioData 不提供 codec / 位深 等任何元数据）
- **分析**：自己实现 FFT（radix-2）、统计计算，全部原生 JS

### 数据流

```
拖入文件
 → file.arrayBuffer()                    读入内存
 → detectFormat()                        按魔数探测：WAV / FLAC / AIFF / OGG / MP4(M4A) / MP3
     （parseWAV / parseFLAC / parseAIFF / parseMP3 / parseOgg / parseMP4）
 → decodeAudioData()                     解码为 Float32 PCM
 → computeReport()                       统计 + FFT + 截止频率 + 假无损/过采样判定 + 评级
 → renderReport() / drawSpectrum() / drawSpectrogram()   渲染
```

### 代码地图（index.html ≈1066 行）

> 行号会随改动漂移，以函数名为准。

| 位置 | 内容 |
| --- | --- |
| 前 1/3 | 内联 CSS 与 HTML 骨架 |
| `showLoading` / `hideLoading` / `setProgress` / `showError` | 加载与错误提示 |
| `readStr` / `parseExtendedFloat` | 字节读取；AIFF 的 80-bit 扩展浮点解析 |
| `parseWAV` / `parseFLAC` / `parseAIFF` / `parseMP3` / `parseOgg` / `parseMP4` | 6 个文件头解析器 |
| `detectFormat` | 按签名分发，汇总容器 / 编码器 / 采样率 / 位深 |
| `getAudioContext` / `analyzeFile` | 解码主流程（含 5 分钟分析时长限制） |
| `computeReport` | **核心分析函数**（统计算术、DR、频谱、判定、评级全在这里） |
| `hannWindow` / `averageSpectrum` / `fft` | DSP 工具 |
| `renderReport` | 把结果写进 DOM |
| `drawSpectrum` / `drawSpectrogram` / `spectrogramColor` | 频谱 / 频谱图绘制 |
| `formatTime` / `formatBytes` / `toKHz` / `dbFS` | 格式化工具 |

### 关键常量（调参先看这张表）

| 参数 | 值 | 说明 |
| --- | --- | --- |
| 分析时长上限 | 前 **300 秒** | `sampleRate * 300`，防止大文件卡死 |
| 频谱 FFT | **8192** 点，hop 4096，Hann 窗 | 单声道 + 左右声道各算一份谱 |
| 频谱图 FFT | **2048** 点，hop 512 | 逐列绘制，对数频率映射 |
| 截止检测阈值 | 峰值幅度 × **0.005** | 从高频往低频扫，第一个过线 bin 为截止点 |
| 削波阈值 | \|sample\| > **0.9999** | ≈ -0.0009 dBFS |
| DR 分块 | **3 秒**/块，取最响 **20%** 块 | 见 §2.2 |
| 过采样检测 | 仅当采样率 > 48 kHz | 见 §2.5 |

## 2. 关键算法口径（"跟别的软件对数"就靠这一节）

### 2.1 RMS（最容易错的口径）

- ❌ 把 L、R 混成单声道 `(L+R)/2` 再求 RMS —— 立体声内容会偏差多达约 3 dB（旧版 bug）
- ✅ 左右声道 RMS 的均方平均：`sqrt( (ΣL² + ΣR²) / (2N) )`
- 校验：-3 dBFS 正弦波 → RMS 应为 **-6.01 dBFS**

### 2.2 动态范围 DR（与 foobar2000 / DR Meter 对齐）

- 分块 **3 秒**，块内 RMS 用 §2.1 的口径
- 取**最响的 20% 块**（`max(1, floor(块数 × 0.2))`）求 RMS 平均
- `DR = 20·log10( 峰值 / 最响20%块的RMS均值 )`
- ❌ 已弃用旧做法：0.5 秒块 + 95 百分位 —— 用户拿真实文件对比其他工具时发现对不上
- 校验：恒定幅度信号 DR == Crest Factor（正弦波 = 3.01 dB）

### 2.3 Crest Factor

`20·log10(峰值 / RMS)`。正弦波理论值 3.01 dB。

### 2.4 削波

阈值 `|s| > 0.9999`。旧值 0.999 会把"接近满幅但没削波"的正常母带误报 —— 已修。

### 2.5 频率截止 / 假无损 / 过采样

- **截止频率**：频谱从最高 bin 往下扫，第一个超过「峰值 × 0.005」的位置
- **假无损**（只对无损文件判定）：
  - 截止 ≤ **16 kHz** → 失败（疑似低码率有损转码）
  - 截止 ≤ **20.5 kHz** → 提示（符合 MP3/AAC 转码特征）
- **过采样**（只对采样率 > 48 kHz 的文件判定）：
  - 截止 < **22 kHz** 且 >20kHz 能量占比 < **0.1%** → 失败（疑似升频假 Hi-Res）
  - 截止 < **24 kHz** 且占比 < **0.5%** → 提示
- 阈值均为经验值；正常 CD / 真 Hi-Res 不应触发

### 2.6 质量分级

- 无损：采样率 ≥ 88.2 kHz 且位深 ≥ 24 且无假无损失败 → 「高解析无损」；否则「CD 级普通无损」
- 有损：按总码率（`文件大小×8÷时长`）三档 → ≤80 语音级 / ≤192 标准 / >192 高品质

## 3. 踩坑记录（按时间顺序）

### 3.1 ffmpeg.wasm 联网加载失败 → 全盘改原生（最大的坑）

- 首版用 CDN 加载 ffmpeg.wasm（约 30MB）做解码，实际使用时**加载失败**
- 且它违背"纯本地不联网"的硬需求：每次打开都要联网拉引擎
- **解决**：彻底移除。改 `decodeAudioData` + 自写文件头解析
- **教训**：定了"离线"就贯彻到底 —— 不引 CDN、不引外部字体、不引外部图片

### 3.2 decodeAudioData 不提供任何元数据

只给 PCM 和采样率。codec / 位深 / 容器 / 码率全得自己从文件头读 —— 这是 6 个 `parse*` 函数存在的原因。

### 3.3 AIFF 的采样率是 80-bit 扩展浮点

公式：`sign × mantissa/2^63 × 2^(exp − 16383)`。开发时测试编码器漏了 +63 修正导致误判过一次（页面代码本身是对的）。
**调试技巧**：用已知数值手工构造字节流做单元测试最可靠。

### 3.4 "和其他网站数值对不上"事件（2026-07-28，实测反馈）

拿真实文件对比其他在线工具，三处不一致，逐一排查修复：

1. **RMS / 响度** → §2.1 口径问题（约 3 dB）
2. **削波次数** → §2.4 阈值过宽，误报
3. **动态范围** → §2.2 分块和取值口径都不同

另：**总采样数**与其他工具可能有微小出入 —— `decodeAudioData` 的解码结果包含编码器延迟/填充采样，属正常现象，以浏览器解码为准，不是 bug。

### 3.5 ALAC / M4A 位深度读不到

MP4 容器需从 `stsd` 的 `alac` box 读 ALACSpecificConfig：位深位于 +17、声道 +21、采样率 +32（大端）。
必须做合法性校验（位深 8–32、声道 1–8、采样率 8000–384000），否则可能误命中文件里其他 'alac' 字样。

### 3.6 性能

- 全曲 FFT 会卡死 → 只分析前 5 分钟
- 频谱图：逐列降采样 + `putImageData` 一次性写像素（不要逐点绘制）

### 3.7 浏览器解码能力天生有差异

| 浏览器 | 大致支持 |
| --- | --- |
| Chrome / Edge | WAV、MP3、FLAC、OGG、OPUS、AAC(M4A) |
| Safari | 上述全部 + ALAC、AIFF |

解码失败时页面提示换格式 / 换 Safari —— 这是特性，别"假装"能解。

### 3.8 各解析器速记

- **WAV**：RIFF → 遍历 chunk 找 `fmt `；注意 chunk 补齐（奇数 size 补 1 字节）与 WAVE_FORMAT_EXTENSIBLE(0xFFFE)
- **FLAC**：`fLaC` 后第一个 metadata block = STREAMINFO；采样率 20 bit / 声道 3 bit / 位深 5 bit 都在同一段紧凑位域（文件偏移 18–21）
- **MP3**：先跳过 ID3v2（synchsafe 长度），再找帧同步 `0xFFEx`，从帧头位域查采样率表 / 声道模式（注意 MPEG 版本对应的表不同）
- **OGG**：容器里搜 `OpusHead`（声道 @+9，采样率 @+12 小端，缺省 48000）或 `\x01vorbis`（声道 @+5，采样率 @+6 小端）
- **MP4/M4A**：找 `ftyp` 品牌；`alac` → ALAC；`mp4a`/`enca` → AAC

## 4. 测试方法（每次改完必跑）

### 4.1 语法检查（可选，需 Node）

```bash
node -e "const fs=require('fs');const h=fs.readFileSync('index.html','utf8');const m=h.match(/<script>([\s\S]*?)<\/script>/);fs.writeFileSync('_chk.js',m[1]);"
node --check _chk.js && rm _chk.js
```

（没装 Node 就跳过，靠 4.3 手工清单兜底。）

### 4.2 算法回归（正弦波理论值对照）

用 Node 生成 Float32 正弦波直接跑算法，对照理论值：

| 测试信号 | 指标 | 理论值 |
| --- | --- | --- |
| -3 dBFS / 1 kHz 正弦（L=R） | RMS | ≈ **-6.01 dBFS** |
| 同上 | Crest Factor | ≈ **3.01 dB** |
| 恒定幅度信号 | DR | ≈ **3.01**（= Crest） |
| L=R | 立体声相关性 | **1.0000** |
| 混入 100 个满刻度采样 | 削波计数 | **100**（0.99885 的样本不应计入） |

### 4.3 手工验收清单

- [ ] WAV 44.1k / 16bit：正常出报告
- [ ] FLAC 96k / 24bit：应判「高解析无损」
- [ ] 由 MP3 转码的"假 FLAC"：应提示 / 失败
- [ ] MP3 320k：应判「高品质有损」
- [ ] M4A(AAC)、OGG、OPUS 各一
- [ ] 截止频率与 Spek 等工具对照（口径不同，允许 ±1~2 kHz 出入）

## 5. 继续开发的工作流

**不需要任何构建 / 依赖 / 环境**：编辑 `index.html` → 浏览器打开 → 拖文件测。

- **GitHub 网页直接改**：文件页右上角铅笔图标，或把网址的 `.com` 改成 `.dev`（在线 VS Code）
- **新电脑起步**：下载仓库 ZIP 解压，或 `git clone https://github.com/gooooddayy/Audio-Analyzer.git`
- **提交前自查**：
  1. 拖 3 个以上不同格式文件全流程走一遍
  2. 全局搜索确认没有引入外部资源：`http`、`<script src`、`@import` 应保持 0 处
  3. （有 Node 的话）跑 §4.1 语法检查
- **建议**：下一个功能版本顺手给页面加个版本号（目前没有版本标记，建议加在 `<title>` 或 HTML 注释里）

## 6. 已知限制与可扩展方向

**已知限制**

- 只分析前 5 分钟（超长曲目截断分析，评级基于前段）
- 总采样数与部分桌面软件有编码器延迟级微差（§3.4，正常）
- 截止检测阈值为经验值，极安静或高频稀疏的曲目可能偏差
- 有损文件位深度无意义（显示 N/A，特性）

**可扩展方向**

- LUFS 响度（ITU-R BS.1770）
- 批量分析、报告导出（JSON / CSV / 截图）
- 波形图、播放试听 A/B
- 更多容器解析（APE / WavPack / DSD 检测）
- 深色 / 浅色主题切换
