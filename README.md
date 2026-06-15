# DualGap 使用说明

## 1. Skill 介绍

DualGap （ **dual-domain research gap analysis**）是一个用于阅读分析两个不同研究方向的论文，得到reashech ideas的Skill；

主要是用来找交叉方向的Ideas 。例如回答：方向A和方向B相结合有什么挑战没有解决？有什么独特的挑战？这一类问题

**skills输入**：两个文件夹，每个文件夹中存放的是一个方向对应的论文pdf。****
**skills输出**：每篇论文的逐篇深度笔记、独立质量审查记录、方向内总结报告、跨方向对比报告、research gaps（挑战/待解决）报告、ranked improvement ideas （调研得到的Ideas）。

整个工作流基于LLM API实现，避免了因上下文爆炸导致的调研失败问题；

clawhub链接：
https://clawhub.ai/zza234s/dualgap

## 2. 环境准备

### 2.1 安装 Skill 到 Codex （或其他Harness）

将当前DualGap文件夹复制到skills目录：

安装后重启Harness，使新skill被加载。重启后可以用下面的触发名调用：

```text
$dualgap
```

### 2.2 依赖安装

#### (1) 进入 skill 目录：

```powershell
cd <repo-root>\skills\dualgap
python -m pip install -r requirements.txt
```

#### (2) 准备论文 PDF

将两个研究方向的论文分别放到两个文件夹中，例如：

```text
<workspace>\papers\GNN
<workspace>\papers\FL
```

每个目录中放对应方向的 PDF 文件。

#### (3) 准备API Key环境文件

该skills依赖于大模型API key调用 （推荐使用Qwen）

需要您创建一个本地env文件，例如命名为：

```text
qwen.env
```

文件内容示例：

```env
QWEN_API_KEY=your_api_key_here
QWEN_BASE_URL=https://your-openai-compatible-endpoint/v1
QWEN_MODEL=your_model_name
```

其中 `Env prefix: QWEN` 表示程序会读取 `QWEN_API_KEY`、`QWEN_BASE_URL` 和 `QWEN_MODEL` 这一组变量。不要把真实 API key 提交到仓库。



## 3. Quick Start

### 以Prompt方式调用 Skills

- 下面以分析GNN和FL两个方向的论文作为示例
- 在实际使用时请替换下面示例中的 **路径/目录** 信息

在 Codex 中输入：

```text
Use $dualgap.

LLM API env file:
<workspace>\config\qwen.env

Env prefix:
QWEN

Direction A PDF directory:
<workspace>\papers\GNN

Direction A name:
Graph Neural Networks

Direction B PDF directory:
<workspace>\papers\FL

Direction B name:
Federated Learning

Output directory:
<workspace>\outputs\gnn_vs_fl

Agenda:
Please run dual-domain research gap analysis. Focus on how Graph Neural Networks and Federated Learning can be combined, what technical gaps remain, what mechanisms could transfer across domains, and how to validate the ideas. Do not focus mainly on privacy, fairness, or poisoning attacks.
```

### 以命令行方式调用

```powershell
python scripts\run_literature_workflow.py `
  --dir-a <workspace>\papers\GNN `
  --dir-b <workspace>\papers\FL `
  --name-a "Graph Neural Networks" `
  --name-b "Federated Learning" `
  --out <workspace>\outputs\gnn_vs_fl `
  --env-file <workspace>\config\qwen.env `
  --env-prefix QWEN `
  --agenda "Find concrete research gaps at the intersection of GNN and FL, with mechanisms, validation plans, scalability, and cost analysis." `
  --batch-size 10 `
  --api-retries 5 `
  --api-timeout 180
