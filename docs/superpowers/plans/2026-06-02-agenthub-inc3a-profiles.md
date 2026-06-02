# agenthub 增量 3a — Profiles 核心 + 激活流水线 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: superpowers:subagent-driven-development。每个 Step 用 `- [ ]`。环境 gotcha 见下方 Tech Stack;提交前必跑 `cargo fmt`。

**Goal:** 新增 **profiles**(per-tool 命名配置):绑定一个 provider + 一组启用内容(skills/commands/agents/mcp,按**字面名**),并提供**激活流水线**——切 provider + 把 4 类内容的启用标志翻成 profile 的精确集合并落盘——以及 Profiles 面板(建/激活/编辑、active 徽章)。一致性:激活 profile = 展示用真相源。

**Architecture:** ProfileService 是**无状态 unit struct**(仿 `ProviderService`/`McpService`,所有方法取 `&AppState`),激活时**复用** `ProviderService::switch` + 4 个既有 reconciler(`SkillService::sync_to_app`、`CommandService/AgentService::reconcile`、`McpService::sync_all_enabled`)。**安全核心**:激活只翻 DB enable-flag 然后跑 reconciler——reconciler 已被验证只遍历 DB 行/跳过未知用户文件,从而收敛磁盘到 profile 精确集且**绝不删用户文件**。schema 升 v13。前端整套克隆已交付的 agents 特性。

**关键决策(本计划已锁定,经深度设计 + 对抗审查):**
1. **apply_manifest 推迟到 3b**:4 类内容的 reconciler 已能安全收敛(flag 翻转 + reconcile 即删掉上个 profile 多余文件),manifest 在 3a 是冗余甚至会与 reconciler 打架。**v13 仍创建 apply_manifest + profile_dotfiles 表**(避免 3b 再迁移),但其 DAO/写入/diff-删除全部在 3b 实现。3a 的 `activate()` 是**纯 flag+reconcile**。
2. **Skill 同名冲突硬化(blocker 修复)**:激活启用 skill 时,若 `~/.<app>/skills/<dir>` 是**用户的真实(非软链)目录**,既有 `sync_to_app_dir` 会 `remove_dir_all` 销毁它。本增量改为**跳过 + 警告**,绝不删用户真实目录(同时保护逐项 toggle 路径)。
3. **reconciler 恒在最后无条件执行**:即使 provider switch 走了 proxy 热切换/OMO 提前返回分支,激活仍无条件跑 4 个 reconciler(显式不变量,防未来重构破坏收敛)。

**Tech Stack:** 同增量 1/2/2.5(Tauri 2 + Rust 1.95 + React/TS/vitest)。**仓库:`/Users/fsm/project/MyProject/agentplugin/cc-switch-cloud`(分支 main)。** 环境 gotcha(务必遵守):
- cargo 在 `~/.cargo/bin` → 命令前 `export PATH="$HOME/.cargo/bin:$PATH"`。
- `cc` 被劫持 → 仓库已有 gitignored `.cargo/config.toml` 强制 `/usr/bin/cc`,勿删。
- 前端**直接调** `./node_modules/.bin/vitest run` 与 `./node_modules/.bin/tsc --noEmit`(别用 `pnpm test:unit`/`pnpm typecheck`)。
- 慢网:cargo 前缀 `CARGO_NET_RETRY=10 CARGO_HTTP_LOW_SPEED_LIMIT=0`。
- **提交前 `cargo fmt`**;`cargo clippy -- -D warnings` 须净(新 pub 方法须被消费,否则 dead_code)。
- 纯 DB DAO 测试用 `Database::memory()`,**无** `#[serial]`;**触磁盘**的测试用 `CC_SWITCH_TEST_HOME` + `#[serial]`(见 backup.rs:784)。
- 验证别用 `… | grep -c …` 收尾(零匹配返回 1 假报失败);cargo 退出码用 `> log 2>&1; echo $?` 捕获。

**安全(不可弱化,继承自 commands/agents 并新增 skill 硬化):**
- 激活只写 DB flag + 跑 reconciler;**不**枚举目录删未知文件。
- `CommandService/AgentService::reconcile` 只遍历 DB 行 + `validate_name` 拒穿越。
- `SkillService::sync_to_app` 孤儿清理跳过未知非软链目录;**新增**:写入遍亦绝不删用户真实目录(T5)。
- 必须有「A→B 切换后用户自建 command/agent/skill 幸存」+「spec 列穿越名被 reconcile 拒」的测试(T7)。

---

## 范围 / 不做(YAGNI)

- **做**:profiles/profile_dotfiles/apply_manifest 表(v13);profiles DAO;激活流水线(provider + 4 类内容字面名);Profiles 面板 + active 徽章;skill 同名硬化;WebDAV 接线。
- **不做(后续)**:dotfile 渲染 / settings.json 深合并 / statusline.sh / CLAUDE.md 仲裁(3b);`${VAR}` 模板引擎(3b);apply_manifest 的写入/删除/diff(3b);@tag 解析 + skills.tags 列(3c);项目级通道(增量 4);多 app profile 创作(v1 UI 硬编码 `app='claude'`)。"Save current state to profile" 快照按钮(可选,默认不做)。
- `profile_dotfiles` / `apply_manifest` 表在 v13 **建好但 3a 不读写**(无 Rust DAO,纯 DDL,无 dead_code 风险)。

