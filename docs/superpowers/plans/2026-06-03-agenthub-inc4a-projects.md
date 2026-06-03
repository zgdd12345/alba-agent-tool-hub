# AgentHub Increment 4a — Projects (Claude content materialization, HARDENED)

For agentic workers: REQUIRED SUB-SKILL: superpowers:subagent-driven-development — execute each task below as its own subagent loop (write failing test → run → minimal impl → run → commit). Use the `- [ ]` checkbox syntax to track step completion; check a box only after the run command prints the expected PASS output.

**Goal:** Let a user bind a real on-disk project directory to an own content set (skills/commands/agents) and COPY-materialize Claude content into `<project>/.claude/{skills,commands,agents}/`, recording every written file in a `project_id`-scoped manifest so detach can safely owned-delete exactly what it wrote — never touching user files.

**Architecture:** A new device-local `projects` table (schema v17) plus a `project_id` column on `apply_manifest` give each binding an identity. A `ProjectApplyService` mirrors the global `ProfileService::activate/deactivate` flow but writes into an explicit project base dir (not `~/.claude`), forces `SyncMethod::Copy` for skills, and gates every write behind a canonical path-safety perimeter that makes a `<project>/.claude == ~/.claude` HOME-collapse impossible. The manifest channel key is `project:<canonical-abspath>` and reconcile is keyed on `project_id` so re-cloning a repo at a stale path can never owned-delete the new clone.

**Tech Stack:** Rust 1.95 (`agenthub_lib` crate, rusqlite, sha2, uuid v4, chrono, serial_test, tempfile), Tauri 2 commands, React + TypeScript + TanStack Query + i18next + Tauri `plugin-dialog`, vitest.

**Repo:** `/Users/fsm/project/MyProject/agentplugin/cc-switch-cloud` (branch `main`, schema v16 → v17).

**ENV GOTCHAS — every Rust run/commit command in this plan is prefixed accordingly:**
- `cargo` lives at `~/.cargo/bin` → prefix every cargo command with `export PATH="$HOME/.cargo/bin:$PATH";`.
- Slow net → also prefix `CARGO_NET_RETRY=10 CARGO_HTTP_LOW_SPEED_LIMIT=0`.
- Keep the gitignored `.cargo/config.toml` (forces `/usr/bin/cc`); do NOT delete it.
- Rust tests touching global HOME state use `#[serial]` + a `TempHome` guard that sets `HOME`/`USERPROFILE`/`CC_SWITCH_TEST_HOME`.
- Run full `cargo test` (NOT `--lib`) and `cargo clippy --all-targets` so the integration-test crate compiles.
- `cargo fmt` before EVERY commit.
- Frontend: run `./node_modules/.bin/vitest run` and `./node_modules/.bin/tsc --noEmit` DIRECTLY from the repo root (NOT `pnpm test:unit`/`pnpm typecheck`).
- RECURRING LESSON: when changing a struct field or a `pub` API, `grep tests/` (the integration-test crate) too — `--lib` can pass while `--all-targets`/full `cargo test` fail to compile.

All Rust commands below assume CWD `/Users/fsm/project/MyProject/agentplugin/cc-switch-cloud/src-tauri`; all frontend commands assume CWD `/Users/fsm/project/MyProject/agentplugin/cc-switch-cloud`.

---

## Task 1 — Schema v17: `projects` table + `apply_manifest.project_id` (base CREATE + migration)

**Files:**
- Modify `src-tauri/src/database/mod.rs` (line 52: `SCHEMA_VERSION`)
- Modify `src-tauri/src/database/schema.rs` (base `create_tables_on_conn` ~line 185 after the `apply_manifest` CREATE; dispatch loop ~line 540-544; new `migrate_v16_to_v17` after `migrate_v15_to_v16` ~line 1436)
- Test: `src-tauri/src/database/tests.rs` (new tests in the existing `#[cfg(test)]`-free module file; tests are top-level `#[test]` fns there)

**Key facts verified:** `Database::memory()` calls `create_tables()` but NOT `apply_schema_migrations()`, so a memory DB stays at `user_version=0` with the latest base schema. Therefore the base CREATE path MUST contain the `projects` table and the `project_id` column, AND the migration must be idempotent (guarded by `table_exists`/`has_column`). `add_column_if_missing(conn, table, col, def) -> Result<bool>` errors if the table is missing, so the migration guards with `table_exists` first.

**DESIGN DECISION (no FK on `apply_manifest.project_id`) — brief §4 alternative, chosen here:** SQLite cannot add a column with an inline `FK` via `ALTER TABLE`, so a migrated DB and a fresh DB could never converge on an FK. Brief §4 explicitly permits this alternative: *"add a plain `project_id TEXT` column and enforce the relationship in the DAO … either is acceptable (channel + project_id together are the key)."* We therefore add a plain `project_id TEXT` column to `apply_manifest` in BOTH the base CREATE TABLE and the migration — with NO `FOREIGN KEY (project_id) REFERENCES projects(id)` clause. Rationale beyond convergence: `Database::memory()` runs `PRAGMA foreign_keys = ON` (database/mod.rs:166), so an FK on `project_id` would make it impossible to insert the *foreign/orphan* manifest rows that Task 9 deliberately creates to PROVE the §3.2 path-reuse protection (a stale `project:<path>` row whose `project_id` points at a deleted project, and a path-gone orphan row). The relationship is enforced in the DAO + the apply-time reconcile guard (which skips rows whose `project_id != current binding`), and the delete-cascade for a deleted project is handled by the apply-time / doctor prune (`prune_orphan_project_channels`, Task 9) plus `delete_manifest_entries`, NOT by SQLite CASCADE. The `profiles` FK on `profile_id` is UNCHANGED.

### Steps

1. - [ ] Write failing test `schema_v17_creates_projects_table_and_manifest_project_id` in `src-tauri/src/database/tests.rs`. Mirror `schema_migration_v12_to_v13_reaches_current` (tests.rs:172). Append:
   ```rust
   #[test]
   fn schema_v17_creates_projects_table_and_manifest_project_id() {
       let conn = Connection::open_in_memory().expect("open memory db");
       Database::create_tables_on_conn(&conn).expect("create tables");
       // fresh base schema must already contain v17 shape
       let n: i64 = conn
           .query_row(
               "SELECT count(*) FROM sqlite_master WHERE type='table' AND name='projects'",
               [],
               |r| r.get(0),
           )
           .expect("count projects");
       assert_eq!(n, 1, "projects table must exist in base create_tables");
       assert!(
           Database::has_column(&conn, "apply_manifest", "project_id").expect("has_column"),
           "apply_manifest.project_id must exist in base create_tables"
       );
   }

   #[test]
   fn schema_migration_v16_to_v17_reaches_current() {
       let conn = Connection::open_in_memory().expect("open memory db");
       Database::create_tables_on_conn(&conn).expect("create tables");
       Database::set_user_version(&conn, 16).expect("seed v16");
       Database::apply_schema_migrations_on_conn(&conn).expect("apply migration");
       assert_eq!(
           Database::get_user_version(&conn).expect("version"),
           SCHEMA_VERSION
       );
       assert_eq!(SCHEMA_VERSION, 17, "SCHEMA_VERSION must be bumped to 17");
       let n: i64 = conn
           .query_row(
               "SELECT count(*) FROM sqlite_master WHERE type='table' AND name='projects'",
               [],
               |r| r.get(0),
           )
           .expect("count");
       assert_eq!(n, 1, "projects table must exist after migration");
       assert!(
           Database::has_column(&conn, "apply_manifest", "project_id").expect("has_column"),
           "apply_manifest.project_id must exist after migration"
       );
   }
   ```

2. - [ ] Run it — expect FAIL (compile error / version mismatch):
   `export PATH="$HOME/.cargo/bin:$PATH"; CARGO_NET_RETRY=10 cargo test --lib schema_migration_v16_to_v17_reaches_current schema_v17_creates_projects_table_and_manifest_project_id 2>&1 | tail -20`
   Expected: `assertion ... left: 16, right: 17` (SCHEMA_VERSION still 16) OR the projects-table count assert fails (`left: 0, right: 1`).

3. - [ ] Bump `SCHEMA_VERSION` in `src-tauri/src/database/mod.rs:52`:
   ```rust
   pub(crate) const SCHEMA_VERSION: i32 = 17;
   ```

4. - [ ] Add the base CREATE TABLE in `create_tables_on_conn` (schema.rs) and alter the existing `apply_manifest` CREATE to include the plain `project_id TEXT` column (NO inline FK — see DESIGN DECISION above). First add `project_id` to the base `apply_manifest` CREATE (the block at schema.rs:171-185) by inserting ONE column line; the existing `profiles` FK is UNCHANGED and no FK is added for `project_id`:
   ```rust
           content_hash TEXT,
           project_id   TEXT,
           FOREIGN KEY (profile_id) REFERENCES profiles(id) ON DELETE CASCADE
   ```
   Then add the new `projects` CREATE TABLE. Because `apply_manifest` no longer references `projects` via FK, ordering is irrelevant; insert the `projects` block immediately AFTER the `apply_manifest` CREATE (the one ending at ~line 185), i.e. just after the `// 6e. Apply Manifest 表` block:
   ```rust
           // 6e0. Projects 表（schema v17）—— 设备本地，绑定真实项目目录到自有内容集
           conn.execute(
               "CREATE TABLE IF NOT EXISTS projects (
           id           TEXT PRIMARY KEY,
           project_path TEXT NOT NULL,
           entered_path TEXT NOT NULL,
           app_type     TEXT NOT NULL,
           name         TEXT,
           spec         TEXT NOT NULL DEFAULT '{}',
           enabled      BOOLEAN NOT NULL DEFAULT 1,
           created_at   INTEGER NOT NULL DEFAULT 0,
           updated_at   INTEGER NOT NULL DEFAULT 0,
           UNIQUE (project_path, app_type)
       )",
               [],
           )
           .map_err(|e| AppError::Database(e.to_string()))?;
   ```

5. - [ ] Add the `migrate_v16_to_v17` fn in schema.rs after `migrate_v15_to_v16` (~line 1436):
   ```rust
   /// v16 -> v17 迁移：新增 projects 表 + apply_manifest.project_id 列（设备本地项目绑定）。
   ///
   /// 注意（与 migrate_v13_to_v14 同理）：`create_tables_on_conn` 已按最新（v17）结构建表，
   /// 全新库会从 0 直接跑到 SCHEMA_VERSION，此时 projects 已存在、project_id 已含，故用
   /// `CREATE TABLE IF NOT EXISTS` + `add_column_if_missing`（幂等）。设计决策：apply_manifest
   /// 的 project_id 为裸 `TEXT` 列、**不带 FK**（base CREATE 与迁移一致，从而新库/旧库收敛；
   /// 且 FK 会阻挡 Task 9 故意写入的“外来/孤儿”行，那些行正是用来证明 §3.2 路径复用保护的）。
   /// 身份关系在 DAO + apply 重算守卫（按 project_id 跳过外来行）+ prune 中强制，
   /// channel + project_id 共同构成身份键。
   fn migrate_v16_to_v17(conn: &Connection) -> Result<(), AppError> {
       conn.execute(
           "CREATE TABLE IF NOT EXISTS projects (
       id           TEXT PRIMARY KEY,
       project_path TEXT NOT NULL,
       entered_path TEXT NOT NULL,
       app_type     TEXT NOT NULL,
       name         TEXT,
       spec         TEXT NOT NULL DEFAULT '{}',
       enabled      BOOLEAN NOT NULL DEFAULT 1,
       created_at   INTEGER NOT NULL DEFAULT 0,
       updated_at   INTEGER NOT NULL DEFAULT 0,
       UNIQUE (project_path, app_type)
   )",
           [],
       )
       .map_err(|e| AppError::Database(e.to_string()))?;

       if Self::table_exists(conn, "apply_manifest")? {
           Self::add_column_if_missing(conn, "apply_manifest", "project_id", "TEXT")?;
       }
       log::info!("v16 -> v17 migration done: projects table + apply_manifest.project_id");
       Ok(())
   }
   ```

6. - [ ] Wire the dispatch arm. In the `match version` loop (schema.rs), change the `15 => { ... set_user_version(conn, 16)? }` arm's neighbours by adding a NEW arm after it (before the `_ =>` catch-all at ~line 545):
   ```rust
                       16 => {
                           log::info!("migrating db v16 -> v17 (projects + apply_manifest.project_id)");
                           Self::migrate_v16_to_v17(conn)?;
                           Self::set_user_version(conn, 17)?;
                       }
   ```

7. - [ ] Run the tests — expect PASS:
   `export PATH="$HOME/.cargo/bin:$PATH"; CARGO_NET_RETRY=10 cargo test --lib schema_migration_v16_to_v17_reaches_current schema_v17_creates_projects_table_and_manifest_project_id 2>&1 | tail -15`
   Expected: `test result: ok. 2 passed`.

8. - [ ] fmt + commit:
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo fmt; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && git add -A && git commit -m "feat(db): schema v17 — projects table + apply_manifest.project_id"`

---

## Task 2 — `Project` / `ProjectSpec` structs in `app_config.rs`

**Files:**
- Modify `src-tauri/src/app_config.rs` (after `ProfileSpec`/`Profile` ~line 291)
- Test: inline `#[cfg(test)]` round-trip in `app_config.rs` (serde shape)

**Key facts verified:** `ProfileContent` (skills/commands/agents/mcp: `Vec<String>`) at app_config.rs:252; `ProfileSpec { content, vars: serde_json::Map }` at 265. `Profile` uses `#[serde(rename_all = "camelCase")]`. We REUSE `ProfileContent` per brief §3.

### Steps

1. - [ ] Write failing test. Add to the existing test module in `app_config.rs` (or create one at file end if none — there is no struct-level test mod for ProfileSpec, so add a new `#[cfg(test)] mod project_struct_tests`):
   ```rust
   #[cfg(test)]
   mod project_struct_tests {
       use super::{Project, ProjectSpec};

       #[test]
       fn project_spec_serde_camelcase_roundtrip() {
           let p = Project {
               id: "proj:1".into(),
               project_path: "/abs/canon/repo".into(),
               entered_path: "~/repo".into(),
               app_type: "claude".into(),
               name: Some("Repo".into()),
               spec: ProjectSpec::default(),
               enabled: true,
               created_at: 10,
               updated_at: 20,
           };
           let json = serde_json::to_string(&p).expect("serialize");
           // camelCase keys for the cross-FFI fields
           assert!(json.contains("\"projectPath\":\"/abs/canon/repo\""), "json={json}");
           assert!(json.contains("\"enteredPath\":\"~/repo\""), "json={json}");
           assert!(json.contains("\"appType\":\"claude\""), "json={json}");
           let back: Project = serde_json::from_str(&json).expect("deserialize");
           assert_eq!(back.id, "proj:1");
           assert!(back.enabled);
           assert!(back.spec.content.skills.is_empty());
       }
   }
   ```

2. - [ ] Run it — expect FAIL (compile error: `Project`/`ProjectSpec` unresolved):
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo test --lib project_spec_serde_camelcase_roundtrip 2>&1 | tail -15`
   Expected: `cannot find type \`Project\` in this scope` / `ProjectSpec`.

3. - [ ] Add the structs in `app_config.rs` immediately after the `Profile` struct (after line 291):
   ```rust
   /// Project spec: own content set (same JSON shape as ProfileSpec) + reserved vars for 4b.
   #[derive(Debug, Clone, Default, Serialize, Deserialize)]
   pub struct ProjectSpec {
       #[serde(default)]
       pub content: ProfileContent, // REUSE existing struct (skills/commands/agents/mcp)
       #[serde(default)]
       pub vars: serde_json::Map<String, serde_json::Value>, // reserved for 4b dotfiles/${VAR}
   }

   /// A device-local binding of a real project directory to an own content set.
   #[derive(Debug, Clone, Serialize, Deserialize)]
   #[serde(rename_all = "camelCase")]
   pub struct Project {
       pub id: String,
       /// Canonical absolute directory path — identity + manifest channel key.
       pub project_path: String,
       /// User-entered path — display only.
       pub entered_path: String,
       pub app_type: String,
       #[serde(default, skip_serializing_if = "Option::is_none")]
       pub name: Option<String>,
       #[serde(default)]
       pub spec: ProjectSpec,
       #[serde(default)]
       pub enabled: bool,
       #[serde(default)]
       pub created_at: i64,
       #[serde(default)]
       pub updated_at: i64,
   }
   ```

4. - [ ] Run it — expect PASS:
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo test --lib project_spec_serde_camelcase_roundtrip 2>&1 | tail -10`
   Expected: `test result: ok. 1 passed`.

5. - [ ] fmt + commit:
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo fmt; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && git add -A && git commit -m "feat(config): Project / ProjectSpec structs (reuse ProfileContent)"`

---

## Task 3 — `projects` DAO CRUD

**Files:**
- Create `src-tauri/src/database/dao/projects.rs`
- Modify `src-tauri/src/database/dao/mod.rs` (add `pub mod projects;` after the `pub mod profiles;` line ~11)
- Test: inline `#[cfg(test)]` in `projects.rs`

**Key facts verified:** Mirror `profiles.rs` CRUD shape: `get_all`/`get_for_app`/`get`/`save`(INSERT OR REPLACE)/`delete`. `projects` has NO single-active-row invariant (unlike `set_active_profile`), so we do NOT add a `set_active` method. `spec` serializes via `serde_json::to_string(&p.spec)`. `enabled` is `bool` → rusqlite maps to BOOLEAN. We also add `get_all_project_channels()` (for the path-gone prune in Task 9) and `prune_orphan_project_manifests()` deferred to Task 9 (kept in the apply service module). Here: pure CRUD + a `get_project_by_path_and_app` for UNIQUE lookups.

### Steps

1. - [ ] Add the module declaration. In `src-tauri/src/database/dao/mod.rs`, after `pub mod profiles;` (line 11) add:
   ```rust
   pub mod projects;
   ```

2. - [ ] Write the failing test as the body of the new file (write the whole file with tests first; the `impl Database` block is added in step 4). Create `src-tauri/src/database/dao/projects.rs` with ONLY the test module + imports so it fails to compile (methods not yet defined):
   ```rust
   //! Projects 数据访问对象（schema v17，设备本地）。

   use crate::app_config::Project;
   use crate::database::{lock_conn, Database};
   use crate::error::AppError;
   use rusqlite::params;

   #[cfg(test)]
   mod tests {
       use super::*;
       use crate::app_config::{ProfileContent, ProjectSpec};
       use crate::database::Database;

       fn make_project(id: &str, canon: &str, app: &str) -> Project {
           Project {
               id: id.into(),
               project_path: canon.into(),
               entered_path: format!("entered:{canon}"),
               app_type: app.into(),
               name: Some(format!("name:{id}")),
               spec: ProjectSpec {
                   content: ProfileContent {
                       skills: vec!["skill-a".into()],
                       commands: vec!["cmd-foo".into()],
                       agents: vec!["agent-x".into()],
                       mcp: vec![],
                   },
                   vars: serde_json::Map::new(),
               },
               enabled: true,
               created_at: 100,
               updated_at: 200,
           }
       }

       #[test]
       fn project_dao_crud_roundtrip() -> Result<(), AppError> {
           let db = Database::memory()?;
           let p = make_project("proj:1", "/abs/repo", "claude");
           db.save_project(&p)?;

           let got = db.get_project("proj:1")?.expect("exists");
           assert_eq!(got.project_path, "/abs/repo");
           assert_eq!(got.entered_path, "entered:/abs/repo");
           assert_eq!(got.spec.content.skills, vec!["skill-a"]);
           assert!(got.enabled);

           assert_eq!(db.get_all_projects()?.len(), 1);
           assert_eq!(db.get_projects_for_app("claude")?.len(), 1);
           assert_eq!(db.get_projects_for_app("codex")?.len(), 0);

           let by_path = db
               .get_project_by_path_and_app("/abs/repo", "claude")?
               .expect("found by path");
           assert_eq!(by_path.id, "proj:1");

           // update via save (INSERT OR REPLACE on PK id)
           let mut p2 = got.clone();
           p2.enabled = false;
           p2.name = Some("renamed".into());
           db.save_project(&p2)?;
           let got2 = db.get_project("proj:1")?.expect("exists");
           assert!(!got2.enabled);
           assert_eq!(got2.name.as_deref(), Some("renamed"));

           assert!(db.delete_project("proj:1")?);
           assert!(db.get_project("proj:1")?.is_none());
           Ok(())
       }

       #[test]
       fn project_unique_path_app_enforced() -> Result<(), AppError> {
           let db = Database::memory()?;
           db.save_project(&make_project("proj:1", "/abs/repo", "claude"))?;
           // same canonical path + app under a DIFFERENT id violates UNIQUE(project_path, app_type)
           let dup = make_project("proj:2", "/abs/repo", "claude");
           let err = db.save_project(&dup);
           assert!(err.is_err(), "duplicate (path, app) must be rejected");
           Ok(())
       }
   }
   ```

