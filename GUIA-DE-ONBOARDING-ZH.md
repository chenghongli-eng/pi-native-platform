# Pi Native CRM 上手阅读路线（面向不熟 Next.js / React 的读者）

> **这份文档解决什么**：你不需要会写 React 或 Next.js，也能从**架构**和**分模块**两个角度读懂这个仓库。
>
> **定位**：本文是**阅读路线**，不是规范。项目的不可协商约定以 `CLAUDE.md` 为准，文档优先级见 `docs/index.md`：
> `CLAUDE.md` > `docs/specs/` > `docs/prd/` > `HANDOFF-*.md` > `README.md`。
>
> 本文提到的配套中文解析材料在**仓库之外**（随源码一起分发的 `AI-Native-CRM-Architecture-Wiki`），路径见第 0 阶段。

---

## 0. 阶段 0 —— 先读文档，不读代码（1~2 小时）

同目录分发的 `AI-Native-CRM-Architecture-Wiki` 压缩包解开后即为配套解析材料，**优先读它**，不要一上来啃源码：

```
README.md              ← 全景架构 + Agent 智能体工程深度剖析（核心必读）
QA-Interview-Guide.md  ← 18 道架构深度问答，每题带「高分回答策略」
pages/01.md  02.md     ← 业务全景：一笔生意怎么走完全程
pages/03.md            ← 代码地图：请求穿过哪些层（★最重要）
pages/04~06.md         ← 数据地图 / 权限与租户隔离 / REST 约定
pages/14~21.md         ← Agent 执行与治理（本项目的核心创新）
pages/22~25.md         ← 记忆（SQL 权威 vs Mem0 投影）/ Wiki / RAG
pages/26,27,35.md      ← Eval / Trace / 完成度与证据（★防过度解读必读）
```

> ⚠️ **`pages/35.md` 必读**：它专门讲「代码存在 ≠ 调用方接通 ≠ 测试通过 ≠ 生产验收」。
> 这个仓库很多文档是**某一天的快照**，直接照单全收会得出错误结论。

仓库内文档的推荐入口（`docs/index.md` §1 已给出完整清单）：

```
README.md           项目定位与本地开发
VISION.md           为什么 self-host、给谁用
ARCHITECTURE.md     架构一页纸
AGENTS.md           给任意代码 Agent 的可移植契约
CLAUDE.md           不可协商的约定（改代码前必读）
docs/current-state.md  什么是「已完成」的诚实快照（含过时声明警告）
docs/index.md       154 篇 md 的地图
```

---

## 1. 你只需要先懂 5 个框架概念

**不需要会写 React**，看懂这 5 件事就够读架构：

| 概念 | 含义 | 本项目对应位置 |
| --- | --- | --- |
| **文件即路由** | 文件夹/文件名决定 URL，没有独立路由表 | `app/api/v1/contacts/route.ts` → `GET/POST /api/v1/contacts` |
| **Server / Client 组件** | 默认在服务器执行；标了 `"use client"` 的才进浏览器 | `app/app/contacts/page.tsx`（服务器）vs `app/app/contacts/_client.tsx`（浏览器） |
| **Route Handler** | `route.ts` 里 `export async function GET/POST/PATCH/DELETE` 即 HTTP 接口 | `app/api/v1/**/route.ts`（全仓 371 个） |
| **Hook** | 一个函数，封装「取数据 / 改数据」 | `hooks/contacts/useContactList.ts` 等 |
| **Zod schema** | 运行时校验器（不只是编译期类型） | `lib/schemas/contacts.ts` |

---

## 2. 分层心智模型

```
浏览器（React 组件 + hooks + TanStack Query）
  │  只持有展示状态，不知道业务真相
  │  HTTP（cookie 会话 / Bearer token）
  ▼
边缘代理 proxy.ts          ← 注入 X-Request-Id、校验会话
  ▼
Route Handler              ← 薄壳：鉴权 + Zod + ok()/fail()
  app/api/v1/**/route.ts
  ▼
_handler.ts                ← 真正的业务逻辑（REST 与 MCP 共用同一份）
  ▼
lib/**                     ← 领域服务（agent-engine / auth / ai / waha / …）
  ▼
Supabase Postgres          ← 唯一真相源，RLS 行级安全 = 多租户物理隔离
  ▲
event_log → workers/       ← 副作用一律异步：发消息 / 调模型 / RAG 索引
```

### 三条铁律（理解本项目 80% 的价值）

1. **浏览器只存展示状态** —— 真相永远在 Postgres。
2. **Postgres 触发器绝不发 HTTP**（架构红线）—— 副作用走 `event_log` + worker 异步消费。
3. **租户 ID 只从可信凭证解析**（`resolveActiveOrg()`）—— **绝不信 request body**。

---

## 3. 目录地图 = 模块地图

