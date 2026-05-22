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

---

## §10 A3 详细设计 (grill-with-docs 逐项记录)

以下决策来自 A3 (详细设计) 的逐项讨论，是对本 ADR 架构决策 (A2) 的下层细化。每条均经过过程纪律 + A1 + A2 回溯检查。

### Q1 — Segment 数据格式

**决策**：

```typescript
interface Segment {
  start:   number;        // float 秒，FunASR SenseVoice 原生输出
  end:     number;        // float 秒
  text:    string;
  speaker: string | null; // null=无法识别, 非null=姓名或聚类标签
}
```

| 决策点 | 选择 | 理由 |
|---|---|---|
| 时间单位 | float 秒 | FunASR 上游原生输出，零转换，全链路不做时间算术 |
| speaker 默认 | `string \| null` | 三态语义：`null`=无法识别、`"...姓名"`=已注册、`"说话人 A"`=聚类标签。TypeScript 对 null 有编译期强制检查 |
| 段间边界 | 允许 gap | fsmn-vad 丢弃静音段，gap 如实反映原始音轨，不伪造无缝衔接 |

**回溯检查**：
- 过程纪律: 02a (segment deserialization) TDD 子任务，类型已定 → Spec 就绪 无环境依赖
- A1: FR #1/#3 均满足；null 语义映射三态需求
- A2: ADR-003 定过 `{start, end, text, speaker}` 四字段，细化类型不改字段名。无矛盾

### Q2 — Sidecar JSON 格式

**决策**：JSONL 格式 (每行一个 segment)，追加 `seq` + `named` 字段。

```jsonl
{"seq":0,"start":0.0,"end":2.3,"text":"大家好","speaker":"说话人 A","named":false}
{"seq":1,"start":3.2,"end":5.1,"text":"讨论Q2","speaker":"张甜甜","named":true}
```

| 决策点 | 选择 | 理由 |
|---|---|---|
| 格式 | JSONL (非完整 JSON 数组) | 追加写只需 `O_APPEND`，无需读全文件→改数组→回写；崩溃后文件自然截断到最后完整行，残行丢弃。一个残行 vs 整个 JSON 数组损坏 |
| `seq` | int (递增序号) | 恢复时用于跳过已处理段，比用时间戳去重可靠 |
| `named` | bool (默认 false) | 区分"聚类标签等待命名"与"已确认姓名待落 DB"。恢复时只重试 `named=true` 的 save-transcript |

**回溯检查**：
- 过程纪律: sidecar 非 TDD 子任务，属 B4 集成接线。正确性由 E2E 冒烟和崩溃恢复测试验证。无矛盾 无环境依赖
- A1: 补充需求「每 5 分钟增量追加 segments 到 sidecar。sidecar 是 segments 恢复的唯一载体」。JSONL 符合增量追加，崩溃截断后残行丢弃比 JSON 数组全损强一个数量级。无矛盾
- A2: ADR-003 §7「Rust 创建 sidecar JSON (~/.meetily/recovery/{meeting_id}.json)」— 定了路径和类型 (JSON)，未定数组 vs JSONL。细化不改格式大类。无矛盾

### Q3 — WAV 双路存档

**决策**：单个 48kHz stereo WAV (左声道=mic，右声道=system)。

| 决策点 | 选择 | 理由 |
|---|---|---|
| 存档格式 | 单 stereo WAV | WAV 原生支持多声道；一个 meeting = 一个录音文件，管理简单；崩溃恢复时只需传一个文件路径；v2 source-aware 聚类时分离声道比拆两个文件简单 |
| 混音送转录 | Rust 侧 mono downmix `(L+R)/2` → 48k→16k 重采样 | 复用现有 `audio_processing::resample_audio()` |

**回溯检查**：
- 过程纪律: 非 TDD 子任务，属 B4 管线重构。正确性由 E2E 冒烟验证。无矛盾 无环境依赖
- A1: 补充需求「48kHz WAV，边录边写」+「双路存档」。单 stereo WAV 满足双路存储，WAV chunk-by-chunk 追加写可行。无矛盾
- A2: ADR-003 §7「双路 48kHz 录音 (chunk-by-chunk 写磁盘) + 16kHz 混音送转录」。未指定单立体声 vs 双单声道。细化不改架构。无矛盾

### Q4 — Session 超时清理

