# 小红书采集 MCP 接入说明

本文档面向需要接入小红书采集能力的外部 agent，例如云服务器上的 Hermes、Codex 或其他自动化系统。

接入后，agent 可以通过 MCP 工具创建采集任务、查询任务状态、读取 JSON 结果，并可选回传下游系统写入情况。

公开 GitHub 文档仓库：

```text
https://github.com/chenchen1010/xiaohongshu-mcp-public-hermes-usage
```

后续如果本地文档 `docs/mcp-xhs-api-v1-usage.md` 有更新，需要同步推送到该 GitHub 仓库。

## 1. 用户后台、API Key 和充值

用户后台地址：

```text
https://xhsapi.qwjxqn.xyz/admin-dashboard.html
```

每个用户需要先进入用户后台注册或登录账号，然后创建或获取 API Key。

API Key 是调用采集服务的凭证，也是计费归属依据。飞书边栏插件、Hermes、Codex 或其他 agent 可以使用同一个 API Key。

用户需要在用户后台充值采集额度。充值后，额度会进入该用户账号下，可供该账号的 API Key 使用。

建议：

- 飞书边栏插件可以使用用户自己的 API Key。
- Hermes、Codex 等 agent 也可以使用同一个 API Key。
- 如果需要单独管理 agent 权限，建议在用户后台为 agent 单独创建一个 API Key，方便后续禁用、轮换或查看使用记录。

## 2. 扣费规则

采集任务提交时，系统会先根据任务目标数量预留额度。

当任务完成并生成可计费 JSON records 时，系统会按生成的 records 数量结算采集额度。

扣费规则：

- 生成多少条可计费 JSON records，就扣多少采集额度。
- 没有生成的记录不扣费。
- 任务提交时预留但最终未生成 records 的额度，会释放回用户余额。

示例：

- 用户提交目标数量 `10`
- 系统先预留 `10` 个采集额度
- 任务完成后生成 `7` 条 JSON records
- 最终扣费 `7`
- 释放剩余 `3`

`writeback_xhs_task` 不决定扣费。它只用于 agent 把 records 写入自己的业务系统后，回传下游写入情况，例如写入了哪些飞书记录、数据库记录或 CRM 记录。已结算任务调用 writeback 不会重复扣费。

## 3. 服务地址和环境变量

生产服务地址：

```text
https://xhsapi.qwjxqn.xyz
```

在运行 MCP server 的环境里配置：

```bash
export XHS_API_BASE_URL="https://xhsapi.qwjxqn.xyz"
export XHS_API_KEY="xhs_xxx"
```

PowerShell：

```powershell
$env:XHS_API_BASE_URL = "https://xhsapi.qwjxqn.xyz"
$env:XHS_API_KEY = "xhs_xxx"
```

`XHS_API_KEY` 是用户后台生成的 API Key。

请不要把 API Key 写进公开代码仓库、日志或聊天记录。

## 4. 启动 MCP Server

公网服务不直接提供 `/mcp` HTTP 路由。MCP server 运行在 agent 所在机器上，例如云服务器、本地电脑或 agent 容器里。

Hermes、Codex 或其他 agent 通过本机 stdio 连接这个 MCP server；MCP server 再使用 `XHS_API_KEY` 调用生产 HTTP API。

在项目根目录运行：

```bash
python mcp/xhs-api-v1-server/server.py
```

PowerShell：

```powershell
python .\mcp\xhs-api-v1-server\server.py
```

## 5. 可用工具

### `create_xhs_task`

创建采集任务。

```json
{
  "input": "关键词、商品链接或店铺链接",
  "limit": 20,
  "sort_mode": "sales"
}
```

说明：

- `input` 必填，可以是关键词、商品链接或店铺链接。
- `limit` 可选，表示目标采集数量。
- `sort_mode` 可选，支持 `sales` 或 `comprehensive`。
- 商品链接任务固定按单条商品处理，即使传入更大的 `limit` 也只按 `1` 条处理。

### `list_xhs_tasks`

查看最近任务。

```json
{
  "limit": 20
}
```

### `get_xhs_task`

查看单个任务状态。

```json
{
  "task_id": 123
}
```

常见状态：

- `pending`：等待采集
- `running`：采集中
- `completed`：已完成
- `partial`：部分完成
- `failed`：失败
- `cancelled`：已取消

### `get_xhs_task_progress`

查看任务进度和事件。

```json
{
  "task_id": 123
}
```

### `get_xhs_task_result`

读取任务 JSON 结果。

```json
{
  "task_id": 123
}
```

返回结果中会包含 `records` 和 `billing`。通常任务完成时已经按生成的可计费 records 数量结算；重复读取不会重复扣费。

### `writeback_xhs_task`

可选。回传下游系统写入结果。它不决定扣费，也不会对已结算任务重复扣费。