3. - [ ] Run it — expect FAIL (compile error: methods undefined):
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo test --lib project_dao_crud_roundtrip 2>&1 | tail -20`
   Expected: `no method named \`save_project\` found for ... Database`.

4. - [ ] Add the `impl Database` block to `projects.rs` BETWEEN the imports and the `#[cfg(test)]` module:
   ```rust
   impl Database {
       /// 获取所有 Projects（设备本地）
       pub fn get_all_projects(&self) -> Result<Vec<Project>, AppError> {
           let conn = lock_conn!(self.conn);
           let mut stmt = conn
               .prepare(
                   "SELECT id, project_path, entered_path, app_type, name, spec, enabled,
                           created_at, updated_at
                    FROM projects ORDER BY created_at ASC, id ASC",
               )
               .map_err(|e| AppError::Database(e.to_string()))?;
           let rows = stmt
               .query_map([], Self::map_project_row)
               .map_err(|e| AppError::Database(e.to_string()))?;
           let mut out = Vec::new();
           for r in rows {
               out.push(r.map_err(|e| AppError::Database(e.to_string()))?);
           }
           Ok(out)
       }

       /// 获取某 app_type 的所有 Projects
       pub fn get_projects_for_app(&self, app_type: &str) -> Result<Vec<Project>, AppError> {
           let conn = lock_conn!(self.conn);
           let mut stmt = conn
               .prepare(
                   "SELECT id, project_path, entered_path, app_type, name, spec, enabled,
                           created_at, updated_at
                    FROM projects WHERE app_type = ?1 ORDER BY created_at ASC, id ASC",
               )
               .map_err(|e| AppError::Database(e.to_string()))?;
           let rows = stmt
               .query_map([app_type], Self::map_project_row)
               .map_err(|e| AppError::Database(e.to_string()))?;
           let mut out = Vec::new();
           for r in rows {
               out.push(r.map_err(|e| AppError::Database(e.to_string()))?);
           }
           Ok(out)
       }

       /// 获取单个 Project（按 id）
       pub fn get_project(&self, id: &str) -> Result<Option<Project>, AppError> {
           let conn = lock_conn!(self.conn);
           let mut stmt = conn
               .prepare(
                   "SELECT id, project_path, entered_path, app_type, name, spec, enabled,
                           created_at, updated_at
                    FROM projects WHERE id = ?1",
               )
               .map_err(|e| AppError::Database(e.to_string()))?;
           match stmt.query_row([id], Self::map_project_row) {
               Ok(p) => Ok(Some(p)),
               Err(rusqlite::Error::QueryReturnedNoRows) => Ok(None),
               Err(e) => Err(AppError::Database(e.to_string())),
           }
       }

       /// 按规范路径 + app 查找 Project（UNIQUE 键查询，用于绑定冲突检测）
       pub fn get_project_by_path_and_app(
           &self,
           project_path: &str,
           app_type: &str,
       ) -> Result<Option<Project>, AppError> {
           let conn = lock_conn!(self.conn);
           let mut stmt = conn
               .prepare(
                   "SELECT id, project_path, entered_path, app_type, name, spec, enabled,
                           created_at, updated_at
                    FROM projects WHERE project_path = ?1 AND app_type = ?2",
               )
               .map_err(|e| AppError::Database(e.to_string()))?;
           match stmt.query_row(params![project_path, app_type], Self::map_project_row) {
               Ok(p) => Ok(Some(p)),
               Err(rusqlite::Error::QueryReturnedNoRows) => Ok(None),
               Err(e) => Err(AppError::Database(e.to_string())),
           }
       }

       /// 保存 Project（INSERT OR REPLACE on PK id；UNIQUE(project_path, app_type) 仍会拒绝冲突）
       pub fn save_project(&self, p: &Project) -> Result<(), AppError> {
           let spec_json = serde_json::to_string(&p.spec)
               .map_err(|e| AppError::Config(format!("project spec serialize failed: {e}")))?;
           let conn = lock_conn!(self.conn);
           conn.execute(
               "INSERT OR REPLACE INTO projects
                (id, project_path, entered_path, app_type, name, spec, enabled,
                 created_at, updated_at)
                VALUES (?1, ?2, ?3, ?4, ?5, ?6, ?7, ?8, ?9)",
               params![
                   p.id,
                   p.project_path,
                   p.entered_path,
                   p.app_type,
                   p.name,
                   spec_json,
                   p.enabled,
                   p.created_at,
                   p.updated_at,
               ],
           )
           .map_err(|e| AppError::Database(e.to_string()))?;
           Ok(())
       }

       /// 删除 Project；返回是否找到行。注意：project_id 列无 FK CASCADE（设计决策，见 Task 1），
       /// 该项目残留的 manifest 行由 detach / `prune_orphan_project_channels`（Task 9）清理。
       pub fn delete_project(&self, id: &str) -> Result<bool, AppError> {
           let conn = lock_conn!(self.conn);
           let affected = conn
               .execute("DELETE FROM projects WHERE id = ?1", params![id])
               .map_err(|e| AppError::Database(e.to_string()))?;
           Ok(affected > 0)
       }

       /// 行映射助手（projects 表 → Project）。
       fn map_project_row(row: &rusqlite::Row<'_>) -> rusqlite::Result<Project> {
           let spec_raw: String = row.get(5)?;
           let spec = serde_json::from_str(&spec_raw).unwrap_or_default();
           Ok(Project {
               id: row.get(0)?,
               project_path: row.get(1)?,
               entered_path: row.get(2)?,
               app_type: row.get(3)?,
               name: row.get(4)?,
               spec,
               enabled: row.get(6)?,
               created_at: row.get(7)?,
               updated_at: row.get(8)?,
           })
       }
   }
   ```

5. - [ ] Run it — expect PASS:
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo test --lib project_dao_crud_roundtrip project_unique_path_app_enforced 2>&1 | tail -10`
   Expected: `test result: ok. 2 passed`.

6. - [ ] fmt + commit:
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo fmt; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && git add -A && git commit -m "feat(db): projects DAO CRUD"`

---

## Task 4a — `ManifestEntry.project_id` field + DAO write/read carry it

**Files:**
- Modify `src-tauri/src/app_config.rs` (`ManifestEntry` ~line 305)
- Modify `src-tauri/src/database/dao/manifest.rs` (`record_manifest_entry` :24-42; `get_manifest_for_profile` :45-91)
- Test: inline `#[cfg(test)]` in `manifest.rs`

**Key facts verified:** `ManifestEntry` (app_config.rs:305) has `id, channel, profile_id: Option<String>, app_type, target_path, kind, content_hash: Option<String>, created_at`, `#[serde(rename_all="camelCase")]`. `record_manifest_entry` INSERTs columns `(channel, profile_id, app_type, target_path, kind, content_hash, created_at)`. We add `project_id: Option<String>`. RECURRING LESSON: adding a `ManifestEntry` field breaks `make_entry` in manifest.rs tests AND any `tests/` usage — `grep -rn "ManifestEntry {" src-tauri/src tests` first and fix every literal.

### Steps

1. - [ ] `grep -rn "ManifestEntry {" /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud/src-tauri/src /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud/src-tauri/tests` to enumerate every struct literal (known: `profile_render.rs:143`, `manifest.rs` test `make_entry:151`, `profile.rs` tests). Each must gain `project_id`.

2. - [ ] Write failing test in `manifest.rs` test module — add a `project_id`-carrying assertion to a new test:
   ```rust
   #[test]
   fn manifest_entry_carries_project_id() -> Result<(), AppError> {
       let db = Database::memory()?;
       // NOTE: project_id has NO FK (design decision, Task 1), so a manifest row
       // may carry any project_id string without a matching projects row.
       let e = ManifestEntry {
           id: 0,
           channel: "project:/abs/repo".into(),
           profile_id: None,
           project_id: Some("proj:m".into()),
           app_type: "claude".into(),
           target_path: "/abs/repo/.claude/commands/foo.md".into(),
           kind: "command".into(),
           content_hash: Some("h".into()),
           created_at: 0,
       };
       let id = db.record_manifest_entry(&e)?;
       assert!(id > 0);
       // read back via the project-channel getter (added in Task 4b) — for now read raw:
       let conn = crate::database::lock_conn!(db.conn);
       let pid: Option<String> = conn
           .query_row(
               "SELECT project_id FROM apply_manifest WHERE id = ?1",
               [id],
               |r| r.get(0),
           )
           .expect("query project_id");
       assert_eq!(pid.as_deref(), Some("proj:m"));
       Ok(())
   }
   ```
   Also update `make_entry` (manifest.rs:150) to add `project_id: None,`.

3. - [ ] Run it — expect FAIL (compile error: no field `project_id`):
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo test --lib manifest_entry_carries_project_id 2>&1 | tail -20`
   Expected: `struct \`ManifestEntry\` has no field named \`project_id\``.

4. - [ ] Add the field to `ManifestEntry` (app_config.rs:305), after `profile_id`:
   ```rust
       pub profile_id: Option<String>,
       /// Project binding identity (4a). NULL for global-channel rows.
       #[serde(default, skip_serializing_if = "Option::is_none")]
       pub project_id: Option<String>,
   ```

5. - [ ] Update `record_manifest_entry` (manifest.rs:24) to write `project_id`:
   ```rust
   pub fn record_manifest_entry(&self, e: &ManifestEntry) -> Result<i64, AppError> {
       let conn = lock_conn!(self.conn);
       conn.execute(
           "INSERT INTO apply_manifest
            (channel, profile_id, project_id, app_type, target_path, kind, content_hash, created_at)
            VALUES (?1, ?2, ?3, ?4, ?5, ?6, ?7, ?8)",
           params![
               e.channel,
               e.profile_id,
               e.project_id,
               e.app_type,
               e.target_path,
               e.kind,
               e.content_hash,
               e.created_at,
           ],
       )
       .map_err(|e| AppError::Database(e.to_string()))?;
       Ok(conn.last_insert_rowid())
   }
   ```

6. - [ ] Update `get_manifest_for_profile` (manifest.rs:45) to SELECT + map `project_id`. Change the SELECT column list and the tuple/row construction:
   ```rust
       let mut stmt = conn
           .prepare(
               "SELECT id, channel, profile_id, project_id, app_type, target_path, kind, content_hash, created_at
                FROM apply_manifest
                WHERE profile_id = ?1 AND app_type = ?2
                ORDER BY id ASC",
           )
           .map_err(|e| AppError::Database(e.to_string()))?;

       let rows = stmt
           .query_map([profile_id, app_type], |row| {
               Ok((
                   row.get::<_, i64>(0)?,
                   row.get::<_, String>(1)?,
                   row.get::<_, Option<String>>(2)?,
                   row.get::<_, Option<String>>(3)?,
                   row.get::<_, String>(4)?,
                   row.get::<_, String>(5)?,
                   row.get::<_, String>(6)?,
                   row.get::<_, Option<String>>(7)?,
                   row.get::<_, i64>(8)?,
               ))
           })
           .map_err(|e| AppError::Database(e.to_string()))?;

       let mut entries = Vec::new();
       for row_res in rows {
           let (id, channel, p_id, proj_id, a_type, target_path, kind, content_hash, created_at) =
               row_res.map_err(|e| AppError::Database(e.to_string()))?;
           entries.push(ManifestEntry {
               id,
               channel,
               profile_id: p_id,
               project_id: proj_id,
               app_type: a_type,
               target_path,
               kind,
               content_hash,
               created_at,
           });
       }
       Ok(entries)
   ```

7. - [ ] Fix `ManifestEntry { ... }` in `profile_render.rs:143` (add `project_id: None,` after `profile_id`):
   ```rust
       Ok(Some(ManifestEntry {
           id: 0,
           channel: "global".to_string(),
           profile_id: Some(profile_id.to_string()),
           project_id: None,
           app_type: app_type.as_str().to_string(),
           target_path: path.to_string_lossy().to_string(),
           kind: "whole_file".to_string(),
           content_hash: Some(new_hash),
           created_at: 0,
       }))
   ```
   Then fix any other literals found in step 1 the same way (`project_id: None,`).

8. - [ ] Run full lib tests for manifest + profile_render to confirm no regression:
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo test --lib manifest_ manifest_entry_carries_project_id profile_render 2>&1 | tail -15`
   Expected: `test result: ok` (manifest_crud_roundtrip + manifest_entry_carries_project_id + render tests all pass).

9. - [ ] Verify the integration-test crate still compiles (RECURRING LESSON):
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo test --no-run --all-targets 2>&1 | tail -15`
   Expected: `Finished` with no `has no field named` errors. If a `tests/*.rs` literal broke, add `project_id: None,` there.

10. - [ ] fmt + commit:
    `export PATH="$HOME/.cargo/bin:$PATH"; cargo fmt; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && git add -A && git commit -m "feat(db): ManifestEntry.project_id + carry through record/get"`

---

## Task 4b — Channel-scoped manifest DAO: `get_manifest_for_channel` / `clear_manifest_for_channel` / `get_all_project_channels`

**Files:**
- Modify `src-tauri/src/database/dao/manifest.rs` (add three methods after `clear_manifest_for_profile` :126)
- Test: inline `#[cfg(test)]` in `manifest.rs`

**Key facts verified:** Existing `get_manifest_for_profile`/`clear_manifest_for_profile` filter by `(profile_id, app_type)` ONLY — never by channel. The project flow keys on the channel string `project:<canonpath>`. These new methods filter by `channel` so global rows (channel `global`) and project rows never cross-contaminate.

### Steps

1. - [ ] Write failing test in `manifest.rs` test module:
   ```rust
   #[test]
   fn manifest_channel_scoped_queries() -> Result<(), AppError> {
       let db = Database::memory()?;
       // project_id has NO FK (Task 1 design decision) — no projects row needed.
       let chan = "project:/abs/repo";

       // one global row + two project rows on the same channel
       let g = ManifestEntry {
           id: 0, channel: "global".into(), profile_id: None, project_id: None,
           app_type: "claude".into(), target_path: "/g".into(), kind: "command".into(),
           content_hash: None, created_at: 0,
       };
       db.record_manifest_entry(&g)?;
       for tp in ["/abs/repo/.claude/commands/a.md", "/abs/repo/.claude/agents/b.md"] {
           db.record_manifest_entry(&ManifestEntry {
               id: 0, channel: chan.into(), profile_id: None, project_id: Some("proj:c".into()),
               app_type: "claude".into(), target_path: tp.into(), kind: "command".into(),
               content_hash: Some("h".into()), created_at: 0,
           })?;
       }

       let chan_rows = db.get_manifest_for_channel(chan)?;
       assert_eq!(chan_rows.len(), 2, "only the 2 project-channel rows");
       assert!(chan_rows.iter().all(|r| r.channel == chan));
       assert!(chan_rows.iter().all(|r| r.project_id.as_deref() == Some("proj:c")));

       let channels = db.get_all_project_channels()?;
       assert!(channels.contains(&chan.to_string()));
       assert!(!channels.contains(&"global".to_string()), "global is not a project channel");

       db.clear_manifest_for_channel(chan)?;
       assert_eq!(db.get_manifest_for_channel(chan)?.len(), 0);
       // global row untouched
       let conn = crate::database::lock_conn!(db.conn);
       let n: i64 = conn
           .query_row("SELECT count(*) FROM apply_manifest WHERE channel='global'", [], |r| r.get(0))
           .expect("count global");
       assert_eq!(n, 1, "global rows must survive channel clear");
       Ok(())
   }
   ```

2. - [ ] Run it — expect FAIL (compile error: methods undefined):
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo test --lib manifest_channel_scoped_queries 2>&1 | tail -15`
   Expected: `no method named \`get_manifest_for_channel\``.

3. - [ ] Add the three methods to `manifest.rs` after `clear_manifest_for_profile` (line 126):
   ```rust
       /// 获取某 channel 的全部 manifest 记录（项目通道 = "project:<canonpath>"）。
       /// 与 get_manifest_for_profile 不同：仅按 channel 过滤，确保全局与项目互不串扰。
       pub fn get_manifest_for_channel(
           &self,
           channel: &str,
       ) -> Result<Vec<ManifestEntry>, AppError> {
           let conn = lock_conn!(self.conn);
           let mut stmt = conn
               .prepare(
                   "SELECT id, channel, profile_id, project_id, app_type, target_path, kind, content_hash, created_at
                    FROM apply_manifest
                    WHERE channel = ?1
                    ORDER BY id ASC",
               )
               .map_err(|e| AppError::Database(e.to_string()))?;
           let rows = stmt
               .query_map([channel], |row| {
                   Ok(ManifestEntry {
                       id: row.get(0)?,
                       channel: row.get(1)?,
                       profile_id: row.get(2)?,
                       project_id: row.get(3)?,
                       app_type: row.get(4)?,
                       target_path: row.get(5)?,
                       kind: row.get(6)?,
                       content_hash: row.get(7)?,
                       created_at: row.get(8)?,
                   })
               })
               .map_err(|e| AppError::Database(e.to_string()))?;
           let mut out = Vec::new();
           for r in rows {
               out.push(r.map_err(|e| AppError::Database(e.to_string()))?);
           }
           Ok(out)
       }

       /// 清除某 channel 的全部 manifest 记录（仅该 channel；全局 / 其它项目不受影响）。
       pub fn clear_manifest_for_channel(&self, channel: &str) -> Result<(), AppError> {
           let conn = lock_conn!(self.conn);
           conn.execute(
               "DELETE FROM apply_manifest WHERE channel = ?1",
               params![channel],
           )
           .map_err(|e| AppError::Database(e.to_string()))?;
           Ok(())
       }

       /// 列出所有项目通道（channel LIKE 'project:%'）的去重列表，供路径失效清理使用。
       pub fn get_all_project_channels(&self) -> Result<Vec<String>, AppError> {
           let conn = lock_conn!(self.conn);
           let mut stmt = conn
               .prepare(
                   "SELECT DISTINCT channel FROM apply_manifest WHERE channel LIKE 'project:%'",
               )
               .map_err(|e| AppError::Database(e.to_string()))?;
           let rows = stmt
               .query_map([], |row| row.get::<_, String>(0))
               .map_err(|e| AppError::Database(e.to_string()))?;
           let mut out = Vec::new();
           for r in rows {
               out.push(r.map_err(|e| AppError::Database(e.to_string()))?);
           }
           Ok(out)
       }
   ```

