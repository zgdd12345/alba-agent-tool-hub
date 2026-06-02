# agenthub 增量 2.5 — Agents（仅 Claude）Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: superpowers:subagent-driven-development. Steps use `- [ ]`.

**Goal:** 新增 **agents** 受管内容类型(Claude subagents,单 `.md` 文件,仅 Claude),写到 `~/.claude/agents/<name>.md`。这是增量 2(commands)的**近乎克隆**;前提增量,让 3a 的 profile 能纳管 agents。**填充** cc-switch 现有的占位 `agents` 视图(`AgentsPanel.tsx` 当前是 "under development" 空壳;View union/VALID_VIEWS/`case "agents"`/`t("agents.title")` 已在 App.tsx 注册)。

**Architecture:** 完全镜像 commands(增量 2,已实现+安全评审+合并)。蓝本提交:`git log b5348364..ca2df1eb`(commands 的 6 个提交)+ 计划 `docs/superpowers/plans/2026-06-02-agenthub-inc2-commands.md`。Deltas:`command→agent`、`~/.claude/commands/→~/.claude/agents/`、schema v11→**v12**、填充现有占位面板而非新建 tab。同时修 study 发现的 WebDAV 同步漏洞。

**Tech Stack:** 同增量 1/2。**环境 gotcha 同前**:cargo 在 `~/.cargo/bin`、`.cargo/config.toml` 强制 clang、前端用 `./node_modules/.bin/{vitest,tsc}`、慢网 `CARGO_NET_RETRY=10 CARGO_HTTP_LOW_SPEED_LIMIT=0`、Rust 测试 `CC_SWITCH_TEST_HOME`、**提交前 `cargo fmt`**、Rust 测试隔离用 `#[serial]`。

**安全(从 commands 继承,不可弱化):** AgentService 的 `validate_name`(`^[A-Za-z0-9._-]+$`,拒 `.`/`..`/分隔符)+ `file_path` 父目录断言 + 原子写 + **reconcile 只遍历 DB 行、绝不枚举目录删除**(用户自有 .md 不动)。必须有「symlink/unrelated user file 在 reconcile 后存活」+「路径穿越被拒」的测试。

---

## A1：agents 表(v12)+ InstalledAgent + DAO [Sonnet]

镜像 commands C1+C2(`commit b5348364` schema + `33a3e5a6` DAO/model)。

**Files:** Modify `database/mod.rs`(SCHEMA_VERSION 11→12)、`database/schema.rs`(agents 表 + `migrate_v11_to_v12`)、`database/tests.rs`;Create `database/dao/agents.rs`;Modify `database/dao/mod.rs`、`app_config.rs`(InstalledAgent)。

- **- [ ] Step 1 (TDD):** tests.rs 加 `fresh_db_has_agents_table`(同 commands 的 `fresh_db_has_commands_table`,改 agents);版本断言用符号常量(无需改字面)。跑确认失败。
- **- [ ] Step 2:** SCHEMA_VERSION 11→12。
- **- [ ] Step 3:** `create_tables_on_conn` 加 `agents` 表 DDL(与 commands 完全同构):`CREATE TABLE IF NOT EXISTS agents (id TEXT PRIMARY KEY, name TEXT NOT NULL, content TEXT NOT NULL DEFAULT '', description TEXT, tags TEXT, enabled_claude BOOLEAN NOT NULL DEFAULT 0, installed_at INTEGER NOT NULL DEFAULT 0)`。
- **- [ ] Step 4:** 加 `migrate_v11_to_v12`(仿 `migrate_v10_to_v11`,创建 agents 表 + log)+ 接进 `apply_schema_migrations_on_conn` 的 `11 =>` arm（`set_user_version(conn, 12)`)。
- **- [ ] Step 5:** `app_config.rs` 加 `InstalledAgent`(克隆 `InstalledCommand`:id/name/content/description/tags/enabled_claude/installed_at,camelCase)。
- **- [ ] Step 6:** `dao/agents.rs` 克隆 `dao/commands.rs`:`get_all_installed_agents/get_installed_agent/save_agent/set_agent_enabled/delete_agent`(tags JSON ser/de 同 commands);`dao/mod.rs` 加 `mod agents;`。TDD `agent_crud_roundtrip`(克隆 commands)。
- **- [ ] Step 7:** `cargo test --lib database::`(全绿)+ `cargo fmt`。
- **- [ ] Step 8:** commit `feat(db): add agents table (schema v12) + InstalledAgent + dao`。

## A2：AgentService + Tauri 命令 + 注册 + WebDAV 修复 [Sonnet]

镜像 commands C3+C4(`fa04988a` service + `e8c966fc` 命令);加 WebDAV 修复。

**Files:** Modify `config.rs`(get_agents_dir);Create `services/agent.rs`、`commands/agent.rs`;Modify `services/mod.rs`、`commands/mod.rs`、`lib.rs`、`services/webdav_auto_sync.rs`。

