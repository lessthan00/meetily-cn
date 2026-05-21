# TDD 02b — WAV header 构造（16kHz mono PCM）

**父 Issue**: [#02 FunASR 转录链路](../issues/02-funasr-transcription-pipeline.md)

## 测试范围

Rust 端将原始 PCM bytes 封装为有效 WAV 格式，FunASR 能正确解析。

## RED — 先写失败测试

```rust
#[test]
fn wav_header_magic_bytes() {
    let pcm = vec![0i16; 16000];  // 1 秒 16kHz mono
    let wav = construct_wav(&pcm, 16000, 1);
    // RIFF header
    assert_eq!(&wav[0..4], b"RIFF");
    assert_eq!(&wav[8..12], b"WAVE");
}


#[test]
fn wav_header_sample_rate() {
    let wav = construct_wav(&vec![0i16; 8000], 16000, 1);
    // fmt chunk: sample rate at offset 24 (4 bytes, little-endian)
    let sr = u32::from_le_bytes([wav[24], wav[25], wav[26], wav[27]]);
    assert_eq!(sr, 16000);
}


#[test]
fn wav_header_num_channels() {
    let wav = construct_wav(&vec![0i16; 8000], 16000, 1);
    let ch = u16::from_le_bytes([wav[22], wav[23]]);
    assert_eq!(ch, 1);  // mono
}


#[test]
fn wav_data_size_matches_input() {
    let pcm = vec![0i16; 32000];  // 2 秒
    let wav = construct_wav(&pcm, 16000, 1);
    let data_size = u32::from_le_bytes([wav[40], wav[41], wav[42], wav[43]]);
    assert_eq!(data_size as usize, pcm.len() * 2);  // i16 = 2 bytes
}


#[test]
fn wav_total_file_size() {
    let pcm = vec![0i16; 16000];
    let wav = construct_wav(&pcm, 16000, 1);
    // RIFF size = file_size - 8
    let riff_size = u32::from_le_bytes([wav[4], wav[5], wav[6], wav[7]]);
    assert_eq!(riff_size as usize, wav.len() - 8);
}


#[test]
fn empty_pcm_produces_valid_wav() {
    // 静音 chunk — VAD 判为静音，但 WAV 仍应合法
    let wav = construct_wav(&[], 16000, 1);
    assert_eq!(&wav[0..4], b"RIFF");
    // data size = 0
    let data_size = u32::from_le_bytes([wav[40], wav[41], wav[42], wav[43]]);
    assert_eq!(data_size, 0);
}


#[test]
fn python_can_parse_output() {
    // 集成验证：用 Python wave 模块读取 Rust 生成的 WAV
    let wav = construct_wav(&vec![100i16, 200i16, 300i16], 16000, 1);
    // 写入临时文件 → subprocess python → 验证 sr/channels/samples
    // (此测试需要 Python 环境，标记为 #[cfg(feature = "integration")])
}
```

## GREEN — 最小实现

在 `funasr_client.rs` 或独立 `wav.rs` 中实现：

```rust
fn construct_wav(pcm: &[i16], sample_rate: u32, channels: u16) -> Vec<u8>
```

- 写 RIFF header（4 bytes）
- 写文件大小（4 bytes LE）
- 写 WAVE + fmt chunk（24 bytes）
- 写 data chunk header（8 bytes）+ PCM data
- 注意字节序（little-endian）

## REFACTOR

- 考虑用 `byteorder` crate 消除手动 LE 字节拼接
- 如果后续需要 stereo（系统音频 + 麦克风分离），预留 channels 参数

## 预计耗时

< 1 小时（WAV header 是固定 44 bytes 格式）
