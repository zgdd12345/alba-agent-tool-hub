# agenthub 增量 2 — Commands（仅 Claude）Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax.

**Goal:** 在 AgentHub fork 中新增一个受管内容类型 **commands**（Claude slash-commands，单 `.md` 文件,仅 Claude),镜像现有 skills 功能但**简化**(无 GitHub 导入/更新、无 SSOT+软链——内容存 DB,启用时直写 `~/.claude/commands/<name>.md`)。产出:后端 commands 表+DAO+service+Tauri 命令,前端 Commands tab+面板,全套测试绿。

**Architecture（镜像 skills,简化）:** 新 `commands` 表(SCHEMA_VERSION 10→11)持有内容;`CommandService` 在 enable/create 时把 DB 内容写到 `~/.claude/commands/<name>.md`、disable/delete 时删文件(镜像 skills 的 `toggle_app`/`sync_to_app` reconcile **流程**,但单文件直写而非目录 SSOT+软链);Tauri 命令镜像 `commands/skill.rs` 的核心 CRUD;前端 Commands tab 镜像 skills 的 `UnifiedSkillsPanel`(去掉 GitHub/repo/backup/多 app 切换机制)。

**设计决策(请审阅):** 内容存 **DB `content` 列**(对齐 spec §2① 的 content 列 + cc-switch 的 prompts 模式),启用时**直写文件**(RENDER)而非 spec §6 所述"软链"。理由:单文件用户原创内容,直写比 SSOT+软链简单,且与增量 3 apply pipeline 的「RENDER 自有文件」一致。若你坚持软链模型,告知后我改为 SSOT+file-symlink。

**Tech Stack:** Tauri 2 / Rust 1.95 / rusqlite / React 18 + TS / TanStack Query;同增量 1。

**镜像来源(实现者按需读这些现有文件):** `src-tauri/src/{services/skill.rs, commands/skill.rs, database/dao/skills.rs, database/schema.rs, app_config.rs, config.rs, lib.rs}`;`src/{App.tsx, components/skills/UnifiedSkillsPanel.tsx, hooks/useSkills.ts, lib/api/skills.ts}`。来源情报由 2026-06-02 分层 study workflow 实测。设计 spec:`2026-06-01-agenthub-gui-ccswitch-fork-design.md` §2①/§6/§7。

---

## 前置约定（每个任务通用）

- 工作目录:`/Users/fsm/project/MyProject/agentplugin/cc-switch-cloud`,新建分支 `inc2-commands`(从 main)。
- **环境**(同增量 1):cargo 在 `~/.cargo/bin`(`export PATH="$HOME/.cargo/bin:$PATH"`);仓库 `.cargo/config.toml` 已强制真 clang;前端用 `./node_modules/.bin/{vitest run,tsc --noEmit}`(勿用 pnpm wrapper);慢网加 `CARGO_NET_RETRY=10 CARGO_HTTP_LOW_SPEED_LIMIT=0`;Rust 测试用 `CC_SWITCH_TEST_HOME` 隔离 HOME。
- 验证命令:`cargo test --manifest-path src-tauri/Cargo.toml`;`./node_modules/.bin/vitest run`;`./node_modules/.bin/tsc --noEmit`;`cargo fmt --check --manifest-path src-tauri/Cargo.toml`;`cargo clippy --manifest-path src-tauri/Cargo.toml -- -D warnings`。
- 安全网:每任务后跑相关测试 + 任务末跑全套;基线 main = 1484 Rust + 284 前端,全绿。

## 文件结构（本增量）

- **新建后端**:`src-tauri/src/database/dao/commands.rs`、`src-tauri/src/services/command.rs`、`src-tauri/src/commands/command.rs`
- **改后端**:`src-tauri/src/database/schema.rs`(commands 表 + migrate_v10_to_v11)、`src-tauri/src/database/mod.rs`(SCHEMA_VERSION)、`src-tauri/src/database/dao/mod.rs`(声明 commands)、`src-tauri/src/database/tests.rs`(断言)、`src-tauri/src/app_config.rs`(InstalledCommand 结构)、`src-tauri/src/config.rs`(get_commands_dir)、`src-tauri/src/commands/mod.rs` + `src-tauri/src/services/mod.rs`(模块声明)、`src-tauri/src/lib.rs`(generate_handler 注册 + CommandService state)
- **新建前端**:`src/lib/api/commands.ts`、`src/hooks/useCommands.ts`、`src/components/commands/CommandsPanel.tsx`、`src/components/commands/CommandEditDialog.tsx`
- **改前端**:`src/App.tsx`(View/VALID_VIEWS/hasCommandsSupport/ref/toolbar/header/renderContent)、`src/i18n/locales/{en,zh,zh-TW,ja}.json`(commands.* 命名空间)、`tests/components/CommandsPanel.test.tsx`(新)

