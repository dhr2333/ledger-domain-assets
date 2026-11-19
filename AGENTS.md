# AGENTS.md — 账本写入规则

本文件是写给 AI 助手（Agent）的写入契约，不是给人看的使用说明；人类用户请读 `README.md` 与用户指南。

本目录是 **Beancount 账本**，不是代码仓库。账本文件是唯一真相源。

## 唯一写者

- **`ledger-domain-core` 负责整个账本的写入。**
- 交易追加到 `trans/collect.bean`；`balance` / `pad` 追加到 `trans/reconciliation.bean`。
- `trans/main.bean` 只 include 这两个文件，不写交易。
- `main.bean` / `account/*.bean` 的结构与账户变更也由 Core 写。
- **禁止**直接编辑 `main.bean`、`account/*.bean`、`trans/*.bean`、`.ledger/*`。
- 初始模板创建是 bootstrap，不属于运行时写入；之后所有 `.bean` 写入走 Core。
- `trans/collect.bean` 是 Core 的条目目标，不是平台写入区；这里没有 Beancount-Trans Backend。

## 写入路径

| 操作 | 路径 | 人工确认 |
|---|---|---|
| 记交易 | Core `submit-draft` → DSH 面板 `approve-draft` → `trans/collect.bean` | 有 |
| 对账余额 | Core 生成 `balance` / `pad` → `trans/reconciliation.bean` | 有 |
| 账户 / 别名 / 对账周期 | Core `metadata`（MVP-0）；人通过 DSH 编辑 | 人确认 |
| 新增账户 | Core `account-open`（MVP-0）→ `account/*.bean` | 人确认 |
| 查询 | 只读 BQL / `bean-query` | 只读 |

## 硬规则

1. **永不直接改账本文件。** 所有 `.bean` 写入走 Core；`main.bean` / `account/*.bean` 也一样。
2. **交易只进草稿队列。** 草稿出现 `unresolved_account` / `before_account_open` / `closed_account` / `unbalanced` 时停下来报告，不要自己补科目或改金额。
3. **对账只走 Core，且只对资产/负债。** 不要手写 `balance` 或 `pad`；差额由 Core 按 `balance + pad` 生成。支出/收入/权益不参与对账。
4. **查询只读。** BQL 只能是 `SELECT`；不得用查询结果反推写入。
5. **账户显式。** 新账户走 Core 的 account-open（MVP-0）；不要手工加账户，也不要依赖隐式账户。
6. **不删除审计。** `.ledger/` 是 Core 运行态；不要删除草稿、审计或锁文件。
7. **不提交运行态。** `.ledger/`、`.venv/` 已在 `.gitignore`。

## 当前阶段

- MVP-0：从已有账本开始 → 账户 alias/reconcile/新增 → 手动记账/审核 → `trans/collect.bean`；只读 BQL。初始化不在 MVP-0，后续 DSH 实现。
- MVP-1：Core 对账 `balance + pad`，写入 `trans/reconciliation.bean`。

## 排查

- `bean-check main.bean` 失败时，先看错误里的文件名和行号；不要靠猜。
- 账本结构是 `main.bean` → `account/*.bean` + `trans/main.bean`；`trans/main.bean` 只 include `collect.bean` 与 `reconciliation.bean`。
- 写入失败时 Core 会返回 `code/message/details`；按错误码处理，不要绕过 Core 直接改文件。
