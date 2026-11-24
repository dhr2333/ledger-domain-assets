# AGENTS.md — 账本写入规则

本目录是 Beancount 账本，不是代码仓库；账本文件是唯一真相源。本文件写给 Agent，人的说明见 `README.md`。

## 唯一写者

- `beancount-guard`（Core）是唯一写者：交易追加到 `trans/collect.bean`，`balance` / `pad` 追加到 `trans/reconciliation.bean`；`main.bean` / `account/*.bean` 的结构变更也由 Core 写。
- 禁止直接编辑 `main.bean`、`account/*.bean`、`trans/*.bean`、`.ledger/*`。初始模板创建之后，所有 `.bean` 写入都走 Core。
- 你只能提交提案（草稿、结构操作）；批准、驳回、冲销只在面板里发生。

## 导入：先登记，再解析，最后核对

一次导入 = 一份附件一个批次。漏掉一整份账单不会有任何报错：

1. 解析前用 `ledger_declare_sources` 登记本次会话的全部附件：`fileName`、`fileHash`（sha256）、`rows`。
2. 每份附件用 `ledger_propose_batch` 提交一个批次，`source.fileHash` 与登记项一致。
3. 回复前调用 `ledger_import_status`；`uncoveredCount > 0` 就是漏了账单：补上，或在回复里说明为什么不记。

银行流水和渠道账单是同一批交易的两个视角，两边都要有批次；重叠的行在银行侧声明为跳过。金额能对上不代表它不用记账。

## 覆盖与跳过

- 覆盖是行索引双射：`source.rows` 必填，每条草稿的 `source.index` 必填，`skipped` 每行写 `index`；`drafts[].source.index` 与 `skipped[].index` 必须恰好各覆盖 `0..rows-1` 一次。Core 会拒绝重复、越界、冲突和未覆盖的行，按错误码处理，不要靠改下标绕过。
- `skipped[].reasonCode` 是闭集：`closed` / `exact_refund` / `zero_amount` / `internal_transfer` / `interest` / `duplicate_source`。`exact_refund` 需要 `refundOf` 或 `refundOfUuid`，`zero_amount` 需要 `amount: "0.00"`，`duplicate_source` 需要 `duplicateOf`；`reason` 只是备注，不参与判定。
- 每条草稿都带 `source.evidence`：把该条账单原始行按「列名 → 值」放进去（交易时间 / 交易对方 / 商品说明 / 支付方式 / 交易状态 / 交易单号）。它只作核对，不写入账本，不要编造。

## 资金账户

按证据顺序取，命中即停：① 账单「支付方式 / 收付款方式」，剥掉 `&红包`、`&立减金`、`&优惠` 等营销后缀；② 对方账号 / 卡号后四位；③ 正文明文卡号。把原文放进 posting 的 `fundingHint`。

- 资金账户必须是已存在的 `Assets:` / `Liabilities:` 叶子账户；例外是明确记一笔权益调整时的 `Equity:Adjustments`。
- 钱包账户（支付宝 / 余额宝 / 微信 / 零钱通）只能用于账单明写「余额 / 零钱 / 余额宝」的行；账单写银行卡却记钱包会报 `funding_mismatch`。
- 只有一个候选：直接用。不存在：提交 `account-open`，不要编造。多个候选（同一家银行多张卡），或账单只写银行名、没写卡号：不猜、不新开户、不用兜底账户；把 `account` 留空，`suggested` 写账单原文，标 `role: "funding"`，并在回复里列出候选账户交给人。
- 同一家银行多张卡时一卡一户：路径和别名都带后四位（如 `Assets:Savings:Bank:BOC0814` / `中国银行 0814`），别名必须全局唯一。
- 开立日晚于交易就提交 `account-redate`；绝不用兜底账户代替资金账户。

## 分类、退款与内部划转

- 分类不确定时用 `account: null` + `suggested` + `role: "category"`；面板批准时按 `fallback` 或金额符号归入 `Expenses:Uncategorized` / `Income:Uncategorized`。不要为了消除 `unresolved_account` 去猜分类或新建科目。
- 部分退款记负支出，并带 `fallback: "expense"`；等额原路退回按 `exact_refund` 跳过，且原单不得同时有草稿。
- 内部划转（零钱提现、零钱通互转、信用卡还款、本人卡间转账）只记一次，记在正在导入的那一侧；另一侧按 `internal_transfer` 跳过并写明记在哪里。

## 条目元数据

- 交易 metadata 通过草稿的 `metadata` 字段提交，禁止直接写 `.bean`。
- 约定键：`time`（`HH:MM:SS`，账单有具体时间才带）、`uuid`（渠道交易号）、`status`（例如 `WeChat - 已转账`）；可另加合法的小写键。值必须是字符串；账单没有就省略，不要编造。只影响新条目，不回填历史。
- `metadata.uuid` 是幂等键：重复提交会被 `duplicate_transaction` 拒绝，不要靠改 uuid 绕过。

## 文件与运行态

- 账本目录只放账本文件：`main.bean`、`account/`、`trans/`、`AGENTS.md`、`README.md`、`LICENSE`、`.gitignore`。
- 一次性结果直接回复，不落盘；需要给人翻看的清单、报告、原始账单放 `documents/`（`YYYY-MM-DD-主题.md`）。`documents/` 不是账本真相，不能作为写入依据；金额 / 日期必须来自账单本身。
- `.ledger/` 是 Core 的私有运行态（草稿 / 锁 / 审计），只读不写，不删除。
- 查询只读（BQL 只能是 `SELECT`）；不得用查询结果反推写入。
- 所有失败按 Core 返回的 `code` / `message` / `details` 处理；不要绕过 Core 直接改文件。

## 硬规则

1. 永不直接改账本文件；所有写入走 Core。
2. 草稿出现 `unbalanced` / `funding_unresolved` / `no_real_account` / `before_account_open` / `closed_account` 时停下来报告，不要自己改金额或日期。
3. 不删除 `.ledger/`、审计或锁文件；不把运行态提交到 Git。
4. 只提案，不批准；只报告「已入队 / 跳过 / 重复」。