---

## Task 0：建分支 + 确认基线绿

**- [ ] Step 1:** `cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && git checkout main && git checkout -b inc2-commands`
**- [ ] Step 2:** 确认基线:`export PATH="$HOME/.cargo/bin:$PATH"; cargo test --manifest-path src-tauri/Cargo.toml`(期望 1484/0)+ `./node_modules/.bin/vitest run`(期望 284)。Expected: 全绿。

---

## Task 1：DB schema — 新增 `commands` 表(v10→v11）

**Files:** Modify `src-tauri/src/database/mod.rs`、`src-tauri/src/database/schema.rs`、`src-tauri/src/database/tests.rs`

**- [ ] Step 1: 写失败测试** — 在 `src-tauri/src/database/tests.rs` 加:
```rust
#[test]
fn fresh_db_has_commands_table_and_v11() -> Result<(), AppError> {
    let db = Database::memory()?;
    let conn = crate::database::lock_conn!(db.conn);
    // table exists
    let n: i64 = conn.query_row(
        "SELECT count(*) FROM sqlite_master WHERE type='table' AND name='commands'",
        [], |r| r.get(0))?;
    assert_eq!(n, 1, "commands table must exist");
    Ok(())
}
```
并把现有断言 `SCHEMA_VERSION` 的测试(tests.rs:154/172 附近)预期值改为 11。

**- [ ] Step 2: 跑测试确认失败** — `cargo test --manifest-path src-tauri/Cargo.toml --lib database::tests::fresh_db_has_commands_table_and_v11`。Expected: FAIL(no such table / version mismatch)。

**- [ ] Step 3: bump 版本** — `src-tauri/src/database/mod.rs:52` `SCHEMA_VERSION: i32 = 10` → `11`。

**- [ ] Step 4: 加表 DDL** — 在 `src-tauri/src/database/schema.rs` 的 `create_tables_on_conn`(约 line 24,紧邻 skills 表 83-104)加:
```rust
conn.execute_batch(
    "CREATE TABLE IF NOT EXISTS commands (
        id             TEXT PRIMARY KEY,
        name           TEXT NOT NULL,
        content        TEXT NOT NULL DEFAULT '',
        description    TEXT,
        tags           TEXT,
        enabled_claude BOOLEAN NOT NULL DEFAULT 0,
        installed_at   INTEGER NOT NULL DEFAULT 0
    );",
)?;
```
(无 repo/content_hash/updated_at/其他 app 列 —— 仅 Claude;`tags` 为 JSON 数组文本,为增量 3 的 @tag 预留。)

**- [ ] Step 5: 加增量迁移** — 仿 `migrate_v9_to_v10`(schema.rs:1181)写 `fn migrate_v10_to_v11(conn) -> Result<()>`(`CREATE TABLE IF NOT EXISTS commands ...` 同上 + `log::info!("v10 -> v11 迁移完成：已添加 commands 表")`),并接进 `apply_schema_migrations_on_conn`(schema.rs:365)的版本匹配(380-439,仿 v9→v10 那条 arm)。

**- [ ] Step 6: 跑测试确认通过** — `cargo test --manifest-path src-tauri/Cargo.toml --lib database::`。Expected: commands 表测试 + 版本测试 PASS。

**- [ ] Step 7: 提交** — `git add src-tauri/src/database/ && git commit -m "feat(db): add commands table (schema v11)"`

---

## Task 2：DAO + 结构体 — `InstalledCommand` + `dao/commands.rs`

**Files:** Create `src-tauri/src/database/dao/commands.rs`;Modify `src-tauri/src/database/dao/mod.rs`、`src-tauri/src/app_config.rs`

**- [ ] Step 1: 加结构体** — 在 `src-tauri/src/app_config.rs`(紧邻 `InstalledSkill`:169)加:
```rust
#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct InstalledCommand {
    pub id: String,
    pub name: String,
    pub content: String,
    pub description: Option<String>,
    #[serde(default)]
    pub tags: Vec<String>,
    pub enabled_claude: bool,
    pub installed_at: i64,
}
```

