# KDC Idol Ledger｜生产可用性验收页

GPT 可直接读取的静态 Demo：<https://kdc-idol-ledger-demo.vercel.app/>

完整 Sites 运行版（当前明确为 Demo 模式）：<https://idol-ledger-20260818.qkdckzzz.chatgpt.site/>

> 当前结论：Code Ready；Environment Configured = No；External E2E Verified = No。  
> EXTERNAL CREDENTIAL E2E NOT EXECUTED

以下内容与当前提交 `01c79eeb1be91805ca2193ef090247c833f8b1e9` 的真实实现一致。

## 项目说明

面向 KDC 地下偶像联合运营团队的中文财务 Web App。系统把经营收入、项目成本、项目账户余额、成员垫付、成员代收和艺人结算分开核算，避免提现、内部转账、报销或项目投入被重复算作经营收支。

当前首页固定显示：

```text
KDC
偶像运营账本
```

首页不读取或显示当前用户名，也不包含“早上好 / 下午好 / 晚上好 / 欢迎回来”等个性化问候。

## 当前实现

- 手机优先 Dashboard、收入结构、最近流水和可点击的待确认 AI 流水
- KDC 默认成员：`zyy`、`ly`、`fq`
- KDC 官方账户：KDC 公账、KDC 官号微信、KDC 官号支付宝、KDC 微店
- 每位成员的微信、支付宝、银行卡均建模为成员个人账户
- 手工收入/支出快速录入；付款账户会自动决定是否形成成员垫付
- 成员详情优先展示累计垫付、已报销、待报销、垫付明细、累计代收、已转公账、待转公账
- Supabase 邮箱登录、项目/成员/艺人/活动/账户管理、RLS、私有凭证 Bucket 和审计日志
- 普通流水保存应结算、已结算、待结算金额，自动推导未结算/部分结算/已结算
- 正式流水的财务影响不可改写；只允许更新金额化结算进度，其他错误必须冲正
- 艺人拍切按当月实际参与场次计算场均，自动应用 65/35、60/40、55/45 阶梯
- 相纸成本固定 ¥6/张并从艺人所得扣除，艺人比例在函数与数据库双重限制为最高 45%
- 月结确认保存规则版本、比例、来源流水和完整不可变快照
- 金额统一使用整数“分”，当前有 39 项单元测试、3 项跨流程集成测试、3 项数据库契约测试和 1 项 SSR 测试

运行模式必须显式设为 `APP_MODE=demo` 或 `APP_MODE=production`。Demo 会明确显示“不会保存”；Production 缺少任一必需外部配置或数据库读取失败时会失败关闭，不会静默回退到 Demo。

## KDC 收入分类

Demo 和新建 KDC 项目的收入分类按以下顺序创建：

1. 官号收账
2. 微店电切收入
3. 票房
4. 商务
5. 周边
6. 其他收入

前三项是当前主要经营收入。微店收入进入 KDC 微店账户；之后从微店提现至 KDC 公账属于内部转账，不会再次增加经营收入。

## 账户归属与成员垫付

付款账户是分类规则的唯一事实来源，用户不需要重复选择“是否垫付”。

| 场景 | 项目成本 | 付款账户 | 成员待报销 | KDC 公账 |
| --- | ---: | ---: | ---: | ---: |
| ly 支付宝支付项目摄影费 ¥1,000 | +¥1,000 | ly 支付宝 −¥1,000 | ly +¥1,000 | 不变 |
| KDC 官号支付宝支付摄影费 ¥1,000 | +¥1,000 | KDC 官号支付宝 −¥1,000 | 不变 | 不变 |
| fq 微信私人消费 ¥300，标记为非项目 | 不变 | 不进入项目账本 | 不变 | 不变 |

所有 `zyy / ly / fq` 个人账户确认属于 KDC 的支出，都会自动改写为 `advance`，自动带出成员并创建金额化待报销记录。KDC 项目账户直接支出保持 `expense`，不产生成员报销。非项目私人消费在 API 层返回忽略，数据库也禁止其进入正式账本。

报销记录使用：

- `original_advance_amount_cents`
- `reimbursed_amount_cents`
- `remaining_reimbursement_amount_cents`
- `unreimbursed / partial / reimbursed`

## AI 截图识别闭环

`POST /api/ai/upload` 接收图片并写入 Supabase Private Storage，服务端通过 OpenAI Responses API 图像输入和严格 JSON Schema Structured Output 提取候选字段。Zod 再次校验后，确定性规则按账户归属生成 Pending Transaction。当前服务端只接受 JPG / PNG / WebP / HEIC / HEIF，同时校验 MIME、扩展名、真实文件头、单张 10MB 与单批最多 20 张，并以 2–5 个并发任务处理：