4. - [ ] Run it — expect PASS:
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo test --lib manifest_channel_scoped_queries 2>&1 | tail -10`
   Expected: `test result: ok. 1 passed`.

5. - [ ] fmt + commit:
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo fmt; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && git add -A && git commit -m "feat(db): channel-scoped manifest DAO (get/clear_for_channel, project channels)"`

---

## Task 5a — `AppType::project_dotdir` / `project_memory_filename`

**Files:**
- Modify `src-tauri/src/app_config.rs` (in `impl AppType`, after `as_str` :471 / before `is_additive_mode` :487)
- Test: inline `#[cfg(test)]` in `app_config.rs`

**Key facts verified:** `AppType` (app_config.rs:455) has Claude/ClaudeDesktop/Codex/Gemini/OpenCode/OpenClaw/Hermes. Per brief §6: claude→`.claude`/`CLAUDE.md`, codex→`.codex`/`AGENTS.md`, opencode→`.opencode`/`AGENTS.md`, gemini→`.gemini`/`GEMINI.md`. 4a only USES Claude, but we define all so 4c is clean. `project_memory_filename` is reserved for 4b (not yet used) but defined now.

### Steps

1. - [ ] Write failing test (new mod in app_config.rs):
   ```rust
   #[cfg(test)]
   mod app_type_project_dirs {
       use super::AppType;
       #[test]
       fn project_dotdir_and_memory_filename_per_app() {
           assert_eq!(AppType::Claude.project_dotdir(), ".claude");
           assert_eq!(AppType::Codex.project_dotdir(), ".codex");
           assert_eq!(AppType::OpenCode.project_dotdir(), ".opencode");
           assert_eq!(AppType::Gemini.project_dotdir(), ".gemini");
           assert_eq!(AppType::Claude.project_memory_filename(), "CLAUDE.md");
           assert_eq!(AppType::Codex.project_memory_filename(), "AGENTS.md");
           assert_eq!(AppType::OpenCode.project_memory_filename(), "AGENTS.md");
           assert_eq!(AppType::Gemini.project_memory_filename(), "GEMINI.md");
       }
   }
   ```

2. - [ ] Run it — expect FAIL: `no method named \`project_dotdir\``.
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo test --lib project_dotdir_and_memory_filename_per_app 2>&1 | tail -12`

3. - [ ] Add the methods inside `impl AppType` after `as_str` (line 481):
   ```rust
       /// Per-app project dotdir name (4a uses Claude only; others defined for 4c).
       pub fn project_dotdir(&self) -> &str {
           match self {
               AppType::Claude | AppType::ClaudeDesktop => ".claude",
               AppType::Codex => ".codex",
               AppType::Gemini => ".gemini",
               AppType::OpenCode => ".opencode",
               AppType::OpenClaw => ".openclaw",
               AppType::Hermes => ".hermes",
           }
       }

       /// Per-app project memory filename (reserved for 4b project dotfiles).
       pub fn project_memory_filename(&self) -> &str {
           match self {
               AppType::Claude | AppType::ClaudeDesktop => "CLAUDE.md",
               AppType::Codex | AppType::OpenCode | AppType::OpenClaw => "AGENTS.md",
               AppType::Gemini => "GEMINI.md",
               AppType::Hermes => "HERMES.md",
           }
       }
   ```

4. - [ ] Run it — expect PASS:
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo test --lib project_dotdir_and_memory_filename_per_app 2>&1 | tail -10`
   Expected: `test result: ok. 1 passed`.

5. - [ ] fmt + commit:
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo fmt; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && git add -A && git commit -m "feat(config): AppType::project_dotdir / project_memory_filename"`

---

## Task 5b — `services/project_paths.rs`: path-safety perimeter gate (CRITICAL #1)

**Files:**
- Create `src-tauri/src/services/project_paths.rs`
- Modify `src-tauri/src/services/mod.rs` (add `pub mod project_paths;`)
- Test: inline `#[cfg(test)]` in `project_paths.rs` (uses `serial_test` + `TempHome` guard)

**Key facts verified:** `crate::config::get_home_dir()` (config.rs:22) honors `CC_SWITCH_TEST_HOME`. `get_app_config_dir()` (config.rs:100) → `~/.agenthub`. `get_claude_config_dir()` (config.rs:37) → `~/.claude` (override-aware). The TempHome guard pattern (sets `HOME`/`USERPROFILE`/`CC_SWITCH_TEST_HOME`, restores on Drop) is at command.rs/profile.rs tests. This gate is the heart of CRITICAL #1: it must REFUSE binding/applying if the canonical project root equals/ancestors/descends `$HOME`, any tool dir (`~/.claude`, `~/.codex`, `~/.config/opencode`, `~/.gemini`), `~/.agenthub`, or `/`; AND if `<root>/.claude` canonicalizes to `~/.claude`.

### Steps

1. - [ ] Add the module declaration to `src-tauri/src/services/mod.rs` (after `pub mod profile_render;`):
   ```rust
   pub mod project_paths;
   ```

2. - [ ] Write the failing test file FIRST. Create `src-tauri/src/services/project_paths.rs` with imports + the `ProjectBase` doc-comment placeholder and the full test module, but NO impl yet:
   ```rust
   //! Project path-safety perimeter (increment 4a, CRITICAL #1).
   //!
   //! Before ANY write and on EVERY apply we canonicalize the project root and
   //! REFUSE to operate if the canonical path equals/ancestors/descends $HOME, any
   //! tool config dir (~/.claude, ~/.codex, ~/.config/opencode, ~/.gemini),
   //! ~/.agenthub, or "/", OR if <root>/.claude canonicalizes to ~/.claude. This
   //! makes the <project>/.claude == ~/.claude HOME-collapse impossible, which
   //! would otherwise let a global deactivate owned-delete project files (and vice
   //! versa) because get_manifest_for_profile filters by (profile_id, app_type)
   //! only, not by channel.

   use std::path::{Path, PathBuf};

   use crate::app_config::AppType;
   use crate::config::{get_app_config_dir, get_home_dir};
   use crate::error::AppError;

   #[cfg(test)]
   mod tests {
       use super::*;
       use serial_test::serial;
       use std::env;
       use tempfile::TempDir;

       struct TempHome {
           dir: TempDir,
           original_home: Option<String>,
           original_userprofile: Option<String>,
           original_test_home: Option<String>,
       }
       impl TempHome {
           fn new() -> Self {
               let dir = TempDir::new().expect("tempdir");
               let original_home = env::var("HOME").ok();
               let original_userprofile = env::var("USERPROFILE").ok();
               let original_test_home = env::var("CC_SWITCH_TEST_HOME").ok();
               env::set_var("HOME", dir.path());
               env::set_var("USERPROFILE", dir.path());
               env::set_var("CC_SWITCH_TEST_HOME", dir.path());
               Self { dir, original_home, original_userprofile, original_test_home }
           }
           fn home(&self) -> &Path { self.dir.path() }
       }
       impl Drop for TempHome {
           fn drop(&mut self) {
               match &self.original_home { Some(v) => env::set_var("HOME", v), None => env::remove_var("HOME") }
               match &self.original_userprofile { Some(v) => env::set_var("USERPROFILE", v), None => env::remove_var("USERPROFILE") }
               match &self.original_test_home { Some(v) => env::set_var("CC_SWITCH_TEST_HOME", v), None => env::remove_var("CC_SWITCH_TEST_HOME") }
           }
       }

       #[test]
       #[serial]
       fn accepts_normal_project_dir() {
           let home = TempHome::new();
           let proj = home.home().join("work").join("repo");
           std::fs::create_dir_all(&proj).expect("mkdir proj");
           let base = ProjectBase::resolve(proj.to_str().unwrap(), &AppType::Claude)
               .expect("a normal project dir must be accepted");
           assert_eq!(base.dotdir(), base.root().join(".claude"));
       }

       #[test]
       #[serial]
       fn refuses_home_itself() {
           let home = TempHome::new();
           let err = ProjectBase::resolve(home.home().to_str().unwrap(), &AppType::Claude)
               .expect_err("binding $HOME must be refused (collapse)");
           assert!(err.to_string().contains("HOME") || err.to_string().contains("拒绝"), "err={err}");
       }

       #[test]
       #[serial]
       fn refuses_claude_config_dir() {
           let home = TempHome::new();
           let claude = home.home().join(".claude");
           std::fs::create_dir_all(&claude).expect("mkdir .claude");
           assert!(ProjectBase::resolve(claude.to_str().unwrap(), &AppType::Claude).is_err());
       }

       #[test]
       #[serial]
       fn refuses_tool_dirs() {
           let home = TempHome::new();
           for d in [".codex", ".gemini"] {
               let p = home.home().join(d);
               std::fs::create_dir_all(&p).expect("mkdir tool dir");
               assert!(ProjectBase::resolve(p.to_str().unwrap(), &AppType::Claude).is_err(), "{d} must be refused");
           }
           let oc = home.home().join(".config").join("opencode");
           std::fs::create_dir_all(&oc).expect("mkdir opencode");
           assert!(ProjectBase::resolve(oc.to_str().unwrap(), &AppType::Claude).is_err(), ".config/opencode must be refused");
       }

       #[test]
       #[serial]
       fn refuses_app_config_dir() {
           let home = TempHome::new();
           let ah = home.home().join(".agenthub");
           std::fs::create_dir_all(&ah).expect("mkdir .agenthub");
           assert!(ProjectBase::resolve(ah.to_str().unwrap(), &AppType::Claude).is_err());
       }

       #[test]
       #[serial]
       fn refuses_root() {
           let _home = TempHome::new();
           assert!(ProjectBase::resolve("/", &AppType::Claude).is_err());
       }

       #[test]
       #[serial]
       fn refuses_ancestor_of_home() {
           let home = TempHome::new();
           // parent of $HOME is an ancestor → refuse
           let parent = home.home().parent().expect("home has parent").to_path_buf();
           assert!(ProjectBase::resolve(parent.to_str().unwrap(), &AppType::Claude).is_err());
       }

       #[test]
       #[serial]
       fn refuses_descendant_of_claude_dir() {
           let home = TempHome::new();
           let inside = home.home().join(".claude").join("skills");
           std::fs::create_dir_all(&inside).expect("mkdir inside .claude");
           assert!(ProjectBase::resolve(inside.to_str().unwrap(), &AppType::Claude).is_err());
       }

       #[test]
       #[serial]
       fn refuses_when_dotdir_symlinks_to_claude() {
           let home = TempHome::new();
           let proj = home.home().join("work").join("trick");
           std::fs::create_dir_all(&proj).expect("mkdir proj");
           let real_claude = home.home().join(".claude");
           std::fs::create_dir_all(&real_claude).expect("mkdir .claude");
           // <proj>/.claude -> ~/.claude : the dotdir collapse the gate must catch
           #[cfg(unix)]
           std::os::unix::fs::symlink(&real_claude, proj.join(".claude")).expect("symlink");
           #[cfg(unix)]
           assert!(ProjectBase::resolve(proj.to_str().unwrap(), &AppType::Claude).is_err());
       }
   }
   ```

3. - [ ] Run it — expect FAIL (compile error: `ProjectBase` undefined):
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo test --lib project_paths:: 2>&1 | tail -20`
   Expected: `cannot find ... \`ProjectBase\``.

4. - [ ] Add the `ProjectBase` value object + gate impl ABOVE the test module:
   ```rust
   /// A validated project base: the canonical project root + its app dotdir.
   /// Construct ONLY via `resolve`, which runs the path-safety perimeter gate.
   #[derive(Debug, Clone)]
   pub struct ProjectBase {
       root: PathBuf,   // canonical absolute project root
       dotdir: PathBuf, // <root>/<app.project_dotdir()>
   }

   impl ProjectBase {
       pub fn root(&self) -> &Path {
           &self.root
       }
       pub fn dotdir(&self) -> &Path {
           &self.dotdir
       }
       /// Canonical channel key for the manifest: "project:<canonical-abspath>".
       pub fn channel(&self) -> String {
           format!("project:{}", self.root.to_string_lossy())
       }

       /// Resolve + validate a user-entered project root for `app`.
       ///
       /// Returns Err (refusing all writes) when the canonical root equals,
       /// is an ancestor of, or is a descendant of any forbidden anchor, or when
       /// <root>/.claude collapses onto ~/.claude.
       pub fn resolve(entered: &str, app: &AppType) -> Result<ProjectBase, AppError> {
           let trimmed = entered.trim();
           if trimmed.is_empty() {
               return Err(AppError::InvalidInput("project root 为空".into()));
           }
           let raw = PathBuf::from(trimmed);
           if !raw.is_absolute() {
               return Err(AppError::InvalidInput(format!(
                   "project root 必须为绝对路径: {trimmed}"
               )));
           }
           // Canonicalize the root. The root must exist (a real dir to bind).
           let root = raw.canonicalize().map_err(|e| {
               AppError::InvalidInput(format!("无法解析 project root（需为已存在目录）: {trimmed}: {e}"))
           })?;
           if !root.is_dir() {
               return Err(AppError::InvalidInput(format!(
                   "project root 不是目录: {}",
                   root.display()
               )));
           }

           let home = get_home_dir();
           let home_canon = home.canonicalize().unwrap_or(home.clone());

           // Build the set of forbidden anchors (canonicalized where they exist).
           let mut anchors: Vec<PathBuf> = vec![home_canon.clone(), PathBuf::from("/")];
           for sub in [".claude", ".codex", ".gemini", ".agenthub"] {
               anchors.push(canon_or_join(&home_canon, sub));
           }
           // ~/.config/opencode
           anchors.push(canon_or_join(&home_canon.join(".config"), "opencode"));
           // app-config dir (override-aware) — usually ~/.agenthub
           let app_cfg = get_app_config_dir();
           anchors.push(app_cfg.canonicalize().unwrap_or(app_cfg));

           for anchor in &anchors {
               if &root == anchor {
                   return Err(refuse(&root, "等于受保护目录(HOME/工具目录/应用配置/根)"));
               }
               if anchor.starts_with(&root) {
                   // root is an ancestor of a protected anchor
                   return Err(refuse(&root, "是受保护目录的祖先(HOME/工具目录/应用配置/根)"));
               }
               if root.starts_with(anchor) {
                   // root is a descendant of a protected anchor
                   return Err(refuse(&root, "位于受保护目录内(HOME/工具目录/应用配置/根)"));
               }
           }

           // <root>/.claude collapse check: if the app dotdir canonicalizes to ~/.claude, refuse.
           let dotdir = root.join(app.project_dotdir());
           let claude_canon = canon_or_join(&home_canon, ".claude");
           if let Ok(dotdir_canon) = dotdir.canonicalize() {
               if dotdir_canon == claude_canon {
                   return Err(refuse(&root, "<root>/.claude 解析后等于 ~/.claude（HOME 坍缩）"));
               }
           }

           Ok(ProjectBase { root, dotdir })
       }
   }

   fn refuse(root: &Path, why: &str) -> AppError {
       AppError::InvalidInput(format!(
           "拒绝绑定/应用：project root {} {why} [HOME-safety]",
           root.display()
       ))
   }

   /// Canonicalize `base/sub` if it exists, else return the lexical join (still a
   /// valid anchor for prefix checks).
   fn canon_or_join(base: &Path, sub: &str) -> PathBuf {
       let p = base.join(sub);
       p.canonicalize().unwrap_or(p)
   }
   ```

5. - [ ] Run all gate tests — expect PASS:
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo test --lib project_paths:: 2>&1 | tail -15`
   Expected: `test result: ok. 9 passed` (accepts_normal + 8 refusal cases).

6. - [ ] clippy (the gate is the security perimeter; keep it warning-clean):
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo clippy --all-targets -- -D warnings 2>&1 | tail -8`
   Expected: `Finished` (no warnings).

7. - [ ] fmt + commit:
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo fmt; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && git add -A && git commit -m "feat(projects): ProjectBase + path-safety perimeter gate (CRITICAL #1 HOME-collapse)"`

---

## Task 6 — Shared base-aware content-file writer (extract `validate_name` + parent-escape)

**Files:**
- Modify `src-tauri/src/services/project_paths.rs` (add `validate_content_name` + `content_file_path` free fns)
- Test: inline `#[cfg(test)]` in `project_paths.rs`

**Key facts verified:** `command.rs:62`/`agent.rs:62` define an IDENTICAL private `validate_name` (`^[A-Za-z0-9._-]+$`, reject `.`/`..`/separators) and a private `file_path` that asserts the resolved file's parent equals the target dir (command.rs:87-100). These are private + hardcoded to `commands_dir()`/`agents_dir()`. Per brief §1 we extract a base-aware shared helper rather than reuse. We do NOT change command.rs/agent.rs in 4a (they stay byte-identical for the global channel — zero regression); the project writer uses the new shared helper.

### Steps

1. - [ ] Write failing test in `project_paths.rs` test module:
   ```rust
   #[test]
   fn content_file_path_validates_and_resolves_under_base() {
       let dir = TempDir::new().expect("tmp");
       let base = dir.path();
       let ok = content_file_path(base, "my-cmd").expect("valid name");
       assert_eq!(ok, base.join("my-cmd.md"));
       assert_eq!(ok.parent().unwrap(), base);

       // path-escape / traversal must be rejected
       for bad in ["..", ".", "a/b", "a\\b", "../escape", ""] {
           assert!(content_file_path(base, bad).is_err(), "name {bad:?} must be rejected");
       }
   }

   #[test]
   fn validate_content_name_matches_command_rules() {
       assert!(validate_content_name("ok_name-1.2").is_ok());
       assert!(validate_content_name("bad/name").is_err());
       assert!(validate_content_name("..").is_err());
       assert!(validate_content_name("").is_err());
       assert!(validate_content_name("space name").is_err());
   }
   ```

2. - [ ] Run it — expect FAIL: `cannot find function \`content_file_path\``.
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo test --lib content_file_path_validates_and_resolves_under_base validate_content_name_matches_command_rules 2>&1 | tail -15`

3. - [ ] Add the two free fns to `project_paths.rs` (after the `canon_or_join` helper):
   ```rust
   /// Validate a command/agent content name. Mirrors the (private) rules in
   /// command.rs / agent.rs `validate_name`: non-empty, `^[A-Za-z0-9._-]+$`, not
   /// "." / "..", no path separators. This is the SAFETY gate for the base-aware
   /// project content-file writer.
   pub fn validate_content_name(name: &str) -> Result<(), AppError> {
       if name.is_empty() {
           return Err(AppError::InvalidInput("内容名不能为空".into()));
       }
       if name == "." || name == ".." {
           return Err(AppError::InvalidInput(format!("非法内容名: {name}")));
       }
       if name.contains('/') || name.contains('\\') {
           return Err(AppError::InvalidInput(format!(
               "内容名不能包含路径分隔符: {name}"
           )));
       }
       if !name
           .chars()
           .all(|c| c.is_ascii_alphanumeric() || matches!(c, '.' | '_' | '-'))
       {
           return Err(AppError::InvalidInput(format!(
               "内容名只能包含字母、数字、'.'、'_'、'-': {name}"
           )));
       }
       Ok(())
   }

   /// Resolve `<base>/<name>.md`, validating the name and asserting (defence in
   /// depth) that the resolved file's parent is exactly `base`.
   pub fn content_file_path(base: &Path, name: &str) -> Result<PathBuf, AppError> {
       validate_content_name(name)?;
       let path = base.join(format!("{name}.md"));
       match path.parent() {
           Some(parent) if parent == base => Ok(path),
           _ => Err(AppError::InvalidInput(format!(
               "内容文件路径逃逸出目标目录: {name}"
           ))),
       }
   }
   ```

4. - [ ] Run it — expect PASS:
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo test --lib content_file_path_validates_and_resolves_under_base validate_content_name_matches_command_rules 2>&1 | tail -10`
   Expected: `test result: ok. 2 passed`.

