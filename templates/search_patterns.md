# 搜索模式参考

第二阶段使用的 8 种搜索方法。每种方法覆盖不同维度，全部执行才能确保不遗漏。

> **搜索顺序建议**：先执行方法 1（文件名）和方法 3（API 调用），这两个最快且命中率高。然后用方法 2（代码变量）深入扫描。方法 4-8 补充细节。

## 方法 1：文件名模式匹配

搜索以下文件名模式：
- `*prompt*.md` / `*prompt*.txt` / `*prompt*.yaml` / `*prompt*.json`
- `*prompts*.md` / `*prompts*.txt`
- `*template*.md` / `*template*.txt`
- `*instruction*.md` / `*instruction*.txt`
- `*system*.txt` / `*system*.md`
- `*persona*.md` / `*persona*.yaml`

## 方法 2：代码变量搜索

在所有代码文件中搜索以下正则模式：

**通用模式（Python / JS / TS / Go 等）：**
- `\w+[_-]?prompt\s*=` — 匹配 `system_prompt =`、`userPrompt =` 等
- `\w+[_-]?instruction\s*=`
- `PROMPT_\w+\s*=` — 匹配常量定义
- `role["']\s*:\s*["'](system|user|assistant)` — 匹配消息角色定义
- `\w+[_-]?message\s*=.*system` — 匹配消息构建

**Python 框架特征：**
- `PromptTemplate`、`ChatPromptTemplate`、`SystemMessagePromptTemplate`（LangChain）
- `SystemMessage\(`、`HumanMessage\(`、`AIMessage\(`
- `CrewBase`、`@agent`、`@task`、`@crew`（CrewAI）
- `AssistantAgent`、`UserProxyAgent`、`GroupChat`（AutoGen）
- `VectorStoreIndex`、`SimpleDirectoryReader`、`ServiceContext`（LlamaIndex）
- `dspy\.Signature`、`dspy\.Module`、`dspy\.ChainOfThought`（DSPy）
- `instructor\.patch`、`instructor\.from_openai`（Instructor）
- `@server\.call_tool`、`@server\.list_tools`、`McpServer`（MCP Server SDK）

**JavaScript / TypeScript 特征：**
- `const\s+\w*[Pp]rompt\s*=`
- `new OpenAI\(`、`new Anthropic\(`
- `createChat`、`useChat`、`useCompletion`（Vercel AI SDK）
- `generateText`、`streamText`、`generateObject`（Vercel AI SDK v3+）
- 模板字符串中的提示词（搜索包含 `You are`、`Act as`、`Your task` 的模板字面量）
- `server\.tool\(`、`server\.prompt\(`（MCP TypeScript SDK）

## 方法 3：API 调用特征搜索

搜索常见的大模型 API 调用关键词：
- `openai.ChatCompletion`、`chat.completions`、`client.chat`
- `anthropic.messages`、`messages.create`、`client.messages`
- `generate\(`、`prompt=`、`system=`、`messages=\[`
- `ollama`、`llama_cpp`、`vllm`（本地模型调用）
- `litellm`、`completion\(`（多模型统一调用）
- `google.generativeai`、`genai`（Gemini）

> 工具相关的搜索模式见方法 8（工具 Schema 提取）。

## 方法 4：配置文件检查

检查以下配置文件：
- `config.yaml` / `config.yml` / `config.json`
- `.env.example` / `.env.template`（注意：`.env` 可能含敏感信息，仅记录结构不提取值）
- `settings.py`
- `pyproject.toml`（可能含模型配置）
- `docker-compose.yml`（环境变量中可能含模型 endpoint 配置）
- 框架专属配置：`agents.yaml`（CrewAI）、`OAI_CONFIG_LIST`（AutoGen）
- `claude_desktop_config.json`、`mcp.json`（MCP 配置）

## 方法 5：RAG 相关搜索

搜索检索增强生成相关关键词：
- `embedding`、`vectorstore`、`vector_store`
- `retriever`、`similarity_search`、`semantic_search`
- `chromadb`、`pinecone`、`faiss`、`weaviate`、`qdrant`、`milvus`、`pgvector`
- `chunk`、`splitter`、`RecursiveCharacterTextSplitter`
- `VectorStoreIndex`、`SimpleDirectoryReader`（LlamaIndex）
- `knowledge`、`knowledge_source`（CrewAI Knowledge）

## 方法 6：模型调用参数搜索

搜索模型调用参数，记录每个调用点的配置：
- `temperature\s*=`、`max_tokens\s*=`、`top_p\s*=`
- `model\s*=\s*["']`、`model_name\s*=`
- `stop\s*=`、`stream\s*=`
- `response_format`、`json_mode`（结构化输出）
- `tool_choice`、`function_call`（工具调用控制）

## 方法 7：Few-shot 示例搜索

搜索提示词中内嵌的示例模式：
- `Example:`、`Examples:`、`Few-shot`
- `Input:.*Output:`、`Q:.*A:`
- `<example>`、`</example>`
- `Human:.*Assistant:`
- `<user>.*</user>`、`<assistant>.*</assistant>`

## 方法 8：工具 Schema 提取

独立于提示词提取，专门搜索为大模型定义的工具：
1. 搜索 API 请求中的 `tools` 参数定义，提取完整的 JSON Schema
2. 搜索 `@tool` 装饰器、`StructuredTool.from_function`、`BaseTool` 子类
3. 搜索 MCP 工具定义：`server.tool(`、`@server.call_tool`、`inputSchema`
4. 搜索 `function_declarations`（Gemini）、`tools=[{`（通用）
5. 搜索提示词中以自然语言描述的可用工具列表
6. 对每个工具记录：工具名、description（如为英文翻译成中文）、参数 Schema、调用场景
