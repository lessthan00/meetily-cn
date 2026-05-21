# TDD 06b — 摘要 prompt 构造

**父 Issue**: [#06 摘要服务验证 + Markdown 导出对齐](../issues/06-summary-service-and-export.md)

## 测试范围

将 segments（含说话人标注）格式化为 Ollama 摘要 prompt。确保说话人信息被正确传递。

## RED — 先写失败测试

```python
# test_summary_prompt.py
import pytest


def test_prompt_includes_all_speakers():
    """prompt 中出现每个说话人的标注"""
    segments = [
        {"start": 0.0, "end": 2.0, "text": "我觉得可行", "speaker": "张甜甜"},
        {"start": 2.5, "end": 4.0, "text": "我也同意", "speaker": "李四"},
    ]
    prompt = build_summary_prompt(segments)
    assert "张甜甜" in prompt
    assert "李四" in prompt


def test_prompt_includes_transcript_text():
    """prompt 包含完整转录文本"""
    segments = [
        {"start": 0.0, "end": 2.0, "text": "方案A更好", "speaker": "张甜甜"},
    ]
    prompt = build_summary_prompt(segments)
    assert "方案A更好" in prompt


def test_prompt_has_chinese_instruction():
    """提示词为中文，要求 Ollama 用中文摘要"""
    segments = [{"start": 0.0, "end": 1.0, "text": "test", "speaker": "A"}]
    prompt = build_summary_prompt(segments)
    assert any(word in prompt for word in ["摘要", "纪要", "会议", "总结"])


def test_empty_segments_safe():
    """空转录 → prompt 仍有基本结构（不崩）"""
    prompt = build_summary_prompt([])
    assert len(prompt) > 0
    assert "会议" in prompt or "转录" in prompt


def test_prompt_not_too_long():
    """超长会议 → prompt 截断到 Ollama 上下文窗口内"""
    segments = [
        {"start": float(i), "end": float(i+1), "text": "测试" * 500, "speaker": "说话人 A"}
        for i in range(100)
    ]
    prompt = build_summary_prompt(segments)
    # qwen3:8b 上下文约 32K tokens，prompt 应控制在 ~8000 chars 以内
    assert len(prompt) < 16000  # 约 8000 Chinese chars


def test_unknown_speaker_labeled():
    """未注册说话人 → prompt 中标注为「说话人 X」"""
    segments = [
        {"start": 0.0, "end": 1.0, "text": "hello", "speaker": "说话人 A"},
    ]
    prompt = build_summary_prompt(segments)
    assert "说话人 A" in prompt


def test_mixed_known_unknown_speakers():
    """混合场景 → 已知姓名和聚类标签都出现"""
    segments = [
        {"start": 0.0, "end": 1.0, "text": "A", "speaker": "张甜甜"},
        {"start": 1.0, "end": 2.0, "text": "B", "speaker": "说话人 A"},
    ]
    prompt = build_summary_prompt(segments)
    assert "张甜甜" in prompt
    assert "说话人 A" in prompt
```

## GREEN — 最小实现

```python
def build_summary_prompt(segments: list[dict]) -> str:
    transcript_lines = []
    for seg in segments:
        speaker = seg.get("speaker") or "未知说话人"
        ts = format_timestamp(seg["start"])
        transcript_lines.append(f"[{ts}] {speaker}: {seg['text']}")

    transcript_text = "\n".join(transcript_lines)

    # 控制长度：超出时截断
    max_chars = 8000
    if len(transcript_text) > max_chars:
        transcript_text = transcript_text[:max_chars] + "\n[...转录内容因长度截断...]"

    prompt = f"""你是一个专业的会议纪要助手。请根据以下会议转录内容，生成简洁的会议纪要。

要求：
- 用中文输出
- 包含：会议主题、关键讨论点、决策事项、待办事项
- 用 Markdown 格式
- 保留说话人标注以便追踪谁说了什么

## 会议转录

{transcript_text}

## 会议纪要
"""
    return prompt
```

## REFACTOR

- 提取 prompt 模板到独立文件（如 `prompts/summary.txt`），方便非程序员调整
- 考虑摘要质量评估测试（用 mock Ollama 响应验证 prompt 效果）

## 预计耗时

< 45 分钟
