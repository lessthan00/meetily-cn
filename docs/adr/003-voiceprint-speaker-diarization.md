# ADR 003: 声纹识别与说话人分离架构

**日期**: 2026-05-17  
**状态**: 已接受（实施中 — 代码尚未完全对齐）  
**决策者**: AI + 用户协同

---

## 背景

meetily 是一款本地优先的会议记录与纪要工具。现有代码库基于 Rust 内嵌 Whisper 做转录，但缺乏以下能力：

1. **中文转录质量不足** — Whisper 对中文长尾词汇、专有名词识别欠佳
2. **说话人标注缺失** — 无法区分"谁说了什么"
3. **离线性要求** — 所有数据必须在本地处理，不得上传云端
4. **声纹注册与识别** — 需要通过声纹将说话人片段与真实姓名关联

## 决策

### 1. 转录引擎：FunASR (SenseVoiceSmall)

**弃用**: Rust 内嵌 Whisper、parakeet_engine

**选择**: FunASR (本地 Python 服务, 端口 8178)

**理由**:

| 维度 | Whisper (Rust) | FunASR (Python) |
|---|---|---|
| 中文准确率 | 中等 (~85% CER) | 优秀 (~95% CER, 中文专优) |
| VAD 集成 | 需手动拼接 | 内置 fsmn-vad |
| 声纹联动 | 需额外工程 | 生态内 CAM++ 无缝衔接 |
| 热词支持 | 有限 | 丰富 (IT、医疗、法律) |
| 部署难度 | 零 (内嵌编译) | 低 (pip install funasr) |

**后果**:
- Rust 代码库需一次性删除 `whisper_engine/`、`parakeet_engine/`、`audio/stt.rs`、`audio/transcription/`
- 新增 `funasr_client.rs` (HTTP 客户端)
- 用户需安装 Python 3.9+ 及 funasr 依赖
- **迁移策略**: 一次性全删旧引擎，不保留 Whisper 降级兜底（容灾见第 9 节）

### 2. 声纹引擎：CAM++ (合并进 FunASR 进程)

**选择**: 与 FunASR 共用 :8178 端口，不拆独立服务

**理由**:

| 维度 | 合并 (:8178) | 独立 (:8179) |
|---|---|---|
| CAM++ 模型大小 | 30MB（可忽略） | 30MB |
| 内存增量 | 无 | +1GB（Python 进程开销） |
| 并发冲突风险 | 低（声纹比对纯 CPU） | 无 |
| AI 编码复杂度 | 低 | 中 |
| 未来演进 | 可随时拆分 | 需重构前端 |

**失败模式与缓解**:
- 如声纹识别接口响应变慢 → 拆分为独立进程 + nginx 转发
- 当前阶段 (<10 人会议) 不会触发瓶颈

### 3. 说话人聚类（Speaker Clustering）

**功能**:
1. **已注册说话人** — CAM++ 提取声纹向量 → 余弦相似度与注册库比对 → 标注姓名
2. **未注册说话人** — 自动聚类为「说话人 A / 说话人 B / ...」，会后提示用户命名并注册声纹

**相似度阈值**: 默认 0.7，用户可通过前端设置页调整。阈值存 FunASR 侧 `config` 表 (`key='similarity_threshold', value='0.7'`)。`POST /config` 写入时同步更新内存变量，即时生效，无需重启

**聚类算法**: 在线 k-means 变体 + 自适应扩容

**k 值来源**: 会议开始前提示用户输入预期说话人数，默认 4。已知 k 消除自动判断人数的痛点，提升聚类稳定性。

**聚类流程**:

```
会议开始前 → 用户输入预期说话人数 k（默认 4）
→ 初始化 k 个空聚类质心
→ 每段语音提取 embedding 后:
   ├ 与声纹库匹配 → 余弦 ≥ 0.7 → 标注姓名（计入对应聚类）
   ├ 与 k 个质心比对 → 最近且 ≥ 0.7 → 归入该聚类
   ├ 与 k 个质心都 < 0.5 → 自动 k+1，新建聚类「说话人 X」
   └ 介于 0.5–0.7 → 归入最近聚类（模糊地带，容忍一次）
```

