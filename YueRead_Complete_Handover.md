# 月读 YueRead · 完整项目交接文档

> **交接对象**：Trae（接手开发）
> **交接时间**：2026-11-20
> **文档性质**：从项目最初构想到当前状态的完整移交，含需求原始意图、设计决策、否决方案、技术实测结论、已知问题与后续计划
> **与代码的一致性**：本文所有表名、函数名、路由、action、文件路径均已对照当前代码核实（2026-11-20 13:50）
> **配套材料**：`HANDOFF.md`（快速上手版）、`docs/AI_MODEL_GATEWAY.md`（AI 系统设计与实施细节，§10–§12 为最新）、`docs/ARCHITECTURE.md`、`docs/DATABASE.md`、`docs/PLAN.md`、`docs/RISKS.md`

## 标记约定

| 标记 | 含义 |
|---|---|
| 【用户明确要求】 | 用户原话明确提出，不可擅自更改 |
| 【讨论后确定】 | 双方讨论后达成的方案，有当时的取舍理由 |
| 【技术建议】 | 开发方的技术判断，用户未明确表态，可重新评估 |
| 【不确定】 | 无法从对话或代码中确证，需要向用户确认 |
| ✅ | 已实现并实测通过 |
| 🟡 | 部分实现 |
| ⬜ | 未实现 |

---

# 一、项目最初的设计理念

## 1.1 项目定位

**月读 YueRead** 是一个**个人阅读记录与知识管理平台**。

它的核心不是"再做一个电子书阅读器"，而是：**把微信读书里已经读过的书，转化成可沉淀、可检索、可回顾、可分享的知识资产**。

一句话定位（项目 README 原文）：

> 月有盈亏，阅读不止。让每一本读过的书，都留下思想的痕迹。

## 1.2 要解决的核心问题

微信读书本身是一个很好的阅读工具，但读完之后的数据是"散"的：

- 划线、想法、书评散落在 App 各处，**无法按自己的结构重新组织**；
- 阅读时长、读完书目、阅读趋势**只能看平台给的统计，不能自己定义口径**；
- 想回顾"我去年读了什么、想了什么"时，**没有一份属于自己的年度报告**；
- 想分享阅读感悟，**只能截图**，不能形成结构化的、可长期访问的页面。

月读要做的，是**以用户自己的微信读书数据为原料，重建一层属于用户的知识库**：自己定义的目录、自己写的观点、自己挑的金句、自己记的心得，再配上自己口径的统计与报告。

## 1.3 目标用户

- **主要用户：个人用户（自用为主）**。项目最初就是围绕"一个人管理自己的阅读"设计的，不是面向多人协作或多租户 SaaS。
- 但因为做成了完整的多用户系统（用户名密码注册、RLS 行级隔离），**具备扩展为小范围多人使用的能力**，目前注册是开放的（任何人可注册）。
- 【不确定】是否要做成对外公开运营的产品（涉及备案、内容合规、成本模型），用户未明确表态。这一点 Trae 需要确认。

## 1.4 产品风格与设计理念

【用户明确要求】整体气质是**"月光 / 夜晚 / 安静"**，不是常见的明亮效率工具风。具体表现为：

- **大量留白**，页面不堆信息，阅读类产品要有"呼吸感"；
- **视觉重心是月亮与星空**，配合暖色微光，营造夜间阅读的氛围；
- **高级克制**：可以用渐变和发光，但**不能过度**，不能有"廉价霓虹感"；
- **有生命感的细节**：星星闪烁、月光呼吸、吉祥物微动效，但要轻、要不打扰；
- **响应式优先手机端**：因为阅读记录这种场景，用户多在半躺状态下用手机看。

## 1.5 对产品的期待

- 【用户明确要求】这个项目要能**长期自用**，不是 demo；
- 数据要**真实**（真实绑定微信读书账号、真实同步），不能是假数据撑场面；
- 【用户明确要求】**不因引入 AI 而破坏已有的阅读数据与功能**；
- 【用户明确要求】**不擅自购买额外云服务**，在现有 CloudBase 环境内解决问题；
- 【用户明确要求】成本要可控，尤其是模型调用成本。

---

# 二、全部功能需求

> 说明：本节按功能域组织。每项都标注了实现状态。**"部分实现"与"未实现"的具体缺口见第六章。**

## 2.1 用户账户与身份

| 需求 | 状态 | 细节 |
|---|---|---|
| 用户名 + 密码注册/登录 | ✅ | 【讨论后确定】**放弃微信授权登录**，理由见 2.9 否决方案 |
| 公开注册（未登录可注册） | ✅ | 走 HTTP 网关 `POST /api/auth-register`，`auth=false` |
| 修改密码 | ✅ | 原生 fetch 打 CloudBase Auth 的 `sudo` + `PATCH password` 接口 |
| 修改昵称/头像 | 🟡 | 昵称可改（`profiles` 表）；**头像上传未实现** |
| 退出登录 | ✅ | |
| 多账号数据隔离 | ✅ | PostgreSQL RLS，`owner_id = auth.uid()` |
| 管理员角色 | ✅ | `ai_admin_users` 白名单，role 分 `admin` / `viewer` |

【用户明确要求】不同账号之间**数据完全不可见**，不能出现"看到别人书架"的情况。

## 2.2 微信读书接入

| 需求 | 状态 | 细节 |
|---|---|---|
| 绑定微信读书 API Key | ✅ | 加密存储（AES-256-GCM），`weread_credentials` 表 |
| 解绑 | ✅ | 删除凭证，保留已同步数据 |
| 绑定状态查询 | ✅ | `weread-bind` action=`status` |
| 手动同步 | ✅ | `weread-sync` 默认分支，需登录 |
| 自动定时同步 | ✅ | timer 触发器 `weread-sync-timer`，cron `0 0 * * * * *`（每小时整点） |
| 同步开关（用户可关） | 🟡 | 后端有 `auto_sync_enabled` 字段并被定时任务读取；**前端没有开关 UI** |
| 同步全量历史 | ⬜ | 目前只回溯**近 3 个月**；全量回溯是待办 #17 |