## 已记录的既定行为(对抗审查产出,实现者须知,无需额外代码)

- **再激活会重写 managed 文件**:reconcile 从 DB 重新落盘所有启用项;用户**直接在磁盘**改 managed 文件(非经 app)的内容会在下次激活被覆盖。未纳管文件(无 DB 行)安全。
- **跨设备悬挂 FK**:profiles 跨设备同步后,`current_provider_id` 可能指向本设备不存在的 provider(WebDAV import 时 foreign_keys=OFF,SET NULL 不触发)。3a 依赖**激活期 missing-provider 守卫**(warn + 跳过 switch + 仍激活内容)优雅降级,不崩溃。
- **foreign_keys pragma 已确认 ON**(`mod.rs:107`/`:166`),故 in-app DELETE 的 SET NULL/CASCADE 会真正触发。

---

## T1:Schema v13 — profiles/profile_dotfiles/apply_manifest 表 [Opus]

**Files:** Modify `src-tauri/src/database/mod.rs`(SCHEMA_VERSION 12→13)、`src-tauri/src/database/schema.rs`(3 表 DDL + `migrate_v12_to_v13` + match arm)、`src-tauri/src/database/tests.rs`(新 fresh-DB 测试 + 迁移测试)。

- **- [ ] Step 1 (TDD):** `tests.rs` 加三个 fresh-DB 测试,克隆 `fresh_db_has_agents_table`(tests.rs:767-778),分别断言 `profiles`/`profile_dotfiles`/`apply_manifest` 表存在:

```rust
#[test]
fn fresh_db_has_profiles_table() -> Result<(), AppError> {
    let db = Database::memory()?;
    let conn = crate::database::lock_conn!(db.conn);
    let n: i64 = conn.query_row(
        "SELECT count(*) FROM sqlite_master WHERE type='table' AND name='profiles'",
        [],
        |r| r.get(0),
    )?;
    assert_eq!(n, 1, "profiles table must exist on a fresh DB");
    Ok(())
}
// 同构再加 fresh_db_has_profile_dotfiles_table（name='profile_dotfiles'）
// 同构再加 fresh_db_has_apply_manifest_table（name='apply_manifest'）
```
跑 `export PATH="$HOME/.cargo/bin:$PATH"; cargo test --lib database::tests::fresh_db_has_profiles_table`,期望 FAIL(表不存在)。

- **- [ ] Step 2:** `mod.rs:52` 把 `pub(crate) const SCHEMA_VERSION: i32 = 12;` 改为 `= 13;`。

- **- [ ] Step 3:** `schema.rs` `create_tables_on_conn` 在 agents 块(122-134)之后、`Ok(())`(385)之前,加三个 `CREATE TABLE IF NOT EXISTS`(与 migrate 中字面一致):

```rust
        // 7. Profiles 表（增量 3a）
        conn.execute(
            "CREATE TABLE IF NOT EXISTS profiles (
            id                  TEXT PRIMARY KEY,
            app_type            TEXT NOT NULL,
            name                TEXT NOT NULL,
            description         TEXT,
            is_active           BOOLEAN NOT NULL DEFAULT 0,
            current_provider_id TEXT,
            spec                TEXT NOT NULL DEFAULT '{}',
            sort_index          INTEGER NOT NULL DEFAULT 0,
            created_at          INTEGER NOT NULL DEFAULT 0,
            UNIQUE (app_type, name),
            FOREIGN KEY (current_provider_id, app_type)
                REFERENCES providers(id, app_type) ON DELETE SET NULL
        )",
            [],
        )
        .map_err(|e| AppError::Database(e.to_string()))?;
        // 7b. profile_dotfiles（v13 建表，3a 不读写，3b 消费）
        conn.execute(
            "CREATE TABLE IF NOT EXISTS profile_dotfiles (
            profile_id TEXT NOT NULL,
            rel_path   TEXT NOT NULL,
            content    TEXT NOT NULL DEFAULT '',
            PRIMARY KEY (profile_id, rel_path),
            FOREIGN KEY (profile_id) REFERENCES profiles(id) ON DELETE CASCADE
        )",
            [],
        )
        .map_err(|e| AppError::Database(e.to_string()))?;
        // 7c. apply_manifest（v13 建表，3a 不读写，3b 消费）
        conn.execute(
            "CREATE TABLE IF NOT EXISTS apply_manifest (
            id          INTEGER PRIMARY KEY AUTOINCREMENT,
            channel     TEXT NOT NULL DEFAULT 'global',
            profile_id  TEXT,
            app_type    TEXT NOT NULL,
            target_path TEXT NOT NULL,
            kind        TEXT NOT NULL,
            created_at  INTEGER NOT NULL DEFAULT 0,
            FOREIGN KEY (profile_id) REFERENCES profiles(id) ON DELETE CASCADE
        )",
            [],
        )
        .map_err(|e| AppError::Database(e.to_string()))?;
```

- **- [ ] Step 4:** `schema.rs` 在 `migrate_v11_to_v12`(1263-1281)之后加 `migrate_v12_to_v13`(三表同上 DDL,字面与 Step 3 完全一致):

