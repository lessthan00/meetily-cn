# TDD 03a — 余弦相似度计算

**父 Issue**: [#03 声纹注册 + 实时说话人标注](../issues/03-voiceprint-enrollment-and-identification.md)

## 测试范围

给定两个 256 维向量，计算余弦相似度。覆盖正常值、边界、异常输入。

## RED — 先写失败测试

```python
# test_cosine_similarity.py
import math
import pytest


def test_identical_vectors():
    """相同向量 → 相似度 1.0"""
    v = [0.0625] * 256  # 归一化后每个元素 = 1/sqrt(256) = 0.0625
    assert cosine_similarity(v, v) == pytest.approx(1.0, abs=1e-6)


def test_orthogonal_vectors():
    """正交向量 → 相似度 0.0"""
    a = [1.0] + [0.0] * 255
    b = [0.0] + [1.0] + [0.0] * 254
    assert cosine_similarity(a, b) == pytest.approx(0.0, abs=1e-6)


def test_opposite_vectors():
    """完全相反 → 相似度 -1.0"""
    a = [0.0625] * 256
    b = [-0.0625] * 256
    assert cosine_similarity(a, b) == pytest.approx(-1.0, abs=1e-6)


def test_real_world_range():
    """CAM++ 实际输出通常在 0.3–0.9 之间"""
    import random
    random.seed(42)
    a = [random.uniform(-0.1, 0.1) for _ in range(256)]
    b = [x + random.uniform(-0.05, 0.05) for x in a]  # 相似但不完全相同
    sim = cosine_similarity(a, b)
    assert 0.7 < sim < 1.0


def test_zero_vector_rejected():
    """零向量（未检测到语音）应抛异常或返回特殊值"""
    with pytest.raises(ValueError, match="zero"):
        cosine_similarity([0.0] * 256, [0.1] * 256)


def test_dimension_mismatch():
    """非 256 维向量应报错"""
    with pytest.raises(ValueError, match="256"):
        cosine_similarity([0.1] * 128, [0.1] * 256)


def test_batch_comparison():
    """一对多比对：一个向量 vs 声纹库中 N 个向量"""
    query = [0.0625] * 256
    registry = {
        1: {"name": "A", "embedding": [0.0625] * 256},  # 完全相同
        2: {"name": "B", "embedding": [-0.0625] * 256}, # 完全相反
    }
    best_id, best_score = find_best_match(query, registry)
    assert best_id == 1
    assert best_score == pytest.approx(1.0, abs=1e-4)


def test_batch_no_match_above_threshold():
    """所有相似度都低于阈值 → 返回 None"""
    query = [0.0625] * 256
    registry = {1: {"name": "X", "embedding": [-0.0625] * 256}}
    best_id, best_score = find_best_match(query, registry, threshold=0.7)
    assert best_id is None
```

## GREEN — 最小实现

```python
def cosine_similarity(a: list[float], b: list[float]) -> float:
    if len(a) != 256 or len(b) != 256:
        raise ValueError(f"Expected 256-dim vectors, got {len(a)} and {len(b)}")
    dot = sum(x * y for x, y in zip(a, b))
    norm_a = math.sqrt(sum(x * x for x in a))
    norm_b = math.sqrt(sum(y * y for y in b))
    if norm_a == 0.0 or norm_b == 0.0:
        raise ValueError("Zero-norm vector")
    return dot / (norm_a * norm_b)


def find_best_match(query, registry, threshold=0.7):
    best_id, best_score = None, -1.0
    for sid, entry in registry.items():
        score = cosine_similarity(query, entry["embedding"])
        if score > best_score and score >= threshold:
            best_id, best_score = sid, score
    return best_id, best_score
```

## REFACTOR

- 考虑用 `numpy` 向量化 `find_best_match`（批量比对用 dot product 一次完成）—— 仅当 registry > 50 人时有收益
- 纯 Python 版本可预计算 registry 向量的 L2 范数避免重复 `sqrt`

## 预计耗时

< 45 分钟
