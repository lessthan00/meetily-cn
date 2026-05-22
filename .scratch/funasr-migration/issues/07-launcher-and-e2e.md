# 07 — 一键启动脚本 + 端到端测试

**Status**: ready-for-agent  
**Type**: HITL

## Parent

ADR-003: 声纹识别与说话人分离架构

## What to build

创建一键启动脚本，让用户双击一个文件即可启动所有服务（FunASR :8178 + 摘要服务 :5167 + Tauri App），并完成端到端验证。

端到端行为：用户双击 `start.bat`（Windows）/ `start.sh`（Linux/macOS）→ 脚本自动检查 Python 环境、安装依赖、依次启动 FunASR 和摘要服务 → 最后启动 Tauri App → 用户可在 App 中完成完整的会议录制流程。

### 脚本设计 (grill-with-docs 确认)

**拆分为三个独立启动脚本 + 一个编排脚本**——而非单一 400+ 行文件。仅支持 Windows + Debian。

```
start.sh              ← 自动安装依赖 → 顺序启动 → 等待 → 打开 Tauri
  ├── start-funasr.sh  ← python services/funasr/main.py (日志 → logs/funasr.log)
  ├── start-backend.sh ← python backend/app/main.py (日志 → logs/backend.log)
```

**依赖安装策略**：

| 依赖 | 策略 |
|---|---|
| pip 包 (funasr, fastapi, uvicorn, aiosqlite) | 自动 pip install |
| Python 3.9+ | 自动安装: Debian `sudo apt install -y python3` / Windows `winget install Python.Python.3.12` |
| Ollama | 自动安装: Debian `curl -fsSL https://ollama.com/install.sh \| sh` / Windows `winget install Ollama.Ollama` |

**`start.sh` (Debian 编排脚本)**：

```bash
#!/bin/bash
set -e

# 进度条辅助函数
progress() { echo ">>> [$(date +%H:%M:%S)] $*"; }

# 1. 检查并自动安装 Python
progress "检查 Python 环境..."
python3 -c "import sys; assert sys.version_info >= (3,9)" 2>/dev/null || {
    progress "正在安装 Python 3.9+..."
    sudo apt update -qq && sudo apt install -y python3 python3-pip
}

# 2. 检查并自动安装 Ollama
progress "检查 Ollama..."
curl -s localhost:11434 > /dev/null 2>&1 || {
    progress "正在安装 Ollama..."
    curl -fsSL https://ollama.com/install.sh | sh
}
progress "Ollama 已就绪"

# 3. 自动安装 pip 依赖
progress "安装 Python 依赖 (funasr, fastapi, ...)..."
pip install funasr fastapi uvicorn aiosqlite | tail -1
progress "Python 依赖已就绪"

# 4. 顺序启动服务 (后台)
progress "启动 FunASR 语音引擎 (首次需下载模型 ~230MB)..."
./start-funasr.sh & FPID=$!
progress "启动摘要服务..."
./start-backend.sh & BPID=$!

# 5. 等待 FunASR 就绪 (含模型下载进度)
for i in $(seq 1 60); do
    RESP=$(curl -s localhost:8178/health)
    STATUS=$(echo "$RESP" | python3 -c "import sys,json; print(json.load(sys.stdin)['status'])")
    case "$STATUS" in
        "loading")
            PROG=$(echo "$RESP" | python3 -c "import sys,json; d=json.load(sys.stdin); print(f\"{d['model']} {d['progress']:.0f}%\")")
            printf "\r>>> 模型下载中: %s" "$PROG"
            ;;
        "ok")
            printf "\r>>> FunASR 语音引擎就绪           \n"
            break
            ;;
        "error")
            printf "\n!!! 模型下载失败，检查 logs/funasr.log\n"
            exit 1
            ;;
    esac
    sleep 2
done

# 6. 启动 Tauri (前台)
progress "启动 Meetily 桌面端..."
./start-tauri.sh
kill $FPID $BPID 2>/dev/null
```

**`start.bat` (Windows 编排脚本)**：同样逻辑，使用 `winget` + `pip`，进度条用 `echo` 单行刷新。

**`start.bat` (Windows 编排脚本)**：同样逻辑，使用 `winget` + `pip`。

**`start-funasr.sh`** — 启动 FunASR (:8178)，stdout/stderr 写入 `logs/funasr.log`。首次下载时 `/health` 返回 `{"status": "loading", "progress": 45}`，前端或 start.sh 轮询等待。

**`start-backend.sh`** — 启动 FastAPI (:5167)，stdout/stderr 写入 `logs/backend.log`。

### 脚本需要做的事

### Ollama 管理（前端引导，不由脚本静默处理）

**Ollama 未安装**：
- 前端「生成摘要」区域显示横幅：「摘要功能需要 Ollama」
- 「安装 Ollama」按钮 → 打开系统浏览器跳转到 https://ollama.com/download
- 安装完成后用户重启应用，横幅消失

**Ollama 已安装但缺少 qwen3:8b**：
- 前端「生成摘要」区域显示：「尚未下载摘要模型 qwen3:8b (~5GB)」
- 「下载模型」按钮 → Tauri 调用 `ollama pull qwen3:8b`，前端显示实时进度条（解析 ollama pull 的 stdout 进度）
- 下载完成后按钮变为「生成摘要」，可正常使用

**Ollama + 模型均就绪**：
- 「生成摘要」按钮正常可点击
- 摘要服务 (:5167) 已由启动脚本在后台启动
6. Ctrl+C 时依次关闭所有后台进程

### 端到端测试场景

需要人工验证以下完整流程：

- [ ] 双击启动脚本，所有 3 个服务正常运行
- [ ] 声纹注册：注册 2 个说话人（不同人音频、固定朗读文本）
- [ ] 声纹库空 → 弹窗引导注册。跳过 → 正常开始会议
- [ ] 会议录音：2 个已注册说话人 + 1 个未注册说话人进行 3 分钟对话
- [ ] 实时转录：前端按 segments 显示中文转录 + 已注册者姓名 + 未注册者「说话人 A/B/C」
- [ ] 会中命名：点击说话人标签 → 输入姓名 → 后续该人 segments 即刻标注新姓名
- [ ] 停止录音：弹出未注册说话人批量命名界面 → 提示是否注册声纹
- [ ] 自动标题生成：会后标题从日期时间替换为 AI 生成标题
- [ ] 手动点击「生成摘要」→ Ollama 生成纪要，在前端展示
- [ ] Markdown 导出：含 AI 摘要 + 说话人标注 + 未知 speaker 处理，Typora 渲染正常
- [ ] 音频导出：导出原始 WAV 录音
- [ ] 崩溃恢复：录音中途 kill FunASR → 断路器打开 → 前端横幅 → 重启 FunASR → 健康探测恢复 → seq 去重续传 → 获得完整转录
- [ ] 已知问题记录：任何边界 case 或未覆盖场景记录为新的 issue

## Acceptance criteria

- [ ] Windows 下 `start.bat` 双击可用
- [ ] 脚本自动安装缺失的 pip 依赖（funasr、fastapi 等）。系统级依赖（Python、Ollama）仅检查并打印安装指令
- [ ] 首次启动模型下载时，前端显示进度条（含百分比和当前模型名）
- [ ] 模型下载完成后，进度条消失，自动进入正常模式
- [ ] 脚本 Ctrl+C 退出时无后台进程残留
- [ ] 完整端到端测试流程可走通
- [ ] 无崩溃/panic/报错
- [ ] 测试日志/录屏保留以供归档

## Blocked by

#01-#06 全部完成