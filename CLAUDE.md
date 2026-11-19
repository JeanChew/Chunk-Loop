# Chunk Loop：项目说明（给 Claude Code 看的交接文件）

## 用户和目标

- 用户在台北，中文母语，**编程零基础**。用简体中文交流，解释要具体、少术语，一次只给一个能直接照做的步骤。
- 痛点：真实母语者视频（vlog、采访、美剧、综艺、演讲）里有连读、弱读、吞音、背景人声、抢话，**循环很多遍也听不出原句**。不是要跟读评分。
- 用户网速很慢、常断线（会用代理）。**任何设计都要少上传、能断点续取、不让用户白等或白传。**
- 设备：公司 Windows（Edge/Chrome）、家里 Mac、iPhone。
- 网址：https://jeanchew.github.io/Chunk-Loop/ （GitHub Pages，main 分支根目录的 `index.html`）。

## 现在的架构

- **整个工具就是一个 `index.html`**：HTML + CSS + JS 都在里面，没有构建步骤、没有后端。改完提交到 main，GitHub Pages 一两分钟后自动更新。
- 用户的 ElevenLabs API key 存在浏览器 localStorage（`chunkloop:key`），只开了 Speech to Text 权限。**永远不要把 key 写进仓库。**
- 转写：ElevenLabs Scribe（`scribe_v3`，失败自动退回 `scribe_v2`），`POST https://api.elevenlabs.io/v1/speech-to-text`，`timestamps_granularity=word`、`diarize`、`tag_audio_events`。
  - YouTube：只发 `source_url`，ElevenLabs 自己下载，不占用户上传。YouTube 有时拒绝下载（Shorts 常见）。
  - 本地文件：页面先解码，转成 16 kHz 单声道 WAV 再上传（`compactAudio`）；超过 300 MB 的文件不自动上传，提示用户先用 QuickTime「导出为 → 仅音频」或 iPhone 快捷指令导出声音。
  - 先尝试异步：`webhook=true` 拿到 `transcription_id`，再轮询 `GET /v1/speech-to-text/transcripts/{id}`。进行中的任务存在 `chunkloop:pending`，页面重开会自动接着取。若被拒（错误含 webhook），设 `chunkloop:nosync=1` 以后直接同步等待。**这条异步路径只用模拟接口测过，还没用真 key 验证。**
  - 上传用 XHR 以显示真实上传进度；进度面板分步：抽出声音 / 上传 / ElevenLabs 转写（只有已等时间和粗估）/ 整理。
- 播放：YouTube 用 IFrame Player API（本地 file:// 打开会报错 153，必须用 Pages 网址）；本地音频解码成 WAV 后用 `<audio>`，循环边界精确；本地视频文件用 `<video>` 在侧栏显示画面（用户要看口型），声音另外解码只用于上传和导出单句。浏览器放不出画面时（如 Windows 上的 HEVC 录屏）自动退回只放声音。`MP` 对象统一两种播放器，YouTube 报时较粗，用插值并在跳转后防抖。
- 分句：`segment()` 按句末标点、停顿、说话人切换切句，长句按逗号、停顿、连词再切成 chunk。都是"建议切分"，用户可切开、合并、微调起止。
- "为什么听不出来"：`AZ.analyze()` 是**纯规则**（文字 + Scribe 时间戳），标出缩读（gonna 等）、同化（did you → didja）、闪音、连读（含 w/y 滑音、h 脱落）、吞音、失爆、弱读，并用时间戳给出"这个词只有 xx 毫秒"的证据。它是推测，不是从声音里听出来的。
- "问 Claude"：用户标记没听出的句子 → 一键生成提问 → 用户贴到 Claude 聊天 → 把回答（每句以 `#编号` 开头，编号是该句第一个词的索引）贴回页面，挂到对应句子下。
- 存储（**不自动删除，删不删由用户决定**）：IndexedDB 库 `chunkloop`（版本 2），表 `tx`（转写文字，旧版 localStorage 的 `chunkloop:tx:*` 启动时自动搬过来）、`media`（声音：音频 ≤300 MB 原样存，视频只存 22 kHz 单声道 WAV）、`video`（≤2 GB 的视频原文件，能放出画面时才存）、`handle`（Chrome/Edge 的文件位置，用 showOpenFilePicker 或拖放 getAsFileSystemHandle 拿到）。键都是转写 id（`yt-<videoId>` 或 `file-<文件名>-<大小>`）。启动时 `initStore()` 把文字和大小读进内存 `DB`。存前用 `navigator.storage.estimate()` 检查空间，并申请 `persist()`。最近列表 `chunkloop:recent`（不限条数，显示 8 条 + "显示全部"），标记与修改 `chunkloop:<id>:<词数>`，都在 localStorage。数据不在设备间同步。
- 从历史重开本地文件 `openFileItem()`：存着视频 → 直接放画面；记得文件位置 → 点一次「允许」；只存着声音 → 放声音，顶部提示「打开原视频」；什么都没存 → 提示「打开原文件」（写明不用网络、不收费）。没声音时点句子只让提示框闪一下，不再显示重复的红字。
- 历史列表每条有「删画面」（只删 video 表）和「删除」（整条：文字、标记、声音、画面、文件位置），都先 confirm。
- 测试钩子：`window.__cl` 暴露内部状态；`window.__CL_API` 可把接口指向本地假服务器。

