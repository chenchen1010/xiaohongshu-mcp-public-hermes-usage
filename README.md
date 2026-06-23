# 小红书笔记采集 / 笔记评论采集 MCP 公网调用说明

这份文档给外部 Hermes / Codex 用户使用。你只需要配置公网 MCP 地址和 API key，然后通过 MCP tool 调用小红书笔记采集、笔记详情读取和笔记评论采集能力。

这个服务背后连接真实 Android 手机和小红书 App。所有用户共享同一台手机资源，所以采集任务会串行执行，请不要并发压测或批量长时间抓取。

## 连接信息

MCP endpoint：

```text
https://xhs-mcp.movingcostcheck.com/mcp
```

协议：

```text
MCP Streamable HTTP
```

鉴权方式：

```text
Authorization: Bearer <XHS_MCP_API_KEY>
```

也可以使用：

```text
X-API-Key: <XHS_MCP_API_KEY>
```

注意：API key 只放在 HTTP headers 里，不要放进任何 tool 参数。

## Hermes 配置示例

把 `<XHS_MCP_API_KEY>` 替换成服务提供者发给你的 `xhs_live_xxx` key。

```yaml
mcp_servers:
  xiaohongshu:
    url: "https://xhs-mcp.movingcostcheck.com/mcp"
    headers:
      Authorization: "Bearer <XHS_MCP_API_KEY>"
    timeout: 180
    connect_timeout: 60
```

如果你的客户端使用 JSON 配置，等价写法如下：

```json
{
  "mcpServers": {
    "xiaohongshu": {
      "baseUrl": "https://xhs-mcp.movingcostcheck.com/mcp",
      "headers": {
        "Authorization": "Bearer <XHS_MCP_API_KEY>"
      },
      "timeout": 180,
      "connectTimeout": 60
    }
  }
}
```

配置完成后，可以先运行 Hermes 自带的 MCP 连接测试。如果看到已连接并发现工具，说明公网地址、鉴权和 MCP 协议层已经正常。

## 推荐首次验证流程

第一次接入后，建议按这个顺序测试：

```text
1. check_login_status
2. search_notes(keyword: "杭州心理咨询", limit: 3, duration: 12)
3. 从 search_notes 返回结果里选择 1 条 note_id 和 xsec_token
4. get_note_detail(note_id: "...", xsec_token: "...")
5. 如需评论，再调用 get_note_comments(note_id: "...", xsec_token: "...")
```

如果 `check_login_status` 成功，说明 MCP、公网、鉴权、服务端手机和小红书登录态基本可用。

如果 `search_notes` 或后续采集工具返回空结果，不一定是连接失败。小红书页面、关键词、账号状态、目标笔记可见性都会影响采集结果。

## 常用 tools

### check_login_status

检查服务端手机、ADB 和小红书 App 状态。

费用：0 credits

参数：

```json
{}
```

### list_tools

列出当前 MCP 暴露的工具名。通常只用于连接验证。

费用：0 credits

参数：

```json
{}
```

### search_notes

按关键词搜索小红书笔记。

费用：成功且有结果时 3 credits

参数示例：

```json
{
  "keyword": "杭州心理咨询",
  "limit": 3,
  "duration": 12
}
```

常用参数：

```text
keyword: 必填，搜索关键词
limit: 返回数量上限，建议首次使用 3 到 10
duration: 手机采集等待秒数，建议 12 到 25
include_raw: 是否返回底层原始字段，默认 false
```

返回结果里通常会包含：

```text
note_id
xsec_token
title
desc
user
interact_info
```

后续调用详情、评论时，建议携带搜索结果里的 `note_id` 和 `xsec_token`。

### get_note_detail

获取单条笔记详情。

费用：成功且有结果时 2 credits

参数示例：

```json
{
  "note_id": "NOTE_ID",
  "xsec_token": "XSEC_TOKEN",
  "duration": 12
}
```

`xsec_token` 可选，但建议携带。缺少 `xsec_token` 时，小红书页面可能打不开或返回不稳定。

### get_note_comments

获取单条笔记评论。

费用：成功且有结果时 2 credits

参数示例：

```json
{
  "note_id": "NOTE_ID",
  "xsec_token": "XSEC_TOKEN",
  "limit": 20,
  "scrolls": 3,
  "duration": 20
}
```

建议先用较小的 `limit` 和 `scrolls` 验证目标笔记确实有评论，再扩大范围。

### get_note_replies

获取某条评论下的二级回复。

费用：成功且有结果时 2 credits

参数示例：

```json
{
  "note_id": "NOTE_ID",
  "comment_id": "COMMENT_ID",
  "limit": 20,
  "reply_taps": 8,
  "scrolls": 4,
  "duration": 20
}
```

建议先调用 `get_note_comments`，从返回的评论里选择确实有二级回复的 `comment_id`。如果目标评论没有二级回复，会返回空结果且不扣费。

### get_homefeed

抓取小红书首页推荐流。

费用：成功且有结果时 3 credits