同步的数据范围（已实现）：
- 书架（跳过 `mpbook`）、有声书 `albums[]`
- 章节/目录
- 划线（含 `removed[]` 的已删划线处理，避免脏数据）
- 想法/书评
- 阅读时长（`readingTime`）
- 阅读统计（月度，按 `baseTime` 回溯）

## 2.3 阅读记录管理

| 需求 | 状态 | 细节 |
|---|---|---|
| 书架展示（封面墙） | ✅ | `ShelfView`，支持状态标签：想读 / 在读 / 读完 |
| 书籍详情 | 🟡 | 已有基本信息、目录、金句、心得；**目录时间轴、金句卡片化视觉未完成**（#23） |
| 章节管理 | 🟡 | 可查看；**手动增删章节未实现**（#24） |
| 金句管理 | ✅ | 增删改查，`quotes` 表 |
| 心得/书评 | ✅ | 增删改查，`reflections` 表 |
| 观点（insights） | ✅ | `insights` 表，数据层已实现 |
| 阅读状态切换 | ✅ | want / reading / done |

## 2.4 统计与报告

| 需求 | 状态 | 细节 |
|---|---|---|
| 阅读时长统计 | ✅ | `reading_stats`，支持月度 / 季度 / 年度 |
| 阅读趋势 | ✅ | `ReportView` 有"阅读趋势"区块 |
| 分类分布 | ✅ | `ReportView` 有"分类分布"区块 |
| 月度阅读海报 | ✅ | `PosterView`，可生成月度海报 |
| 季度 / 年度报告 | 🟡 | 数据层支持 `quarterly` / `yearly`；**前端报告页主要以月度为主，季度/年度入口与展示不完整** |
| AI 年度书评 | ✅ | `ai-assistant` action=`review` |

## 2.5 社交分享

| 需求 | 状态 | 细节 |
|---|---|---|
| 生成分享链接 | ✅ | `share-manager` action=`create`，token 化 |
| 公开访问分享页（未登录） | ✅ | 网关 `POST /api/share`，`auth=false` |
| 分享列表 | ✅ | action=`list` |
| 撤销分享 | ✅ | action=`revoke` |
| 分享页视觉 | ✅ | `ShareView` |
| 【用户明确要求】分享不能泄露隐私 | ✅ | 分享走**列级授权**，公开读不到 `owner_id` 等敏感列 |

## 2.6 AI 阅读辅助（月野兔）

【用户明确要求】这是后期新增的重点需求，原始要求如下（尽量保留原意）：

1. 平台管理员可统一配置多个大模型 API（DeepSeek / Kimi / 豆包 / 混元 / GLM 等）；
2. **所有 API Key 只存服务端安全配置，禁止暴露到前端**；
3. 用户可自主选择可用模型，界面要展示**模型名称、可用状态、平台设置的额度信息**；
4. 有统一的模型调用服务，支持切换；
5. **额度不足 / 调用失败 / 服务不可用时自动切换备用模型**；
6. **自动切换前后要向用户说明当前使用的是哪个模型**；
7. 用户可设置**是否允许自动切换**；
8. 管理后台可增删改启停模型、配置调用额度、Token 预算、优先级、每日上限；
9. 记录每用户调用次数、输入/输出 Token、使用模型；管理员可看总消耗、剩余额度估算、调用日志；
10. 防止用户绕过限制消耗他人额度；每用户有频率与额度限制，防恶意刷接口；
11. **支持用户自己增加大模型 API Key，并提供路径**；
12. 区分**供应商官方额度**与**平台内部预算**，分别处理；
13. 保证对话上下文与用户数据安全；
14. **不因自动切换丢失对话历史**；
15. 【用户明确要求】先输出系统架构、数据库设计、模型适配方案和成本控制方案，**确认后分阶段开发，不要直接重写现有项目**。

AI 功能入口（已实现）：

| 入口 | action | 状态 |
|---|---|---|
| 书籍问答 | `ask` | ✅ |
| 自由对话 | `chat` | ✅ |
| 文本润色 | `polish` | ✅ |
| 年度书评 | `review` | ✅ |
| 历史会话列表/详情 | `sessions` / `history` | ✅ |
| 用量查询 | `usage` / `usage-log` | ✅ |
| 额度查询 | `quota` | ✅ |
| 模型目录 | `models` | ✅ |
| 用户设置 | `settings` | ✅ |
| 供应商列表 | `providers` | ✅ |
| 自带 Key 增删 | `user-key` | ✅ |
| 管理后台 | `admin-check` / `admin-models` / `admin-dashboard` / `admin-logs` / `admin-admins` | ✅ |

## 2.7 数据管理

| 需求 | 状态 | 细节 |
|---|---|---|
| 数据导出 | ⬜ | 未实现（#24） |
| 数据恢复/导入 | ⬜ | 未实现（#24） |
| 封面 / 海报上传云存储 | ⬜ | 未实现（#24） |

## 2.8 运维与可观测

| 需求 | 状态 | 细节 |
|---|---|---|
| 同步状态记录 | 🟡 | `sync_runs` 表已建、同步时会写；**前端没有展示** |
| 失败原因记录 | 🟡 | 有 `trigger_type` 等字段；**失败原因归集与展示未做** |
| 定时任务执行记录 | 🟡 | 同上 |
| 云函数错误日志归集 | ⬜ | 未实现（#25） |
| 空态与失败态区分 | ⬜ | 目前部分页面"没数据"与"加载失败"不分（#25） |

## 2.9 ★ 被否决的方案（务必不要重复提出）

> 这一节是为了防止 Trae 走回头路。每条都附了否决理由。