```rust
    /// v12 -> v13 迁移：添加 profiles / profile_dotfiles / apply_manifest 表
    fn migrate_v12_to_v13(conn: &Connection) -> Result<(), AppError> {
        conn.execute("CREATE TABLE IF NOT EXISTS profiles ( ... 同 Step 3 ... )", [])
            .map_err(|e| AppError::Database(e.to_string()))?;
        conn.execute("CREATE TABLE IF NOT EXISTS profile_dotfiles ( ... 同 Step 3 ... )", [])
            .map_err(|e| AppError::Database(e.to_string()))?;
        conn.execute("CREATE TABLE IF NOT EXISTS apply_manifest ( ... 同 Step 3 ... )", [])
            .map_err(|e| AppError::Database(e.to_string()))?;
        log::info!("v12 -> v13 迁移完成：已添加 profiles / profile_dotfiles / apply_manifest 表");
        Ok(())
    }
```

- **- [ ] Step 5:** `schema.rs` `apply_schema_migrations_on_conn` 的 match 中,在 `11 =>` arm(469-473)之后、`_ =>` 错误 arm(474-478)之前,插入:

```rust
                    12 => {
                        log::info!("迁移数据库从 v12 到 v13（添加 profiles 相关表）");
                        Self::migrate_v12_to_v13(conn)?;
                        Self::set_user_version(conn, 13)?;
                    }
```

- **- [ ] Step 6 (TDD):** `tests.rs` 加迁移测试(仿 `schema_migration_sets_user_version_when_missing` tests.rs:154-170):建内存 conn → `create_tables_on_conn` → `set_user_version(&conn, 12)` → `apply_schema_migrations_on_conn` → 断言 `get_user_version == SCHEMA_VERSION`(13) 且三表存在(`sqlite_master` count)。

```rust
#[test]
fn schema_migration_v12_to_v13_reaches_current() {
    let conn = Connection::open_in_memory().expect("open memory db");
    Database::create_tables_on_conn(&conn).expect("create tables");
    Database::set_user_version(&conn, 12).expect("seed v12");
    Database::apply_schema_migrations_on_conn(&conn).expect("apply migration");
    assert_eq!(Database::get_user_version(&conn).expect("version"), SCHEMA_VERSION);
    for t in ["profiles", "profile_dotfiles", "apply_manifest"] {
        let n: i64 = conn
            .query_row(
                "SELECT count(*) FROM sqlite_master WHERE type='table' AND name=?1",
                [t],
                |r| r.get(0),
            )
            .expect("count");
        assert_eq!(n, 1, "table {t} must exist after migration");
    }
}
```

- **- [ ] Step 7:** `cargo test --lib database::`(全绿)+ `cargo fmt`。
- **- [ ] Step 8:** commit `feat(db): add profiles/profile_dotfiles/apply_manifest tables (schema v13)`。

## T2:Rust structs Profile/ProfileSpec/ProfileContent [Sonnet]

**Files:** Modify `src-tauri/src/app_config.rs`(在 `InstalledAgent` 附近加)。

- **- [ ] Step 1:** 加结构(serde camelCase,默认值齐全;**不**加 ApplyManifestEntry——3b):

```rust
#[derive(Debug, Clone, Default, serde::Serialize, serde::Deserialize)]
pub struct ProfileContent {
    #[serde(default)]
    pub skills: Vec<String>,   // 字面 skill.directory
    #[serde(default)]
    pub commands: Vec<String>, // 字面 command.name
    #[serde(default)]
    pub agents: Vec<String>,   // 字面 agent.name
    #[serde(default)]
    pub mcp: Vec<String>,      // 字面 mcp server id
}

#[derive(Debug, Clone, Default, serde::Serialize, serde::Deserialize)]
pub struct ProfileSpec {
    #[serde(default)]
    pub content: ProfileContent,
    #[serde(default)]
    pub vars: serde_json::Map<String, serde_json::Value>, // 预留 3b ${VAR}
}

#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct Profile {
    pub id: String,
    pub app_type: String,
    pub name: String,
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub description: Option<String>,
    #[serde(default)]
    pub is_active: bool,
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub current_provider_id: Option<String>,
    #[serde(default)]
    pub spec: ProfileSpec,
    #[serde(default)]
    pub sort_index: i64,
    #[serde(default)]
    pub created_at: i64,
}
```

- **- [ ] Step 2:** `cargo build`(编译过;字段未用的 warning 由 T3 消费消除)。无独立测试(T3 round-trip 覆盖)。`cargo fmt`。
- **- [ ] Step 3:** commit `feat(model): add Profile/ProfileSpec/ProfileContent structs`。

## T3:profiles DAO(CRUD + set/clear_active)[Sonnet]

**Files:** Create `src-tauri/src/database/dao/profiles.rs`;Modify `src-tauri/src/database/dao/mod.rs`(加 `mod profiles;`)。蓝本:`dao/agents.rs`(序列化模式)+ `dao/providers.rs:290-310`(set_current_provider 单行不变量事务)。