- 成员个人账户 + 确认属于项目 → 推荐“成员垫付”、成员、原垫付金额、待报销金额和“未报销”
- KDC 项目账户 + 确认属于项目 → 推荐普通项目支出，不产生待报销
- 非项目消费 → 推荐忽略，不进入正式流水
- 商户关键词明确时可推荐练习室租赁、摄影、摄影场地费、美工、交通、住宿、服装、妆造、相纸、周边制作、宣传或设备租赁
- 无法确定的字段返回 `null`，不编造
- 重复检测依次使用外部交易号、订单号、平台+金额+时间+交易对象、图片 SHA-256
- 疑似重复默认禁止入账，可查看已有、忽略、强制入账或合并
- 单张失败不会中断整批；已存储原图的失败项可以单独重试
- OpenAI 请求使用 30 秒超时、有界重试与指数退避，日志只记录批次计数和安全错误码

AI 详情抽屉显示私有签名原图、提取字段、账户、分类、KDC 归属、推荐原因、预计待报销和重复状态。`POST /api/ai/pending/:id/confirm` 只有在用户确认后才创建正式流水；AI 无权直接正式入账。该确认由数据库 RPC 持有 Pending 行锁并原子完成，唯一索引确保重复点击不会生成第二笔流水。

真实识别需要同时配置 Supabase 与服务端 `OPENAI_API_KEY`。外部凭据未就绪时只能显式部署 `APP_MODE=demo`；不得把这种部署标记为生产环境。

## 艺人拍切分成

规则存于 `photo_split_rule_versions`，支持 KDC Project Default Rule 与 Artist Override：

| 当月该艺人场均 | KDC | 艺人 | 比例 |
| --- | ---: | ---: | --- |
| `<100` | 65% | 35% | 6.5 : 3.5 |
| `>=100 且 <200` | 60% | 40% | 6 : 4 |
| `>=200` | 55% | 45% | 5.5 : 4.5 |

场均只使用“该艺人当月拍切总张数 ÷ 该艺人实际参与的拍切活动场数”。0 场返回 `no_data` 和 `null average`。单场页面属于预估；月底由 `POST /api/artists/:id/settlements` 按最终整月数据生成并锁定 `artist_settlements.snapshot`。

## 技术栈

- React 19、TypeScript、Tailwind CSS 4
- Next.js App Router 兼容目录；本地预览与 Sites 使用 Vinext
- Supabase Auth、PostgreSQL、Storage 与 Row Level Security
- Zod 输入校验与 AI Structured Output Schema
- Node.js 内置测试运行器
- Lucide 图标

## 本地启动

需要 Node.js 22.13 或更高版本，推荐 pnpm 11。

```bash
pnpm install
cp .env.example .env.local
## 在 .env.local 中显式保留 APP_MODE=demo
pnpm dev
```

打开 `http://localhost:3000`。演示模式不会写入 Supabase。

## 环境变量

```dotenv
APP_MODE=demo
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
OPENAI_API_KEY=
OPENAI_VISION_MODEL=gpt-5.4
SUPABASE_STORAGE_BUCKET=transaction-receipts
NEXT_PUBLIC_APP_URL=http://localhost:3000
AI_MAX_FILES=20
AI_MAX_FILE_BYTES=10485760
AI_MAX_AMOUNT_CENTS=100000000
AI_CONCURRENCY=3
```

`SUPABASE_SERVICE_ROLE_KEY` 和 `OPENAI_API_KEY` 只能在服务端使用，不得提交到 Git。
正式上线时必须改为 `APP_MODE=production`。可访问 `GET /api/health` 查看模式和各项服务是否已配置；返回中不包含密钥值。

## Supabase 设置

1. 创建 Supabase 项目并安装 CLI。
2. 执行 `supabase link --project-ref <project-ref>`。
3. 执行 `supabase db push`。
4. 在 Supabase Auth Redirect URLs 中加入本地和生产域名的 `/auth/callback`。

迁移：

- `supabase/migrations/202608180001_phase1.sql`：基础账本、权限、审计、AI 待确认结构
- `supabase/migrations/202608190001_kdc_business_rules.sql`：KDC 账户归属、自动垫付、金额化报销和新项目默认资料
- `supabase/migrations/202608190002_ai_settlement_artist_splits.sql`：普通流水结算金额、完整 AI Pending 字段/索引/RLS、版本化拍切规则、45% 约束和不可变月结快照
- `supabase/migrations/202608190003_production_readiness.sql`：原子记账/AI 确认/报销/转账/冲正/月度艺人结算 RPC、幂等索引、跨项目引用校验、流水派生账户余额和私有存储加固

开发种子数据位于 `supabase/seed.sql`，仅供 `supabase db reset` 的本地测试使用；正式 `supabase db push` 只应执行 migrations，不得导入 Demo 流水。