| 被否决的方案 | 否决理由 | 最终采用 |
|---|---|---|
| **微信授权登录** | 需要微信开放平台资质，约 300 元/年，且个人主体审核麻烦 | 用户名 + 密码（CloudBase 原生 Auth） |
| **CloudBase 控制台「自定义模型提供商」** | provider 是**环境级**配置，一个 Key 全用户共用，违背"防止用户消耗他人额度"的要求，也无法按用户计费 | 自建统一 OpenAI 兼容适配层 + Key 加密存 PG + 按用户/模型记账 |
| **首期启用"平台自配通道"（②）** | 需要平台自己掏供应商账单；而环境资源点余额充足（约 3.8 万），托管模型够用 | 首期只做 **① 托管 + ③ 用户自带** |
| **管理后台用 `authz.user.rego` 鉴权** | rego 是**环境级**策略、粒度到云函数，表达不了"同一函数内只有 `admin-*` action 受限"；也无法区分 admin/viewer | `ai_admin_users` 白名单表 |
| **用 CloudBase 文档数据库（NoSQL）** | 该环境 `nosql:false`，且业务强关系（书-章-句-心得）适合关系型 + RLS | PostgreSQL |
| **采购第三方 AI 网关/SaaS** | 用户明确要求不擅自购买额外服务 | 自建 |
| **原 SVG 手绘兔子** | 【用户明确否决】原话："这个月亮兔子不好看" | 改用两张真实插画（夜间为用户提供的原图抠底，白天为同原型 AI 生成） |
| **首页主视觉保留 SVG 月亮圆盘** | 新夜间插画自带满月，再画就"两个月亮" | 主视觉直接用插画，只留背景光晕 |

---

# 三、UI 和视觉设计要求

## 3.1 总体视觉方向

| 要求 | 状态 | 说明 |
|---|---|---|
| 三主题：白昼 / 星空夜间 / 跟随系统 | ✅ | `html[data-theme=day\|night\|auto]`，`color-scheme` + `light-dark()` 单表 token |
| 切换入口在右上角 + 个人中心 | ✅ | `ThemeToggle.vue`（compact / segmented 两种形态）+ ProfileView 内的 segmented |
| 平滑过渡 | ✅ | 切换时 `<html>` 加 `.theme-fade`，520ms 后移除 |
| 本地持久化 | ✅ | localStorage `yuread.theme` |
| **按账号隔离偏好** | ✅ | localStorage `yuread.theme.<uid>`，切账号用各自主题 |
| 无闪烁（FOUC） | ✅ | `initTheme()` 在 `createApp()` 之前调用 |
| 响应式优先（手机端） | 🟡 | 整体做了，但**没有真机回归**（见第六章） |

## 3.2 夜间模式的具体要求

【用户明确要求】：

- **白色月亮作为主视觉**，带光晕、月光呼吸感；
- **星星闪烁**；
- 卡片用**深蓝灰**，不能纯黑；
- **必须保证文字对比度**可读；
- 用 CSS / SVG / 轻量图形实现，**不依赖大图片**（注：兔子插画是后来用户主动提供的例外）。

实现情况：

| 要求 | 状态 | 实现位置 |
|---|---|---|
| 月亮主视觉 | ✅ | `MoonHero.vue`（已改为直接用插画） |
| 光晕/呼吸 | ✅ | CSS 动画，`prefers-reduced-motion` 下降级 |
| 星星闪烁 | ✅ | `StarField.vue`，78 颗**确定性伪随机**星（避免每次渲染跳动） |
| 深蓝灰卡片 + 对比度 | ✅ | `tokens.css` 中的 `light-dark()` 双值 token |
| 桌面/手机适配 | 🟡 | 布局自适应，但未真机验证 |

## 3.3 页面级设计要求

| 页面 | 设计要求 | 状态 |
|---|---|---|
| 首页 | 月亮主视觉 + 星星 + 统计概览 + 封面墙 + 最近阅读/心得/金句 | ✅（区块已核实存在） |
| 书架 | 封面墙 + 状态标签 | ✅ |
| 书籍详情 | 目录时间轴 + 金句卡片 + 心得展示 | 🟡 数据齐，**时间轴/卡片化视觉未完成** |
| 阅读报告 | 月相装饰 + 趋势可视化 | 🟡 趋势与分类分布已有，月相装饰未做 |
| 个人中心 | 主题设置 + 阅读成就 + 头像 | 🟡 主题设置 ✅，**头像与成就未做** |
| 月度海报 | 可生成、可分享 | ✅ |

## 3.4 「月野兔」吉祥物

### 形象定位

【用户明确要求】月读的专属吉祥物叫**月野兔**，是"月宫玉兔"的现代可爱化演绎，定位是**陪伴阅读的小伙伴**，不是品牌 Logo。

### ★ 用户对旧形象的不满（重要，别重蹈覆辙）

- 第一版是**纯 SVG 手绘**的兔子。
- 【用户明确否决】原话："这个月亮兔子不好看"，并**提供了一张参考图**。
- 参考图内容（已核实）：**奶白色胖兔子、粉色内耳、眯眼笑、双手捧着一本蓝色月亮封面的书、坐在淡紫色云朵上、背后是一轮暖黄色满月，纯白背景**。
- 【用户明确要求】这张图**作为夜间模式**的形象；**白天模式以这只兔子为原型，设计一个新的场景**。

### 最终实现

| 项 | 实现 |
|---|---|
| 夜间素材 | 用户原图 → 抠掉白底（保留兔子+云+月亮）→ `web/src/assets/rabbit-night.webp` |
| 白天素材 | 以夜间图为参考做图生图 → 同形象坐书堆/蓝天暖阳 → 去水印 → 抠蓝天 → `rabbit-day.webp` |
| 格式优化 | PNG（各 1.2 MB）转 WebP（107 KB / 74 KB），母版 PNG 保留 |
| 组件 | `MoonRabbit.vue`，`variant=auto\|night\|day` |
| 互动 | 点击弹寄语（4 档 mood，按阅读状态给不同鼓励） |
| 动效 | CSS 呼吸 / 蹦跳，`prefers-reduced-motion` 下降级 |
| 约束 | 【用户明确要求】**不遮挡文字与按钮** |

## 3.5 明确认可 / 否定的清单

| 类型 | 内容 |
|---|---|
| ✅ 认可 | 三主题系统、月亮+星星的夜间氛围、新兔子插画（用户提供原图后确认采用）、整体克制的留白风格 |
| ❌ 否定 | 原 SVG 兔子形象；首页保留 SVG 月亮圆盘（会导致双月） |
| 【不确定】 | 白天兔子是 AI 生成的，用户尚未明确评价"是否满意"，**需要主动询问** |

---

# 四、技术架构和开发细节

> **重要**：本节只描述**实际已实现**的技术。计划但未做的，明确标注 ⬜。