**自适应扩容**:
- 有人在开会后加入（未预先登记）→ 其 embedding 与所有质心相似度 < 0.5 → 自动 k+1
- 前端提示：「检测到未预期的说话人，已自动标注」
- 多个未登记说话人逐一出现时，分别自动扩容

**选择在线 k-means 而非贪心聚类的理由**:
- 已知 k 使质心分配精准，不会过度碎片化
- 比阈值贪心更抗噪声——偶发的低质量 embedding 不会错误建新聚类
- 用户填错或中途加人时，自适应扩容兜底

### 4. 摘要引擎：Ollama + qwen3:8b

**弃用**: Claude API、Groq API、OpenRouter API

**选择**: 仅 Ollama 本地部署，复用 `meeting-minutes-cn/backend/` 代码

**理由**:
- **保密性优先** — 会议内容不得离开本机
- qwen3:8b 中文摘要质量优秀，8B 参数可在 16GB RAM 上流畅运行
- `meeting-minutes-cn` 已有成熟的 Ollama-only + 中文摘要流水线

### 5. 部署方式：一键脚本

**弃用**: Tauri Sidecar (PyInstaller 打包)、Docker Compose

**选择**: `start.bat` / `start.sh`

**理由**:

| 维度 | 一键脚本 | Sidecar | Docker |
|---|---|---|---|
| 用户体验 | 一次点击 | 一次点击（无感） | 一次点击 |
| 预装依赖 | Python + pip | 无 | Docker Desktop |
| GPU 利用 | 自动 | 难处理 | 自动 |
| AI 编码复杂度 | 低 | 极高 | 中 |
| 跨平台 | 天然 | 不可跨平台编译 | 天然 |
| 热更新/调试 | 随改随试 | 需重打包 | 需重建镜像 |

### 6. 声纹注册流程

**文本策略**: 使用固定朗读文本，尽管 CAM++ 是文本无关模型。理由是：固定文本可以验证用户确实在录音（防止空录音/环境噪音误注册），并能检测音频质量是否合格。

```
用户点击"注册声纹"
→ 显示固定朗读文本（2-3 句, ~10 秒）
→ 麦克风录音 (16kHz, mono)
→ POST /enroll {name, audio}
→ 服务端校验: 音频时长 ≥ 3s, 信噪比合格
→ CAM++ 提取 256 维声纹向量
→ 存入 SQLite speaker_embeddings 表
→ 反馈"注册成功"或"语音太短/质量不足，请重试"
```

### 7. 实时会议流程

**VAD 策略**: Rust 不做 VAD。Rust 以固定 2 秒 chunk 发送原始音频（16kHz PCM），由 FunASR 内置的 fsmn-vad 独做语音活动检测。避免 Rust 和 Python 两套 VAD 算法切割边界不一致的问题。

**静音 chunk 处理**: Rust 对音频内容做零判断——2s 到了就发，不预判是否有语音。fsmn-vad 处理静音 chunk 耗时可忽略 (<5ms)，无需 Rust 侧 RMS 预判。单向责任：Rust 只管发送，FunASR 决定有没有语音。

**API 调用策略**: 实时模式下使用 `POST /process-audio?mode=realtime` 一次调用，同时返回转录文本 + 说话人标注。不再串行调用 `/transcribe` + `/identify`（减少一个 HTTP 往返，避免"转录先到、声纹后到"的竞态）。

**会话管理**: Rust 在开始录音时调用 `POST /session/start` 传入 k 值（前端输入），获得 `session_id`。后续每个 2s chunk 的 `/process-audio` 请求携带 `session_id`，FunASR 据此定位对应的 k-means 实例。停止录音时调用 `POST /session/end` 获取聚类摘要，销毁内存状态。

**说话人切换处理**: fsmn-vad 将音频切割为语音段后，每个语音段独立跑 CAM++ 提取声纹。因此一个 2s chunk 可能返回多个 segment，每个 segment 可标注不同说话人。避免抢话场景下声纹被混淆。

