# 项目分析五阶段流程

本文档是项目分析阶段（Phase 2）的详细执行指南。分析当前工作目录下的开源大模型项目：找出所有提示词和工具定义，翻译成中文，梳理 LLM 使用架构，生成分析报告和产品说明书。

> 本技能灵感来源于 howPrompt by @comeonzhj

## 五阶段概览

每完成一个阶段，简要汇报进度（用精确数字，不用"大部分"等模糊词）。

### 第一阶段：项目探索

确定项目根目录（向上查找 `.git/`、`package.json`、`pyproject.toml`、`go.mod`、`Cargo.toml`），找不到则用当前目录并提示用户确认。

快速建立项目认知：
1. 读 README.md 了解定位
2. 读依赖文件，识别 LLM 库（`langchain`、`openai`、`anthropic`、`crewai`、`autogen`、`llamaindex`、`dspy`、`instructor` 等）
3. 扫描目录结构，定位关键目录
4. 如果没有发现任何 LLM 相关依赖或提示词文件——告知用户并终止，不要在非 LLM 项目上硬凑内容
5. 评估项目规模：超过 1000 个文件的项目先扫核心目录（`src/`、`app/`、`lib/`），小项目全量扫描

搜索时排除：`node_modules/`、`vendor/`、`site-packages/`、`.venv/`、`venv/`、`dist/`、`build/`、`__pycache__/`。

### 第二阶段：提示词与工具识别

读取 [../templates/search_patterns.md](../templates/search_patterns.md) 获取 8 种搜索方法。每种方法覆盖不同维度，全部执行才能确保不遗漏。

**大型项目效率策略**：超过 500 个源文件时，用 Agent 子代理并行执行不同类别的搜索，然后合并结果。

区分生产代码和测试代码（`test/`、`tests/`、`__tests__/`、`*_test.*`、`*.spec.*`）。

**追踪引用关系**：一个提示词可能在 A 文件定义、B 文件组装、C 文件调用。发现可疑文件后，追踪导入和引用链。

搜索完成后去重，然后生成**提示词主清单**（MANIFEST）并写入文件：
- ≤100 个提示词：`ai_analysis/translated_prompts/MANIFEST.md`
- \>100 个：按模块拆分为 `MANIFEST_[模块名].md`

MANIFEST 格式见 [verification.md](verification.md)。

### 第二阶段出口 → 第三阶段入口

根据 MANIFEST 中的提示词数量，读取 [scale-strategies.md](scale-strategies.md) 选择执行策略（小型 ≤30 / 中型 31-100 / 大型 101-300 / 超大型 300+）。

### 第三阶段：提示词文档化

读取 [../templates/doc_template.md](../templates/doc_template.md) 获取翻译文档和索引模板。

为每个提示词创建独立的翻译文档（原文 + 中文翻译 + 关键参数 + 上下游链路）。

**并行策略**：超过 10 个提示词时，按功能模块分组，用 Agent 子代理并行生成。每个子代理负责 5-10 个提示词。并行不超过 6 个，超出则分波调度。

**子代理可能失败**。每个子代理完成后用 Glob 验证文件实际写入。失败时按 [fault-handling.md](fault-handling.md) 处理。

**验证门控**：第三阶段结束前，比对 MANIFEST 与实际生成的翻译文件。确认 0 个缺失后才生成 INDEX.md。完整的 6 步验证流程见 [verification.md](verification.md)。

### 第四阶段：项目分析报告

读取 [../templates/report_template.md](../templates/report_template.md) 获取 8 章报告模板。生成 `ai_analysis/AI_MODEL_USAGE_ANALYSIS.md`。

报告面向技术读者，所有结论必须有代码位置支撑。如果某个章节不适用，写明"该项目未涉及"而不是跳过。

**重要**：报告第 2 章和第 4 章会包含 Mermaid 图表（架构图、时序图、链路图），这些图表在 Phase 3 归档时会被自动转换为飞书白板。

### 第五阶段：产品说明书

读取 [../templates/guide_template.md](../templates/guide_template.md) 获取 12 章说明书模板。生成 `ai_analysis/PRODUCT_GUIDE.md`。

目标读者是没有编程经验的人。写作时站在"想用这个项目但不知道从哪开始"的用户视角。

## 输出目录

```
ai_analysis/
├── translated_prompts/
│   ├── MANIFEST*.md        ← 提示词主清单
│   ├── INDEX.md            ← 翻译文档索引
│   └── [各翻译文档].md
├── AI_MODEL_USAGE_ANALYSIS.md
├── PRODUCT_GUIDE.md
└── ERRORS.md               ← 异常日志（有异常时才创建）
```

## 执行原则

### 上下文节约

主 agent 的上下文窗口是最宝贵的资源：
- 搜索阶段只记录文件路径列表，不 Read 内容
- 子代理 prompt 中给出模板路径让子代理自行 Read
- 验证阶段只比对文件名列表
- 汇报时使用简洁的统计数字和表格

### 翻译质量

- 保持技术术语准确（"chain of thought" → "思维链"）
- 占位符保持原样（`{variable}`、`{{template}}`）
- 保留原始格式

### 诚实报告

用精确数字（"已完成 67/72 个"），如有异常引导用户查看 ERRORS.md。

## 断点续传

发现 `ai_analysis/` 已有部分产出时不从头开始：

| 检查文件 | 说明 | 动作 |
|---------|------|------|
| `MANIFEST*.md` | 第二阶段已完成 | 跳过搜索，读取 MANIFEST |
| 翻译文档但无 INDEX.md | 第三阶段部分完成 | 从验证步骤开始 |
| `INDEX.md` | 第三阶段已完成 | 跳过翻译 |
| `AI_MODEL_USAGE_ANALYSIS.md` | 第四阶段已完成 | 跳过报告 |
| `PRODUCT_GUIDE.md` | 第五阶段已完成 | 跳过说明书 |

## 特殊情况

- **中文提示词**：标注位置，跳过翻译
- **加密/混淆内容**：标注 [无法解析]
- **动态生成的提示词**：追踪函数调用链，用 `{动态部分}` 标记
- **外部托管**：标注 [内容托管于外部平台: 平台名称]
- **不确定的内容**：标注 [待确认] 继续推进
