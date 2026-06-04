# AgentHub Increment 4b-3 — Project `<root>/.mcp.json` MERGE

REQUIRED-SUB-SKILL: superpowers:test-driven-development

## Goal

Materialize a bound project's selected MCP servers (`project.spec.content.mcp`
ids/`@tags`) into a deep-MERGED `<project>/.mcp.json` at the **project ROOT**
(not under `.claude/`), reusing the 4b-2 `settings_merge` engine **verbatim**.
Only the `mcpServers` subtree is touched; user-authored servers and other
top-level keys survive both re-apply and detach. Detach (and the apply
pre-delete sweep) reverse exactly our leaves via the shared `reverse_merge`.

## Architecture

- **Path**: `ProjectBase::mcp_file()` = `root.join(".mcp.json")` (ROOT level).
  BYPASS the home-global `config::get_claude_mcp_path()` entirely.
- **Server resolution**: `state.db.get_all_mcp_servers()` →
  `ProfileService::resolve_selectors("mcp", &project.spec.content.mcp, &items)`
  (same selector logic the global activate path uses, `profile.rs` §3d).
- **Strip**: a private `ProjectApplyService::strip_mcp_ui_fields` mirrors
  `claude_mcp.rs::set_mcp_servers_map`'s inline 8-field strip (+ legacy
  `{"server":{..}}` unwrap), MINUS Windows `cmd /c` wrapping (cross-machine repo).
- **Merge / teardown**: REUSE `services::settings_merge::{merge_with_snapshot,
  reverse_merge, OwnedKeysEnvelope, OWNED_KEYS_VERSION}` verbatim — they are
  file-agnostic over any `serde_json::Value`. NO new merge/teardown code.
- **Manifest**: `kind="mcp_merge"`, `content_hash=None`,
  `owned_keys=Some(envelope)` — identical shape to `settings_merge`. Teardown
  dispatch (`teardown_manifest_row`) gets ONE combined arm covering both merge
  kinds, BEFORE the catch-all `else`.
- **kind→enum: DEFERRED.** No `ManifestKind` enum, no DAO serde / migration /
  `profile.rs` change. The kind stays a `String`. Single-source-of-truth is two
  module-level `const &str` (`KIND_SETTINGS_MERGE`, `KIND_MCP_MERGE`) that just
  NAME the existing DB-stored literals.
- **Frontend (verified)**: `ProjectBindDialog.tsx` does NOT yet expose an MCP
  selector — line 61 hardcodes `mcp: []`. T6 ADDS the `mcp` includes field
  (mirroring the agents field) + i18n in all 4 locales. `content.mcp` already
  exists in `projects.ts` and the Rust `ProfileContent`; this is only the editor
  surface + wiring.

## Tech Stack

- Rust (crate `agenthub` / lib `agenthub_lib`), `serde_json`, `rusqlite`,
  `indexmap`, `tempfile`, `serial_test`. Schema **v18** (owned_keys present — NO
  new schema/migration).
- React + TypeScript + `react-i18next`; Vitest + `tsc` for the frontend gate.
- Repo: `/Users/fsm/project/MyProject/agentplugin/cc-switch-cloud`, branch
  `main`, base HEAD `6f1965f4`.

### Verified anchors (HEAD 6f1965f4, real signatures)

- `services/project_paths.rs`: `ProjectBase { root, dotdir }`; `pub fn root(&self)
  -> &Path` (L33); `pub fn settings_file(&self) -> PathBuf` (L55-57) =
  `self.dotdir().join("settings.json")`; insertion point for `mcp_file()` is
  AFTER `settings_file()` (after L57). The `settings_file` doc comment ends with
  "(`.mcp.json` lands in 4b-3)." (L54) — drop that clause.
- `services/project_apply.rs`:
  - `apply` materialize order: commands (L100-125) → agents (L127-151) → skills
    (L153-181) → project_memory CLAUDE.md (L183-210) → **settings.json merge
    block (L212-304)** → `Ok(result)` (L306). The mcp block inserts AFTER L304,
    BEFORE L306.
  - existing settings_merge `Self::row(... "settings_merge", None, Some(owned_json))`
    at L290-297.
  - `teardown_manifest_row(r, app, warnings)` L356-383: `skill` arm L361-369;
    `} else if r.kind == "settings_merge" {` arm L370-375 (calls `reverse_merge`);
    catch-all `} else {` L376-381 (`remove_whole_file_if_owned`). Doc comment
    L343-355 has "(e.g. 4b-3 adds an `mcp_merge` arm here, not in two loops)."
  - `fn row(channel, project_id, target, kind, content_hash: Option<&str>,
    owned_keys: Option<String>) -> ManifestEntry` L385-405 (6-arg).
- `services/settings_merge.rs`: `pub fn merge_with_snapshot(user: &mut Value,
  frag: &Value, path: &mut Vec<String>, out: &mut Vec<OwnedKey>)` (L53);
  `pub fn reverse_merge(file_path: &Path, owned_json: Option<&str>, warnings:
  &mut Vec<String>) -> Result<(), AppError>` (L113); `pub struct OwnedKeysEnvelope
  { pub v: u32, pub keys: Vec<OwnedKey> }` (L35); `pub const OWNED_KEYS_VERSION:
  u32 = 1` (L41).
- `database/dao/mcp.rs`: `pub fn get_all_mcp_servers(&self) ->
  Result<IndexMap<String, McpServer>, AppError>` (L13, `ORDER BY name ASC, id
  ASC`); `pub fn save_mcp_server(&self, server: &McpServer) -> Result<(),
  AppError>` (L70).
- `app_config.rs`: `pub struct McpServer { id: String, name: String, server:
  serde_json::Value, apps: McpApps, description: Option<String>, homepage:
  Option<String>, docs: Option<String>, tags: Vec<String> }` (L399-412);
  `McpApps` derives `Default`, 5 bools `{claude,codex,gemini,opencode,hermes}`
  (L8-20); `ProfileContent { skills, commands, agents, mcp: Vec<String> }`
  derives `Default` (L251-261); `ManifestEntry` (L363, `kind: String`,
  `content_hash`/`owned_keys` are `Option`, unchanged).
- `services/profile.rs`: `pub(crate) fn resolve_selectors(type_label: &str, spec:
  &[String], items: &[(String, Vec<String>)]) -> (HashSet<String>, Vec<String>)`
  (L452) — visible from `project_apply.rs` (same crate, already called there at
  L106/133/159). §3d global mcp usage at L133-148.
- `claude_mcp.rs` L405-427: the inline 8-field strip + legacy `server` unwrap
  the helper mirrors (`enabled,source,id,name,description,tags,homepage,docs`).
- `config.rs`: `pub(crate) fn sort_json_keys(value: &Value) -> Value` (L147);
  `pub fn atomic_write(path, data)` (L187) — `atomic_write` is already imported
  in `project_apply.rs` (L17).
- `lib.rs`: re-exports `McpApps, McpServer, ProfileContent, Project,
  ProjectDotfiles, ProjectSpec` (L39-42), `OwnedKeysEnvelope` (L62),
  `ProjectApplyService, ProjectBase` (via `services::*`) — all available to
  `tests/project_apply_e2e.rs`.
- Frontend: `src/components/projects/ProjectBindDialog.tsx` (hardcodes `mcp: []`
  L61, agents field L113-114, useState/useEffect L31/L42); `src/lib/api/projects.ts`
  L5 `content` already includes `mcp: string[]`; `src/i18n/locales/{en,zh,ja,
  zh-TW}.json` — `projects` section has `skills/commands/agents` (+Placeholder)
  but NO `projects.mcp`/`projects.mcpPlaceholder` (those keys exist only under
  `profiles`). en `projects` block L2775-2817; agents keys en L2806-2807.

