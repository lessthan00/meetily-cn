# Meetily

隐私优先的 AI 会议助手。捕获、转录、摘要会议内容，全部在本地基础设施上运行。

## Language

### 核心概念

**Meeting (会议)**:
一次完整的录音会话，包含转录文本、说话人标注、AI 摘要。从录音开始到会后命名完成为其完整生命周期。

**Transcript (转录)**:
会议中所有语音的文本记录。由 segments 数组组成，每个 segment 带时间范围和说话人。

**Segment (语音段)**:
转录的最小单元。包含 `start`、`end`、`text`、`speaker` 四个字段。

**Summary (摘要)**:
由 LLM 根据完整转录生成的会议纪要，Markdown 格式。用户手动触发，非自动生成。

### 声纹与说话人

**Voiceprint (声纹)**:
CAM++ 模型从语音中提取的 256 维 float32 向量，用于唯一标识说话人。
_Avoid_: embedding, feature vector

**Voiceprint Enrollment (声纹注册)**:
用户朗读固定文本、录制音频、系统提取声纹并存入声纹库的过程。注册后该说话人在任何会议中均可被识别。

**Speaker (说话人)**:
会议中的发言者。分两类：
- **Registered Speaker (已注册说话人)**: 有声纹在库中，会议中自动标注姓名
- **Unregistered Speaker (未注册说话人)**: 无存储声纹，会议中标注为临时聚类标签

**Speaker Label (说话人标签)**:
显示在转录旁的说话人标识。已注册者为真实姓名（如"张甜甜"），未注册者为聚类标签（如"说话人 A"）。
_Avoid_: speaker name, speaker ID

**Speaker Clustering (说话人聚类)**:
对未注册说话人按声纹相似度在线分组，使用 k-means 算法。k 值由用户在会议开始前输入（默认 4）。

### 会话

**Session (会话)**:
一次会议的说话人聚类上下文。会议开始时创建（`/session/start`），初始化 k-means 聚类状态；会议结束时销毁（`/session/end`），返回聚类摘要后释放内存。

**Post-meeting Naming (会后命名)**:
会议结束后，用户将临时聚类标签（如"说话人 A"）映射为真实姓名（如"李四"）的过程。命名完成后 segments 才写入数据库。

**Offline Audio Import (离线音频导入)**:
用户手动导入中文音频文件，系统离线处理为完整会议记录（含转录 + 说话人分离 + 摘要）。复用 `/process-audio?mode=offline` 接口。

**Crash Recovery (崩溃容灾)**:
FunASR 服务中途崩溃时，断路器停止发送音频 chunk，保存完整 WAV 录音到磁盘，健康探测恢复后离线重处理未转录部分。

## Relationships

- 一个 **Meeting** 拥有一个 **Transcript** 和一个可选 **Summary**
- 一个 **Transcript** 包含多个 **Segment**
- 每个 **Segment** 属于一个 **Speaker**
- 一个 **Speaker** 最多有一个 **Voiceprint**
- 一个 **Meeting** 拥有一个 **Session**（1:1，录音期间）
- 一次 **Voiceprint Enrollment** 创建一个 **Speaker**

## Example dialogue

> **Dev**: "如果一个未注册说话人在会议中间发言，Segment 的 speaker 字段填什么？"
> **Domain expert**: "填聚类标签，如「说话人 A」。如果这个说话人与所有现有质心的相似度都低于 0.5，会触发自适应扩容，新建一个聚类。"
>
> **Dev**: "会后命名时，已注册说话人需要出现在命名列表中吗？"
> **Domain expert**: "不需要。已注册说话人在会议中已经标注了真实姓名，只有临时标签才需要命名。"
>
> **Dev**: "如果用户跳过命名直接关掉弹窗，数据库里存什么？"
> **Domain expert**: "存的就是临时标签「说话人 A」。用户之后可以从会议详情页点击标签重新命名。"

## Flagged ambiguities

- 旧代码中"transcription"既指 Whisper 推理过程也指转录结果文本——FunASR 迁移后转录引擎为单一 FunASR Python 服务。
