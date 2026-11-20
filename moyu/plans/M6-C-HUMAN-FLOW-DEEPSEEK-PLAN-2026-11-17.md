# MoYu M6-C 真人验收修复计划（DeepSeek 执行）

基线 `9b9bff9850d5f9d8f724f1205b40dc0bcb2be17e`。已比对 GitHub main 和现场 release。实现前重查 main，记录实际完整 SHA。依次读仓库 AGENTS.md、docs/DEPLOYMENT.md、docs/DEVELOPMENT-GUIDE.md、docs/DESIGN.md、docs/API-CONTRACTS.md。按 A、B、C 分段交付，D 独立调查。使用独立测试仓、运行实例和数据库。

## A. 接入仓库时准备 Git 副本，并解释失败和进度

- 已定位：`apps/api/src/routes/repositories.ts:166` 已调用 supervisor prepare/wake，但它没有替代任务的首次仓库克隆。`apps/supervisor/src/manager.ts:56` 生成 `git clone --no-tags`、再 checkout 锁定提交的 executor 命令；`integrations/superagent/patches/0024-pause-before-start.patch` 的 prepareDirectory 在 worker 启动前受 PREPARE_TIMEOUT_MS 约束。真人复现两次超时；下载包曾持续增长，约 30 KB/s，时限 600 秒。确认了完整克隆与时限冲突，线路限速成因待证。平台在此期间显示「已入队」，失败后缺清楚解释。
- 按仓库接入流程准备可复用 Git 副本，使用平台现有鉴权、实例及状态机制。提供准备中、成功、失败原因及仓库层重试，准备好才接受执行。进度必须来源于实际可观测阶段或 Git 输出，未知百分比时展示阶段即可。既有仓库可进入补准备流程，已创建任务可在原任务上恢复。
- 任务从该仓库的副本建立隔离工作区，必要时只更新所需对象，核对冻结提交并保留正确 GitHub origin、提交身份和推送行为。覆盖并发准备、失败后的半成品、重启、权限及仓库隔离。把任务 checkout 的准备阶段、超时原因和是否启动 worker 从 Bridge 传到会话，提供可读中文和实际可用的恢复入口。不要通过单纯延长任务超时解决。
- 验收：零任务时完成真实仓库准备；首任务不重复完整远端克隆且检出确切 SHA；人为注入下载失败、超时、重复点击与重启，能从仓库层恢复；失败任务不换任务 ID 即可重试，证明单 worker、无重复副作用。生产 lessons Issue 原单留作部署后验收，不重发。

## B. 修复 PR 返修路由、执行交接和接单解释

- 已定位：返修时 Bridge 发出 `decision.ignored`，其中 `reason.code=routed` 指向 PR #12 原 ticket。`integrations/superagent/patches/0022-link-and-ignore-events.patch` 将 reason 带到 Bridge；`apps/dispatcher/src/sync.ts:671` 的 applyIgnoredDecision 不区分原因，一律尝试投影为 not_accepted，API/会话只得到通用状态。普通非工单请求也因此看不到实际拒绝原因，且曾观察到回复 outbox 为 channelGone。channelGone 的触发条件须单独复现。
- 交接断点：输入已在原 ticket inputs 持久化，但无 deliveredTo/readAt，旧 assignment 标 running，execution.assignment 为空。原会话当前关联 PR #13，历史内还有 PR #12。尚未证明哪个分支造成状态不一致。先复现待审阅旧票与新任务入口转交，追查 admission、租约、运行中 assignment 与实际 worker 进程、undelivered 队列、PR 关联重绑和 outbox；记录触发条件后修复。已通过的 waiting_input 原会话回复是回归基线。
- 路由成功要显示「已转交」并能跳到合法同仓目标；符合条件时唤醒或调度目标工单并真实交接，遵守手动暂停和单执行槽，重复事件无重复工作。拒绝原因独立于自然语言回信落库，中文说明原因和下一步。历史 PR 链接保真，关联当前 PR 时明确身份，不把历史 PR 悄悄换成另一单的 PR。
- 验收：在隔离仓生成 PR，提交一项返修，证明同 ticket 接收/已读并更新同一 PR，仅修改约定文件，原会话有中文完成回执。覆盖新入口的 routed、普通未接单、outbox 失败、旧 assignment 未真正运行、重复事件、等待回复路径；不凭 running 标志或 HTTP 202 判定成功。若需改上游，只提交 MoYu 的补丁和契约测试。

## C. 恢复页面位置、修主题样式并落实中文提示词