### ENV / gate prelude (run before every cargo command)

```bash
export PATH="$HOME/.cargo/bin:$PATH"
export CARGO_NET_RETRY=10
cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud/src-tauri
```

Frontend gate (from repo root `cc-switch-cloud`): `./node_modules/.bin/tsc
--noEmit` and `./node_modules/.bin/vitest run` (NOT pnpm). All HOME-touching
Rust tests use `#[serial]` + the existing `TempHome` helper. `cargo fmt` BEFORE
every commit; clippy is `--all-targets -D warnings`; tests are the FULL suite
(NOT `--lib`). Baseline per brief: lib `1584` passing (capture the REAL number
on the first run and use it as your delta anchor).

---

## T1 — `ProjectBase::mcp_file()` (ROOT-level path) + path-shape guard

**Files**
- Modify: `src-tauri/src/services/project_paths.rs` (add `mcp_file()` after
  `settings_file()` ~L57; edit the `settings_file` doc comment L51-54).
- Test: `src-tauri/src/services/project_paths.rs` (inline `#[cfg(test)] mod
  tests`, add after `settings_file_is_under_dotdir` ~L422).

### Steps

1. **Write the failing test.** Append to the inline test module (after the
   `settings_file_is_under_dotdir` test):
   ```rust
   #[test]
   #[serial]
   fn mcp_file_is_root_level_not_under_dotdir() {
       let home = TempHome::new();
       let proj = home.home().join("work").join("mcprepo");
       std::fs::create_dir_all(&proj).expect("mkdir proj");
       let base = ProjectBase::resolve(proj.to_str().unwrap(), &AppType::Claude).expect("resolve");
       // .mcp.json lives at the project ROOT, NOT under .claude/ (community
       // convention; Claude reads project-root .mcp.json).
       assert_eq!(base.mcp_file(), base.root().join(".mcp.json"));
       assert_ne!(
           base.mcp_file(),
           base.settings_file(),
           "mcp_file must be root-level, NOT the .claude/ settings path"
       );
       assert_ne!(
           base.mcp_file(),
           base.dotdir().join(".mcp.json"),
           "regression guard: must NOT be .claude/.mcp.json"
       );
   }
   ```

2. **Run — expect FAIL (compile error: no method `mcp_file`).**
   ```bash
   cargo test -p agenthub_lib --lib services::project_paths::tests::mcp_file_is_root_level_not_under_dotdir 2>&1 | tail -20
   ```
   Expected: `error[E0599]: no method named `mcp_file` found for ... ProjectBase`.

3. **Minimal impl.** Add the method on `impl ProjectBase` AFTER `settings_file()`
   (after L57):
   ```rust
   /// Project-root MCP config file (4b-3: <root>/.mcp.json for Claude). ROOT level,
   /// NOT under dotdir() — Claude reads the project-root .mcp.json (community
   /// convention, mirrored by mcp/validation.rs). Project-scoped counterpart to the
   /// home-global ~/.claude.json that config::get_claude_mcp_path() returns — 4b-3
   /// must NOT use that fn.
   pub fn mcp_file(&self) -> PathBuf {
       self.root.join(".mcp.json")
   }
   ```
   And drop the stale reservation clause from the `settings_file` doc comment —
   change the L54 sentence ending from:
   `settings.json is a Claude-only kind (`.mcp.json` lands in 4b-3).`
   to:
   `settings.json is a Claude-only kind (.mcp.json is the root-level mcp_file()).`

4. **Run — expect PASS.**
   ```bash
   cargo test -p agenthub_lib --lib services::project_paths::tests::mcp_file_is_root_level_not_under_dotdir 2>&1 | tail -10
   ```
   Expected: `test ... ok. 1 passed`.

5. **fmt + commit.**
   ```bash
   cargo fmt
   git add -A && git commit -m "feat(project): add ProjectBase::mcp_file() root-level .mcp.json path (4b-3)

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
   ```

---

## T2 — `KIND_*` consts + retrofit existing `settings_merge` literals (pure rename)

**Files**
- Modify: `src-tauri/src/services/project_apply.rs` (add module-level consts ~L22;
  swap the `"settings_merge"` literal at the `Self::row` call L294 and the
  teardown arm L370).
- Test: no new test — the gate is that ALL existing tests stay green (the
  DB-stored string value is unchanged).

### Steps

1. **Failing test = the compile/behavior gate.** No new test code. First confirm
   the literal you are about to centralize is still produced byte-for-byte by
   running the existing assertion that pins the stored kind:
   ```bash
   cargo test -p agenthub_lib --lib services::project_apply::tests::apply_merges_settings_and_preserves_unrelated_user_keys 2>&1 | tail -10
   ```
   Expected: PASS (this is your green baseline before the rename).

2. **Add the consts** near the top of the module, just after the `use` block
   (~L22, before `pub struct ProjectApplyResult`):
   ```rust
   /// Single-source-of-truth names for the two MERGE manifest kinds (the
   /// kind→enum-DEFER mitigation, 4b-3). These NAME the existing DB-stored string
   /// literals; they do NOT change the stored values. Used by both the writer
   /// (Self::row) and the reader (teardown_manifest_row) so a typo can't desync
   /// the two sites.
   const KIND_SETTINGS_MERGE: &str = "settings_merge";
   const KIND_MCP_MERGE: &str = "mcp_merge";
   ```

3. **Retrofit the settings_merge writer.** At the existing settings_merge
   `Self::row` call (L290-297), change the kind argument from the literal
   `"settings_merge"` to `KIND_SETTINGS_MERGE`:
   ```rust
                       state.db.record_manifest_entry(&Self::row(
                           &channel,
                           &project.id,
                           &target,
                           KIND_SETTINGS_MERGE,
                           None,
                           Some(owned_json),
                       ))?;
   ```

4. **Retrofit the teardown reader.** In `teardown_manifest_row`, change the arm
   guard (L370) from `} else if r.kind == "settings_merge" {` to
   `} else if r.kind == KIND_SETTINGS_MERGE {` (body unchanged for now; T5
   widens it to also cover `KIND_MCP_MERGE`).

5. **Run — expect PASS (FULL suite, no behavior change).**
   ```bash
   cargo test -p agenthub_lib --lib 2>&1 | tail -15
   ```
   Expected: same pass count as the captured baseline (1584), zero failures.
   (Pure rename — the stored kind string is identical, so every assertion on
   `r.kind == "settings_merge"` still holds.)

6. **fmt + clippy + commit.**
   ```bash
   cargo fmt
   cargo clippy --all-targets -- -D warnings 2>&1 | tail -5
   git add -A && git commit -m "refactor(project): name merge-kind literals as KIND_* consts (4b-3)

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
   ```

---

## T3 — `strip_mcp_ui_fields` helper + unit tests

**Files**
- Modify: `src-tauri/src/services/project_apply.rs` (add private assoc fn on
  `impl ProjectApplyService`, place near `fn row` ~L385).
- Test: `src-tauri/src/services/project_apply.rs` (inline `#[cfg(test)] mod
  tests`, append after the last settings test ~L1616).

### Steps

