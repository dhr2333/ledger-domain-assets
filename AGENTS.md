# AGENTS.md — 账本写入规则

本文件是写给 AI 助手（Agent）的写入契约，不是给人看的使用说明；人类用户请读 `README.md` 与用户指南。

本目录是 **Beancount 账本**，不是代码仓库。账本文件是唯一真相源。

## 唯一写者

- **`beancount-guard` 负责整个账本的写入。**
- 交易追加到 `trans/collect.bean`；`balance` / `pad` 追加到 `trans/reconciliation.bean`。
- `trans/main.bean` 只 include 这两个文件，不写交易。
- `main.bean` / `account/*.bean` 的结构与账户变更也由 Core 写。
- **禁止**直接编辑 `main.bean`、`account/*.bean`、`trans/*.bean`、`.ledger/*`。
- 初始模板创建是 bootstrap，不属于运行时写入；之后所有 `.bean` 写入走 Core。
- `trans/collect.bean` 是 Core 的条目目标，不是平台写入区；这里没有 Beancount-Trans Backend。

## 写入路径

| 操作 | 路径 | 人工确认 |
|---|---|---|
| 记交易 | Core `submit-draft` / `submit-batch` → DSH 面板 `approve-draft` → `trans/collect.bean` | 有 |
| 对账余额 | Core 生成 `balance` / `pad` → `trans/reconciliation.bean` | 有 |
| 账户 / 别名 / 对账周期 | Core `metadata`（MVP-0）；人通过 DSH 编辑 | 人确认 |
| 新增账户 | Core `account-open`（MVP-0）→ `account/*.bean` | 人确认 |
| 查询 | 只读 BQL / `bean-query` | 只读 |

## 导入批次与 Agent 工具

优先用 DSH 的 `ledger_*` 工具；没有工具时用 Core CLI。四个工具都是**只提案**：

| 工具 | 作用 |
|---|---|
| `ledger_context` | 账本根、币种、账户/别名/开立日/周期、待审计数、批次契约 |
| `ledger_propose_batch` | 一次提交 `source` + `claimed` + `skipped` + `drafts` + 可选 `operations` |
| `ledger_submit_operations` | 单独提交开户/开立日调整/余额 |
| `ledger_queue` | 只读待审草稿、结构操作、批次 |

批次规则：

- `source.rows` 是账单原始行数；`skipped` 每行写 `index` 与 `reason`。面板用 `drafts + skipped + duplicates + failed == rows` 做覆盖校验。
- 每条草稿都带 `source.evidence`：把该条对应的账单原始行按「列名 → 值」放进去（如 `交易时间` / `交易对方` / `商品说明` / `支付方式` / `交易状态` / `交易单号`）。Core 全量保存，DSH 面板在「只看异常」时展示，供人改描述和资金账户时核对；`evidence` 只作核对，不写入账本，不要编造。
- 草稿 posting 可带 `role`（`category` / `funding`）与 `fallback`（`expense` / `income`）。分类不确定时用 `account: null` + `suggested` + `role: "category"`；不要新建支出/收入科目。
- 部分退款记负支出，并带 `fallback: "expense"`；与关闭单等额的原路退回不记。
- 资金账户必须能唯一确定，并且是已存在的 `Assets:` / `Liabilities:`：
  - 只有一个候选：直接用；
  - 不存在：提交 `account-open`，不要编造；
  - 多个候选（如同一家银行有多张卡），或账单只写了银行名、没写卡号：**不猜、不新开户、不用兜底账户**。把该 posting 的 `account` 留空，`suggested` 写账单原文，标 `role: "funding"`，并在回复里列出候选账户，交给人选定。
- 同一家银行多张卡时一卡一户：路径带卡号后四位（如 `Assets:Savings:Bank:BOC0814`），别名同样带后四位（如 `中国银行 0814`）；不要两张卡共用银行名作别名，别名必须全局唯一。
- 开立日晚于交易就提交 `account-redate`；绝不用兜底账户代替资金账户。
- `metadata.uuid` 是幂等键。同一交易重复提交会被 `duplicate_transaction` 拒绝；不要靠改 uuid 绕过。
- 提交后只报告「已入队 / 跳过 / 重复」，**不要调用批准**。

