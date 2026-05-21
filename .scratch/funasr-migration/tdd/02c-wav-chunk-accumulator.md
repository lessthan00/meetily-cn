# TDD 02c — WAV chunk 累加器

**父 Issue**: [#02 FunASR 转录链路](../issues/02-funasr-transcription-pipeline.md) — Commit 02-1

## 测试范围

接收 16kHz f32 音频流，按固定 2 秒 (32000 采样) 窗口累加，满窗时吐出一个全窗 chunk，余量滚入下一轮。

纯逻辑模块，无 I/O，无 HTTP，无 GPU。极适合 TDD。

## RED — 先写失败测试

```rust
// test_chunk_accumulator.rs

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn empty_input_produces_no_output() {
        let mut acc = ChunkAccumulator::new(/* chunk_samples: */ 32000);
        let chunks = acc.push(&[]);
        assert!(chunks.is_empty());
        assert_eq!(acc.remaining(), 0); // 无残留
    }

    #[test]
    fn exactly_one_chunk() {
        let mut acc = ChunkAccumulator::new(32000);
        let input = vec![0.0f32; 32000];
        let chunks = acc.push(&input);
        assert_eq!(chunks.len(), 1);
        assert_eq!(chunks[0].len(), 32000);
    }

    #[test]
    fn exactly_two_chunks() {
        let mut acc = ChunkAccumulator::new(32000);
        let input = vec![0.0f32; 64000];
        let chunks = acc.push(&input);
        assert_eq!(chunks.len(), 2);
        assert_eq!(chunks[0].len(), 32000);
        assert_eq!(chunks[1].len(), 32000);
    }

    #[test]
    fn partial_chunk_remains_in_buffer() {
        let mut acc = ChunkAccumulator::new(32000);

        // 推入 50000 采样 → 1 个完整 chunk + 18000 残留
        let input = vec![0.0f32; 50000];
        let chunks = acc.push(&input);
        assert_eq!(chunks.len(), 1);
        assert_eq!(chunks[0].len(), 32000);
        assert_eq!(acc.remaining(), 18000);

        // 再推 14000 → 填满第二个 chunk（18000 + 14000 = 32000）
        let input2 = vec![0.0f32; 14000];
        let chunks2 = acc.push(&input2);
        assert_eq!(chunks2.len(), 1);
        assert_eq!(chunks2[0].len(), 32000);
        assert_eq!(acc.remaining(), 0);
    }

    #[test]
    fn multiple_fills_cross_boundary() {
        let mut acc = ChunkAccumulator::new(32000);

        // 推 10000 × 4 = 超过 32000
        let chunks = acc.push(&vec![0.0f32; 10000]);
        assert!(chunks.is_empty());  // 10000 < 32000
        assert_eq!(acc.remaining(), 10000);

        let chunks = acc.push(&vec![0.0f32; 10000]);
        assert!(chunks.is_empty());  // 20000 < 32000
        assert_eq!(acc.remaining(), 20000);

        let chunks = acc.push(&vec![0.0f32; 10000]);
        assert!(chunks.is_empty());  // 30000 < 32000

        // 再推 10000 → 30000 + 10000 = 40000 → 1 chunk + 8000 残留
        let chunks = acc.push(&vec![0.0f32; 10000]);
        assert_eq!(chunks.len(), 1);
        assert_eq!(chunks[0].len(), 32000);
        assert_eq!(acc.remaining(), 8000);
    }

    #[test]
    fn flush_drains_remaining_samples() {
        let mut acc = ChunkAccumulator::new(32000);
        acc.push(&vec![0.0f32; 18000]);
        assert_eq!(acc.remaining(), 18000);

        let remainder = acc.flush();
        assert_eq!(remainder.len(), 18000);
        assert_eq!(acc.remaining(), 0);
    }

    #[test]
    fn flush_empty_returns_empty() {
        let mut acc = ChunkAccumulator::new(32000);
        let remainder = acc.flush();
        assert!(remainder.is_empty());
    }

    #[test]
    fn reset_clears_buffer() {
        let mut acc = ChunkAccumulator::new(32000);
        acc.push(&vec![0.0f32; 18000]);
        assert_eq!(acc.remaining(), 18000);

        acc.reset();
        assert_eq!(acc.remaining(), 0);
    }
}
```

## GREEN — 最小实现

```rust
/// 固定窗口音频累加器。
/// 收任意长度 f32 采样，满 `chunk_samples` 时吐出完整窗口。
/// 余量滚入下一轮，录制结束时调用 `flush()` 收取残余。
pub struct ChunkAccumulator {
    chunk_samples: usize,
    buffer: Vec<f32>,
}

impl ChunkAccumulator {
    pub fn new(chunk_samples: usize) -> Self {
        Self {
            chunk_samples,
            buffer: Vec::with_capacity(chunk_samples * 2),
        }
    }

    /// 推入音频采样，返回已满的完整窗口列表。
    pub fn push(&mut self, samples: &[f32]) -> Vec<Vec<f32>> {
        self.buffer.extend_from_slice(samples);
        let mut chunks = Vec::new();

        while self.buffer.len() >= self.chunk_samples {
            let chunk: Vec<f32> = self.buffer.drain(..self.chunk_samples).collect();
            chunks.push(chunk);
        }

        chunks
    }

    /// 收取 buffer 中残留的所有采样（不足一个完整窗口）。
    pub fn flush(&mut self) -> Vec<f32> {
        self.buffer.drain(..).collect()
    }

    /// 当前 buffer 中残留的采样数。
    pub fn remaining(&self) -> usize {
        self.buffer.len()
    }

    /// 清空 buffer（用于首次启动或错误恢复）。
    pub fn reset(&mut self) {
        self.buffer.clear();
    }
}
```

## REFACTOR

- `buffer` 预分配不再用 `with_capacity(chunk_samples * 2)`，而用 `Vec::new()` 让 push 时自然扩容——实测两种策略内存差异可忽略
- 考虑导出 `pub const CHUNK_SAMPLES_2S_16KHZ: usize = 32000;` 常量，避免硬编码
- 单元测试覆盖后无需再改

## 预计耗时

< 45 分钟