**决策**：遵从 ADR-003 §7 已定核心逻辑，补充两个细节。

| 决策点 | 选择 | 理由 |
|---|---|---|
| 核心逻辑 | 每 60s 扫描，`last_access` > 4h 自动过期销毁 | A2 已定，不做修改 |
| `last_access` 更新时机 | 每次 `/process-audio?mode=realtime` 携带 `session_id` 时刷新 | 自然对齐——有音频来=会话活跃 |
| 新增端点 | `GET /session/{id}/status` → `{"status": "active", "ttl_remaining": 1800}` | 前端/调试可用，非必需 |
| 过期告警 | v1 不做。v2 考虑 heartbeat | 录音中 session 静默 4h 的边界场景实际不存在 (录音连续发 chunk) |
| 过期后果 | 仅丢失内存中的 k-means 状态。磁盘 WAV 和 sidecar JSONL 已由 Rust 独立保护 | 容灾策略的分层：FunASR 内存是缓存，不是真理源 |

**回溯检查**：
- 过程纪律: session 管理非 TDD 子任务，属 C3.5 集成。正确性由 session 生命周期集成测试 (start→process→end，验证 session_id 不丢失) 验证。无矛盾 无环境依赖
- A1: FR #3 (说话人标注) + 补充需求 (会议时长常超 2h)。4h = 2h 会议 × 2 倍缓冲。无矛盾
- A2: ADR-003 §7「后台 asyncio task 每 60s 扫描，last_access 超 4h 自动过期」已定核心逻辑。补充细节不改变核心。无矛盾
- **标记**：`GET /session/{id}/status` 是新端点，需同步更新 `docs/api/funasr.md`。标记为待办——A3 完成后统一回写

### Q5 — 断路器状态机

**决策**：半开探测 5 条规则，全部遵从 ADR-003 §9 已定框架。

