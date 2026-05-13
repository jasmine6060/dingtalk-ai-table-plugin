---
name: dingtalk-ai-table
description: Integrate with DingTalk「AI 表格」/「多维表」OpenAPI (write/read records). Use when building DingTalk bots that collect data and write into AI 表格 / Notable / 多维表, or when debugging 404/400/403/500 from `/v1.0/notable/...` endpoints. Also covers DingTalk Stream long-connection reconnect patterns (normal vs real failure), QPS 90002 platform-wide rate-limit avoidance, and Windows always-on deployment via Task Scheduler + log rotation. Topics: correct URL paths, auth header, mandatory operatorId as unionId, app permissions, per-column-type value formats, fields→records diagnostic recipe, pythonw + InteractiveToken pitfalls, TimedRotatingFileHandler.
---

# DingTalk AI 表格 (Notable) OpenAPI 集成手册

## 命名约定（关键）

钉钉产品叫「**AI 表格**」（也叫「多维表」），但 OpenAPI 命名空间沿用旧名 `notable`。
**别去搜 "aiTable" / "aiBase" / "aiInteraction"，搜 `notable`。**

文档入口：https://open.dingtalk.com/document/development/api-notable-insertrecords

## 端点（v1.0）

```
POST   /v1.0/notable/bases/{baseId}/sheets/{sheetId}/records   # 写记录
GET    /v1.0/notable/bases/{baseId}/sheets/{sheetId}/records   # 读记录
GET    /v1.0/notable/bases/{baseId}/sheets/{sheetId}/fields    # 取列定义（debug 神器）
```

**Host: `api.dingtalk.com`**，不是 `oapi.dingtalk.com`（那是老接口）。

## 三个正交的必备条件（任一缺失都会失败）

### 1. 路径 — `notable/bases/...`

错误形态：`404 InvalidAction.NotFound — Specified api is not found`
→ 八成把路径写成了 `/v1.0/aiInteraction/aiBaseTables/...` 这类幻觉值

### 2. Access token — 走 v1.0 接口

```
POST https://api.dingtalk.com/v1.0/oauth2/accessToken
body: {"appKey": "...", "appSecret": "..."}
```

返回 `accessToken`（注意是驼峰，不是老接口的 `access_token`），有效期 `expireIn` 秒（典型 7200）。

**Header 名是 `x-acs-dingtalk-access-token`**，不是 `Authorization`。

⚠️ v1.0 token 跟老 `oapi.dingtalk.com/gettoken` 拿到的 token **互不通用**。如果项目同时用通讯录老接口（`/topapi/v2/user/get` 等），要维护两个 token 缓存。

### 3. operatorId — 强制 query 参数，必须是 unionId

错误形态：
- `400 MissingoperatorId — operatorId is mandatory` → 忘了带
- `400 paramError-operatorId` → 带了但值不是 unionId（staffId/userId 全不认）

```python
params = {"operatorId": "<unionId>"}
requests.post(url, params=params, ...)
```

unionId 必须是一个**对该 AI 表格有写权限**的钉钉用户的 unionId。通常用机器人创建者/管理员自己的。

## 应用权限（开发者后台 → 权限管理 → 申请 → 发布新版本）

| Scope | 用途 | 不批的报错 |
|---|---|---|
| `Notable.Base.Write.All` | 写 AI 表格 | 403 `Forbidden.AccessDenied.AccessTokenPermissionDenied` |
| `Notable.Base.Read.All` | 读 AI 表格 | 同上 |
| `qyapi_get_member` | 查用户详情（拿姓名/unionId） | 60011 sub_code |
| `qyapi_get_department_list` | 查部门树 | 60011 sub_code |

**审批通过 ≠ 生效**：还要在开发者后台「版本管理」里**发布新版本**。发布后：
- 新申请的 v1.0 token 立即带新权限
- 已缓存的老 token 不会动态扩权 → 要么重启服务清缓存，要么等 token 过期（默认 7200s）自动重申

