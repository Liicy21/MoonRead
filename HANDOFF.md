# 月读 YueRead · 项目交接文档

> 交接时间：2026-11-20 · 供接手者（Trae / 任意 AI 助手）快速续作
> **读这份之前，先读 `docs/AI_MODEL_GATEWAY.md` 的 §10–§12**（与前文冲突时以 §10–§12 为准）

---

## 0. 一句话现状

月读 YueRead 是一个已上线的个人阅读记录平台（Vue3 + CloudBase PostgreSQL）。
核心功能（登录、微信读书同步、分享、AI 助手多模型、三主题、玉兔吉祥物）**均已实现并实测通过**。
AI 多模型系统 P0–P6 全部闭环，只剩 **P7 增强** 与若干功能缺口待补。
线上地址：`https://lexi-d9gb3ruct4fdb9d0e-1512381244.tcloudbaseapp.com/`

---

## 1. ⚠️ 第一优先级：环境到期

- 环境 `lexi-d9gb3ruct4fdb9d0e`（个人版，ap-shanghai，AppId 1512381244）**2026-11-21 23:59:59 到期**，`IsAutoRenew=false`。
- 不续费 → 云函数 / 网关 / 静态托管 / 登录全部停服；**数据不丢**，续费后同 envId 立即恢复。
- 续费 ¥39.9/月，走 `manageEnv action=renew`，**必须先问用户确认再操作**。
- 备用方案 `licy-21`（体验版，2027-04-21 到期，零费用）已备好，但 envId/域名/PG 全变，需重跑迁移+部署+路由+绑 Key。
- **接手第一件事**：确认环境是否已续费。没续费的话先解决这个，否则改完也访问不了。

---

## 2. 环境 ID 的坑（血泪教训，务必照做）

曾因手打错 envId（`b3truct` vs 正确的 `b3ruc`）误报"环境已销毁"一场。铁律：

1. **永不手打 envId**，一律管道取权威值：
   ```bash
   tcb env list -r ap-shanghai --json | node -e 'let s="";process.stdin.on("data",d=>s+=d).on("end",()=>{process.stdout.write(j.data.find(e=>e.PackageName==="个人版").EnvId)})' > "$LOCALAPPDATA/Temp/lexi_envid.txt"
   # 之后用 "$(cat "$LOCALAPPDATA/Temp/lexi_envid.txt" | tr -d '\n')"
   ```
   注意上面 node 脚本里 `const j=JSON.parse(s)` 要在 end 回调内先解析。
2. MCP 先 `auth(set_env)` 绑定，之后调用**不带 envId 参数**。
3. 怀疑环境坏之前先用管道法复测，别急着让用户去控制台。

---

## 3. 凭据管理（安全红线）

**本仓库不含任何明文密钥/密码**，已用 `.gitignore` 排除：
- `cloudfunctions/.secrets/` —— 微信读书 API Key、加密密钥（真实值只在云端环境变量）
- `web/.env.local` —— 含 publishable key（JWT）
- `.workbuddy/` —— 工作记忆，含账号密码明文

接手后如何拿到凭据（**需要时向用户索取，不要猜、不要写进仓库**）：
- **管理员账号**：用户名 `licy`，密码由用户线下提供（`.workbuddy/memory/` 里有，但那份记忆不入库）。
- **微信读书 Key**：云端 `WEREAD_ENC_KEY` 环境变量 + `weread_credentials` 表加密列，前端不接触。
- **publishable key**：从 CloudBase 控制台 → 环境 → 全局设置 → 客户端 SDK 获取，填入 `web/.env.local`（模板见 `web/.env.example`）。
- `licy` 已在 `public.ai_admin_users` 白名单中（role=admin），登录后侧栏有「管理后台」入口。

改密码的正确手法（**不要猜旧密码**）：注册临时账号取 bcrypt 哈希 → `UPDATE auth.users SET password=(子查询)` → 删临时账号。全程不手打哈希。
`auth.users.password` 是 base64(bcrypt)，前缀 `JDJhJD` = `$2a$`。

---

## 4. 不可绕过的平台事实（全是实测，别再踩）