```
用户点击「开始录音」→ 前端 POST /meetings {title, expected_speakers: k}
  └→ FastAPI 创建 meeting 行 (transcription_status='recording') → 返回 meeting_id

前端 invoke('start_recording', {meeting_id, k: 4, mic: "...", system: "..."})
  └→ Rust 调 POST /session/start {meeting_id, k}
     └→ FunASR 初始化 k-means → 返回 session_id
  └→ Rust 创建 sidecar JSON (~/.meetily/recovery/{meeting_id}.json)

麦克风 → Rust 采集 (16kHz)
→ 每 2 秒 chunk:
   POST /process-audio?mode=realtime {audio_wav, session_id}
      └→ fsmn-vad 切割语音段 (200ms–2s)
         └→ 逐段: SenseVoiceSmall → 文本 + CAM++ → 声纹
            └→ 余弦比对
               ├ 匹配 → 标注注册姓名
               ├ 未匹配但已有聚类 → 标注"说话人 A"
               └ 新说话人 → 增量聚类 → 标注"说话人 B"
→ 返回 segments: [{start, end, text, speaker}, ...]
→ 前端渲染: "张甜甜: 我觉得这个方案可行\n李四: 对，我也这么想"

停止录音 → Rust 调用 POST /session/end {session_id}
           └→ FunASR 返回聚类摘要 → 销毁会话
           └→ Rust emit 给前端 → 弹出「未注册说话人命名」界面
```

**交互控制**: 录音通过应用窗口按钮和系统托盘菜单控制（右键托盘图标 → 开始/停止录音）。默认不注册全局快捷键，避免与其他应用冲突。可选的全局快捷键（如 `Ctrl+Alt+R`）留待 v1.1 的 settings 功能。

**会后流程**:

```
用户点击停止录音
→ Rust 调 POST /session/end → FunASR 返回聚类摘要 → 销毁 session
→ Rust emit "session-ended" 给前端 (聚类摘要)
→ 弹出「未注册说话人命名」界面
   └ 说话人 A → 用户输入姓名 → 确认
   └ 说话人 B → 用户输入姓名 → 确认
→ Rust apply_speaker_names (内存中替换临时标签 → 真实姓名)
→ Rust POST :5167/save-transcript {meeting_id, segments: [{start, end, text, speaker}, ...]}
   └→ FastAPI 写入 meetings.transcript_json 列
→ 每位命名后提示：「是否为「李四」注册声纹？」
   ├ 是 → 跳转到声纹注册页 → 朗读固定文本 → POST /enroll
   └ 否 → 仅保存显示名，本次会议使用该名称
→ 命名全部完成后，回到会议详情页
→ 自动调用 Ollama 生成会议标题（轻量 prompt，~1s 返回）
   └ 默认标题为日期时间（如「2026-05-19 14:30」），Ollama 就绪后替换
→ 转录面板显示最终命名的说话人
→ 用户手动点击「生成摘要」→ POST /process-transcript → Ollama 生成纪要
```

> **关键决策**:
> - 会议开始时不要求用户输入标题，默认用日期时间命名
> - 摘要不自动生成——用户手动触发。避免每次录音结束都触发 Ollama（GPU 资源占用 + 可能不需要每次都生成）
> - 标题生成是轻量操作（10 字以内，~1s），可以自动触发

**系统音频的说话人标注** (v1/v2):

Meetily 采集两路音频：麦克风 + 系统音频（远程会议中远端同事的声音）。v1 不做区分——两路混合后统一聚类。纯线下会议无影响；混合会议中同一个人可能被分到两个聚类（两路声纹略有差异），会后命名时用户手动合并。

v2 计划：
- Rust 发送 chunk 时携带 `source: "mic" | "system"`
- FunASR 按 source 分两个独立 k-means 实例
- 前端区分「本地」和「远端」标注

v1 降低冲突：同一注册姓名同时命中两路时，优先保留麦克风匹配，系统音频降级为聚类标签。

### 8. 导出

会议结束后可通过前端导出。

**音频**: 全程录音自动保存为 WAV 文件。前端提供「导出音频」按钮，直接下载原始录音。

**Markdown 导出**:

| 导出内容 | 文件名 | 格式 |
|---|---|---|
| 转录 | `transcript.md` | `**张甜甜** (00:00:05): 文本内容` |
| 纪要 | `summary.md` | `# 会议纪要\n## 要点\n...` |
| 完整 | `meeting-full.md` | 转录 + 纪要在同一文件 |