## 列值格式（按列类型）

钉钉服务端对**类型不匹配的值会兜底成 `500 internalError`**，错误信息毫无帮助。所以列类型搞错最难 debug。**先用 fields 接口拉真实 schema 是省时间的唯一办法**（见下面的诊断秘籍）。

| 列类型 | 写入值格式 | 例子 |
|---|---|---|
| `text` | 字符串 | `"工作内容": "今天和 A 客户沟通了..."` |
| `date` | **Unix 毫秒整数**（不是字符串，哪怕列 formatter 是 `YYYY-MM-DD HH:mm:ss`） | `"标题": 1778480332020` |
| `user` | **`[{"unionId": "<unionId>"}]` 数组对象** | `"商务负责人": [{"unionId": "rAQBQ..."}]` |
| AI 装饰列（`decorator=ai-agent`） | 不要写，钉钉自动从源列派生 | — |

⚠️ **user 字段格式踩过的坑**：
- `[{"id": "..."}]` → 400 `invalid for field` ❌
- `["unionid"]` → 500 ❌
- `[{"unionId": "..."}]` → ✅

读记录的返回里 user 字段是 `{"unionId": "..."}` 对象，按这个反推写入格式最稳。

## 诊断秘籍

**症状：写入返回 500 `internalError - Please try again later`，重试还是 500**
→ 99% 不是临时故障，是 payload 形态问题（值类型 / 列名不存在 / 没有 base 访问权）

**recipe**：先调读路径，钉钉服务端对读接口的错误处理比写接口靠谱得多。

```python
# 1. 拉列定义 — 看真实列名 + 类型
GET /v1.0/notable/bases/{baseId}/sheets/{sheetId}/fields?operatorId=<unionId>

# 2. 拉 1-3 条已有记录 — 看每个字段实际值长什么样
GET /v1.0/notable/bases/{baseId}/sheets/{sheetId}/records?operatorId=<unionId>&maxResults=3
```

读到的 `createdBy: {"unionId": "..."}` 这种对象就是 user 字段的"长相"，回写格式跟它对齐。

**fields 接口若返回 404 `invalidRequest.document.notFound`**：
不是路径错，是 **baseId 在当前应用/operator 下不可见**。可能是：
- baseId 抄错了（要从 AI 表格分享链接里抠 `?baseId=...`）
- 切了新应用 / 换了组织，旧 base 不在新作用域里
- operator 这个钉钉账号本身没被授权访问这张表

## 错误信息 → 根因速查表

| HTTP | 错误码 | 根因 |
|---|---|---|
| 404 | `InvalidAction.NotFound` | URL 路径不对（不是 notable 命名空间） |
| 404 | `invalidRequest.document.notFound` | baseId 不存在/operator 没权限看 |
| 400 | `MissingoperatorId` | 忘带 operatorId 参数 |
| 400 | `paramError-operatorId` | operatorId 不是合法 unionId |
| 400 | `invalidRequest.inputArgs.invalid - the value 'X' is invalid for field 'Y'` | Y 列的值格式不对（查 fields 看 Y 的 type） |
| 403 | `Forbidden.AccessDenied.AccessTokenPermissionDenied` | 缺 scope（错误信息里有申请链接） |
| 500 | `internalError - Please try again later` | 大概率列类型不匹配，去查 fields 看列 type |

## unionId 怎么拿（不需要管理员）

- **自己的**：登录 https://open-dev.dingtalk.com/ → 个人中心 → 账号信息
- **OpenAPI Explorer**：https://open-dev.dingtalk.com/apiExplorer → 调 `user/get` 传 userid
- **机器人代码里查别人的**：`POST https://oapi.dingtalk.com/topapi/v2/user/get?access_token=<老 token>` 
  body `{"userid": "<staffId>"}`，返回 `result.unionid`
  需要 `qyapi_get_member` 权限

