---
name: lark-project-archive
version: 1.0.1
description: GitHub 项目拆解分析与飞书知识库归档（Analyze GitHub projects and archive to Feishu/Lark wiki）。触发条件——必须同时满足两点：(1) 用户提供了 GitHub 项目地址（URL 或 owner/repo 简写），(2) 用户要求将结果归档/保存到飞书（Feishu/Lark）知识库或 wiki。两个条件缺一不可：仅有 GitHub 项目分析需求但未提及飞书/知识库/wiki 归档的 → 不使用此技能（应使用 study_project 等分析类技能）；仅有飞书操作但无 GitHub 项目分析的 → 不使用此技能（应使用 lark-wiki/lark-doc 等飞书技能）；目标平台是 Notion/Confluence 等非飞书平台的 → 不使用此技能。自动克隆项目、系统化分析提取提示词与架构、将分析报告归档到飞书知识库（含 Mermaid 图表自动转飞书白板可视化）、清理本地文件。典型触发："帮我拆解这个 AI 项目并归档到飞书"、"分析这个 repo 存到知识库"、"学习这个开源项目并保存报告到飞书wiki"、"clone 这个项目分析下然后存飞书"、"analyze this repo and archive to lark wiki"、"study this project and save to feishu"。
allowed-tools: Read Glob Grep Bash Edit Write Agent
metadata:
  requires:
    bins: ["lark-cli", "git", "node"]
---

# GitHub 项目分析与飞书知识库归档

克隆 GitHub 项目 → 系统化分析提取提示词与架构 → 归档到飞书知识库（Mermaid 图表自动转白板） → 清理本地文件。

## 工作流总览

```
Phase 0: 前置检查 → git / lark-cli / Node.js → 统一授权 → 默认知识库设置
    ↓
Phase 1: 克隆项目 → git clone 到临时目录
    ↓
Phase 2: 项目分析 → 五阶段分析（探索/搜索/翻译/报告/说明书）
    ↓  生成 ai_analysis/ 目录
Phase 3: 飞书归档 → 确定知识库 → 创建节点 → Mermaid→白板 → 上传内容
    ↓
Phase 4: 清理 → 保留分析结果 → 删除克隆代码 → 最终报告
```

## Phase 0: 启动检查

每次执行技能前，按以下顺序检查环境状态，**哪一步不通过就停在哪一步处理，通过后继续往下**：

```
Step A: git --version && node --version
         │
         ├─ 都成功 → 进入 Step B
         └─ 失败 → 提示安装缺失工具（Git / Node.js 18+），等用户修复后重试

Step B: lark-cli config show
         │
         ├─ 成功（有 appId）→ 进入 Step C
         └─ 失败（"no apps"）→ 需要首次配置，执行 Step B1
                                 │
                                 ▼
                   Step B1: lark-cli config init --new（background 执行）
                            → 启动后立即读取输出，提取配置链接发给用户
                            → 等待 background 任务完成通知（不要让用户手动确认）
                            → 收到完成通知后自动继续 Step C

Step C: 统一授权（一次性完成飞书授权 + 执行权限，避免后续中断）
         │
         ┌─ C1: 检查飞书用户授权
         │   lark-cli auth status
         │   ├─ identity=user → 跳过 C2，进入 C3
         │   └─ identity=bot（"No user logged in"）→ 执行 C2
         │
         ├─ C2: lark-cli auth login --domain wiki,docs,drive（background 执行）
         │       → 启动后立即读取输出，提取授权链接发给用户
         │       → 等待 background 任务完成通知
         │       → 收到完成通知后继续 C3
         │       ⚠ 不要使用 --scope 参数（会报 "invalid or malformed scopes"），
         │         必须使用 --domain 按域授权
         │
         └─ C3: 检查命令执行权限（仅 Claude Code 环境）
                 如果 ~/.claude/settings.json 存在：
                 检查 permissions.allow 是否包含
                 "Bash(lark-cli *)"、"Bash(npx *)"、"Bash(git *)"
                 ├─ 已包含 → 进入 Step D
                 └─ 未包含 → 提示用户：
                             "本技能需要调用 lark-cli / npx / git，建议加入白名单。"
                             → 用户同意后追加到 permissions.allow → 进入 Step D
                 如果不是 Claude Code → 直接进入 Step D

Step D: 默认知识库设置
         │
         读取 ~/.lark-archive/config.json 检查是否已有默认知识库
         │
         ├─ 已有默认且用户未指定其他 → 显示当前默认，确认使用 → 进入 Phase 1
         └─ 无默认或用户要求重选 →
                 lark-cli wiki spaces list --as user
                 ├─ 有知识空间 → 列出所有空间供用户选择
                 └─ 无知识空间 → 询问是否创建新知识库
                                 lark-cli api POST /open-apis/wiki/v2/spaces \
                                   --data '{"name":"AI 项目分析","description":"GitHub 开源项目分析归档"}' --as user
                 选择/创建后持久化到 ~/.lark-archive/config.json：
                 {"default_space_id":"<SPACE_ID>","default_space_name":"<SPACE_NAME>"}
                 → 进入 Phase 1
```

