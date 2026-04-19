# 飞书知识库操作

本文档包含知识空间管理、节点创建和层级组织的完整操作指南。

## PowerShell 与 JSON 参数

本文件中的很多命令都带 `--params` / `--data` JSON。在 Windows PowerShell 中，如果直接粘贴后出现 `not valid JSON`、`invalid format` 或参数内容丢失，优先使用以下方式：

1. 对支持 stdin 的命令改用 `--params -` / `--data -`
2. 对必须内联传 JSON 的命令，改用 [../scripts/lark_cli_json.py](../scripts/lark_cli_json.py) 直接传 argv
3. 在 PowerShell 中优先用 `--json-env` 从环境变量读取 JSON

示例：

```powershell
$env:LARK_JSON='{"token":"<wiki_token>"}'
python ..\scripts\lark_cli_json.py `
  --json-env params=LARK_JSON `
  -- wiki spaces get_node --as user --format json
```

## 核心概念

- **知识空间（Space）**：知识库的顶层容器，通过 `space_id` 标识
- **节点（Node）**：知识库中的文档/页面，通过 `node_token` 标识
- **对象（Object）**：节点关联的实际文档，通过 `obj_token` 标识，类型为 `obj_type`（docx/sheet/bitable 等）
- **节点层级**：节点可以有父子关系，形成树状结构

## Wiki URL 解析

Wiki URL 格式：`https://xxx.feishu.cn/wiki/<wiki_token>`

Wiki URL 中的 token 不能直接用于文档操作，需要先解析：

```bash
lark-cli wiki spaces get_node --params '{"token":"<wiki_token>"}' --as user
```

返回中提取：
- `node.obj_type`：文档类型（docx/doc/sheet/bitable/slides/file/mindnote）
- `node.obj_token`：真实文档 token（用于后续文档操作）
- `node.node_token`：节点 token（用于节点层级操作）
- `node.title`：文档标题
- `node.space_id`：所属知识空间 ID

## 知识空间操作

### 列出知识空间

```bash
lark-cli wiki spaces list --as user
```

返回用户可访问的所有知识空间列表。

### 获取空间详情

```bash
lark-cli wiki spaces get --params '{"space_id":"<SPACE_ID>"}' --as user
```

### 创建知识空间

lark-cli 原生不提供 space create 快捷命令，使用原始 API：

```bash
lark-cli api POST /open-apis/wiki/v2/spaces \
  --data '{"name":"空间名称","description":"空间描述"}' \
  --as user
```

从返回的 `data.space.space_id` 提取空间 ID。

## 节点操作

### 创建节点（推荐使用 +node-create 快捷命令）

```bash
lark-cli wiki +node-create \
  --space-id <SPACE_ID> \
  --parent-node-token <PARENT_NODE_TOKEN> \
  --title "节点标题" \
  --as user
```

**参数说明**：

| 参数 | 必须 | 说明 |
|------|------|------|
| `--space-id` | 否 | 目标空间 ID；user 身份可用 `my_library` 表示个人文档库 |
| `--parent-node-token` | 否 | 父节点 token，创建为其子节点 |
| `--title` | 否 | 节点标题 |
| `--obj-type` | 否 | 文档类型（docx/sheet/mindnote/bitable/slides），默认 docx |
| `--node-type` | 否 | 节点类型（origin/shortcut），默认 origin |

**空间解析优先级**：`--space-id` > `--parent-node-token`（自动推断空间） > `my_library`

**返回值**：
```json
{
  "resolved_space_id": "实际使用的空间 ID",
  "resolved_by": "解析方式",
  "node_token": "新节点 token",
  "obj_token": "关联文档 token",
  "obj_type": "docx",
  "title": "节点标题"
}
```

**关键**：保存 `node_token`（用于创建子节点）和 `obj_token`（用于写入文档内容）。

### 列出子节点

```bash
lark-cli wiki nodes list \
  --params '{"space_id":"<SPACE_ID>","parent_node_token":"<PARENT_NODE_TOKEN>"}' \
  --as user
```

### 一致性验证

如果同时传 `--space-id` 和 `--parent-node-token`，系统会验证父节点所属空间与指定空间一致。不一致会返回验证错误。

### Bot 身份注意事项

使用 `--as bot` 创建节点时：
- 节点归属 Bot 应用
- 系统会尝试自动授予当前 CLI 用户管理权限（`permission_grant` 字段）
- Bot 没有个人文档库，不能使用 `--space-id my_library`
- Bot 不能添加部门类型成员

**建议**：本技能统一使用 `--as user` 身份操作知识库。

## 成员管理

### 添加成员

```bash
lark-cli wiki members create \
  --params '{"space_id":"<SPACE_ID>"}' \
  --data '{"member_type":"openid","member_id":"<OPEN_ID>","type":"user"}' \
  --as user
```

成员类型：
- `openid`：个人用户
- `openchat`：群组
- `opendepartmentid`：部门（仅 user 身份支持）

### 查找用户 open_id

```bash
lark-cli contact +search-user --keyword "用户姓名" --as user
```

## 归档流程最佳实践

1. **先确认再执行**：创建节点前先展示预期结构让用户确认
2. **逐层创建**：先创建父节点，保存 token，再创建子节点
3. **保存所有 token**：每次创建都保存 `node_token` 和 `obj_token`
4. **验证创建结果**：创建完成后用 `nodes list` 验证层级结构
5. **错误处理**：如果某个节点创建失败，记录错误并继续创建其他节点