```

运行完成后验证输出：

```powershell
python scripts\validate_outputs.py <workspace>\outputs\gnn_vs_fl
```

如果只是试跑，可以增加：

```powershell
--limit-a 1 --limit-b 1 --batch-size 1
```

## 4. 功能介绍、脚本实现和设计原理

DualGap 的核心输出包括：

- 每篇论文的结构化深度笔记
- 每批笔记的独立 LLM reviewer 审查
- 不通过质量门槛时的自动重写
- 两个方向各自的 synthesis
- 跨领域分析
- 目前的research gaps
- ranked improvement ideas (有哪些可以做的研究点)
- audit report 和 workflow manifest

### 核心脚本位置

核心流程由下面的脚本实现：

```text
dualgap/scripts/run_literature_workflow.py
```

它负责：

- 从两个 PDF 目录收集论文
- 使用 `pypdf` 或 `pdftotext` 抽取 PDF 文本
- 调用 OpenAI-compatible API 逐篇生成论文笔记
- 对每篇笔记进行独立 reviewer 审查
- 对不合格笔记自动重写一次
- 基于通过审查的 notes 生成 synthesis、cross-direction analysis、research gaps 和 ranked ideas
- 对 synthesis 文档再次做独立 reviewer 审查
- 写出 audit report 和 workflow manifest

Prompt 设计也集中在这个脚本里：

- `note_prompt(...)`：逐篇论文分析 prompt，位置在 `scripts/run_literature_workflow.py` 的 `note_prompt` 函数。
- `review_prompt(...)`：论文笔记 reviewer prompt，位置在 `review_prompt` 函数。
- `synthesis_prompt(...)`：方向综合、跨方向分析、research gaps、improvement ideas 的生成 prompt，位置在 `synthesis_prompt` 函数。
- `synthesis_review_prompt(...)`：综合文档 reviewer prompt，位置在 `synthesis_review_prompt` 函数。

### 逐篇分析的设计

每篇论文会被抽取为文本后单独送入 API。`note_prompt(...)` 要求模型生成结构化 Markdown 笔记，内容包括：

- 论文基本信息和是否真的属于该方向
- 论文背景和具体动机
- 核心挑战
- 可复现的方法步骤
- 方法为什么可能有效
- 实验和证据边界
- 假设、局限和失败场景
- 对两个研究方向交叉议程的启发
- 需要进一步验证的 open questions

这些结构化笔记为后续 research gap 判断提供可追溯的证据材料。

### Review 机制细节

DualGap 使用独立 LLM 调用审查笔记质量。笔记生成和笔记审查使用不同的 API 请求：

```text
note_prompt(...) -> 生成论文笔记
review_prompt(...) -> 审查论文笔记
```

`review_prompt(...)` 要求 reviewer 只返回 JSON，核心字段包括：

```json
{
  "pass": true,
  "score": 9,
  "failure_reasons": [],
  "rewrite_instructions": [],
  "duplicate_implication": false,
  "needs_pdf_recheck": false,
  "evidence": "short explanation"
}
```

如果出现以下问题，reviewer 必须判为不通过：

- 笔记只是复制或浅层改写论文内容，没有真正理解
- 笔记被截断，或缺少 required sections
- “为什么重要”写得过于泛泛
- 方法描述不足以让读者复现核心流程
- 对研究议程的启发是模板化的，或和前面论文重复
- 缺少假设、局限、失败场景
- 混淆作者证据、模型推断和不确定假设
- 没有明确说明论文和两个研究方向之间的关系
- reviewer 给出了 `rewrite_instructions`

除 LLM reviewer 外，脚本还有确定性的结构检查：`note_structure_errors(...)` 会检查 1-9 节是否存在，以及第 9 节是否疑似截断。任何结构错误都会强制判为不通过。

未通过的笔记会带着 reviewer feedback 自动重写一次，然后再次 review。如果重试后仍不通过，流程会写出 `quality_failure_report.md`，避免把低质量 notes 带入最终 synthesis。

### Synthesis Review

综合文档也会被单独审查：

```text
synthesis_prompt(...) -> 生成 synthesis / research gaps / ideas
synthesis_review_prompt(...) -> 审查综合文档
```

`synthesis_review_prompt(...)` 会检查：

- 文档是否基于 notes 形成具体、可追溯的跨方向分析
- research gap 是否说明“现有论文为什么还没有解决”
- proposed ideas 是否有具体机制、目标瓶颈和验证实验
- 是否遵守用户给定的 agenda 和 exclusions
- 是否区分作者证据、模型推断和不确定假设

如果 synthesis review 不通过，脚本会带着 reviewer feedback 自动重写一次；最终仍不通过则停止并写入质量失败报告。

### 设计原理

DualGap 要求每篇笔记说明论文设置、方法步骤、证据边界、隐含假设、失败场景，以及对目标研究议程的具体启发。笔记生成、笔记审查和 synthesis 审查使用独立 LLM 调用，从而控制模板化内容、幻觉和缺少证据支撑的 gap。

## 5. 更多说明

更完整的使用细节、输出目录结构、smoke test 和验证方式见：

```text
HOW_TO_USE.md
```
