# local/patches — Hermes 本地补丁与恢复资料

> 这个分支存放**本机 Hermes 的本地补丁和恢复资料**。
> 用途：万一本地 Hermes 因为补丁无法启动，从这里取回资料恢复。
> **注意**：这不是给官方发 PR 用的分支，纯粹是本地备份。

## 📁 内容

| 路径 | 说明 |
|---|---|
| `local-patches/RECOVERY.md` | **恢复文档**（改了什么、怎么回滚、怎么验证）|
| `local-patches/0001-crlf-fix.patch` | CRLF 修复补丁文件 |
| `local-patches/backups/utils.py.bak-20260924` | `utils.py` 的**原始版本备份** |
| `utils.py`（本分支） | 已应用 CRLF 修复的版本 |

## 🔧 当前补丁清单

### 0001 — CRLF 修复（`utils.py`）

**改了什么**：
```python
# 第 291 行附近，_atomic_write 内
- os.fdopen(fd, "wb" if binary else "w", encoding=None if binary else encoding) as f:
+ os.fdopen(fd, "wb" if binary else "w", encoding=None if binary else encoding,
+           newline=None if binary else "") as f:
```

**为什么**：Python 文本模式在 Windows 上把 `\n` 转成 `\r\n`，导致技能文件 CRLF、patch 匹配失败、`hermes skills check` 误报、MEMORY.md 分隔符风险。

**上游状态**：官方已知（issue #109674 / PR #109685），未合并 → 本地 workaround。

## 🚨 紧急恢复（Hermes 启动不了时）

```bash
# 方式 1：用备份覆盖（不需要 Hermes）
cp local-patches/backups/utils.py.bak-20260924 <安装目录>/utils.py

# 方式 2：用 git
cd <安装目录> && git checkout utils.py

# 方式 3：交给 AI
「读取 local-patches/RECOVERY.md，按紧急恢复流程恢复 Hermes」
```

安装目录：`D:\Users\86150\AppData\Local\hermes\hermes-agent`

## ⚠️ 注意事项

- `hermes update` 可能冲掉本地补丁 → 更新后检查 `utils.py` 是否还有 `newline` 参数
- 改完代码**必须重启 Hermes** 才生效（Python 启动时加载模块）
- 本分支的 main 保持干净（用于将来可能的 PR）；补丁都放 `local/patches`