## 4.1 技术栈（实际）

| 层 | 选型 | 实际状态 |
|---|---|---|
| 前端 | Vue 3 + TypeScript + Vite + Vue Router + Pinia | ✅ |
| 路由模式 | **hash 模式**（`createWebHashHistory`） | ✅ 静态托管刷新不 404 |
| 样式 | 原生 CSS + CSS 变量（`tokens.css`），无 UI 框架 | ✅ |
| 测试 | Vitest + happy-dom（组件）、独立 Node 脚本（端到端） | ✅ |
| 数据库 | **CloudBase PostgreSQL**（非文档数据库） | ✅ |
| 后端 | CloudBase 云函数（Nodejs18.15）+ HTTP 网关 | ✅ |
| 存储 | CloudBase 云存储（封面/海报） | 🟡 表结构有，上传未做 |
| 部署 | CloudBase 静态网站托管 | ✅ |

## 4.2 CloudBase 环境（实际）

| 项 | 值 |
|---|---|
| EnvId | `lexi-d9gb3ruct4fdb9d0e` |
| AppId | `1512381244` |
| 地域 | `ap-shanghai` |
| 套餐 | 个人版（`RuntimeMode=postgresql`，`nosql:false`） |
| 静态站点 | `https://lexi-d9gb3ruct4fdb9d0e-1512381244.tcloudbaseapp.com/` |
| 网关域名 | `https://lexi-d9gb3ruct4fdb9d0e-1512381244.ap-shanghai.app.tcloudbase.com` |
| ⚠️ 到期 | **2026-11-21 23:59:59**，`IsAutoRenew=false` |
| 资源点 | 总额 40,000，上次实测剩约 38,660 |

> ⚠️ **envId 必须管道取权威值，永不手打**。历史上曾因手打 `b3truct`（正确是 `b3ruc`）误判"环境已销毁"。
> ```bash
> tcb env list -r ap-shanghai --json | node -e 'let s="";process.stdin.on("data",d=>s+=d).on("end",()=>{const j=JSON.parse(s);process.stdout.write(j.data.find(e=>e.PackageName==="个人版").EnvId)})'
> ```

## 4.3 数据库设计（实际 23 张表）

### 业务表（10 张）

| 表 | 用途 | 关键字段 |
|---|---|---|
| `profiles` | 用户资料 | `owner_id`, `nickname`, `avatar_url` |
| `books` | 书目 | `owner_id`, `book_id`, `title`, `author`, `cover_url`, `status`(want/reading/done), `finish_reading`, `progress` |
| `chapters` | 章节/目录 | `owner_id`, `book_id`, `title`, `level`, `order_index` |
| `insights` | 观点 | `owner_id`, `book_id`, `content` |
| `quotes` | 金句/划线 | `owner_id`, `book_id`, `content`, `chapter_title`, `note` |
| `reflections` | 心得/书评 | `owner_id`, `book_id`, `content`, `mood` |
| `reading_stats` | 阅读统计 | `owner_id`, `period_type`(monthly/quarterly/yearly), `period_key`, `duration`, `book_count` |
| `shares` | 分享 | `token`, `owner_id`, `target_type`, `target_id`, `expires_at` |
| `weread_credentials` | 微信读书凭证 | `owner_id`, `api_key_cipher/iv/tag`, `status`, `auto_sync_enabled` |
| `sync_runs` | 同步执行记录 | `owner_id`, `trigger_type`, `status`, 各类计数 |

### AI 表（13 张）

| 表 | 用途 | 敏感度 |
|---|---|---|
| `ai_models` | 模型目录（单价/额度/优先级/状态） | 公开读 |
| `ai_api_keys` | 平台/用户 Key 密文 | 🔴 **对 authenticated/anon 一律不授权** |
| `ai_user_models` | 用户自带模型 | 仅本人 |
| `ai_user_settings` | 用户偏好与日/月额度 | 仅本人 |
| `ai_sessions` | 会话 | 仅本人 |
| `ai_messages` | 消息 | 仅本人 |
| `ai_usage_logs` | 每次调用明细 | 仅本人；管理员可读 |
| `ai_usage_daily` | 用户维度日聚合 | 仅本人 |
| `ai_model_daily` | 模型维度日消耗（看板） | 管理端 |
| `ai_rate_windows` | 分钟级频控窗口 | 服务端 |
| `ai_admin_users` | 管理员白名单 | `owner_id`, `role`(admin/viewer) |
| `ai_period_guard` | 原子额度闸门 | 🔴 **不授权给前端** |
| `ai_chats` | ⚠️ **旧表**，AI 早期版本的对话表，保留并继续写 | 仅本人 |

### 数据库函数（7 个，实际）

| 函数 | 用途 |
|---|---|
| `ai_guard_reserve` | 调用前原子预扣额度 |
| `ai_guard_release` | 被拒时回滚预扣 |
| `ai_guard_settle` | 调用后用实际消耗结算 |
| `ai_rate_hit` | 分钟窗口计数 |
| `ai_purge_rate_windows` | 清理过期窗口 |
| `ai_bump_user_daily` | 用户日聚合累加 |
| `ai_bump_model_daily` | 模型日聚合累加 |

> 全部显式 `REVOKE EXECUTE ... FROM PUBLIC`（`CREATE FUNCTION` 默认授予 PUBLIC，不收回会让人能用别人 uid 篡改额度）。
> 全部显式逐表 `GRANT`，**禁止** `GRANT ... ON ALL TABLES IN SCHEMA public`（共享环境会波及他人）。

### 数据基线（每轮测试后必须复核）

```
books 33 / chapters 1125 / quotes 125 / reflections 13
shares 2 / weread_credentials 1 / reading_stats 7
auth.users 只应有 2 个：administrator + licy
ai_models 49 行（不应有 e2e% 夹具）
```

## 4.4 云函数（实际 5 个）

| 云函数 | 超时 | 内存 | action | 触发方式 |
|---|---|---|---|---|
| `auth-register` | 30s | 256MB | `register`（默认）/ `check` | HTTP 网关（公开） |
| `weread-bind` | 60s | 256MB | `bind` / `status` / `unbind` | 云函数直调（需登录） |
| `weread-sync` | 300s | 512MB | `manual`（默认）/ `auto` | 直调 + **timer 每小时** |
| `share-manager` | 30s | 256MB | `create` / `list` / `revoke` / `view` | 网关（公开）+ 直调 |
| `ai-assistant` | 120s | 512MB | 18 个（见 2.6） | 云函数直调（需登录） |