1. **Write the failing tests.** Append to the inline test module:
   ```rust
   #[test]
   fn strip_mcp_ui_fields_drops_all_eight_and_keeps_connection() {
       let mut spec = serde_json::json!({
           "type": "stdio",
           "command": "node",
           "args": ["server.js"],
           "env": { "K": "v" },
           "enabled": true,
           "source": "registry",
           "id": "srv1",
           "name": "Server One",
           "description": "desc",
           "tags": ["a", "b"],
           "homepage": "https://h",
           "docs": "https://d"
       });
       ProjectApplyService::strip_mcp_ui_fields(&mut spec);
       assert_eq!(
           spec,
           serde_json::json!({
               "type": "stdio",
               "command": "node",
               "args": ["server.js"],
               "env": { "K": "v" }
           }),
           "only the connection fields survive"
       );
   }

   #[test]
   fn strip_mcp_ui_fields_noop_on_clean_stdio_spec() {
       let mut spec = serde_json::json!({
           "type": "stdio", "command": "uvx", "args": ["x"]
       });
       let before = spec.clone();
       ProjectApplyService::strip_mcp_ui_fields(&mut spec);
       assert_eq!(spec, before, "already-clean spec is unchanged");
   }

   #[test]
   fn strip_mcp_ui_fields_keeps_http_url() {
       let mut spec = serde_json::json!({
           "type": "http", "url": "https://mcp.example/api", "name": "X", "enabled": true
       });
       ProjectApplyService::strip_mcp_ui_fields(&mut spec);
       assert_eq!(
           spec,
           serde_json::json!({ "type": "http", "url": "https://mcp.example/api" }),
           "http/sse url survives, UI fields stripped"
       );
   }

   #[test]
   fn strip_mcp_ui_fields_unwraps_legacy_server_wrapper() {
       // legacy {"server":{..real..}, "name":..} → unwrap to the inner spec, then strip.
       let mut spec = serde_json::json!({
           "name": "wrapped",
           "enabled": true,
           "server": { "type": "stdio", "command": "go", "args": ["run"] }
       });
       ProjectApplyService::strip_mcp_ui_fields(&mut spec);
       assert_eq!(
           spec,
           serde_json::json!({ "type": "stdio", "command": "go", "args": ["run"] }),
           "legacy server wrapper unwrapped and stripped"
       );
   }

   #[test]
   fn strip_mcp_ui_fields_non_object_server_value_does_not_panic_or_lose_fields() {
       // a `server` value that is NOT an object is removed (dropped, never
       // reinserted, since we only reassign `*spec` when the unwrapped value
       // is an object); the connection fields survive and there is no panic.
       let mut spec = serde_json::json!({
           "type": "stdio", "command": "x", "server": "not-an-object", "enabled": true
       });
       ProjectApplyService::strip_mcp_ui_fields(&mut spec);
       assert_eq!(
           spec,
           serde_json::json!({ "type": "stdio", "command": "x" }),
           "non-object `server` is removed (dropped, never reinserted); connection fields survive, no panic"
       );
   }
   ```

2. **Run — expect FAIL (no method `strip_mcp_ui_fields`).**
   ```bash
   cargo test -p agenthub_lib --lib services::project_apply::tests::strip_mcp_ui_fields 2>&1 | tail -20
   ```
   Expected: `error[E0599]: no function or associated item named `strip_mcp_ui_fields``.

3. **Minimal impl.** Add the private assoc fn on `impl ProjectApplyService`
   (place it just above `fn row` ~L385):
   ```rust
   /// Strip UI/metadata fields from an MCP server spec before writing it to a project
   /// .mcp.json. Mirrors claude_mcp.rs::set_mcp_servers_map's inline 8-field strip,
   /// minus Windows cmd/c wrapping (omitted for cross-machine repo .mcp.json).
   fn strip_mcp_ui_fields(spec: &mut serde_json::Value) {
       if let Some(obj) = spec.as_object_mut() {
           if let Some(inner) = obj.remove("server") {
               if inner.is_object() {
                   *spec = inner;
               }
           }
       }
       if let Some(obj) = spec.as_object_mut() {
           for k in [
               "enabled",
               "source",
               "id",
               "name",
               "description",
               "tags",
               "homepage",
               "docs",
           ] {
               obj.remove(k);
           }
       }
   }
   ```

4. **Run — expect PASS.**
   ```bash
   cargo test -p agenthub_lib --lib services::project_apply::tests::strip_mcp_ui_fields 2>&1 | tail -15
   ```
   Expected: `5 passed`.

5. **fmt + clippy + commit.**
   ```bash
   cargo fmt
   cargo clippy --all-targets -- -D warnings 2>&1 | tail -5
   git add -A && git commit -m "feat(project): add strip_mcp_ui_fields helper for .mcp.json (4b-3)

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
   ```

---

## T4 — `apply()` mcp_merge materialize block

**Files**
- Modify: `src-tauri/src/services/project_apply.rs` (insert the block in `apply`
  AFTER the settings_merge block L304, BEFORE `Ok(result)` L306).