- 刷新：`WorkspaceShell.tsx:128,141,170,768` 以空值和关闭状态初始化，并回退到 repositories[0]；`SettingsDialog.tsx:118` 及 `SuperagentSettings.tsx:215` 自行初始化分区和分组。让仓库、会话或设置位置可以从 URL 或明确持久化的位置恢复，先完成访问校验再回退。覆盖 beta 会话和 beta 设置→评估刷新、返回、失效或无权限目标、晚到请求。保留草稿恢复与跨会话隔离、设置切仓独立。
- 白框：`WorkspaceShell.module.css:2583` 的 `.newTaskForm select` 覆盖主题背景，变量缺失时回退 `#fff`；`NewTaskPanel.tsx` 的三个选择器共用 selectInput。桌面深色根因已定位，手机加载与未读取状态须先对比计算样式，确认对应控件。使用现有主题色修复，正文、焦点和选中态可读。用移动和桌面、浅色和深色覆盖加载、空/失败、正常态；取消不开任务。
- 中文：真人验收的接单和提问是中文，但 PR #14 的完成正文为英文。循 `apps/api/src/routes/superagent-settings.ts` 的 prompts preview、supervisor 代理与已锁定 Superagent 提示词合成定位实际生效的指令。以统一中文要求覆盖接单、提问、进度、失败和完成正文，文件名、路径、代码标识符与必要术语允许英文。保留已有用户提示词及版本历史。用预览和真实执行的交互与完成回执验证。不可把现象直接归因于某段英文系统词，先读实际合成。

## D. 手机通知调查，不阻塞 A–C

真人通知未通过。当前 `apps/web/src/lib/notifications.ts` 由页面实时事件调用 `new Notification`，并区分权限、可见性和点击跳转。复测时记录手机系统/浏览器、普通标签页或安装态、权限、切后台方式和真实事件；区分事件没到、浏览器能力缺失、通知生成失败和点击定位错误。先验证页面仍打开的支持范围，若关闭页面后也要推送，单列能力缺口和方案。不要劫持现有通知或为此扩大修复范围。

## 交付

每段先补能捕获实际回归的复现，后实现，对照已通过的草稿、设置、取消和等待回复流程。提交独立分支和 PR，按改动运行 `pnpm check`、`pnpm test:integration` 及对应浏览器或隔离 Git/Bridge 验证；确有环境依赖时写实际阻塞。记录红绿命令、退出码、完整 SHA、脱敏证据、未验项；如有迁移附回滚验证。以真实事件、PR diff 和工作区证据证明闭环。生产发布、现有 3082 引导实例和正式数据由所有者走现有流程。

## #79 完成后的遗留 PR 计划

本轮先完成 [Issue #79](https://github.com/bouyer-sutrasky/MoYu/issues/79) 的 A–D 工作和相应验收。以下工作留待 #79 之后再安排。#79 B 段涉及接单与转交时，应覆盖旧事件不释放新租约的回归，之后据此裁剪 #51 的剩余工作。

复审基线为 `main@9b9bff9850d5f9d8f724f1205b40dc0bcb2be17e`。#49 落后 13 个提交，其余均落后 93 个；旧分支 CI 曾通过，八项现有版本均未通过复审。后续在当时最新 main 上重查剩余差异，逐项复用可验证的改动。

| PR | 后续处理与验收 |
| --- | --- |
| #48 / #37 | 优先移植 `0014` 最新迁移保护。保留现有纯 SQL 与哈希校验，验证有后继迁移时拒绝且不改库，最新时可回滚并重新应用。 |
| #49 / #36 | 按现有每仓库身份重做隔离，解决路径替换竞争，实测专用 GitHub 凭据仅可访问目标测试仓。这是后续现场验收的前置。 |
| #50 / #39 | 修复迟到事件可能将执行保留记录改回旧分配的问题。补 Bridge、dispatcher 与两库的真实重启、打开 PR、后续消息及单 worker 回归。 |
| #51 / #38 | 先对照 #79 B 段和 main。仅补仍可复现的代次校验与执行保留记录结清缺口，验证旧事件不释放新租约。 |
| #52 / #40 | 保留有增量的跨任务与事件重放用例；隔离前置通过后补跑两个入口顺序、并发和单一副作用的真链路。 |
| #53 / #41 | 按现有 M6-C unit 移植 cgroup 冷启动与重绑的有效部分。验真实休眠、唤醒、重启、进程归属与租约。 |
| #54 / #42 | 保留现有方案和预检，待隔离前置通过后补测 8 GiB 边界和内存准入迟滞，并记录实际测量。 |
| #55 / #47 | 复用有效文档与 Web 证据，改掉命令参数传令牌和失败仍退出成功的问题，补真实 GitHub 入口的暂停恢复。 |

建议 #79 后先收 #48，再做 #49 与 #53，解除后续现场验收的依赖。分支更新到最新 main 并通过必需 CI，核实原需求和对应实测证据后再合并。