### CloudBase 平台层
1. **未登录调 `/v1/functions/<name>` 被平台注入层拦死**（403 EXCEED_AUTHORITY），与 OPA 无关 → 公开入口只能走 HTTP 网关（`manageGateway createRoute`，`upstreamResourceType=SCF`、`auth=false`、`enableSafeDomain=true`）。
2. **用户类型决定人数配额**：`internalUser` 受套餐限制（个人版 3 个触顶），`externalUser` 不限量 → 注册一律 `externalUser`。
3. **SDK `User.updatePassword()` 必抛** `invalid 【patch】 request method` → 用原生 fetch 复刻 `POST /auth/v1/user/sudo` → `PATCH /auth/v1/user/password`。
4. **网关转发 Event 函数**时 event 变 `{body:"<json>"}`，函数要归一化。
5. `enableSafeDomain:true` 后**网关自己处理 CORS**；函数不要再设 `Access-Control-Allow-Origin`（会变 `<origin>,*` 被浏览器拒）。
6. PG 环境 `ModifyResourcePermission` 被拒，权限只能走 `authz.user.rego`（环境级整份替换）。
7. **Node 下多个 SDK `init()` 实例共享 auth 会话** → "未登录"用例必须排在任何登录动作之前；切换身份前要**重新登录**（登录是覆盖式的）。
8. `tcb fn deploy` 覆盖已有函数用 **`--force`**；部署**不会**清掉线上未在 `cloudbaserc.json` 声明的环境变量（实测 `WEREAD_ENC_KEY` 保留）。
9. `tcb fn env` 只有 pull 没有 set → 设环境变量用 MCP `manageFunctions(action="updateFunctionConfig")`。
10. `managePgDatabase applyMigration` 校验本地文件与 sql 一致，不一致 fail closed；纯 GRANT/CREATE POLICY 用 `action="execute"` + `allowDdlViaExecute=true`。
11. **共享环境禁止 `GRANT ... ON ALL TABLES IN SCHEMA public`**（会波及别人项目）→ 逐表显式授权。
12. 🔴 **CloudBase Uid 是 int64（19 位），JSON/JS 管理接口会丢精度**（`...186`→`...8000`）→ 管用户只能直接写 PG（`auth.users.id` 是 bigint，字面量精确）。
13. 🔴 **PG 的 `auth` schema 可直接写**（`managePgDatabase action=execute role=service_role`）。删账号要连带清 `auth.token`（否则已删用户会话仍有效）、`auth.user_visit_events`、孤儿业务数据；`auth.audit_log_entries` 保留。
14. MCP `managePgDatabase` 的 `previewRows` **只回前 5 行** —— 查全量要用 `string_agg` 聚合到单行。
15. `auth.token.user_id` 是 varchar，`auth.users.id` 是 bigint —— JOIN 必须 `u.id::text = t.user_id`。
16. **`service_role` 下 RLS 完全不生效**（云函数就是这条通道）；前端直连才受 RLS 保护。

### 微信读书 API（真实 Key 实测，官方文档 3 处不实）
1. 阅读时长读 `book.readingTime`，**不是** `book.recordReadingTime`（后者恒 0）。
2. 已读完看 `books[].finishReading === 1`（有书 finishReading=1 但 progress=94，只看进度会漏判）。
3. `/book/bookmarklist` 有 `removed[]`（已删划线），漏掉留脏数据。
4. `/readdata/detail` 只支持 `mode=monthly`；历史周期靠 **`baseTime`** 回溯，季/年本地聚合。
5. 所有时间戳按**东八区日界**解释。书架 `mp`（bookId='mpbook'）跳过；`albums[]` 有声书要同步。
6. 错误 Key → `{"errcode":-2013}`（HTTP 401）。字段映射在 `cloudfunctions/weread-sync/lib/extract.js`。