- **- [ ] Step 1 (TDD):** `dao/profiles.rs` 测试模块(`Database::memory()`,无 `#[serial]`):
  - `profile_dao_crud_roundtrip`:构造 `Profile`(spec.content 含若干 skills/commands/agents/mcp + vars 一键)→ `save_profile` → `get_profile` 取回断言 spec JSON 往返一致 → `get_all_profiles` 长度 1 → `delete_profile` 返回 true → 再取为 None。
  - `set_active_profile_single_row_invariant_per_app`:插 3 个 claude profile;`set_active_profile("claude", A)` 后 `set_active_profile("claude", B)`;断言恰好一个 `is_active=1` 且为 B(`get_active_profile("claude")` 返 B)。再插一个 codex profile 并置活;断言 claude 的 active 不受影响。
  跑确认 FAIL。

- **- [ ] Step 2:** 实现 `impl Database`(放 `dao/profiles.rs`,DAO 用 `lock_conn!(self.conn)`):
  - `get_all_profiles(&self) -> Result<Vec<Profile>, AppError>`、`get_profiles_for_app(&self, app_type: &str)`、`get_profile(&self, id: &str) -> Result<Option<Profile>, AppError>`、`get_active_profile(&self, app_type: &str) -> Result<Option<Profile>, AppError>`(`WHERE app_type=?1 AND is_active=1 LIMIT 1`)。行→Profile 映射:`spec` 列 `serde_json::from_str().unwrap_or_default()`(仿 agents tags 解析的容错)。
  - `save_profile(&self, p: &Profile) -> Result<(), AppError>`:`INSERT OR REPLACE`,`spec` 用 `serde_json::to_string(&p.spec)?`,`is_active` 存 BOOL。
  - `delete_profile(&self, id: &str) -> Result<bool, AppError>`:`DELETE ... WHERE id=?1`,返回 `changes>0`(CASCADE 自动清 dotfiles/manifest)。
  - `set_active_profile(&self, app_type: &str, id: &str) -> Result<(), AppError>`:**事务**——`UPDATE profiles SET is_active=0 WHERE app_type=?1` 再 `UPDATE profiles SET is_active=1 WHERE id=?2 AND app_type=?1`(结构同 `set_current_provider`)。
  - `clear_active_profile(&self, app_type: &str) -> Result<(), AppError>`:`UPDATE profiles SET is_active=0 WHERE app_type=?1`。
- **- [ ] Step 3:** `dao/mod.rs` 加 `mod profiles;`。
- **- [ ] Step 4:** `cargo test --lib database::`(全绿)+ `clippy -- -D warnings`(净;方法将由 T6/T8 消费,若此刻 dead_code 告警,确认 T6/T8 会消费即可——本任务后若仍告警则保留到 T6 合并验证)+ `cargo fmt`。
- **- [ ] Step 5:** commit `feat(db): profiles DAO (crud + set/clear active profile)`。

## T4:解锁 Command/AgentService reconcile(去 dead_code)[Sonnet]

**Files:** Modify `src-tauri/src/services/command.rs`(212)、`src-tauri/src/services/agent.rs`(212)。

- **- [ ] Step 1:** 删除两处 `reconcile` 上方的 `#[allow(dead_code)]`(因 T6 即将调用,不再 dead)。**不改任何逻辑**。
- **- [ ] Step 2:** `cargo build`(此刻可能短暂 dead_code 告警,因 T6 尚未调用——若 clippy -D 失败,本步可暂留 `#[allow(dead_code)]` 并在 T6 移除;优先做法:本任务与 T6 在同一审查周期连续执行,T6 commit 时确认告警消失)。现有 `reconcile_does_not_delete_unrelated_user_file`(command.rs 测试)等仍过。`cargo fmt`。
- **- [ ] Step 3:** commit `refactor(services): un-gate command/agent reconcile for profile activation`。

> 注:为避免 dead_code 真空期,subagent-driven 执行时 **T4 与 T6 连续执行、同一轮审查**;若单独提交 T4 触发 `clippy -D warnings`,实现者改为在 T6 内一并移除两处 `#[allow(dead_code)]`(把本任务并入 T6 Step)。

## T5:Skill 安全硬化 — sync_to_app_dir 绝不删用户真实目录 [Opus]

**Files:** Modify `src-tauri/src/services/skill.rs`(`sync_to_app_dir` 约 1586-1647 及其 `dest.exists() && !is_symlink` 覆盖分支 ~1603-1609 → `replace_dest_with_copy` ~1714-1718)。

**不变量:** `sync_to_app_dir` **绝不** `remove_dir_all` 一个**真实(非软链)目录**。仅当 `dest` 不存在、或 `dest` 是软链(我方可安全替换的受管产物)时才物化;若 `dest` 是真实目录,**跳过 + 警告**,保留用户数据。

- **- [ ] Step 1:** 先读 `sync_to_app_dir` 与 `replace_dest_with_copy`/`is_symlink`/`is_symlink_to_ssot` 实体,确认 symlink-mode 与 copy-mode 两条路径。
- **- [ ] Step 2 (TDD,触磁盘 → `CC_SWITCH_TEST_HOME` + `#[serial]`):** 加测试 `sync_to_app_dir_preserves_user_real_dir_on_name_collision`:
  - 设 SSOT 有一个 skill `dir=collide`(含合法 `SKILL.md`)。
  - 在 `get_app_skills_dir(Claude)/collide` 预先创建一个**真实目录**,内放 sentinel 文件 `USER_OWNED.txt` 内容 `"keep me"`。
  - 调 `SkillService::sync_to_app_dir("collide", &AppType::Claude)`。
  - 断言:`USER_OWNED.txt` 仍存在且内容不变(用户目录未被销毁)。
  跑确认 FAIL(现有实现会删)。
