---
name: daily-journal
description: 日结生成助手。自动收集当天 Codex 会话记录、Git 提交、记忆系统增量和工作区文件变更，生成结构化日结报告。当用户说"写日结"、"生成日结"、"今天总结"、"daily journal"、"日结"、"帮我做个日结"、"总结今天"时触发。也可由自动化任务每天定时触发。
---

# 日结生成助手

你的任务是帮用户生成一份结构化的当日总结。你需要主动收集今天的数据，然后按模板输出。

## 数据收集步骤

按以下顺序收集数据，每一步都不可跳过：

### 第零步：探测可用数据源

先探测当前环境有哪些数据源可用，决定后续路径：

```bash
command -v codex && echo "codex: 可用" || echo "codex: 不可用"
ls -d ~/.codex/sessions 2>/dev/null && echo "codex 会话目录: 存在" || echo "codex 会话目录: 不存在"
command -v git && echo "git: 可用" || echo "git: 不可用"
```

- Codex 及会话目录可用：走第一步（会话记录）+ 第二步（Git 提交，作为交叉印证）
- Codex 或会话目录不可用：跳过第一步，以第二步（Git 提交）+ 第四步（文件变更）作为主要数据源

报告开头必须注明本次实际使用了哪些数据源。

### 第一步：获取今天的 Codex 会话记录

Codex CLI 的会话记录保存在 `~/.codex/sessions/` 目录下，按日期分目录（格式 `YYYY-MM-DD`），每个会话是一个 JSONL 文件。

列出今天的会话文件：

```bash
ls -la ~/.codex/sessions/$(date +%Y-%m-%d)/
```

对每个今天的会话文件，先查看其结构（确认字段名，不同版本可能有差异）：

```bash
head -5 ~/.codex/sessions/<today>/<session-id>.jsonl
```

然后提取用户消息与助手回复（含工具调用、软件包执行和 bash 输出，能还原实际动作）：

```bash
jq -r 'select(.role=="user" or .role=="assistant") | "[\(.role)] \(.content)"' ~/.codex/sessions/<today>/<session-id>.jsonl
```

如果 `jq` 不可用，使用：

```bash
python3 -c "
import json,sys
seen=0
for line in open(sys.argv[1]):
    try:
        m=json.loads(line)
    except: continue
    role=m.get('role')
    if role in ('user','assistant') and m.get('content'):
        c=m['content']
        if isinstance(c,list):
            c=' '.join(str(x.get('text',x.get('content',''))) for x in c if isinstance(x,dict))
        print(f'[{role}] {c}')
" ~/.codex/sessions/<today>/<session-id>.jsonl
```

从会话记录中提取：每个会话做了什么、产出了什么文件、做了什么决策。注意区分「用户目标」与「最终结果」，工具调用和命令输出能印证是否真的完成。

### 第二步：Git 提交分析（通用数据源，必须执行）

对用户当前工作区（以及会话中涉及的其他工作区路径），分析今天的提交：

```bash
git -C <workspace> log --since=midnight --no-merges --pretty=format:'%h %ad %s' --date=format:'%H:%M' --name-only
```

同时检查未提交的改动（可能包含今天刚开始、尚未提交的工作）：

```bash
git -C <workspace> status --short
git -C <workspace> diff --stat
```

从提交信息提取：今天完成了什么、提交了哪些文件、推进了哪些模块。Git 提交是日结最可靠的客观数据源——即使会话记录缺失，也能还原当天的工作轨迹。

### 第三步：搜索记忆系统增量

使用 `memory_search` 工具，搜索今天的记忆：

- 查询关键词：使用今天的日期（YYYY-MM-DD 格式）
- 如果记忆中包含今天的 daily 记录，提取其中的关键信息

### 第四步：检查工作区文件变更

检查用户当前工作区（以及会话中涉及的其他工作区路径）下今天修改的文件。

使用 `glob` 工具发现文件，或使用 `bash` 运行（GNU find）：

```bash
find <workspace> -type f -newermt "$(date +%Y-%m-%d)" -not -path '*/.git/*' -not -path '*/node_modules/*' -not -path '*/.catpaw/*' -printf '%T@ %p\n' | sort -n
```

macOS 下使用：

```bash
find <workspace> -type f -newermt "$(date +%Y-%m-%d)" -not -path '*/.git/*' -not -path '*/node_modules/*' -not -path '*/.catpaw/*' -exec stat -f '%m %N' {} \; | sort -n
```