5. - [ ] fmt + commit:
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo fmt; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && git add -A && git commit -m "feat(projects): base-aware content-file name gate + path resolver"`

---

## Task 7 — `SkillService::sync_to_project_dir` / `remove_from_project_dir` (force COPY)

**Files:**
- Modify `src-tauri/src/services/skill.rs` (add two `pub fn` in the existing `impl SkillService`, near `sync_to_app_dir` :1589 / `remove_from_app` :1832)
- Test: inline `#[cfg(test)]` in `skill.rs` (uses `serial_test` + TempHome + SSOT skill)

**Key facts verified:** `sync_to_app_dir` (skill.rs:1589) computes `app_dir = get_app_skills_dir(app)` then `dest = app_dir.join(directory)`, guards `dest.exists() && !is_symlink` (skip to protect user data), then branches on `get_sync_method()`. `replace_dest_with_copy` (private, :1698) does the real copy with `validate_sync_source_dir` (requires `SKILL.md`). `is_managed_app_skill`/`remove_managed_app_skill` (private, :1777/:1812) is the 3a-hardened content-hash discriminator that never `remove_dir_all`s a real user dir. `get_ssot_dir` (:480) + `copy_dir_recursive` (:2391) + `compute_dir_hash` (:831) + `is_symlink` (:1572) are reachable. The project variant takes an explicit `skills_base: &Path` (= `<project>/.claude/skills`) and FORCES copy (skips the symlink branch entirely).

### Steps

1. - [ ] Write failing test in `skill.rs` test module (mirror `materialize_ssot_skill` helper from profile.rs:504):
   ```rust
   #[test]
   #[serial]
   fn sync_to_project_dir_copies_real_dir_not_symlink() {
       let _home = crate::services::skill::tests_support::TempHome::new();
       // materialize an SSOT skill with SKILL.md
       let ssot = SkillService::get_ssot_dir().expect("ssot");
       let sdir = ssot.join("proj-skill");
       std::fs::create_dir_all(&sdir).expect("mkdir ssot skill");
       std::fs::write(sdir.join("SKILL.md"), "---\nname: proj-skill\n---\n# x\n").expect("write");

       let tmp = tempfile::TempDir::new().expect("proj tmp");
       let skills_base = tmp.path().join(".claude").join("skills");

       SkillService::sync_to_project_dir("proj-skill", &skills_base, &AppType::Claude)
           .expect("sync to project");

       let dest = skills_base.join("proj-skill");
       assert!(dest.join("SKILL.md").is_file(), "skill must be materialized");
       // CRITICAL: it is a REAL COPY, not a symlink
       let meta = std::fs::symlink_metadata(&dest).expect("meta");
       assert!(!meta.file_type().is_symlink(), "project skill MUST be a copy, not a symlink");

       // remove path is owned-safe (managed copy) → removed
       SkillService::remove_from_project_dir("proj-skill", &skills_base, &AppType::Claude)
           .expect("remove");
       assert!(!dest.exists(), "managed copy should be removed on detach");
   }

   #[test]
   #[serial]
   fn remove_from_project_dir_skips_real_user_dir() {
       let _home = crate::services::skill::tests_support::TempHome::new();
       let tmp = tempfile::TempDir::new().expect("proj tmp");
       let skills_base = tmp.path().join(".claude").join("skills");
       let user_dir = skills_base.join("user-skill");
       std::fs::create_dir_all(&user_dir).expect("mkdir user dir");
       std::fs::write(user_dir.join("notes.txt"), "user data").expect("write user file");
       // no SSOT source for "user-skill" → not a managed copy → must be skipped
       SkillService::remove_from_project_dir("user-skill", &skills_base, &AppType::Claude)
           .expect("remove (skip)");
       assert!(user_dir.join("notes.txt").is_file(), "user dir must NOT be deleted");
   }
   ```
   NOTE: if `skill.rs` has no reusable `TempHome`, reference the one in its own test module (search `skill.rs` for the existing TempHome guard used by `replace_dest_with_copy_rejects_empty_source...` at :3181 and reuse it; if it is private to the test mod, the new tests live in the same mod so they share it — drop the `tests_support::` path and call `TempHome::new()` directly).

2. - [ ] Run it — expect FAIL: `no function ... sync_to_project_dir`.
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo test --lib sync_to_project_dir_copies_real_dir_not_symlink remove_from_project_dir_skips_real_user_dir 2>&1 | tail -20`

3. - [ ] Add the two methods to `impl SkillService` near `sync_to_app_dir`:
   ```rust
       /// 同步 Skill 到 *项目* skills 目录（显式 base，强制 COPY；绝不 symlink）。
       ///
       /// 与 sync_to_app_dir 的区别：base 由调用方给定（= <project>/.claude/skills），
       /// 且无视全局 SyncMethod 设置 —— 项目内容必须自包含、git/WebDAV 可携带，故恒用 copy。
       /// 复用 3a 的数据安全不变式：dest 是真实(非 symlink) 用户目录时跳过 + warn，绝不销毁。
       pub fn sync_to_project_dir(
           directory: &str,
           skills_base: &Path,
           app: &AppType,
       ) -> Result<()> {
           if matches!(app, AppType::ClaudeDesktop) {
               return Ok(());
           }
           let ssot_dir = Self::get_ssot_dir()?;
           let source = ssot_dir.join(directory);
           Self::validate_sync_source_dir(&source, directory)?;

           fs::create_dir_all(skills_base)?;
           let dest = skills_base.join(directory);

           // 数据安全不变式（与 sync_to_app_dir 对称）：真实(非 symlink) 用户目录 → 跳过保护。
           if dest.exists() && !Self::is_symlink(&dest) && !Self::is_managed_app_skill(&dest, directory) {
               log::warn!(
                   "project skill {directory}: a real user directory exists at the project skills path; skipping to protect user data"
               );
               return Ok(());
           }

           // 强制 COPY（不走 symlink 分支）。
           Self::replace_dest_with_copy(&source, &dest, directory)?;
           log::debug!("Skill {directory} 已通过复制同步到项目目录 {}", dest.display());
           Ok(())
       }

       /// 从 *项目* skills 目录删除 Skill（仅删除 AgentHub 托管产物：symlink 或哈希匹配 SSOT 的 copy）。
       pub fn remove_from_project_dir(
           directory: &str,
           skills_base: &Path,
           app: &AppType,
       ) -> Result<()> {
           if matches!(app, AppType::ClaudeDesktop) {
               return Ok(());
           }
           let dest = skills_base.join(directory);
           Self::remove_managed_app_skill(&dest, directory, app)
       }
   ```

4. - [ ] Run it — expect PASS:
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo test --lib sync_to_project_dir_copies_real_dir_not_symlink remove_from_project_dir_skips_real_user_dir 2>&1 | tail -12`
   Expected: `test result: ok. 2 passed`.

5. - [ ] clippy + fmt + commit:
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo clippy --all-targets -- -D warnings 2>&1 | tail -5; cargo fmt; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && git add -A && git commit -m "feat(projects): SkillService::sync_to_project_dir / remove_from_project_dir (force COPY)"`

---

## Task 8 — `ProjectApplyService::apply/detach` skeleton: resolve_selectors + commands/agents writer + manifest (write-then-record)

**Files:**
- Create `src-tauri/src/services/project_apply.rs`
- Modify `src-tauri/src/services/mod.rs` (add `pub mod project_apply;`)
- Test: inline `#[cfg(test)]` in `project_apply.rs`

**Key facts verified:** `ProfileService::resolve_selectors` (profile.rs:452) is PRIVATE — signature `fn resolve_selectors(type_label: &str, spec: &[String], items: &[(String, Vec<String>)]) -> (HashSet<String>, Vec<String>)`. It is not callable cross-module. Per brief §1 ("Reuse ProfileService::resolve_selectors for literal + @tag") we must make it reusable: change it to `pub(crate)` (one-word visibility bump, no behavior change, global byte-identical). `state.db` is `Arc<Database>`. `config::atomic_write(path, bytes)` writes atomically. `chrono::Utc::now().timestamp()`. This task does commands + agents only (skills added in Task 9 split) plus the manifest write-then-record + crash-ordering test. `apply` clears the project channel first, then for each enabled command/agent: write file → IMMEDIATELY record manifest row (write-then-record).

### Steps

1. - [ ] Make `resolve_selectors` reusable. In `profile.rs:452` change:
   ```rust
       pub(crate) fn resolve_selectors(
   ```
   (was `fn resolve_selectors`). No other change — global behavior byte-identical.

2. - [ ] Add the module declaration to `services/mod.rs` (after `pub mod project_paths;`):
   ```rust
   pub mod project_apply;
   ```

3. - [ ] Write the failing test file. Create `src-tauri/src/services/project_apply.rs` with imports + the result struct doc + a full test module (impl added in step 5). The first test exercises commands/agents materialization + manifest + the write-then-record crash-ordering invariant:
   ```rust
   //! ProjectApplyService (increment 4a, Claude only).
   //!
   //! Materializes a project's own content set (skills COPY / commands / agents)
   //! into <project>/.claude/{skills,commands,agents}/ and records every written
   //! file as a project-channel apply_manifest row (channel="project:<canonpath>",
   //! project_id, kind, content_hash). Detach owned-deletes exactly those rows.
   //!
   //! SAFETY: every path goes through ProjectBase::resolve (CRITICAL #1). Reconcile
   //! is keyed on project_id: snapshot rows whose project_id differs from the
   //! currently-bound project are FOREIGN and are skipped+warned, never deleted
   //! (CRITICAL #1 corollary / path-reuse). write-then-record ordering mirrors the
   //! global path so a crash leaves at most a recorded-and-written file.

   use std::collections::HashSet;
   use std::path::Path;

   use crate::app_config::{AppType, ManifestEntry, Project};
   use crate::config::atomic_write;
   use crate::error::AppError;
   use crate::services::profile::ProfileService;
   use crate::services::project_paths::{content_file_path, ProjectBase};
   use crate::store::AppState;

   #[cfg(test)]
   mod tests {
       use super::*;
       use crate::app_config::{ProfileContent, ProjectSpec};
       use crate::database::Database;
       use serial_test::serial;
       use std::env;
       use std::sync::Arc;
       use tempfile::TempDir;

       struct TempHome {
           dir: TempDir,
           oh: Option<String>,
           ou: Option<String>,
           ot: Option<String>,
       }
       impl TempHome {
           fn new() -> Self {
               let dir = TempDir::new().expect("tmp");
               let oh = env::var("HOME").ok();
               let ou = env::var("USERPROFILE").ok();
               let ot = env::var("CC_SWITCH_TEST_HOME").ok();
               env::set_var("HOME", dir.path());
               env::set_var("USERPROFILE", dir.path());
               env::set_var("CC_SWITCH_TEST_HOME", dir.path());
               Self { dir, oh, ou, ot }
           }
           fn home(&self) -> &Path { self.dir.path() }
       }
       impl Drop for TempHome {
           fn drop(&mut self) {
               match &self.oh { Some(v) => env::set_var("HOME", v), None => env::remove_var("HOME") }
               match &self.ou { Some(v) => env::set_var("USERPROFILE", v), None => env::remove_var("USERPROFILE") }
               match &self.ot { Some(v) => env::set_var("CC_SWITCH_TEST_HOME", v), None => env::remove_var("CC_SWITCH_TEST_HOME") }
           }
       }

       fn seed_command(db: &Database, name: &str, content: &str) {
           let c = crate::app_config::InstalledCommand {
               id: format!("local:{name}"),
               name: name.into(),
               content: content.into(),
               description: None,
               tags: vec![],
               enabled_claude: false,
               installed_at: 0,
           };
           db.save_command(&c).expect("save command");
       }
       fn seed_agent(db: &Database, name: &str, content: &str) {
           let a = crate::app_config::InstalledAgent {
               id: format!("local:{name}"),
               name: name.into(),
               content: content.into(),
               description: None,
               tags: vec![],
               enabled_claude: false,
               installed_at: 0,
           };
           db.save_agent(&a).expect("save agent");
       }

       fn project_at(home: &Path, sub: &str, content: ProfileContent) -> (Project, std::path::PathBuf) {
           let root = home.join(sub);
           std::fs::create_dir_all(&root).expect("mkdir proj");
           let canon = root.canonicalize().expect("canon");
           let p = Project {
               id: format!("proj:{sub}"),
               project_path: canon.to_string_lossy().to_string(),
               entered_path: root.to_string_lossy().to_string(),
               app_type: "claude".into(),
               name: Some(sub.into()),
               spec: ProjectSpec { content, vars: serde_json::Map::new() },
               enabled: true,
               created_at: 1,
               updated_at: 1,
           };
           (p, canon)
       }

       #[test]
       #[serial]
       fn apply_writes_commands_and_agents_then_records_manifest() {
           let home = TempHome::new();
           crate::settings::reload_settings().ok();
           let db = Arc::new(Database::memory().expect("db"));
           let state = AppState::new(db.clone());
           seed_command(&db, "cmd-foo", "FOO");
           seed_agent(&db, "agent-bar", "BAR");

           let (proj, canon) = project_at(
               home.home(),
               "work-a",
               ProfileContent {
                   skills: vec![],
                   commands: vec!["cmd-foo".into()],
                   agents: vec!["agent-bar".into()],
                   mcp: vec![],
               },
           );
           db.save_project(&proj).expect("save project");

           let res = ProjectApplyService::apply(&state, &proj.id).expect("apply");
           assert!(res.warnings.is_empty(), "no warnings: {:?}", res.warnings);

           let cmd_file = canon.join(".claude").join("commands").join("cmd-foo.md");
           let agent_file = canon.join(".claude").join("agents").join("agent-bar.md");
           assert_eq!(std::fs::read_to_string(&cmd_file).unwrap(), "FOO");
           assert_eq!(std::fs::read_to_string(&agent_file).unwrap(), "BAR");

           // write-then-record: every written file has a manifest row on the project channel,
           // tied to project_id, with content_hash. (crash-ordering: no unrecorded written file.)
           let chan = format!("project:{}", canon.to_string_lossy());
           let rows = db.get_manifest_for_channel(&chan).expect("rows");
           assert_eq!(rows.len(), 2, "two rows: one command, one agent");
           for r in &rows {
               assert_eq!(r.project_id.as_deref(), Some(proj.id.as_str()));
               assert_eq!(r.channel, chan);
               assert!(r.content_hash.is_some(), "every recorded file must carry a content_hash");
               assert!(Path::new(&r.target_path).is_file(), "recorded target must exist on disk");
           }
       }

       #[test]
       #[serial]
       fn detach_owned_deletes_recorded_files_and_clears_rows() {
           let home = TempHome::new();
           crate::settings::reload_settings().ok();
           let db = Arc::new(Database::memory().expect("db"));
           let state = AppState::new(db.clone());
           seed_command(&db, "cmd-foo", "FOO");
           let (proj, canon) = project_at(
               home.home(),
               "work-b",
               ProfileContent { skills: vec![], commands: vec!["cmd-foo".into()], agents: vec![], mcp: vec![] },
           );
           db.save_project(&proj).expect("save");
           ProjectApplyService::apply(&state, &proj.id).expect("apply");

           let cmd_file = canon.join(".claude").join("commands").join("cmd-foo.md");
           assert!(cmd_file.is_file());

           // user writes their OWN file into the same dir — must survive detach
           let user_file = canon.join(".claude").join("commands").join("user-own.md");
           std::fs::write(&user_file, "USER").expect("write user file");

           ProjectApplyService::detach(&state, &proj.id).expect("detach");

           assert!(!cmd_file.exists(), "owned command file must be removed on detach");
           assert!(user_file.is_file(), "user file must NOT be touched on detach");
           let chan = format!("project:{}", canon.to_string_lossy());
           assert_eq!(db.get_manifest_for_channel(&chan).unwrap().len(), 0, "rows cleared");
           // dir itself is left in place (brief §2: no directory removal on detach)
           assert!(cmd_file.parent().unwrap().is_dir(), ".claude/commands dir stays");
       }

       #[test]
       #[serial]
       fn detach_skips_user_edited_owned_file() {
           let home = TempHome::new();
           crate::settings::reload_settings().ok();
           let db = Arc::new(Database::memory().expect("db"));
           let state = AppState::new(db.clone());
           seed_command(&db, "cmd-foo", "FOO");
           let (proj, canon) = project_at(
               home.home(),
               "work-c",
               ProfileContent { skills: vec![], commands: vec!["cmd-foo".into()], agents: vec![], mcp: vec![] },
           );
           db.save_project(&proj).expect("save");
           ProjectApplyService::apply(&state, &proj.id).expect("apply");

           // user EDITS the materialized file → hash no longer matches → detach must skip+warn
           let cmd_file = canon.join(".claude").join("commands").join("cmd-foo.md");
           std::fs::write(&cmd_file, "USER EDITED").expect("edit");
           ProjectApplyService::detach(&state, &proj.id).expect("detach");
           assert_eq!(std::fs::read_to_string(&cmd_file).unwrap(), "USER EDITED", "user edit preserved");
       }
   }
   ```

