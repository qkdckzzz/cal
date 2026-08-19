# KDC 偶像运营账本｜第二轮完整版验收页

GPT 可直接读取的公开 Demo：<https://kdc-idol-ledger-demo.vercel.app/>

完整 Sites 运行版：<https://idol-ledger-20260818.qkdckzzz.chatgpt.site/>

本文件只描述当前仓库中的真实实现。线上 Demo 未配置用户的 Supabase 和 OpenAI 密钥时使用不写库的演示数据；真实上传、OpenAI 图像理解、Pending、确认入账和月结 API 已实现，但只有配置外部服务后才会在生产环境实际调用。

## 基础信息

| 项目 | 当前实现 |
| --- | --- |
| 唯一 Demo 项目名 | KDC |
| 首页标题 | KDC / 偶像运营账本 |
| 用户问候 | 已删除；不显示早上好、下午好、晚上好、欢迎回来或用户名 |
| 默认成员 | zyy / ly / fq |
| 主要收入 | 官号收账 / 微店电切收入 / 票房 |
| KDC 项目账户 | KDC 公账 / 官号微信 / 官号支付宝 / 微店 |
| 成员个人账户 | zyy / ly / fq 各自的微信、支付宝、银行卡 |

`accounts.owner_member_id` 是账户归属的事实来源。成员个人账户确认属于 KDC 的项目支出自动成为成员垫付；KDC 项目账户支出保持普通项目成本；私人消费不得进入正式流水。

## 收入与内部资金移动

- 官号收账、微店电切收入、票房首次确认时增加经营收入。
- ly 微信等成员账户收到 KDC 收入时记录成员代收，增加经营收入和该成员待转公账，不改变 KDC 公账。
- 成员代收转公账和 KDC 微店提现只移动账户资金，不重复增加经营收入。
- 项目投入只增加项目资金与成员投入，不计经营收入或经营利润。

## 成员垫付与报销

例如 fq 微信支付 XX练习室 ¥600 并确认属于 KDC：

```text
项目成本 +¥600
fq 个人账户 −¥600
fq 待报销 +¥600
KDC 公账不变
```

`member_advances` 保存：

- `original_advance_amount_cents`
- `reimbursed_amount_cents`
- `remaining_reimbursement_amount_cents`
- `reimbursement_status`

报销状态由金额自动推导：0 为未报销，介于 0 与原额之间为部分报销，剩余 0 为已报销。Demo 覆盖 zyy 已报销、ly 部分报销、fq 未报销与部分报销。

## 普通流水结算状态

`transactions` 保存：

- `settlement_amount_cents`
- `settled_amount_cents`
- `remaining_settlement_amount_cents`
- `settlement_status = unsettled / partial / settled`

结算状态由金额触发器自动推导，并在首页最近流水、账本列表、金额详情和筛选中显示。正式流水的财务影响仍不可改写；只允许更新 `settled_amount_cents`，其他错误必须通过冲正处理。

金额化结算状态用于普通业务流水与艺人月结，`reimbursement_status` 只用于成员垫付，两者没有混用。

## AI 真实识别闭环

当前链路：

```text
图片上传
→ Supabase Private Storage
→ 服务端 OpenAI Responses API 图像理解
→ 严格 JSON Schema Structured Output
→ Zod 二次校验
→ 确定性 KDC 账户规则
→ ai_pending_transactions
→ 用户查看、修改、忽略或确认
→ transactions 正式流水
```

实现文件和接口：

- `lib/ai/receipt.ts`：OpenAI 图像输入、Structured Output 和服务端解析
- `POST /api/ai/upload`：图片哈希、私有上传、识别、账户匹配、重复检测和 Pending 创建
- `GET/PATCH /api/ai/pending/:id`：签名原图、完整字段、修改、忽略
- `POST /api/ai/pending/:id/confirm`：用户确认后才创建正式流水；AI 无权直接入账
- `POST /api/ai/recommend`：保留的确定性账户规则接口

至少提取平台、交易类型、金额、时间、交易对象、商户、账户、订单号、外部交易号、描述和置信度；无法确认的字段返回 `null`，不得编造。

AI 列表卡片整卡可点击，显示“建议记账”“AI识别 · 待确认”“查看并确认”，不再显示“97%可信”或“自动推荐”。详情抽屉显示原始截图、全部字段、KDC 归属、预计待报销、推荐原因和重复状态，并提供修改、忽略、确认、查看原图、查看已有、强制入账和合并。