- **- [ ] Step 3:** 改 `sync_to_app_dir`:在物化前判断 `dest`——
  - `dest` 不存在 → 照常物化(symlink/copy)。
  - `dest` 是软链 → 照常替换(移除软链再建)。
  - `dest` 是**真实目录(非软链)** → **不删**,`log::warn!("skill '{dir}' 同名用户目录已存在于 {app} skills 目录,跳过以保护用户数据")` 并 `return Ok(())`。
  - 用既有 `Self::is_symlink(&dest)` 判别。**优先做法:真实目录一律跳过+warn。**
- **- [ ] Step 4:** 跑**全部** skill 测试:`cargo test --lib services::skill`。
  - **若**某个 copy-mode「已物化的 managed 目录被再次同步以更新内容」的既有测试因此失败(真实目录被跳过导致不更新):改用 **content-hash 区分**——对真实目录计算 `compute_dir_hash`(skill.rs:830)与 SSOT 源比较:hash 相同→无需更新,跳过;hash 不同→无法区分是"用户改了 managed 拷贝"还是"用户自建同名目录",**保守跳过+warn**(宁可不更新也不销毁)。在计划注记此 copy-mode 取舍。
  - **若**无既有测试因此失败:维持 Step 3 的"真实目录一律跳过"。
- **- [ ] Step 5:** `cargo test --lib services::`(全绿)+ `clippy -- -D warnings` + `cargo fmt`。
- **- [ ] Step 6:** commit `fix(skill): never delete a user real dir on name collision (skip + warn)`。

## T6:ProfileService::activate + deactivate 流水线 [Opus]

**Files:** Create `src-tauri/src/services/profile.rs`;Modify `src-tauri/src/services/mod.rs`(加 `pub mod profile;`)。**若 T4 未单独提交**,在本任务移除 command.rs/agent.rs 两处 `#[allow(dead_code)]`。

**ProfileService = 无状态 unit struct**(`pub struct ProfileService;`),方法取 `&AppState`。`ActivateResult` 仿 `SwitchResult`(provider/mod.rs:48-53):

```rust
#[derive(Debug, serde::Serialize, Default)]
#[serde(rename_all = "camelCase")]
pub struct ActivateResult {
    pub warnings: Vec<String>,
}
```

**activate 算法(有序,不变量:第 4 步 reconciler 恒在最后、无条件执行):**
1. `let profile = state.db.get_profile(profile_id)?.ok_or_else(|| AppError::Message("profile 不存在".into()))?;` 断言 `profile.app_type == app_type.as_str()`(否则 Err)。
2. **provider 切换(复用)**:若 `profile.current_provider_id == Some(pid)`:
   - 若 `state.db.get_provider_by_id(&pid, app_type.as_str())?.is_none()` → `result.warnings.push(format!("provider 不存在，已跳过切换: {pid}"))`,**不中止**。
   - 否则 `let sr = ProviderService::switch(state, app_type, &pid)?; result.warnings.extend(sr.warnings);`(此调用透传完成 device-settings/`is_current`/native live 写入/Hermes/MCP sync;proxy 热切换提前返回自动遵守)。
   - 若 `None` → 跳过切换。
3. **翻 4 类 enable-flag 到 spec 精确集**(仅 DB 写,字面名,**全覆盖**;收集未匹配名进 warnings):
   - commands:`let want: HashSet<&str> = profile.spec.content.commands.iter().map(String::as_str).collect();` `for c in state.db.get_all_installed_commands()? { state.db.set_command_enabled(&c.id, want.contains(c.name.as_str()))?; }` 之后 `for n in &profile.spec.content.commands { if !commands.iter().any(|c| &c.name==n) { warnings.push(format!("command 未找到: {n}")); } }`。
   - agents:同构(`get_all_installed_agents` / `set_agent_enabled`)。
   - skills(per-app,键=directory):`for skill in get_all_installed_skills().values() { let mut apps = skill.apps.clone(); apps.set_enabled_for(&app_type, want_skills.contains(skill.directory.as_str())); state.db.update_skill_apps(&skill.id, &apps)?; }`(仅动本 app 标志);未匹配 directory → warn。
   - mcp(per-app,键=server id):`for server in get_all_mcp_servers().values_mut() { server.apps.set_enabled_for(&app_type, want_mcp.contains(server.id.as_str())); state.db.save_mcp_server(server)?; }`;未匹配 id → warn。
4. **reconcile 一次/类(恒在最后,无条件)**:
   - `SkillService::sync_to_app(&state.db, &app_type)?;`
   - `CommandService::new(state.db.clone()).reconcile()?;`(临时构造,因其持 Arc<Database>)
   - `AgentService::new(state.db.clone()).reconcile()?;`
   - `McpService::sync_all_enabled(state)?;`(**必须再跑**:第 2 步内的 MCP sync 发生在第 3 步翻 flag 之前;幂等双循环可重复)
5. `state.db.set_active_profile(app_type.as_str(), profile_id)?;`(**最后**做,中途失败则保留旧 active)。
6. `Ok(result)`。