实现位置：FastAPI 摘要服务 (:5167)，`GET /export/{meeting_id}?format=md`

---

## API 接口汇总

### FunASR + 声纹服务 (:8178)

| 方法 | 路径 | 功能 |
|---|---|---|
| POST | `/process-audio?mode=realtime` | **实时音频 chunk** → 转录 + 说话人标注（主要方式） |
| POST | `/process-audio?mode=offline` | **离线音频文件** → 转录 + 说话人标注 |
| POST | `/enroll` | 注册声纹 {name, audio} |
| POST | `/identify` | 独立识别说话人 {audio}（管理/调试用） |
| POST | `/transcribe` | 独立转录 {audio}（管理/调试用） |
| GET  | `/speakers` | 获取声纹库列表 |
| DELETE | `/speakers/{id}` | 删除声纹 |
| POST | `/speakers/export` | 导出加密声纹文件 (AES-256-GCM)。请求体: `{password}` → `.json.enc` 文件 |
| POST | `/speakers/import` | 导入加密声纹。multipart: `password` + `file` → 解密后增量合并（同名覆盖） |
| GET  | `/config` | 读取配置 (如 `similarity_threshold`) |
| POST | `/config` | 更新配置 `{key, value}`。即时生效 |
| POST | `/session/start` | 初始化 k-means 会话 {meeting_id, k} → {session_id} |
| POST | `/session/end` | 结束会话，返回聚类摘要 {session_id} → {speakers: [{label, segment_count, total_duration}]}，销毁内存状态 |
| GET  | `/health` | 健康检查 + 模型下载进度上报。响应: `{"status": "ok"}` / `{"status": "loading", "model": "SenseVoiceSmall", "progress": 45.0}` / `{"status": "error", "message": "..."}` |

> **注**: `/process-audio?mode=realtime` 是实时会议的主入口。`/transcribe` 和 `/identify` 保留为独立端点，用于声纹注册引导、调试等非实时场景。
>
> **会话生命周期**: `/session/start` → 多次 `/process-audio?mode=realtime` (携带 session_id) → `/session/end`。聚类状态（k-means 质心 + 标签分配）仅在 FunASR 内存中存在，`/session/end` 返回摘要后销毁。`/session/end` 的响应是前端弹出「未注册说话人命名」界面的数据来源。

**声纹导出/导入**:

用于跨设备迁移声纹库。导出为单个 JSON 文件，包含所有已注册说话人的 256 维向量。

```json
{
  "version": 1,
  "exported_at": "2026-05-19T10:00:00",
  "speakers": [
    {"name": "张甜甜", "embedding": [0.12, 0.34, ...], "created_at": "..."},
    {"name": "李四", "embedding": [0.08, 0.21, ...], "created_at": "..."}
  ]
}
```

- `GET /speakers/export` — 返回 AES-256-GCM 加密文件 (`.json.enc`)
- `POST /speakers/import` — 接收加密文件 + 密码，解密后增量合并（同名覆盖，不同名新增）
- 前端「声纹管理」页增加「导出全部」和「导入」按钮
- 导出/导入时弹出密码输入框
- **加密方案 (grill-with-docs 确认)**: AES-256-GCM + PBKDF2 口令派生。GCM authentication tag 防篡改——密文被修改时导入端检测到并拒绝。用户自行保管密码，无需密钥管理基础设施
- 导入时提示：「声纹数据包含生物特征，请确认来源可信」

### 9. 容灾与恢复

FunASR 服务崩溃时，Rust 侧按错误类型分层处理，而非一概而论。

**崩溃检测 (断路器)**

| 错误类型 | 判定 | 处理 |
|---|---|---|
| `ECONNREFUSED` (连接拒绝) | 进程已死 | 立即标记 `partial` |
| `ECONNRESET` (连接重置) | 进程崩溃中 | 立即标记 `partial` |
| `ETIMEDOUT` (超时) | 可能 GPU hang | 重试 1 次，仍超时则标记 `partial` |
| HTTP 5xx (服务内部错误) | 活着但出错 | 连续 3 次后标记 `partial` |

**断路器阈值**: 任意错误类型触发标记后，断路器打开，停止发送后续 chunk，避免无意义的重试风暴。

