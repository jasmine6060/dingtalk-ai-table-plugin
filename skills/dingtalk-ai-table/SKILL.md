---
name: dingtalk-ai-table
description: Integrate with DingTalk「AI 表格」/「多维表」OpenAPI (write/read records). Use when building DingTalk bots that collect data and write into AI 表格 / Notable / 多维表, or when debugging 404/400/403/500 from `/v1.0/notable/...` endpoints. Covers correct URL paths, auth header, mandatory operatorId, app permissions, per-column-type value formats, and a diagnostic recipe.
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

## 参考实现

本 skill 沉淀自周报收集机器人项目（D:\工作\AI\Projects\周报收集）。参考实现：
- `core/ai_table.py` — token 缓存 + 读写客户端
- `core/contact.py` — staffId → (name, dept, unionid) 查询
- `handlers/dispatcher.py` — Stream 回调 → 按列类型构造 payload
- `inspect_sheet.py` / `inspect_records.py` — fields/records 诊断脚本（每次新企业接入复制改 baseId 就能用）