4. - [ ] Run it — expect FAIL: `ProjectApplyService` undefined.
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo test --lib apply_writes_commands_and_agents_then_records_manifest 2>&1 | tail -20`

5. - [ ] Add the service impl ABOVE the test module in `project_apply.rs`:
   ```rust
   /// Result of a project apply/detach: non-fatal warnings (mirrors ActivateResult).
   #[derive(Debug, Default, serde::Serialize)]
   #[serde(rename_all = "camelCase")]
   pub struct ProjectApplyResult {
       pub warnings: Vec<String>,
   }

   pub struct ProjectApplyService;

   impl ProjectApplyService {
       /// Apply a project's own content set into <project>/.claude (Claude only, 4a).
       pub fn apply(state: &AppState, project_id: &str) -> Result<ProjectApplyResult, AppError> {
           let mut result = ProjectApplyResult::default();

           let project = state
               .db
               .get_project(project_id)?
               .ok_or_else(|| AppError::Message(format!("project not found: {project_id}")))?;
           if project.app_type != AppType::Claude.as_str() {
               return Err(AppError::Message(format!(
                   "4a applies Claude only; project app_type is {}",
                   project.app_type
               )));
           }
           if !project.enabled {
               result.warnings.push("project is paused (enabled=false); apply skipped".into());
               return Ok(result);
           }
           let app = AppType::Claude;

           // CRITICAL #1: re-run the path-safety gate on EVERY apply.
           let base = ProjectBase::resolve(&project.entered_path, &app)?;
           let channel = base.channel();

           // CRITICAL #1 corollary (path-reuse): inspect the previous channel snapshot.
           // Any row whose project_id differs from THIS project's id is foreign — skip+warn,
           // never reconcile/delete. Then owned-delete + clear only OUR rows before re-apply.
           let prev = state.db.get_manifest_for_channel(&channel)?;
           let (ours, foreign): (Vec<_>, Vec<_>) = prev
               .into_iter()
               .partition(|r| r.project_id.as_deref() == Some(project.id.as_str()));
           if !foreign.is_empty() {
               result.warnings.push(format!(
                   "channel {channel} has {} row(s) bound to a different project; skipping them (path reused?)",
                   foreign.len()
               ));
           }
           // owned-delete OUR previous rows' targets, then clear OUR rows (idempotent re-apply).
           for r in &ours {
               crate::services::profile_render::remove_whole_file_if_owned(
                   &r.target_path,
                   r.content_hash.as_deref(),
               )?;
           }
           let our_ids: Vec<i64> = ours.iter().map(|r| r.id).collect();
           state.db.delete_manifest_entries(&our_ids)?;

           let content = &project.spec.content;

           // ---- commands (key = command.name) ----
           let cmds = state.db.get_all_installed_commands()?;
           let items: Vec<(String, Vec<String>)> =
               cmds.iter().map(|c| (c.name.clone(), c.tags.clone())).collect();
           let (want, w) = ProfileService::resolve_selectors("command", &content.commands, &items);
           result.warnings.extend(w);
           let cmd_dir = base.dotdir().join("commands");
           for c in &cmds {
               if !want.contains(&c.name) {
                   continue;
               }
               let target = content_file_path(&cmd_dir, &c.name)?;
               // write-then-record: write the file FIRST, then record the manifest row.
               atomic_write(&target, c.content.as_bytes())?;
               let hash = crate::services::profile_render::content_hash(c.content.as_bytes());
               state.db.record_manifest_entry(&Self::row(
                   &channel, &project.id, &target, "command", &hash,
               ))?;
           }

           // ---- agents (key = agent.name) ----
           let agents = state.db.get_all_installed_agents()?;
           let items: Vec<(String, Vec<String>)> =
               agents.iter().map(|a| (a.name.clone(), a.tags.clone())).collect();
           let (want, w) = ProfileService::resolve_selectors("agent", &content.agents, &items);
           result.warnings.extend(w);
           let agent_dir = base.dotdir().join("agents");
           for a in &agents {
               if !want.contains(&a.name) {
                   continue;
               }
               let target = content_file_path(&agent_dir, &a.name)?;
               atomic_write(&target, a.content.as_bytes())?;
               let hash = crate::services::profile_render::content_hash(a.content.as_bytes());
               state.db.record_manifest_entry(&Self::row(
                   &channel, &project.id, &target, "agent", &hash,
               ))?;
           }

           // NOTE: skills (COPY) are added in Task 9.
           let _ = &result; // keep mutable in this task
           Ok(result)
       }

       /// Detach: owned-delete every recorded file for this project's channel, then clear rows.
       /// Leaves empty directories in place (brief §2). Never removes user-edited/unowned files.
       pub fn detach(state: &AppState, project_id: &str) -> Result<ProjectApplyResult, AppError> {
           let mut result = ProjectApplyResult::default();
           let project = state
               .db
               .get_project(project_id)?
               .ok_or_else(|| AppError::Message(format!("project not found: {project_id}")))?;
           let app = AppType::Claude;
           let base = ProjectBase::resolve(&project.entered_path, &app)?;
           let channel = base.channel();

           let rows = state.db.get_manifest_for_channel(&channel)?;
           for r in &rows {
               // CRITICAL #1 corollary: only touch OUR rows.
               if r.project_id.as_deref() != Some(project.id.as_str()) {
                   result.warnings.push(format!(
                       "skipping foreign manifest row (project_id={:?}) on detach",
                       r.project_id
                   ));
                   continue;
               }
               // owned-delete: only if on-disk hash == recorded hash.
               crate::services::profile_render::remove_whole_file_if_owned(
                   &r.target_path,
                   r.content_hash.as_deref(),
               )?;
           }
           // clear only OUR rows; leave foreign rows intact.
           let our_ids: Vec<i64> = rows
               .iter()
               .filter(|r| r.project_id.as_deref() == Some(project.id.as_str()))
               .map(|r| r.id)
               .collect();
           state.db.delete_manifest_entries(&our_ids)?;
           Ok(result)
       }

       fn row(
           channel: &str,
           project_id: &str,
           target: &Path,
           kind: &str,
           content_hash: &str,
       ) -> ManifestEntry {
           ManifestEntry {
               id: 0,
               channel: channel.to_string(),
               profile_id: None,
               project_id: Some(project_id.to_string()),
               app_type: AppType::Claude.as_str().to_string(),
               target_path: target.to_string_lossy().to_string(),
               kind: kind.to_string(),
               content_hash: Some(content_hash.to_string()),
               created_at: chrono::Utc::now().timestamp(),
           }
       }
   }
   ```

6. - [ ] Run the three tests — expect PASS:
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo test --lib apply_writes_commands_and_agents_then_records_manifest detach_owned_deletes_recorded_files_and_clears_rows detach_skips_user_edited_owned_file 2>&1 | tail -12`
   Expected: `test result: ok. 3 passed`.

7. - [ ] clippy + fmt + commit:
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo clippy --all-targets -- -D warnings 2>&1 | tail -5; cargo fmt; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && git add -A && git commit -m "feat(projects): ProjectApplyService apply/detach (commands+agents, write-then-record, owned-delete)"`

---

## Task 9 — ProjectApplyService: add skills COPY + path-reuse-guard test + path-gone prune

**Files:**
- Modify `src-tauri/src/services/project_apply.rs` (add skills loop in `apply`; add `prune_orphan_project_channels`; WIRE it as apply-time cleanup at the start of `apply`)
- Test: inline `#[cfg(test)]` in `project_apply.rs`

**WIRING (brief §3.2 — prune MUST be reachable in production):** Brief §3.2 requires the path-gone prune to run via "a Tauri command OR apply-time cleanup". We choose apply-time cleanup (the lower-cost option): `apply` calls `prune_orphan_project_channels` best-effort at its very start (before reading the channel snapshot), so every apply sweeps away stale `project:<canonpath>` manifest rows whose underlying directory has been deleted. Because the design deliberately drops FK CASCADE (DESIGN DECISION, Task 1), this is the SOLE production mechanism that cleans up orphaned project-channel rows — so it MUST be invoked from a real code path, not only its unit test. Step 5b below adds the call; the new assertion in step 1 (`apply_prunes_path_gone_channel_before_reconcile`) exercises it through `apply()` (a production entry point), not just the standalone `prune_removes_channels_whose_path_is_gone` unit test.

**Key facts verified:** `SkillService::sync_to_project_dir(directory, skills_base, app)` (Task 7) forces copy. For the manifest, a copied skill is a directory, not a single file; we record ONE manifest row per skill with `kind="skill"`, `target_path = <skills_base>/<directory>`, and `content_hash = SkillService::compute_dir_hash(dest)` (pub, skill.rs:831) so detach can verify ownership of the copied dir before removing it via `remove_from_project_dir`. Detach must branch: `kind=="skill"` → `remove_from_project_dir` (dir-aware, content-hash-safe); else → `remove_whole_file_if_owned`.

### Steps

1. - [ ] Write failing tests in `project_apply.rs` test module — skills COPY, foreign-project path-reuse protection, and path-gone prune:
   ```rust
   #[test]
   #[serial]
   fn apply_materializes_skills_as_copy_and_detach_removes() {
       let home = TempHome::new();
       crate::settings::reload_settings().ok();
       let db = Arc::new(Database::memory().expect("db"));
       let state = AppState::new(db.clone());

       // SSOT skill + DB installed skill row keyed by directory
       let ssot = crate::services::SkillService::get_ssot_dir().expect("ssot");
       let sdir = ssot.join("proj-skill");
       std::fs::create_dir_all(&sdir).expect("mkdir");
       std::fs::write(sdir.join("SKILL.md"), "---\nname: proj-skill\n---\n# x\n").expect("write");
       // InstalledSkill does NOT derive Default (app_config.rs:167 derives only
       // Debug/Clone/Serialize/Deserialize), so spell out EVERY field — no `..Default::default()`.
       // SkillApps DOES derive Default, so SkillApps::default() is fine.
       let skill = crate::app_config::InstalledSkill {
           id: "local:proj-skill".into(),
           name: "proj-skill".into(),
           description: None,
           directory: "proj-skill".into(),
           repo_owner: None,
           repo_name: None,
           repo_branch: None,
           readme_url: None,
           apps: crate::app_config::SkillApps::default(),
           installed_at: 0,
           content_hash: None,
           updated_at: 0,
           tags: vec![],
       };
       db.save_skill(&skill).expect("save skill");

       let (proj, canon) = project_at(
           home.home(),
           "work-s",
           ProfileContent { skills: vec!["proj-skill".into()], commands: vec![], agents: vec![], mcp: vec![] },
       );
       db.save_project(&proj).expect("save");
       ProjectApplyService::apply(&state, &proj.id).expect("apply");

       let dest = canon.join(".claude").join("skills").join("proj-skill");
       assert!(dest.join("SKILL.md").is_file(), "skill copied");
       assert!(!std::fs::symlink_metadata(&dest).unwrap().file_type().is_symlink(), "must be COPY");

       let chan = format!("project:{}", canon.to_string_lossy());
       let rows = db.get_manifest_for_channel(&chan).unwrap();
       assert!(rows.iter().any(|r| r.kind == "skill" && r.target_path == dest.to_string_lossy()));

       ProjectApplyService::detach(&state, &proj.id).expect("detach");
       assert!(!dest.exists(), "copied skill removed on detach");
   }

   #[test]
   #[serial]
   fn apply_does_not_reconcile_foreign_project_rows() {
       let home = TempHome::new();
       crate::settings::reload_settings().ok();
       let db = Arc::new(Database::memory().expect("db"));
       let state = AppState::new(db.clone());
       seed_command(&db, "cmd-foo", "FOO");

       let (proj, canon) = project_at(
           home.home(),
           "reused-path",
           ProfileContent { skills: vec![], commands: vec!["cmd-foo".into()], agents: vec![], mcp: vec![] },
       );
       db.save_project(&proj).expect("save");
       let chan = format!("project:{}", canon.to_string_lossy());

       // Simulate a STALE row from a DIFFERENT project at the same canonical path
       // (old repo deleted + re-cloned; project_id differs). Its target is a real user file.
       // This row's project_id ("proj:OLD") has NO matching projects row — that is only
       // insertable because project_id carries NO FK (design decision, Task 1); with the
       // FK + PRAGMA foreign_keys=ON this record_manifest_entry would fail and the test
       // could never exercise the §3.2 path-reuse protection.
       let foreign_target = canon.join(".claude").join("commands").join("foreign.md");
       std::fs::create_dir_all(foreign_target.parent().unwrap()).unwrap();
       std::fs::write(&foreign_target, "FOREIGN USER DATA").unwrap();
       db.record_manifest_entry(&ManifestEntry {
           id: 0, channel: chan.clone(), profile_id: None, project_id: Some("proj:OLD".into()),
           app_type: "claude".into(), target_path: foreign_target.to_string_lossy().to_string(),
           kind: "command".into(),
           content_hash: Some(crate::services::profile_render::content_hash(b"FOREIGN USER DATA")),
           created_at: 0,
       }).unwrap();

       // Apply the NEW project — must NOT delete the foreign row's file (skip+warn), only manage its own.
       let res = ProjectApplyService::apply(&state, &proj.id).expect("apply");
       assert!(res.warnings.iter().any(|w| w.contains("different project")), "warn about foreign rows");
       assert!(foreign_target.is_file(), "foreign project's file MUST survive (path-reuse protection)");
       assert_eq!(std::fs::read_to_string(&foreign_target).unwrap(), "FOREIGN USER DATA");
   }

   #[test]
   #[serial]
   fn prune_removes_channels_whose_path_is_gone() {
       let _home = TempHome::new();
       let db = Arc::new(Database::memory().expect("db"));
       // a project-channel row pointing at a non-existent path, whose project_id
       // ("proj:gone") has no projects row — insertable only because project_id
       // carries NO FK (design decision, Task 1).
       db.record_manifest_entry(&ManifestEntry {
           id: 0, channel: "project:/no/such/path/xyz".into(), profile_id: None,
           project_id: Some("proj:gone".into()), app_type: "claude".into(),
           target_path: "/no/such/path/xyz/.claude/commands/a.md".into(),
           kind: "command".into(), content_hash: Some("h".into()), created_at: 0,
       }).unwrap();
       let pruned = ProjectApplyService::prune_orphan_project_channels(&db).expect("prune");
       assert!(pruned >= 1);
       assert_eq!(db.get_manifest_for_channel("project:/no/such/path/xyz").unwrap().len(), 0);
   }

   #[test]
   #[serial]
   fn apply_prunes_path_gone_channel_before_reconcile() {
       // Brief §3.2: the path-gone prune MUST be reachable in production. apply()
       // runs prune_orphan_project_channels at its start, so a pre-seeded stale
       // channel (its dir deleted) is swept away by a real apply — proving the
       // prune is wired to a production entry point, not just its own unit test.
       let home = TempHome::new();
       crate::settings::reload_settings().ok();
       let db = Arc::new(Database::memory().expect("db"));
       let state = AppState::new(db.clone());
       seed_command(&db, "cmd-foo", "FOO");

       // a stale project-channel row whose path no longer exists (old repo deleted).
       let gone_chan = "project:/no/such/gone/repo";
       db.record_manifest_entry(&ManifestEntry {
           id: 0, channel: gone_chan.into(), profile_id: None,
           project_id: Some("proj:GONE".into()), app_type: "claude".into(),
           target_path: "/no/such/gone/repo/.claude/commands/old.md".into(),
           kind: "command".into(), content_hash: Some("h".into()), created_at: 0,
       }).unwrap();
       assert_eq!(db.get_manifest_for_channel(gone_chan).unwrap().len(), 1);

       // a real, live project at an existing path — apply it normally.
       let (proj, _canon) = project_at(
           home.home(),
           "live-repo",
           ProfileContent { skills: vec![], commands: vec!["cmd-foo".into()], agents: vec![], mcp: vec![] },
       );
       db.save_project(&proj).expect("save");
       ProjectApplyService::apply(&state, &proj.id).expect("apply");

       // apply-time cleanup pruned the path-gone channel (production entry point).
       assert_eq!(
           db.get_manifest_for_channel(gone_chan).unwrap().len(),
           0,
           "apply() must prune the stale path-gone channel via apply-time cleanup"
       );
   }
   ```
   NOTE: the `InstalledSkill` literal above is spelled out in full (all 13 fields: id, name, description, directory, repo_owner, repo_name, repo_branch, readme_url, apps, installed_at, content_hash, updated_at, tags) because `InstalledSkill` does NOT derive `Default` (verified at app_config.rs:167). Do NOT use `..Default::default()` for it. If `save_skill` is named differently, grep `src-tauri/src/database/dao/skills.rs` for the insert method.

2. - [ ] Run it — expect FAIL (skill row not recorded / `prune_orphan_project_channels` undefined / apply does not yet prune):
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo test --lib apply_materializes_skills_as_copy_and_detach_removes apply_does_not_reconcile_foreign_project_rows prune_removes_channels_whose_path_is_gone apply_prunes_path_gone_channel_before_reconcile 2>&1 | tail -20`

3. - [ ] Add the skills loop in `apply` (in `project_apply.rs`), AFTER the agents loop, BEFORE `Ok(result)`. Replace the `// NOTE: skills ...` / `let _ = &result;` lines with:
   ```rust
           // ---- skills (key = skill.directory; COPY, real dir) ----
           let skills = state.db.get_all_installed_skills()?;
           let items: Vec<(String, Vec<String>)> = skills
               .values()
               .map(|s| (s.directory.clone(), s.tags.clone()))
               .collect();
           let (want, w) = ProfileService::resolve_selectors("skill", &content.skills, &items);
           result.warnings.extend(w);
           let skills_base = base.dotdir().join("skills");
           for s in skills.values() {
               if !want.contains(&s.directory) {
                   continue;
               }
               // write-then-record: copy the skill dir FIRST, then record the manifest row
               // (content_hash = dir hash, so detach can verify ownership of the copied dir).
               crate::services::SkillService::sync_to_project_dir(&s.directory, &skills_base, &app)
                   .map_err(|e| AppError::Message(format!("project skill copy failed: {e}")))?;
               let dest = skills_base.join(&s.directory);
               if let Ok(hash) = crate::services::SkillService::compute_dir_hash(&dest) {
                   state.db.record_manifest_entry(&Self::row(
                       &channel,
                       &project.id,
                       &dest,
                       "skill",
                       &hash,
                   ))?;
               }
           }
   ```
   Confirm `get_all_installed_skills` returns a `HashMap`-like with `.values()` (it does — profile.rs:120 uses `skills.values()`).

4. - [ ] Update `detach` to branch on `kind == "skill"`. Replace the owned-delete call inside the detach loop with:
   ```rust
               if r.kind == "skill" {
                   // dir-aware, content-hash-safe removal (never remove_dir_all a real user dir)
                   if let Some(parent) = Path::new(&r.target_path).parent() {
                       let dir_name = Path::new(&r.target_path)
                           .file_name()
                           .map(|n| n.to_string_lossy().to_string())
                           .unwrap_or_default();
                       crate::services::SkillService::remove_from_project_dir(
                           &dir_name,
                           parent,
                           &app,
                       )
                       .map_err(|e| AppError::Message(format!("project skill remove failed: {e}")))?;
                   }
               } else {
                   crate::services::profile_render::remove_whole_file_if_owned(
                       &r.target_path,
                       r.content_hash.as_deref(),
                   )?;
               }
   ```
   Also mirror the same skill-vs-file branch in the `apply` owned-delete of OUR previous rows (the `for r in &ours { ... }` block): apply the identical branch so re-apply of a skills project cleans old copies dir-safely.

5. - [ ] Add `prune_orphan_project_channels` to `impl ProjectApplyService`:
   ```rust
       /// Apply-time / doctor-style prune: drop manifest rows for any project
       /// channel whose underlying directory no longer exists (the path-gone case).
       /// INVOKED in production from `apply` (apply-time cleanup, step 5b) so this
       /// hygiene actually runs — not just from its unit test. Because project_id
       /// has NO FK CASCADE (design decision, Task 1), the project-row-deleted case
       /// is also covered here: a deleted project leaves its `project:<canonpath>`
       /// rows behind, and if its dir is gone this prune removes them; if the dir
       /// still exists, detach (run before delete) or a later re-bind reconcile
       /// clears them. Returns the number of channels pruned.
       pub fn prune_orphan_project_channels(
           db: &crate::database::Database,
       ) -> Result<usize, AppError> {
           let mut pruned = 0usize;
           for channel in db.get_all_project_channels()? {
               let path = channel.strip_prefix("project:").unwrap_or(&channel);
               if !Path::new(path).exists() {
                   db.clear_manifest_for_channel(&channel)?;
                   pruned += 1;
               }
           }
           Ok(pruned)
       }
   ```

5b. - [ ] WIRE the prune into `apply` as apply-time cleanup (brief §3.2: prune MUST be reachable in production). In `project_apply.rs`, inside `ProjectApplyService::apply`, immediately AFTER the path-safety gate resolves `base`/`channel` and BEFORE the `let prev = state.db.get_manifest_for_channel(&channel)?;` snapshot read, insert a best-effort prune (log on error; never fail the apply for a hygiene sweep). Find the lines added in Task 8 step 5:
   ```rust
           let base = ProjectBase::resolve(&project.entered_path, &app)?;
           let channel = base.channel();
   ```
   and insert directly after them:
   ```rust
           // Brief §3.2 apply-time cleanup: best-effort sweep of stale project channels
           // whose underlying dir is gone (the SOLE production path that runs the prune,
           // since project_id carries NO FK CASCADE — design decision, Task 1). Never
           // fail the apply for a hygiene sweep; log and continue.
           if let Err(e) = Self::prune_orphan_project_channels(&state.db) {
               log::warn!("prune_orphan_project_channels (apply-time cleanup) failed: {e}");
           }
   ```
   This makes `apply()` a real production entry point for the prune; `apply_prunes_path_gone_channel_before_reconcile` (step 1) now exercises it. NOTE: `&state.db` is `&Arc<Database>` which auto-derefs to the `&Database` the prune expects.

