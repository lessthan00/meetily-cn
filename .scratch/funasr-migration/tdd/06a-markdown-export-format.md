# TDD 06a — Markdown 导出格式

**父 Issue**: [#06 摘要服务验证 + Markdown 导出对齐](../issues/06-summary-service-and-export.md)

## 测试范围

给定 segments 数组（含说话人标注），生成正确格式的 Markdown 导出文件。

## RED — 先写失败测试

```python
# test_markdown_export.py
import pytest


def test_transcript_only_export():
    """仅转录（无摘要）→ 生成 transcript.md"""
    segments = [
        {"start": 0.0, "end": 2.5, "text": "我觉得这个方案可行", "speaker": "张甜甜"},
        {"start": 3.0, "end": 5.0, "text": "但是有一些问题", "speaker": "说话人 A"},
    ]
    md = export_transcript_markdown(segments, title="测试会议")
    assert "**张甜甜** (00:00:00): 我觉得这个方案可行" in md
    assert "**说话人 A** (00:00:03): 但是有一些问题" in md


def test_timestamp_formatting():
    """时间戳格式 HH:MM:SS"""
    segments = [
        {"start": 0.0, "end": 2.5, "text": "hello", "speaker": None},
        {"start": 3661.0, "end": 3663.0, "text": "hello2", "speaker": None},
    ]
    md = export_transcript_markdown(segments)
    lines = md.split("\n")
    assert "(00:00:00)" in lines[0]
    assert "(01:01:01)" in lines[1]  # 3661s = 1h1m1s


def test_unknown_speaker_rendered_as_unknown():
    """speaker=None → 「未知」"""
    segments = [{"start": 0.0, "end": 1.0, "text": "test", "speaker": None}]
    md = export_transcript_markdown(segments)
    assert "**未知**" in md or "**Unknown**" in md


def test_full_export_includes_summary():
    """完整导出 → 转录 + AI 摘要"""
    segments = [{"start": 0.0, "end": 1.0, "text": "hi", "speaker": "张甜甜"}]
    summary = "## 要点\n- 讨论方案可行性"
    md = export_full_markdown(segments, summary, title="测试")
    assert "# 会议记录 — 测试" in md
    assert "## AI 摘要" in md
    assert summary in md
    assert "## 完整转录" in md
    assert "**张甜甜**" in md


def test_empty_segments():
    """空 segments → 仍然生成有效 Markdown"""
    md = export_transcript_markdown([], title="空会议")
    assert "## 完整转录" in md
    assert len(md.split("\n")) < 10  # 只有标题无内容


def test_special_characters_not_break_markdown():
    """说话人名字含特殊字符 → 不破坏 Markdown 格式"""
    segments = [{"start": 0.0, "end": 1.0, "text": "test", "speaker": "张*甜[甜]"}]
    md = export_transcript_markdown(segments)
    assert "**张*甜[甜]**" in md or "**张\\*甜\\[甜\\]**" in md


def test_consecutive_same_speaker_not_merged():
    """同一说话人连续两段 → 各一行，不合并"""
    segments = [
        {"start": 0.0, "end": 1.0, "text": "第一句", "speaker": "张甜甜"},
        {"start": 1.5, "end": 2.5, "text": "第二句", "speaker": "张甜甜"},
    ]
    md = export_transcript_markdown(segments)
    count = md.count("**张甜甜**")
    assert count == 2  # 两行都出现
```

## GREEN — 最小实现

```python
def _format_timestamp(seconds: float) -> str:
    h = int(seconds // 3600)
    m = int((seconds % 3600) // 60)
    s = int(seconds % 60)
    return f"{h:02d}:{m:02d}:{s:02d}"

def export_transcript_markdown(segments, title="会议记录"):
    lines = [f"# {title}\n", "## 完整转录\n"]
    for seg in segments:
        ts = _format_timestamp(seg["start"])
        speaker = seg.get("speaker") or "未知"
        lines.append(f"**{speaker}** ({ts}): {seg['text']}")
    return "\n".join(lines) + "\n"

def export_full_markdown(segments, summary, title="会议记录"):
    transcript = export_transcript_markdown(segments, title)
    return f"{transcript}\n## AI 摘要\n\n{summary}\n"
```

## REFACTOR

- 考虑用 Jinja2 模板替代字符串拼接（当格式变复杂时）
- 提取公共常量 `UNKNOWN_SPEAKER = "未知"`

## 预计耗时

< 45 分钟
