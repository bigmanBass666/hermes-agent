# Hermes 本地补丁恢复说明

> **这份文档的目的**：万一 Hermes 因为本地补丁改动而无法启动，任何人（用户本人 / Codex / 其他 AI）都能照着它恢复。
> 最后更新：2026-09-24

---

## 📍 基本信息

| 项 | 值 |
|---|---|
| Hermes 版本 | v0.21.4 (2026.9.21) |
| 安装目录 | `D:\Users\86150\AppData\Local\hermes\hermes-agent` |
| 数据目录 | `D:\Users\86150\AppData\Local\hermes` |
| 补丁目录 | `D:\Users\86150\AppData\Local\hermes\local-patches` |
| git 状态 | 浅克隆（shallow），origin = 官方仓库 NousResearch/hermes-agent |
| Python 环境 | `D:\Users\86150\AppData\Local\hermes\hermes-agent\venv`（editable 安装，改源码即生效） |

**关键机制**：Hermes 是 **editable 安装**（`.pth` 指向源码目录），所以**改 `hermes-agent/` 下的源码会直接影响运行**，改完重启 Hermes 即生效。

---

## 🔧 本地补丁清单

### 补丁 1：CRLF 修复（`utils.py`）

**目的**：阻止 Windows 上写文件时 `\n` 被转成 `\r\n`

**状态**：✅ **已应用**（2026-09-24）

> ⚠️ **重要：改完必须重启 Hermes 才生效！**
> Python 在**启动时**加载模块，正在运行的进程内存里仍是旧代码。
> **实测证据**（2026-09-24）：改完后在当前会话里创建技能，产出的**仍是 CRLF**；
> 而新起一个 Python 进程调用同一函数，产出的是**纯 LF**。
> → 所以：**改完必须重启 Hermes，重启后才会真正生效**。

**改哪里**：`utils.py` 第 291 行附近，函数 `_atomic_write` 内

```python
# 改前：
with os.fdopen(fd, "wb" if binary else "w", encoding=None if binary else encoding) as f:

# 改后：
with os.fdopen(fd, "wb" if binary else "w", encoding=None if binary else encoding,
               newline=None if binary else "") as f:
```

**为什么改**：Python 文本模式在 Windows 上默认把 `\n` 转成 `os.linesep`（= `\r\n`）。后果：
- 技能文件（SKILL.md）写成 CRLF → LLM 的 patch 精确匹配失败
- `hermes skills check` 永远误报 "update_available"（哈希不匹配）
- `MEMORY.md` 的 `\n§\n` 分隔符匹配失败风险（官方 issue #85825，P0）

**上游状态**：官方已知（issue #109674、PR #109685 未合并）。本补丁是本地 workaround。

---

## 🚨 紧急恢复流程（Hermes 启动不了时）

### 方法 1：复制备份（最快，不需要 Hermes）

1. 打开 `D:\Users\86150\AppData\Local\hermes\local-patches\backups\`
2. 把 `utils.py.bak-20260924` 复制到 `D:\Users\86150\AppData\Local\hermes\hermes-agent\`
3. 重命名为 `utils.py`（覆盖原文件）
4. 重新启动 Hermes

命令行版本：
```bash
cp "D:/Users/86150/AppData/Local/hermes/local-patches/backups/utils.py.bak-20260924" "D:/Users/86150/AppData/Local/hermes/hermes-agent/utils.py"
```

### 方法 2：用 git 回滚

```bash
cd D:/Users/86150/AppData/Local/hermes/hermes-agent
git checkout utils.py
```

### 方法 3：交给 Codex

给 Codex 的指令：
> 读取 `D:\Users\86150\AppData\Local\hermes\local-patches\RECOVERY.md`，
> 按「紧急恢复流程」恢复 Hermes。改的是 utils.py 的 `_atomic_write` 函数。

---

## ✅ 改完后如何验证（3 步）

```bash
cd D:/Users/86150/AppData/Local/hermes/hermes-agent

# ① 语法/导入检查（最重要，秒级）
python -c "import utils; print('导入 OK')"

# ② 能启动吗
./venv/Scripts/hermes.exe --version

# ③ 实际功能验证：写文件看字节
python -c "
import sys; sys.path.insert(0,'.')
from utils import atomic_write_text
from pathlib import Path
p = Path('D:/Users/86150/AppData/Local/hermes/cache/scratch/newline-test.txt')
atomic_write_text(p, 'a\nb\nc\n')
print(repr(p.read_bytes()))
"
# 期望：b'a\nb\nc\n'  （纯 LF）
# 如果出现 b'a\r\nb\r\n' → 修复未生效
```

---

## ⚠️ hermes update 之后要检查

`hermes update` 会 `git reset --hard origin/main`，**可能冲掉本地补丁**。

**更新后检查**：
```bash
grep -n "os.fdopen" "D:/Users/86150/AppData/Local/hermes/hermes-agent/utils.py"
```
- 如果那行**没有** `newline` → 补丁被冲掉 → 按「本地补丁清单」重新应用
- 如果有 → 补丁还在 ✅

**注意**：Hermes 的 update 有 autostash 机制，对**未提交的工作区改动**会自动 stash 并在更新后恢复。但如果 update 走了 `reset --hard`（历史分叉时），可能被冲掉——所以每次更新后都检查一下。

---

## 📋 变更记录

| 日期 | 变更 |
|---|---|
| 2026-09-24 | 建立本文档；备份 utils.py 原始版本 |
| 2026-09-24 | ✅ 应用 CRLF 补丁（utils.py 第 291 行加 `newline` 参数）；语法/启动/字节级验证全部通过；生成 `0001-crlf-fix.patch` |
| 2026-09-24 | ⏳ 待重启 Hermes 使补丁真正生效（运行中的进程仍是旧代码） |