**- [ ] Step 2: 写失败测试** — 在 `dao/commands.rs`(新文件)底部 `#[cfg(test)] mod tests`:save→get round-trip、update_enabled、delete。例:
```rust
#[test]
fn command_crud_roundtrip() -> Result<(), AppError> {
    let db = Database::memory()?;
    let c = InstalledCommand { id: "local:fix".into(), name: "fix".into(),
        content: "# fix\n".into(), description: None, tags: vec![], enabled_claude: false, installed_at: 1 };
    db.save_command(&c)?;
    assert_eq!(db.get_all_installed_commands()?.len(), 1);
    db.set_command_enabled("local:fix", true)?;
    assert!(db.get_installed_command("local:fix")?.unwrap().enabled_claude);
    assert!(db.delete_command("local:fix")?);
    assert_eq!(db.get_all_installed_commands()?.len(), 0);
    Ok(())
}
```

**- [ ] Step 3: 跑确认失败** — `cargo test --manifest-path src-tauri/Cargo.toml --lib commands::tests::command_crud_roundtrip`。Expected: FAIL(方法未定义)。

**- [ ] Step 4: 实现 DAO** — 在 `dao/commands.rs` 写 `impl Database`,镜像 `dao/skills.rs` 的 `params![]`/row-mapping 形态:
- `get_all_installed_commands(&self) -> Result<Vec<InstalledCommand>, AppError>`(SELECT 全列;tags 列 JSON 解析为 Vec,空/NULL→vec![])
- `get_installed_command(&self, id: &str) -> Result<Option<InstalledCommand>, AppError>`
- `save_command(&self, c: &InstalledCommand) -> Result<(), AppError>`(`INSERT OR REPLACE INTO commands (...) VALUES (?1..?7)`;tags 序列化为 JSON 文本)
- `set_command_enabled(&self, id: &str, enabled: bool) -> Result<bool, AppError>`(`UPDATE commands SET enabled_claude=?1 WHERE id=?2`)
- `delete_command(&self, id: &str) -> Result<bool, AppError>`

**- [ ] Step 5: 声明模块** — `src-tauri/src/database/dao/mod.rs` 加 `mod commands;`(仿 `mod skills;` 所在行)。

**- [ ] Step 6: 跑确认通过** — `cargo test --manifest-path src-tauri/Cargo.toml --lib commands::tests`。Expected: PASS。

**- [ ] Step 7: 提交** — `git add src-tauri/src/database/ src-tauri/src/app_config.rs && git commit -m "feat(commands): InstalledCommand model + dao CRUD"`

---

## Task 3：Service — `services/command.rs`(直写 ~/.claude/commands/）

**Files:** Create `src-tauri/src/services/command.rs`;Modify `src-tauri/src/config.rs`、`src-tauri/src/services/mod.rs`

**- [ ] Step 1: 加路径解析** — `src-tauri/src/config.rs`(紧邻 `get_claude_config_dir`:36)加:
```rust
/// Claude slash-commands dir (~/.claude/commands)
pub fn get_commands_dir() -> PathBuf { get_claude_config_dir().join("commands") }
```

**- [ ] Step 2: 写失败测试**(用 `CC_SWITCH_TEST_HOME` 隔离,仿 skill 测试)— 在 `services/command.rs` 测试:
```rust
// set CC_SWITCH_TEST_HOME to a tempdir; create CommandService;
// save+enable a command -> ~/.claude/commands/<name>.md exists with content;
// disable -> file removed; reconcile removes managed-but-disabled file.
```
(用现有测试夹具风格;参考 skill.rs 测试如何设 TEST_HOME。)

**- [ ] Step 3: 跑确认失败。**

**- [ ] Step 4: 实现 CommandService** — `struct CommandService { db: Arc<Database> }`,方法:
- `commands_dir() -> Result<PathBuf>`(`get_commands_dir()` + `create_dir_all`)
- `file_path(name) -> PathBuf`(`commands_dir().join(format!("{name}.md"))`;name 已 slug 校验:`[A-Za-z0-9._-]+`,拒绝 `/`、`..`)
- `write_command_file(c: &InstalledCommand)`(原子写:temp + rename,仿 `config.rs::atomic_write` 若可复用,否则 write+rename;内容 = `c.content`)
- `remove_command_file(name)`(存在则删)
- `apply_enabled(c)`(enabled_claude→write_command_file,else→remove_command_file)
- `create(db, name, content, description, tags) -> InstalledCommand`(校验 name slug + 唯一;`id="local:<name>"`;`save_command`;若 enabled 则写文件)
- `update(db, id, content, description, tags)`(save_command;若 enabled 则重写文件)
- `set_enabled(db, id, enabled)`(`db.set_command_enabled` + apply_enabled)
- `delete(db, id)`(remove_command_file + `db.delete_command`)
- `reconcile(db)`(遍历 commands_dir 的 `*.md`:managed-but-disabled→删;再为每个 enabled 命令写文件。**只删 DB 里记录为本应用管理的文件,绝不删未知 .md**——镜像 skill.rs `sync_to_app` 的安全语义)
- `scan_unmanaged(db) -> Vec<UnmanagedCommand{name,path}>`(commands_dir 里存在但 DB 无记录的 .md)
- `import_from_disk(db, paths) -> Vec<InstalledCommand>`(读 .md 文件→create,enabled_claude=true)