## 最小可运行写入示例

```python
import time, requests

# 1. 拿 v1.0 token
tok = requests.post(
    "https://api.dingtalk.com/v1.0/oauth2/accessToken",
    json={"appKey": APP_KEY, "appSecret": APP_SECRET},
).json()["accessToken"]

# 2. 写一条
resp = requests.post(
    f"https://api.dingtalk.com/v1.0/notable/bases/{BASE_ID}/sheets/{SHEET_ID}/records",
    params={"operatorId": OPERATOR_UNION_ID},          # ← unionId
    headers={"x-acs-dingtalk-access-token": tok},
    json={
        "records": [{
            "fields": {
                "标题":     int(time.time() * 1000),   # date 列：Unix ms 整数
                "工作内容": "今天和 A 客户沟通续约方案", # text 列：字符串
                "所属部门": "商务",                      # text 列
                "商务负责人": [{"unionId": OPERATOR_UNION_ID}],  # user 列：unionId 数组对象
            }
        }]
    },
)
print(resp.status_code, resp.text)
```

## 关联：钉钉 Stream 机器人接消息（一般场景）

如果是"机器人收群消息 → 落 AI 表格"的整套场景：
- 长连接 SDK：`dingtalk-stream` (pip)
- 触发器：消息 `@` 机器人即推 Stream 回调
- 回调里能拿到 `senderStaffId`，但**没有 unionId** —— 要写 user 列必须用 `qyapi_get_member` 反查 unionId
- 失败兜底：unionId 拿不到时**直接跳过 user 列**（其他列照写），不要写错误格式触发 500

## Stream 长连接：正常断连 vs 真故障

Stream SDK 长连接**每隔几小时会自动断一次重连**，这是钉钉服务端对长连接做负载均衡的正常行为，不是故障。日志里会看到这些 ERROR / INFO 交替出现：

```
[ERROR] dingtalk_stream.client - [start] network exception, error=no close frame received or sent
[ERROR] dingtalk_stream.client - [start] network exception, error=sent 1011 (internal error) keepalive ping timeout; no close frame received
[INFO]  dingtalk_stream.client - received disconnect topic=disconnect, ... "reason":"persistent connection is timeout"
[INFO]  dingtalk_stream.client - open connection, url=https://api.dingtalk.com/v1.0/gateway/connections/open
[INFO]  dingtalk_stream.client - endpoint is {...}
```

**这些都是无害的**：紧跟在 ERROR 行后面的 `open connection` + `endpoint is` 就是 SDK 自动重连成功的证据。生产环境典型频率 1–3 小时一次。

**真故障的信号**：连续多次 `network exception` **没有**跟随 `endpoint is` 成功行 —— 说明无法重连。排查方向：
- 网络层（防火墙/代理屏蔽 `wss-open-connection-union.dingtalk.com:443`）
- token 过期或被吊销（看 access token 接口最近是否 401）
- 应用在开发者后台是否被禁用 / 撤销发布

**告警过滤建议**：别对 `dingtalk_stream.client` 单条 ERROR 触发告警，会一直误报。用"5 分钟窗口内 ≥ N 次 ERROR 且窗口结束时没有 endpoint 重建"这种模式判定。

## QPS 限制 (90002)：错开整点 + 缓存

典型错误：

```
errcode: 88
sub_code: 90002
sub_msg: 当前所有钉钉应用调用该接口次数过多，超出了该接口承受的最大qps，请求被暂时限制了，建议错开整点时刻调用该接口
apiPath: dingtalk.oapi.v2.department.listsub
从 2026-05-12 10:00:00 到 2026-05-12 10:00:00 请求总次数超过 1200 次
```

注意"**所有钉钉应用**" —— 这不是你这一个 app 的额度，而是**钉钉平台级别**的全局接口配额。整点（10:00:00 / 11:00:00 这种秒级整点）是流量 spike 高峰，最容易撞限流。

