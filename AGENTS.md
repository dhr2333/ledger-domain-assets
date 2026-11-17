# AGENTS.md — 账本写入规则

本目录是 **Beancount 账本**，不是代码仓库。账本文件是唯一真相源。

## 唯一写入目标

- **`trans/main.bean` 是唯一写入目标。**
- 所有交易、`balance`、`pad` 必须由 `ledger-domain-core` 生成，并追加到 `trans/main.bean`。
- **禁止**直接编辑 `main.bean`、`account/*.bean`、`trans/main.bean`、`.ledger/*`。
- **唯一例外：人做 setup 时可以改 `main.bean` 与 `account/*.bean`；Agent 不得做，Core 不写。**
- 禁止把 `trans/` 当成平台写入区；这里没有 Beancount-Trans Backend，也没有 `trans/collect.bean`。

## 写入路径

| 操作 | 路径 | 人工确认 |
|---|---|---|
| 记交易 | Core `submit-draft` → DSH 面板 `approve-draft` | 有 |
| 对账余额 | Core 生成 `balance`（MVP-1 起同时生成 `pad`） | 有 |
| 账户 / 别名 / 对账周期 | setup 在 `account/*.bean` 人工维护；Core 只读 | 人确认 |
| 查询 | 只读 BQL / `bean-query` | 只读 |

## 硬规则

1. **日常永不直接改账本文件。** 交易/对账写入走 Core；`main.bean` / `account/*.bean` 只在 setup 时由人维护，Agent 不得做。
2. **交易只进草稿队列。** 草稿出现 `unresolved_account` / `before_account_open` / `closed_account` / `unbalanced` 时停下来报告，不要自己补科目或改金额。
3. **对账只走 Core，且只对资产/负债。** 不要手写 `balance` 或 `pad`；差额由 Core 按 `balance + pad` 生成。支出/收入/权益不参与对账。
4. **查询只读。** BQL 只能是 `SELECT`；不得用查询结果反推写入。
5. **账户显式。** 新账户在 `account/*.bean` 里按同一格式声明；不要依赖隐式账户。
6. **不删除审计。** `.ledger/` 是 Core 运行态；不要删除草稿、审计或锁文件。
7. **不提交运行态。** `.ledger/`、`.venv/` 已在 `.gitignore`。

## 当前阶段

- MVP-0：手动记账 → 审核 → 写入 `trans/main.bean`；只读 BQL。
- MVP-1：Core 对账 `balance + pad`，写入同一文件。

## 排查

- `bean-check main.bean` 失败时，先看错误里的文件名和行号；不要靠猜。
- 账本结构是 `main.bean` → `account/*.bean` → `trans/main.bean`；`trans/main.bean` 为空是正常的。
- 写入失败时 Core 会返回 `code/message/details`；按错误码处理，不要绕过 Core 直接改文件。