## 4.5 API 接口

### HTTP 网关路由（仅 2 条公开）

| 路由 | 上游 | 鉴权 |
|---|---|---|
| `POST /api/auth-register` | `auth-register` | `auth=false` |
| `POST /api/share` | `share-manager` | `auth=false` |

> 【技术事实】未登录直调 `/v1/functions/<name>` 会被平台注入层拦死（403 `EXCEED_AUTHORITY`），与 OPA 策略无关。所以**公开入口只能走网关**。

### 前端调用层（实际文件）

| 文件 | 职责 |
|---|---|
| `web/src/lib/cloudbase.ts` | SDK 单例（39 行） |
| `web/src/lib/functions.ts` | 云函数调用封装（35 行） |
| `web/src/lib/auth-api.ts` | 原生 fetch：注册 / 改密（167 行） |
| `web/src/lib/share-api.ts` | 分享接口（86 行） |
| `web/src/lib/ai-api.ts` | AI 旧调用层（154 行） |
| `web/src/lib/ai-models.ts` | AI 新调用层（293 行，第二代） |
| `web/src/lib/ai-admin.ts` | 管理后台接口（249 行） |
| `web/src/lib/markdown.ts` | Markdown 渲染（126 行） |
| `web/src/api/data.ts` | 业务数据访问（516 行） |
| `web/src/stores/auth.ts` | 登录态（76 行） |
| `web/src/stores/ai.ts` | AI 状态（203 行） |

## 4.6 前端路由（实际 11 条）

| 路径 | 页面 | 需登录 |
|---|---|---|
| `/` | 首页 | ✅ |
| `/shelf` | 我的书架 | ✅ |
| `/book/:id` | 书籍详情 | ✅ |
| `/reports` | 阅读报告 | ✅ |
| `/poster` | 月度海报 | ✅ |
| `/profile` | 个人中心 | ✅ |
| `/ai` | AI 助手设置 | ✅ |
| `/admin` | 管理后台 | ✅（页面内再判角色） |
| `/login` | 登录 | ❌ |
| `/share/:token` | 分享页 | ❌ |
| `/:pathMatch(.*)*` | 404 | ❌ |

## 4.7 大模型接入方式（实际）

**三通道设计**，首期只启用 ①③：

| 通道 | provider | Key 存放 | 计费方 | 实现状态 |
|---|---|---|---|---|
| ① 托管 | `cloudbase` | 平台（无需 Key） | **平台资源点** | ✅ |
| ② 平台自配 | 各厂商 | 平台加密存 PG | 平台供应商账单 | ⬜ 首期不做 |
| ③ 用户自带 | 各厂商 | 用户加密存 PG | 用户自己的厂商账户 | ✅ |

**实际默认配置（线上查得）**：`is_default=true` 的模型是 **`qwen3.5-flash`**，单价 **0.2 点/千 token 输入、2 点/千 token 输出**，`daily_quota_points = null`（不限）。

**计费公式（已实测校准）**：

```
points = input/1000 × price_input + output/1000 × price_output + cache/1000 × price_cache
```

实测：`qwen3.5-flash` 一次调用 usage={input 60, output 778, cache 838}，扣 1.568 点，与公式误差 0。

**三层预算**：

| 层 | 含义 | 落库 |
|---|---|---|
| L1 供应商官方额度 | CloudBase 资源点 / 用户厂商余额 | 不落库，看账单 |
| L2 平台内部预算 | 平台给每个模型设的上限 | `ai_models.daily_quota_*` |
| L3 用户额度 | 每用户日/月上限 | `ai_user_settings.*` |

> ⚠️ **0 与 NULL 语义（最隐蔽的坑）**：
> - `ai_models.daily_quota_points = 0` → **禁用该模型**（可自动切换）；`NULL` = 不限
> - `ai_user_settings.daily_point_limit = 0` → **一律拒绝**；`NULL` = 不限
> - `total_quota_points = 0/NULL` → 不限
> - `rate_limit_per_min = 0` → 不限流
> - 前两类**必须在进 SQL 前用 JS 判定**，否则静默变"无限制"，白烧点。

**自动切换规则**：错误分 6 类（config / params / quota / rate_limit / timeout / upstream），只有 **params 与 config 不自动切换**，其余切备用。切换前后必须告知用户。失败尝试只写 `ai_usage_logs(status='failed')`，**不写 `ai_messages`**，所以不丢历史。

**唯一出口纪律**【技术建议，但已作为硬约束】：`ai-assistant/index.js` 不允许直接调 `app.ai()` 或 fetch 厂商，必须走 `lib/orchestrator.js` 的 `invokeWithFallback()`。

## 4.8 微信读书数据获取（实际，官方文档有 3 处不实）

| 实测结论 | 说明 |
|---|---|
| 阅读时长读 `book.readingTime` | **不是** `book.recordReadingTime`（后者恒 0） |
| 已读完看 `finishReading === 1` | 有书 `finishReading=1` 但 `progress=94`，只看进度会漏判 |
| `/book/bookmarklist` 有 `removed[]` | 已删划线，不处理会留脏数据 |
| `/readdata/detail` 只支持 `mode=monthly` | 季/年靠 `baseTime` 回溯后本地聚合 |
| 时间戳按东八区日界 | 所有日/月/分钟桶按北京时间切 |
| 书架 `mp`（bookId='mpbook'）跳过 | |
| 错误 Key → `{"errcode":-2013}`（HTTP 401） | |

字段映射集中在 `cloudfunctions/weread-sync/lib/extract.js`。

## 4.9 用户身份认证（实际）

- CloudBase 原生 Auth（用户名 + 密码）；
- 注册走 `auth-register` 云函数，用管理端 `CreateUser` 建 **`externalUser`**（个人版 `internalUser` 只有 3 个配额，会触顶）；
- 改密码不用 SDK 的 `User.updatePassword()`（必抛 `invalid 【patch】 request method`），改用原生 fetch：`POST /auth/v1/user/sudo` → `PATCH /auth/v1/user/password`；
- 前端直连受 RLS 保护；云函数用 `service_role`，**RLS 对云函数不生效**。

