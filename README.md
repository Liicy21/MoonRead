# 月读 YueRead

> 月有盈亏，阅读不止。让每一本读过的书，都留下思想的痕迹。

个人阅读记录与知识管理平台。绑定微信读书账号，把阅读数据同步成可沉淀的
目录、观点、金句与心得；也支持导入自己的 TXT 电子书在站内阅读，或让 AI
按书名整理一份内容提要。

**使用者请看 [`USAGE.md`](./USAGE.md)** —— 从注册、导入书籍到月兔聊天的完整说明。

> **接手本项目请按此顺序读**：
> 1. **`TRAE_START_HERE.md`** —— 一页纸：先读什么、遵守什么、先问什么
> 2. **`YueRead_Complete_Handover.md`** —— 完整交接：需求来龙去脉、设计决策、
>    否决方案、技术实测、已知问题、后续计划
> 3. **`HANDOFF.md`** —— 凭据获取、平台坑速查、验证命令
> 4. `docs/AI_MODEL_GATEWAY.md` —— AI 系统设计（§10–§12 为最新实施现状）

## 技术栈

| 层 | 选型 | 说明 |
|---|---|---|
| 前端 | Vue 3 + TypeScript + Vite + Vue Router + Pinia | H5，响应式优先，hash 路由 |
| 数据库 | CloudBase PostgreSQL | **23 张表**（业务 10 + AI 13），RLS 行级隔离 |
| 后端 | CloudBase 云函数 + HTTP 网关路由 | **5 个云函数** + 2 条公开路由 |
| AI | 多模型网关（托管 + 用户自带 Key） | 统一适配层、自动切换、额度闸门、BYOK、管理后台 |
| 视觉 | CSS 变量 + `light-dark()` | 三主题：白昼 / 星空夜间 / 跟随系统 |
| 存储 | CloudBase 云存储 | ⬜ 封面/海报上传**尚未实现** |
| 部署 | CloudBase 静态网站托管 | |

## 目录结构

```
yuread/
├── TRAE_START_HERE.md            接手指引（一页纸）
├── YueRead_Complete_Handover.md  完整交接文档
├── HANDOFF.md                    凭据 / 平台坑 / 验证命令
├── docs/
│   ├── AI_MODEL_GATEWAY.md       AI 多模型系统设计与实施
│   ├── ARCHITECTURE.md           技术架构（环境实测 + API 契约）
│   ├── DATABASE.md               数据库设计
│   ├── PLAN.md                   六阶段开发计划
│   └── RISKS.md                  技术风险
├── web/                          Vue 3 前端
│   ├── src/
│   │   ├── api/data.ts           业务数据访问层
│   │   ├── lib/                  cloudbase / functions / auth / share / ai 调用层
│   │   ├── stores/               auth / ai（Pinia）
│   │   ├── components/           MoonHero / MoonRabbit / StarField / ThemeToggle / Ai*
│   │   ├── views/                11 个页面
│   │   └── styles/tokens.css     三主题设计 token
│   ├── scripts/                  真实验证脚本（e2e / 探针，非 mock）
│   └── tests/                    vitest + happy-dom 组件测试
├── cloudfunctions/
│   ├── auth-register/            用户名注册（externalUser）
│   ├── weread-bind/              API Key 绑定 / 解绑 / 状态
│   ├── weread-sync/              数据同步（手动 + 每小时定时）
│   ├── share-manager/            分享创建 / 列表 / 撤销 / 查看
│   └── ai-assistant/             AI 助手 + 额度 + 管理后台（18 个 action）
└── cloudbase/
    └── migrations/               PG 迁移 SQL（版本化，8 个）
```

## 本地开发

```bash
cd web
npm install
cp .env.example .env.local   # 填入 publishable key
npm run dev                  # http://127.0.0.1:5173
npm run build                # 产物在 dist/
npx vitest run               # 组件测试
```

## 当前状态

**核心功能全部上线可用**：用户系统、微信读书同步、书架/详情/金句/心得、TXT 导入与站内阅读器、
AI 导入（按书名生成内容提要）、阅读统计与报告、月度海报、公开分享、三主题、月兔吉祥物、
AI 多模型与月兔原文问答（P0–P6 闭环）。

**未完成**：P7 增强（流式输出等）、数据导出/恢复、封面上传、运维监控、
全量历史回溯、部分视觉收尾。详见 `YueRead_Complete_Handover.md` 第六、七章。

**实测基线**：e2e 60/60、AI 离线 103+83+79、前端渲染 47/47。

## 环境

- EnvId：`lexi-d9gb3ruct4fdb9d0e`（个人版，ap-shanghai，AppId 1512381244）
- 静态域名：`https://lexi-d9gb3ruct4fdb9d0e-1512381244.tcloudbaseapp.com`
- 网关域名：`https://lexi-d9gb3ruct4fdb9d0e-1512381244.ap-shanghai.app.tcloudbase.com`
  （公开入口：`POST /api/auth-register`、`POST /api/share`）
- ⚠️ 该环境 **2026-11-21 23:59:59 到期**，`IsAutoRenew: false` ——
  不续费会停服（数据不丢，¥39.9/月，需先确认）
- ⚠️ envId 永不手打，用 `tcb env list` 管道取权威值（见 `HANDOFF.md` §2）