## 已做的决定（以及原因）

- 转写用 ElevenLabs Scribe：用户实测比其他 App 准，能处理背景人声和多人说话。
- "听音频的分析"要换另一个模型，不让 Scribe 兼做。Claude API 目前不能直接收音频。Gemini 能听音频和 YouTube 链接，正在测试中：
  - 盲听测试：能听出具体口语，听不清处会标"听不清"，表现不错。
  - 陷阱测试（故意给写错的原文，看它会不会纠正）**还没做**，这决定能否把 Scribe 原文喂给它。
  - 注意：Gemini 对 YouTube 链接指定起止时间时，据报告音频仍按整个视频计费。所以以后只在本地音频模式下，把单句几秒的音频切出来发给它（`sliceWav` 已有）。
- UI 原则：默认只显示文字和播放键；点一句就循环；只有选中的那句展开分析和按钮。用户嫌复杂时宁可收起来。

## 待办（按优先级）

1. 用户用真 key 验证异步转写和上传进度。
2. 复习清单：跨视频收集"没听出"的句子，隔天、三天、一周推回来复听。
3. 听写模式：先打出听到的，再和原文对比，自动标出听错的词。
4. 备份与恢复：导出 / 导入一个文件，方便手机和电脑之间搬（存储已换 IndexedDB）。
5. 下一步（已和用户对齐）：大文件和 MKV 用 ffmpeg.wasm 在本地分块抽出声音、压成小格式再上传，解除 300 MB 限制。工具约 30 MB，只在拖进大文件时自动下载、能断点续传、先告诉用户大小；用户不用自己装任何东西。
6. 简化设置：每遍停顿（短/中/长）、备份与恢复；保留「导出字幕（SRT）」；新加「打印 / 存成 PDF」（全文+时间点，没听出的句子高亮并附分析和 Claude 解释，用浏览器打印实现）；余量、重新切分收进默认收起的"更多"。
7. 视 Gemini 陷阱测试结果，接入"听音频分析"（用户自带 Gemini key）。
8. 以后：TTS 清晰版（VoiceStudio 本地）、双耳 ASMR（binaural-voice，Python）——在用户 Mac 上做。

## 改代码时的要求

- 保持单文件、无构建；外部库只用公共 CDN。
- 改完要实际在浏览器里测（之前用 Playwright + 模拟的 YouTube、ElevenLabs 接口），包括手机宽度 390px。不要只看语法。
- 报错信息一律写成用户看得懂的中文，说明发生了什么、该怎么办。
- 涉及花钱（转写长视频）先告诉用户大概费用再继续。
