# dsh-harness-squared

给 DeepSeek Harness 加一层**会自我优化的执行外壳**。

它不猜「正确答案」，也不给轨迹打分。它只做一件事：把一段真实执行轨迹里**数得出来的**行为差异拿来对比——工具错误几次、重复调用几次、几步收尾、产出几个文件、模型读了多少字符——然后只保留**可观测变好**的外壳改动。

这层外壳可以改的东西是封闭的：提示词段落、探测模型路由、预算。改不了工具，改不了权限，改不了内核。

在 DSH `0.2.2-rc.1`（Windows 桌面版 / `dsh web`）上实测通过：57 项离线测试全绿，客户端控制栏与宿主侧状态文件均在真实运行中验证。

---

## 1. 它做什么

一次 Harness² 运行（PPC）分三段：

| 阶段 | 做的事 | 产物 |
| --- | --- | --- |
| **Propose** | 从基线轨迹里读信号：有工具错误 → 生成「调用前校验」候选；有重复调用 → 生成「不许重复同参」候选；领域已知 → 生成领域候选；步数多且零产出 → 生成「先定产物路径」候选 | 1–3 个候选，每个都带它回答的那个数字 |
| **Probe** | 每个候选在**隔离子会话**里把同一个任务再跑一遍，候选只注册在子会话自己的上下文里，工作目录是临时沙箱 | 一份完整 Append-only 轨迹 |
| **Compose** | 用行为对比判定：变好才收；破坏执行（回合没结束、错误爆炸）直接 `unsafe` 拒绝；同一段提示词的候选冲突时留证据更强的那个 | 生效的外壳叠加层 + 经验条目 + 运行记录 |

`unsafe` 是硬闸，不是权重：一个让回合无法收尾的改动，无论省了多少错误都不会被平均掉。

**不做的事**：不评分、不比对参考轨迹、不生成「正确答案应为」这类判断、不改 DSH 源码或 `dsh-base`、不注册任何未经用户触发的模型调用。

---

## 2. 装上以后多出什么

| 入口 | 内容 |
| --- | --- |
| **工具** | `harness2_status`、`harness2_recall`、`harness2_experience`、`harness2_optimize` |
| **命令** | `/harness2 status \| list \| recall <查询> \| optimize [领域] \| forget <EXP-xxxx>`，直接在会话输入框执行，**零 token** |
| **侧边栏** | 左侧导航出现 `Harness²`，点开是控制栏：运行循环说明、五个命令的一键复制、落盘位置 |
| **系统提示** | 一段约十行的常驻说明（`harness-squared:shell`），让模型知道这些工具存在 |
| **动态注入** | 命中经验时，在 `system-prompt/assemble` 瀑布里追加一段围栏化的证据块 |

经验库落在磁盘上（默认 `~/.dsh/harness-squared/`）：

```
~/.dsh/harness-squared/
├── INDEX.md                       # 索引，每次写入原子重写
├── status.json                    # 最新状态快照（控制栏与排查用）
├── overlay.json                   # 当前生效的外壳叠加层
├── domains/<领域>/EXP-0001.md     # 经验条目，EXPERIENCE.md 结构
└── runs/RUN-*.json                # 每次 PPC 的完整运行记录（含被拒原因）
```

一条经验长这样（`EXPERIENCE.md` 结构，四段缺一不可）：

```markdown
---
id: EXP-0001
domain: database
kind: invariant
confidence: 0.75
origin: ppc
createdAt: 2026-11-06T11:20:04Z
tags: [database, invariant, migration]
---

# database: 调用前先确认目标 revision 存在

## 情境
任务：数据库迁移脚本一直失败
基线轨迹：read_file → read_file → write_file
触发信号：2 tool error(s) in 5 call(s)

## 变更内容
Before a tool call that writes or executes, confirm the target exists…

## 证据摘要
行为对比（基线 → 候选）：
- 工具错误：2 → 0
- 重复调用：2 → 0
- 步数：3 → 2
- 结论：better: better on toolErrors, repeatedToolCalls

## 复用条件
- 数据库
- migration
- alembic
```

---

## 3. 配置

行配置写在 bundle 的 `cordis.patch.yml`（随包发布），也可以在 Profiles → 该行里覆盖。所有字段都有可用默认值，空配置也是合法配置。

### 顶层

| 字段 | 默认 | 说明 |
| --- | --- | --- |
| `enabled` | `true` | 总开关；`false` 时什么都不注册 |
| `root` | `''` → `~/.dsh/harness-squared` | 经验库根目录。**建议写绝对路径**，相对路径会按宿主进程 cwd 解析 |
| `domains` | `[]` | 预先声明的领域；检索优先命中，运行中发现的领域会自动补进来 |

### `recall` — 经验召回

| 字段 | 默认 | 说明 |
| --- | --- | --- |
| `enabled` | `true` | 是否注入经验 |
| `maxEntries` | `3` | 单次注入条目上限（1–12） |
| `minScore` | `0.18` | 词法匹配下限（0–1），嫌吵就调高 |
| `order` | `11800` | 注入段在提示词里的排序位 |
| `describeShell` | `true` | 是否挂那段常驻的工具说明 |

### `ppc` — 递归优化