`deactivate(state, app_type) = state.db.clear_active_profile(app_type.as_str())`(仅清 flag,磁盘不动)。切换 = 再 `activate(other)`,第 3 步全覆盖即移除上个 profile 多余项;无需 deactivate-between。

- **- [ ] Step 1 (TDD):** `services/profile.rs` 测试模块(集成,触磁盘的用 `CC_SWITCH_TEST_HOME` + `#[serial]`;纯 DB 的用 `Database::memory()`)。先写下列测试并确认 FAIL:
  - `activate_flips_command_and_agent_flags_to_exactly_spec`:seed commands [a,b,c] + agents [x,y] 全 enabled;profile spec commands=[b] agents=[x];activate;断言 DB 中 b enabled、a/c disabled、x enabled、y disabled。
  - `activate_flips_skill_and_mcp_flags_for_app_only`:seed skills/mcp 各带混合 per-app flag;claude profile spec 含子集;activate;断言每个 claude flag 与 spec 成员一致 **且** 同一项的非 claude(如 codex)flag **不变**。
  - `activate_with_missing_provider_warns_not_aborts`:profile.current_provider_id 指向不存在 provider;activate 返回 Ok,warnings 含 "provider 不存在",内容 flag 仍翻、is_active 仍置。
  - `activate_reuses_provider_switch_sets_is_current`:profile 绑一个存在的(非 proxy)provider;activate 后该 provider `is_current=1`(证明复用了 switch,非仅翻 flag)。
  - `activate_collects_warnings_for_unknown_spec_names`:spec 列一个不存在的 command 名 → warnings 含 "command 未找到: …",其余项正常收敛。
  - `deactivate_clears_active_leaves_disk`:activate A(文件落盘)→ deactivate(claude)→ `get_active_profile==None` 且 A 的启用文件仍在。
- **- [ ] Step 2:** 实现 `ProfileService`(上述算法)。`services/mod.rs` 加 `pub mod profile;`。若 T4 未提交,本步移除两处 `#[allow(dead_code)]`。
- **- [ ] Step 3:** 跑测试至全绿;`cargo test --lib services::`(无回归)。
- **- [ ] Step 4:** `clippy -- -D warnings`(净;T3 的 DAO 方法、T2 字段此刻应全部被消费)+ `cargo fmt`。
- **- [ ] Step 5:** commit `feat(profile): ProfileService activate/deactivate pipeline (flag+reconcile, no manifest)`。

## T7:对抗式安全测试套 [Opus]

**Files:** Modify `src-tauri/src/services/profile.rs`(tests 模块,接 T6)。依赖 T5(skill 硬化)、T6(activate)。

- **- [ ] Step 1:** `switching_profiles_never_deletes_user_authored_command_file`(+ agent twin):在 `~/.claude/commands/` 放一个**无 DB 行**的用户 `mine.md`;activate A(commands=[a])再 activate B(commands=[b]);断言 `mine.md` 仍在(reconcile 只遍历 DB 行)。agent 同构。
- **- [ ] Step 2:** `switching_profiles_never_deletes_user_authored_skill_dir`(`CC_SWITCH_TEST_HOME` + `#[serial]`):在 app skills dir 放一个**真实非软链**用户目录(名与某 enabled managed skill 的 directory **相同**,含 sentinel);activate;断言 sentinel 幸存(依赖 T5 硬化)。
- **- [ ] Step 3:** `profile_name_path_traversal_blocked`:profile spec.content.commands 含 `"../evil"`、`"a/b"`;activate;断言不在 managed dir 外写文件、不返回 Err(无匹配 DB 行→no-op;若构造了非法名的 DB 行,reconcile 的 `validate_name` 跳过)。
- **- [ ] Step 4:** `cargo test --lib services::profile`(全绿)+ `cargo fmt`。
- **- [ ] Step 5:** commit `test(profile): adversarial safety suite (user-file preservation + traversal)`。

## T8:Tauri 命令层 + handler 注册 [Sonnet]

**Files:** Create `src-tauri/src/commands/profile.rs`;Modify `src-tauri/src/commands/mod.rs`、`src-tauri/src/lib.rs`。蓝本:`commands/agent.rs` + `commands/provider.rs:84-109`(switch_provider 的 State→&AppState 委派)。

- **- [ ] Step 1:** `commands/profile.rs` 8 个 `#[tauri::command]`(全部 `state: State<'_, AppState>` 直委派,**无** ServiceState):
  - `get_profiles(state) -> Result<Vec<Profile>, String>` → `db.get_all_profiles()`。
  - `get_profiles_for_app(app: String, state)` → `db.get_profiles_for_app(&app)`。
  - `create_profile(app, name, description, currentProviderId, spec, state) -> Result<Profile, String>`:生成 id(`format!("local:{}", uuid)` 或现有 id 生成惯例)、`created_at`=now、`sort_index`=max+1、`is_active=false`、`save_profile`;UNIQUE(app_type,name) 冲突映射友好错误。
  - `update_profile(id, name, description, currentProviderId, spec, state) -> Result<Profile, String>`:load→改→`save_profile`(**不**落盘)。
  - `delete_profile(id, state) -> Result<bool, String>` → `db.delete_profile`(CASCADE;3a 即使 active 也不做磁盘 teardown)。
  - `activate_profile(app, id, state) -> Result<ActivateResult, String>`:`AppType::from_str(&app)` → `ProfileService::activate(&state, app_type, &id)`。
  - `deactivate_profile(app, state) -> Result<(), String>` → `ProfileService::deactivate`。
  - `get_active_profile(app, state) -> Result<Option<Profile>, String>` → `db.get_active_profile`。
  - (**不做** get_profile_manifest——3b。)