| 决策点 | 选择 | 理由 |
|---|---|---|
| 探测成功阈值 | 连续 2 个成功才切到关闭 | 1 个探活可能赶上瞬时恢复又崩 → 反复开闭 (flapping)。A2 已定「连续 2 次成功」 |
| 半开时正常 chunk 请求 | 继续丢弃 | 半开是探测阶段不是恢复阶段，确认健康后才恢复发送。否则刚发一个 chunk 又崩溃，chunk 丢了还没 sidecar 保护 |
| 探测失败后间隔 | 回到首分钟 2s，从头走渐进 (2s → 15s) | 崩溃后快速重试，符合「渐进式探测」意图，不必等 15s |
| 状态持久化 | Rust 内存，不持久化 | Tauri 崩 = 断路器也没了，重启从零开始。Rust 不持有 SQLite (编程约束 #9) |
| 探测超时 (ETIMEDOUT) | 视为失败 | 半开只看「通了没通」，不区分超时和拒绝 |

**回溯检查**：
- 过程纪律: 05a-circuit-breaker 是 TDD 子任务。3 状态 × 5 错误类型 + 半开逻辑组合爆炸 → 05a 只测纯状态转移表，错误分类在集成阶段人工验证。无矛盾，但需细化 05a spec cargo test 增量 < 60s。无矛盾
- A1: FR #6「断路器 + 渐进式健康探测 + resume_from 续传」。5 条全部在需求范围内。无矛盾
- A2: ADR-003 §9 已定「半开 + 2s 探活 + 稳定 15s + 连续 2 次成功」。逐条对齐无矛盾

### Q6 — ChunkAccumulator 边界行为

**决策**：三条边界行为。

| 边界 | 场景 | 决策 | 理由 |
|---|---|---|---|
| 最终不足 2s 的余数 | 录音停在 1.3s，buffer 有 20800 samples | flush 时吐出最后一个不足 2s 的 chunk | FunASR fsmn-vad 不依赖固定 2s，给 1.3s 照样切出语音段。丢弃 = 丢内容 |
| 首个 chunk 前 buffer 不满 2s | 录音刚开始 | 不发。`accumulate(samples)` → `Option<Vec<f32>>`，不满 32000 返回 `None` | 首 2s 后才能出第一个字幕。2s 初始延迟对用户体验可接受 |
| 跨 chunk 语音段拼接 | fsmn-vad 在 chunk 边界切错 | Rust 侧不处理跨 chunk 拼接。每个 chunk 独立发独立返回 | 丢失 ≤ 200ms (VAD 最小窗口)，v1 接受，v2 在 FunASR 侧做跨 chunk VAD 状态机 |

**回溯检查**：
- 过程纪律: 02c 是 TDD 子任务。三条边界正好是 02c 测试应覆盖的——flush 不足 2s chunk、首个 chunk 返回 None、跨 chunk 独立处理。每个边界可写一条 assert。无矛盾 无环境依赖
- A1: FR #1 实时转录每 2s chunk。flush 不足 2s 是收尾不是常态，不违反固定 2s 语义。无矛盾
- A2: 管线 B1 定了 ChunkAccumulator =「收 16kHz f32 → 满 32000 → 吐 chunk → 余数滚存」。flush 和首 chunk 行为是细化语义，不改接口。无矛盾

### Q7 — 三级说话人标注：匹配/聚类/扩容

**决策**：5 条细化规则，全部在 ADR-003 §7 框架内。

| 决策点 | 选择 | 理由 |
|---|---|---|
| CAM++ 向量提取时机 | 与 transcription 同一次 `/process-audio` 调用返回 | ADR-003 §7 已定「一次调用同时返回转录+说话人标注」。每个 2s chunk 返回 segments 时，每个 segment 已带 embedding 和 speaker 字段 |
| ≥0.7 匹配粒度 | 逐 segment 与已注册声纹库逐一比较 | 注册声纹量级小 (≤ 20 人)，O(k×n) 可接受 |
| k-means 聚类模式 | 实时增量更新 (每 segment 即更新质心) | 每 2s chunk ≤ 3 segments，增量开销可忽略。批处理反而引入"多长跑一次"的延迟参数 |
| 自适应扩容阈值 | 余弦距离 < 0.5 (同一人) / ≥ 0.5 (不同人) → 创建 k+1 | CAM++ 论文基准同人余弦相似度 ≥ 0.7 (距离 ≤ 0.3)。放宽到 0.5 给 k-means 容错空间 |
| 命名落 DB | 用户命名后 → FunASR SQLite 存 (name, embedding)。不追溯更新已记录 segments 的 speaker 字段 | 旧 segments 保持聚类标签 "说话人 A"，避免"命名一个=更新历史全部"的复杂一致性 |
| 阈值存储 | FunASR `config` K-V 表。两个键：`similarity_threshold` (0.7) + `cluster_create_threshold` (0.5) | 启动时读取。幂等初始化 `INSERT OR IGNORE` |
| 阈值热更新 | `POST /config {key, value}` 写入 DB + 同步更新 FunASR 内存变量，即时生效 | 在线调参不需重启 |

**回溯检查**：
- 过程纪律: 03a/03b 是 TDD 子任务。决策定了 03a/03b 的输入输出和阈值。无矛盾 无环境依赖。无矛盾
- A1: FR #3「3 级决策」。7 条全部在需求范围内。无矛盾
- A2: ADR-003 §7 定了 CAM++ + 余弦相似度 + k-means + SQLite。细化不改框架。无矛盾

### Q8 — 声纹注册流程

**决策**：5 条细化规则，全部在 ADR-003 §7 框架内。

| 决策点 | 选择 | 理由 |
|---|---|---|
| 朗读轮次 | 单轮即可 (朗读"北风与太阳"一次，约 30s) | CAM++ 3-5s 语音即稳定提取 embedding。150 字远超最低要求，多轮平均边际收益极低 |
| 语音质量检查 | v1 不做 | 用户注册时通常安静环境。v2 加 SNR 阈值 |
| 存储结构 | 一对多 (一个姓名可存多个 embedding 样本) | 同一人不同设备/距离/mood 语音特征漂移，多样本提高匹配率。匹配取与所有样本的最高相似度 |
| 重名检测 | 覆盖旧样本。弹确认框后删除旧 embedding 重新提取 | 语义：「我重新录一次自己的声纹」= 替换，不是「加一个样本」 |
| 注册端点 | `POST /voiceprints/register` → FunASR SQLite | FunASR 是声纹数据唯一 owner，不经过 FastAPI 中转 |
| 首次声纹库为空 | 会议开始前弹窗：「你还没有注册过声纹，是否现在注册？」→「跳过」/「去注册」 | 引导而非强制。跳过 → 正常开会，全部聚类标签 |

**回溯检查**：
- 过程纪律: 声纹注册非 TDD 子任务 (注册页属前端 UI)。正确性由人工验证。无矛盾 需 FunASR 运行。Issue-0 已覆盖依赖安装。无矛盾
- A1: FR #2「固定文本朗读，CAM++ 256 维向量，FunASR 侧 SQLite」。6 条全部在需求范围内。无矛盾
- A2: ADR-003 §7「声纹注册：用户朗读固定文本 → CAM++ → 256 维向量 → 存 FunASR SQLite」。细化不改架构。无矛盾

### Q9 — 会后命名流程

**决策**：5 条细化规则，全部在 ADR-003 §7 框架内。

| 决策点 | 选择 | 理由 |
|---|---|---|
| 命名时机 | 会议中即允许命名 + 会后弹窗批量命名 | 会中点名 (即刻生效) + 会后扫尾 (统一处理未命名)。互补，非替代 |
| 会中触发方式 | 点击说话人标签 → 弹出输入框或选择已注册声纹 | 两路径：输入新姓名 (创建声纹) / 选择已有姓名 (关联声纹) |
| 会后触发方式 | `/session/end` 返回聚类摘要 → 弹出未注册说话人汇总 (label + segment_count) → 用户批量输入姓名 | 数据来源：session/end 的 speakers 数组。已命名 (named=true) 自动排除 |
| 跳过支持 | 允许关闭弹窗不命名，保留聚类标签 | 不强制命名。详情页可随时重命名 |
| 是否重新聚类 | 不重聚类 | k-means 在 FunASR 内存中增量更新 (Q7)，命名不改变 embedding 分布 |
| 新 segments 自动匹配 | 自动匹配 | 命名后后续 segments 即刻用已注册声纹匹配 |
| sidecar 中命名信息 | 通过 `named` 字段标记 (Q2)。新 segments speaker 直接写姓名 + named=true | sidecar 是 segments 恢复载体，不是操作日志 |

**回溯检查**：
- 过程纪律: 会后命名不属 TDD 子任务 (命名 UI 属前端布局)。无矛盾 依赖 FunASR SQLite + FastAPI。Issue-0 已覆盖。无矛盾
- A1: FR #4「聚类标签 → 姓名，命名后落 DB，sidecar 防丢」。5 条全部对齐。无矛盾
- A2: ADR-003 §7「k-means 临时标签 + 会后/实时命名 + 存 FunASR SQLite」。细化不改架构。无矛盾

### Q10 — resume_from 续传机制

**决策**：5 条细化规则，全部在 ADR-003 §9 框架内。

| 决策点 | 选择 | 理由 |
|---|---|---|
| 续传起点 | 从 sidecar 最后一条 segment 的 `seq` 继续。下一次 `/process-audio` 带 `resume_from=seq+1` | Q2 已定 `seq` 字段，自然用于续传对齐 |
| k-means 状态恢复 | 不恢复，重新聚类 | session 仅 FunASR 内存 (Q4)，crash 后质心丢失。sidecar 中旧 segments 的 speaker 字段保留，聚类从零重建 |
| 声纹匹配 | SQLite 持久化 → crash 后声纹库仍在，匹配继续 | 聚类重建、匹配继续，两个系统独立恢复 |
| seq 去重 | 前端侧按 `meeting_id + seq` 去重 | 简单且在数据源头附近。Rust emit 所有 segments，前端过滤 |
| resume_from 参数 | `POST /process-audio?mode=realtime&session_id=X&resume_from=42`。可选，省略时从 0 开始 | FunASR 侧 session 丢失 → 创建新 session → resume_from 对齐 seq |

**回溯检查**：
- 过程纪律: 续传不属 TDD 子任务，属 B4 集成接线。正确性由 E2E 冒烟验证。无矛盾 依赖 sidecar JSONL 可读。无环境依赖
- A1: FR #6「resume_from 续传」。5 条全部对齐。无矛盾
- A2: ADR-003 §9「resume_from 从 sidecar 恢复未发送 chunk → HTTP 续传」。细化不改架构。无矛盾

### Q11 — 声纹导入/导出

**决策**：5 条细化规则，全部在 ADR-003 §7 框架内。

| 决策点 | 选择 | 理由 |
|---|---|---|
| 导出格式 | JSON 数组。每项 `{name, embedding[256], created_at}` | 明文、人可读、256 维 float32。200 个声纹 ≈ 200KB |
| 重名处理 | 三选项弹窗：跳过 / 覆盖 / 追加为多样本。默认追加 | Q8 一对多结构天然支持多样本 |
| 导出粒度 | 全量导出 | 声纹量级小 (≤ 20 人)，v1 简化 |
| 导入验证 | 逐条验证。embedding 维度 ≠ 256 → 跳过并报告。name 缺失 → 跳过并报告。全部失败 → 不回滚。部分成功 → 导入成功的、报告失败的 | 防御性导入，不因一条坏数据阻塞全部 |
| 触发侧 | 前端触发。Tauri 命令 `export_voiceprints` / `import_voiceprints` → FunASR API | 用户选择文件路径，Tauri 文件对话框天然支持 |

**回溯检查**：
- 过程纪律: 声纹导入导出不属 TDD 子任务。无矛盾 依赖 FunASR `/voiceprints` 端点。Issue-0 已覆盖。无矛盾
- A1: FR #5「v1 明文 JSON，v2 加密」。v2 不在本次范围。无矛盾
- A2: ADR-003 §7「声纹存储在 FunASR SQLite」。IO 通过 FunASR API，不直接操作 SQLite。无矛盾

### Q12 — Markdown 导出格式

**决策**：5 条细化规则。

| 决策点 | 选择 | 理由 |
|---|---|---|
| 内容结构 | header (标题+日期) + 可选 `## AI 摘要` + `## 完整转录` + `**姓名** (HH:MM:SS): 内容` | 已生成摘要时展示摘要 section，未生成则跳过 |
| 摘要来源 | FastAPI `meetings.summary_json` (已存储的摘要文本) | 导出时从 DB 读取，与转录一起写入 Markdown |
| 时间戳 | 绝对时间 HH:MM:SS，从录音 00:00:00 起算 | SRT/VTT 等字幕格式同用绝对时间。Q1 的 float 秒直接除以取模 |
| speaker = null | 显示 `**未知**` | 与"说话人 A"区分——null = 无法识别，聚类标签 = 未命名 |
| 触发方式 | 前端按钮 → Rust `export_markdown(meeting_id)` → 读 sidecar JSONL + FastAPI summary → 写文件到用户选择路径 | 双数据源：转录从 sidecar，摘要从 FastAPI |
| 导出时机 | 允许会议进行中导出 | sidecar 已有 segments 立即可导出，无技术阻塞点 |

**回溯检查**：
- 过程纪律: Markdown 导出不属 TDD 子任务。无矛盾 依赖 sidecar JSONL。无环境依赖
- A1: FR #8「`**姓名** (HH:MM:SS): 内容`」。5 条全部对齐。无矛盾
- A2: ADR-003 未定义导出格式 (属 A3)。无矛盾

### Q13 — 离线音频导入

**决策**：5 条细化规则。

| 决策点 | 选择 | 理由 |
|---|---|---|
| 导入流程 | 选文件 → ffmpeg 转 16kHz mono WAV → `POST /process-audio?mode=offline` → 返回全部 segments | 与实时转录共用同一 FunASR 端点，mode 区分。不经过 ChunkAccumulator |
| 大文件 | v1 单次直传。v2 分片上传 | 离线导入非热路径，用户可接受等待 |
| ffmpeg 定位 | `tauri::api::path::resource_dir()` 定位 bundled ffmpeg 二进制 | 补充需求「ffmpeg 随应用自带」 |
| session/sidecar | FunASR 创建临时 offline session → 返回带 speaker 的 segments → Rust 创建 sidecar JSONL | 命名/导出/Markdown 等后续操作与实时会议一致 |
| 输入格式 | 不限制后缀。ffmpeg 检测 → 无法转码 → 报错 | ffmpeg 支持几乎所有格式，后缀白名单无意义 |

**回溯检查**：
- 过程纪律: 离线导入不属 TDD 子任务。无矛盾 依赖 bundled ffmpeg。Issue-0 需验证 bundling 配置。无矛盾
- A1: FR #10 + 补充需求「ffmpeg 做格式转换 → 16kHz mono WAV」。5 条全部对齐。无矛盾
- A2: ADR-003 未详细定义离线导入流程 (属 A3)。无矛盾

### Q14 — 双路混音具体参数

**决策**：5 条细化规则。

| 决策点 | 选择 | 理由 |
|---|---|---|
| RMS ducking 阈值 | 系统音频 RMS > -20dBFS → 麦克风衰减 6-12dB | 经验值。v1 用默认参数，后续可调 |
| 混音比例 | 等权重 0.5 + 0.5 混合 | RMS ducking 已保护麦克风，等权最自然 |
| clipping 防止 | tanh 软限幅，不硬截断 | 硬截断产生高频谐波，对转录质量有负面影响 |
| 48k→16k 重采样阶段 | 混音后重采样：48kHz stereo → downmix → tanh → 48kHz mono → 16kHz | 先混音再降采样，混音质量更高 |
| VAD 静音 gate | RMS > -40dBFS 才送 ChunkAccumulator | 低于此值标记静音，减少无效传输。对齐 fsmn-vad 内部静音检测 |

**回溯检查**：
- 过程纪律: 混音参数不属 TDD 子任务。现有 `pipeline.rs` 已有混音实现，本次参数调优。无矛盾 纯 Rust 计算。无环境依赖
- A1: FR #11「双路混音 + 48kHz 双路存档 + 16kHz 混音送转录」。5 条全部对齐。无矛盾
- A2: ADR-003 §7 + 现有 pipeline.rs RMS ducking。细化不改架构。无矛盾

### Q15 — 断路器 6 种错误类型 (差异化策略)

**决策**：错误类型差异化处理——进程级错误立即熔断，服务级错误容忍抖动。

| # | 错误类型 | 策略 | 理由 |
|---|---|---|---|
| 1 | ECONNREFUSED | 立即打开 | 进程已死，无需再试 |
| 2 | ECONNRESET | 立即打开 | 进程崩溃中 |
| 3 | ETIMEDOUT (> 5s) | 重试 1 次，仍失败则打开 | 瞬态网络可自愈 |
| 4 | HTTP 5xx | 连续 3 次后打开 | 服务活着但出错，容忍抖动 |
| 5 | 503 `{"status":"unhealthy"}` | 连续 3 次后打开 | 模型加载中，类似 5xx |
| 6 | 反序列化失败 | 立即打开 | HTTP 200 + JSON 损坏 = 半崩溃状态 |

**05a 状态转移表** (无 IO、无网络)：

```
关闭 → on_failure_instant → 打开 (类型 1/2/6)
关闭 → on_failure_accumulate 连续3次 → 打开 (类型 4/5)
关闭 → on_failure_retry → 重试1次 → 仍失败 → 打开 (类型 3)
打开 → timeout T 后 → 半开
半开 → on_success 连续2次 → 关闭
半开 → on_failure 1次 → 打开
成功 (任一时机) → 重置所有计数器
```

**回溯检查**：
- 过程纪律: 05a 测试覆盖 3 种失败事件路径 + 状态转移。~12 条测试。无组合爆炸。无矛盾 纯状态机。无矛盾
- A1: FR #6「断路器 + 渐进式健康探测」。差异化更贴合渐进探测意图。无矛盾
- A2: ADR-003 §9 三状态 + 探测间隔。细化不改框架。无矛盾

### Q16 — 摘要生成

**决策**：6 条细化规则。

| 决策点 | 选择 | 理由 |
|---|---|---|
| 输入内容 | 全量 segments。从 sidecar JSONL 拼接为带说话人标签的文本 | qwen3:8b 上下文 32K tokens。2h 会议 ~18K tokens，通常够用。超长 → v2 分段摘要 |
| prompt 结构 | 中文 system prompt (概述/决议/待办) + user 传 transcript。Prompt 放 FastAPI 侧 | 换 prompt 不需发新 Tauri 版本 |
| 存储位置 | FastAPI SQLite `meetings` 表新增 `summary` 列。API: `POST /meetings/{id}/summarize` | FunASR 不管摘要——这是 FastAPI + Ollama 的工作 |
| 触发方式 | 前端按钮 → Rust invoke → FastAPI → Ollama。生成中 disabled + loading | P3 手动触发，非自动 |
| Ollama 不在线 | FastAPI 检查 `GET localhost:11434` → 不通 → 返回中文错误 "Ollama 未运行" | 用户可读的错误，非系统级报错 |
| Ollama 未安装 | 前端「生成摘要」区域横幅 + 「安装 Ollama」按钮 → 打开浏览器 | Tauri 不替用户装系统软件 |
| qwen3:8b 未下载 | 前端「下载模型」按钮 → Tauri 调 `ollama pull qwen3:8b` → 解析 stdout 进度条 | Ollama CLI 自带进度输出 |
| 状态检测 | `GET /meetings/{id}/summary-status` → `{ollama: "missing"\|"no-model"\|"ready"}` | 一次调用拿全部状态 |
| 重复生成 | 覆盖旧摘要。每次用最新 segments 重新生成 | v1 简单，不做幂等/去重 |

**回溯检查**：
- 过程纪律: 摘要不属 FunASR 迁移 TDD 子任务。正确性由人工验证。无矛盾 依赖 Ollama + qwen3:8b。Issue-0 需验证。无矛盾
- A1: FR #7「Ollama qwen3:8b，用户手动触发」。6 条全部对齐。无矛盾
- A2: ADR-003 不覆盖 Ollama 细节 (属 FunASR 架构决策)。A2 定了三方服务拓扑 (:8178/:5167/:11434)。无矛盾

### Q17 — 自动标题生成

**决策**：会后自动调用 Ollama 生成 ≤ 10 字标题。

| 决策点 | 选择 | 理由 |
|---|---|---|
| 触发时机 | `/session/end` 返回后自动触发 | 会后自然时机，用户不用手动操作 |
| prompt | "请用 10 字以内概括以下会议内容" + 前 10 条 segments | 轻量 prompt，~1s。前 10 条足以判断主题 |
| 存储 | FastAPI `meetings.title` 列 (Issue 01 migration 已预留) | 标题是会议元信息，归 FastAPI |
| 失败处理 | 生成失败 → 保持日期时间标题，不重试 | 标题非关键功能，静默降级 |
| 端点 | `POST /meetings/{id}/generate-title` | 读 transcript_json 前 10 条 → Ollama → 写 title 列 |

**回溯检查**：
- 过程纪律: 非 TDD 子任务。正确性由人工验证。无矛盾
- A1: UX 增强，不与任何 FR 冲突。无矛盾
- A2: FastAPI 作为 meetings 数据归属，标题存 FastAPI 一致。无矛盾

### Q15 补充 — `/health` loading 进度态

**决策**：`GET /health` 三态响应 + `loading` 不触发断路器。

| 状态 | 响应 | 断路器行为 |
|---|---|---|
| `loading` | `{"status":"loading", "model":"SenseVoiceSmall", "progress":45.0}` | 不触发断路器。Rust 等待，前端展示进度条 |
| `ok` | `{"status":"ok"}` | 正常恢复信号 |
| `error` | `{"status":"error", "message":"..."}` | 计为 `on_failure`，触发断路器计数 |

**回溯检查**：
- 过程纪律: 非新增 TDD 子任务。无矛盾
- A1: FR #6 需要知道服务是否恢复中。loading 态是恢复路径的自然延伸。无矛盾
- A2: ADR-003 定了健康探测，loading 是探测响应的细化。无矛盾

### Q10 补充 — save-transcript 失败容灾

**决策**：FastAPI `/save-transcript` 失败时 segments 写入 sidecar，启动时自动重试。

| 决策点 | 选择 | 理由 |
|---|---|---|
| 检测时机 | Rust 调 `POST /save-transcript` 返回非 200 | ECONNREFUSED / ETIMEDOUT / HTTP 5xx 都触发 |
| 容灾策略 | segments 写入 sidecar (每行一个 segment，已含最终姓名) | sidecar 已在录音中持续写入。save 失败时额外保留这些 segments |
| 恢复触发 | Rust 启动时扫描 sidecar → 尝试 `POST /save-transcript` 重试 | 与 FunASR 崩溃恢复共用扫描机制 |
| 重试次数 | 最多 3 次，间隔 5s。全部失败 → 保留 sidecar，前端横幅提示 | 不无限重试，但数据不丢 |
| 成功后 | save 成功 → 删除 sidecar 文件 | 数据已安全落 DB |

**回溯检查**：
- 过程纪律: 非 TDD 子任务，属 C1 集成接线。无矛盾
- A1: FR #6「崩溃容灾」。save-transcript 失败是容灾子场景。无矛盾
- A2: ADR-003 §7「sidecar 是 segments 恢复的唯一载体」。save 失败也走 sidecar，一致。无矛盾


