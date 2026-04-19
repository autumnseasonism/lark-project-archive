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

本技能统一使用 `domain` 维度授权，避免在长 scope 串上发生格式错误或漏配：

```bash
lark-cli auth login --domain wiki,docs,drive
```

执行后终端会显示授权链接，将链接提供给用户在浏览器中完成授权。

> `lark-cli auth login --scope ...` 在 CLI 中虽然存在，但本技能不推荐使用。原因是：
> - 长 scope 串在实际执行中更容易写错
> - 与主技能文档的 `--domain wiki,docs,drive` 策略不一致
> - 某些环境下会出现 `invalid or malformed scopes`

**多次 login 权限累积**：每次 `auth login` 授予的权限会累加，不会覆盖之前的。如果后续操作需要额外权限，优先按 domain 增量授权：

```bash
lark-cli auth login --domain <missing_domain>
```

### PowerShell 与 JSON 参数

本技能里部分命令会带 `--params` / `--data` JSON。若在 Windows PowerShell 中遇到 `not valid JSON` 或 `invalid format`，优先按以下顺序处理：

1. 对支持 stdin 的命令改用 `--params -` / `--data -`
2. 对必须内联传 JSON 的命令，改用 [../scripts/lark_cli_json.py](../scripts/lark_cli_json.py) 直接传 argv
3. 在 PowerShell 中优先用 `--json-env` 从环境变量读取 JSON，避免手写引号转义

## 权限不足处理

当 API 调用返回权限错误时，返回内容包含：
- `permission_violations`：缺少的 scope 列表
- `console_url`：开发者控制台权限配置链接
- `hint`：建议的修复命令

**User 身份处理**：
```bash
lark-cli auth login --domain <缺少的domain>
```

**Bot 身份处理**：
提供 `console_url` 链接给用户，引导在开发者控制台添加缺少的权限。Bot 不需要也不能执行 `auth login`。

## 本技能所需的完整 scope（参考）

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
