# ledger-domain-assets

`ledger-domain-core` 的本地 Beancount 账本模板。基于
[Beancount-Trans-Assets](https://github.com/Beancount-Trans/Beancount-Trans-Assets)
精简，保留其科目命名习惯，并按本地 Core 的写入模型做了两处调整：

1. **Core 负责整个账本的写入。** 交易追加到 `trans/collect.bean`，`balance` / `pad` 追加到 `trans/reconciliation.bean`；`trans/main.bean` 只 include 这两个文件。`main.bean` / `account/*.bean` 的结构与账户变更也由 Core 写。不再按 `YYYY/MM.bean` 分月写入。
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
│   ├── main.bean          # 只 include collect + reconciliation
│   ├── collect.bean       # 交易条目（Core 写）
│   └── reconciliation.bean # balance / pad（Core 写）
├── AGENTS.md              # Agent 写入规则
└── LICENSE
```

## Git 可选

账本目录就是普通目录，**用户用 Git 完全没问题**：`.bean` 都是纯文本，Core 与 DSH
都不依赖 `.git`，有没有 Git 功能一致。模板自带的 `.gitignore` 已经忽略 `.ledger/`、
`.venv/` 运行态；`trans/collect.bean` 与 `trans/reconciliation.bean` 是账本真相，
**必须被跟踪**，不要加入忽略列表。

当前没有打通的是 Git 与操作流程的集成：Core 不会自动 `git add/commit/push`，面板也不提供
提交、回滚、冲突处理。提交节奏、分支与冲突由用户自己负责；对账历史在 `.ledger/`，
不进 Git，重建时按账本生成简版。

`trans/main.bean`、`collect.bean`、`reconciliation.bean` 仍必须存在并被 include；
它们即使暂时为空，也带注释头一起保留。

## 使用

```bash
bean-check main.bean
fava main.bean   # 可选，只读报表
```

把 `ledger-domain-core` 的 `root` 指向这个账本目录；DSH 账本面板会通过 Core 读写。新账本初始化由 DSH 插件实现：复制模板到 staging → `bean-check` → 原子 rename；也可以手工复制本模板，再把 Core / 面板的 `root` 指向它。
## 账户

起步账户：

- 资产（6）：现金、中国银行、支付宝、余额宝、微信、零钱通
- 负债（1）：招商银行
- 支出（29）：使用 Assets 的完整支出科目树
- 收入（17）：使用 Assets 的完整收入科目树
- 权益（2）：期初余额、调整

资产/负债没有照搬 Backend 的“自动创建”：Backend 在数据库里创建子账户时自动补父账户，是平台行为；Beancount 不要求父账户 `open`。MVP0 只给最小资产/负债集；新增银行卡/信用卡走 Core 的 account-open（MVP-0），不在运行期手工加，也不做自动创建。

`reconcile` metadata 只出现在资产/负债账户；默认只给「余额宝」「零钱通」配置按天对账，其他账户不预置，需要时在账户页开启。支出/收入/权益不参与对账，也不配 `reconcile`。

模板中所有账户都带唯一 `alias`（含支出/收入），面板默认显示中文别名；新增账户时也应补 alias，避免显示 canonical 名称。

账户 alias / reconcile 由 DSH 账户页调用 Core `metadata` 编辑。
需要新账户时走 Core 的 account-open（MVP-0）；银行账户沿用
Beancount-Trans-Assets 的科目树：中国银行默认 `Assets:Savings:Bank:BOC`，
同一家银行多张卡时在 `BOC` 后接卡号后四位，例如 `Assets:Savings:Bank:BOC0814`。

账户的 `open` 日期目前是模板创建日（2026-11-17）。DSH 创建账本时不会改写开立日；
运行期不要手改，后续由 Core 的账户接口处理。

## 状态

模板与 Core 的写入目标已经对齐：交易写 `trans/collect.bean`，对账写
`trans/reconciliation.bean`，`trans/main.bean` 只 include 这两个文件；Core 的
MVP-0 / MVP-1 已按此实现，`bean-check main.bean` 通过。

当前模板共 55 个账户（资产 6、负债 1、支出 29、收入 17、权益 2），全部带
`alias`；只预置「余额宝」「零钱通」按天对账。新账本初始化由 DSH 插件实现：
复制模板到 staging → `bean-check` → 原子 rename；也可手工复制本模板，再把
Core / 面板的 `root` 指向它。

## 写入规则

- **不要手动编辑任何 `.bean`**；Core 负责整个账本写入，交易追加到 `trans/collect.bean`，对账 `balance` / `pad` 追加到 `trans/reconciliation.bean`，账户/结构变更也走 Core。
- 交易先入 Core 草稿队列，再由人在 DSH 面板批准。
- `main.bean` / `account/*.bean` 是账户与结构配置，不是日常写入目标。
- Agent 规则见 [AGENTS.md](AGENTS.md)。

## License

MIT。账户命名与模板结构来自 Beancount-Trans-Assets，原项目版权归其作者。