### 多模型系统（P1–P6 实测坑）
1. `app.rdb()` 在 PG 环境返回 `PostgrestClient`（有 `.rpc()`）；MySQL 才是无 rpc 的 `MySqlClient`。
2. **PostgREST 按参数名匹配实参**，写错是静默 404/PGRST202。
3. `CREATE FUNCTION` 默认 `EXECUTE` 授予 PUBLIC → 必须 `REVOKE … FROM PUBLIC`，否则登录用户能用别人 uid 篡改额度（SECURITY DEFINER 绕 RLS）。
4. PG 代理 upsert 是整行覆盖 → 额度闸门必须原子（`ai_guard_reserve` 预扣 → `ai_guard_release` 回滚 → `ai_guard_settle` 结算）。
5. 云函数 Nodejs18.15 全局 fetch 实验性 → 加 `node-fetch@^2` 兜底（v3 纯 ESM）。
6. **0 / NULL 语义**：`ai_models.daily_quota_*` 的 0 = 零预算禁用该模型；`ai_user_settings.daily_*_limit` 的 0 = 一律拒绝；`rate_limit_per_min` 的 0 = 不限流；`total_quota_points` 的 0/NULL = 不限。前两类必须进 SQL 前用 JS 判定。
7. 用户层超限（L3）硬失败；模型层超限（L2）自动切备用。**模型预算被拒也算一次切换**。
8. 🔴 **路由名 `action` 与业务过滤条件不能同名** —— `event.action`('admin-logs') 被透传成 `WHERE action='admin-logs'` → 永远 0 条日志。改用 `logAction`；前端 `invoke` 必须 `{ ...data, action }`（路由参数永远赢）。
9. 🔴 **断言太弱让 bug 溜过** —— 只验 `Array.isArray(logs)`（0 条也算过）。断言必须验条数与内容。
10. 🔴 **测试夹具不能写死日期** —— 跨零点后 `dayKey()` 过滤全空，看着像回归其实是测试腐烂，一律动态取值。
11. **管理后台鉴权 = `ai_admin_users` 白名单**（非 rego）：rego 是环境级、粒度到函数，表达不了「同函数内只有 admin-* action 受限」；白名单还能表达 admin/viewer 两级。
12. **`ai_api_keys` 与 `ai_period_guard` 对 authenticated/anon 一律不授权**；密文三元组绝不开给管理员。
13. **唯一出口纪律**：`ai-assistant/index.js` 不许直接调 `app.ai()` 或 fetch 厂商，必须走 `lib/orchestrator.js` 的 `invokeWithFallback()`。
14. **托管目录 `InputOutputUnit` 不可信** —— 单价以 `lib/pricing.js` 为准。
15. 第三方端点 base_url 多已含版本段，path 必须覆盖（Kimi/豆包/GLM/百炼/混元），否则 404。

---

## 5. 已完成 / 未完成清单

### ✅ 已完成并实测
- 第一阶段：环境勘察、PG 10 张表 + RLS、前端骨架 7 页、设计系统
- 第二阶段：CloudBase 用户名密码登录、公开注册网关、改密、端到端 12/12
- 第三阶段：微信读书绑定/同步（手动 + 定时），真实 Key 实测，字段映射已校正
- 第四阶段：分享系统，端到端实测
- 第五阶段：静态托管部署 + 全链路回归
- 三主题系统（白昼 / 星空夜 / 跟随系统）：`color-scheme` + `light-dark()` 单表 token，按账号隔离，`initTheme()` 必须在 `createApp()` 前调用
- 月宫玉兔吉祥物：两张真实插画（`rabbit-night.webp` / `rabbit-day.webp`），夜间是用户原图抠底，白天是同原型 AI 生成，点击弹寄语、呼吸/蹦跳动效
- AI 多模型系统 **P0–P6**：三通道（首期托管 + 用户自带）、统一适配层、自动切换不丢历史、额度闸门、BYOK、用户设置页、管理后台（CRUD + 看板 + 日志 + 白名单）

### ⬜ 未完成（待 Trae 续作）
| 任务 | 说明 | 优先级 |
|---|---|---|
| **P7 增强** | 流式输出、批量任务、平台自配 provider 通道（`docs/AI_MODEL_GATEWAY.md` §10.6） | 中 |
| #17 | `weread-sync` 全量历史回溯（baseTime 逐年 / monthly 循环），保留上限保护防超时 | 中 |
| #20 | 云端现状审计与升级方案（实查云函数/PG/触发器/存储/日志/权限，带文件行号，不虚构） | 低 |
| #23 | 全站视觉升级收尾：书籍详情目录时间轴+金句卡片、个人中心头像/阅读成就 | 低（主视觉/主题/玉兔已做） |
| #24 | 功能缺口：导出与恢复、封面/海报上传云存储、自动同步开关、分享管理列表与撤销、AI 历史/用量面板、章节手动增删 | 高 |
| #25 | 运维监控与全站错误处理：同步状态/失败原因、定时任务执行记录、错误日志归集、空态与失败态区分、权限安全复核 | 高 |
| 双主题全站视觉回归 | 本机浏览器二进制被网络策略拦掉，无法真机截图。替代：vitest + happy-dom 真 DOM 断言（已过）。建议用户自己过一遍三主题 | 需用户配合 |

