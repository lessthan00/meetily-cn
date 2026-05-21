# TDD 03b — k-means 质心分配 + 自适应扩容

**父 Issue**: [#03 声纹注册 + 实时说话人标注](../issues/03-voiceprint-enrollment-and-identification.md)

## 测试范围

在线 k-means 聚类的核心逻辑：接收一个 embedding，分配到最近质心或触发扩容。

## RED — 先写失败测试

```python
# test_kmeans_clustering.py
import pytest


def test_initial_state_empty_clusters():
    """初始化时 k 个质心均为 None"""
    c = SpeakerClusterer(k=3)
    assert len(c.centroids) == 3
    assert all(v is None for v in c.centroids)


def test_first_segment_claims_first_cluster():
    """第一个语音段：分配到 cluster 0"""
    c = SpeakerClusterer(k=3)
    emb = [0.0625] * 256
    label = c.assign(emb)
    assert label == "说话人 A"
    assert c.centroids[0] is not None


def test_same_speaker_assigned_same_label():
    """同一说话人的 embedding 连续到达，应归入同一聚类"""
    c = SpeakerClusterer(k=3)
    emb1 = [0.0625] * 256
    emb2 = [0.0630] * 256  # 极其相似
    assert c.assign(emb1) == "说话人 A"
    assert c.assign(emb2) == "说话人 A"  # 不是 B


def test_different_speaker_gets_new_label():
    """不同说话人 → 新聚类标签"""
    c = SpeakerClusterer(k=3)
    c.assign([0.0625] * 256)  # → 说话人 A
    label = c.assign([-0.0625] * 256)  # → 完全不同
    assert label == "说话人 B"


def test_adaptive_expansion_when_k_exhausted():
    """k 个聚类都被不同的说话人占用后，新说话人应触发扩容"""
    c = SpeakerClusterer(k=2)
    # 填充 2 个聚类
    c.assign([0.0625] * 256)   # A
    c.assign([-0.0625] * 256)  # B
    # 第三个完全不同的人，但与所有质心都 < 0.5
    emb3 = [0.03 if i % 2 == 0 else -0.03 for i in range(256)]
    label = c.assign(emb3)
    assert label == "说话人 C"  # 自动扩容到 k=3
    assert c.k == 3


def test_no_expansion_for_close_match():
    """与已有聚类相似度 ≥ 0.7 → 归入该聚类，不扩容"""
    c = SpeakerClusterer(k=2)
    c.assign([0.0625] * 256)  # A
    # 第二个 voice 与 A 相似度很高（接近 1.0）
    emb = [0.0624] * 256
    label = c.assign(emb)
    assert label == "说话人 A"
    assert c.k == 2  # 没有扩容


def test_boundary_zone_tolerated():
    """相似度 0.5–0.7 → 归入最近聚类（容忍），不扩容"""
    c = SpeakerClusterer(k=2)
    c.assign([0.0625] * 256)  # A
    # 构造一个相似度约 0.6 的向量
    # cos(a,b) ≈ 0.6 → 一半相同一半正交
    emb = [0.0625] * 128 + [0.0] * 128
    label = c.assign(emb)
    assert label == "说话人 A"  # 归入最近，不是新建
    assert c.k == 2


def test_centroid_updated_after_assignment():
    """质心应随新样本归入而移动"""
    c = SpeakerClusterer(k=1)
    c.assign([1.0] + [0.0] * 255)
    old_centroid = c.centroids[0][:]
    c.assign([0.0] + [1.0] + [0.0] * 254)
    new_centroid = c.centroids[0]
    # 质心应移动：从 [1,0,0,...] 变为 [0.5,0.5,0,...]
    assert new_centroid[0] == pytest.approx(0.5, abs=0.1)
    assert new_centroid[1] == pytest.approx(0.5, abs=0.1)


def test_known_speaker_uses_name_not_label():
    """已注册说话人 → 用姓名而非「说话人 X」"""
    c = SpeakerClusterer(k=3)
    # 模拟注册库匹配
    label = c.assign_with_name([0.0625] * 256, known_name="张甜甜")
    assert label == "张甜甜"
    # 但仍计入聚类（后续质心更新可用）
    assert c.centroids[0] is not None
```

## GREEN — 最小实现

```python
class SpeakerClusterer:
    def __init__(self, k=4):
        self.k = k
        self.centroids = [None] * k
        self.counts = [0] * k  # 每个聚类样本数，用于加权更新
        self.label_pool = []    # ["说话人 A", "说话人 B", ...]

    def _next_label(self):
        n = len(self.label_pool)
        self.label_pool.append(f"说话人 {chr(65 + n)}")  # A, B, C, ...
        return self.label_pool[-1]

    def assign(self, emb, known_name=None):
        if known_name:
            # 已注册 → 直接用姓名
            idx = self._find_or_claim_empty(emb)
            self._update_centroid(idx, emb)
            return known_name

        idx, score = self._find_nearest(emb)
        if idx is not None and score >= 0.7:
            self._update_centroid(idx, emb)
            return self.label_pool[idx]
        elif idx is not None and score >= 0.5:
            self._update_centroid(idx, emb)
            return self.label_pool[idx]
        else:
            # < 0.5 → 自适应扩容
            return self._expand_and_assign(emb)
```

## REFACTOR

- 质心归一化存储（避免每次比对重新 sqrt）
- 考虑滑动窗口：仅用最近 N 个样本更新质心，对抗概念漂移

## 预计耗时

< 1.5 小时（逻辑最复杂的子任务）
