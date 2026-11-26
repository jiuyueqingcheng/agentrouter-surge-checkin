# AgentRouter Surge 每日签到模块

AgentRouter（https://agentrouter.org）每日登录签到模块：**登录本身就是签到**，每天登录一次得 **$25 额度**（站方 FAQ 原话：“签到领＄25额度 | $25 claim”，“需要退出后重新登陆才会到账”）。

## 安装

1. 在 Surge 里打开模块安装链接：

   ```
   https://cdn.jsdelivr.net/gh/JiuYueQingCheng/agentrouter-surge-checkin@main/agentrouter.sgmodule
   ```

2. 导入后在模块的「编辑参数」里填写：

   - `username`：AgentRouter 账号
   - `password`：AgentRouter 密码
   - `checkin_hour`：每天签到小时（0-23，默认 9）
   - `checkin_minute`：每天签到分钟（0-59，默认 0）

   例如每天 8 点 30 分签到，就填 `checkin_hour=8`、`checkin_minute=30`。

3. Surge 会在设定时间每天自动运行一次。想手动跑一次：Surge → 模块 → AgentRouter 签到 → 运行脚本（或在脚本页面手动触发）。

## 账号说明（重要）

- 如果账号是用 **GitHub / LinuxDo 第三方登录**创建的（例如用户名 `github_830975`），先在网站个人设置里给账号设一个密码，否则账号密码登录会失败；GitHub 登录本身无法在 Surge 定时任务里完成。
- 密码里有 `&` 请写成 `%26`，有 `=` 请写成 `%3D`（脚本会解码回来）。
- 请勿把账号密码发到聊天或公开仓库，只填在 Surge 本地。

## 运行结果

| 通知标题 | 含义 |
|---|---|
| ✅ 今日签到已确认 | 登录成功且找到今日签到记录，显示本次/今日奖励（+$25）、当前余额、累计消耗、累计调用次数 |
| ⚠️ 签到待确认 | 登录成功但没找到今日签到记录，或余额查询失败——请到网站核对签到记录与余额 |
| ❌ 登录失败 | 账号或密码不对，或网站需要验证码——请先在网页确认再更新模块参数 |
| 🚫 未配置账号 | 模块参数里没填账号密码 |

## 原理

AgentRouter 是老版 New-API（`/api/status` 已实测验证，2026-11-25）：

| 步骤 | 请求 |
|---|---|
| 查余额显示配置 | `GET /api/status` → `display_in_currency`、`quota_per_unit` |
| 登录（= 签到） | `POST /api/user/login` `{"username","password"}` → `data.id`、`data.checked_in`、Set-Cookie `session=` |
| 查余额 | `GET /api/user/self`（带 `Cookie: session=…` 和 `New-API-User: <id>`）→ `quota`、`used_quota`、`request_count` |
| 确认签到记录 | `GET /api/log/self?p=1&page_size=20` → 找当天 `type=4`、内容含“签到成功”的记录，格式如“每日签到成功， 增加额度 $25 额度” |

本站 `/api/status` 已实测验证（2026-11-25）：

- `version: "init-20261016-861e4557"` —— 老版 New-API，登录即签到的流程适用
- `display_in_currency: true`，`quota_per_unit: 500000` —— 余额按 `$ = 额度 / 500000` 显示
- `password_login_enabled` —— 未在状态里单独列出，账号密码登录为默认登录方式
- `github_oauth: true`，`linuxdo_oauth: true` —— 支持 GitHub / LinuxDo 登录，但无法在 Surge 定时任务里完成
- `turnstile_check: false` —— 登录不需要 Turnstile 票据

## 文件

| 文件 | 说明 |
|---|---|
| `agentrouter.js` | 签到脚本（Surge / Loon / Quantumult X / Stash 通用） |
| `agentrouter.sgmodule` | Surge 模块（cron + 模块参数） |
| `README.md` | 本说明 |

## 来源

改编自 MaYIHEI 的 paperclip 仓库（app/agentrouter，原作者 @773075692，改编者 MaYIHEI），本版按 agentrouter.org 的 `/api/status` 适配，增加了 Surge 字符串 `$argument` 解析和 Surge 自动 Cookie 管理的容错：

- 原仓库：https://github.com/MaYIHEI/paperclip/tree/main/app/agentrouter

## 未验证的部分（实话）

- **真实账号的完整链路**：脚本用模拟的老版接口响应做了两轮测试——本次登录新签到（`checked_in=false`，日志里有今日 $25 记录）、今天已签到过（`checked_in=true`，日志里有今日记录）——以及真站假账号登录（正确报“登录失败”），三条分支都通过了。但没有真实账号，无法在 AgentRouter 真站上跑通登录成功后的完整链路。第一次用真账号跑如果显示“待确认”，请把通知原文发我，我再改。
- **签到奖励金额格式**：按原脚本的日志格式解析（“每日签到成功， 增加额度 $25 额度”）；如果本站返回的是原始额度数字（例如“增加额度 5000 额度”），脚本会按 `quota_per_unit=500000` 换算成美元显示；两种都匹配不上时，通知会原样贴出日志内容的前 60 字，并提示到网站核对，但签到本身仍算已确认。
- **日志类型**：脚本按 New-API 通用的 `type=4` 找签到记录。如果 AgentRouter 的签到记录用了别的 type，首次运行会报「未找到今日签到记录」，到时把网站使用日志里那条签到记录的 type 告诉我，我改一行就行。