**- [ ] Step 5: 声明模块** — `src-tauri/src/services/mod.rs` 加 `pub mod command;`(仿 skill)。

**- [ ] Step 6: 跑确认通过** — `cargo test --manifest-path src-tauri/Cargo.toml --lib command`。

**- [ ] Step 7: 提交** — `git add src-tauri/src/services/ src-tauri/src/config.rs && git commit -m "feat(commands): CommandService (write/remove ~/.claude/commands, reconcile, import)"`

---

## Task 4：Tauri 命令层 + 注册

**Files:** Create `src-tauri/src/commands/command.rs`;Modify `src-tauri/src/commands/mod.rs`、`src-tauri/src/lib.rs`

**- [ ] Step 1: 实现命令** — `commands/command.rs` 仿 `commands/skill.rs`(`CommandServiceState(Arc<CommandService>)` + 从 AppState 取 db),`#[tauri::command]`:
- `get_installed_commands(app_state) -> Result<Vec<InstalledCommand>, String>`
- `create_command(name, content, description: Option<String>, tags: Vec<String>, app_state) -> Result<InstalledCommand, String>`
- `update_command(id, content, description, tags, app_state) -> Result<InstalledCommand, String>`
- `delete_command(id, app_state) -> Result<bool, String>`
- `set_command_enabled(id, enabled: bool, app_state) -> Result<bool, String>`
- `scan_unmanaged_commands(app_state) -> Result<Vec<UnmanagedCommand>, String>`
- `import_commands_from_disk(paths: Vec<String>, app_state) -> Result<Vec<InstalledCommand>, String>`

**- [ ] Step 2: 声明模块** — `src-tauri/src/commands/mod.rs` 加 `mod command;` + `pub use command::*;`(仿 skill 所在行 commands/mod.rs:26,59)。

**- [ ] Step 3: 注册 handler** — `src-tauri/src/lib.rs` 的 `generate_handler!`(skill 区块 1209-1234 附近)加 7 个 `commands::<fn>`;并初始化 CommandService state(仿 skill service manage,lib.rs:873-875)。若 CommandService 仅需 db,可直接在命令里从 AppState 取 db,免单独 manage。

**- [ ] Step 4: 验证** — `export PATH="$HOME/.cargo/bin:$PATH"; cargo build --manifest-path src-tauri/Cargo.toml`(编译通过)+ `cargo test --manifest-path src-tauri/Cargo.toml`(全绿,无回归)。

**- [ ] Step 5: 提交** — `git add src-tauri/src/commands/ src-tauri/src/lib.rs && git commit -m "feat(commands): tauri command surface + handler registration"`

---

## Task 5：前端 — Commands tab + 面板 + i18n

**Files:** Create `src/lib/api/commands.ts`、`src/hooks/useCommands.ts`、`src/components/commands/CommandsPanel.tsx`、`src/components/commands/CommandEditDialog.tsx`、`tests/components/CommandsPanel.test.tsx`;Modify `src/App.tsx`、`src/i18n/locales/{en,zh,zh-TW,ja}.json`

**- [ ] Step 1: API 封装** — `src/lib/api/commands.ts` 仿 `lib/api/skills.ts`,`InstalledCommand` 类型(id/name/content/description?/tags/enabledClaude/installedAt)+ `commandsApi.{getInstalled,create,update,delete,setEnabled,scanUnmanaged,importFromDisk}`(各 `invoke(...)` 对应 Task 4 命令名)。

**- [ ] Step 2: hooks** — `src/hooks/useCommands.ts` 仿 `useSkills.ts` 核心子集:`useInstalledCommands()`(query key `["commands","installed"]`,staleTime Infinity)、`useCreateCommand()`、`useUpdateCommand()`、`useDeleteCommand()`、`useSetCommandEnabled()`(各 mutation invalidate `["commands","installed"]`)。

**- [ ] Step 3: 面板组件** — `CommandsPanel.tsx`(forwardRef,handle `{ openCreate() }`)仿 `UnifiedSkillsPanel`,**去掉** AppToggleGroup/AppCountBar/backup/zip/discovery/update;列表用 `ListItemRow` 每项:name+description(左)、enable 开关(Claude)+edit+delete(右);`ConfirmDialog` 删除确认;`CommandEditDialog.tsx`(name+content+description+tags 表单,CodeMirror/textarea 编辑 content)。

