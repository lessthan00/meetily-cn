# 00 — 环境门禁

**Status**: ready-for-agent

## What to build

在任何 TDD 循环开始之前，验证全部工具链可运行。此 issue 不产生功能代码——只产生环境就绪信号。

### 验证清单 (按顺序，逐项通过)

1. **Python 3.9+**:
   ```bash
   python3 -c "import sys; assert sys.version_info >= (3, 9)"
   ```

2. **pip install funasr**:
   ```bash
   pip install funasr fastapi uvicorn aiosqlite
   python3 -c "import funasr; import fastapi; import uvicorn; import aiosqlite"
   ```

3. **cargo build (现有代码)**:
   ```bash
   cd frontend/src-tauri && cargo build 2>&1
   ```
   必须成功。如失败，记录错误并停止。

4. **cargo test 空跑**:
   ```bash
   cd frontend/src-tauri && cargo test 2>&1
   ```
   所有已有测试通过。测量单测增量编译时间 (修改一个 `#[test]` 后重跑)，< 60s 否则配 sccache：
   ```bash
   cargo install sccache
   export RUSTC_WRAPPER=sccache
   ```

5. **pytest 空跑**:
   ```bash
   cd backend && python3 -m pytest --co 2>&1 || echo "pytest installed"
   ```

6. **Ollama 可用 (best-effort)**:
   ```bash
   curl -s localhost:11434 > /dev/null && echo "OK" || echo "摘要功能需 Ollama (不影响 TDD)"
   ```

### 输出

通过后输出：

```
=== 环境就绪 ===
Python: 3.x
funasr: <version>
cargo build: OK
cargo test: OK (<time>s each)
pytest: OK
Ollama: <ok|missing>
```

然后标记此 issue completed，开始 Issue A。

## Acceptance criteria

- [ ] pip install funasr 成功，import 无报错
- [ ] cargo build 成功，无编译错误
- [ ] cargo test 全部已有测试通过
- [ ] pytest 可执行
- [ ] `cargo test` 增量编译时间记录在案

## Blocked by

None — 最上游，无依赖
