# TDD 03d — 声纹加密导出/导入

**父 Issue**: [#03 声纹注册 + 实时说话人标注](../issues/03-voiceprint-enrollment-and-identification.md)

## 测试范围

声纹库导出为 AES-256-GCM 加密文件 → 导入时解密并校验完整性，密码错误/密文篡改均应拒绝。

## RED — 先写失败测试

```python
# test_speaker_import_export.py
import json
import pytest


# --- 加密/解密 ---

def test_encrypt_decrypt_roundtrip():
    """加密 → 解密 → 原文一致"""
    plaintext = b'{"version": 1, "speakers": []}'
    password = "my-secret-pw"
    ciphertext = encrypt_speakers_export(plaintext, password)
    decrypted = decrypt_speakers_import(ciphertext, password)
    assert decrypted == plaintext


def test_wrong_password_rejected():
    """密码错误 → 解密失败"""
    plaintext = b'{"version": 1, "speakers": []}'
    ciphertext = encrypt_speakers_export(plaintext, "correct-pw")
    with pytest.raises(ValueError, match="password|decrypt"):
        decrypt_speakers_import(ciphertext, "wrong-pw")


def test_tampered_ciphertext_rejected():
    """密文被篡改 → GCM tag 校验失败"""
    plaintext = b'{"version": 1, "speakers": []}'
    ciphertext = bytearray(encrypt_speakers_export(plaintext, "pw"))
    ciphertext[-1] ^= 0xFF  # 翻转最后一个字节
    with pytest.raises(ValueError, match="tamper|integrity|tag"):
        decrypt_speakers_import(bytes(ciphertext), "pw")


def test_empty_password_rejected():
    """空密码 → 拒绝"""
    with pytest.raises(ValueError, match="password"):
        encrypt_speakers_export(b"data", "")


# --- JSON 格式校验 (加密内层) ---

def test_export_empty_registry():
    """空库 → JSON 仅有元数据"""
    j = export_speakers_json([])
    data = json.loads(j)
    assert data["version"] == 1
    assert "exported_at" in data
    assert data["speakers"] == []


def test_export_one_speaker():
    """一个说话人 → JSON 包含完整向量"""
    speakers = [{"id": 1, "name": "张甜甜", "embedding": [0.1] * 256}]
    j = export_speakers_json(speakers)
    data = json.loads(j)
    assert len(data["speakers"]) == 1
    assert data["speakers"][0]["name"] == "张甜甜"
    assert len(data["speakers"][0]["embedding"]) == 256


def test_full_roundtrip_encrypted():
    """完整链路: JSON → 加密 → 解密 → JSON → merge"""
    original = [{"id": 1, "name": "张甜甜", "embedding": [0.1] * 256}]
    json_str = export_speakers_json(original)
    ciphertext = encrypt_speakers_export(json_str.encode(), "secure-pw")
    decrypted_bytes = decrypt_speakers_import(ciphertext, "secure-pw")
    imported = parse_import_json(decrypted_bytes.decode())
    assert len(imported) == 1
    assert imported[0]["name"] == "张甜甜"
    assert imported[0]["embedding"] == [0.1] * 256


def test_import_merge_new_speakers():
    """增量合并：新说话人添加，已有说话人不删"""
    existing = {1: {"name": "张甜甜", "embedding": [0.1] * 256}}
    j = export_speakers_json([
        {"id": 2, "name": "李四", "embedding": [0.2] * 256}
    ])
    merge_import_json(existing, j)
    assert len(existing) == 2


def test_import_overwrite_same_name():
    """同名覆盖：相同的 name → 更新 embedding 和时间戳"""
    existing = {1: {"name": "张甜甜", "embedding": [0.1] * 256}}
    j = export_speakers_json([
        {"name": "张甜甜", "embedding": [0.9] * 256}
    ])
    merge_import_json(existing, j)
    assert existing[1]["embedding"][0] == 0.9


def test_import_invalid_json_rejected():
    """非 JSON 输入 → 报错"""
    with pytest.raises(ValueError):
        parse_import_json("not json")


def test_import_wrong_version_rejected():
    """版本号不兼容 → 拒绝"""
    j = json.dumps({"version": 99, "speakers": []})
    with pytest.raises(ValueError, match="version"):
        parse_import_json(j)


def test_import_missing_embedding_rejected():
    """speaker 缺少 embedding 字段 → 跳过或报错"""
    j = json.dumps({"version": 1, "exported_at": "", "speakers": [
        {"name": "测试"}
    ]})
    with pytest.raises(ValueError, match="embedding"):
        parse_import_json(j)


def test_import_truncated_embedding_rejected():
    """embedding 不是 256 维 → 拒绝"""
    j = json.dumps({"version": 1, "exported_at": "", "speakers": [
        {"name": "测试", "embedding": [0.1] * 128}
    ]})
    with pytest.raises(ValueError, match="256"):
        parse_import_json(j)
```

## GREEN — 最小实现

```python
import json
import os
from datetime import datetime
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC
from cryptography.hazmat.primitives import hashes

EXPORT_VERSION = 1
PBKDF2_ITERATIONS = 600_000
GCM_NONCE_BYTES = 12
SALT_BYTES = 16


def _derive_key(password: str, salt: bytes) -> bytes:
    if not password:
        raise ValueError("Password must not be empty")
    kdf = PBKDF2HMAC(algorithm=hashes.SHA256(), length=32, salt=salt, iterations=PBKDF2_ITERATIONS)
    return kdf.derive(password.encode())


def encrypt_speakers_export(plaintext: bytes, password: str) -> bytes:
    salt = os.urandom(SALT_BYTES)
    key = _derive_key(password, salt)
    nonce = os.urandom(GCM_NONCE_BYTES)
    aesgcm = AESGCM(key)
    ct = aesgcm.encrypt(nonce, plaintext, None)
    return salt + nonce + ct  # salt(16) + nonce(12) + ciphertext+tag


def decrypt_speakers_import(ciphertext: bytes, password: str) -> bytes:
    salt = ciphertext[:SALT_BYTES]
    nonce = ciphertext[SALT_BYTES:SALT_BYTES + GCM_NONCE_BYTES]
    ct = ciphertext[SALT_BYTES + GCM_NONCE_BYTES:]
    key = _derive_key(password, salt)
    aesgcm = AESGCM(key)
    try:
        return aesgcm.decrypt(nonce, ct, None)
    except Exception:
        raise ValueError("Decryption failed: wrong password or corrupted file")


def export_speakers_json(speakers: list[dict]) -> str:
    data = {
        "version": EXPORT_VERSION,
        "exported_at": datetime.utcnow().isoformat(),
        "speakers": [
            {"name": s["name"], "embedding": s["embedding"], "created_at": s.get("created_at", "")}
            for s in speakers
        ]
    }
    return json.dumps(data, ensure_ascii=False, indent=2)


def parse_import_json(j: str) -> list[dict]:
    data = json.loads(j)
    if data.get("version") != EXPORT_VERSION:
        raise ValueError(f"Unsupported version: {data.get('version')}")
    speakers = []
    for s in data["speakers"]:
        if "embedding" not in s:
            raise ValueError(f"Missing embedding for speaker: {s.get('name')}")
        if len(s["embedding"]) != 256:
            raise ValueError(f"Embedding must be 256-dim for speaker: {s['name']}")
        speakers.append({"name": s["name"], "embedding": s["embedding"]})
    return speakers


def merge_import_json(existing: dict, j: str):
    imported = parse_import_json(j)
    for spk in imported:
        for sid, entry in existing.items():
            if entry["name"] == spk["name"]:
                entry["embedding"] = spk["embedding"]
                break
        else:
            new_id = max(existing.keys(), default=0) + 1
            existing[new_id] = spk
```

## REFACTOR

- 考虑用 `pydantic` model 做 JSON schema 校验而非手动逐字段检查
- 大文件（> 100 人）时用流式 JSON 解析

## 预计耗时

< 1.5 小时
