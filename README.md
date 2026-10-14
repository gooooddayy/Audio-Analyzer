# Audio-Analyzer

**本地音频质量分析器 — 免费音频质量分析工具**

- 频谱、动态范围、削波、假无损检测等功能
- 100% 隐私保护 — 在您的浏览器本地中运行
- 单文件 HTML、零依赖、纯离线

**在线使用**：https://gooooddayy.github.io/Audio-Analyzer/

## 功能

拖入音频文件即可查看其名称、大小、时长、有损/无损压缩、Sample Rate、位深度等完整报告：

- **多格式支持**：FLAC、WAV、AIFF、ALAC、MP3、AAC、OGG、M4A、OPUS
- **格式检测**：识别 codec、容器、sample rate、bit depth 和 bitrate
- **有损 / 无损质量等级**：低码率有损（语音级）、标准有损（流媒体标准音质）、高品质有损（流媒体高清/超清音质）、CD 级普通无损（标准无损）、高解析无损
- **假无损检测**：分辨是真无损，还是由有损源转码而来的文件
- **过采样检测**：识别从低分辨率源升频而来的"假高解析"音频
- **频率频谱分析**：可视化从 20 Hz 到奈奎斯特频率的能量分布
- **频谱图可视化**：显示整首曲目频率内容随时间的变化
- **动态范围与响度**：测量 Crest Factor、峰值电平和 RMS
- **削波检测**、**立体声相关性分析**
- **质量评估**：为您音频的每个维度提供 通过 / 提示 / 警告 / 失败 的评级

## 使用

1. 打开 `index.html`（本地双击即可，或使用上方在线地址）
2. 把音频文件拖入页面上方
3. 等待分析完成 —— 全部分析在浏览器本地完成，**不会上传任何数据**

## 仓库文件

| 文件 | 说明 |
| --- | --- |
| `index.html` | **主版本**（WorkBuddy 构建）。单文件、零依赖、纯离线，当前维护版本 |
| `audio-quality-analyzer-TRAE.html` | TRAE 构建的独立版本（备用） |
| [DEVELOPMENT.md](DEVELOPMENT.md) | **开发笔记与踩坑记录** —— 改 bug / 加功能前先读 |
| [CHANGELOG.md](CHANGELOG.md) | 更新记录 |

## 技术要点

- 解码：浏览器原生 Web Audio API（`decodeAudioData`）
- 元数据：自实现的文件头解析（WAV / FLAC / AIFF / MP3 / OGG / MP4）
- 分析：自实现 FFT（radix-2 + Hann 窗）
- 全部离线运行，打开即用，无需安装任何环境

## 浏览器兼容

解码能力取决于浏览器：

| 浏览器 | 支持情况 |
| --- | --- |
| Chrome / Edge | WAV、MP3、FLAC、OGG、OPUS、AAC(M4A) 等 |
| Safari | 上述全部 + ALAC、AIFF |
| Firefox | 主流格式（以实际解码能力为准） |

若某格式在当前浏览器无法解码，页面会给出提示 —— 换用 Chrome 或 Safari 即可。

## 在线版本（历史分享）

- 【WorkBuddy 版】https://workbuddy.link/p/jaWmqVwyre12UeunwCEOJz?ext2=copy_link
- 【TRAE 版】https://share.traecontent.cn/artifact/810XJV_51SVXR6

## 隐私

分析全程在浏览器本地进行，音频文件不会上传到任何服务器。

## 界面预览

<img width="1920" height="934" alt="image" src="https://github.com/user-attachments/assets/1f15f735-c358-4edd-a6e1-11ab53dfe542" />
<img width="1920" height="2298" alt="image" src="https://github.com/user-attachments/assets/ac4e1337-536f-46c5-8f46-0d471cb9950d" />
