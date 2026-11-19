# ledger-domain-assets

`ledger-domain-core` 的本地 Beancount 账本模板。科目命名沿用
[Beancount-Trans-Assets](https://github.com/Beancount-Trans/Beancount-Trans-Assets)，
并按 Core 的写入模型调整：Core 是唯一写者，账户全部显式声明，不启用隐式账户。

## 目录

```text
.
├── main.bean               # 账本入口
├── account/*.bean          # 账户定义
├── trans/
│   ├── main.bean           # 只 include 下面两个文件
│   ├── collect.bean        # 交易（Core 写）
│   └── reconciliation.bean # balance / pad（Core 写）
├── AGENTS.md               # Agent 写入规则
└── LICENSE
```

`trans/collect.bean` 与 `trans/reconciliation.bean` 是账本真相，必须被 Git 跟踪；即使暂时为空，也要保留并被 include。

## 使用

```bash
bean-check main.bean
fava main.bean   # 可选，只读报表
```

新账本由 DSH 插件初始化：复制模板到 staging → `bean-check` → 原子 rename。也可以手工复制本模板，再把 Core / 面板的 `root` 指向它。

## 账户

- 55 个账户：资产 6、负债 1、支出 29、收入 17、权益 2，全部带唯一 `alias`；
- 只有资产/负债参与对账；默认「余额宝」「零钱通」按天，其他在账户页开启；
- 新账户走 Core 的 `account-open`；银行账户沿用 Assets 科目树，例如 `Assets:Savings:Bank:BOC0814`。

## Git

模板自带 `.gitignore` 忽略 `.ledger/`、`.venv/`；`trans/collect.bean` 与 `trans/reconciliation.bean` 必须跟踪。Core 不会自动 `git add/commit/push`，提交节奏由用户负责。

## 写入规则

- 不要手动编辑任何 `.bean`；Core 负责整个账本写入；
- 交易追加到 `trans/collect.bean`，对账 `balance` / `pad` 追加到 `trans/reconciliation.bean`；
- `main.bean` / `account/*.bean` 的结构与账户变更也走 Core；
- 交易先入草稿队列，再由人在 DSH 面板批准；
- Agent 规则见 [AGENTS.md](AGENTS.md)。

## License

MIT。账户命名与模板结构来自 Beancount-Trans-Assets，版权归原作者。