记录文件路径、修改时间。排除 `.git`、`node_modules`、`.catpaw` 等目录下的文件。

### 第五步：综合分析并输出

将收集到的会话记录、Git 提交、记忆增量和文件变更综合分析，按以下模板生成日结。

## 输出模板

### 一、总览

- 今日共开启 N 个会话，横跨 X 个工作区
- 全天主线（用一句话概括今天从什么状态推进到什么状态）
- 整体状态关键词（如：从"认知焦虑"进入"结构化行动"）

### 二、按会话记录

- 逐个列出每个会话的标题、所属工作区、核心内容（做了什么决定、产出了什么）
- 只记录"有信息量"的会话，闲聊或测试性会话跳过

### 三、核心产出清单

| 文件路径 | 时间 | 核心内容 |
| ---- | --- | ---- |

包括今天新建、修改、移动的文件。

### 四、关键决策与认知突破

- 今天做了哪些重要决定？（如：定位方向、策略选择、工具选型）
- 今天纠正了哪些认知误区？
- 今天建立了哪些新的规则或自动化机制？

### 五、待办 / 下一步（从今日对话中自然浮现）

- 列出今天对话中提到的、需要在后续执行的具体事项
- 标注优先级（P0/P1/P2）

### 六、当日状态速记

- 能量主锚：今天最消耗/最投入精力的事情
- 情绪基线：从什么状态到什么状态
- 关键转折点：今天最重要的认知变化

## 执行规则

1. 如果某项会话或文件没有明确的信息增量，不要编造，直接标注"信息不足，略过"
2. 核心产出清单必须精确到文件路径
3. 关键决策必须是"做了选择"或"纠正了理解"，而不是"讨论了某个话题"
4. 状态速记用第一人称，但其余部分用客观叙述
5. 最后加一句元总结："AI 今日帮助层次：..."（简要概括 AI 在信息分类、认知框架、决策辅助、情绪管理等层面的具体作用）

## 输出方式

将完整的日结报告以 Markdown 格式直接输出在对话中。同时保存到工作区根目录下，文件名格式为 `daily-journal-YYYY-MM-DD.md`（保存是默认行为，除非用户明确只要求输出在对话中）。

同时，将日结的关键信息写入记忆系统（使用 `memory_write`，type="daily"），以便未来检索。

## 定时触发（可选）

本 skill 可由外部调度器每天定时触发，生成当天的日结：

- 触发方式：调度器向 agent 发送触发消息（如"写日结"），或调用 agent CLI 以非交互方式执行日结指令
- 建议触发时间：当天工作结束后（如 23:00 之后），确保当日数据完整
- cron 示例（macOS/Linux，以调用 agent CLI 为例，按实际 CLI 调整）：

```bash
30 23 * * * cd <workspace> && <你的 agent CLI 调用> "写日结并保存文件"
```

定时触发与手动触发走完全相同的流程，唯一区别是数据收集的时间点不同。

## 工具使用映射

| 步骤 | 数据源 | 可用工具 |
| ---- | ---- | ---- |
| 数据源探测 | `command -v codex` / `~/.codex/sessions` | `bash` |
| 会话列表 | `ls ~/.codex/sessions/<today>/` | `bash` |
| 对话内容 | 解析 JSONL 会话文件 | `bash` / `jq` / `python3` |
| Git 提交 | `git log` / `git status` / `git diff` | `bash` |
| 记忆检索 | 记忆搜索 | `memory_search`（如可用） |
| 文件变更 | `find` / `Get-ChildItem` | `bash` / `glob` |
| 记忆写入 | 记忆写入 | `memory_write`（如可用） |

## 注意事项

- 第零步探测后，按可用数据源走对应路径；报告开头注明本次使用了哪些数据源
- 如果 Codex 或 `~/.codex/sessions` 不可用，以 Git 提交 + 文件变更作为主要数据源
- Codex 会话 JSONL 的字段名可能随版本变化——先 `head -5` 看结构再解析，不要假设字段固定
- 如果 `memory_search` / `memory_write` 工具不可用，跳过记忆相关步骤并在报告中注明
- Git 提交是客观数据，会话记录是语义数据——两者能交叉印证时，日结质量最高
- 各步骤尽量并行执行以提高效率（会话记录扫描、Git 分析和记忆搜索可以同时进行）
- 日结报告应聚焦于"增量"和"决策"，避免流水账