## 4.10 环境变量与配置（实际）

| 位置 | 内容 | 是否入库 |
|---|---|---|
| `cloudbaserc.json` | envId、5 个函数定义、timer 触发器 | ✅ 入库（无密钥） |
| `web/.env.example` | 模板：`VITE_CLOUDBASE_ENV_ID` / `VITE_CLOUDBASE_GATEWAY_URL` / `VITE_CLOUDBASE_PUBLISHABLE_KEY` | ✅ 入库 |
| `web/.env.local` | 真实 publishable key | ❌ **已 gitignore** |
| `cloudfunctions/.secrets/` | 微信读书 API Key、`WEREAD_ENC_KEY` | ❌ **已 gitignore** |
| 云端函数环境变量 | `WEREAD_ENC_KEY` 等 | 只在云端 |

## 4.11 部署方式（实际）

```bash
# 取权威 envId
tcb env list -r ap-shanghai --json | node -e '...'

# 部署单个云函数（覆盖必须 --force）
tcb fn deploy ai-assistant --force -e "$ENV" -r ap-shanghai

# 前端构建 + 上传静态托管
cd web && npm run build
tcb hosting deploy dist -e "$ENV" -r ap-shanghai
```

> 已知现象：`tcb hosting deploy` 末尾 CLI 会调 `reg.exe`，可能被安全策略拦，但**上传本身已成功**。用线上 HTTP 200 + 主包哈希核对，别误判失败。CDN 可能缓存旧哈希，加 `-H "Cache-Control: no-cache"` 复核。

---

# 五、重要注意事项（用户反复强调）

> 这些是用户的**硬约束**，不是建议。违反会被明确纠正。

| # | 约束 | 来源 | 说明 |
|---|---|---|---|
| 1 | **不要直接重写现有项目** | 【用户明确要求】 | 用户原话："先输出系统架构、数据库设计、模型适配方案和成本控制方案，确认后分阶段开发。不要直接重写现有项目。" |
| 2 | **不要破坏已有功能** | 【用户明确要求】 | 每次改动都要回归已有链路 |
| 3 | **保护用户 API Key** | 【用户明确要求】 | 只存服务端加密，前端连密文都读不到 |
| 4 | **不泄露用户隐私** | 【用户明确要求】 | 分享页列级授权；日志不记 prompt/answer |
| 5 | **保留现有 CloudBase 环境** | 【用户明确要求】 | 不擅自换环境、不新建环境 |
| 6 | **不随意修改数据库** | 【用户明确要求】 | 改表要写迁移、逐表授权、先备份思路 |
| 7 | **不过度复杂化** | 【用户明确要求】 | 能简单解决不要上架构 |
| 8 | **控制开发成本与调用成本** | 【用户明确要求】 | 默认选最便宜模型；不擅自买服务 |
| 9 | **每阶段必须实测，不虚构结果** | 【用户明确要求】 | 这是贯穿全程的纪律，说"通过"必须有脚本输出 |
| 10 | **自动切换要告知用户** | 【用户明确要求】 | 切换前后说明当前模型 |
| 11 | **不因切换丢对话历史** | 【用户明确要求】 | messages 原样传递，失败只记 log 不写 message |
| 12 | **区分供应商额度与平台预算** | 【用户明确要求】 | UI 上必须分开显示 L1 / L2 |

---

# 六、当前版本的问题

## 6.1 🔴 紧急：环境即将到期

- 环境 **2026-11-21 23:59:59 到期**，`IsAutoRenew=false`，**就在明天**。
- 不续费：云函数、网关、静态托管、登录全部停服；**数据不丢**，续费后同 envId 立即恢复。
- 续费 ¥39.9/月，需用户确认后才能操作。
- **Trae 接手第一件事必须确认这个**，否则改了也访问不了。

## 6.2 已知 Bug / 配置隐患

| # | 问题 | 严重度 | 详情 |
|---|---|---|---|
| 1 | **所有模型的 L2 平台预算都是 `null`（不限）** | 🟡 中 | 线上 `ai_models.daily_quota_points` 全为 NULL → 平台内部安全闸形同虚设，只剩用户层额度兜底。建议给热门模型设合理日预算。 |
| 2 | `pricing.js` 里 `DEFAULT_MODEL = 'hy3'` 与线上默认（`qwen3.5-flash`）不一致 | 🟡 中 | 线上默认由 `ai_models.is_default` 决定；代码常量若被当兜底会造成偏差。需确认 `DEFAULT_MODEL` 的实际引用点。 |
| 3 | 两代 AI 调用层并存 | 🟡 中 | `ai-api.ts`（旧）+ `ai-models.ts`（新）；两套逻辑容易改一处漏一处。建议收敛到一套。 |
| 4 | 旧表 `ai_chats` 与新表 `ai_sessions`/`ai_messages` 并存 | 🟢 低 | `ai_chats` 仍在写入，属于历史包袱。 |
| 5 | 探针脚本仍带夹具密码 `MoonRead2026` | 🟢 低 | `probe-identity-matrix.cjs`、`verify-*.cjs` 里给**临时注册账号**用，非真实凭据，但最好也参数化。 |
| 6 | `probe-*.cjs` 是调试脚本，未清理 | 🟢 低 | 6 个 probe 脚本是排障时写的，可归档。 |
| 7 | 独立跑 Node 脚本调 SDK 常被 SIGTERM 卡死 | 🟢 低 | 已知现象，放进 e2e 里跑就正常（e2e 会先发一次网关请求）。 |

## 6.3 页面显示 / 交互问题

| # | 问题 | 详情 |
|---|---|---|
| 1 | **无真机视觉回归** | 本机浏览器二进制被网络策略拦（Chromium 下载 TLS handshake eof、Edge 进程被 SIGTERM），三主题全站效果**没有人工过一遍**。已用 vitest + happy-dom 做 DOM 断言（47/47），但**不等于视觉正确**。需用户自己在浏览器检查。 |
| 2 | 书籍详情视觉未完成 | 目录时间轴、金句卡片、心得展示区仍是基础样式 |
| 3 | 个人中心缺头像与成就 | 只有昵称与主题设置 |
| 4 | 空态与失败态不分 | 部分页面"没数据"和"加载失败"显示一样，用户无法判断是没内容还是出错 |

