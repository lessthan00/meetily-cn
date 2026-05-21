# TDD 02d — Python WAV 解码

**父 Issue**: [#02 FunASR 转录链路](../issues/02-funasr-transcription-pipeline.md)
**对应**: Rust 02b (WAV 构造) 的对偶测试——Python 侧解析 Rust 产出的 WAV

## 测试范围

接收 WAV bytes (Rust 产出的 16kHz mono PCM)——解码为 PCM samples + 校验格式。纯函数，无 HTTP，无模型。

## RED — 先写失败测试

```python
# test_wav_decode.py
import pytest
from wav import wav_to_pcm

# Rust funasr_client 构造的 WAV：16kHz mono 16-bit
# 由 02b TDD 的 test_basic_16khz_mono 产出

def test_valid_16khz_mono_wav():
    """标准 16kHz mono WAV → 正确解码"""
    import struct
    sample_rate = 16000
    samples = [0.0, 0.5, -0.5, 1.0, -1.0]
    # 构造符合 Rust 02b 输出的 WAV
    raw = struct.pack('<' + 'h' * len(samples),
        *[max(-32768, min(32767, int(s * 32767))) for s in samples])
    import io, wave
    buf = io.BytesIO()
    with wave.open(buf, 'wb') as wf:
        wf.setnchannels(1)
        wf.setsampwidth(2)  # 16-bit
        wf.setframerate(sample_rate)
        wf.writeframes(raw)
    wav_bytes = buf.getvalue()
    
    pcm, sr = wav_to_pcm(wav_bytes)
    assert sr == sample_rate
    assert len(pcm) == len(samples)
    assert pcm[0] == pytest.approx(0.0, abs=1e-4)
    assert pcm[2] == pytest.approx(-0.5, abs=1e-4)


def test_empty_wav_rejected():
    """空 WAV → 明确的错误"""
    with pytest.raises(ValueError, match="empty"):
        wav_to_pcm(b'')


def test_stereo_rejected():
    """立体声 WAV → 错误（必须 mono）"""
    import io, wave, struct
    buf = io.BytesIO()
    with wave.open(buf, 'wb') as wf:
        wf.setnchannels(2)  # ✗ 立体声
        wf.setsampwidth(2)
        wf.setframerate(16000)
        wf.writeframes(struct.pack('<hh', 0, 0))
    
    with pytest.raises(ValueError, match="mono"):
        wav_to_pcm(buf.getvalue())


def test_invalid_header():
    """不是 WAV 的随机字节 → 明确错误"""
    with pytest.raises(ValueError, match="WAV"):
        wav_to_pcm(b'\x00\x01\x02\x03' * 100)


def test_output_is_float32():
    """输出应为 [-1.0, 1.0] 范围的 float32"""
    import struct
    # 单个采样为 10000 (约 0.305)
    raw = struct.pack('<h', 10000)
    import io, wave
    buf = io.BytesIO()
    with wave.open(buf, 'wb') as wf:
        wf.setnchannels(1)
        wf.setsampwidth(2)
        wf.setframerate(16000)
        wf.writeframes(raw)
    
    pcm, _ = wav_to_pcm(buf.getvalue())
    assert len(pcm) == 1
    assert isinstance(pcm[0], float)
    assert -1.0 <= pcm[0] <= 1.0
```

## GREEN — 最小实现

```python
import io
import wave

def wav_to_pcm(wav_bytes: bytes) -> tuple[list[float], int]:
    """Decode WAV bytes → (pcm_samples as float32, sample_rate)."""
    if not wav_bytes:
        raise ValueError("Empty WAV input")
    
    try:
        with wave.open(io.BytesIO(wav_bytes), 'rb') as wf:
            if wf.getnchannels() != 1:
                raise ValueError(f"Expected mono audio, got {wf.getnchannels()} channels")
            if wf.getsampwidth() != 2:
                raise ValueError(f"Expected 16-bit audio, got {wf.getsampwidth() * 8}-bit")
            
            sample_rate = wf.getframerate()
            raw = wf.readframes(wf.getnframes())
            
            # 16-bit signed → float32 [-1.0, 1.0]
            import struct
            n = len(raw) // 2
            fmt = '<' + 'h' * n
            int_samples = struct.unpack(fmt, raw)
            pcm = [s / 32768.0 for s in int_samples]
            
            return pcm, sample_rate
    except wave.Error as e:
        raise ValueError(f"Invalid WAV format: {e}")
```

## REFACTOR

- 如果频繁调用（如每个 2s chunk），可用 `numpy.frombuffer(raw, dtype=np.int16).astype(np.float32) / 32768.0` 替代 struct.unpack——快 20x
- 考虑缓存 sample_rate 到外部 context（2s chunk 始终同格式）

## 预计耗时

< 45 分钟