**- [ ] Step 4: App.tsx 接线** — 按 study 精确点:`View` union 加 `"commands"`;`VALID_VIEWS` 加;`const hasCommandsSupport = sharedFeatureApp === "claude" || sharedFeatureApp === "claude-desktop"`;`const commandsPanelRef = useRef<any>(null)`;app-switcher 工具栏加 Commands 图标按钮(Lucide `Terminal`,visibility 用 hasCommandsSupport,仿 skills 按钮);header title 加 `{currentView === "commands" && t("commands.title")}`;`renderContent` 加 `case "commands": return <CommandsPanel ref={commandsPanelRef} />`;header action 区加 `{currentView === "commands" && <Button onClick={() => commandsPanelRef.current?.openCreate()}>...新建...</Button>}`。

**- [ ] Step 5: i18n** — 4 个 locale 加 `commands.*` 命名空间(参照 skills.* 但仅核心:title/manage/empty/noInstalled/create/edit/delete/removeConfirm/enable/save/name/content/description/tags/import/installed/count…约 25 键)。en/zh/zh-TW/ja 都要加,值对应翻译。

**- [ ] Step 6: 前端测试** — `tests/components/CommandsPanel.test.tsx`(仿 `tests/components/ProviderPresetSelector.test.tsx`/skills 测试):mock `commandsApi`,渲染面板,断言空态 + 列表渲染 + 删除确认流程。

**- [ ] Step 7: 验证** — `./node_modules/.bin/tsc --noEmit`(净)+ `./node_modules/.bin/vitest run`(全绿,含新测试)。

**- [ ] Step 8: 提交** — `git add src/ tests/ && git commit -m "feat(commands): Commands tab + panel + hooks/api + i18n (Claude-only)"`

---

## Task 6：最终全绿验证 + 冒烟 + 总评审

**- [ ] Step 1: 全套检查** — `export PATH="$HOME/.cargo/bin:$PATH"; cargo fmt --check --manifest-path src-tauri/Cargo.toml; cargo clippy --manifest-path src-tauri/Cargo.toml -- -D warnings; cargo test --manifest-path src-tauri/Cargo.toml; ./node_modules/.bin/tsc --noEmit; ./node_modules/.bin/vitest run`。Expected: 全绿、clippy 零警告。fmt 若有差异先 `cargo fmt` 再提交。

**- [ ] Step 2: 冒烟** — `pnpm tauri dev`:Claude 视图出现 Commands tab → 新建一个命令(name=`hello`, content=`# hello`)并启用 → 确认 `~/.agenthub` 不受影响、`~/.claude/commands/hello.md` 写出且内容正确 → 停用 → 确认文件被删 → 删除命令。

**- [ ] Step 3: 总评审** — 派 Opus code-reviewer 审整个 `main..inc2-commands`(数据模型/迁移/service 安全删除语义/命令注册/前端接线/无回归)。

**- [ ] Step 4: 收尾** — 用 superpowers:finishing-a-development-branch(合并到 main,合并后复验测试)。

---

## 自查（spec 覆盖 / 占位符 / 类型一致）

- **spec 覆盖**:对应 spec §6(commands 仅 Claude)+ §2①(commands 表含 content)+ §7(Commands tab)。设计偏差(直写而非软链)已在 Architecture 标注待审。
- **占位符**:无 TBD;每个改动给了文件/位置/具体 schema/签名/命令名,或"镜像具体现有文件 + 这些具体改动"。
- **类型一致**:`InstalledCommand`(Rust app_config.rs ↔ TS lib/api/commands.ts,camelCase 对齐);DAO 方法名(save_command/get_all_installed_commands/set_command_enabled/delete_command)在 Task 2 定义,Task 3 service + Task 4 命令一致引用;Tauri 命令名(get_installed_commands/create_command/update_command/delete_command/set_command_enabled/scan_unmanaged_commands/import_commands_from_disk)在 Task 4 定义、Task 5 api 一致 invoke。

## 明确不做（v1 范围外）

- GitHub/上游导入与更新(skill 的 discover/check_updates/update/repos)——增量 5(source ingestion)再说。
- 多工具(Codex/OpenCode 等)的 command——仅 Claude;其它工具命令约定待实机验证后续增量。
- backup/restore、ZIP 导入——v1 不做(skill 有,commands 暂略)。
- @tag 驱动的 profile 选择——`tags` 列已预留,实际消费在增量 3(profiles)。
