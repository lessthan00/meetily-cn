# TDD 01a — speaker_embeddings 表 CRUD

**父 Issue**: [#01 数据库迁移 + 声纹库表](../issues/01-database-migration.md)

## 测试范围

对 `speaker_embeddings` 表的增删查操作进行独立单元测试（不依赖 FunASR 进程）。

## RED — 先写失败测试

```python
# test_speaker_embedding_crud.py

def test_insert_and_query_speaker(db):
    """插入一条声纹记录后可以查询到"""
    db.insert_speaker(name="张甜甜", embedding=[0.1] * 256)
    speakers = db.list_speakers()
    assert len(speakers) == 1
    assert speakers[0]["name"] == "张甜甜"
    assert len(speakers[0]["embedding"]) == 256


def test_delete_speaker(db):
    """删除后列表不再包含该记录"""
    db.insert_speaker(name="测试", embedding=[0.0] * 256)
    sid = db.list_speakers()[0]["id"]
    db.delete_speaker(sid)
    assert len(db.list_speakers()) == 0


def test_unique_name_constraint(db):
    """同名插入应失败或覆盖（取决于设计选择）"""
    db.insert_speaker(name="张甜甜", embedding=[0.1] * 256)
    # 第二次插入同名 → 应报错或更新
    with pytest.raises(DuplicateSpeakerError):
        db.insert_speaker(name="张甜甜", embedding=[0.2] * 256)


def test_embedding_dimension_validation(db):
    """非 256 维向量应被拒绝"""
    with pytest.raises(ValueError, match="256"):
        db.insert_speaker(name="测试", embedding=[0.5] * 128)


def test_update_speaker_name(db):
    """改名：UPDATE name，不改 embedding"""
    db.insert_speaker(name="张甜甜", embedding=[0.1] * 256)
    sid = db.list_speakers()[0]["id"]
    db.update_speaker(sid, name="张甜")
    speakers = db.list_speakers()
    assert speakers[0]["name"] == "张甜"
    assert speakers[0]["embedding"] == [0.1] * 256


def test_rename_to_existing_name_rejected(db):
    """改名时目标名字已被占用 → 拒绝"""
    db.insert_speaker(name="张甜甜", embedding=[0.1] * 256)
    db.insert_speaker(name="李四", embedding=[0.2] * 256)
    sid = db.list_speakers()[0]["id"]
    with pytest.raises(DuplicateSpeakerError):
        db.update_speaker(sid, name="李四")


def test_empty_name_rejected(db):
    """空名字应被拒绝"""
    with pytest.raises(ValueError):
        db.insert_speaker(name="", embedding=[0.1] * 256)
```

## GREEN — 最小实现

- `speaker_embeddings` 表: `id INTEGER PRIMARY KEY, name TEXT UNIQUE, embedding BLOB (256×4 bytes float32), created_at TEXT`
- `DatabaseManager.insert_speaker(name, embedding)` — 校验维度 + 唯一性 → 插入
- `DatabaseManager.list_speakers()` — 返回 `[{id, name, embedding, created_at}]`
- `DatabaseManager.update_speaker(id, name)` — 按 id 改名，不改 embedding。目标名已被占用 → DuplicateSpeakerError
- `DatabaseManager.delete_speaker(id)` — 按 id 删除

## REFACTOR

- embedding 存储格式统一（BLOB vs JSON string）—— 选 BLOB，紧凑且比对时直接解包
- 确认 `aiosqlite` 异步模式与测试的同步 SQLite fixture 不冲突

## 预计耗时

< 1 小时（纯 SQL + 校验逻辑）