参数示例：

```json
{
  "limit": 10,
  "duration": 12,
  "launch": true
}
```

首页推荐流受当前小红书账号兴趣、近期行为和平台推荐影响，结果不保证稳定复现。

### get_user_profile

根据小红书用户主页 URL 抓取用户主页信息。

费用：成功且有结果时 3 credits

参数示例：

```json
{
  "profile_url": "https://www.xiaohongshu.com/user/profile/USER_ID",
  "duration": 12
}
```

当前版本只接受 `profile_url`。不要只传 `user_id` 或 `red_id`。

## 响应字段

成功响应通常包含：

```text
ok: true
tool: 工具名
result_count: 本次返回的数据条数
items / comments / replies / profiles: 对应结果列表
```

每次响应还会带计费摘要：

```text
billing_mode
cost_credits
would_cost_credits
charged
balance_after
usage_event_id
```

字段含义：

```text
cost_credits: 本次实际扣费
would_cost_credits: 如果成功且有结果，理论价格是多少
charged: 是否已扣费
balance_after: 本次调用后的余额
usage_event_id: 本次调用的服务端记录 ID
```

## 计费规则

额度单位是整数 `credits`。

免费工具：

```text
check_login_status: 0 credits
list_tools: 0 credits
```

收费工具：

```text
search_notes: 3 credits
get_homefeed: 3 credits
get_note_detail: 2 credits
get_note_comments: 2 credits
get_note_replies: 2 credits
get_user_profile: 3 credits
```

只有同时满足以下条件才会扣费：

```text
tool 执行完成
返回 ok: true
result_count > 0
```

以下情况不扣费，但会写 usage 记录：

```text
无效 API key
参数校验失败
余额不足
队列满且未进入队列
空结果
超时
平台失败
底层采集失败
服务端异常
```

## 调用建议

做选题调研时建议：

```text
1. 先 check_login_status
2. 用 search_notes 搜 1 到 3 个关键词，每个关键词 limit 3 到 10
3. 从搜索结果里挑标题相关、互动较高的笔记
4. 对少量代表笔记调用 get_note_detail
5. 需要评论洞察时，再调用 get_note_comments
6. 需要二级回复时，从评论结果里挑 comment_id 调用 get_note_replies
7. 需要作者信息时，再调用 get_user_profile(profile_url)
```

不要一上来对所有搜索结果批量调用详情、评论和回复。评论区采集会占用真实手机较久，也会消耗 credits。

## 并发与超时

这个服务共享同一台真实手机，采集任务会串行执行。

建议：

```text
一次只发一个采集任务
等当前任务完成后再发下一个任务
搜索 duration 建议 12 到 25 秒
详情 duration 建议 12 到 20 秒
评论和回复 duration 建议 20 到 45 秒
评论 scrolls 首次建议 3 到 5
回复 reply_taps 首次建议 6 到 10
```

如果正在排队或手机被占用，请等待当前任务结束后重试，不要并发重试。

## 常见错误

### UNAUTHORIZED

通常表示没有带 API key、API key 无效，或 key 已被禁用。

处理方式：

```text
检查 Authorization header
确认格式是 Bearer xhs_live_xxx
不要把 API key 放在 tool 参数里
如果仍失败，联系服务提供者换 key
```

### INSUFFICIENT_CREDITS

表示余额不足。

处理方式：

```text
联系服务提供者充值
不要重复重试同一个收费任务
```

### QUEUE_FULL

表示当前手机采集队列已满。

处理方式：

```text
稍后重试
降低并发
缩小 limit、scrolls 或 duration
```

### CAPTURE_FAILED / TIMEOUT

表示服务端手机采集失败或超时。这类错误通常不是云端 Hermes 配置问题。

处理方式：

```text
先不要无限重试
保留 error_code、error_message、usage_event_id
换一个关键词或目标笔记再试一次
如果连续失败，联系服务提供者排查服务端手机或小红书页面状态
```

### result_count 为 0

表示本次调用执行完成，但没有采集到有效结果。

常见原因：

```text
关键词结果太少
目标笔记不可见或已失效
目标评论没有二级回复
duration 太短
小红书页面暂时没有触发对应接口
```

空结果不扣费。可以换关键词、换笔记、延长 duration，或稍后再试。

## 当前验证状态

截至 2026-06-19，已通过公网 HTTPS 和外部 API key 实测：

```text
MCP 连接和工具发现正常
check_login_status 可用
search_notes 可返回搜索结果并正常扣费
get_note_detail 可返回笔记详情并正常扣费
get_note_comments 可返回评论并正常扣费
get_note_replies 可返回二级回复并正常扣费
get_homefeed 可返回首页推荐并正常扣费
get_user_profile 可返回用户主页并正常扣费
空结果和失败请求不扣费
```

## 相关文档

- 电商采集 API v1 JSON-only MCP wrapper 接入说明：[`xhs-api-v1-json-only-mcp-usage.md`](./xhs-api-v1-json-only-mcp-usage.md)
