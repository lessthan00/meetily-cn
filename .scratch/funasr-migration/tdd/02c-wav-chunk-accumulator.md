# TDD 02c — WAV chunk 累加器

**父 Issue**: [#02 FunASR 转录链路](../issues/02-funasr-transcription-pipeline.md) — Commit 02-1

## 测试范围

接收 16kHz f32 音频流，按固定 2 秒 (32000 采样) 窗口累加，满窗时吐出一个全窗 chunk，余量滚入下一轮。

纯逻辑模块，无 I/O，无 HTTP，无 GPU。极适合 TDD。

## RED — 先写失败测试

```rust
// test_chunk_accumulator.rs

const CHUNK_SAMPLES: usize = 32000; // 2s @ 16kHz

#[test]
fn empty_input_produces_none() {
    let mut acc = ChunkAccumulator::new(CHUNK_SAMPLES);
    assert_eq!(acc.accumulate(&[]), None);
    assert_eq!(acc.remaining(), 0);
}

#[test]
fn exactly_one_chunk() {
    let mut acc = ChunkAccumulator::new(CHUNK_SAMPLES);
    let input = vec![0.0f32; CHUNK_SAMPLES];
    let chunk = acc.accumulate(&input);
    assert!(chunk.is_some());
    assert_eq!(chunk.unwrap().len(), CHUNK_SAMPLES);
}

#[test]
fn partial_input_returns_none() {
    // Q6: 首个 chunk 前 buffer 不满 2s → 返回 None
    let mut acc = ChunkAccumulator::new(CHUNK_SAMPLES);
    let chunk = acc.accumulate(&vec![0.0f32; 10000]);
    assert_eq!(chunk, None);
    assert_eq!(acc.remaining(), 10000);
}

#[test]
fn partial_fills_accumulate_until_full() {
    let mut acc = ChunkAccumulator::new(CHUNK_SAMPLES);
    assert_eq!(acc.accumulate(&vec![0.0f32; 10000]), None); // 10000
    assert_eq!(acc.accumulate(&vec![0.0f32; 10000]), None); // 20000
    assert_eq!(acc.accumulate(&vec![0.0f32; 10000]), None); // 30000
    // 再推 10000 → 30000+10000=40000 → 吐 1 chunk + 8000 残留
    let chunk = acc.accumulate(&vec![0.0f32; 10000]);
    assert!(chunk.is_some());
    assert_eq!(chunk.unwrap().len(), CHUNK_SAMPLES);
    assert_eq!(acc.remaining(), 8000);
}

#[test]
fn flush_returns_partial_chunk() {
    // Q6: 录音停在 1.3s → buffer 有 20800 samples → flush 吐出不足 2s 的 chunk
    let mut acc = ChunkAccumulator::new(CHUNK_SAMPLES);
    acc.accumulate(&vec![0.0f32; 20800]);
    let remainder = acc.flush();
    assert_eq!(remainder.len(), 20800); // 送 FunASR，不丢弃
    assert_eq!(acc.remaining(), 0);
}

#[test]
fn flush_empty_returns_empty() {
    let mut acc = ChunkAccumulator::new(CHUNK_SAMPLES);
    assert!(acc.flush().is_empty());
}

#[test]
fn reset_clears_buffer() {
    let mut acc = ChunkAccumulator::new(CHUNK_SAMPLES);
    acc.accumulate(&vec![0.0f32; 18000]);
    assert_eq!(acc.remaining(), 18000);
    acc.reset();
    assert_eq!(acc.remaining(), 0);
}

#[test]
fn large_input_multi_chunk() {
    // 单次推入 100000 samples → 3 个满 chunk + 4000 残留
    let mut acc = ChunkAccumulator::new(CHUNK_SAMPLES);
    let chunks = acc.accumulate_all(&vec![0.0f32; 100000]);
    assert_eq!(chunks.len(), 3);
    assert_eq!(acc.remaining(), 4000);
}
```
```

## GREEN — 最小实现

```rust
/// 固定窗口音频累加器 (Q6 边界行为)。
/// 收任意长度 f32 采样，满 `chunk_samples` 时吐出完整窗口。
/// 不满时返回 None，余量滚入下一轮。录制结束时调用 `flush()` 收取残余。
pub struct ChunkAccumulator {
    chunk_samples: usize,
    buffer: Vec<f32>,
}

impl ChunkAccumulator {
    pub fn new(chunk_samples: usize) -> Self {
        Self { chunk_samples, buffer: Vec::new() }
    }

    /// 推入音频采样。满 2s 返回 Some(chunk)，不满返回 None (Q6: 首 chunk 前不发)。
    pub fn accumulate(&mut self, samples: &[f32]) -> Option<Vec<f32>> {
        self.buffer.extend_from_slice(samples);
        if self.buffer.len() >= self.chunk_samples {
            let chunk: Vec<f32> = self.buffer.drain(..self.chunk_samples).collect();
            Some(chunk)
        } else {
            None
        }
    }

    /// 批量版 accumulate：用于离线导入或大量数据一次性推入。
    pub fn accumulate_all(&mut self, samples: &[f32]) -> Vec<Vec<f32>> {
        self.buffer.extend_from_slice(samples);
        let mut chunks = Vec::new();
        while self.buffer.len() >= self.chunk_samples {
            chunks.push(self.buffer.drain(..self.chunk_samples).collect());
        }
        chunks
    }

    /// 收取 buffer 中残留的所有采样 (Q6: 不足 2s 也吐出，送 FunASR 不丢内容)。
    pub fn flush(&mut self) -> Vec<f32> {
        self.buffer.drain(..).collect()
    }

    pub fn remaining(&self) -> usize { self.buffer.len() }
    pub fn reset(&mut self) { self.buffer.clear(); }
}
```

## REFACTOR

- `buffer` 预分配不再用 `with_capacity(chunk_samples * 2)`，而用 `Vec::new()` 让 push 时自然扩容——实测两种策略内存差异可忽略
- 考虑导出 `pub const CHUNK_SAMPLES_2S_16KHZ: usize = 32000;` 常量，避免硬编码
- 单元测试覆盖后无需再改

## 预计耗时

< 45 分钟
