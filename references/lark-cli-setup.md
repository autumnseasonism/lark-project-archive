# lark-cli 配置与认证

本技能依赖 `lark-cli` 命令行工具操作飞书 API。本文档包含安装、配置、认证和权限管理的完整流程。

## 安装

```bash
npm install -g @larksuite/cli && npx skills add larksuite/cli -g -y
```

安装后验证：
```bash
lark-cli --version
```

## 配置初始化

首次使用需初始化应用配置：

```bash
lark-cli config init --new
```

按提示填写飞书开发者平台的 appId 和 appSecret。这些信息在 [飞书开放平台](https://open.feishu.cn) 的应用管理页面获取。

## 身份类型

lark-cli 支持两种身份，通过 `--as` 参数切换：

| 身份 | 参数 | 访问方式 | 适用场景 |
|------|------|----------|----------|
| 用户 | `--as user` | `lark-cli auth login` 授权 | 访问用户个人资源（日历、云文档、知识库等） |
| 应用 | `--as bot` | 自动（仅需 appId + appSecret） | 应用级操作 |

**本技能主要使用 `--as user` 身份**，因为需要访问用户的知识库和文档。

### 身份选择原则

- Bot 无法看到用户资源（日历、云文档、邮件）
- Bot 无法代表用户操作（创建的文档归属 Bot）
- Bot 权限：只需在开发者控制台配置 scope
- User 权限：需要控制台 scope + 用户 auth login

## 用户认证

本技能需要以下权限 scope：

```bash
lark-cli auth login --scope "wiki:space:read wiki:space:retrieve wiki:node:read wiki:node:create wiki:node:retrieve docx:document:write_only docx:document:readonly drive:drive:write drive:drive:read"
```

执行后终端会显示授权链接，将链接提供给用户在浏览器中完成授权。

**多次 login 权限累积**：每次 `auth login` 授予的 scope 会累加，不会覆盖之前的。如果后续操作需要额外 scope，可以增量授权：

```bash
lark-cli auth login --scope "<missing_scope>"
```

## 权限不足处理

当 API 调用返回权限错误时，返回内容包含：
- `permission_violations`：缺少的 scope 列表
- `console_url`：开发者控制台权限配置链接
- `hint`：建议的修复命令

**User 身份处理**：
```bash
lark-cli auth login --scope "<缺少的scope>"
```

**Bot 身份处理**：
提供 `console_url` 链接给用户，引导在开发者控制台添加缺少的权限。Bot 不需要也不能执行 `auth login`。

## 本技能所需的完整 scope

| scope | 用途 |
|-------|------|
| `wiki:space:read` | 查询知识空间信息 |
| `wiki:space:retrieve` | 列出知识空间 |
| `wiki:node:read` | 读取知识库节点 |
| `wiki:node:create` | 创建知识库节点 |
| `wiki:node:retrieve` | 列出知识库节点 |
| `docx:document:write_only` | 写入文档内容 |
| `docx:document:readonly` | 读取文档内容（验证归档结果） |
| `drive:drive:write` | 云空间写入（白板上传） |
| `drive:drive:read` | 云空间读取（白板查询验证） |

## 版本更新

lark-cli 命令执行后，如果 JSON 输出包含 `_notice.update` 字段，说明有新版本。提示用户更新：

```bash
npm update -g @larksuite/cli && npx skills add larksuite/cli -g -y
```

更新后需退出并重新打开 AI Agent。

## 安全规则

- 禁止将 appSecret、accessToken 等敏感信息以明文输出到终端
- 写入/删除操作前确认用户意图
- 使用 `--dry-run` 预览危险操作