## 6.4 未实现模块（汇总）

- P7 增强：流式输出、批量任务、平台自配 provider 通道
- 数据导出/恢复、封面/海报上传云存储、分享管理列表前端、章节手动增删
- 同步状态/失败原因前端展示、定时任务执行记录、错误日志归集
- 微信读书全量历史回溯（目前只近 3 个月）
- 季度/年度报告的完整前端呈现

## 6.5 技术债务

- 两代 AI 调用层并存（见 6.2 #3）
- `ai_chats` 旧表未迁移
- `DEFAULT_MODEL` 常量与线上配置不一致（见 6.2 #2）
- 探针脚本未整理
- TaskList 状态与实际进度有偏差（部分已完成任务仍标 pending），**不要以 TaskList 为准，以本文与代码为准**

## 6.6 已尝试但没解决的问题

| 问题 | 尝试过什么 | 结论 |
|---|---|---|
| 浏览器真机截图/回归 | ① `agent-browser install` 下载 Chrome → TLS handshake eof；② 用本机 Edge `--executable-path` → 进程被环境 SIGTERM | **环境层面拦死**，只能改用 happy-dom DOM 断言；真机视觉需用户手动确认 |
| 独立脚本登录卡死 | 直接 `node scripts/xxx.cjs` 登录 → 卡住超时 | 放进 e2e 主流程里跑就正常 |
| 管理后台日志 0 条 | 查 RLS、查授权、查角色，均正常 | 真因是 `event.action` 与日志过滤字段撞名，已改用 `logAction` 修复 |

---

# 七、后续开发计划（按优先级）

## P0 · 明天之前必须处理

### 任务 0：确认环境续费
- **目标**：确保 11-21 之后站点仍可用
- **动作**：向用户确认是否续费 → `manageEnv action=renew`
- **注意**：**必须用户确认**，不可自行扣费

## P1 · 高优先级

### 任务 1：补齐云端功能缺口（原 #24）
- **目标**：数据导出/恢复、封面上传云存储、自动同步开关、分享管理列表与撤销、AI 历史/用量面板、章节手动增删
- **涉及页面**：ProfileView、ShelfView、BookDetailView、AiSettingsView、ShareView
- **可能涉及文件**：`web/src/api/data.ts`、`web/src/lib/share-api.ts`、`web/src/views/*.vue`、`share-manager/index.js`、新增导出云函数
- **技术难点**：
  - 导出要包含用户全部数据（books/chapters/quotes/reflections），格式建议 JSON + Markdown
  - 封面上传需接 CloudBase 云存储 + 权限校验
  - 章节手动增删要考虑与微信读书同步的冲突（同步会覆盖吗？需设计"手动优先"标记）
- **注意**：导出属于敏感操作，要有明确的范围确认

### 任务 2：运维监控与全站错误处理（原 #25）
- **目标**：同步状态/失败原因展示、定时任务执行记录、错误日志归集、空态与失败态区分
- **涉及文件**：`sync_runs` 表已有、`ProfileView.vue`、各 View 的加载态
- **技术难点**：区分"空"与"错"需要统一的请求状态机（idle/loading/success/empty/error）
- **注意**：`sync_runs` 表已建但前端未用，先查清字段

## P2 · 中优先级

### 任务 3：P7 增强
- **3.1 流式输出**（优先级最高）
  - 目标：AI 回答逐字输出，改善等待体验
  - 涉及：`ai-assistant`、`model-gateway.js`、`orchestrator.js`、`BookDetailView.vue`
  - 难点：**云函数 + 网关不支持原生 SSE**，需评估"轮询 + 分段"方案；费用记账要在流结束后结算
  - 注意：流式下自动切换更复杂（已吐出的内容怎么办），建议首期流式**不自动切换**
- **3.2 批量任务**：批量生成金句摘要/批量润色
- **3.3 平台自配 provider 通道**：需用户确认是否愿意承担供应商账单

### 任务 4：微信读书全量历史回溯（原 #17）
- **目标**：从"近 3 个月"扩展到可配置的全量
- **涉及**：`cloudfunctions/weread-sync/index.js`、`lib/extract.js`
- **难点**：`baseTime` 逐年回溯 + `mode=monthly` 循环，**300s 超时是硬约束**，需分批/断点续传
- **注意**：全量会显著增加出网流量与数据库容量，先与用户确认成本可接受

### 任务 5：全站视觉升级收尾（原 #23）
- 书籍详情：目录时间轴、金句卡片、心得展示
- 个人中心：头像、阅读成就
- 阅读报告：月相装饰
- 涉及：`BookDetailView.vue`、`ProfileView.vue`、`ReportView.vue`、`tokens.css`

## P3 · 低优先级

### 任务 6：云端现状审计（原 #20）
- 目标：输出当前架构图、问题清单（带文件行号）、分阶段升级方案
- 注意：**不虚构任何 API 或部署状态**，全部实查

### 任务 7：技术债清理
- 收敛两代 AI 调用层
- 统一 `DEFAULT_MODEL` 来源
- 整理 probe 脚本
- 考虑 `ai_chats` 迁移

## 贯穿所有任务的要求

1. 改完必须**实测**，贴出脚本输出，不虚构
2. 测试后**清理测试数据**，复核数据基线（见 4.3）
3. 改动涉及数据库要**写迁移文件**，不要手工改线上
4. 部署后**核对线上产物**（HTTP 200 + 哈希）

---

# 八、交接清单

## 8.1 已交付给 Trae 的材料

| 文件 | 用途 |
|---|---|
| `YueRead_Complete_Handover.md`（本文） | 完整交接 |
| `HANDOFF.md` | 快速上手版（含凭据获取、平台坑、验证命令） |
| `TRAE_START_HERE.md` | 一页纸接手指引 |
| `docs/AI_MODEL_GATEWAY.md` | AI 系统设计（§10–§12 最新） |
| `docs/ARCHITECTURE.md` | 技术架构与环境实测 |
| `docs/DATABASE.md` | 数据库设计 |
| `docs/PLAN.md` | 六阶段开发计划 |
| `docs/RISKS.md` | 技术风险 |
| `README.md` | 项目简介（已更新指向 HANDOFF） |
| `yuread-github.zip` | 可直接上传 GitHub 的干净工程包（116 文件） |