6. - [ ] Run the four tests — expect PASS:
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo test --lib apply_materializes_skills_as_copy_and_detach_removes apply_does_not_reconcile_foreign_project_rows prune_removes_channels_whose_path_is_gone apply_prunes_path_gone_channel_before_reconcile 2>&1 | tail -12`
   Expected: `test result: ok. 4 passed`.

7. - [ ] clippy + fmt + commit:
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo clippy --all-targets -- -D warnings 2>&1 | tail -5; cargo fmt; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && git add -A && git commit -m "feat(projects): skills COPY materialization + path-reuse guard + orphan-channel prune"`

---

## Task 10a — Tauri commands: project CRUD + set_enabled + manifest read

**Files:**
- Create `src-tauri/src/commands/project.rs`
- Modify `src-tauri/src/commands/mod.rs` (`mod project;` + `pub use project::*;`)
- Test: inline `#[cfg(test)]` round-trip via a helper fn (mirrors `commands/profile.rs` tests which bypass `State<AppState>`)

**Key facts verified:** `commands/profile.rs` registers `#[tauri::command]` fns taking `State<'_, AppState>`; tests use a `run_*_logic(state, ...)` helper bypassing the Tauri wrapper. `commands/mod.rs` has `mod profile;` + `pub use profile::*;`. The seed-from-profile snapshot logic (decision #3) lives in `project_save`: when a `seedFromProfileId` is provided AND the project has no spec content yet, COPY that profile's `spec.content` into the project's `spec.content` (a one-time snapshot). `project_save` runs `ProjectBase::resolve` to compute/validate the canonical path before persisting.

### Steps

1. - [ ] Add module wiring to `commands/mod.rs`: after `mod profile;` add `mod project;`, and after `pub use profile::*;` add `pub use project::*;`.

2. - [ ] Write the failing test (whole file with a test mod whose helpers call the same logic). Create `src-tauri/src/commands/project.rs` with imports + a test mod referencing yet-undefined free helpers, plus the seed-snapshot test:
   ```rust
   //! Project 命令层（增量 4a，Claude only）。
   //!
   //! 暴露项目绑定的 CRUD / set_enabled / apply / detach / manifest 读取。
   //! seed-from-profile 是一次性快照（COPY spec.content），非活链接——
   //! 之后编辑 profile 不会改变已绑定项目（决策 #3）。

   use crate::app_config::{ManifestEntry, Project, ProjectSpec};
   use crate::app_config::AppType;
   use crate::services::project_apply::{ProjectApplyResult, ProjectApplyService};
   use crate::services::project_paths::ProjectBase;
   use crate::store::AppState;
   use std::str::FromStr;
   use tauri::State;

   #[cfg(test)]
   mod tests {
       use super::*;
       use crate::app_config::{ProfileContent, ProfileSpec};
       use crate::database::Database;
       use serial_test::serial;
       use std::env;
       use std::sync::Arc;
       use tempfile::TempDir;

       struct TempHome { dir: TempDir, oh: Option<String>, ou: Option<String>, ot: Option<String> }
       impl TempHome {
           fn new() -> Self {
               let dir = TempDir::new().expect("tmp");
               let (oh, ou, ot) = (env::var("HOME").ok(), env::var("USERPROFILE").ok(), env::var("CC_SWITCH_TEST_HOME").ok());
               env::set_var("HOME", dir.path());
               env::set_var("USERPROFILE", dir.path());
               env::set_var("CC_SWITCH_TEST_HOME", dir.path());
               Self { dir, oh, ou, ot }
           }
       }
       impl Drop for TempHome {
           fn drop(&mut self) {
               match &self.oh { Some(v)=>env::set_var("HOME",v), None=>env::remove_var("HOME") }
               match &self.ou { Some(v)=>env::set_var("USERPROFILE",v), None=>env::remove_var("USERPROFILE") }
               match &self.ot { Some(v)=>env::set_var("CC_SWITCH_TEST_HOME",v), None=>env::remove_var("CC_SWITCH_TEST_HOME") }
           }
       }

       #[test]
       #[serial]
       fn save_seed_from_profile_is_a_snapshot_not_a_live_link() {
           let home = TempHome::new();
           crate::settings::reload_settings().ok();
           let db = Arc::new(Database::memory().expect("db"));
           let state = AppState::new(db.clone());

           // a profile whose content we will SEED from
           let prof = crate::app_config::Profile {
               id: "local:claude:Src".into(), app_type: "claude".into(), name: "Src".into(),
               description: None, is_active: false, current_provider_id: None,
               spec: ProfileSpec {
                   content: ProfileContent { skills: vec![], commands: vec!["seeded-cmd".into()], agents: vec![], mcp: vec![] },
                   vars: serde_json::Map::new(),
               },
               sort_index: 0, created_at: 0,
           };
           db.save_profile(&prof).expect("save profile");

           let root = home.dir.path().join("seedproj");
           std::fs::create_dir_all(&root).expect("mkdir");

           // create the project, seeding from the profile (one-time COPY)
           let created = run_save_logic(
               &state, None, "claude", root.to_str().unwrap(),
               Some("Seeded".into()), ProjectSpec::default(), Some("local:claude:Src".into()),
           ).expect("save");
           assert_eq!(created.spec.content.commands, vec!["seeded-cmd"], "seed copied profile content");

           // EDIT the profile afterwards — must NOT change the already-bound project
           let mut prof2 = db.get_profile("local:claude:Src").unwrap().unwrap();
           prof2.spec.content.commands = vec!["CHANGED".into()];
           db.save_profile(&prof2).expect("update profile");

           let reloaded = db.get_project(&created.id).unwrap().unwrap();
           assert_eq!(reloaded.spec.content.commands, vec!["seeded-cmd"], "project is a snapshot; profile edit must not leak in");
       }

       #[test]
       #[serial]
       fn save_refuses_home_collapse_path() {
           let home = TempHome::new();
           crate::settings::reload_settings().ok();
           let db = Arc::new(Database::memory().expect("db"));
           let state = AppState::new(db.clone());
           // binding $HOME must be refused by the gate inside project_save
           let err = run_save_logic(
               &state, None, "claude", home.dir.path().to_str().unwrap(),
               None, ProjectSpec::default(), None,
           );
           assert!(err.is_err(), "binding HOME must be refused at save time");
       }
   }
   ```

3. - [ ] Run it — expect FAIL: `run_save_logic` undefined.
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo test --lib save_seed_from_profile_is_a_snapshot_not_a_live_link save_refuses_home_collapse_path 2>&1 | tail -20`

4. - [ ] Add the commands + the shared `run_save_logic` helper to `project.rs` (between imports and the test mod). The `#[tauri::command]` fns delegate to `run_save_logic` so the logic is unit-testable without a Tauri runtime:
   ```rust
   /// List all projects (device-local).
   #[tauri::command]
   pub fn project_list(state: State<'_, AppState>) -> Result<Vec<Project>, String> {
       state.db.get_all_projects().map_err(|e| e.to_string())
   }

   /// Get a single project by id.
   #[tauri::command]
   pub fn project_get(id: String, state: State<'_, AppState>) -> Result<Option<Project>, String> {
       state.db.get_project(&id).map_err(|e| e.to_string())
   }

   /// Create or update a project binding. Runs the path-safety gate; optionally
   /// seeds spec.content from a profile as a ONE-TIME snapshot (not a live link).
   #[tauri::command]
   #[allow(clippy::too_many_arguments)]
   pub fn project_save(
       id: Option<String>,
       app: String,
       entered_path: String,
       name: Option<String>,
       spec: ProjectSpec,
       seed_from_profile_id: Option<String>,
       state: State<'_, AppState>,
   ) -> Result<Project, String> {
       run_save_logic(&state, id, &app, &entered_path, name, spec, seed_from_profile_id)
           .map_err(|e| e.to_string())
   }

   /// Delete a project. Detach FIRST (owned-delete its files + clear its manifest
   /// rows), THEN drop the projects row. project_id has NO FK CASCADE (design
   /// decision, Task 1), so deleting the row alone would orphan its manifest rows
   /// and leave its materialized files on disk; detach-then-delete avoids that.
   /// detach is idempotent and safe to run even if nothing was applied.
   #[tauri::command]
   pub fn project_delete(id: String, state: State<'_, AppState>) -> Result<bool, String> {
       // best-effort detach (owned-delete + clear rows) before removing the row;
       // ignore "project not found" so a never-applied project still deletes.
       if state.db.get_project(&id).map_err(|e| e.to_string())?.is_some() {
           ProjectApplyService::detach(&state, &id).map_err(|e| e.to_string())?;
       }
       state.db.delete_project(&id).map_err(|e| e.to_string())
   }

   /// Pause/resume a project (enabled flag; NOT an exclusive switch).
   #[tauri::command]
   pub fn project_set_enabled(
       id: String,
       enabled: bool,
       state: State<'_, AppState>,
   ) -> Result<bool, String> {
       let Some(mut p) = state.db.get_project(&id).map_err(|e| e.to_string())? else {
           return Ok(false);
       };
       p.enabled = enabled;
       p.updated_at = chrono::Utc::now().timestamp();
       state.db.save_project(&p).map_err(|e| e.to_string())?;
       Ok(true)
   }

   /// Apply a project's content set into <project>/.claude (Claude only).
   #[tauri::command]
   pub fn project_apply(
       id: String,
       state: State<'_, AppState>,
   ) -> Result<ProjectApplyResult, String> {
       ProjectApplyService::apply(&state, &id).map_err(|e| e.to_string())
   }

   /// Detach: owned-delete the project's materialized files, keep dirs.
   #[tauri::command]
   pub fn project_detach(
       id: String,
       state: State<'_, AppState>,
   ) -> Result<ProjectApplyResult, String> {
       ProjectApplyService::detach(&state, &id).map_err(|e| e.to_string())
   }

   /// Read the applied-files manifest for a project (its project channel).
   #[tauri::command]
   pub fn project_manifest(
       id: String,
       state: State<'_, AppState>,
   ) -> Result<Vec<ManifestEntry>, String> {
       let Some(p) = state.db.get_project(&id).map_err(|e| e.to_string())? else {
           return Ok(vec![]);
       };
       let channel = format!("project:{}", p.project_path);
       state
           .db
           .get_manifest_for_channel(&channel)
           .map_err(|e| e.to_string())
   }

   /// Shared, unit-testable save logic (bypasses the Tauri State wrapper).
   pub(crate) fn run_save_logic(
       state: &AppState,
       id: Option<String>,
       app: &str,
       entered_path: &str,
       name: Option<String>,
       mut spec: ProjectSpec,
       seed_from_profile_id: Option<String>,
   ) -> Result<Project, crate::error::AppError> {
       let app_type = AppType::from_str(app)?;
       // CRITICAL #1: validate + canonicalize at save time.
       let base = ProjectBase::resolve(entered_path, &app_type)?;
       let canon = base.root().to_string_lossy().to_string();

       // One-time seed-from-profile snapshot (decision #3): only when the project
       // has no own content yet AND a profile id is given. COPY spec.content.
       if let Some(pid) = seed_from_profile_id {
           let empty = spec.content.skills.is_empty()
               && spec.content.commands.is_empty()
               && spec.content.agents.is_empty()
               && spec.content.mcp.is_empty();
           if empty {
               if let Some(prof) = state.db.get_profile(&pid)? {
                   spec.content = prof.spec.content.clone(); // snapshot, not a live link
               }
           }
       }

       let now = chrono::Utc::now().timestamp();
       let (id, created_at) = match id.as_deref().and_then(|i| {
           state.db.get_project(i).ok().flatten().map(|p| (i.to_string(), p.created_at))
       }) {
           Some((existing_id, created)) => (existing_id, created),
           None => (id.unwrap_or_else(|| format!("proj:{}", uuid::Uuid::new_v4())), now),
       };

       let project = Project {
           id,
           project_path: canon,
           entered_path: entered_path.to_string(),
           app_type: app.to_string(),
           name,
           spec,
           enabled: true,
           created_at,
           updated_at: now,
       };
       state.db.save_project(&project)?;
       Ok(project)
   }
   ```

5. - [ ] Run the tests — expect PASS:
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo test --lib save_seed_from_profile_is_a_snapshot_not_a_live_link save_refuses_home_collapse_path 2>&1 | tail -12`
   Expected: `test result: ok. 2 passed`.

6. - [ ] clippy + fmt + commit:
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo clippy --all-targets -- -D warnings 2>&1 | tail -5; cargo fmt; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && git add -A && git commit -m "feat(projects): Tauri command layer (CRUD/apply/detach/manifest) + seed-from-profile snapshot"`

---

## Task 10b — Register the project commands in `generate_handler!` + extend crate-root re-exports

**Files:**
- Modify `src-tauri/src/lib.rs` (the `tauri::generate_handler!` list — insert after the Profile block at line 1284, after `commands::get_profile_manifest`; AND the crate-root `pub use` re-exports — `pub use app_config::{...}` at line 39 and the `pub use services::{...}` block at lines 54-60)

**Key facts verified:** lib.rs:1272-1284 registers `commands::get_profiles ... commands::get_profile_manifest`. The handler entries are `commands::<fn_name>`. `commands::*` re-exports come from `commands/mod.rs`. CRITICAL for Task 13: every source module is PRIVATE (`mod app_config;`, `mod database;`, `mod services;`, `mod store;` at lib.rs:1-37 — NOT `pub mod`), so the integration crate canNOT use module-qualified paths like `agenthub_lib::app_config::Project`. The crate exposes only FLAT re-exports (`pub use ...`). Currently re-exported and reusable as-is: `AppType`, `InstalledSkill`, `Database` (line 44), `AppState` (line 62), `AppError`. NOT yet re-exported (Task 13 needs them): `Project`, `ProjectSpec`, `ProfileContent`, `InstalledCommand`, `ProjectApplyService`, `ProjectBase`. This task adds those so Task 13's flat `use agenthub_lib::{...}` compiles. (Existing integration tests `tests/skill_sync.rs` / `tests/provider_service.rs` confirm the flat convention: `use agenthub_lib::{AppType, InstalledSkill, ...}`.)

### Steps

1. - [ ] Add the registration block after `commands::get_profile_manifest,` (lib.rs:1284):
   ```rust
               // Project bindings (increment 4a, device-local)
               commands::project_list,
               commands::project_get,
               commands::project_save,
               commands::project_delete,
               commands::project_set_enabled,
               commands::project_apply,
               commands::project_detach,
               commands::project_manifest,
   ```

2. - [ ] Extend the crate-root re-exports so the integration-test crate (Task 13) can reach the new types via the flat `agenthub_lib::{...}` path. In `src-tauri/src/lib.rs`, change the `app_config` re-export (line 39) to add `InstalledCommand`, `ProfileContent`, `Project`, `ProjectSpec`:
   ```rust
   pub use app_config::{
       AppType, InstalledCommand, InstalledSkill, McpApps, McpServer, MultiAppConfig, ProfileContent,
       Project, ProjectSpec, SkillApps,
   };
   ```
   Then extend the `services` re-export block (lines 54-60) to add the two project-service items, keeping the existing entries:
   ```rust
   pub use services::{
       agent::AgentService,
       command::CommandService,
       project_apply::ProjectApplyService,
       project_paths::ProjectBase,
       skill::{migrate_skills_to_ssot, ImportSkillSelection},
       ConfigService, EndpointLatency, McpService, PromptService, ProviderService, ProxyService,
       SkillService, SpeedtestService,
   };
   ```
   (NOTE: `InstalledCommand` / `ProfileContent` / `Project` / `ProjectSpec` are all defined in `app_config.rs`; if `InstalledCommand` turns out to live in a different module, `grep -rn "pub struct InstalledCommand" src-tauri/src` and re-export from wherever it is defined — but it is in `app_config.rs` alongside `InstalledSkill`.)

3. - [ ] Build to confirm the macro list AND the re-exports compile (no test yet — registration + re-export are compile-time concerns):
   `export PATH="$HOME/.cargo/bin:$PATH"; CARGO_NET_RETRY=10 cargo build 2>&1 | tail -10`
   Expected: `Finished` (no "cannot find function `project_list` in module `commands`", no "unresolved import" for the new `pub use` items).

4. - [ ] Confirm the integration-test crate compiles end-to-end with the new re-exports (RECURRING LESSON — `--lib` alone would not catch a missing re-export Task 13 needs):
   `export PATH="$HOME/.cargo/bin:$PATH"; CARGO_NET_RETRY=10 cargo test --no-run --all-targets 2>&1 | tail -10`
   Expected: `Finished` with no `unresolved import` / `is private` errors from the `tests/` crate.

5. - [ ] fmt + commit:
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo fmt; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && git add -A && git commit -m "feat(projects): register project_* commands + re-export Project/ProjectSpec/ProjectApplyService/ProjectBase"`

---

## Task 11 — WebDAV device-local assertion (projects NOT in sync whitelist)

**Files:**
- Modify `src-tauri/src/services/webdav_auto_sync.rs` (extend the existing test `should_trigger_sync_for_config_tables_only` at :212)

**Key facts verified:** `should_trigger_for_table` (webdav_auto_sync.rs:43) returns true for `profiles`/`profile_dotfiles` but the list does NOT contain `projects` (verified — current code at :45-59). `apply_manifest` already false, tested at :221. Decision #4: projects are DEVICE-LOCAL → must NOT trigger sync. We DO NOT modify `should_trigger_for_table` itself; we add an assertion locking the current correct behavior.

### Steps

1. - [ ] Add assertions to the existing test (webdav_auto_sync.rs:212). Insert before the closing `}` of `should_trigger_sync_for_config_tables_only`:
   ```rust
       // projects bindings are DEVICE-LOCAL (decision #4) — must NOT trigger WebDAV sync
       assert!(!should_trigger_for_table("projects"));
   ```

2. - [ ] Run it — it should PASS immediately (the behavior is already correct; this test LOCKS it):
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo test --lib should_trigger_sync_for_config_tables_only 2>&1 | tail -10`
   Expected: `test result: ok. 1 passed`.
   (If a future edit ever adds `projects` to the whitelist, this assertion fails — that is the guard.)

3. - [ ] fmt + commit:
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo fmt; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && git add -A && git commit -m "test(webdav): assert projects is device-local (not in sync whitelist)"`

---

## Task 12a — Frontend: `projectsApi` (api/projects.ts) + unit test

**Files:**
- Create `src/lib/api/projects.ts`
- Create `src/lib/api/projects.test.ts`

**Key facts verified:** `lib/api/profiles.ts` uses `invoke` from `@tauri-apps/api/core`, camelCase interfaces matching the Rust `#[serde(rename_all="camelCase")]`. `ManifestEntry` interface already exists in profiles.ts:36 (reuse the shape). Commands return raw types. Tests are vitest; mock `@tauri-apps/api/core`'s `invoke`.

### Steps