- Test: `src-tauri/src/services/project_apply.rs` (inline tests; add a
  `seed_mcp` helper near `seed_command` ~L590, and the apply tests after T3's).

### Steps

1. **Write the failing tests.** First add a seed helper next to `seed_command`
   (~L590):
   ```rust
   fn seed_mcp(db: &Database, id: &str, server: serde_json::Value, tags: Vec<String>) {
       let s = crate::app_config::McpServer {
           id: id.into(),
           name: id.into(),
           server,
           apps: crate::app_config::McpApps::default(),
           description: None,
           homepage: None,
           docs: None,
           tags,
       };
       db.save_mcp_server(&s).expect("save mcp server");
   }
   ```
   Then append the apply tests:
   ```rust
   #[test]
   #[serial]
   fn apply_writes_mcp_json_at_root_with_stripped_servers_and_envelope() {
       // Matrix #1 (happy) + path guard: two servers selected → ROOT .mcp.json,
       // stripped, ONE mcp_merge row, owned_keys envelope v=1.
       let home = TempHome::new();
       crate::settings::reload_settings().ok();
       let db = Arc::new(Database::memory().expect("db"));
       let state = AppState::new(db.clone());
       seed_mcp(
           &db,
           "srv-a",
           serde_json::json!({"type":"stdio","command":"a","enabled":true,"name":"A"}),
           vec![],
       );
       seed_mcp(
           &db,
           "srv-b",
           serde_json::json!({"type":"http","url":"https://b","source":"reg"}),
           vec![],
       );
       let (proj, canon) = project_at(
           home.home(),
           "mcp-a",
           ProfileContent {
               skills: vec![],
               commands: vec![],
               agents: vec![],
               mcp: vec!["srv-a".into(), "srv-b".into()],
           },
       );
       db.save_project(&proj).expect("save");

       // Pre-seed an existing on-disk .mcp.json with an EMPTY mcpServers object so the
       // engine RECURSES into mcpServers (both user + frag are objects) and records a
       // per-server leaf for each inserted key. Without this, an absent file collapses to
       // user_opt=Some({}); the top-level `mcpServers` key is absent in user, so the engine
       // hits the `None` arm ONCE at path ["mcpServers"] (prior.present=false) and inserts
       // the WHOLE subtree withOUT recursing — yielding NO ["mcpServers","srv-a"] leaf. The
       // absent-file shape is pinned separately by apply_mcp_absent_file_created_with_only_our_subtree
       // (T4 #4); here we want the per-server-leaf "covers both servers" shape.
       let target = canon.join(".mcp.json");
       std::fs::write(&target, br#"{"mcpServers":{}}"#).unwrap();

       let res = ProjectApplyService::apply(&state, &proj.id).expect("apply");
       assert!(res.warnings.is_empty(), "no warnings: {:?}", res.warnings);

       // .mcp.json at ROOT, NOT under .claude/.
       assert!(target.is_file(), "root .mcp.json written");
       assert!(
           !canon.join(".claude").join(".mcp.json").exists(),
           "must NOT be written under .claude/"
       );
       let disk: serde_json::Value =
           serde_json::from_slice(&std::fs::read(&target).unwrap()).unwrap();
       assert_eq!(
           disk["mcpServers"]["srv-a"],
           serde_json::json!({"type":"stdio","command":"a"}),
           "srv-a stripped (enabled/name gone)"
       );
       assert_eq!(
           disk["mcpServers"]["srv-b"],
           serde_json::json!({"type":"http","url":"https://b"}),
           "srv-b stripped (source gone)"
       );

       // exactly one mcp_merge row: content_hash None, owned_keys parses to v=1.
       let chan = format!("project:{}", canon.to_string_lossy());
       let rows = db.get_manifest_for_channel(&chan).unwrap();
       let m: Vec<_> = rows.iter().filter(|r| r.kind == "mcp_merge").collect();
       assert_eq!(m.len(), 1, "exactly one mcp_merge row");
       assert_eq!(m[0].content_hash, None, "merge rows carry NO content_hash");
       assert_eq!(m[0].target_path, target.to_string_lossy());
       let env: crate::services::settings_merge::OwnedKeysEnvelope =
           serde_json::from_str(m[0].owned_keys.as_deref().expect("owned_keys"))
               .expect("parse env");
       assert_eq!(env.v, crate::services::settings_merge::OWNED_KEYS_VERSION);
       // Pre-seeded empty mcpServers → engine recurses → ONE per-server leaf each
       // (both absent in the empty user map → prior.present=false). Covers BOTH servers.
       assert!(
           env.keys
               .iter()
               .any(|k| k.path == vec!["mcpServers".to_string(), "srv-a".to_string()]),
           "per-server leaf for srv-a recorded"
       );
       assert!(
           env.keys
               .iter()
               .any(|k| k.path == vec!["mcpServers".to_string(), "srv-b".to_string()]),
           "per-server leaf for srv-b recorded"
       );
   }

   #[test]
   #[serial]
   fn apply_empty_mcp_selection_writes_no_file_no_row() {
       // Matrix #6 empty-collapse: content.mcp=[] → no file, no row.
       let home = TempHome::new();
       crate::settings::reload_settings().ok();
       let db = Arc::new(Database::memory().expect("db"));
       let state = AppState::new(db.clone());
       seed_mcp(&db, "srv-a", serde_json::json!({"type":"stdio","command":"a"}), vec![]);
       let (proj, canon) = project_at(home.home(), "mcp-empty", ProfileContent::default());
       db.save_project(&proj).expect("save");
       ProjectApplyService::apply(&state, &proj.id).expect("apply");
       assert!(!canon.join(".mcp.json").exists(), "no file for empty selection");
       let chan = format!("project:{}", canon.to_string_lossy());
       assert!(
           !db.get_manifest_for_channel(&chan)
               .unwrap()
               .iter()
               .any(|r| r.kind == "mcp_merge"),
           "no mcp_merge row for empty selection"
       );
   }

   #[test]
   #[serial]
   fn apply_non_object_server_spec_is_skipped_with_warning() {
       // Matrix #8: server C spec is a String → A written, C skipped + warn.
       let home = TempHome::new();
       crate::settings::reload_settings().ok();
       let db = Arc::new(Database::memory().expect("db"));
       let state = AppState::new(db.clone());
       seed_mcp(&db, "srv-a", serde_json::json!({"type":"stdio","command":"a"}), vec![]);
       seed_mcp(&db, "srv-c", serde_json::json!("i-am-not-an-object"), vec![]);
       let (proj, canon) = project_at(
           home.home(),
           "mcp-nonobj",
           ProfileContent {
               skills: vec![],
               commands: vec![],
               agents: vec![],
               mcp: vec!["srv-a".into(), "srv-c".into()],
           },
       );
       db.save_project(&proj).expect("save");
       let res = ProjectApplyService::apply(&state, &proj.id).expect("apply");
       assert!(
           res.warnings.iter().any(|w| w.contains("srv-c") && w.contains("not a JSON object")),
           "warns about non-object srv-c: {:?}",
           res.warnings
       );
       let disk: serde_json::Value =
           serde_json::from_slice(&std::fs::read(canon.join(".mcp.json")).unwrap()).unwrap();
       assert!(disk["mcpServers"].get("srv-a").is_some(), "srv-a written");
       assert!(disk["mcpServers"].get("srv-c").is_none(), "srv-c skipped");
   }

   #[test]
   #[serial]
   fn apply_mcp_malformed_disk_skips_byte_identical_no_row() {
       // Matrix #7: pre-existing .mcp.json is invalid JSON → warn, byte-identical, NO row.
       let home = TempHome::new();
       crate::settings::reload_settings().ok();
       let db = Arc::new(Database::memory().expect("db"));
       let state = AppState::new(db.clone());
       seed_mcp(&db, "srv-a", serde_json::json!({"type":"stdio","command":"a"}), vec![]);
       let (proj, canon) = project_at(
           home.home(),
           "mcp-mal",
           ProfileContent {
               skills: vec![],
               commands: vec![],
               agents: vec![],
               mcp: vec!["srv-a".into()],
           },
       );
       db.save_project(&proj).expect("save");
       let target = canon.join(".mcp.json");
       std::fs::write(&target, b"not json {{").unwrap();
       let res = ProjectApplyService::apply(&state, &proj.id).expect("apply");
       assert!(
           res.warnings.iter().any(|w| w.contains("not a JSON value")),
           "warns on malformed disk: {:?}",
           res.warnings
       );
       assert_eq!(std::fs::read(&target).unwrap(), b"not json {{", "byte-identical");
       let chan = format!("project:{}", canon.to_string_lossy());
       assert!(
           !db.get_manifest_for_channel(&chan)
               .unwrap()
               .iter()
               .any(|r| r.kind == "mcp_merge"),
           "no row when disk skipped"
       );
   }

   #[test]
   #[serial]
   fn apply_mcp_absent_file_created_with_only_our_subtree() {
       // Matrix #4: merge into absent file → created with mcpServers.A, prior absent.
       let home = TempHome::new();
       crate::settings::reload_settings().ok();
       let db = Arc::new(Database::memory().expect("db"));
       let state = AppState::new(db.clone());
       seed_mcp(&db, "srv-a", serde_json::json!({"type":"stdio","command":"a"}), vec![]);
       let (proj, canon) = project_at(
           home.home(),
           "mcp-absent",
           ProfileContent {
               skills: vec![],
               commands: vec![],
               agents: vec![],
               mcp: vec!["srv-a".into()],
           },
       );
       db.save_project(&proj).expect("save");
       ProjectApplyService::apply(&state, &proj.id).expect("apply");
       let disk: serde_json::Value =
           serde_json::from_slice(&std::fs::read(canon.join(".mcp.json")).unwrap()).unwrap();
       assert_eq!(disk, serde_json::json!({"mcpServers":{"srv-a":{"type":"stdio","command":"a"}}}));
       let chan = format!("project:{}", canon.to_string_lossy());
       let m: Vec<_> = db
           .get_manifest_for_channel(&chan)
           .unwrap()
           .into_iter()
           .filter(|r| r.kind == "mcp_merge")
           .collect();
       let env: crate::services::settings_merge::OwnedKeysEnvelope =
           serde_json::from_str(m[0].owned_keys.as_deref().unwrap()).unwrap();
       let leaf = env
           .keys
           .iter()
           .find(|k| k.path == vec!["mcpServers".to_string()])
           .expect("mcpServers subtree leaf (absent-file → ONE whole-subtree leaf)");
       assert!(!leaf.prior.present, "prior.present=false for absent-file merge");
   }

   #[test]
   #[serial]
   fn apply_mcp_is_idempotent_one_row_identical_bytes() {
       // Matrix #9: apply [A] twice → identical bytes, ONE row.
       let home = TempHome::new();
       crate::settings::reload_settings().ok();
       let db = Arc::new(Database::memory().expect("db"));
       let state = AppState::new(db.clone());
       seed_mcp(&db, "srv-a", serde_json::json!({"type":"stdio","command":"a"}), vec![]);
       let (proj, canon) = project_at(
           home.home(),
           "mcp-idem",
           ProfileContent {
               skills: vec![],
               commands: vec![],
               agents: vec![],
               mcp: vec!["srv-a".into()],
           },
       );
       db.save_project(&proj).expect("save");
       ProjectApplyService::apply(&state, &proj.id).expect("apply 1");
       let bytes1 = std::fs::read(canon.join(".mcp.json")).unwrap();
       ProjectApplyService::apply(&state, &proj.id).expect("apply 2");
       let bytes2 = std::fs::read(canon.join(".mcp.json")).unwrap();
       assert_eq!(bytes1, bytes2, "re-apply produces identical bytes");
       let chan = format!("project:{}", canon.to_string_lossy());
       let count = db
           .get_manifest_for_channel(&chan)
           .unwrap()
           .iter()
           .filter(|r| r.kind == "mcp_merge")
           .count();
       assert_eq!(count, 1, "re-apply stays idempotent (one mcp_merge row)");
   }
   ```

2. **Run — expect FAIL** (no `.mcp.json` written / no `mcp_merge` row yet — the
   apply does nothing for mcp):
   ```bash
   cargo test -p agenthub_lib --lib services::project_apply::tests::apply_writes_mcp_json_at_root_with_stripped_servers_and_envelope 2>&1 | tail -20
   ```
   Expected: panic `root .mcp.json written` (the file is never created).

3. **Minimal impl.** Insert this block in `apply` AFTER the settings_merge block
   (after L304), BEFORE `Ok(result)` (L306):
   ```rust
           // ---- project .mcp.json (mcpServers subtree, deep MERGE; 4b-3, Claude only) ----
           // REUSES the 4b-2 settings_merge engine verbatim. The pre-delete sweep already
           // reverse_merged our prior mcp_merge row, so the on-disk .mcp.json is the USER
           // baseline here. Merge ONLY the mcpServers subtree at the project ROOT
           // (base.mcp_file()), NOT the home-global ~/.claude.json. NO ${VAR}, NO cmd/c.
           {
               let servers = state.db.get_all_mcp_servers()?;
               let items: Vec<(String, Vec<String>)> = servers
                   .values()
                   .map(|s| (s.id.clone(), s.tags.clone()))
                   .collect();
               let (want, w) =
                   ProfileService::resolve_selectors("mcp", &project.spec.content.mcp, &items);
               result.warnings.extend(w);

               let mut mcp_map = serde_json::Map::new();
               for server in servers.values() {
                   if !want.contains(&server.id) {
                       continue;
                   }
                   let mut spec = server.server.clone();
                   if !spec.is_object() {
                       result.warnings.push(format!(
                           "project .mcp.json: server '{}' spec is not a JSON object; skipping",
                           server.id
                       ));
                       continue;
                   }
                   Self::strip_mcp_ui_fields(&mut spec);
                   mcp_map.insert(server.id.clone(), spec);
               }

               // EMPTY-COLLAPSE: no servers resolve → record NO row, write NOTHING (the
               // pre-delete sweep already reversed our prior row → clean user-baseline file;
               // never an empty {"mcpServers":{}} stub).
               if !mcp_map.is_empty() {
                   let target = base.mcp_file();
                   let frag =
                       serde_json::json!({ "mcpServers": serde_json::Value::Object(mcp_map) });
                   // Option<Value> sentinel (None == skip), NOT Value::Null — mirror settings_merge.
                   let user_opt: Option<serde_json::Value> = if target.exists() {
                       match std::fs::read(&target)
                           .ok()
                           .and_then(|b| serde_json::from_slice(&b).ok())
                       {
                           Some(v) => Some(v),
                           None => {
                               result.warnings.push(format!(
                                   "project .mcp.json on disk is not a JSON value; skipping merge: {}",
                                   target.display()
                               ));
                               None
                           }
                       }
                   } else {
                       Some(serde_json::Value::Object(serde_json::Map::new()))
                   };
                   if let Some(mut user) = user_opt {
                       let mut owned = Vec::new();
                       let mut path = Vec::new();
                       crate::services::settings_merge::merge_with_snapshot(
                           &mut user, &frag, &mut path, &mut owned,
                       );
                       let bytes = serde_json::to_vec_pretty(&crate::config::sort_json_keys(&user))
                           .map_err(|e| AppError::Message(format!("serialize .mcp.json: {e}")))?;
                       atomic_write(&target, &bytes)?;
                       let env = crate::services::settings_merge::OwnedKeysEnvelope {
                           v: crate::services::settings_merge::OWNED_KEYS_VERSION,
                           keys: owned,
                       };
                       let owned_json = serde_json::to_string(&env)
                           .map_err(|e| AppError::Message(format!("serialize owned_keys: {e}")))?;
                       state.db.record_manifest_entry(&Self::row(
                           &channel,
                           &project.id,
                           &target,
                           KIND_MCP_MERGE,
                           None,
                           Some(owned_json),
                       ))?;
                   }
               }
           }

   ```
   Notes: `ProfileService` is already imported (L19); `atomic_write` (L17),
   `AppError` (L18), `config::sort_json_keys` are in scope; `KIND_MCP_MERGE` is
   the T2 const. The `Self::row` here is the 6-arg form (kind=`KIND_MCP_MERGE`,
   content_hash=`None`, owned_keys=`Some(owned_json)`).

4. **Run — expect PASS (the 6 T4 tests).**
   ```bash
   cargo test -p agenthub_lib --lib services::project_apply::tests::apply_writes_mcp_json_at_root_with_stripped_servers_and_envelope services::project_apply::tests::apply_empty_mcp_selection_writes_no_file_no_row services::project_apply::tests::apply_non_object_server_spec_is_skipped_with_warning services::project_apply::tests::apply_mcp_malformed_disk_skips_byte_identical_no_row services::project_apply::tests::apply_mcp_absent_file_created_with_only_our_subtree services::project_apply::tests::apply_mcp_is_idempotent_one_row_identical_bytes 2>&1 | tail -15
   ```
   Expected: `6 passed`.

5. **fmt + clippy + commit.**
   ```bash
   cargo fmt
   cargo clippy --all-targets -- -D warnings 2>&1 | tail -5
   git add -A && git commit -m "feat(project): merge selected MCP servers into <root>/.mcp.json (4b-3)

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
   ```

---

## T5 — teardown arm (COMBINE `settings_merge || mcp_merge`) + detach tests

**Files**
- Modify: `src-tauri/src/services/project_apply.rs` (widen the
  `KIND_SETTINGS_MERGE` arm in `teardown_manifest_row` L370-375 to also match
  `KIND_MCP_MERGE`; update the fn doc comment L343-355).
- Test: `src-tauri/src/services/project_apply.rs` (inline tests, after T4's).

### Steps

1. **Write the failing tests.** Append to the inline test module:
   ```rust
   #[test]
   #[serial]
   fn detach_removes_our_servers_preserves_user_servers_and_other_keys() {
       // Matrix #2 + #5 (whole-file-delete regression): user server + other top
       // key pre-exist; apply [A]; detach → user-x + otherTop survive, A gone,
       // FILE NOT whole-file-deleted (mcp_merge routes to reverse_merge).
       let home = TempHome::new();
       crate::settings::reload_settings().ok();
       let db = Arc::new(Database::memory().expect("db"));
       let state = AppState::new(db.clone());
       seed_mcp(&db, "srv-a", serde_json::json!({"type":"stdio","command":"a"}), vec![]);
       let (proj, canon) = project_at(
           home.home(),
           "mcp-detach",
           ProfileContent {
               skills: vec![],
               commands: vec![],
               agents: vec![],
               mcp: vec!["srv-a".into()],
           },
       );
       db.save_project(&proj).expect("save");

       // user already has a .mcp.json with their OWN server + an unrelated top key.
       let target = canon.join(".mcp.json");
       std::fs::write(
           &target,
           serde_json::to_vec_pretty(&serde_json::json!({
               "mcpServers": { "user-x": { "type": "stdio", "command": "u" } },
               "otherTop": 1
           }))
           .unwrap(),
       )
       .unwrap();

       ProjectApplyService::apply(&state, &proj.id).expect("apply");
       // after apply: both servers present, otherTop intact.
       let merged: serde_json::Value =
           serde_json::from_slice(&std::fs::read(&target).unwrap()).unwrap();
       assert!(merged["mcpServers"].get("srv-a").is_some(), "ours added");
       assert!(merged["mcpServers"].get("user-x").is_some(), "user server kept on apply");

       ProjectApplyService::detach(&state, &proj.id).expect("detach");
       assert!(target.exists(), "file survives detach (NOT whole-file-deleted)");
       let after: serde_json::Value =
           serde_json::from_slice(&std::fs::read(&target).unwrap()).unwrap();
       assert!(after["mcpServers"].get("srv-a").is_none(), "our server removed on detach");
       assert_eq!(
           after["mcpServers"]["user-x"],
           serde_json::json!({"type":"stdio","command":"u"}),
           "user server survives detach"
       );
       assert_eq!(after["otherTop"], serde_json::json!(1), "other top key survives");

       let chan = format!("project:{}", canon.to_string_lossy());
       assert_eq!(db.get_manifest_for_channel(&chan).unwrap().len(), 0, "rows cleared");
   }

   #[test]
   #[serial]
   fn mcp_merge_row_routes_to_reverse_merge_not_whole_file_delete() {
       // Matrix #5 (explicit regression): a synthetic content_hash=None mcp_merge
       // row whose owned_keys removes nothing → teardown must call reverse_merge,
       // never remove_whole_file_if_owned → file SURVIVES.
       let home = TempHome::new();
       crate::settings::reload_settings().ok();
       let db = Arc::new(Database::memory().expect("db"));
       let state = AppState::new(db.clone());
       let (proj, canon) = project_at(home.home(), "mcp-regress", ProfileContent::default());
       db.save_project(&proj).expect("save");

       let target = canon.join(".mcp.json");
       std::fs::write(&target, br#"{"mcpServers":{"user-x":{"command":"u"}}}"#).unwrap();
       let chan = format!("project:{}", canon.to_string_lossy());
       // an owned_keys envelope that owns NOTHING currently on disk (wrote a leaf
       // the user later changed) → reverse_merge leaves the file intact.
       let env = serde_json::to_string(&crate::services::settings_merge::OwnedKeysEnvelope {
           v: crate::services::settings_merge::OWNED_KEYS_VERSION,
           keys: vec![crate::services::settings_merge::OwnedKey {
               path: vec!["mcpServers".into(), "ghost".into()],
               prior: crate::services::settings_merge::PriorLeaf { present: false, value: None },
               wrote: serde_json::json!({"command":"never-on-disk"}),
           }],
       })
       .unwrap();
       db.record_manifest_entry(&ManifestEntry {
           id: 0,
           channel: chan.clone(),
           profile_id: None,
           project_id: Some(proj.id.clone()),
           app_type: "claude".into(),
           target_path: target.to_string_lossy().to_string(),
           kind: "mcp_merge".into(),
           content_hash: None,
           owned_keys: Some(env),
           created_at: 0,
       })
       .unwrap();

       ProjectApplyService::detach(&state, &proj.id).expect("detach");
       assert!(target.exists(), "mcp_merge row must NOT whole-file-delete the file");
       let after: serde_json::Value =
           serde_json::from_slice(&std::fs::read(&target).unwrap()).unwrap();
       assert_eq!(
           after["mcpServers"]["user-x"],
           serde_json::json!({"command":"u"}),
           "user server fully intact (reverse_merge, not whole-file delete)"
       );
   }

   #[test]
   #[serial]
   fn detach_user_field_inside_our_server_survives() {
       // Matrix #10 (per-field): pre-seed mcpServers.A={command,userField}; apply
       // frag A={command,args}; detach → pin to the engine's exact per-leaf result.
       // merge_with_snapshot recurses A (both objects): writes [A,args] (absent)
       // and overwrites [A,command] (prior "u"→"a"); userField is NOT in our frag
       // so it is untouched. reverse_merge removes [A,args], restores [A,command]
       // to "u"; userField survives → A = {"command":"u","userField":true}.
       let home = TempHome::new();
       crate::settings::reload_settings().ok();
       let db = Arc::new(Database::memory().expect("db"));
       let state = AppState::new(db.clone());
       seed_mcp(
           &db,
           "srv-a",
           serde_json::json!({"type":"stdio","command":"a","args":["x"]}),
           vec![],
       );
       let (proj, canon) = project_at(
           home.home(),
           "mcp-perfield",
           ProfileContent {
               skills: vec![],
               commands: vec![],
               agents: vec![],
               mcp: vec!["srv-a".into()],
           },
       );
       db.save_project(&proj).expect("save");

       let target = canon.join(".mcp.json");
       std::fs::write(
           &target,
           br#"{"mcpServers":{"srv-a":{"command":"u","userField":true}}}"#,
       )
       .unwrap();

       ProjectApplyService::apply(&state, &proj.id).expect("apply");
       let merged: serde_json::Value =
           serde_json::from_slice(&std::fs::read(&target).unwrap()).unwrap();
       // our frag overwrote command + added args + type; userField untouched.
       assert_eq!(merged["mcpServers"]["srv-a"]["command"], serde_json::json!("a"));
       assert_eq!(merged["mcpServers"]["srv-a"]["args"], serde_json::json!(["x"]));
       assert_eq!(merged["mcpServers"]["srv-a"]["userField"], serde_json::json!(true));

       ProjectApplyService::detach(&state, &proj.id).expect("detach");
       let after: serde_json::Value =
           serde_json::from_slice(&std::fs::read(&target).unwrap()).unwrap();
       assert_eq!(
           after["mcpServers"]["srv-a"],
           serde_json::json!({"command":"u","userField":true}),
           "per-leaf reverse: command restored, args/type removed, userField survives"
       );
   }
   ```
   (Matrix #3 "our server removed on detach" is covered by the first test's
   `srv-a` assertion; per-leaf engine output is pinned by #10. The
   `OwnedKey`/`PriorLeaf` paths used in the regression test are `pub` in
   `settings_merge` — already re-exported / crate-visible.)

2. **Run — expect FAIL.** Before the arm change, a `mcp_merge` row falls into the
   catch-all `else` → `remove_whole_file_if_owned`. With `content_hash=None`,
   `remove_whole_file_if_owned` skips deletion (unowned), so the file may
   survive but our leaves are NEVER reversed — the per-field / user-server
   assertions fail:
   ```bash
   cargo test -p agenthub_lib --lib services::project_apply::tests::detach_user_field_inside_our_server_survives 2>&1 | tail -20
   ```
   Expected: panic on `after["mcpServers"]["srv-a"]` — args/type still present
   (no reverse_merge ran), so the assertion against `{"command":"u","userField":true}`
   fails. (This proves the arm is needed.)

3. **Minimal impl.** Widen the existing `KIND_SETTINGS_MERGE` arm in
   `teardown_manifest_row` (L370-375) to cover BOTH merge kinds — do NOT add a
   second arm:
   ```rust
           } else if r.kind == KIND_SETTINGS_MERGE || r.kind == KIND_MCP_MERGE {
               // settings_merge (4b-2) + mcp_merge (4b-3): per-leaf reverse_merge
               // restores/removes ONLY the leaves we wrote, preserving user-authored
               // keys. Same fail-closed engine; only the kind discriminant differs.
               // CRITICAL: this MUST precede the catch-all else — a merge row in the
               // else would whole-file delete the user's settings.json/.mcp.json.
               crate::services::settings_merge::reverse_merge(
                   std::path::Path::new(&r.target_path),
                   r.owned_keys.as_deref(),
                   warnings,
               )?;
           } else {
   ```
   Then update the `teardown_manifest_row` doc comment (L343-355): change the
   `settings_merge` bullet to name both kinds and rewrite the parenthetical from
   future to past tense, e.g.:
   - bullet: `/// - `settings_merge` / `mcp_merge` → per-leaf `reverse_merge`
     (restores/removes only OUR leaves).`
   - CRITICAL note: `... Keeping the dispatch in one fn single-sources that
     invariant: the `settings_merge` (4b-2) and `mcp_merge` (4b-3) kinds SHARE
     one arm here, not two loops.`

4. **Run — expect PASS (the 3 T5 tests + the whole lib suite for regression).**
   ```bash
   cargo test -p agenthub_lib --lib services::project_apply::tests::detach_removes_our_servers_preserves_user_servers_and_other_keys services::project_apply::tests::mcp_merge_row_routes_to_reverse_merge_not_whole_file_delete services::project_apply::tests::detach_user_field_inside_our_server_survives 2>&1 | tail -15
   cargo test -p agenthub_lib --lib 2>&1 | tail -10
   ```
   Expected: `3 passed` for the targeted run; full lib = baseline `1584` + 14
   new (T1:1, T3:5, T4:6, T5:3) with zero failures. (Confirm the exact total
   against your captured baseline.)

5. **fmt + clippy + commit.**
   ```bash
   cargo fmt
   cargo clippy --all-targets -- -D warnings 2>&1 | tail -5
   git add -A && git commit -m "feat(project): combine settings_merge||mcp_merge teardown arm (4b-3)

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
   ```

---

## T6 — Frontend: add the MCP includes field to `ProjectBindDialog` + i18n (4 locales)

**Decision (verified):** `ProjectBindDialog.tsx` does NOT currently expose an
MCP selector — line 61 hardcodes `mcp: []`, and there is no `mcpServers` state.
`projects.ts` already types `content.mcp: string[]` and the Rust spec already
carries it; the `projects` i18n section has NO `mcp`/`mcpPlaceholder` keys
(those live only under `profiles`). So T6 ADDS the editor surface: a comma-list
input mirroring the existing `agents` field, wired into `spec.content.mcp`, plus
`projects.mcp` + `projects.mcpPlaceholder` in all 4 locales.

**Files**
- Modify: `src/components/projects/ProjectBindDialog.tsx`.
- Modify: `src/i18n/locales/{en,zh,ja,zh-TW}.json` (`projects` section).
- Test: `tsc --noEmit` (type gate) + `vitest run` (existing suite stays green);
  there is no component test harness for this dialog, so the verification is the
  type-check + i18n key presence + a manual render note. (If a
  `ProjectBindDialog.test.tsx` exists, extend it; none is present at HEAD.)

### Steps

1. **Add the i18n keys first (so `t()` calls resolve).** In EACH of the four
   locale files, inside the `projects` object, add `mcp` + `mcpPlaceholder`
   immediately after the `agents`/`agentsPlaceholder` pair. Use these strings
   (mirroring the `profiles.mcp*` wording already present in each locale):
   - `en.json` (after L2807):
     ```json
       "mcp": "MCP Servers",
       "mcpPlaceholder": "e.g. my-mcp-server, @backend (comma-separated)",
     ```
   - `zh.json`:
     ```json
       "mcp": "MCP 服务器",
       "mcpPlaceholder": "例如：my-mcp-server, @backend（逗号分隔）",
     ```
   - `ja.json`:
     ```json
       "mcp": "MCP サーバー",
       "mcpPlaceholder": "例：my-mcp-server, @backend（カンマ区切り）",
     ```
   - `zh-TW.json`:
     ```json
       "mcp": "MCP 伺服器",
       "mcpPlaceholder": "例如：my-mcp-server, @backend（逗號分隔）",
     ```
   (Match each file's existing comma/indentation style; keep JSON valid —
   trailing commas only where the existing keys have them.)

2. **Run the type gate — expect FAIL** (the dialog does not yet read/write mcp;
   no type error yet, but this step establishes the green baseline before wiring):
   ```bash
   ./node_modules/.bin/tsc --noEmit 2>&1 | tail -15
   ```
   Expected: PASS at this point (adding i18n keys alone does not break types).
   This is the pre-change baseline; the real "failing" signal is behavioral
   (the UI still can't set mcp), which step 3 fixes.

3. **Wire the field into the dialog.** Edit `ProjectBindDialog.tsx`:
   - add state next to `agents` (after L31):
     ```tsx
     const [mcp, setMcp] = useState("");
     ```
   - hydrate it in the `useEffect` (after the `agents` line L42):
     ```tsx
     setMcp((project?.spec.content.mcp ?? []).join(", "));
     ```
   - use it in `onSave` (replace the hardcoded `mcp: []` at L61):
     ```tsx
           mcp: splitList(mcp),
     ```
   - add the input after the agents field (after L114):
     ```tsx
     <label className="text-sm font-medium">{t("projects.mcp")}</label>
     <Input className="mt-1 mb-3" value={mcp} onChange={(e) => setMcp(e.target.value)} placeholder={t("projects.mcpPlaceholder")} />
     ```

4. **Run the gates — expect PASS.**
   ```bash
   ./node_modules/.bin/tsc --noEmit 2>&1 | tail -10
   ./node_modules/.bin/vitest run 2>&1 | tail -15
   ```
   Expected: `tsc` clean (0 errors); vitest all green (the new field does not
   touch existing tested code paths). Manually confirm via `./node_modules/.bin/vitest`
   that no snapshot of `ProjectBindDialog` exists that would need updating (none
   at HEAD).

5. **commit.**
   ```bash
   git add -A && git commit -m "feat(ui): expose MCP server selector in ProjectBindDialog + i18n (4b-3)

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
   ```

---

## T7 — e2e test + full gate suite

**Files**
- Modify: `src-tauri/tests/project_apply_e2e.rs` (add an mcp lifecycle test after
  `e2e_settings_merge_apply_then_detach_preserves_user_keys`).
- Test: the e2e file itself + the FULL gate suite.

### Steps

1. **Write the failing e2e test.** Add the lib re-exports needed (`McpServer`,
   `McpApps`) to the `use agenthub_lib::{...}` block at the top of the file
   (both are already re-exported from lib.rs L40), then append:
   ```rust
   #[test]
   #[serial]
   fn e2e_mcp_merge_apply_then_detach_preserves_user_servers() {
       let home = TempHome::new();
       let db = Arc::new(Database::memory().expect("db"));
       let state = AppState::new(db.clone());

       // a server in the catalog, selected by the project's content.mcp.
       db.save_mcp_server(&McpServer {
           id: "e2e-srv".into(),
           name: "E2E Server".into(),
           server: serde_json::json!({"type":"stdio","command":"node","args":["s.js"],"enabled":true}),
           apps: McpApps::default(),
           description: None,
           homepage: None,
           docs: None,
           tags: vec![],
       })
       .unwrap();

       let root = home.dir.path().join("e2e-mcp");
       std::fs::create_dir_all(&root).unwrap();
       let canon = root.canonicalize().unwrap();
       // user pre-existing .mcp.json with their OWN server + a sibling top key.
       let target = canon.join(".mcp.json");
       std::fs::write(
           &target,
           br#"{"mcpServers":{"user-srv":{"type":"stdio","command":"u"}},"keep":42}"#,
       )
       .unwrap();

       let mut spec = ProjectSpec::default();
       spec.content.mcp = vec!["e2e-srv".into()];
       let proj = Project {
           id: "proj:e2e-mcp".into(),
           project_path: canon.to_string_lossy().to_string(),
           entered_path: root.to_string_lossy().to_string(),
           app_type: "claude".into(),
           name: Some("E2E MCP".into()),
           spec,
           enabled: true,
           created_at: 1,
           updated_at: 1,
       };
       db.save_project(&proj).unwrap();

       // apply: our server merged + stripped, ROOT .mcp.json, envelope v=1 row.
       ProjectApplyService::apply(&state, &proj.id).expect("apply");
       assert!(
           !canon.join(".claude").join(".mcp.json").exists(),
           "never under .claude/"
       );
       let merged: serde_json::Value =
           serde_json::from_slice(&std::fs::read(&target).unwrap()).unwrap();
       assert_eq!(
           merged["mcpServers"]["e2e-srv"],
           serde_json::json!({"type":"stdio","command":"node","args":["s.js"]}),
           "our server merged + stripped (enabled gone)"
       );
       assert!(merged["mcpServers"].get("user-srv").is_some(), "user server kept");
       assert_eq!(merged["keep"], serde_json::json!(42), "sibling key kept");

       let chan = format!("project:{}", canon.to_string_lossy());
       let rows = db.get_manifest_for_channel(&chan).unwrap();
       let env: OwnedKeysEnvelope = serde_json::from_str(
           rows.iter()
               .find(|r| r.kind == "mcp_merge")
               .unwrap()
               .owned_keys
               .as_deref()
               .unwrap(),
       )
       .unwrap();
       assert_eq!(env.v, 1);

       // detach: our server reversed; user server + sibling key survive; file stays.
       ProjectApplyService::detach(&state, &proj.id).expect("detach");
       assert!(target.exists(), ".mcp.json survives detach");
       let after: serde_json::Value =
           serde_json::from_slice(&std::fs::read(&target).unwrap()).unwrap();
       assert!(after["mcpServers"].get("e2e-srv").is_none(), "our server removed");
       assert_eq!(
           after["mcpServers"]["user-srv"],
           serde_json::json!({"type":"stdio","command":"u"}),
           "user server survives detach"
       );
       assert_eq!(after["keep"], serde_json::json!(42), "sibling key survives");
   }
   ```

2. **Run — expect FAIL only if T4/T5 not built; otherwise this is the
   integration confirmation.** Run it in isolation first:
   ```bash
   cargo test -p agenthub --test project_apply_e2e e2e_mcp_merge_apply_then_detach_preserves_user_servers 2>&1 | tail -20
   ```
   Expected (with T1-T5 in place): PASS. If run before T4, it FAILS on the merge
   assertion (`our server merged`). (`db.save_mcp_server` + `McpServer`/`McpApps`
   are public; confirm the e2e `use` line includes `McpServer, McpApps`.)

3. **Full gate suite — run ALL gates and capture real numbers.**
   ```bash
   cargo fmt --check 2>&1 | tail -5
   cargo clippy --all-targets -- -D warnings 2>&1 | tail -8
   cargo test 2>&1 | tail -30          # FULL suite (NOT --lib): lib + all tests/*.rs
   ```
   From repo root `cc-switch-cloud`:
   ```bash
   ./node_modules/.bin/tsc --noEmit 2>&1 | tail -10
   ./node_modules/.bin/vitest run 2>&1 | tail -15
   ```
   Expected: `fmt --check` clean; clippy 0 warnings (`--all-targets` covers
   tests + e2e); full `cargo test` green with lib = baseline `1584` + 14 new
   unit tests, and `project_apply_e2e` = prior count + 1. tsc 0 errors; vitest
   green. RECORD the actual lib/e2e/vitest counts in the commit body.

4. **Final commit.**
   ```bash
   cargo fmt
   git add -A && git commit -m "test(project): e2e mcp_merge apply/detach + 4b-3 full gate pass

lib <ACTUAL_N> passing (baseline 1584 + 14), project_apply_e2e +1, tsc/vitest green.

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
   ```

---

## Final invariants checklist (verify before declaring done)

- [ ] `.mcp.json` written at project ROOT (`base.mcp_file()`), NEVER under
      `.claude/`; `get_claude_mcp_path` is NOT called anywhere in the new code.
- [ ] NO new merge/teardown code: only `merge_with_snapshot` + `reverse_merge`
      are called; no second `reverse_merge` arm — `settings_merge` and
      `mcp_merge` share ONE combined arm BEFORE the catch-all `else`.
- [ ] NO new schema / migration; `ManifestEntry.kind` stays `String`; no
      `ManifestKind` enum; `profile.rs` / DAO serde untouched.
- [ ] `KIND_SETTINGS_MERGE` / `KIND_MCP_MERGE` consts are the single source for
      both writer (`Self::row`) and reader (teardown arm); stored string values
      unchanged.
- [ ] Empty selection → no file, no row; malformed disk → `Option<Value>` None
      sentinel → warn + byte-identical + no row; non-object server → warn + skip.
- [ ] Whole-file-delete regression test passes (mcp_merge row → `reverse_merge`,
      never `remove_whole_file_if_owned`).
- [ ] Per-field server survival pinned to the engine's exact per-leaf output.
- [ ] T6 verified-then-acted: dialog now feeds `spec.content.mcp`; all 4 locales
      have `projects.mcp` + `projects.mcpPlaceholder`.
- [ ] FULL gate suite green (fmt --check, clippy --all-targets -D warnings, full
      cargo test, tsc, vitest) with real numbers recorded.