**超时配置**:
- 连接超时: 5s (TCP 握手)
- 请求超时: 10s (GPU 推理，正常 <1s)

**容灾流程**:

1. **音频保存**: 断路器打开后立即将已录制音频保存为 WAV 文件，数据库 meeting 表字段：
   - `transcription_status`: `recording` | `complete` | `partial` (崩溃标记为 `partial`)
   - `audio_file_path`: 完整原始录音路径
   - 崩溃前已转录的 segments 保留不动

2. **健康探测**: Rust 侧使用渐进式间隔探测 `GET /health`——首分钟 2s 一次（模型下载期间快速反馈），稳定后 15s 一次。连续 2 次成功 → 认为服务已恢复 → 前端弹出横幅：「检测到服务已恢复，点击重新处理」

3. **恢复触发**:
   - **自动**: Rust 启动时 / 健康探测恢复后扫描 `partial` 状态的会议，前端弹出横幅
   - **手动**: 会议详情页顶部横幅 + 「重新处理」按钮

4. **重处理**: 调用 `POST /process-audio?mode=offline` 处理完整音频文件。CAM++ 缓存随 FunASR 崩溃而销毁，因此：
   - 崩溃前已转录部分：保留原有文本 + 说话人标注
   - 崩溃后未转录部分：获得文本，说话人标记为「说话人 1」、「说话人 2」（按聚类自动编号）
   - 前端明确标注：「说话人信息因服务中断而不完整」

5. **v2 改进方向**: Rust 侧 SQLite 缓存最近 50 条声纹 embedding → FunASR 重启后自动 `/enroll` 恢复

### 摘要服务 (:5167)

| 方法 | 路径 | 功能 |
|---|---|---|
| POST | `/meetings` | 创建 meeting，返回 meeting_id。请求体: `{title, expected_speakers}`。转录开始前由前端调用 |
| POST | `/save-transcript` | 保存最终 segments (带最终姓名) 到 meetings 表 `transcript_json` 列。会后命名完成后由 Rust 调用 |
| POST | `/process-transcript` | 生成摘要 (Ollama)。读 `transcript_json` 列 → 摘要 prompt → 写入 `summary_json` 列 |
| POST | `/generate-title` | 生成会议标题 (Ollama, 轻量, ~1s) |
| GET  | `/export/{meeting_id}?format=md` | Markdown 导出。读 `transcript_json` + `summary_json` |

> **阈值策略 (grill-with-docs 确认)**: 相似度阈值默认 0.7，用户可通过前端设置页修改，值存入 FunASR 侧 `speaker_embeddings` SQLite 的配置表。**v1 仅启动时读取**——在线会议中修改阈值的生效需重启 FunASR。v2 加强热更新。详见 #03 issue。

---

## 替代方案

### 为什么不用 pyannote-audio？

- pyannote 是 HuggingFace 上的说话人分离库，性能好但模型需从 HF Hub 下载（联网）
- 离线场景下需预下载模型（> 1GB），且接口与 FunASR 生态不一致
- CAM++ + FunASR 的 VAD 已覆盖相同需求，模型更小（30MB）

### 为什么不用 WhisperX？

- WhisperX 支持说话人分离，但中文转录质量不如 SenseVoiceSmall
- 需要额外的说话人注册逻辑，工程量大

### 为什么保留 Tauri 而不是纯 Web？

- Tauri 提供系统麦克风权限、系统托盘、开机自启等桌面能力
- 已是现有代码库的核心框架，迁移成本高

---

## 实施顺序

1. 搭建 FunASR + CAM++ 服务 (Python, :8178) — `backend/app/main.py` 按本 ADR 的 API 表对齐
2. 搭建 Ollama 摘要服务 (Python, :5167)
3. 修改 Rust 转录客户端 (`funasr_client.rs`) — 端点命名对齐本 ADR
4. 一次性清理 Rust 旧转录代码 (Whisper / Parakeet)
5. 实现容灾恢复流程（数据库标记 + 音频保存 + 重处理按钮）
6. 修改前端 UI（声纹注册页、说话人面板、导出按钮、崩溃恢复横幅）
7. 编写一键启动脚本 (`start.bat` / `start.sh`)
8. 端到端测试（含 FunASR 手动 kill 崩溃恢复测试）
