# TDD 03c — 音频时长/质量校验

**父 Issue**: [#03 声纹注册 + 实时说话人标注](../issues/03-voiceprint-enrollment-and-identification.md)

## 测试范围

声纹注册时，对上传音频做基本校验：时长、音量/信噪比、格式。

## RED — 先写失败测试

```python
# test_audio_validation.py
import pytest


def test_too_short_rejected():
    """< 3 秒 → 拒绝"""
    # 16kHz mono, 2 秒 = 32000 samples
    wav = make_fake_wav(duration_sec=2.0)
    result = validate_enrollment_audio(wav)
    assert result.valid is False
    assert "太短" in result.error


def test_minimum_duration_accepted():
    """≥ 3 秒 → 通过"""
    wav = make_fake_wav(duration_sec=3.0)
    result = validate_enrollment_audio(wav)
    assert result.valid is True


def test_silent_audio_rejected():
    """全是静音（振幅 < 阈值）→ 拒绝"""
    wav = make_fake_wav(duration_sec=5.0, amplitude=0.0)
    result = validate_enrollment_audio(wav)
    assert result.valid is False
    assert "无有效语音" in result.error or "静音" in result.error


def test_low_snr_rejected():
    """噪音太多 → 拒绝"""
    wav = make_fake_wav(duration_sec=5.0, signal_ratio=0.1)  # 90% 噪音
    result = validate_enrollment_audio(wav)
    assert result.valid is False
    assert "噪音" in result.error or "质量" in result.error


def test_normal_audio_accepted():
    """正常语音 → 通过"""
    wav = make_fake_wav(duration_sec=5.0, signal_ratio=0.9)
    result = validate_enrollment_audio(wav)
    assert result.valid is True


def test_non_wav_format_rejected():
    """非 WAV 格式 → 报错"""
    result = validate_enrollment_audio(b"not a wav file")
    assert result.valid is False
    assert "格式" in result.error


def test_wrong_sample_rate_rejected():
    """非 16kHz → 拒绝或自动重采样（取决于设计）"""
    wav = make_fake_wav(duration_sec=5.0, sample_rate=44100)
    result = validate_enrollment_audio(wav)
    assert result.valid is False or result.needs_resample


def test_stereo_accepted_or_downmixed():
    """立体声 → 接受但下混为 mono"""
    wav = make_fake_wav(duration_sec=5.0, channels=2)
    result = validate_enrollment_audio(wav)
    # 至少不应崩溃
    assert result.valid or result.needs_downmix
```

## GREEN — 最小实现

```python
@dataclass
class AudioValidationResult:
    valid: bool
    error: Optional[str] = None
    needs_resample: bool = False
    needs_downmix: bool = False


def validate_enrollment_audio(wav_bytes: bytes) -> AudioValidationResult:
    # 1. WAV header 解析
    # 2. duration = data_size / (sample_rate * channels * 2)
    # 3. RMS amplitude 检测
    # 4. 简单的信噪比估计（信号 RMS / 最低 5% 段的平均 RMS）
```

> **简化策略**: v1 不做完整 SNR 计算，用 RMS ≥ 阈值的比率（`active_ratio`）近似。> 80% 的样本接近零 = 静音。

## REFACTOR

- 考虑用 `librosa` 或 `soundfile` 做解析，替代手动 WAV header 解析
- SNR 估算日后可换为更准确的方法

## 预计耗时

< 1 小时