应对：
1. **启动/定时任务别踩整点**：cron 表达式不要写 `0 * * * *`，加 30–90s 的固定或随机偏移
2. **`listsub` 这类全量查询要驻内存缓存**：部门树启动时预热一次就够了，不要每次回调都重查
3. **遇到 90002 要带抖动重试**：`time.sleep(random.uniform(1, 5))` 之后再重试，避免几个实例同时撞同一秒
4. **降级路径**：如果通讯录查询限流且短时间无法绕开，应让"主业务（写表）"继续走，user/部门字段降级为空字符串或 staffId，不要让限流卡死消息处理

## Stale-state 排查

- 改了 scope / 改了 token cache 后**新 token 才带新权限**，老的 cached token 不会动态扩权。处理方式：重启服务或手动清缓存。
- AI 表格的列定义改了之后，先重新跑一次 fields 诊断，别凭印象写 payload。

## 跨企业复用要做的两件事

如果同一份代码要在多个企业 / 多张 AI 表格里跑：

1. **列名做成 .env 配置**，不要写死在代码里 — 不同企业搭表格时同一个逻辑列经常起不同名字（一处叫"提交时间"，另一处叫"标题"；一处叫"商务负责人"，另一处叫"对接人"）。一致的做法：
   ```python
   # config.py
   COL_SUBMIT_TIME = _optional("DING_COL_SUBMIT_TIME", "提交时间")  # 默认值兜底
   COL_NAME        = _optional("DING_COL_NAME",        "商务负责人")
   ...
   # schema.py
   WORK_FIELDS = {"submit_time": config.COL_SUBMIT_TIME, "name": config.COL_NAME, ...}
   ```
   切企业只改 .env，代码零改动。

2. **baseId / sheetId 抠链接时别复制错**。AI 表格分享链接里的 `?baseId=...&sheetId=...` 字段经常超过 20 字符；
   一次实操中曾出现 baseId 被截成 21 字符（真值 32 字符），导致 fields 接口 `404 invalidRequest.document.notFound`，
   但应用 scope/操作人都正确。**新企业接入时，必跑一次 fields 诊断验证 baseId 长度和可访问性，比线上 500 时回头排查省时间。**

## Windows 上长期常驻部署

让 bot 脱离 Claude Code / 终端会话，关机重启后自动起来。最轻量方案是 **Windows Task Scheduler**（零依赖、不用装服务框架）。

关键设置：
- **触发器**：登录时启动，延迟 10s 等网络就绪
- **动作**：`pythonw.exe main.py`（无 console 窗口）+ `WorkingDirectory` 指向项目根
- **失败重启**：`<RestartOnFailure>`，1 分钟间隔 × 10 次
- **执行时长**：不限制（`<ExecutionTimeLimit>PT0S</ExecutionTimeLimit>`）
- **断电策略**：不暂停（笔记本拔电源也继续跑）
- **多实例**：`IgnoreNew`（避免重复 Stream 连接抢同一个 ticket）

**坑 1 — 用户级 Python 安装时不能用 SYSTEM 跑**：如果 Python 在 `%LOCALAPPDATA%\Programs\Python\...`（典型 user install），任务必须以**你的用户身份**跑（`<LogonType>InteractiveToken</LogonType>`）。用 SYSTEM 账号会找不到 python.exe。这也意味着电脑开机但还在锁屏 / 未登录用户时 bot **不会启动**，登录后才起。如果要真正"开机即跑"，得改成 `<LogonType>Password</LogonType>` 并存一次 Windows 密码。

**坑 2 — XML 必须 UTF-16 LE with BOM**：`schtasks /Create /XML` 对 UTF-8 编码的 XML 在部分 Windows 版本上行为不稳。用 PowerShell 写文件时显式指定：