| 字段 | 默认 | 说明 |
| --- | --- | --- |
| `enabled` | `true` | 是否允许跑 PPC |
| `maxCandidates` | `3` | 单次运行候选上限（1–8），**探测前**检查 |
| `maxRounds` | `1` | 递归轮次上限（1–3） |
| `tokenBudget` | `120000` | 单次运行估算 token 上限 |
| `probeModel` | `deepseek-account / deepseek-flash` | 探测用便宜模型 |
| `reflectModel` | `deepseek-account / deepseek-pro`，`reasoningEffort: high` | 反思与合成用的高推理路由 |
| `probeTimeoutMs` | `180000` | 单个探测墙钟上限 |
| `minImprovement` | `1` | 至少几项指标变好才接受 |
| `isolateWorkspace` | `true` | 每个探测用独立临时 cwd；`false` 则 fork 会话并共享工作区 |

### `distill` / `evidence`

| 字段 | 默认 | 说明 |
| --- | --- | --- |
| `distill.enabled` | `true` | 是否把运行结果沉淀为经验 |
| `distill.onTurnEnd` | `false` | 是否把「干净且有产出的普通回合」也记为观察条目（置信度 0.35，标注"未经验证"） |
| `evidence.maxChars` | `1200` | 单条经验里任何文本的上限 |
| `evidence.maxInjectChars` | `2400` | 单次注入块上限 |

---

## 4. 安装

要求 DSH `0.2.x`。

```powershell
node install.mjs                       # 默认 profile: desktop
node install.mjs --profile desktop     # 显式指定
node install.mjs --dry-run             # 只打印计划
```

脚本做四件事，可重复执行：

1. 复制本包到 `<profile>/plugins/dsh-harness-squared`
2. 在 `<profile>/package.json` 写 `"dsh-harness-squared": "link:./plugins/dsh-harness-squared"`
3. 把 `"dsh-harness-squared"` 追加进 `dsh.profile.bundles` —— 这一步才会让宿主应用本包的 `cordis.patch.yml`
4. 建 `<profile>/node_modules/dsh-harness-squared` 目录联接（junction）指向第 1 步目录，即 `pnpm install` 本会生成的链接

完成后**完全退出并重启 DSH**。

手动安装等价于上面四步；第 4 步不能省，否则按包名解析不到模块。

### 验证

1. 新会话执行 `/harness2 status` → 立即返回经验库与预算状态（零 token）
2. 左侧栏出现 `Harness²` 条目
3. `~/.dsh/harness-squared/status.json` 出现，内容含解析后的配置

### 卸载

```powershell
node install.mjs --uninstall
```

反向四步。**经验库不删**——它按设计独立于插件存在。

---

## 5. 已知限制

这些是当前实现的真实边界，不是待办清单。

- **控制栏不做实时读数。** 第三方 bundle 无法向 Remote 命名空间注册方法（那是 Typert 代码生成面），转发事件白名单也是固定的，所以没有可用的宿主 RPC。宿主把状态写进 `status.json` / `overlay.json` / `runs/`，控制栏指向这些文件；要看即时状态用 `/harness2 status`（零 token）。这是平台约束，不是省事。
- **探测的隔离是「工作目录 + 提示词作用域」，不是工具级沙箱。** 探测拿不到主会话的提示词，写文件也落在会被删除的临时目录里；但 `tools.restrict()` 对未注册的工具名会抛错，所以没有强行按工具名禁用。
- **`isolateWorkspace: false` 会放弃文件隔离。** 那种模式下探测 fork 父会话、共用工作区；换来的是继承完整对话上下文。
- **单进程。** 同一时刻只允许一次运行；两个宿主进程共用一个 root 没有文件锁，`INDEX.md` 的原子性只保证进程内并发安全。
- **经验注入只是提示词文本。** 注入的内容无法自行授予权限、改工具或改路由；清洗器会把形如指令的行改成 `[quoted]` 并整块围栏化，但清洗器不是沙箱。
- **反思阶段不调用模型。** `reflectModel` 目前用于记录与未来扩展（合成是确定性算法）；把它接进合成会引入「模型说好不好」的判断，与"无标准答案"原则冲突。
- **探测花的是真钱。** 默认 3 候选 × 1 轮，Flash 路由；预算超了运行会停在记录里写明 `budget`。

---

## 6. 文件结构

```
dsh-harness-squared/
├── package.json           # dsh.bundle.patch + dsh.client 声明
├── cordis.patch.yml       # bundle 层配置（含完整注释）
├── tsconfig.json
├── src/                   # TypeScript 源码（严格模式，零 any 逃逸）
├── lib/                   # 编译产物 + 手写的浏览器半 bundle
│   ├── index.js           # 宿主半入口
│   └── client.js          # 浏览器半（lazy-CJS 工厂，无需构建）
├── test/                  # 离线测试套件（57 项）
├── docs/
├── install.mjs
├── README.md
└── DEVELOPMENT.md         # 架构、API 出处、测试映射、扩展指南
```

重新构建：`npx tsc -p tsconfig.json`（或 `npm run build`）。`lib/client.js` 是手写的，不会被 tsc 覆盖。

---

## 7. 许可

MIT