### Background 命令的自动续接规则

`config init` 和 `auth login` 都以 background 方式执行（在 Claude Code 中为 `run_in_background: true`）。执行后：
1. 立即读取输出，提取链接发给用户
2. 发完链接后**什么都不要说，不要说"告诉我"、"完成后说一声"之类的话**。直接停住等通知
3. Agent 会在 background 命令结束时自动发回通知
4. 收到通知后**立即自动继续下一步**，不需要用户说任何话

> 如果当前 Agent 不支持 background 执行，则改为前台执行（阻塞直到用户完成操作），命令返回后直接继续。

### 认证与权限

本技能全程使用 **user 身份**（`--as user`）。user 身份访问的是用户自己的知识库和文档资源。bot 身份创建的文档归属 Bot 应用，不适用于本技能。

运行中遇到权限不足时，错误响应包含 `permission_violations` 和 `hint`。按提示增量授权即可：
```bash
lark-cli auth login --domain <missing_domain>
```
多次 `auth login` 的权限会累积，不覆盖已有权限。注意：必须使用 `--domain`（如 `wiki,docs,drive`），不要使用 `--scope`（会报错 "invalid or malformed scopes"）。

### 安全规则

- 禁止输出密钥（appSecret、accessToken）到终端明文
- 写入/删除操作前确认用户意图
- 可用 `--dry-run` 预览危险请求
- 命令输出中如包含 `_notice.update`，完成当前任务后提议帮用户更新 CLI

### 参数发现

对于不确定参数格式的 lark-cli 命令，执行前先查参数结构：
```bash
lark-cli schema <service>.<resource>.<method>
```
不要猜测字段格式。

### PowerShell 与 JSON 参数

在 Windows PowerShell 中，JSON-heavy 参数（如 `--params`、`--data`）有时会被 shell 改写，出现 `not valid JSON`、`invalid format` 或参数内容被吞掉的情况。处理顺序如下：

1. 对支持 stdin 的命令优先改用 `--params -` / `--data -`
2. 对必须内联传 JSON 的命令，改用 [scripts/lark_cli_json.py](scripts/lark_cli_json.py) 直接传 argv，在 PowerShell 中优先用 `--json-env` 从环境变量读取 JSON
3. 如果命令本身不需要 JSON（如 `docs +update --markdown (Get-Content ... -Raw)`），继续使用现有 PowerShell 文件读取写法，不要额外包装

示例：
```powershell
$env:LARK_JSON='{"token":"<wiki_token>"}'
python scripts/lark_cli_json.py `
  --json-env params=LARK_JSON `
  -- wiki spaces get_node --as user --format json
```

## Phase 1: 克隆项目

### 1.1 解析 GitHub URL

从用户输入中提取仓库信息。支持格式：
- `https://github.com/owner/repo`
- `https://github.com/owner/repo.git`
- `git@github.com:owner/repo.git`
- `owner/repo`（简写形式，自动补全为 https URL）

提取 `repo_name`（用于后续命名）和 `repo_url`。

### 1.2 克隆到用户当前工作目录

**克隆前冲突检查**：如果 `./<repo_name>` 目录已存在：
1. 检查是否包含 `.git/` — 如果是完整克隆，提示用户选择：复用现有目录或删除后重新克隆
2. 如果是残留目录（无 `.git/` 或目录为空）— 尝试 `rm -rf ./<repo_name>`
3. 删除失败（Windows 文件锁等）— 改用 `./<repo_name>_<timestamp>` 作为克隆目标

```bash
git clone --depth 1 <repo_url> ./<repo_name>
```

使用 `--depth 1` 浅克隆加速。如果克隆失败（私有仓库等），提示用户检查访问权限。

### 1.3 确认项目根目录

进入克隆目录，确认存在项目标志文件（`.git/`、`package.json`、`pyproject.toml`、`go.mod`、`Cargo.toml` 等）。