## 8.2 凭据交接（不入库）

| 凭据 | 获取方式 |
|---|---|
| 管理员账号 | 用户名 `licy`，**密码由用户线下提供**（本文不写明文） |
| `licy` 的 uid | `2123393937498513410`（19 位，手打易错，建议用 SQL 查） |
| publishable key | CloudBase 控制台 → 环境 → 全局设置 → 客户端 SDK；填进 `web/.env.local` |
| 微信读书 API Key | 用户提供；真实值只在 `cloudfunctions/.secrets/` 与云端环境变量 |
| `WEREAD_ENC_KEY` | 云端函数环境变量，**不要动**（动了旧密文全解不开） |

## 8.3 验证脚本（改动对应模块后必须跑）

| 命令 | 项数 | 说明 |
|---|---|---|
| `node web/scripts/verify-auth-gateway.cjs` | 12 | 登录端到端 |
| `node web/scripts/verify-weread-mapping.cjs` | 32 | 真打微信读书接口 |
| `node web/scripts/verify-share-logic.cjs` | 32 | 纯离线 |
| `node web/scripts/verify-share-e2e.cjs` | 11 | 分享端到端 |
| `node web/scripts/verify-ai-gateway.cjs` | 103 | AI 网关离线 |
| `node web/scripts/verify-ai-orchestrator.cjs` | 83 | 编排离线 |
| `node web/scripts/verify-ai-admin.cjs` | 79 | 管理后台离线 |
| `AI_E2E_ADMIN_USER=licy AI_E2E_ADMIN_PW='<密码>' node web/scripts/verify-ai-e2e.cjs` | 60 | 真调云端+模型 |
| `cd web && npx vitest run` | 47 | 前端 DOM 渲染 |
| `cd web && npx vue-tsc --noEmit` | — | 类型检查 |

## 8.4 不确定、需要向用户确认的事项

| # | 待确认 | 为什么重要 |
|---|---|---|
| 1 | **是否续费环境**（11-21 到期） | 不续费明天就停服 |
| 2 | **白天版兔子是否满意** | 是 AI 生成的，用户还没正式评价 |
| 3 | **是否要做成对外公开运营的产品** | 涉及备案、合规、成本模型 |
| 4 | 季度/年度报告的具体呈现形式 | 数据层有，前端未定 |
| 5 | 全量历史回溯是否值得增加流量成本 | 800 点/GB 出网 |
| 6 | 是否启用平台自配通道（②） | 要平台掏供应商账单 |
| 7 | 头像/成就的具体设计 | 只有方向，没有细节 |

---

# 附录 A：关键平台坑速查（Trae 必读）

> 这些全部是实测得出的，网上查不到，踩一次要花几小时。

1. **未登录调 `/v1/functions/<name>` 被平台拦死**（403 EXCEED_AUTHORITY），公开入口只能走网关。
2. **`internalUser` 受套餐配额限制**（个人版 3 个触顶），注册一律 `externalUser`。
3. **`User.updatePassword()` 必抛** `invalid 【patch】 request method` → 用原生 fetch 打 `sudo` + `PATCH`。
4. **网关转发 Event 函数**时 event 变 `{body:"<json>"}`，函数要归一化。
5. **`enableSafeDomain:true` 后网关自己处理 CORS**，函数别再设 `Access-Control-Allow-Origin`。
6. **PG 环境 `ModifyResourcePermission` 被拒**，权限只能走 `authz.user.rego`。
7. **Node 下多个 SDK `init()` 实例共享 auth 会话** → 切身份前必须重新登录（登录是覆盖式的）。
8. **`tcb fn deploy` 覆盖用 `--force`**；部署**不清**线上未声明的环境变量。
9. **`tcb fn env` 只能 pull 不能 set** → 设变量用 MCP `manageFunctions(updateFunctionConfig)`。
10. **`applyMigration` 会校验本地文件与线上 SQL 一致**，不一致 fail closed。
11. **共享环境禁止 `GRANT ... ON ALL TABLES IN SCHEMA public`**，会波及别人项目。
12. **CloudBase Uid 是 int64（19 位）**，JSON/JS 管理接口会丢精度，管用户只能直接写 PG。
13. **`auth.token.user_id` 是 varchar，`auth.users.id` 是 bigint** → JOIN 要 `u.id::text = t.user_id`。
14. **`service_role` 下 RLS 完全不生效**（云函数就是这条通道）；前端直连才受 RLS 保护。
15. **`app.rdb()` 在 PG 返回 `PostgrestClient`（有 `.rpc()`）**；`CREATE FUNCTION` 默认授 PUBLIC，必须 `REVOKE`。
16. **PostgREST 按参数名匹配实参**，写错是静默 404 / PGRST202。
17. **路由名 `action` 不能与业务过滤字段同名**，否则 `WHERE action='admin-logs'` 永远 0 条；前端 `invoke` 必须 `{ ...data, action }`。
18. **MCP `previewRows` 只回前 5 行**，查全量用 `string_agg` 聚合。

---

# 附录 B：本交接文档的可信度说明

- **与代码核实过的**：表名（23 张）、数据库函数（7 个）、云函数 action（18+3+1+4+1）、前端路由（11 条）、文件行数、cloudbaserc 配置、线上 `ai_models` 默认配置与单价。
- **来自开发记录的**：所有"实测"结论（计费误差 0、微信读书 3 处字段偏差、平台坑）。
- **来自用户原话的**：第二节加【用户明确要求】标记的条目、第三节 3.4 对兔子的不满。
- **【不确定】的**：1.3 是否公开运营、1.5 长期规划细节、7 章各任务的优先级权重（是我基于"用户关心数据完整性与成本"做的排序建议，用户可调整）。
- **未包含的**：任何明文密码、API Key、JWT。凭据一律标注"向用户索取"。