```powershell
[System.IO.File]::WriteAllText(
  $path, $content,
  [System.Text.UnicodeEncoding]::new($false, $true)   # UTF-16 LE + BOM
)
```

**坑 3 — `Last Result: 267009` 不是错误**：等于 `0x41301 = SCHED_S_TASK_RUNNING`，含义是"任务正在运行中"。`0x0` 才表示"上次成功完成"。新手容易误报警。

**完整可用的 XML 骨架**（替换 DOMAIN\username / 路径即可）：

```xml
<?xml version="1.0" encoding="UTF-16"?>
<Task version="1.4" xmlns="http://schemas.microsoft.com/windows/2004/02/mit/task">
  <Triggers>
    <LogonTrigger>
      <Enabled>true</Enabled>
      <UserId>DOMAIN\username</UserId>
      <Delay>PT10S</Delay>
    </LogonTrigger>
  </Triggers>
  <Principals>
    <Principal id="Author">
      <UserId>DOMAIN\username</UserId>
      <LogonType>InteractiveToken</LogonType>
      <RunLevel>LeastPrivilege</RunLevel>
    </Principal>
  </Principals>
  <Settings>
    <MultipleInstancesPolicy>IgnoreNew</MultipleInstancesPolicy>
    <DisallowStartIfOnBatteries>false</DisallowStartIfOnBatteries>
    <StopIfGoingOnBatteries>false</StopIfGoingOnBatteries>
    <AllowHardTerminate>true</AllowHardTerminate>
    <StartWhenAvailable>true</StartWhenAvailable>
    <ExecutionTimeLimit>PT0S</ExecutionTimeLimit>
    <RestartOnFailure>
      <Interval>PT1M</Interval>
      <Count>10</Count>
    </RestartOnFailure>
  </Settings>
  <Actions>
    <Exec>
      <Command>C:\Users\you\AppData\Local\Programs\Python\Python312\pythonw.exe</Command>
      <Arguments>main.py</Arguments>
      <WorkingDirectory>D:\path\to\project</WorkingDirectory>
    </Exec>
  </Actions>
</Task>
```

注册 / 启停 / 删除：

```powershell
schtasks /Create /TN "MyBot" /XML task.xml /F
schtasks /Run    /TN "MyBot"
schtasks /End    /TN "MyBot"
schtasks /Query  /TN "MyBot" /V /FO LIST
schtasks /Delete /TN "MyBot" /F
```

### 配套：日志按天切分

bot 长期常驻 + 单文件日志 = 越长越大、检索越来越慢。换成按天滚动：

```python
from logging.handlers import TimedRotatingFileHandler

file_handler = TimedRotatingFileHandler(
    log_dir / "agent.log",
    when="midnight",
    interval=1,
    backupCount=30,        # 保留最近 30 天，更老的自动删
    encoding="utf-8",
    utc=False,             # 用本地时区
)
file_handler.suffix = "%Y-%m-%d"   # 切分后命名：agent.log.2026-05-13
```

注意：rotation 是**进程内**触发的。如果进程在午夜前后正好崩了（即使被计划任务立刻拉起），可能错过那一次切分 —— 但新进程会从新的 `agent.log` 写起，损失是"昨天的尾巴跟今天的开头共用一个文件"，不是数据丢失。

## 参考实现

本 skill 沉淀自周报收集机器人项目（D:\工作\AI\Projects\周报收集）。参考实现：
- `core/ai_table.py` — token 缓存 + 读写客户端
- `core/contact.py` — staffId → (name, dept, unionid) 查询
- `handlers/dispatcher.py` — Stream 回调 → 按列类型构造 payload
- `inspect_sheet.py` / `inspect_records.py` — fields/records 诊断脚本（每次新企业接入复制改 baseId 就能用）
- `main.py` — `TimedRotatingFileHandler` + Stream 自动重连入口
- Windows 计划任务部署：`schtasks /Query /TN "DingTalkWorkCollectorBot"` 可查实际运行规则