```json
{
  "task_id": 123,
  "written_count": 1,
  "record_ids": ["rec_xxx"],
  "failed_count": 0
}
```

说明：

- `written_count` 是成功写入业务系统的记录数。
- `record_ids` 是下游系统里的记录 ID，可选但建议传。
- `failed_count` 是写入失败数量。
- 如果写入过程中发生错误，可以传 `error` 字段。

### `cancel_xhs_task`

取消任务。

```json
{
  "task_id": 123
}
```

### `retry_xhs_task`

重试任务。

```json
{
  "task_id": 123
}
```

### `extend_xhs_task`

追加任务目标数量。

```json
{
  "task_id": 123,
  "target_count": 50
}
```

`target_count` 必须大于任务当前目标数量。

## 6. 标准调用流程

### 第一步：创建任务

```json
{
  "input": "蓝牙耳机",
  "limit": 20,
  "sort_mode": "sales"
}
```

调用 `create_xhs_task` 后会返回任务 ID：

```json
{
  "ok": true,
  "task_id": 123,
  "task_type": "keyword",
  "status": "pending",
  "target_count": 20
}
```

### 第二步：轮询状态

调用 `get_xhs_task`：

```json
{
  "task_id": 123
}
```

当状态变为 `completed`、`partial`、`failed` 或 `cancelled` 时，表示任务已经进入终态。

### 第三步：读取结果

调用 `get_xhs_task_result`：

```json
{
  "task_id": 123
}
```

读取返回的 `records`。任务完成生成 records 时，系统已经按可计费 records 数量扣费，并释放未生成部分的预留额度。重复读取结果不会重复扣费。

### 第四步：可选，回传下游写入结果

如果 agent 把 records 写入了飞书、多维表格、数据库或其他业务系统，可以调用 `writeback_xhs_task` 回传写入情况。

```json
{
  "task_id": 123,
  "written_count": 18,
  "record_ids": ["rec_1", "rec_2", "rec_3"],
  "failed_count": 2
}
```

## 7. 输入示例

### 关键词采集

```json
{
  "input": "蓝牙耳机",
  "limit": 20,
  "sort_mode": "sales"
}
```

### 商品链接采集

```json
{
  "input": "https://www.xiaohongshu.com/goods-detail/68f89ac3ce94ee0001604395",
  "limit": 1
}
```

### 店铺链接采集

```json
{
  "input": "https://www.xiaohongshu.com/vendor/64abc",
  "limit": 50
}
```

## 8. 常见问题

### 用户后台在哪里？

```text
https://xhsapi.qwjxqn.xyz/admin-dashboard.html
```

用户可以在后台注册或登录账号、创建 API Key、充值额度、查看任务和使用记录。

### API Key 可以和飞书边栏插件共用吗？

可以。飞书边栏插件、Hermes、Codex 和其他 agent 都可以使用同一个 API Key。

如果希望分开管理权限或查看不同 agent 的使用情况，可以在用户后台创建多个 API Key。

### 什么时候会扣费？

任务创建时会预留额度。任务完成并生成可计费 JSON records 时，系统会按 records 数量扣费，并释放未生成部分的预留额度。

### 只是读取结果会扣费吗？

不会重复扣费。任务完成生成 records 时已经结算；读取结果只是查看已经生成的 records 和 billing 状态。

如果 agent 后续会把 records 写入业务系统，可以调用 `writeback_xhs_task` 回传写入情况，但这不会重复扣费。

### 任务失败会扣费吗？

如果没有返回可计费 JSON records，不会扣除采集额度。任务预留额度会按结算结果释放。

### API Key 无效怎么办？

请检查：

- `XHS_API_KEY` 是否正确
- API Key 是否已在用户后台启用
- API Key 所属账号是否还有可用额度

### 额度不足怎么办？

请用户到用户后台充值采集额度，或降低任务 `limit`。

## 9. 直接 HTTP 调用

如果接入方不使用 MCP，也可以直接调用 HTTP API。

HTTP 服务地址：

```text
https://xhsapi.qwjxqn.xyz
```

鉴权方式：

```http
Authorization: Bearer xhs_xxx
```

创建任务：

```http
POST /api/v1/xhs/tasks
Content-Type: application/json
Authorization: Bearer xhs_xxx

{
  "input": "蓝牙耳机",
  "limit": 20,
  "sort_mode": "sales"
}
```

读取结果：

```http
GET /api/v1/xhs/tasks/{task_id}/result
Authorization: Bearer xhs_xxx
```

可选，回传下游写入结果：

```http
POST /api/v1/xhs/tasks/{task_id}/writeback
Content-Type: application/json
Authorization: Bearer xhs_xxx

{
  "written_count": 18,
  "record_ids": ["rec_1", "rec_2"],
  "failed_count": 0
}
```