## 财务事件

| 业务事件 | 经营收入 | 项目成本 | 公账 | 成员待报销 | 成员待转公账 |
| --- | ---: | ---: | ---: | ---: | ---: |
| 官号 / 微店 / 票房收入 | 增加 | — | 视收款账户 | — | — |
| 成员代收 | 增加 | — | 不变 | — | 增加 |
| 代收转公账 | 不变 | — | 增加 | — | 减少 |
| 成员个人账户项目支出 | — | 增加 | 不变 | 增加 | — |
| KDC 项目账户直接支出 | — | 增加 | 视付款账户 | 不变 | — |
| 公账报销 | — | 不变 | 减少 | 减少 | — |
| 普通账户转账 / 微店提现 | 不变 | 不变 | 视方向 | — | — |

## 验证

```bash
pnpm lint
pnpm typecheck
pnpm test
pnpm test:integration
pnpm test:contract
APP_MODE=demo pnpm build
APP_MODE=demo pnpm test:render
supabase db reset
supabase test db
```

测试明确覆盖：

- ly 支付宝项目摄影费自动成为 ly 垫付
- KDC 官号支付宝项目支出不产生成员报销
- fq 微信私人消费不进入 KDC 财务流水
- 微店收入后提现不重复增加经营收入
- AI 根据 fq 微信和练习室商户推荐成员垫付；不确定分类时返回空
- 普通结算与成员报销状态均由金额自动推导，且互不混用
- 场均 80 / 120 / 220 分别得到 35% / 40% / 45%，0 场不除零
- 100 张相纸成本为 ¥600，销售额 ¥6,000 的基础档艺人最终所得为 ¥1,500
- 月中档位变化按月底最终场均结算，已确认历史快照不受未来规则修改影响
- 99.99 / 100 / 199.99 / 200 的阶梯边界
- 生产模式缺配置时失败关闭，上传数量/体积/MIME/扩展名/真实文件头边界
- OpenAI 非法 JSON、null 字段和超额金额边界
- Project A/B RLS 隔离、私有凭证跨项目禁止、AI 确认幂等、报销原子性与月结快照不可变（`supabase/tests/*.sql`）

## 生产上线状态

| 层级 | 状态 | 说明 |
| --- | --- | --- |
| Code Ready | 是 | 代码、迁移、RPC、上传限制、健康检查和测试脚本已就绪 |
| Environment Configured | 否 | 当前工作区与 Sites 运行时均未发现 Supabase / OpenAI 凭据 |
| External E2E | 未执行 | `EXTERNAL CREDENTIAL E2E NOT EXECUTED` |

当前发布站点只能作为显式 Demo，不是生产财务环境。Vercel 镜像是为 GPT 可读性提供的静态 Demo，不承载动态 API，因此当前不能标记为 `Vercel Production Ready`。

## 生产配置 Checklist

- [ ] 创建 Supabase Project
- [ ] 执行全部 migrations，并运行 `supabase test db`
- [ ] 配置 Auth 与生产域名 `/auth/callback`
- [ ] 确认 `transaction-receipts` 是 Private Storage Bucket
- [ ] 验证所有业务表与 Storage 的 RLS 正、负向用例
- [ ] 在生产运行时配置 `APP_MODE=production`
- [ ] 配置 `NEXT_PUBLIC_SUPABASE_URL`
- [ ] 配置 `NEXT_PUBLIC_SUPABASE_ANON_KEY`
- [ ] 配置仅服务端可见的 `SUPABASE_SERVICE_ROLE_KEY`
- [ ] 配置仅服务端可见的 `OPENAI_API_KEY`
- [ ] 配置 `SUPABASE_STORAGE_BUCKET`、`NEXT_PUBLIC_APP_URL` 与 AI 限额
- [ ] Redeploy Production
- [ ] 检查 `/api/health` 返回 HTTP 200、`mode=production` 且全部服务为 `true`
- [ ] 验证登录、退出、Session refresh 与过期 Session
- [ ] 验证真实数据库持久化，不出现 Demo 数据
- [ ] 验证真实中文微信/支付宝截图识别
- [ ] 验证 Pending 修正、原子 Confirm 与重复点击幂等
- [ ] 验证重复检测、成员垫付、报销、转公账与内部转账
- [ ] 验证月结快照及数据库不可变约束
- [ ] 开启 Supabase 项目级数据库备份/PITR，并演练恢复
- [ ] 配置运行时错误监控与数据库告警

CSV 数据导出尚未实现，不属于本轮 P0。上线后应按数据量和合规要求设置支付截图保留周期；当前私有 Bucket 支持管理员按项目删除，但尚未配置自动清理任务。

架构细节见 `docs/architecture.md`。
