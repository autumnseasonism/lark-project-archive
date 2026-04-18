# 飞书文档与白板操作

本文档包含文档内容写入和白板集成的完整操作指南。

## 文档操作

### 文档 Token 类型

| URL 格式 | Token 类型 | 用法 |
|----------|-----------|------|
| `/docx/doxcnxxx` | file_token | 直接使用 |
| `/doc/doccnxxx` | file_token | 直接使用 |
| `/wiki/wikcnxxx` | wiki_token | ⚠️ 需先解析为 obj_token |
| `/sheets/shtcnxxx` | file_token | 直接使用 |

**Wiki 文档特殊处理**：通过 `wiki +node-create` 创建的文档返回 `obj_token`，可直接用于 `docs +update`。

### 创建文档

```bash
lark-cli docs +create --title "文档标题" --folder-token <FOLDER_TOKEN> --as user
```

> 本技能中不需要单独创建文档——`wiki +node-create` 会自动创建关联的 docx 文档。

### 写入文档内容

```bash
lark-cli docs +update --doc <OBJ_TOKEN> --mode overwrite \
  --markdown "<markdown_content>" --as user
```

**强约束**：只有在内容非常短且确认没有经过 JSON / 字符串转义时，才允许直接传 `"<markdown_content>"`。对模板生成内容、LLM 产出的多段 Markdown、首页导航文档，统一先写入 `.md` 文件，再从文件读取上传；**不要**上传包含字面量 `\n` 的单行字符串。

**mode 参数**：
- `overwrite`：覆盖全部内容（适合首次写入）
- `append`：追加到末尾
- `replace`：替换指定内容

**大文件处理**：如果 markdown 内容太长无法通过命令行传递（超过系统命令行长度限制），将内容写入临时文件：

```bash
# 1. 将内容写入临时文件
# 2. 使用 cat 传递
lark-cli docs +update --doc <OBJ_TOKEN> --mode overwrite \
  --markdown "$(cat /tmp/content.md)" --as user
```

PowerShell 环境等价写法：

```powershell
lark-cli docs +update --doc <OBJ_TOKEN> --mode overwrite `
  --markdown (Get-Content .\content.md -Raw) --as user
```

如果上传后 `docs +fetch` 返回的正文里出现字面量 `\n`、`\t`，说明上传前内容已被错误转义；不要继续 append，直接从原始 `.md` 文件重新 `overwrite`。

如果仍然过长，可以分段上传：先 `overwrite` 第一部分，再 `append` 后续部分。

### 获取文档内容

```bash
lark-cli docs +fetch --doc <OBJ_TOKEN> --as user
```

返回文档的 markdown 内容。可用于验证写入是否成功。

## 白板集成

### 在文档中创建白板

在 markdown 内容中插入白板占位符：

```markdown
这是正文内容

<whiteboard type="blank"></whiteboard>

更多正文内容
```

上传包含占位符的 markdown 后，飞书会自动创建空白白板。

### 获取白板 Token

通过 `docs +update` 上传包含白板占位符的 markdown 后，返回结果中的 `data.board_tokens` 数组包含所有新创建白板的 token。

**Token 顺序**：`board_tokens` 数组中的 token 与文档中白板占位符的出现顺序一一对应。

### 白板内容上传

白板创建后需要单独填充内容（空白白板 ≠ 完成任务）。

**Mermaid 格式上传**：
```bash
npx -y @larksuite/whiteboard-cli@^0.2.0 --to openapi \
  -i /tmp/diagram.mmd --format json | \
  lark-cli whiteboard +update \
    --whiteboard-token <BOARD_TOKEN> \
    --source - --yes --as user
```

**DSL 格式上传**（需要精确控制样式时）：
```bash
npx -y @larksuite/whiteboard-cli@^0.2.0 --to openapi \
  -i /tmp/diagram.json --format json | \
  lark-cli whiteboard +update \
    --whiteboard-token <BOARD_TOKEN> \
    --source - --yes --as user
```

### 白板安全检查

写入已有内容的白板时，**必须先执行 dry-run**：

```bash
npx -y @larksuite/whiteboard-cli@^0.2.0 --to openapi \
  -i /tmp/diagram.mmd --format json | \
  lark-cli whiteboard +update \
    --whiteboard-token <BOARD_TOKEN> \
    --source - --overwrite --dry-run --as user
```

如果日志显示 "XX whiteboard nodes will be deleted"，暂停并让用户确认。

> 本技能创建的是全新白板（通过 `<whiteboard type="blank"></whiteboard>`），所以无需 dry-run——新白板没有现有内容。

### 白板预览

上传后可以导出预览确认：

```bash
lark-cli whiteboard +query --whiteboard-token <BOARD_TOKEN> --output_as image --as user
```

### 完整的文档+白板创建流程

1. 准备 markdown 内容
2. 找到所有 Mermaid 代码块
3. 替换为 `<whiteboard type="blank"></whiteboard>`
4. 记录每个占位符对应的 Mermaid 源码
5. 上传修改后的 markdown → `docs +update`
6. 从返回中提取 `data.board_tokens`
7. 将 board_tokens 与 Mermaid 源码按序配对
8. 逐个渲染并上传白板内容

## 搜索文档

```bash
lark-cli docs +search --query "关键词" --as user
```

可用于搜索知识库中已有的文档，避免重复创建。

## 常见错误

| 错误 | 原因 | 解决方案 |
|------|------|----------|
| `not exist` | Token 错误 | 检查 token 类型；wiki URL 需先解析 |
| `permission denied` | 权限不足 | 检查身份是否有文档权限 |
| `invalid file_type` | 文件类型参数错误 | 使用正确的 obj_type |
