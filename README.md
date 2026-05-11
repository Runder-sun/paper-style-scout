# Paper Style Scout

A standalone skill for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and [Codex](https://openai.com/index/codex/) that automatically analyzes top papers in your field and generates a project-specific writing skill before you start writing.

**Zero dependencies.** No external scripts, no other skills required. All API calls are inline Python using only the standard library.

## What It Does

Given your research direction, idea, and claims, Paper Style Scout:

1. **Scans your repository** to understand your actual experiments, methods, and results
2. **Searches 3 latest papers** from arXiv most relevant to your work
3. **Searches 2 highest-cited accepted papers** from the past 2 years via Semantic Scholar
4. **Analyzes all 5 papers** across 6 dimensions: story arc, abstract pattern, introduction structure, method presentation, experiment design, and language style
5. **Generates a project-level writing skill** at `.claude/skills/paper-writing-guide/SKILL.md` (for Claude Code) and `.agents/skills/paper-writing-guide/SKILL.md` (for Codex) — proper SKILL.md files with frontmatter that both agents can discover and load, containing:
   - Reference paper analysis summary
   - Recommended narrative structure
   - Section-by-section writing templates (Abstract, Introduction, Related Work, Method, Experiments, Conclusion)
   - Concrete writing techniques extracted from top papers
   - **Mandatory guardrails** (anti-AI-speak, anti-overclaim, data grounding, consistency, honesty)
6. **Registers the skill** in your project's `CLAUDE.md` and `AGENTS.md` so both agents automatically load and follow it

## Quick Start

### 1. Install

Copy the skill to your project's skill directory:

```bash
# For Claude Code (project-level)
cp -r paper-style-scout/ /path/to/your/project/.claude/skills/paper-style-scout/

# For Codex (project-level)
cp -r paper-style-scout/ /path/to/your/project/.agents/skills/paper-style-scout/

# Or install globally for Claude Code
cp -r paper-style-scout/ ~/.claude/skills/paper-style-scout/

# Or install globally for Codex
cp -r paper-style-scout/ ~/.agents/skills/paper-style-scout/
```

### 2. Run

In Claude Code:

```
/paper-style-scout "your research direction, idea, and claims"
```

Or provide more context:

```
/paper-style-scout "We propose a new attention mechanism for efficient long-context reasoning. Our method reduces memory by 40% while maintaining accuracy on LongBench."
```

### 3. Write

After the skill is generated, use your normal paper writing workflow. Both Claude Code and Codex will automatically load and follow the generated writing skill.

## Guardrails

The generated writing skill includes mandatory rules that the agent cannot bypass:

### Anti-AI-Speak

Banned words: `delve`, `pivotal`, `landscape`, `leveraging`, `comprehensive`, `facilitate`, `underscore`, `testament`, `tapestry`, `myriad`, `paramount`, `intricate`, `elucidate`...

Banned phrases: *"it is worth noting that"*, *"plays a crucial role"*, *"has attracted significant attention"*, *"paves the way for"*...

### Anti-Overclaim

- No "first" / "novel" / "SOTA" without evidence
- "Significantly" requires statistical tests
- Every claim must be scoped to the actual experimental setting

### Data Grounding

- Every number must trace to an experiment output file
- Missing data gets `[TODO]`, never a fabricated number
- No rounding up or beautifying results

### Consistency & Honesty

- One term per concept throughout the paper
- Include a Limitations section
- Report negative results; do not cherry-pick

## Configuration

| Constant | Default | Description |
|----------|---------|-------------|
| `NUM_LATEST_PAPERS` | 3 | Latest papers from arXiv |
| `NUM_CITED_PAPERS` | 2 | Highest-cited papers from Semantic Scholar |
| `CITED_YEAR_RANGE` | 2024-2026 | Year range for high-citation search |
| `CITED_MIN_CITATIONS` | 30 | Minimum citation threshold |

Overrides:

```
/paper-style-scout "topic" - latest: 5
/paper-style-scout "topic" - cited: 3
/paper-style-scout "topic" - min-citations: 100
/paper-style-scout "topic" - year: 2023-2026
```

## Project Structure After Running

```
your-project/
├── CLAUDE.md                              # Updated with paper writing section
├── AGENTS.md                              # Updated with paper writing section
├── .claude/skills/
│   └── paper-writing-guide/
│       └── SKILL.md                       # Generated skill (Claude Code discovers this)
├── .agents/skills/
│   └── paper-writing-guide/
│       └── SKILL.md                       # Same skill (Codex discovers this)
├── papers/
│   └── reference_papers/                  # Downloaded reference PDFs
└── ... (your existing project files)
```

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) or [Codex CLI](https://openai.com/index/codex/)
- Internet access (for arXiv and Semantic Scholar APIs)
- Python 3 (stdlib only — no pip packages needed)

## License

MIT

---

# Paper Style Scout [中文]

一个适用于 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 和 [Codex](https://openai.com/index/codex/) 的独立技能，在写论文之前自动分析你所在领域的顶会论文，生成项目专属的写作技能。

**零依赖。** 不需要外部脚本或其他技能。所有 API 调用都使用 Python 标准库内联完成。

## 功能

给定你的研究方向、idea 和 claims，Paper Style Scout 会：

1. **扫描你的仓库**，理解实际的实验、方法和结果
2. **搜索 3 篇最新相关论文**（arXiv）
3. **搜索 2 篇近两年最高引中稿论文**（Semantic Scholar）
4. **对 5 篇论文做 6 维度分析**：故事线、Abstract 模式、Introduction 结构、方法呈现、实验设计、语言风格
5. **生成项目级写作技能** — 在 `.claude/skills/paper-writing-guide/SKILL.md`（Claude Code 发现路径）和 `.agents/skills/paper-writing-guide/SKILL.md`（Codex 发现路径）各生成一份，包含：
   - 参考论文分析总结
   - 推荐的叙事结构
   - 逐章节写作模板
   - 从顶会论文中提炼的具体写作技巧
   - **强制性约束**（反 AI 口癖、反 overclaim、数据真实性、一致性、诚实性）
6. **注册到项目配置** — 自动更新 `CLAUDE.md` 和 `AGENTS.md`，确保两个 agent 都会加载并遵循该技能

## 快速开始

### 安装

将 skill 复制到项目的 skill 目录：

```bash
# Claude Code（项目级）
cp -r paper-style-scout/ /path/to/your/project/.claude/skills/paper-style-scout/

# Codex（项目级）
cp -r paper-style-scout/ /path/to/your/project/.agents/skills/paper-style-scout/

# 或全局安装（Claude Code）
cp -r paper-style-scout/ ~/.claude/skills/paper-style-scout/

# 或全局安装（Codex）
cp -r paper-style-scout/ ~/.agents/skills/paper-style-scout/
```

### 运行

```
/paper-style-scout "你的研究方向、idea 和 claims"
```

### 写论文

技能生成后，正常使用论文写作流程即可。Claude Code 和 Codex 都会自动加载并遵循生成的写作技能。

## 约束规则

### 反 AI 口癖

禁用词汇：`delve`、`pivotal`、`landscape`、`leveraging`、`comprehensive`、`facilitate`、`underscore`、`testament`、`tapestry`、`myriad`、`paramount`、`intricate`、`elucidate`...

禁用句式：*"it is worth noting that"*、*"plays a crucial role"*、*"has attracted significant attention"*、*"paves the way for"*...

### 反 Overclaim

- 禁止无证据使用 "first" / "novel" / "SOTA"
- "Significantly" 必须有统计检验支持
- 所有 claim 必须限定在实验范围内

### 数据真实性

- 每个数字必须追溯到实验输出文件
- 缺失数据用 `[TODO]` 标记，绝不编造
- 不美化实验结果

### 一致性与诚实

- 同一概念全文使用统一术语
- 包含 Limitations 章节
- 如实报告负面结果，不 cherry-pick

## 配置

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `NUM_LATEST_PAPERS` | 3 | arXiv 搜索的最新论文数 |
| `NUM_CITED_PAPERS` | 2 | Semantic Scholar 搜索的高引论文数 |
| `CITED_YEAR_RANGE` | 2024-2026 | 高引论文年份范围 |
| `CITED_MIN_CITATIONS` | 30 | 最低引用数阈值 |

## 要求

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 或 [Codex CLI](https://openai.com/index/codex/)
- 网络访问（arXiv 和 Semantic Scholar API）
- Python 3（仅标准库，无需 pip 安装）

## License

MIT