## Phase 2: 项目分析

本阶段对项目进行系统化分析，产出翻译文档集、技术分析报告和产品说明书。完整的五阶段流程、模板和执行策略详见参考文档。

**读取顺序**：
1. 先读 [references/analysis-workflow.md](references/analysis-workflow.md) 了解五阶段流程
2. 执行第二阶段时读 [templates/search_patterns.md](templates/search_patterns.md) 获取 8 种搜索方法
3. 第二阶段完成后读 [references/scale-strategies.md](references/scale-strategies.md) 选择执行策略
4. 第三阶段读 [templates/doc_template.md](templates/doc_template.md) 获取翻译模板
5. 第三阶段验证时读 [references/verification.md](references/verification.md)
6. 第四阶段读 [templates/report_template.md](templates/report_template.md) 获取报告模板
7. 第五阶段读 [templates/guide_template.md](templates/guide_template.md) 获取说明书模板
8. 遇到故障时读 [references/fault-handling.md](references/fault-handling.md)

### 五阶段概要

| 阶段 | 产出 | 关键动作 |
|------|------|----------|
| 1. 项目探索 | 项目概况 | 读 README、识别 LLM 依赖、扫描目录 |
| 2. 提示词搜索 | MANIFEST.md | 8 种方法全覆盖搜索，生成主清单 |
| 3. 提示词文档化 | translated_prompts/*.md + INDEX.md | 逐个翻译，并行加速，验证完整性 |
| 4. 分析报告 | AI_MODEL_USAGE_ANALYSIS.md | 8 章技术分析，含 Mermaid 架构图 |
| 5. 产品说明书 | PRODUCT_GUIDE.md | 12 章用户友好说明书 |

**终止条件**：如果第一阶段未发现任何 LLM 相关依赖或提示词，告知用户并终止，跳过后续所有阶段（包括归档）。

### 分析产出目录

```
<repo_name>/ai_analysis/
├── translated_prompts/
│   ├── MANIFEST*.md
│   ├── INDEX.md
│   └── [各翻译文档].md
├── AI_MODEL_USAGE_ANALYSIS.md
├── PRODUCT_GUIDE.md
└── ERRORS.md（有异常时才创建）
```

## Phase 3: 飞书知识库归档

将分析产出归档到飞书知识库，自动将 Mermaid 图表转为飞书白板嵌入文档。

### 3.1 确定归档位置

**步骤 1：使用 Phase 0 Step D 中设置的默认知识库**

读取 `~/.lark-archive/config.json` 中的 `default_space_id`：
- 如果用户在本次运行中**明确指定**了其他知识库 → 使用指定的 `space_id`
- 否则 → 直接使用默认知识库，**无需再向用户确认**

> Phase 0 已完成知识库选择并持久化。除非用户明确要求更换，此步骤应零交互。

**步骤 2：定位 `student` 容器节点**

所有项目归档统一挂在名为 `student` 的容器节点下。先查找是否已存在，不存在则新建：

```bash
# 查找空间根目录下的子节点
lark-cli wiki nodes list --params '{"space_id":"<SPACE_ID>"}' --as user
```

在返回的节点列表中查找 `title == "student"` 的节点：
- **找到** → 提取其 `node_token` → `STUDENT_NODE_TOKEN`
- **未找到** → 创建：
  ```bash
  lark-cli wiki +node-create \
    --space-id <SPACE_ID> \
    --title "student" \
    --as user
  ```
  保存返回的 `node_token` → `STUDENT_NODE_TOKEN`

> 如果用户额外指定了父节点（通过 URL 或 token），则在该父节点下查找/创建 `student` 节点，而不是在空间根目录下。Wiki URL 格式 `https://xxx.feishu.cn/wiki/<token>` 需先解析：
> ```bash
> lark-cli wiki spaces get_node --params '{"token":"<wiki_token>"}' --as user
> ```
> 从返回中提取 `node.node_token`，在该节点下执行上述查找/创建逻辑。

### 3.2 提取项目类型

从分析报告 `AI_MODEL_USAGE_ANALYSIS.md` 中自动提取项目类型。读取报告的"第 8 章：关键发现与总结"部分，根据"核心使用模式"字段判断：

| 报告中的模式 | 项目类型标签 |
|-------------|------------|
| Agent | AI Agent |
| RAG | RAG 框架 |
| 嵌入型 | LLM 应用 |
| 混合 | AI 平台 |
| 多 Agent | 多智能体框架 |
| 工具/CLI | AI 工具 |

如果无法自动判断，使用 "AI 项目" 作为默认值。

### 3.3 创建节点层级

按以下顺序创建，每一步保存返回的 token 供后续使用：

**步骤 1：在 `student` 节点下创建项目根节点**
```bash
lark-cli wiki +node-create \
  --parent-node-token <STUDENT_NODE_TOKEN> \
  --title "[项目类型]-[repo_name]" \
  --as user
```

保存返回的 `node_token` → `ROOT_NODE_TOKEN`，`obj_token` → `ROOT_OBJ_TOKEN`。

**步骤 2：创建「技术分析报告」子节点**
```bash
lark-cli wiki +node-create \
  --parent-node-token <ROOT_NODE_TOKEN> \
  --title "技术分析报告" \
  --as user
```
保存 `node_token` → `REPORT_NODE_TOKEN`，`obj_token` → `REPORT_OBJ_TOKEN`。

**步骤 3：创建「产品说明书」子节点**
```bash
lark-cli wiki +node-create \
  --parent-node-token <ROOT_NODE_TOKEN> \
  --title "产品说明书" \
  --as user
```
保存 `obj_token` → `GUIDE_OBJ_TOKEN`。

**步骤 4：为每个翻译文档创建子节点**

读取 `ai_analysis/translated_prompts/INDEX.md`，为每个翻译文档创建节点（作为技术分析报告的子节点）：

```bash
# 对每个翻译文档执行
lark-cli wiki +node-create \
  --parent-node-token <REPORT_NODE_TOKEN> \
  --title "[翻译文档标题]" \
  --as user
```

保存每个文档的 `obj_token` 用于后续内容上传。

> **效率提示**：翻译文档较少（≤10 个）时逐个创建即可。较多时（10+），按以下策略优化：
> - 逐个执行、每次保存 token（不要并行创建，避免顺序混乱）
> - 每创建 5 个节点后做一次轻量验证（确认前面的 token 都有效）
> - 如果数量超过 30 个，考虑将翻译文档合并为按模块分组的文档（减少节点数量），在合并文档内用标题区分各提示词

### 3.4 Mermaid 图表 → 飞书白板

分析报告中包含 Mermaid 图表（架构图、序列图、链路图等），归档时自动转为飞书白板嵌入文档。

详细的 Mermaid 渲染和上传流程见 [references/mermaid-rendering.md](references/mermaid-rendering.md)。

**核心流程**：

1. **提取 Mermaid 代码块**：使用 `node -e "..."` 读取 markdown 文件，用正则提取所有 Mermaid 代码块（不要使用 `python3`，Windows 上可能不可用）
2. **Mermaid 语法清洗**（渲染前必做）：
   - 将全角中文标点替换为 ASCII 等价物：`（→(`、`）→)`、`【→[`、`】→]`、`，→,`、`：→:`
   - 对包含中文或特殊字符的 subgraph 名称加引号：`subgraph 增量提取流程` → `subgraph id["增量提取流程"]`
   - 对包含中文的节点标签确保使用引号：`A[增量提取]` → `A["增量提取"]`
   - 验证修改后的语法：`npx -y @mermaid-js/mermaid-cli mmdc -i <file> -o /dev/null` 快速检查
3. **替换为白板占位符**：将每个 Mermaid 代码块替换为 `<whiteboard type="blank"></whiteboard>`
4. **保存修改后的 markdown** 到临时文件（使用 `${TMPDIR:-/tmp}` 作为临时目录，Windows 上需先 `mkdir -p "${TMPDIR:-/tmp}"`）
5. **上传到文档**：
   ```bash
   lark-cli docs +update --doc <OBJ_TOKEN> --mode overwrite \
     --markdown "$(cat "${TMPDIR:-/tmp}/modified_content.md")" --as user
   ```
6. **获取白板 token**：从返回的 `data.board_tokens` 数组中获取（顺序与占位符对应）
7. **渲染并上传每个白板**：
   ```bash
   # 对每个 (board_token, mermaid_code) 对
   cat > "${TMPDIR:-/tmp}/diagram.mmd" << 'MERMAID_EOF'
   <sanitized_mermaid_code>
   MERMAID_EOF
   npx -y @larksuite/whiteboard-cli@^0.2.0 --to openapi \
     -i "${TMPDIR:-/tmp}/diagram.mmd" --format json | \
     lark-cli whiteboard +update \
       --whiteboard-token <BOARD_TOKEN> \
       --source - --yes --as user
   ```

**注意事项**：
- 如果 markdown 内容太长无法通过命令行传递，将内容写入临时文件后使用文件路径
- 白板上传后无法修改，确保 Mermaid 代码**经过语法清洗**后再渲染
- 如果 Mermaid 渲染失败（语法错误等），保留原始代码块作为文本，在 ERRORS.md 中记录
- `/tmp` 路径在 Windows 上不存在，始终使用 `${TMPDIR:-/tmp}` 并在首次使用前 `mkdir -p`

### 3.5 上传文档内容

**技术分析报告**：执行 3.4 中的 Mermaid 处理流程（报告中包含架构图等）。

**产品说明书**：
```bash
lark-cli docs +update --doc <GUIDE_OBJ_TOKEN> --mode overwrite \
  --markdown "$(cat ai_analysis/PRODUCT_GUIDE.md)" --as user
```
> 说明书中一般不含 Mermaid 图表，如果有也同样执行 3.4 的流程。

**翻译文档**：对每个翻译文档：
```bash
lark-cli docs +update --doc <PROMPT_OBJ_TOKEN> --mode overwrite \
  --markdown "$(cat ai_analysis/translated_prompts/<filename>.md)" --as user
```

**大文件处理**：上传前检查文件大小（`wc -c < file`）。超过 **30KB** 的文件会触发 Windows 上的 `Argument list too long` 错误，需要自动分割处理：
1. 将文件按行数均分为 ≤30KB 的分段（在段落或标题边界切割，避免截断内容）
2. 第一段用 `--mode overwrite` 上传
3. 后续每段用 `--mode append` 追加
4. 所有临时文件使用 `${TMPDIR:-/tmp}` 路径

详见 [references/doc-whiteboard.md](references/doc-whiteboard.md) 的大文件处理章节。

**根节点文档**：为根节点写入简介内容（项目概览 + 子节点导航）：
```markdown
# [项目类型]-[项目名]

## 项目概述
[从分析报告第 1 章提取的项目概述]

## 文档导航
- **技术分析报告**：面向开发者和架构师的深度技术分析
- **产品说明书**：面向非技术用户的使用指南
- **提示词翻译文档**：项目中所有提示词的中英对照翻译（共 N 个）

## 分析信息
- 分析日期: [当前日期]
- 项目来源: [GitHub URL]
- 核心模式: [项目类型]
```

**必须先落盘再上传**：将上述内容保存为 `ai_analysis/ROOT_OVERVIEW.md`，然后从文件读取上传，**禁止**直接拼接内联字符串（例如 `--markdown "# 标题\n\n## 项目概述..."`）。在 Bash / zsh 中使用：
```bash
lark-cli docs +update --doc <ROOT_OBJ_TOKEN> --mode overwrite \
  --markdown "$(cat ai_analysis/ROOT_OVERVIEW.md)" --as user
```

在 PowerShell 中使用：
```powershell
lark-cli docs +update --doc <ROOT_OBJ_TOKEN> --mode overwrite `
  --markdown (Get-Content ai_analysis/ROOT_OVERVIEW.md -Raw) --as user
```

> 原因：PowerShell 和 Bash 都不会把普通字符串里的字面量 `\n` 自动转换成换行。若先把 Markdown 序列化成单行字符串再上传，飞书会把 `\n` 当正文文本显示，导致首页渲染异常。

### 3.6 验证归档结果

归档完成后，验证所有文档已成功创建：

1. 列出根节点下的子节点确认结构正确：
   ```bash
   lark-cli wiki nodes list --params '{"space_id":"<SPACE_ID>","parent_node_token":"<ROOT_NODE_TOKEN>"}' --as user
   ```
2. 检查关键文档内容非空，且首页未出现转义上传：
   ```bash
   lark-cli docs +fetch --doc <ROOT_OBJ_TOKEN> --as user | head -20
   lark-cli docs +fetch --doc <REPORT_OBJ_TOKEN> --as user | head -20
   ```
   如果根节点首页内容中出现字面量 `\n`、`\t`，或标题/小节没有真实换行，判定为“Markdown 被转义后上传”。此时必须重新生成 `ai_analysis/ROOT_OVERVIEW.md`，并从文件重新执行一次 `overwrite` 上传。
3. 向用户报告归档结果：
   - 知识空间名称和链接
   - 创建的节点数量
   - 白板转换数量（成功/失败）
   - 如有失败项，列出详情

## Phase 4: 清理

### 4.1 保留分析结果

将 `ai_analysis/` 目录从克隆项目中移出到用户当前工作目录，并重命名：

```bash
mv ./<repo_name>/ai_analysis ./<repo_name>-ai_analysis
```

> 保存在用户当前工作目录（clone 项目的同级目录）。方便用户查阅本地副本，同时不污染项目原始结构。

### 4.2 删除克隆项目

```bash
rm -rf ./<repo_name>
```

> 执行前向用户确认，避免误删用户自己的工作。

**Windows 文件锁处理**：`rm -rf` 在 Windows 上可能因 `.git/` 中的文件锁而失败。处理策略：
1. 首次失败后等待 3 秒重试一次
2. 仍然失败 → 告知用户手动关闭占用该目录的进程（如 IDE、文件管理器），然后重试
3. 用户确认后再次尝试删除。如仍无法删除，记录路径提示用户稍后手动清理

### 4.3 最终报告

向用户汇报完整执行结果：

```
✅ 项目分析与归档完成

📊 分析结果:
- 发现 X 个提示词，Y 个工具定义
- 核心模式: [项目类型]

📁 飞书知识库:
- 空间: [空间名称]
- 根节点: [项目类型]-[repo_name]
  - 技术分析报告 (含 N 个白板图表)
  - 产品说明书
  - M 个提示词翻译文档

💾 本地副本: ./<repo_name>-ai_analysis/

⚠️ 异常 (如有): [异常摘要，引导查看 ERRORS.md]
```

## 权限总表

### 归档阶段（写入）

| 操作 | 命令 | scope |
|------|------|-------|
| 列出知识空间 | `wiki spaces list` | `wiki:space:retrieve` |
| 查询空间信息 | `wiki spaces get` | `wiki:space:read` |
| 解析 Wiki URL | `wiki spaces get_node` | `wiki:node:read` |
| 创建知识库节点 | `wiki +node-create` | `wiki:node:create` |
| 列出子节点 | `wiki nodes list` | `wiki:node:retrieve` |
| 写入文档内容 | `docs +update` | `docx:document:write_only` |
| 获取文档内容 | `docs +fetch` | `docx:document:readonly` |
| 更新白板内容 | `whiteboard +update` | `drive:drive:write` |
| 查询白板 | `whiteboard +query` | `drive:drive:read` |

### 初始化授权命令

```bash
lark-cli auth login --domain wiki,docs,drive
```

> ⚠ 不要使用 `--scope` 参数列举具体权限点，会报 "invalid or malformed scopes" 错误。使用 `--domain` 按域授权，CLI 会自动关联该域下所有必要的权限点。

---

## 参考文档索引

### 飞书操作
| 文档 | 用途 | 何时读取 |
|------|------|----------|
| [references/lark-cli-setup.md](references/lark-cli-setup.md) | lark-cli 配置、认证、权限 | Phase 0 前置检查 |
| [references/wiki-archive.md](references/wiki-archive.md) | 知识库空间和节点操作 | Phase 3 归档 |
| [references/doc-whiteboard.md](references/doc-whiteboard.md) | 文档创建更新和白板集成 | Phase 3 内容上传 |
| [references/mermaid-rendering.md](references/mermaid-rendering.md) | Mermaid → 飞书白板完整流程 | Phase 3.4 |

### 项目分析
| 文档 | 用途 | 何时读取 |
|------|------|----------|
| [references/analysis-workflow.md](references/analysis-workflow.md) | 五阶段分析完整流程 | Phase 2 开始时 |
| [references/scale-strategies.md](references/scale-strategies.md) | 按规模分级的执行策略 | Phase 2 第二阶段完成后 |
| [references/verification.md](references/verification.md) | MANIFEST 格式和验证流程 | Phase 2 第三阶段验证时 |
| [references/fault-handling.md](references/fault-handling.md) | 故障分类和降级交付 | 遇到故障时 |

### 模板
| 文档 | 用途 | 何时读取 |
|------|------|----------|
| [templates/search_patterns.md](templates/search_patterns.md) | 8 种搜索方法 | Phase 2 第二阶段 |
| [templates/doc_template.md](templates/doc_template.md) | 翻译文档模板 | Phase 2 第三阶段 |
| [templates/report_template.md](templates/report_template.md) | 分析报告模板 | Phase 2 第四阶段 |
| [templates/guide_template.md](templates/guide_template.md) | 产品说明书模板 | Phase 2 第五阶段 |