重复检测严格按以下顺序：

1. `external_transaction_id`
2. `order_id`
3. `platform + amount + transaction_time + counterparty`
4. 图片 SHA-256

疑似重复默认禁止正式入账。

## 艺人拍切规则

规则版本存于 `photo_split_rule_versions`。一条活跃记录可作为 KDC Project Default Rule；`artists.photo_split_rule_override_id` 可指向 Artist Override。规则不会散落在 JSX。

| 当月该艺人场均 | KDC | 艺人 | 比例 |
| --- | ---: | ---: | --- |
| `<100` | 65% | 35% | 6.5 : 3.5 |
| `>=100 且 <200` | 60% | 40% | 6 : 4 |
| `>=200` | 55% | 45% | 5.5 : 4.5 |

艺人最高 45%。相纸成本固定 ¥6 / 张，并从艺人原始拍切分成中扣除。

场均公式：

```text
当月该艺人拍切总张数 ÷ 当月该艺人实际参与的拍切活动场数
```

不使用项目全部活动场数或月历天数。0 场返回 `no_data` 和 `null average`，界面显示暂无拍切记录并保留基础规则，不伪造“已计算场均 0”。

Demo 三个艺人分别覆盖：

- 3 场 / 240 张 / 场均 80：艺人 35%，比例 6.5 : 3.5
- 4 场 / 480 张 / 场均 120：艺人 40%，比例 6 : 4
- 3 场 / 660 张 / 场均 220：艺人 45%，比例 5.5 : 4.5

艺人详情页包含基本资料、本月数据、拍切规则、阶梯、相纸成本、拍切记录、月结计算和结算历史。手机端使用卡片，不使用超宽表格。

## 月度最终结算快照

`POST /api/artists/:id/settlements` 按最终整月场均重新确定档位，使用最终比例生成 `artist_settlements`。确认时保存月份、艺人、张数、场次、场均、档位、双方基点、比例显示、相纸单价/总成本、销售额、艺人原始/最终分成、项目分成、其他收入/扣款、最终应结、来源流水、规则版本、操作人和时间。

`artist_settlements.snapshot` 与明确列共同保存结果；`snapshot_locked` 及数据库触发器禁止已确认快照被规则变更重算。历史月结保持原比例。

## Migration

本轮新增 migration：`supabase/migrations/202608190002_ai_settlement_artist_splits.sql`

新增或扩展：

- `transaction_settlement_status` 枚举及 transactions 金额字段、约束、状态触发器、筛选索引和受限更新 RLS
- AI 图片 hash、匹配账户、质量提示、重复方法/候选字段及 4 组重复索引、Pending insert RLS
- `photo_split_rule_versions` 表、项目默认/艺人覆盖唯一索引、45% check constraints 与 RLS
- `photo_sales` 的日期、规则版本、相纸单价/总成本、双方基点字段与索引
- `artist_settlements` 的完整月结列、剩余金额、来源流水、规则版本、快照锁与不可变触发器

## 验证结果

- ESLint：通过
- TypeScript `tsc --noEmit`：通过
- 财务单元测试：26/26 通过
- 跨流程集成测试：3/3 通过
- Vinext 生产构建：通过
- SSR 测试：1/1 通过，断言 KDC、三类收入、AI 新文案与三种结算状态存在，并断言用户问候、“97%可信”“自动推荐”不存在
- 390 × 844 移动端浏览器验收：通过；首页、AI 详情抽屉、结算列表和艺人 B 月结详情可交互，控制台 0 warning / 0 error

总计：30 个自动化测试，30 通过，0 失败。

## 运行前配置状态

- 无 Supabase / OpenAI 环境变量：完整 KDC Demo 可浏览与交互；上传打开演示 Pending，不会调用外部服务或写库。
- 配置 Supabase migrations、Auth、Private Storage 和 `OPENAI_API_KEY` 后：真实图片上传、OpenAI Vision、Pending、重复检测、确认入账与月结持久化链路启用。

因此，本文件不会把“线上未配置外部密钥”写成“生产已经实际调用 OpenAI”；代码闭环已完成，外部运行凭证仍由部署环境提供。
