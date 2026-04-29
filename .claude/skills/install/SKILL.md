---
name: install
description: 触发条件:用户在 deeparch-archcheck 仓库中输入 /install,或说"装 archcheck"、"安装 archcheck"、"install archcheck"。把当前仓库的 archcheck/ 目录复制到 ~/.claude/skills/archcheck/,使 /archcheck 在 Claude Code 全局可用。等价于手动运行 `cp -r archcheck ~/.claude/skills/`。
---

# /install 技能

## 作用

把本仓库根目录下的 `archcheck/` 复制到 `~/.claude/skills/archcheck/`,完成安装。

## 触发

- `/install`
- "装 archcheck" / "安装 archcheck" / "install archcheck"

## 执行

按顺序运行:

```bash
test -f archcheck/SKILL.md || { echo "请在 deeparch-archcheck 仓库根目录下运行"; exit 1; }
mkdir -p ~/.claude/skills
rm -rf ~/.claude/skills/archcheck
cp -r archcheck ~/.claude/skills/
test -s ~/.claude/skills/archcheck/SKILL.md && echo "✓ 装好了"
```

完成后告诉用户:

> ✓ archcheck 已安装到 `~/.claude/skills/archcheck/`,Claude Code 里 `/archcheck <项目路径>` 即用。
