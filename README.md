# daily-journal · 日结生成 Skill

一个可复用的 **日结生成助手** Skill：自动收集当天的会话记录、Git 提交、记忆增量和文件变更，生成结构化的日结报告。

适用于 Codex、Claude Code、以及支持 Skills 机制（`skills/<name>/SKILL.md`）的各类 Agent 环境。

## ✨ 功能

- **多数据源自动收集**：CatDesk 会话日志、Git 提交分析、记忆系统增量、工作区文件变更
- **智能降级**：CatDesk 不可用时自动以 Git 提交 + 文件变更为主要数据源，任何有 git 的环境都能工作
- **六段结构化输出**：总览 → 按会话记录 → 核心产出清单 → 关键决策与认知突破 → 待办/下一步 → 当日状态速记
- **抗编造**：信息不足时标注"信息不足，略过"，绝不编造
- **持久化**：日结报告自动保存为 `daily-journal-YYYY-MM-DD.md`，并写入记忆系统（如可用）
- **可定时触发**：支持通过 cron 等调度器每日自动生成

## 📦 安装

### 方式一：复制到 Agent 的 skills 目录（推荐）

将本仓库的 `skills/daily-journal` 目录复制到你的 Agent 支持的 skills 根目录下：

```bash
# 示例：复制到标准 skills 目录
cp -r skills/daily-journal <你的skills根目录>/daily-journal
```

具体路径取决于你的 Agent：

- **DSH (DeepSeek Harness)**：放入 preset 的 `skills/` 目录，如 `~/.dsh/.agent-presets/<preset>/skills/daily-journal/`
- **Codex / Claude Code**：放入其 skills 配置目录（如 `~/.codex/skills/`、`~/.claude/skills/`）

### 方式二：整个 preset（DSH 用户）

如果你使用 DSH，也可以将本仓库的结构放入用户 preset 目录，作为完整的「日结模式」preset。

## 🚀 使用

装好后，在任何会话中说以下任一触发词：

> "写日结"、"生成日结"、"今天总结"、"daily journal"、"日结"、"帮我做个日结"、"总结今天"

### 定时触发（可选）

```bash
# 每天 23:30 自动生成日结（macOS/Linux cron）
30 23 * * * cd <workspace> && <你的 agent CLI 调用> "写日结并保存文件"
```

## 📋 输出示例

```
一、总览
二、按会话记录
三、核心产出清单（文件路径 | 时间 | 核心内容）
四、关键决策与认知突破
五、待办 / 下一步（P0/P1/P2）
六、当日状态速记
AI 今日帮助层次：...
```

完整模板见 [`SKILL.md`](skills/daily-journal/SKILL.md)。

## 🗂 目录结构

```
daily-journal/
└── skills/
    └── daily-journal/
        └── SKILL.md        # Skill 本体（含完整数据收集流程与输出模板）
```

## 📄 License

[MIT](LICENSE)