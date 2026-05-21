# Tauri Events 契约 (Rust → React)

SidebarProvider (React Context) 骨架不变，只新增 event handler。

---

## transcript-update (原有，payload 变更)

**emit**: Rust funasr_transcription_task 收到 FunASR 响应后。
**listen**: 转录面板组件。

**Payload**:

```typescript
interface TranscriptUpdate {
  text: string;           // 所有 segment 拼接 (冗余，方便简单渲染)
  segments: Segment[];    // 按 segment 逐条渲染
}

interface Segment {
  start: number;          // 秒
  end: number;
  text: string;
  speaker: string | null; // 已注册=姓名，未注册=聚类标签，声纹未接=null
}
```

**与旧版差异**：旧版 payload 可能只含 `text` 或单一片段。新版必须是 segments 数组。

---

## session-ended (新增)

**emit**: Rust 在 /session/end 返回后 (segments 已写入 sidecar)。
**listen**: 命名弹窗组件。

**Payload**:

```typescript
interface SessionEnded {
  meeting_id: string;
  speakers: SpeakerSummary[];
}

interface SpeakerSummary {
  label: string;           // "张甜甜" / "说话人 A"
  segment_count: number;
  total_duration: number;  // 秒
}
```

**界面行为**：前端弹出未注册说话人命名 UI。已注册说话人 (真实姓名) 不在列表中。用户输入映射 {说话人 A → "李四"} 或跳过。

---

## speakers-renamed (新增)

**emit**: Rust apply_speaker_names 完成后。
**listen**: 转录面板组件 (更新 speaker 渲染)。

**Payload**:

```typescript
interface SpeakersRenamed {
  meeting_id: string;
  segments: Segment[];  // 更新后的全量 segments (speaker 字段已替换为真实姓名)
}
```

**界面行为**：前端重新渲染转录面板，临时标签替换为真实姓名。

---

## recovery-banner (新增)

**emit**: Rust 启动时扫描 sidecar 发现 `status: "partial"` / `raw_segments` 非 null / `named_segments` 非 null。
**listen**: 全局横幅组件。

**Payload**:

```typescript
interface RecoveryBanner {
  meeting_id: string;
  status: "partial" | "pending_naming" | "pending_save";
}
```

**界面行为**：
- `partial` → 显示 "转录服务中断，音频已保存" + "重新处理"按钮
- `pending_naming` → 重新弹出命名 UI
- `pending_save` → 自动重试 save-transcript，成功后横幅消失

---

## FunASR 健康状态 (前端直连，不经过 Tauri event)

前端 2s 轮询 `GET :8178/health`：

| 响应 status | 界面 |
|---|---|
| `ok` | 无横幅 |
| `loading` | 显示进度条 "正在下载模型 {model} {progress}%" |
| `error` | 显示错误横幅 + "启动引擎"按钮 (调 start-funasr.sh) |

---

## 前端 ↔ Rust 的读写路径

| 操作 | 路径 | 原因 |
|---|---|---|
| GET /health, GET /speakers, GET|POST /config | 前端直连 FunASR | 只读/纯配置，无副作用 |
| POST /enroll, DELETE /speakers/{id}, export/import | 前端 → Rust → FunASR | 有副作用，需 Rust 协调 |
| 转录链路 | Rust → FunASR | 音频来源在 Rust |
| meetings CRUD | 前端 → FastAPI | 会议数据归属 FastAPI |
