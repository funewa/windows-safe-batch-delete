# windows-safe-batch-delete

> WorkBuddy Skill — `windows-safe-batch-delete`

## 简介

在 Windows 上安全批量删除大目录（优先移入回收站、可验证、可回滚）。当用户要求"删掉某个临时/构建目录""清理 C 盘某个文件夹""这些文件可以删除吗"且涉及 GB 级或多达上百个条目时使用。内置：只读盘点 → 危险内容识别（WSL vhdx / 虚拟磁盘）→ 逐项回收并校验 → 部分删除后的还原。

## 安装

```bash
git clone https://github.com/funewa/windows-safe-batch-delete.git
cp -r windows-safe-batch-delete ~/.workbuddy/skills/windows-safe-batch-delete
```

## 使用

在 WorkBuddy 中直接描述你的需求，Agent 会依据 `SKILL.md` 的 `description` 自动匹配本技能。详见 `SKILL.md`。

## 目录结构

```
├── SKILL.md      # 技能定义（frontmatter + 说明）
└── README.md
```

---

Published from the `user`-level WorkBuddy skill library.