## 派生文档与原始账单

账本目录只承载账本文件：`main.bean`、`account/`、`trans/`、`AGENTS.md`、`README.md`、`LICENSE`、`.gitignore`。
Agent 不得在账本根目录、`account/`、`trans/` 下新建 `.md`、`.csv`、`.txt`、截图等非账本文件。

- 一次性结果（查询输出、少量说明）直接回复在对话或 DSH 面板，不落盘。
- 需要给人翻看的清单、报告、原始账单（导入进度、待批准列表、对账说明、CSV/PDF 账单）放到 `documents/`，报告命名为 `YYYY-MM-DD-主题.md`。
- `documents/` 不是账本真相：报告是输出，不得作为写入依据；原始账单可以被读取用于生成草稿，但金额/日期必须来自账单本身，不能从报告反推。
- `documents/` 已加入 `.gitignore`，可随时删除，不参与账本真相与 Git 跟踪。
- 不要写进 `.ledger/`：那是 Core 的私有运行态（草稿/锁/审计），Agent 只读不写。

## 条目元数据

- 交易 metadata 通过草稿的 `metadata` 字段提交，**禁止直接写 `.bean`**。
- 约定键：`time`（`HH:MM:SS`，账单有具体时间才带）、`uuid`（渠道交易号）、`status`（例如 `WeChat - 已转账`，渠道体现在这里）；也可以另加合法的小写键。
- 值必须是字符串；账单没有的信息就省略，不要编造。
- 这条规则只影响新草稿；历史条目不需要回填。

## 硬规则

1. **永不直接改账本文件。** 所有 `.bean` 写入走 Core；`main.bean` / `account/*.bean` 也一样。
2. **交易只进草稿队列。** 草稿出现 `unbalanced` / `funding_unresolved` / `before_account_open` / `closed_account` / `no_real_account` 时停下来报告，不要自己改金额或日期。分类不确定时把账户留空（或用 `suggested`），不要猜科目；面板批准时按 `fallback` 或金额符号归入 `Expenses:Uncategorized` / `Income:Uncategorized`。资金账户有多个候选时同样停下，不要靠猜或新开户消除歧义。
3. **对账只走 Core，且只对资产/负债。** 不要手写 `balance` 或 `pad`；差额由 Core 按 `balance + pad` 生成。支出/收入/权益不参与对账。
4. **查询只读。** BQL 只能是 `SELECT`；不得用查询结果反推写入。
5. **账户显式。** 新账户走 Core 的 account-open（MVP-0）；不要手工加账户，也不要依赖隐式账户。
6. **不删除审计。** `.ledger/` 是 Core 运行态；不要删除草稿、审计或锁文件。
7. **不提交运行态。** `.ledger/`、`.venv/` 已在 `.gitignore`。
8. **派生文档与原始账单只进 `documents/`。** 不得在账本根目录或 `account/`、`trans/` 下新建非账本文件；`documents/` 不进 Git。

## 当前阶段

- MVP-0：从已有账本开始 → 账户 alias/reconcile/新增 → 手动记账/审核 → `trans/collect.bean`；只读 BQL。初始化不在 MVP-0，后续 DSH 实现。
- MVP-1：Core 对账 `balance + pad`，写入 `trans/reconciliation.bean`。

## 排查

- `bean-check main.bean` 失败时，先看错误里的文件名和行号；不要靠猜。
- 账本结构是 `main.bean` → `account/*.bean` + `trans/main.bean`；`trans/main.bean` 只 include `collect.bean` 与 `reconciliation.bean`。
- 写入失败时 Core 会返回 `code/message/details`；按错误码处理，不要绕过 Core 直接改文件。
