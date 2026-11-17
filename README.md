# ledger-domain-assets

`ledger-domain-core` 的本地 Beancount 账本模板。基于
[Beancount-Trans-Assets](https://github.com/Beancount-Trans/Beancount-Trans-Assets)
精简，保留其科目命名习惯，并按本地 Core 的写入模型做了两处调整：

1. **Core 只写 `trans/main.bean`。** 所有交易、`balance`、`pad` 都由 Core 追加到这个文件；不再按 `YYYY/MM.bean` 分月写入。
2. **账户显式声明。** 支出/收入/权益固定使用 Beancount-Trans-Assets 的科目树；资产/负债只放最小起步集，更多银行卡/信用卡由 setup 手工扩展。不启用 `auto_accounts`，避免隐式账户绕过账户管理。

## 目录

```text
.
├── main.bean              # 账本入口：账户 + trans/main.bean
├── account/               # 账户定义
│   ├── assets.bean
│   ├── liabilities.bean
│   ├── expenses.bean
│   ├── income.bean
│   └── equity.bean
├── trans/
│   └── main.bean          # Core 唯一写入目标（不要手改）
├── document/              # 单据附件目录
├── AGENTS.md              # Agent 写入规则
└── LICENSE
```

## 使用

```bash
bean-check main.bean
fava main.bean   # 可选，只读报表
```

把 `ledger-domain-core` 的 `root` 指向本仓库；DSH 账本面板会通过 Core 读写。

## 账户

起步账户：

- 资产（6）：现金、工资卡、支付宝、余额宝、微信、零钱通
- 负债（1）：信用卡
- 支出（29）：使用 Assets 的完整支出科目树
- 收入（17）：使用 Assets 的完整收入科目树
- 权益（2）：期初余额、调整

资产/负债没有照搬 Backend 的“自动创建”：Backend 在数据库里创建子账户时自动补父账户，是平台行为；Beancount 不要求父账户 `open`。MVP0 只给最小资产/负债集，新增银行卡/信用卡在 `account/assets.bean` / `account/liabilities.bean` 里 setup，不做运行时自动创建。

`reconcile` metadata 只出现在资产/负债账户；支出/收入/权益不参与对账，也不配 `reconcile`。

需要新账户时，在对应 `account/*.bean` 里按同样的 `open` 格式添加；账户名沿用
Beancount-Trans-Assets 的科目树，例如 `Assets:Savings:Bank:Ningbo`。

账户的 `open` 日期目前是模板创建日（2026-11-17）。首次使用时把它改成你真正开始
记账的日期；否则对账周期会从 2026-11-17 起算，历史账户可能显示逾期。

## 状态

本模板按“Core 只写 `trans/main.bean`”的目标模型建立。当前 `ledger-domain-core`
仍按 `YYYY/MM.bean` 分月写入，下一步需要把 `target.py` / `drafts.py` 的目标文件
切到 `trans/main.bean`；在切换完成前，面板手动记账会报 `year_not_managed`。

## 写入规则

- **不要手动编辑 `trans/main.bean`**；Core 的草稿审批、对账 `balance` / `pad` 都会追加到这里。
- 交易先入 Core 草稿队列，再由人在 DSH 面板批准。
- `main.bean` / `account/*.bean` 是账户与结构配置，不是日常写入目标。
- Agent 规则见 [AGENTS.md](AGENTS.md)。

## License

MIT。账户命名与模板结构来自 Beancount-Trans-Assets，原项目版权归其作者。