- **- [ ] Step 2:** `commands/mod.rs` 加 `mod profile; pub use profile::*;`。
- **- [ ] Step 3:** `lib.rs` `generate_handler!` 在 "// Agent management" 块后加 "// Profile management" 块,登记 8 个命令。**无** `.manage()` 新增。
- **- [ ] Step 4:** `cargo build` + `cargo test`(无回归)+ `clippy -- -D warnings` + `cargo fmt`。
- **- [ ] Step 5:** commit `feat(profile): tauri command layer + handler registration`。

## T9:WebDAV 白名单 + skip/preserve [Sonnet]

**Files:** Modify `src-tauri/src/services/webdav_auto_sync.rs`(43-58)、`src-tauri/src/database/backup.rs`(18-34)。

- **- [ ] Step 1:** `should_trigger_for_table` 的 `matches!` 白名单加 `"profiles"` 与 `"profile_dotfiles"`(声明式、跨设备同步)。**不**加 apply_manifest。
- **- [ ] Step 2:** `SYNC_SKIP_TABLES`(18-25)加 `"apply_manifest"`;`SYNC_PRESERVE_TABLES`(27-34)亦加 `"apply_manifest"`(设备本地,仿 proxy_live_backup 同时在两表)。
- **- [ ] Step 3(若有现成 webdav/backup 表分类测试):** 加/扩断言 `should_trigger_for_table("profiles")==true`、`SYNC_SKIP_TABLES.contains("apply_manifest")`。否则跳过(纯常量改动)。
- **- [ ] Step 4:** `cargo test`(无回归)+ `cargo fmt`。
- **- [ ] Step 5:** commit `feat(sync): webdav allowlist profiles/profile_dotfiles; apply_manifest device-local`。

## T10:前端 api+hook+panel+dialog(克隆 agents)[Sonnet]

**Files:** Create `src/lib/api/profiles.ts`、`src/hooks/useProfiles.ts`、`src/components/profiles/ProfilesPanel.tsx`、`src/components/profiles/ProfileEditDialog.tsx`、`tests/components/ProfilesPanel.test.tsx`。蓝本:对应 `agents.ts`/`useAgents.ts`/`AgentsPanel.tsx`/`AgentEditDialog.tsx`(均已交付)。

- **- [ ] Step 1:** `lib/api/profiles.ts`:`InstalledProfile` 接口(`{ id, appType, name, description?, isActive, currentProviderId?, spec:{content:{skills,commands,agents,mcp:string[]},vars}, sortIndex, createdAt }`);`profilesApi`:`getInstalled`(invoke `get_profiles`)、`getForApp`(`get_profiles_for_app`)、`create`(`create_profile`)、`update`(`update_profile`)、`delete`(`delete_profile`)、`activate`(`activate_profile`,返回 `{ warnings: string[] }`)、`deactivate`(`deactivate_profile`)、`getActive`(`get_active_profile`)。invoke 命令名须与 T8 一致。
- **- [ ] Step 2:** `hooks/useProfiles.ts`:query key `["profiles","installed"]`;`useInstalledProfiles`/`useActiveProfile(app)`/`useCreateProfile`/`useUpdateProfile`/`useDeleteProfile`/`useActivateProfile`/`useDeactivateProfile`。`useActivateProfile.onSuccess` 失效 `["profiles","installed"]`、providers query、以及 4 个内容 query key(`["agents","installed"]`、commands、skills、mcp 各自的 key)——使徽章与各面板的逐项开关刷新。
- **- [ ] Step 3:** `components/profiles/ProfilesPanel.tsx`:克隆 AgentsPanel(`forwardRef` + `ProfilesPanelHandle { openCreate(): void }` + 同款 `useImperativeHandle`)。行展示 name/description;`isActive` 时显示 Active 徽章(`useActiveProfile`/列表 isActive,**不**用 proxy active_targets);**Activate 按钮**(`useActivateProfile`,成功后 toast `warnings`,空则 toast 成功);Edit/Delete + ConfirmDialog;空态。
- **- [ ] Step 4:** `components/profiles/ProfileEditDialog.tsx`:克隆 AgentEditDialog;字段:name、description、可选 provider 选择(从 providers query 取,存 `currentProviderId`)、4 个 `spec.content` 列表(skills 按 directory、commands/agents 按 name、mcp 按 server id;逗号/多选编辑)。**Save 仅持久化(create/update),不落盘**(落盘 = 显式 Activate)。
- **- [ ] Step 5:** `tests/components/ProfilesPanel.test.tsx`:克隆 `AgentsPanel.test.tsx`;加 `profiles_panel_active_badge_renders`(isActive 行显示徽章 + `openCreate()` 打开对话框)与 api/hook 形状校验(invoke 命令名映射、query key、activeProfileId 推导)。
- **- [ ] Step 6:** `./node_modules/.bin/tsc --noEmit`(净)+ `./node_modules/.bin/vitest run`(全绿,含新测试)+ `prettier --write` 新文件。
- **- [ ] Step 7:** commit `feat(profile): frontend api/hook/panel/edit-dialog (clone agents)`。