1. - [ ] Write the failing vitest test. Create `src/lib/api/projects.test.ts`:
   ```ts
   import { describe, it, expect, vi, beforeEach } from "vitest";

   const invokeMock = vi.fn();
   vi.mock("@tauri-apps/api/core", () => ({
     invoke: (...args: unknown[]) => invokeMock(...args),
   }));

   import { projectsApi } from "./projects";

   describe("projectsApi", () => {
     beforeEach(() => invokeMock.mockReset());

     it("save passes camelCase args including seedFromProfileId", async () => {
       invokeMock.mockResolvedValue({ id: "p1" });
       await projectsApi.save(null, "claude", "/abs/repo", "Repo", { content: { skills: [], commands: [], agents: [], mcp: [] }, vars: {} }, "local:claude:Src");
       expect(invokeMock).toHaveBeenCalledWith("project_save", {
         id: null,
         app: "claude",
         enteredPath: "/abs/repo",
         name: "Repo",
         spec: { content: { skills: [], commands: [], agents: [], mcp: [] }, vars: {} },
         seedFromProfileId: "local:claude:Src",
       });
     });

     it("apply/detach/list call the right commands", async () => {
       invokeMock.mockResolvedValue({ warnings: [] });
       await projectsApi.apply("p1");
       expect(invokeMock).toHaveBeenCalledWith("project_apply", { id: "p1" });
       await projectsApi.detach("p1");
       expect(invokeMock).toHaveBeenCalledWith("project_detach", { id: "p1" });
       invokeMock.mockResolvedValue([]);
       await projectsApi.list();
       expect(invokeMock).toHaveBeenCalledWith("project_list");
     });
   });
   ```

2. - [ ] Run it — expect FAIL (module not found):
   `cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && ./node_modules/.bin/vitest run src/lib/api/projects.test.ts 2>&1 | tail -15`
   Expected: `Failed to load url ./projects` / cannot find module.

3. - [ ] Create `src/lib/api/projects.ts`:
   ```ts
   import { invoke } from "@tauri-apps/api/core";
   import type { ManifestEntry } from "./profiles";

   export interface ProjectSpec {
     content: { skills: string[]; commands: string[]; agents: string[]; mcp: string[] };
     vars: Record<string, unknown>;
   }

   export interface Project {
     id: string;
     projectPath: string;
     enteredPath: string;
     appType: string;
     name?: string;
     spec: ProjectSpec;
     enabled: boolean;
     createdAt: number;
     updatedAt: number;
   }

   export interface ProjectApplyResult {
     warnings: string[];
   }

   export const projectsApi = {
     async list(): Promise<Project[]> {
       return await invoke("project_list");
     },
     async get(id: string): Promise<Project | null> {
       return await invoke("project_get", { id });
     },
     async save(
       id: string | null,
       app: string,
       enteredPath: string,
       name: string | null,
       spec: ProjectSpec,
       seedFromProfileId: string | null,
     ): Promise<Project> {
       return await invoke("project_save", {
         id,
         app,
         enteredPath,
         name,
         spec,
         seedFromProfileId,
       });
     },
     async delete(id: string): Promise<boolean> {
       return await invoke("project_delete", { id });
     },
     async setEnabled(id: string, enabled: boolean): Promise<boolean> {
       return await invoke("project_set_enabled", { id, enabled });
     },
     async apply(id: string): Promise<ProjectApplyResult> {
       return await invoke("project_apply", { id });
     },
     async detach(id: string): Promise<ProjectApplyResult> {
       return await invoke("project_detach", { id });
     },
     async manifest(id: string): Promise<ManifestEntry[]> {
       return await invoke("project_manifest", { id });
     },
   };
   ```

4. - [ ] Run it — expect PASS:
   `cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && ./node_modules/.bin/vitest run src/lib/api/projects.test.ts 2>&1 | tail -10`
   Expected: `2 passed`.

5. - [ ] tsc + commit:
   `cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && ./node_modules/.bin/tsc --noEmit 2>&1 | tail -5 && git add -A && git commit -m "feat(ui): projectsApi + unit test"`

---

## Task 12b — Frontend: `useProjects` hook

**Files:**
- Create `src/hooks/useProjects.ts`

**Key facts verified:** `useProfiles.ts` uses `@tanstack/react-query` (`useQuery`/`useMutation`/`useQueryClient`/`keepPreviousData`), `staleTime: Infinity`, invalidation on mutation success. Mirror that shape.

### Steps

1. - [ ] Create `src/hooks/useProjects.ts`:
   ```ts
   import {
     useMutation,
     useQuery,
     useQueryClient,
     keepPreviousData,
   } from "@tanstack/react-query";
   import { projectsApi, type Project, type ProjectSpec } from "@/lib/api/projects";

   export function useProjects() {
     return useQuery({
       queryKey: ["projects", "all"],
       queryFn: () => projectsApi.list(),
       staleTime: Infinity,
       placeholderData: keepPreviousData,
     });
   }

   export function useSaveProject() {
     const qc = useQueryClient();
     return useMutation({
       mutationFn: (args: {
         id: string | null;
         app: string;
         enteredPath: string;
         name: string | null;
         spec: ProjectSpec;
         seedFromProfileId: string | null;
       }) =>
         projectsApi.save(
           args.id,
           args.app,
           args.enteredPath,
           args.name,
           args.spec,
           args.seedFromProfileId,
         ),
       onSuccess: () => {
         void qc.invalidateQueries({ queryKey: ["projects", "all"] });
       },
     });
   }

   export function useDeleteProject() {
     const qc = useQueryClient();
     return useMutation({
       mutationFn: (id: string) => projectsApi.delete(id),
       onSuccess: (_r, id) => {
         qc.setQueryData<Project[]>(["projects", "all"], (old) =>
           old ? old.filter((p) => p.id !== id) : old,
         );
       },
     });
   }

   export function useSetProjectEnabled() {
     const qc = useQueryClient();
     return useMutation({
       mutationFn: (a: { id: string; enabled: boolean }) =>
         projectsApi.setEnabled(a.id, a.enabled),
       onSuccess: () => {
         void qc.invalidateQueries({ queryKey: ["projects", "all"] });
       },
     });
   }

   export function useApplyProject() {
     const qc = useQueryClient();
     return useMutation({
       mutationFn: (id: string) => projectsApi.apply(id),
       onSuccess: (_r, id) => {
         void qc.invalidateQueries({ queryKey: ["projects", "manifest", id] });
       },
     });
   }

   export function useDetachProject() {
     const qc = useQueryClient();
     return useMutation({
       mutationFn: (id: string) => projectsApi.detach(id),
       onSuccess: (_r, id) => {
         void qc.invalidateQueries({ queryKey: ["projects", "manifest", id] });
       },
     });
   }

   export function useProjectManifest(id: string | undefined) {
     return useQuery({
       queryKey: ["projects", "manifest", id],
       queryFn: () => projectsApi.manifest(id!),
       staleTime: Infinity,
       placeholderData: keepPreviousData,
       enabled: Boolean(id),
     });
   }

   export type { Project, ProjectSpec } from "@/lib/api/projects";
   ```

2. - [ ] Verify it typechecks (no dedicated unit test for the hook — covered by tsc):
   `cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && ./node_modules/.bin/tsc --noEmit 2>&1 | tail -8`
   Expected: no errors referencing `useProjects.ts`.

3. - [ ] commit:
   `cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && git add -A && git commit -m "feat(ui): useProjects hook (list/save/delete/enable/apply/detach/manifest)"`

---

## Task 12c — Frontend: `ProjectsPanel` (list + per-project Apply/Detach/enable + manifest view)

**Files:**
- Create `src/components/projects/ProjectsPanel.tsx`

**Key facts verified:** `ProfilesPanel.tsx` is a `React.forwardRef<ProfilesPanelHandle, Props>` exposing `openCreate()` via `useImperativeHandle`, renders a list of `ListItemRow`, uses `ConfirmDialog`, `toast` from `sonner`, `useTranslation`. App.tsx calls `profilesPanelRef.current?.openCreate()` and renders `<ProfilesPanel ref={...} currentApp="claude" />`. We mirror this for projects.

### Steps

1. - [ ] Create `src/components/projects/ProjectsPanel.tsx`:
   ```tsx
   import React, { useState } from "react";
   import { useTranslation } from "react-i18next";
   import { FolderGit2, Trash2, Pencil, Link2, Link2Off, Pause, Play } from "lucide-react";
   import { Button } from "@/components/ui/button";
   import { TooltipProvider } from "@/components/ui/tooltip";
   import { ConfirmDialog } from "@/components/ConfirmDialog";
   import { ListItemRow } from "@/components/common/ListItemRow";
   import {
     useProjects,
     useDeleteProject,
     useSetProjectEnabled,
     useApplyProject,
     useDetachProject,
   } from "@/hooks/useProjects";
   import type { Project } from "@/lib/api/projects";
   import { toast } from "sonner";
   import { ProjectBindDialog } from "./ProjectBindDialog";

   export interface ProjectsPanelHandle {
     openCreate(): void;
   }
   interface ProjectsPanelProps {
     currentApp?: string;
   }

   const ProjectsPanel = React.forwardRef<ProjectsPanelHandle, ProjectsPanelProps>(
     (props, ref) => {
       const { t } = useTranslation();
       const { currentApp } = props;
       const { data: projects, isLoading } = useProjects();
       const del = useDeleteProject();
       const setEnabled = useSetProjectEnabled();
       const applyM = useApplyProject();
       const detachM = useDetachProject();

       const [dialogOpen, setDialogOpen] = useState(false);
       const [editing, setEditing] = useState<Project | null>(null);
       const [confirm, setConfirm] = useState<{ title: string; message: string; onConfirm: () => void } | null>(null);

       React.useImperativeHandle(ref, () => ({
         openCreate() {
           setEditing(null);
           setDialogOpen(true);
         },
       }));

       const onApply = async (p: Project) => {
         try {
           const r = await applyM.mutateAsync(p.id);
           if (r.warnings.length) r.warnings.forEach((w) => toast.warning(w));
           else toast.success(t("projects.applySuccess", { name: p.name ?? p.enteredPath }));
         } catch (e) {
           toast.error(t("common.error"), { description: String(e) });
         }
       };
       const onDetach = async (p: Project) => {
         try {
           await detachM.mutateAsync(p.id);
           toast.success(t("projects.detachSuccess", { name: p.name ?? p.enteredPath }));
         } catch (e) {
           toast.error(t("common.error"), { description: String(e) });
         }
       };

       return (
         <div className="px-6 flex flex-col flex-1 min-h-0 overflow-hidden">
           <div className="flex items-center justify-between py-2">
             <span className="text-sm text-muted-foreground">
               {t("projects.count", { count: projects?.length ?? 0 })}
             </span>
           </div>
           <div className="flex-1 overflow-y-auto overflow-x-hidden pb-24">
             {isLoading ? (
               <div className="text-center py-12 text-muted-foreground">{t("common.loading")}</div>
             ) : !projects || projects.length === 0 ? (
               <div className="text-center py-12">
                 <div className="w-16 h-16 mx-auto mb-4 bg-muted rounded-full flex items-center justify-center">
                   <FolderGit2 size={24} className="text-muted-foreground" />
                 </div>
                 <h3 className="text-lg font-medium text-foreground mb-2">{t("projects.empty")}</h3>
                 <p className="text-muted-foreground text-sm">{t("projects.emptyDescription")}</p>
               </div>
             ) : (
               <TooltipProvider delayDuration={300}>
                 <div className="rounded-xl border border-border-default overflow-hidden">
                   {projects.map((p, i) => (
                     <ListItemRow key={p.id} isLast={i === projects.length - 1}>
                       <div className="flex-1 min-w-0">
                         <span className="font-medium text-sm text-foreground truncate block">
                           {p.name ?? p.enteredPath}
                         </span>
                         <p className="text-xs text-muted-foreground truncate" title={p.projectPath}>
                           {p.enteredPath}
                         </p>
                       </div>
                       <div className="flex-shrink-0 flex items-center gap-2">
                         <Button variant="outline" size="sm" className="h-7 text-xs" onClick={() => void onApply(p)} disabled={!p.enabled} title={t("projects.apply")}>
                           <Link2 size={14} className="mr-1" /> {t("projects.apply")}
                         </Button>
                         <Button variant="outline" size="sm" className="h-7 text-xs" onClick={() => void onDetach(p)} title={t("projects.detach")}>
                           <Link2Off size={14} className="mr-1" /> {t("projects.detach")}
                         </Button>
                         <Button variant="ghost" size="icon" className="h-7 w-7" onClick={() => void setEnabled.mutateAsync({ id: p.id, enabled: !p.enabled })} title={p.enabled ? t("projects.pause") : t("projects.resume")}>
                           {p.enabled ? <Pause size={14} /> : <Play size={14} />}
                         </Button>
                         <Button variant="ghost" size="icon" className="h-7 w-7" onClick={() => { setEditing(p); setDialogOpen(true); }} title={t("projects.edit")}>
                           <Pencil size={14} />
                         </Button>
                         <Button variant="ghost" size="icon" className="h-7 w-7 hover:text-red-500" onClick={() => setConfirm({ title: t("projects.delete"), message: t("projects.deleteConfirmDescription", { name: p.name ?? p.enteredPath }), onConfirm: async () => { await del.mutateAsync(p.id); setConfirm(null); toast.success(t("projects.deleteSuccess", { name: p.name ?? p.enteredPath })); } })} title={t("projects.delete")}>
                           <Trash2 size={14} />
                         </Button>
                       </div>
                     </ListItemRow>
                   ))}
                 </div>
               </TooltipProvider>
             )}
           </div>
           {confirm && (
             <ConfirmDialog isOpen title={confirm.title} message={confirm.message} variant="destructive" zIndex="top" onConfirm={() => void confirm.onConfirm()} onCancel={() => setConfirm(null)} />
           )}
           <ProjectBindDialog open={dialogOpen} project={editing} currentApp={currentApp ?? "claude"} onClose={() => { setDialogOpen(false); setEditing(null); }} />
         </div>
       );
     },
   );
   ProjectsPanel.displayName = "ProjectsPanel";
   export default ProjectsPanel;
   ```

2. - [ ] tsc (will FAIL until ProjectBindDialog exists — that is Task 12d; this is expected and resolved next):
   `cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && ./node_modules/.bin/tsc --noEmit 2>&1 | grep -i "ProjectBindDialog\|ProjectsPanel" | head`
   Expected: only `Cannot find module './ProjectBindDialog'` — resolved in 12d. Do NOT commit until 12d typechecks clean.

---

## Task 12d — Frontend: `ProjectBindDialog` (directory picker + seed-from-profile + includes editor + manifest view)

**Files:**
- Create `src/components/projects/ProjectBindDialog.tsx`

**Key facts verified:** `@tauri-apps/plugin-dialog` exports `open({ directory: true })` (dist-js types confirm `directory?: boolean`); the Cargo `tauri-plugin-dialog` is registered in lib.rs:279. `ProfileEditDialog.tsx` is the structural template (form state, save mutation, comma-separated content fields). Includes are comma-separated literal/`@tag` strings parsed into `string[]` (mirror profiles `commandsPlaceholder` "e.g. my-command, @backend"). `useProfiles` (`useInstalledProfiles`) provides the seed-from-profile dropdown source. `useProjectManifest` renders the applied-files list.

### Steps

1. - [ ] Create `src/components/projects/ProjectBindDialog.tsx`:
   ```tsx
   import React, { useEffect, useState } from "react";
   import { useTranslation } from "react-i18next";
   import { open as openDialog } from "@tauri-apps/plugin-dialog";
   import { Button } from "@/components/ui/button";
   import { Input } from "@/components/ui/input";
   import { toast } from "sonner";
   import { useSaveProject, useProjectManifest } from "@/hooks/useProjects";
   import { useInstalledProfiles } from "@/hooks/useProfiles";
   import type { Project, ProjectSpec } from "@/lib/api/projects";

   interface Props {
     open: boolean;
     project: Project | null;
     currentApp: string;
     onClose: () => void;
   }

   const splitList = (s: string): string[] =>
     s.split(",").map((x) => x.trim()).filter(Boolean);

   export const ProjectBindDialog: React.FC<Props> = ({ open, project, currentApp, onClose }) => {
     const { t } = useTranslation();
     const save = useSaveProject();
     const { data: profiles } = useInstalledProfiles();
     const { data: manifest } = useProjectManifest(project?.id);

     const [path, setPath] = useState("");
     const [name, setName] = useState("");
     const [skills, setSkills] = useState("");
     const [commands, setCommands] = useState("");
     const [agents, setAgents] = useState("");
     const [seedId, setSeedId] = useState<string>("");

     useEffect(() => {
       if (!open) return;
       setPath(project?.enteredPath ?? "");
       setName(project?.name ?? "");
       setSkills((project?.spec.content.skills ?? []).join(", "));
       setCommands((project?.spec.content.commands ?? []).join(", "));
       setAgents((project?.spec.content.agents ?? []).join(", "));
       setSeedId("");
     }, [open, project]);

     if (!open) return null;

     const pickDir = async () => {
       const sel = await openDialog({ directory: true, multiple: false });
       if (typeof sel === "string") setPath(sel);
     };

     const onSave = async () => {
       const spec: ProjectSpec = {
         content: {
           skills: splitList(skills),
           commands: splitList(commands),
           agents: splitList(agents),
           mcp: [],
         },
         vars: {},
       };
       try {
         await save.mutateAsync({
           id: project?.id ?? null,
           app: currentApp,
           enteredPath: path,
           name: name || null,
           spec,
           seedFromProfileId: seedId || null,
         });
         toast.success(t(project ? "projects.updateSuccess" : "projects.createSuccess", { name: name || path }));
         onClose();
       } catch (e) {
         toast.error(t("common.error"), { description: String(e) });
       }
     };

     return (
       <div className="fixed inset-0 z-50 flex items-center justify-center bg-black/40" onClick={onClose}>
         <div className="bg-background rounded-xl p-6 w-[560px] max-h-[80vh] overflow-y-auto" onClick={(e) => e.stopPropagation()}>
           <h2 className="text-lg font-semibold mb-4">{t(project ? "projects.edit" : "projects.create")}</h2>

           <label className="text-sm font-medium">{t("projects.directory")}</label>
           <div className="flex gap-2 mt-1 mb-3">
             <Input value={path} onChange={(e) => setPath(e.target.value)} placeholder={t("projects.directoryPlaceholder")} />
             <Button variant="outline" onClick={() => void pickDir()}>{t("projects.browse")}</Button>
           </div>

           <label className="text-sm font-medium">{t("projects.name")}</label>
           <Input className="mt-1 mb-3" value={name} onChange={(e) => setName(e.target.value)} placeholder={t("projects.namePlaceholder")} />

           {!project && (
             <>
               <label className="text-sm font-medium">{t("projects.seedFromProfile")}</label>
               <select className="mt-1 mb-1 w-full rounded-md border border-border-default bg-background p-2 text-sm" value={seedId} onChange={(e) => setSeedId(e.target.value)}>
                 <option value="">{t("projects.seedNone")}</option>
                 {(profiles ?? []).filter((p) => p.appType === currentApp).map((p) => (
                   <option key={p.id} value={p.id}>{p.name}</option>
                 ))}
               </select>
               <p className="text-xs text-muted-foreground mb-3">{t("projects.seedHint")}</p>
             </>
           )}

           <label className="text-sm font-medium">{t("projects.skills")}</label>
           <Input className="mt-1 mb-3" value={skills} onChange={(e) => setSkills(e.target.value)} placeholder={t("projects.skillsPlaceholder")} />
           <label className="text-sm font-medium">{t("projects.commands")}</label>
           <Input className="mt-1 mb-3" value={commands} onChange={(e) => setCommands(e.target.value)} placeholder={t("projects.commandsPlaceholder")} />
           <label className="text-sm font-medium">{t("projects.agents")}</label>
           <Input className="mt-1 mb-3" value={agents} onChange={(e) => setAgents(e.target.value)} placeholder={t("projects.agentsPlaceholder")} />

           {project && (manifest?.length ?? 0) > 0 && (
             <div className="mb-3">
               <label className="text-sm font-medium">{t("projects.appliedFiles")}</label>
               <ul className="mt-1 text-xs text-muted-foreground max-h-32 overflow-y-auto">
                 {manifest!.map((m) => (
                   <li key={m.id} className="truncate" title={m.targetPath}>{m.kind}: {m.targetPath}</li>
                 ))}
               </ul>
             </div>
           )}

           <div className="flex justify-end gap-2 mt-4">
             <Button variant="ghost" onClick={onClose}>{t("projects.cancel")}</Button>
             <Button onClick={() => void onSave()} disabled={!path}>{t("projects.save")}</Button>
           </div>
         </div>
       </div>
     );
   };
   ```
   NOTE: if `@/components/ui/input` does not exist, use the input used in `ProfileEditDialog.tsx` (read its imports and copy the exact import path). Verify `lucide-react` icon names (`FolderGit2`, `Link2`, `Link2Off`, `Pause`, `Play`) exist — if any is missing, substitute a present icon (e.g. `Plug`/`Unplug`/`Folder`).

