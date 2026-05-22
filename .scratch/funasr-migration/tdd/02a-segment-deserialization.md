# TDD 02a — segments JSON 反序列化 + 边界情况

**父 Issue**: [#02 FunASR 转录链路](../issues/02-funasr-transcription-pipeline.md)

## 测试范围

测试 Rust 端 `TranscriptionResult` 的 JSON 反序列化，覆盖正常、异常、边界输入。

## RED — 先写失败测试

```rust
// funasr_client.rs tests

#[test]
fn deserialize_normal_response() {
    let json = r#"{"text": "你好世界", "segments": [
        {"start": 0.0, "end": 1.5, "text": "你好世界", "speaker": null}
    ]}"#;
    let result: TranscriptionResult = serde_json::from_str(json).unwrap();
    assert_eq!(result.text, "你好世界");
    assert_eq!(result.segments.len(), 1);
    assert_eq!(result.segments[0].start, 0.0);
    assert_eq!(result.segments[0].end, 1.5);
    assert_eq!(result.segments[0].speaker, None);  // null → None
}


#[test]
fn deserialize_multiple_segments() {
    let json = r#"{"text": "AB", "segments": [
        {"start": 0.0, "end": 1.0, "text": "A", "speaker": null},
        {"start": 1.2, "end": 2.0, "text": "B", "speaker": "说话人 A"}
    ]}"#;
    let result: TranscriptionResult = serde_json::from_str(json).unwrap();
    assert_eq!(result.segments.len(), 2);
}


#[test]
fn deserialize_empty_segments() {
    let json = r#"{"text": "", "segments": []}"#;
    let result: TranscriptionResult = serde_json::from_str(json).unwrap();
    assert!(result.segments.is_empty());
}


#[test]
fn deserialize_segment_with_speaker() {
    let json = r#"{"text": "你好", "segments": [
        {"start": 0.0, "end": 0.8, "text": "你好", "speaker": "张甜甜"}
    ]}"#;
    let result: TranscriptionResult = serde_json::from_str(json).unwrap();
    assert_eq!(result.segments[0].speaker, Some("张甜甜".to_string()));
}


#[test]
fn deserialize_segment_speaker_null() {
    // CAM++ 无法识别 → speaker=null
    let json = r#"{"text": "...", "segments": [
        {"start": 0.0, "end": 0.5, "text": "...", "speaker": null}
    ]}"#;
    let result: TranscriptionResult = serde_json::from_str(json).unwrap();
    assert_eq!(result.segments[0].speaker, None);
}


#[test]
fn deserialize_missing_segments_field() {
    // 旧版 API 没有 segments → 应失败或给默认值
    let json = r#"{"text": "hello"}"#;
    let result: Result<TranscriptionResult, _> = serde_json::from_str(json);
    assert!(result.is_err());  // segments 为必需字段
}


#[test]
fn reject_out_of_order_timestamps() {
    // end < start → 反序列化后校验拒绝
    let json = r#"{"text": "", "segments": [
        {"start": 1.0, "end": 0.5, "text": "倒序", "speaker": null}
    ]}"#;
    let result: TranscriptionResult = serde_json::from_str(json).unwrap();
    assert!(!result.is_valid());  // 自定义校验方法
}
```

## GREEN — 最小实现

- `TranscriptionResult` 已有骨架，补充 `#[serde(default)]` 策略确认
- 新增 `TranscriptionResult::is_valid()` — 校验 segments 时间戳单调递增、end ≥ start
- `Segment.speaker: Option<String>` — `null` → `None`、`"说话人 A"` → `Some(...)`、`"张甜甜"` → `Some(...)`。三态语义由 caller 解释

## REFACTOR

- 考虑用 `#[serde(deny_unknown_fields)]` 防止未预期字段静默通过

## 预计耗时

< 45 分钟（纯 serde + 校验方法）