- **- [ ] Step 1:** `config.rs` 加 `pub fn get_agents_dir() -> PathBuf { get_claude_config_dir().join("agents") }`。
- **- [ ] Step 2 (TDD):** `services/agent.rs` 克隆 `services/command.rs` 的全部测试(create+enable 写 `~/.claude/agents/<name>.md`、disable 删、delete、**reconcile 不删无关用户文件**、name 校验拒 `../evil`/`a/b`、import、scan、update)。跑确认失败。
- **- [ ] Step 3:** `AgentService` 克隆 `CommandService`(同方法签名,目标改 `get_agents_dir()`/`<name>.md`;**安全语义不变**)。`services/mod.rs` 加 `pub mod agent;`。
- **- [ ] Step 4:** `commands/agent.rs` 克隆 `commands/command.rs`:7 个 `#[tauri::command]`(`get_installed_agents/create_agent/update_agent/delete_agent/set_agent_enabled/scan_unmanaged_agents/import_agents_from_disk`)+ `AgentServiceState`。`commands/mod.rs` 加 `mod agent; pub use agent::*;`。
- **- [ ] Step 5:** `lib.rs` 注册 7 个 handler + `.manage(AgentServiceState(...))`(仿 CommandServiceState)。
- **- [ ] Step 6（WebDAV 修复,study 发现的漏洞）:** `services/webdav_auto_sync.rs` 的 `should_trigger_for_table` 允许列表加 **`"commands"` 和 `"agents"`**(commands 之前漏了 → 顺手补)。
- **- [ ] Step 7:** `cargo build` + `cargo test`(无回归)+ `cargo fmt` + `cargo clippy -- -D warnings`(净;新 service 方法已被命令层消费,无 dead_code)。
- **- [ ] Step 8:** commit `feat(agents): AgentService + tauri commands + registration; fix webdav sync allowlist (commands+agents)`。

## A3：前端 — 填充 Agents 面板 + hooks/api + i18n [Sonnet]

镜像 commands C5(`65543b88`)。**关键:填充现有占位 `AgentsPanel`,不新建 tab**(`agents` 视图已注册)。

**Files:** Create `src/lib/api/agents.ts`、`src/hooks/useAgents.ts`、`src/components/agents/AgentEditDialog.tsx`、`tests/components/AgentsPanel.test.tsx`;Modify `src/components/agents/AgentsPanel.tsx`(替换占位实现)、`src/App.tsx`(agents 视图的 create-action 按钮 + ref + 改为 Claude-only gate)、`src/i18n/locales/{en,zh,zh-TW,ja}.json`(扩 `agents.*`)。

- **- [ ] Step 1:** `lib/api/agents.ts` 克隆 `lib/api/commands.ts`(`InstalledAgent`/`UnmanagedAgent` 类型 + `agentsApi.{getInstalled,create,update,delete,setEnabled,scanUnmanaged,importFromDisk}`,invoke 对应 A2 命令名)。
- **- [ ] Step 2:** `hooks/useAgents.ts` 克隆 `useCommands.ts`(query key `["agents","installed"]`)。
- **- [ ] Step 3:** `components/agents/AgentsPanel.tsx` —— **替换 "under development" 占位**为 CommandsPanel 的克隆(forwardRef + `openCreate()` handle、ListItemRow 列表、Switch/Edit/Delete、ConfirmDialog、空态);`AgentEditDialog.tsx` 克隆 `CommandEditDialog`。
- **- [ ] Step 4:** `App.tsx` —— `agents` 视图已有 `case "agents"`/header title;补:`agentsPanelRef`、header 区 `{currentView === "agents" && <Button onClick={() => agentsPanelRef.current?.openCreate()}>...t("agents.create")...</Button>}`、把 `case "agents"` 渲染改为 `<AgentsPanel ref={agentsPanelRef} />`、并把 agents 视图的可见性 gate 到 Claude(`sharedFeatureApp === "claude"`,仿 hasCommandsSupport;若 agents 视图当前无 gate/toolbar 按钮,新增一个 Lucide 图标按钮如 `Bot`)。**注意不要动 `openclawAgents` 视图(那是 OpenClaw 的,无关)。**
- **- [ ] Step 5:** i18n —— `agents.title` 已存在;在 4 个 locale 补全 `agents.*` 其余键(镜像 `commands.*`:manage/create/edit/delete/deleteConfirm/empty/count/enable/name/content/description/tags/save/cancel/... ~23 键)。真翻译。
- **- [ ] Step 6:** `tests/components/AgentsPanel.test.tsx` 克隆 `CommandsPanel.test.tsx`。
- **- [ ] Step 7:** `./node_modules/.bin/tsc --noEmit`(净)+ `./node_modules/.bin/vitest run`(全绿,含新测试)+ `prettier --write` 新文件。
- **- [ ] Step 8:** commit `feat(agents): fill Agents panel + edit dialog + hooks/api + i18n (Claude-only)`。

## A4：全绿验证 + 总评审 + 收尾 [Opus]

- **- [ ] Step 1:** 全套:`cargo fmt --check` + `cargo clippy -- -D warnings` + `cargo test` + `tsc --noEmit` + `vitest run`。全绿。
- **- [ ] Step 2:** 独立确认 WebDAV 修复(`should_trigger_for_table` 含 commands+agents)。
- **- [ ] Step 3:** Opus 总评审 `main..agents` 分支:跨层一致(命令名/字段)、agents service 安全(克隆 commands 的安全语义未弱化:reconcile 不删未知文件、name 校验)、占位面板已正确替换、无回归、scope 干净;**复用 commands 已验证的安全测试模式**。
- **- [ ] Step 4:** finishing-a-development-branch(合并 main,合并后复验)。

---

## 自查 / 不做

- **spec 覆盖**:对应 spec §6(commands/agents 类内容,仅 Claude)。agents 是 3a profile 纳管 agents 的前提。
- **占位符**:无;每步是"克隆具体 commands 文件 + 这些 deltas"。
- **类型一致**:InstalledAgent(Rust↔TS camelCase);DAO/命令名/api 一致(agent_* / agentsApi)。
- **不做**:多工具 agents(仅 Claude;`~/.claude/agents/`)、GitHub 导入、backup、@tag(tags 列预留,3c 消费)。不动 OpenClaw 的 `openclawAgents` 视图。
- schema 序列:commands=v11,agents=**v12**,后续 3a profiles=v13。