2. - [ ] tsc — expect PASS (ProjectsPanel now resolves):
   `cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && ./node_modules/.bin/tsc --noEmit 2>&1 | tail -8`
   Expected: no errors in `projects/*.tsx`.

3. - [ ] commit:
   `cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && git add -A && git commit -m "feat(ui): ProjectsPanel + ProjectBindDialog (dir picker, seed-from-profile, includes, manifest view)"`

---

## Task 12e — Frontend: App.tsx wiring (View union, VALID_VIEWS, renderContent, toolbar, header, nav)

**Files:**
- Modify `src/App.tsx` (View union :98-114; VALID_VIEWS :145-162; import ~:96; panelRef ~:258; renderContent switch ~:932; header title ~:1157; toolbar ~:1261; nav ~:1568)

**Key facts verified:** App.tsx already imports `ProfilesPanel` (:96), has `profilesPanelRef` (:258), `case "profiles"` (:932), header `{currentView === "profiles" && t("profiles.title")}` (:1157), a toolbar create button (:1261), and a nav button (:1568). We add a `projects` view that mirrors EVERY one of these. Reuse the `Layers`/`FolderGit2` icon already imported or import `FolderGit2`.

### Steps

1. - [ ] Add import after the ProfilesPanel import (App.tsx:96):
   ```tsx
   import ProjectsPanel from "@/components/projects/ProjectsPanel";
   ```

2. - [ ] Add `| "projects"` to the View union (App.tsx:114, after `| "profiles"`):
   ```tsx
     | "projects";
   ```
   and to `VALID_VIEWS` (App.tsx:161, after `"profiles",`):
   ```tsx
     "projects",
   ```

3. - [ ] Add the panel ref near `profilesPanelRef` (App.tsx:258):
   ```tsx
     const projectsPanelRef = useRef<any>(null);
   ```

4. - [ ] Add the renderContent case after `case "profiles":` (App.tsx:933):
   ```tsx
           case "projects":
             return <ProjectsPanel ref={projectsPanelRef} currentApp="claude" />;
   ```

5. - [ ] Add the header title after the profiles title (App.tsx:1157):
   ```tsx
                     {currentView === "projects" && t("projects.title")}
   ```

6. - [ ] Add the toolbar create button after the profiles block (App.tsx:1271):
   ```tsx
                   {currentView === "projects" && (
                     <Button
                       variant="ghost"
                       size="sm"
                       onClick={() => projectsPanelRef.current?.openCreate()}
                       className="hover:bg-black/5 dark:hover:bg-white/5"
                     >
                       <Plus className="w-4 h-4 mr-2" />
                       {t("projects.create")}
                     </Button>
                   )}
   ```

7. - [ ] Add the nav button after the profiles nav button (App.tsx:1573, after the `</Button>` that closes the profiles nav button, before the `</>`). Import `FolderGit2` from lucide-react at the top alongside the existing `Layers` import, then:
   ```tsx
                                 <Button
                                   variant="ghost"
                                   size="sm"
                                   onClick={() => setCurrentView("projects")}
                                   className="text-muted-foreground hover:text-foreground hover:bg-black/5 dark:hover:bg-white/5 w-8 px-2"
                                   title={t("projects.manage")}
                                 >
                                   <FolderGit2 className="w-4 h-4" />
                                 </Button>
   ```

8. - [ ] tsc — expect PASS:
   `cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && ./node_modules/.bin/tsc --noEmit 2>&1 | tail -8`
   Expected: no errors.

9. - [ ] commit:
   `cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && git add -A && git commit -m "feat(ui): wire Projects view into App (union, nav, toolbar, header, renderContent)"`

---

## Task 12f — Frontend: i18n `projects` namespace in all 4 locales

**Files:**
- Modify `src/i18n/locales/en.json`, `zh.json`, `zh-TW.json`, `ja.json` (add a `projects` object alongside the existing `profiles` object)

**Key facts verified:** All 4 locales have a `profiles` object (verified: en/zh/zh-TW/ja all have `profiles.title`). The `projects` namespace does NOT yet exist. Keys referenced by the components above: title, manage, count, create, edit, delete, deleteConfirmDescription, deleteSuccess, createSuccess, updateSuccess, empty, emptyDescription, apply, applySuccess, detach, detachSuccess, pause, resume, directory, directoryPlaceholder, browse, name, namePlaceholder, seedFromProfile, seedNone, seedHint, skills/commands/agents (+Placeholder), appliedFiles, cancel, save.

### Steps

1. - [ ] Add the `projects` object to `src/i18n/locales/en.json` (insert as a sibling of `profiles`). Use a unique top-level key:
   ```json
   "projects": {
     "title": "Projects",
     "manage": "Projects",
     "count": "{{count}} project(s)",
     "create": "Bind Project",
     "edit": "Edit Project",
     "delete": "Unbind Project",
     "deleteConfirmDescription": "Unbind project \"{{name}}\"? Materialized files are not removed by unbinding; use Detach first if you want them cleaned.",
     "deleteSuccess": "Project \"{{name}}\" unbound",
     "createSuccess": "Project \"{{name}}\" bound",
     "updateSuccess": "Project \"{{name}}\" updated",
     "empty": "No projects bound yet",
     "emptyDescription": "Bind a directory to materialize Claude skills/commands/agents into its .claude folder",
     "apply": "Apply",
     "applySuccess": "Applied to \"{{name}}\"",
     "detach": "Detach",
     "detachSuccess": "Detached \"{{name}}\"",
     "pause": "Pause",
     "resume": "Resume",
     "directory": "Project Directory",
     "directoryPlaceholder": "/absolute/path/to/your/project",
     "browse": "Browse",
     "name": "Display Name",
     "namePlaceholder": "Optional friendly name",
     "seedFromProfile": "Seed from Profile (one-time copy)",
     "seedNone": "None — start empty",
     "seedHint": "Copies the profile's content into this project now. Editing the profile later will NOT change this project.",
     "skills": "Skills",
     "skillsPlaceholder": "e.g. my-skill, @backend (comma-separated)",
     "commands": "Commands",
     "commandsPlaceholder": "e.g. my-command, @backend (comma-separated)",
     "agents": "Agents",
     "agentsPlaceholder": "e.g. my-agent, @backend (comma-separated)",
     "appliedFiles": "Applied Files",
     "cancel": "Cancel",
     "save": "Save"
   }
   ```

2. - [ ] Add the same object with translated values to `zh.json`, `zh-TW.json`, `ja.json` (translate the string values; keep the keys identical). Example for `zh.json`:
   ```json
   "projects": {
     "title": "项目",
     "manage": "项目",
     "count": "{{count}} 个项目",
     "create": "绑定项目",
     "edit": "编辑项目",
     "delete": "解绑项目",
     "deleteConfirmDescription": "解绑项目“{{name}}”？解绑不会删除已物化的文件；如需清理请先点击“分离”。",
     "deleteSuccess": "已解绑项目“{{name}}”",
     "createSuccess": "已绑定项目“{{name}}”",
     "updateSuccess": "已更新项目“{{name}}”",
     "empty": "尚未绑定任何项目",
     "emptyDescription": "绑定一个目录，将 Claude 技能/命令/子代理物化到其 .claude 目录",
     "apply": "应用",
     "applySuccess": "已应用到“{{name}}”",
     "detach": "分离",
     "detachSuccess": "已分离“{{name}}”",
     "pause": "暂停",
     "resume": "恢复",
     "directory": "项目目录",
     "directoryPlaceholder": "/项目的绝对路径",
     "browse": "浏览",
     "name": "显示名称",
     "namePlaceholder": "可选的友好名称",
     "seedFromProfile": "从配置文件初始化（一次性复制）",
     "seedNone": "无 — 从空开始",
     "seedHint": "立即将该配置文件的内容复制到此项目。之后编辑配置文件不会改变此项目。",
     "skills": "技能",
     "skillsPlaceholder": "例如 my-skill, @backend（逗号分隔）",
     "commands": "命令",
     "commandsPlaceholder": "例如 my-command, @backend（逗号分隔）",
     "agents": "子代理",
     "agentsPlaceholder": "例如 my-agent, @backend（逗号分隔）",
     "appliedFiles": "已应用文件",
     "cancel": "取消",
     "save": "保存"
   }
   ```
   (Provide the analogous `zh-TW` traditional and `ja` Japanese variants.)

3. - [ ] Validate each JSON parses and has the new key:
   `cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && for f in en zh zh-TW ja; do node -e "const j=require('./src/i18n/locales/$f.json'); if(!j.projects||!j.projects.title){console.error('MISSING projects in $f'); process.exit(1)} console.log('$f ok:', j.projects.title)"; done`
   Expected: 4 `ok:` lines, no MISSING.

4. - [ ] Full frontend gate — tsc + vitest:
   `cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && ./node_modules/.bin/tsc --noEmit 2>&1 | tail -5 && ./node_modules/.bin/vitest run 2>&1 | tail -10`
   Expected: tsc no errors; vitest all files pass (incl. projects.test.ts).

5. - [ ] commit:
   `cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && git add -A && git commit -m "feat(ui): projects i18n in en/zh/zh-TW/ja"`

---

## Task 13 — End-to-end integration test + full-suite verification + manual smoke

**Files:**
- Create `src-tauri/tests/project_apply_e2e.rs` (integration-test crate)
- (verification only — no further source changes expected)

**Key facts verified:** Integration tests live in `src-tauri/tests/` as a separate crate (e.g. `skill_sync.rs`), use the public `agenthub_lib` API. They must be in scope for `--all-targets`/full `cargo test`. The e2e test wires DB + AppState + a real tempdir project and asserts the full apply→detach lifecycle, including the HOME-collapse refusal and user-file safety.

### Steps

1. - [ ] Write the integration test. Create `src-tauri/tests/project_apply_e2e.rs`:
   ```rust
   //! E2E: project bind → apply → detach lifecycle, incl. HOME-collapse refusal
   //! and user-file safety. Uses the public agenthub_lib API.

   use std::sync::Arc;

   // FLAT crate-root re-exports only — every source module in lib.rs is PRIVATE
   // (`mod app_config;` etc.), so module-qualified paths like
   // `agenthub_lib::app_config::Project` do NOT compile. Task 10b step 2 added the
   // Project/ProjectSpec/ProfileContent/InstalledCommand/ProjectApplyService/ProjectBase
   // re-exports; Database + AppState + AppType were already re-exported.
   use agenthub_lib::{
       AppState, AppType, Database, InstalledCommand, ProfileContent, Project, ProjectApplyService,
       ProjectBase, ProjectSpec,
   };
   use serial_test::serial;
   use tempfile::TempDir;

   struct TempHome { dir: TempDir, oh: Option<String>, ot: Option<String> }
   impl TempHome {
       fn new() -> Self {
           let dir = TempDir::new().unwrap();
           let oh = std::env::var("HOME").ok();
           let ot = std::env::var("CC_SWITCH_TEST_HOME").ok();
           std::env::set_var("HOME", dir.path());
           std::env::set_var("CC_SWITCH_TEST_HOME", dir.path());
           Self { dir, oh, ot }
       }
   }
   impl Drop for TempHome {
       fn drop(&mut self) {
           match &self.oh { Some(v)=>std::env::set_var("HOME",v), None=>std::env::remove_var("HOME") }
           match &self.ot { Some(v)=>std::env::set_var("CC_SWITCH_TEST_HOME",v), None=>std::env::remove_var("CC_SWITCH_TEST_HOME") }
       }
   }

   #[test]
   #[serial]
   fn project_apply_detach_lifecycle() {
       let home = TempHome::new();
       let db = Arc::new(Database::memory().unwrap());
       let state = AppState::new(db.clone());

       db.save_command(&InstalledCommand {
           id: "local:e2e-cmd".into(), name: "e2e-cmd".into(), content: "E2E".into(),
           description: None, tags: vec![], enabled_claude: false, installed_at: 0,
       }).unwrap();

       let root = home.dir.path().join("e2e-repo");
       std::fs::create_dir_all(&root).unwrap();
       let canon = root.canonicalize().unwrap();
       let proj = Project {
           id: "proj:e2e".into(),
           project_path: canon.to_string_lossy().to_string(),
           entered_path: root.to_string_lossy().to_string(),
           app_type: "claude".into(),
           name: Some("E2E".into()),
           spec: ProjectSpec { content: ProfileContent { skills: vec![], commands: vec!["e2e-cmd".into()], agents: vec![], mcp: vec![] }, vars: serde_json::Map::new() },
           enabled: true, created_at: 0, updated_at: 0,
       };
       db.save_project(&proj).unwrap();

       // HOME-collapse refusal: binding $HOME is rejected.
       assert!(ProjectBase::resolve(home.dir.path().to_str().unwrap(), &AppType::Claude).is_err());

       // apply → file exists + manifest recorded
       ProjectApplyService::apply(&state, "proj:e2e").unwrap();
       let f = canon.join(".claude").join("commands").join("e2e-cmd.md");
       assert_eq!(std::fs::read_to_string(&f).unwrap(), "E2E");

       // user file in same dir survives detach
       let user = canon.join(".claude").join("commands").join("mine.md");
       std::fs::write(&user, "MINE").unwrap();

       ProjectApplyService::detach(&state, "proj:e2e").unwrap();
       assert!(!f.exists(), "owned file removed");
       assert!(user.is_file(), "user file untouched");
   }
   ```
   NOTE: the crate name is `agenthub_lib` (verified: `[lib] name = "agenthub_lib"` in `src-tauri/Cargo.toml`; other integration tests `tests/skill_sync.rs` / `tests/provider_service.rs` import via the flat `use agenthub_lib::{...}` form). All nine symbols above are FLAT crate-root re-exports: `AppState`/`AppType`/`Database` were already re-exported, and `InstalledCommand`/`ProfileContent`/`Project`/`ProjectSpec`/`ProjectApplyService`/`ProjectBase` were added in Task 10b step 2. Do NOT use module-qualified paths (`agenthub_lib::app_config::...`) — the source modules are private (`mod app_config;` not `pub mod`), so those would fail with "module `app_config` is private". If any symbol fails to resolve, the fix is to add it to the `pub use` lines in lib.rs (Task 10b step 2), NOT to make the module `pub`.

2. - [ ] Run the e2e test — expect PASS:
   `export PATH="$HOME/.cargo/bin:$PATH"; CARGO_NET_RETRY=10 cargo test --test project_apply_e2e 2>&1 | tail -12`
   Expected: `test result: ok. 1 passed`.

3. - [ ] FULL Rust suite (lib + integration crate):
   `export PATH="$HOME/.cargo/bin:$PATH"; CARGO_NET_RETRY=10 cargo test 2>&1 | tail -20`
   Expected: all test binaries `ok`, 0 failed.

4. - [ ] clippy `--all-targets -D warnings`:
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo clippy --all-targets -- -D warnings 2>&1 | tail -8`
   Expected: `Finished`, no warnings.

5. - [ ] fmt check:
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo fmt --check 2>&1 | tail -5`
   Expected: no output (all formatted). If it lists files, run `cargo fmt` and re-commit.

6. - [ ] Frontend full gate:
   `cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && ./node_modules/.bin/tsc --noEmit 2>&1 | tail -5 && ./node_modules/.bin/vitest run 2>&1 | tail -10`
   Expected: tsc clean; vitest all pass.

7. - [ ] Manual smoke (record outcome; do not block on it if dev env unavailable). Build the frontend bundle then run tauri dev:
   `cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && ./node_modules/.bin/vite build 2>&1 | tail -3; export PATH="$HOME/.cargo/bin:$PATH"; cargo tauri dev`
   In the app: open the Projects view → Bind Project → Browse to a throwaway dir → set Commands to an installed command → Save → Apply. Verify the file appears at `<project>/.claude/commands/<name>.md`. Manually create `<project>/.claude/commands/mine.md` with your own text. Click Detach. Verify the owned file is gone and `mine.md` is untouched and `<project>/.claude/commands/` still exists. Attempt to bind `$HOME` → expect a refusal toast (HOME-safety gate).

8. - [ ] Final commit (e2e test + any fmt fixups):
   `export PATH="$HOME/.cargo/bin:$PATH"; cargo fmt; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && git add -A && git commit -m "test(projects): e2e apply/detach lifecycle + HOME-collapse refusal + full-suite green"`

---

## Verification checklist (must all be green before declaring 4a done)

- [ ] `cargo test` (full, NOT --lib) — 0 failed.
- [ ] `cargo clippy --all-targets -- -D warnings` — clean.
- [ ] `cargo fmt --check` — clean.
- [ ] `./node_modules/.bin/tsc --noEmit` — clean.
- [ ] `./node_modules/.bin/vitest run` — all pass.
- [ ] Path-safety gate has explicit failing-then-passing tests for: HOME-itself, HOME-ancestor, claude-dir, tool-dirs (.codex/.gemini/.config/opencode), .agenthub, "/", descendant-of-.claude, and `<root>/.claude → ~/.claude` symlink collapse.
- [ ] Path-reuse: `apply_does_not_reconcile_foreign_project_rows` proves a foreign-`project_id` snapshot row is skipped+warned and its file survives.
- [ ] Path-gone prune wired to production (brief §3.2): `apply()` calls `prune_orphan_project_channels` at its start (apply-time cleanup); `apply_prunes_path_gone_channel_before_reconcile` proves a stale path-gone channel is swept by a real apply (not only the standalone `prune_removes_channels_whose_path_is_gone` unit test).
- [ ] COPY: `sync_to_project_dir_copies_real_dir_not_symlink` asserts `!is_symlink(dest)`.
- [ ] Seed snapshot: `save_seed_from_profile_is_a_snapshot_not_a_live_link` proves a later profile edit does not change the bound project.
- [ ] Device-local: `should_trigger_for_table("projects") == false` asserted.
- [ ] Owned-delete + write-then-record: detach removes only hash-matching files; user-edited and user-own files survive; every recorded row has a content_hash and an on-disk target.
- [ ] Scope: NO settings.json merge / CLAUDE.md / .mcp.json / `${VAR}` / non-Claude code paths were added.