> 任务状态以 `.workbuddy/` 里的 TaskList 为准（不入库）。上面这张表是交接时点的快照。

---

## 6. 验证脚本（改动对应模块后必跑）

| 命令 | 项数 | 说明 |
|---|---|---|
| `node web/scripts/verify-auth-gateway.cjs` | 12 | 登录端到端，真调云端会注册账号，测完清理 |
| `node web/scripts/verify-weread-mapping.cjs` | 32 | 真打微信读书接口，读 `.secrets/WEREAD_API_KEY.txt`，不需 CloudBase |
| `node web/scripts/verify-share-logic.cjs` | 32 | 纯离线 |
| `node web/scripts/verify-share-e2e.cjs` | 11 | 分享端到端，会注册账号 |
| `node web/scripts/verify-ai-gateway.cjs` | 103 | AI 网关纯离线 |
| `node web/scripts/verify-ai-orchestrator.cjs` | 83 | 模拟 DB + 模拟模型 |
| `node web/scripts/verify-ai-admin.cjs` | 79 | 管理后台离线 |
| `AI_E2E_ADMIN_USER=licy AI_E2E_ADMIN_PW='<密码>' node web/scripts/verify-ai-e2e.cjs` | 60 | 真调云端+模型；密码命令行传入不入库；夹具自动种入/删除 |
| `cd web && npx vitest run` | 47 | 前端 DOM 渲染（happy-dom） |
| `cd web && npx vue-tsc --noEmit` | — | 类型检查 |

**e2e 收尾必须清理测试账号**：删 `aie2e%` 用户的 `ai_*` 数据 + `auth.token` + `auth.user_visit_events` + `ai_model_daily`（夹具自动删），复核业务数据基线。

**部署**：
```bash
tcb fn deploy <fn> --force -e "$ENV" -r ap-shanghai
cd web && npm run build && tcb hosting deploy dist -e "$ENV" -r ap-shanghai
```
> `tcb hosting deploy` 末尾 CLI 调 `reg.exe` 可能被安全策略拦，**上传本身已成功**——用线上 HTTP 200 + 主包哈希核对，别误判为失败。
> CDN 可能缓存旧哈希，加 `-H "Cache-Control: no-cache"` 复核；判断更新以本地 dist 哈希为准。

---

## 7. 数据基线（每轮测试后必须复核）

- 业务数据：`books 33 / chapters 1125 / quotes 125 / reflections 13 / shares 2 / weread_credentials 1 / reading_stats 7`
- `auth.users` 只应有 2 个：`administrator` + `licy`
- 模型目录 `ai_models` 49 行（无 `e2e%` 夹具残留）
- 全部 `ai_*` 业务表测试后归零

---

## 8. 本机环境注意事项

- 用户实际 Node 在 `D:\dev\nodejs`（v24.15，用户级 PATH）；agent 沙箱里 `node`/`git` 会被劫持到 `.workbuddy\binaries\`。
- 独立 node 脚本调 SDK 登录常被 SIGTERM 卡死，**放进 e2e 里跑就正常**（e2e 先发一次网关请求）。
- 浏览器真机测试不可用：agent-browser 的 Chromium 下载被网络策略拦（TLS handshake eof），Edge 进程被 SIGTERM。用 vitest + happy-dom 替代。
- Git Bash / MSYS 下不能直接跑 `idf.py`（与本仓库无关，仅供嵌入式任务参考）。

---

## 9. 建议的接手顺序

1. **确认环境是否已续费**（见 §1）——这是所有后续工作的前提。
2. 拿到管理员密码与 publishable key（见 §3），跑一遍 `verify-ai-e2e.cjs`（60 项）确认现状全绿。
3. 从 **#24 功能缺口** 和 **#25 运维监控** 开始（用户明确关心数据完整性与可观测性）。
4. 再做 P7 增强（流式输出优先级最高）。
5. 每完成一阶段：**实测 → 清理测试数据 → 复核数据基线 → 更新 `docs/` 与记忆**。

> 纪律：不虚构测试结果。所有"通过"都必须有脚本输出支撑。