## T11:App.tsx four-touch + i18n(4 locale)[Sonnet]

**Files:** Modify `src/App.tsx`、`src/i18n/locales/{en,zh,zh-TW,ja}.json`。

- **- [ ] Step 1:** `App.tsx` 五处接线(精确点见 study):
  - View union(line 111 后)加 `| "profiles"`;`VALID_VIEWS`(157 后)加 `"profiles"`。
  - `profilesPanelRef`(252-253 旁:`const profilesPanelRef = useRef<any>(null);`)。
  - `renderContent()` 加 `case "profiles": return <ProfilesPanel ref={profilesPanelRef} />;`(仿 agents 901-902)。
  - header title(~1139-1151)加 `{currentView === "profiles" && t("profiles.title")}`。
  - toolbar create 按钮(仿 commands 1233-1243)加 `{currentView === "profiles" && <Button ... onClick={() => profilesPanelRef.current?.openCreate()}>{t("profiles.create")}</Button>}`。
- **- [ ] Step 2:** **nav 按钮(全局,无 app gate)**:复制 **MCP 无条件样式**(App.tsx:1539-1547,`className="...w-8 px-2"`,**无** `hasCommandsSupport`/`cn()` opacity gate),置于 default(非 openclaw/hermes)分支、MCP 旁。选一个 Lucide 图标(如 `Layers`)。**勿**用 commands 的 opacity-0/w-0 收起样式。
- **- [ ] Step 3:** i18n:4 个 locale 加 `profiles.*`,镜像 `agents.*` 全部键(title/manage/create/edit/delete/deleteConfirm/deleteConfirmDescription/empty/emptyDescription/count/enable/name/namePlaceholder/content?/description/descriptionPlaceholder/tags?/save/cancel/createSuccess/deleteSuccess/nameInvalid)**并新增 profiles 专属键**:`activate`、`active`、`activeModified`、`provider`、`activateSuccess`、`activateWarnings`、`skills`/`commands`/`agents`/`mcp`(内容列表标签)。**真翻译**(en/zh/zh-TW/ja)。
- **- [ ] Step 4:** `tsc --noEmit` + `vitest run`(全绿)+ `prettier --write`。
- **- [ ] Step 5:** commit `feat(profile): wire Profiles view into App.tsx + i18n (4 locales)`。

## T12:全绿验证 + Opus 总评审 + 收尾 [Opus]

- **- [ ] Step 1:** 全套(在 cc-switch-cloud,带 PATH/网络前缀):`cargo fmt --check` + `cargo clippy -- -D warnings` + `cargo test`(全绿,记录通过数)+ `./node_modules/.bin/tsc --noEmit` + `./node_modules/.bin/vitest run`。
- **- [ ] Step 2:** 独立确定性核实:WebDAV 三处改动(grep `should_trigger_for_table` 含 profiles/profile_dotfiles;`SYNC_SKIP_TABLES`/`SYNC_PRESERVE_TABLES` 含 apply_manifest)。确认 `profile_dotfiles`/`apply_manifest` 在 3a **无任何 Rust 读写**(grep DAO/service 无引用)。
- **- [ ] Step 3:** Opus 总评审 `main..profiles-inc3a`:① **激活安全**——reconciler 恒最后无条件执行、用户文件零删除(commands/agents/skills)、T5 skill 硬化生效;② 跨层一致(命令名/字段/query key);③ **无 3b/3c scope 泄漏**(无 manifest 写入、无 dotfile 渲染、无 ${VAR}、无 @tag);④ 无回归;⑤ activate 算法与本计划逐条吻合。复用 commands/agents 已验证的安全测试模式作对照。
- **- [ ] Step 4:** finishing-a-development-branch:本地 `--no-ff` 合并到 `main`,**合并后复验全绿**,删除 `profiles-inc3a` 分支。

---

## 自查 / 一致性

- **spec 覆盖**:对应设计 spec 的 Profiles 支柱(per-tool profile = provider + 内容集 + dotfiles 之"内容集 + provider"部分;dotfiles 在 3b)。
- **类型一致**:Profile(Rust↔TS camelCase);DAO/命令名/api 一致(`get_profiles`/`activate_profile` ↔ `profilesApi`/query key `["profiles","installed"]`);`ActivateResult.warnings` 三层贯通(Rust→命令→toast)。
- **占位符**:无;每步给出实体代码或"克隆具体文件 + deltas"。
- **schema 序列**:commands=v11,agents=v12,**profiles 相关=v13**;3b 复用 v13 既建的 profile_dotfiles/apply_manifest,不再迁移。
- **审查修复落点**:blocker→T5;important(reconciler 恒最后)→T6 算法第 4 步不变量;important(再激活重写 managed 文件)→"已记录的既定行为";minor(get_provider_by_id / 悬挂 FK / under-application warnings / manifest scope)→T6 算法 + 决策 1。
- **不做**:见"范围/不做"。