```
app/
├── (public)/        登录、注册、LGPD 同意页
├── app/             业务页面（按功能分：contacts / inbox / kanban / ai / settings …）
└── api/v1/          371 个 REST handler（按资源分）

lib/                 ← 真正的「后端」，75 个子目录
├── agent-runtime/   Pi Agent Core 适配层（Turn 循环、工具桥接、预算）
├── agent-engine/    Agent 执行引擎、Prompt、Context 组装、guardrails
├── ai/              内置 Agent 目录、RAG、Mem0 适配
├── auth/            鉴权、RBAC、MFA
├── mcp/             MCP server（把 CRM 暴露成工具）
├── waha/ channels/  WhatsApp 接入
├── api/ wrappers.ts ok() / fail() 统一响应
├── schemas/         Zod 输入契约
└── …（supabase/ audit/ lgpd/ inbox/ metrics/ …）

supabase/migrations/ 347 个 SQL migration（schema + RLS 策略 + 触发器）
workers/              9 个后台 worker（ai-response / rag-indexer / lgpd-* / voice-agent）
app/api/v1/cron/      10 个 cron 端点，由 scheduler 容器调度消费 workers
```

一句话：**`app/` 是壳，`lib/` 是肉，`supabase/migrations/` 是骨架，`workers/` 是手脚。**

---

## 4. 阶段 1 —— 跑起来 + 只看 3 个关口文件（半天）

```bash
nvm use                                  # .nvmrc = 22
corepack enable && corepack prepare pnpm@9.15.9 --activate
pnpm install --frozen-lockfile           # 必须 pnpm，不要 npm/yarn

test -e .env.local || cp .env.example .env.local
test -e .env || touch .env

docker compose up -d                     # 本地依赖栈（WAHA / Redis 等）
pnpm dev
```

跑起来后**先不读业务代码**，只读这三个「所有请求都要过的关口」：

```
proxy.ts                      ← 边缘网关（4.6 KB，全文读完）
lib/api/wrappers.ts           ← ok() / fail()
app/api/v1/contacts/route.ts  ← 典型 handler 的「薄壳」范式
```

---

## 5. 阶段 2 —— 垂直切片：追一条线到底（1 天）

**选「新建联系人」这条最短链**，把 6 个文件串起来看：

```
components/contacts/NewContactDialog.tsx   表单收集
  → hooks/contacts/useCreateContact.ts      TanStack Query mutation
    → lib/api/client.ts                     HTTP 客户端（超时 / 幂等 / 重试策略）
      → app/api/v1/contacts/route.ts        鉴权 + Zod
        → app/api/v1/contacts/_handler.ts   业务逻辑 + audit
          → supabase/migrations/…            表结构 + RLS 策略
```

看懂这一条，其它模块基本是它的复制粘贴。**这是最高效的学法**，比逐个文件读强一个数量级。

---

## 6. 阶段 3 —— 再看 Agent 部分（2~3 天）

这个项目真正的差异点在 Agent，不在 CRM：

| 主题 | Wiki | 源码入口 |
| --- | --- | --- |
| Pi 运行时：回合、上下文、工具循环 | `pages/20.md` | `lib/agent-runtime/pi/` |
| 一次 Run 的生命周期、队列、取消与恢复 | `pages/14,15.md` | `lib/agent-engine/agent/`、`lib/agent-engine/queue/` |
| 多 Agent 编排：分工、租约、证据回收 | `pages/17.md` | `lib/agent-engine/` |
| 动作提案、人工审批与安全执行 | `pages/18.md` | `lib/agent-engine/guardrails/` |
| BYOK 与模型网关：凭据、绑定、成本、审计 | `pages/19.md` | `lib/agent-runtime/model-gateway.ts` |
| 记忆：SQL 权威 vs Mem0 投影 | `pages/22,23.md` | `lib/ai/integrations/mem0.ts` |
| RAG：索引、检索、版本引用、证据缺口 | `pages/25.md` | `lib/ai/rag/` |
| 四个内置 Agent 定义 | — | `lib/ai/agents/builtins.ts` |

---

## 7. 三个最容易误判的点

1. **工作台里每次发送 = 一个独立 Run**，不等于自动多轮上下文（`pages/02.md` 有明确警告）。
2. **工作台草稿 ≠ 可发送提案** —— start / resume 的发送候选数组当前为空，不能把「有 send 消费器 / 有审批表」推断成「普通 Workbench 已闭环审批发送」。
3. **代码在 ≠ 上线了** —— `docs/current-state.md` 开头声明自己是 dated 快照，审计发现其中 40 处描述已过时。

---

## 8. 捷径：仓库自带的源码地图

随源码分发的 `ai-native-crm-resume-skill` 里已经整理好了导航，不必自己从 4296 个文件里找：

```
references/source-map.md                    源码导航（51 KB）
assets/codewiki-source-map.json             22 篇 CodeWiki 的来源路径与 hash
assets/evidence-ledger.json                 每个技术 Claim 对应的文件 hash 与锚点
assets/candidate-bank.json                  59 个能力候选（按数据层 / Harness / Evo 分组）
```

它相当于一张**带路标的源码地图**，可直接当索引用。

---

## 9. 方法论总结

> **先读 wiki 建地图 → 跑起来 → 追一条垂直切片 → 再横向铺开 Agent 部分。**
>
> 不要从 `app/app/contacts/page.tsx` 开始逐文件读 —— 那会淹死在 React 细节里。
> **先理解「请求怎么流」，再看「每个模块写了什么」。**

配套文档见 `docs/index.md`；实操命令见 `docs/SETUP.md` 与 `docs/runbooks/`。
