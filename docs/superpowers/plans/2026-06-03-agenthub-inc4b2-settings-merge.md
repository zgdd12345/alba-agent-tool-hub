# AgentHub Increment 4b-2 — Project settings.json MERGE + ${VAR}

REQUIRED-SUB-SKILL: superpowers:test-driven-development

## Goal

Bind a real project directory to an own `settings.json` **fragment** (with `${VAR}` rendering) and MERGE it into `<project>/.claude/settings.json` on apply — a USER, version-controlled file. This is the DANGEROUS data-safety core: we must never whole-file-delete or clobber the user's settings.json, must record EXACTLY which leaves we wrote (a per-leaf snapshot), and on teardown restore/remove ONLY our own leaves (per-leaf `cur==wrote` gate), leaving every user key (including emptied user objects, left as `{}`) intact.

A bound project's `spec.dotfiles` gains a `settings: String` field (serde-only, no storage migration). On apply, when `settings` is non-empty, AgentHub renders `${VAR}` (precedence: `project.spec.vars` > active provider `settings_config.env` > allowlisted process env — NOT the global profile vars), parses the rendered fragment as JSON, deep-merges it into the on-disk settings.json (recording a flat list of owned leaves with their prior values), atomic-writes the merged file, and records an `apply_manifest` row `kind="settings_merge"` with a versioned `owned_keys` envelope (schema **v18** adds the `owned_keys` column). Teardown (apply pre-delete sweep + detach) gets a NEW explicit `settings_merge` arm that calls `reverse_merge` — inserted BEFORE the catch-all `else` so a merge row never reaches `remove_whole_file_if_owned` (which would whole-file delete the user's settings.json).

`settings.json` arrays (`permissions.allow`/`deny`, `hooks`) are **WHOLE-ARRAY replace** (project wins on disk; detach restores the prior array via the snapshot). No element-union in v1 — documented loudly in code + UI.

## Architecture

- **Merge engine** — new module `src-tauri/src/services/settings_merge.rs` (registered `pub mod settings_merge;` in `services/mod.rs`). Self-contained:
  - `merge_with_snapshot(user, frag, path, out)` — PURE forward deep-merge that records a FLAT list of independent `OwnedKey` leaves. It mirrors `live.rs::json_deep_merge`'s Object×Object-recurse / else-overwrite shape but does NOT call it (per adversarial fix #2 — no `pub(crate)` promotion, no `live.rs` edit). Absent frag key → ONE whole-subtree leaf (`prior.present=false`). Present-as-object → recurse. Present-as-non-object (or frag-leaf vs user-object) → one leaf `prior.present=true`. Arrays are leaves (whole replace).
  - `reverse_merge(file_path, owned_json, warnings)` — PURE teardown, a function of `(owned_keys + current disk)` only: NO re-render, NO env read. Per-leaf `cur==wrote` gate: a leaf we inserted (`prior.present=false`) and still own (`cur==wrote`) is removed; a leaf we overwrote (`prior.present=true`) and still own is restored to `prior.value`; otherwise (user re-edited) we leave it. **NO `collapse_empty_created_ancestors`** (adversarial fix #1): emptied user objects are left as `{}`. Unknown envelope version, unparseable owned_json, `None`/empty, or invalid on-disk JSON → FAIL-CLOSED (warn, leave file untouched, `Ok(())`).
  - `OwnedKeysEnvelope { v: u32, keys: Vec<OwnedKey> }` (`OWNED_KEYS_VERSION = 1`), `OwnedKey { path: Vec<String>, prior: PriorLeaf, wrote: Value }`, `PriorLeaf { present: bool, value: Option<Value> }`.
  - `sort_json_keys_value(&Value) -> Value` — deterministic key-sorted clone, defined locally (config.rs's `sort_json_keys` is a PRIVATE free fn, not importable), mirrors its recursion shape.
- **owned_keys envelope (schema v18)** — `apply_manifest.owned_keys TEXT` carries the serialized `OwnedKeysEnvelope`. `SCHEMA_VERSION = 18`, base DDL + `migrate_v17_to_v18` (mirror `migrate_v16_to_v17`: `add_column_if_missing`) + dispatch arm. `content_hash` and `owned_keys` are mutually exclusive per row: content/whole-file rows carry `(Some(&hash), None)`; settings_merge rows carry `(None, Some(owned_json))`.
- **Storage (NO migration)** — `ProjectDotfiles` gains `#[serde(default)] pub settings: String` (wire key `settings`). Stored in the existing `projects.spec` JSON blob; old v17 blobs without `settings` deserialize unchanged. The schema bump is ONLY for `apply_manifest.owned_keys`, NOT for storage.
- **Path** — `ProjectBase::settings_file(&self) -> PathBuf = self.dotdir().join("settings.json")` → `<root>/.claude/settings.json`. Inside `.claude/`, past `resolve`'s gate.
- **${VAR}** — new `build_project_var_map(db, app_type, project) -> Result<VarMap, AppError>`: layers low→high = (1) allowlisted process env (`profile_vars::ENV_ALLOWLIST_PREFIXES`), (2) active provider `settings_config.env` (via `get_effective_current_provider` + `get_provider_by_id`), (3) `project.spec.vars` (TOP). Does NOT call `build_var_map` (which would leak the global profile's vars). Reuses `profile_vars::coerce_value` (promoted `pub(crate)`) + `VarMap::from_index_map` (new `pub(crate)`). The fragment is rendered with `substitute_vars(.., json_escape=true, ..)` (the same json_escape contract substitute_vars uses for splicing a value into a JSON string body) BEFORE `serde_json::from_str`. `reverse_merge` does NOT re-render (teardown is a pure fn of the stored snapshot).
- **Loop restructure (CRITICAL, contract c+d)** — both the apply pre-delete sweep AND detach get an explicit `} else if r.kind == "settings_merge" { reverse_merge(..) }` arm inserted BEFORE the catch-all `else`. The apply pre-delete reverse runs BEFORE the materialize block re-reads disk + re-merges (contract d: our prior write never becomes the new "user baseline"). Defense-in-depth (fix #5): settings_merge rows carry `content_hash=None`, so even if a merge row reached the else, `remove_whole_file_if_owned(.., None)` is a no-op (refuses) — a second line, NOT a substitute for the explicit arm. The M10 regression test guards this.
- **Materialize** — a new block in `ProjectApplyService::apply` AFTER the 4b-1 CLAUDE.md block, BEFORE `Ok(result)`. Renders the fragment, parses it (bad render/parse → warn, no write, no row), loads current disk as `Option<Value>` (`None` == skip-the-merge sentinel: absent file → `Some({})`, present+valid → `Some(v)`, present+invalid → `None`+warn — per fix #3, NOT `Value::Null`), `merge_with_snapshot`, atomic-writes the sorted merged value, and records the `settings_merge` row with the envelope.
- **Self::row** — becomes 6-arg: `content_hash: Option<&str>` + new `owned_keys: Option<String>`. The 4 content callers pass `(Some(&hash), None)`; settings_merge passes `(None, Some(owned_json))`. EVERY `ManifestEntry` literal across src AND tests gains `owned_keys` (read-mappings carry the read value; constructed literals carry `None`).
- **Frontend** — `projects.ts` dotfiles type gains `settings: string`; `ProjectBindDialog.tsx` adds a settings `<textarea>` (mirrors the claudeMd textarea); seed-from-profile copies the profile's `settings.json` dotfile into `spec.dotfiles.settings`; i18n keys (`settings`/`settingsPlaceholder`/`settingsHint`) in all 4 locales; vitest round-trip.

## Tech Stack

- Rust 1.95, crate `agenthub_lib` / bin `agenthub`. `serde`, `serde_json`, `rusqlite`, `sha2` (via `profile_render::content_hash`), `indexmap`, `tempfile`, `serial_test`, `chrono`.
- Frontend: React 18 + TypeScript, `react-i18next`, `@tanstack/react-query`, Vitest, `tsc --noEmit`.
- Repo: `/Users/fsm/project/MyProject/agentplugin/cc-switch-cloud`, branch `main`, HEAD `0d2e856b`, schema **v17 → v18**.

### Env gotchas (apply to EVERY run/commit command)
- `cargo` lives at `~/.cargo/bin` → prefix every cargo command with `export PATH="$HOME/.cargo/bin:$PATH";`. Slow net → also `export CARGO_NET_RETRY=10;`.
- Frontend tools run DIRECTLY from `./node_modules/.bin` (NOT pnpm): `./node_modules/.bin/tsc --noEmit`, `./node_modules/.bin/vitest run`.
- HOME-isolation Rust tests use `#[serial]` + the in-module `TempHome` guard (sets HOME/USERPROFILE/CC_SWITCH_TEST_HOME) and call `crate::settings::reload_settings().ok();` before touching the db.
- `cargo fmt` before EVERY commit; gate is `cargo fmt --check`.
- Clippy gate is `--all-targets` (lints tests too): `cargo clippy --all-targets -- -D warnings`.
- Full `cargo test` (NOT `--lib`) — the integration crate `tests/project_apply_e2e.rs` constructs `ProjectSpec` directly and must compile.
- RECURRING LESSON: `owned_keys` touches `ManifestEntry` → grep BOTH src AND tests, and run `cargo test --no-run --all-targets` after the DAO/struct edits (a missed positional-tuple index gives a clear arity error).

### Baseline (HEAD `0d2e856b`)
- lib unit tests: **1551** passing. Vitest: **316+** passing. 4b-2 ADDS tests (it never removes any). The existing `schema_migration_v16_to_v17_reaches_current` test asserts `SCHEMA_VERSION == 17` and MUST be updated in T1 (it becomes `>= 17` / the new v18 test owns the exact-version assert).

---

## T1 — Schema v18 plumbing (SCHEMA_VERSION + base DDL + migrate_v17_to_v18 + dispatch arm)

### Files
- Modify: `src-tauri/src/database/mod.rs` (`SCHEMA_VERSION` :52)
- Modify: `src-tauri/src/database/schema.rs` (base `apply_manifest` DDL :171-186; dispatch ladder :564-570; new `migrate_v17_to_v18` after `migrate_v16_to_v17` :1496)
- Modify: `src-tauri/src/database/tests.rs` (fix `schema_migration_v16_to_v17_reaches_current` :1050; add two new migration tests after :1072)
- Test: `src-tauri/src/database/tests.rs`

### Steps

1. **Write failing tests** — in `tests.rs`, first FIX the existing exact-version assert at `schema_migration_v16_to_v17_reaches_current` (:1059), replacing `assert_eq!(SCHEMA_VERSION, 17, ...)` with a `>=` so the v16→v17 hop test stays valid under v18:
```rust
    assert!(SCHEMA_VERSION >= 17, "SCHEMA_VERSION must be at least 17");
```
   Then append two new tests after `schema_migration_v16_to_v17_reaches_current` (after :1072, before the next `#[test]`):
```rust
#[test]
fn schema_v18_base_create_has_apply_manifest_owned_keys() {
    let conn = Connection::open_in_memory().expect("open memory db");
    Database::create_tables_on_conn(&conn).expect("create tables");
    assert!(
        Database::has_column(&conn, "apply_manifest", "owned_keys").expect("has_column"),
        "apply_manifest.owned_keys must exist in base create_tables (v18)"
    );
}

#[test]
fn schema_migration_v17_to_v18_adds_owned_keys() {
    let conn = Connection::open_in_memory().expect("open memory db");
    Database::create_tables_on_conn(&conn).expect("create tables");
    // Hand-build a v17 apply_manifest WITHOUT owned_keys, then stamp user_version=17.
    conn.execute("DROP TABLE apply_manifest", []).expect("drop");
    conn.execute(
        "CREATE TABLE apply_manifest (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            channel TEXT NOT NULL DEFAULT 'global',
            profile_id TEXT,
            app_type TEXT NOT NULL,
            target_path TEXT NOT NULL,
            kind TEXT NOT NULL,
            created_at INTEGER NOT NULL DEFAULT 0,
            content_hash TEXT,
            project_id TEXT
        )",
        [],
    )
    .expect("create v17 apply_manifest");
    assert!(
        !Database::has_column(&conn, "apply_manifest", "owned_keys").expect("has_column"),
        "precondition: v17 table has no owned_keys"
    );
    Database::set_user_version(&conn, 17).expect("seed v17");
    Database::apply_schema_migrations_on_conn(&conn).expect("apply migration");
    assert_eq!(
        Database::get_user_version(&conn).expect("version"),
        SCHEMA_VERSION
    );
    assert_eq!(SCHEMA_VERSION, 18, "SCHEMA_VERSION must be bumped to 18");
    assert!(
        Database::has_column(&conn, "apply_manifest", "owned_keys").expect("has_column"),
        "apply_manifest.owned_keys must exist after v17 -> v18 migration"
    );
    // idempotent re-run: migrating an already-current DB is a no-op.
    Database::apply_schema_migrations_on_conn(&conn).expect("re-run migration");
    assert_eq!(Database::get_user_version(&conn).expect("version"), 18);
}
```

2. **Run (expect FAIL — compile error: SCHEMA_VERSION is 17, no owned_keys column)**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; export CARGO_NET_RETRY=10; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud/src-tauri && cargo test --lib schema_ 2>&1 | tail -20
```

3. **Minimal impl**:
   - `mod.rs:52`: `pub(crate) const SCHEMA_VERSION: i32 = 18;`
   - `schema.rs` base DDL (the `CREATE TABLE IF NOT EXISTS apply_manifest` at :172-183): add `owned_keys TEXT,` after `project_id   TEXT,` and before the FK line:
```rust
        conn.execute(
            "CREATE TABLE IF NOT EXISTS apply_manifest (
        id           INTEGER PRIMARY KEY AUTOINCREMENT,
        channel      TEXT NOT NULL DEFAULT 'global',
        profile_id   TEXT,
        app_type     TEXT NOT NULL,
        target_path  TEXT NOT NULL,
        kind         TEXT NOT NULL,
        created_at   INTEGER NOT NULL DEFAULT 0,
        content_hash TEXT,
        project_id   TEXT,
        owned_keys   TEXT,
        FOREIGN KEY (profile_id) REFERENCES profiles(id) ON DELETE CASCADE
    )",
            [],
        )
        .map_err(|e| AppError::Database(e.to_string()))?;
```
   - `schema.rs` new migration after `migrate_v16_to_v17` (after :1496), mirroring it:
```rust
    /// v17 -> v18 迁移：为 apply_manifest 新增 owned_keys TEXT 列（4b-2 项目 settings.json
    /// 合并的 per-leaf 快照信封）。与 base CREATE 收敛（IF NOT EXISTS + add_column_if_missing）。
    /// 仅用于 apply 清单；projects.spec 内的 dotfiles.settings 是纯 serde，不涉及 schema。
    fn migrate_v17_to_v18(conn: &Connection) -> Result<(), AppError> {
        if Self::table_exists(conn, "apply_manifest")? {
            Self::add_column_if_missing(conn, "apply_manifest", "owned_keys", "TEXT")?;
        }
        log::info!("v17 -> v18 migration done: apply_manifest.owned_keys");
        Ok(())
    }
```
   - `schema.rs` dispatch ladder: add the `17 =>` arm after the `16 =>` arm (after :570), before `_ =>`:
```rust
                    17 => {
                        log::info!("migrating db v17 -> v18 (apply_manifest.owned_keys)");
                        Self::migrate_v17_to_v18(conn)?;
                        Self::set_user_version(conn, 18)?;
                    }
```

4. **Run (expect PASS)**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; export CARGO_NET_RETRY=10; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud/src-tauri && cargo test --lib schema_ 2>&1 | tail -20
```

5. **Commit**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && cargo fmt && git add -A && git commit -m "feat(db): schema v18 — apply_manifest.owned_keys column + migration

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## T2 — `ManifestEntry.owned_keys` field + DAO (record + both getters) + every literal site

### Files
- Modify: `src-tauri/src/app_config.rs` (`ManifestEntry` struct :358-371)
- Modify: `src-tauri/src/database/dao/manifest.rs` (INSERT :26-40; `get_manifest_for_profile` SELECT + positional tuple + destructure + literal :52-92; `get_manifest_for_channel` SELECT + named mapping :135-156; test literals :215, :272, :288, :335)
- Modify: `src-tauri/src/services/profile_render.rs` (`render_whole_file` literal :143-153)
- Modify: `src-tauri/src/services/project_apply.rs` (test stale-row literals :682, :721, :758 — `Self::row` is handled in T8)
- Test: `src-tauri/src/database/dao/manifest.rs` (extend `manifest_crud_roundtrip`)

### Steps

1. **Write failing test** — extend `manifest_crud_roundtrip` in `manifest.rs` (after the `assert_eq!(rows[0].content_hash, ...)` at :244) to assert `owned_keys` round-trips, and update `make_entry` is left as `None` (a content row). Insert after :244:
```rust
        assert_eq!(rows[0].owned_keys, None, "content rows carry no owned_keys");

        // a settings_merge-style row carrying an owned_keys envelope round-trips.
        let mut e_owned = make_entry(p, "claude", "/home/user/.claude/settings.json");
        e_owned.kind = "settings_merge".into();
        e_owned.content_hash = None;
        e_owned.owned_keys = Some(r#"{"v":1,"keys":[]}"#.into());
        let id_owned = db.record_manifest_entry(&e_owned)?;
        let back = db
            .get_manifest_for_profile(p, "claude")?
            .into_iter()
            .find(|r| r.id == id_owned)
            .expect("owned row");
        assert_eq!(back.owned_keys.as_deref(), Some(r#"{"v":1,"keys":[]}"#));
        assert_eq!(back.content_hash, None);
```

2. **Run (expect FAIL — `ManifestEntry` has no field `owned_keys`)**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; export CARGO_NET_RETRY=10; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud/src-tauri && cargo test --lib manifest_crud_roundtrip 2>&1 | tail -20
```

3. **Minimal impl**:
   - `app_config.rs` `ManifestEntry` — add the field after `content_hash`, before `created_at`:
```rust
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub content_hash: Option<String>,
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub owned_keys: Option<String>,
    pub created_at: i64,
```
   - `manifest.rs` `record_manifest_entry` INSERT (:26-40) — 9 columns, owned_keys between content_hash and created_at:
```rust
        conn.execute(
            "INSERT INTO apply_manifest
             (channel, profile_id, project_id, app_type, target_path, kind, content_hash, owned_keys, created_at)
             VALUES (?1, ?2, ?3, ?4, ?5, ?6, ?7, ?8, ?9)",
            params![
                e.channel,
                e.profile_id,
                e.project_id,
                e.app_type,
                e.target_path,
                e.kind,
                e.content_hash,
                e.owned_keys,
                e.created_at,
            ],
        )
```
   - `manifest.rs` `get_manifest_for_profile` (:52-92) — SELECT adds `owned_keys` (created_at shifts 8→9); the positional tuple gains `row.get::<_, Option<String>>(8)?` for owned_keys and bumps created_at to `row.get::<_, i64>(9)?`; the 9-name destructure gains `owned_keys`; the struct literal gains `owned_keys` (fix #4):
```rust
        let mut stmt = conn
            .prepare(
                "SELECT id, channel, profile_id, project_id, app_type, target_path, kind, content_hash, owned_keys, created_at
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
                    row.get::<_, Option<String>>(8)?,
                    row.get::<_, i64>(9)?,
                ))
            })
            .map_err(|e| AppError::Database(e.to_string()))?;

        let mut entries = Vec::new();
        for row_res in rows {
            let (id, channel, p_id, proj_id, a_type, target_path, kind, content_hash, owned_keys, created_at) =
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
                owned_keys,
                created_at,
            });
        }
```
   - `manifest.rs` `get_manifest_for_channel` (:135-156) — SELECT adds `owned_keys`, named mapping adds `owned_keys: row.get(8)?` and bumps `created_at: row.get(9)?`:
```rust
        let mut stmt = conn
            .prepare(
                "SELECT id, channel, profile_id, project_id, app_type, target_path, kind, content_hash, owned_keys, created_at
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
                    owned_keys: row.get(8)?,
                    created_at: row.get(9)?,
                })
            })
            .map_err(|e| AppError::Database(e.to_string()))?;
```
   - `manifest.rs` test literals: `make_entry` (:215-225) add `owned_keys: None,` after `content_hash`; the `g` literal (:272), the `record_manifest_entry(&ManifestEntry {` loop literal (:288), and the `e` literal (:335) each add `owned_keys: None,` after their `content_hash` field.
   - `profile_render.rs` `render_whole_file` literal (:143-153) add `owned_keys: None,` after `content_hash: Some(new_hash),`.
   - `project_apply.rs` three test stale-row literals at :682, :721, :758 each add `owned_keys: None,` after their `content_hash` field. (`Self::row` itself is migrated in T8.)

4. **Run (expect PASS) + the RECURRING-LESSON full-target compile** (catches any missed `ManifestEntry` literal in src OR tests):
```bash
export PATH="$HOME/.cargo/bin:$PATH"; export CARGO_NET_RETRY=10; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud/src-tauri && cargo test --lib manifest_crud_roundtrip 2>&1 | tail -10 && cargo test --no-run --all-targets 2>&1 | tail -15
```
   NOTE: `Self::row` in `project_apply.rs:283` still constructs `ManifestEntry` with the OLD shape; add `owned_keys: None,` there too as a TEMPORARY field so the crate compiles — T8 changes its signature. Add `owned_keys: None,` after `content_hash: Some(content_hash.to_string()),` in `Self::row` now.

5. **Commit**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && cargo fmt && git add -A && git commit -m "feat(db): ManifestEntry.owned_keys + DAO 9-column read/write

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## T3 — `ProjectBase::settings_file()` path helper

### Files
- Modify: `src-tauri/src/services/project_paths.rs` (add `settings_file` after `memory_file` :49; add test in the inline test module after `memory_file_is_root_level_not_under_dotdir` :398)
- Test: `src-tauri/src/services/project_paths.rs`

### Steps

1. **Write failing test** — append in the test module after `memory_file_is_root_level_not_under_dotdir` (before the closing `}` at :399):
```rust
    #[test]
    #[serial]
    fn settings_file_is_under_dotdir() {
        let home = TempHome::new();
        let proj = home.home().join("work").join("setrepo");
        std::fs::create_dir_all(&proj).expect("mkdir proj");
        let base = ProjectBase::resolve(proj.to_str().unwrap(), &AppType::Claude).expect("resolve");
        // settings.json lives UNDER .claude/, unlike CLAUDE.md (root-level).
        assert_eq!(
            base.settings_file(),
            base.dotdir().join("settings.json")
        );
        assert_ne!(
            base.settings_file(),
            base.root().join("settings.json"),
            "settings_file must use dotdir(), not root()"
        );
    }
```

2. **Run (expect FAIL — no method `settings_file`)**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; export CARGO_NET_RETRY=10; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud/src-tauri && cargo test --lib settings_file_is_under_dotdir 2>&1 | tail -15
```

3. **Minimal impl** — in `impl ProjectBase`, after `memory_file` (:49):
```rust
    /// Project settings file for Claude (4b-2: <root>/.claude/settings.json).
    /// UNDER dotdir() — Claude Code reads project-local settings.json (merged
    /// with ~/.claude/settings.json by Claude itself). NOT app-parameterized in
    /// v1: settings.json is a Claude-only kind (`.mcp.json` lands in 4b-3).
    pub fn settings_file(&self) -> PathBuf {
        self.dotdir().join("settings.json")
    }
```

4. **Run (expect PASS)**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; export CARGO_NET_RETRY=10; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud/src-tauri && cargo test --lib settings_file_is_under_dotdir 2>&1 | tail -10
```

5. **Commit**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && cargo fmt && git add -A && git commit -m "feat(project): ProjectBase::settings_file() under .claude/

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## T4 — profile_vars: promote `coerce_value` pub(crate) + add `VarMap::from_index_map`

### Files
- Modify: `src-tauri/src/services/profile_vars.rs` (`coerce_value` visibility :61; new `VarMap::from_index_map` in `impl VarMap` :49-54; new test in the test module)
- Test: `src-tauri/src/services/profile_vars.rs`

### Steps

1. **Write failing test** — append in the `#[cfg(test)] mod tests` (before its closing `}` at :606):
```rust
    #[test]
    fn varmap_from_index_map_preserves_entries() {
        let mut m: IndexMap<String, String> = IndexMap::new();
        m.insert("A".to_string(), "1".to_string());
        m.insert("B".to_string(), "two".to_string());
        let vm = VarMap::from_index_map(m);
        assert_eq!(vm.get("A"), Some("1"));
        assert_eq!(vm.get("B"), Some("two"));
        assert_eq!(vm.get("MISSING"), None);
    }

    #[test]
    fn coerce_value_is_callable_from_module() {
        // coerce_value is pub(crate) so build_project_var_map (T7) can reuse it.
        assert_eq!(coerce_value(&serde_json::json!("s")), Some("s".to_string()));
        assert_eq!(coerce_value(&serde_json::json!(7)), Some("7".to_string()));
        assert_eq!(coerce_value(&serde_json::json!(true)), Some("true".to_string()));
        assert_eq!(coerce_value(&serde_json::json!({"k": 1})), None);
    }
```

2. **Run (expect FAIL — `from_index_map` not found; `coerce_value` private is fine inside module but `from_index_map` fails compile)**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; export CARGO_NET_RETRY=10; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud/src-tauri && cargo test --lib varmap_from_index_map 2>&1 | tail -15
```

3. **Minimal impl**:
   - Promote `coerce_value` (:61) from `fn` to `pub(crate) fn` (signature unchanged):
```rust
pub(crate) fn coerce_value(v: &Value) -> Option<String> {
```
   - Add `from_index_map` inside `impl VarMap` (after `get`, :53):
```rust
    /// Build a `VarMap` directly from an ordered index map (the inner field is
    /// private; `build_project_var_map` (T7) constructs the layers itself and
    /// hands the finished map in here).
    pub(crate) fn from_index_map(map: IndexMap<String, String>) -> VarMap {
        VarMap(map)
    }
```

4. **Run (expect PASS)**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; export CARGO_NET_RETRY=10; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud/src-tauri && cargo test --lib "varmap_from_index_map varmap coerce_value_is_callable" 2>&1 | tail -10 && cargo build --lib 2>&1 | tail -5
```

5. **Commit**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && cargo fmt && git add -A && git commit -m "feat(vars): VarMap::from_index_map + pub(crate) coerce_value

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## T5 — settings_merge.rs: envelope structs + `merge_with_snapshot` (PURE forward pass)

### Files
- Create: `src-tauri/src/services/settings_merge.rs` (envelope structs + `merge_with_snapshot` + tests)
- Modify: `src-tauri/src/services/mod.rs` (register `pub mod settings_merge;` in alpha order, after `pub mod session_usage_gemini;` / before `pub mod skill;`)
- Test: `src-tauri/src/services/settings_merge.rs` (inline `#[cfg(test)] mod tests`)

### Steps

1. **Write failing test** — create `src-tauri/src/services/settings_merge.rs` with ONLY the envelope structs + `merge_with_snapshot` + this test module (reverse_merge etc. arrive in T6):
```rust
//! Project settings.json MERGE engine (increment 4b-2, Claude only).
//!
//! `merge_with_snapshot` is a PURE forward deep-merge that records a FLAT list of
//! independent `OwnedKey` leaves describing EXACTLY what we wrote and the prior
//! value at each leaf. It intentionally MIRRORS the Object×Object-recurse /
//! else-overwrite shape of `services::provider::live::json_deep_merge` but is
//! SELF-CONTAINED (it does NOT call json_deep_merge — no cross-module coupling).
//! `reverse_merge` (T6) is the pure teardown: a function of (owned_keys + current
//! disk) only — NO re-render, NO env read. There is deliberately NO
//! `collapse_empty_created_ancestors` step: emptied user objects are left as `{}`.

use serde::{Deserialize, Serialize};
use serde_json::Value;

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct PriorLeaf {
    pub present: bool,
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub value: Option<Value>,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct OwnedKey {
    pub path: Vec<String>,
    pub prior: PriorLeaf,
    pub wrote: Value,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct OwnedKeysEnvelope {
    pub v: u32,
    pub keys: Vec<OwnedKey>,
}

pub const OWNED_KEYS_VERSION: u32 = 1;

/// PURE forward deep-merge of `frag` into `user`, recording a flat list of the
/// leaves we wrote (with their prior values) into `out`.
///
/// Mirrors json_deep_merge's recursion shape (Object×Object recurse, else
/// overwrite) but records a snapshot leaf at every overwrite/insert so the
/// teardown can be a pure function of the snapshot. Absent frag key → ONE
/// whole-subtree leaf (`prior.present=false`). Present-as-object → recurse
/// (deeper leaves). Present-as-non-object (or frag-leaf vs user-object) → one
/// leaf `prior.present=true`. Arrays are leaves (WHOLE replace, no element union).
pub fn merge_with_snapshot(
    user: &mut Value,
    frag: &Value,
    path: &mut Vec<String>,
    out: &mut Vec<OwnedKey>,
) {
    match (user, frag) {
        (Value::Object(user_map), Value::Object(frag_map)) => {
            for (key, frag_value) in frag_map {
                path.push(key.clone());
                match user_map.get_mut(key) {
                    Some(uv) if uv.is_object() && frag_value.is_object() => {
                        merge_with_snapshot(uv, frag_value, path, out)
                    }
                    Some(uv) => {
                        out.push(OwnedKey {
                            path: path.clone(),
                            prior: PriorLeaf {
                                present: true,
                                value: Some(uv.clone()),
                            },
                            wrote: frag_value.clone(),
                        });
                        *uv = frag_value.clone();
                    }
                    None => {
                        out.push(OwnedKey {
                            path: path.clone(),
                            prior: PriorLeaf {
                                present: false,
                                value: None,
                            },
                            wrote: frag_value.clone(),
                        });
                        user_map.insert(key.clone(), frag_value.clone());
                    }
                }
                path.pop();
            }
        }
        (user_slot, frag_value) => {
            out.push(OwnedKey {
                path: path.clone(),
                prior: PriorLeaf {
                    present: true,
                    value: Some(user_slot.clone()),
                },
                wrote: frag_value.clone(),
            });
            *user_slot = frag_value.clone();
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use serde_json::json;

    fn run(user: Value, frag: Value) -> (Value, Vec<OwnedKey>) {
        let mut u = user;
        let mut out = Vec::new();
        let mut path = Vec::new();
        merge_with_snapshot(&mut u, &frag, &mut path, &mut out);
        (u, out)
    }

    #[test]
    fn absent_key_is_one_whole_subtree_leaf() {
        let (merged, owned) = run(json!({}), json!({"a": {"b": {"c": 1}}}));
        assert_eq!(merged, json!({"a": {"b": {"c": 1}}}));
        assert_eq!(owned.len(), 1, "absent frag key → exactly ONE leaf");
        assert_eq!(owned[0].path, vec!["a".to_string()]);
        assert!(!owned[0].prior.present);
        assert_eq!(owned[0].prior.value, None);
        assert_eq!(owned[0].wrote, json!({"b": {"c": 1}}));
    }

    #[test]
    fn scalar_overwrite_records_prior_present() {
        let (merged, owned) = run(json!({"k": "old"}), json!({"k": "new"}));
        assert_eq!(merged, json!({"k": "new"}));
        assert_eq!(owned.len(), 1);
        assert_eq!(owned[0].path, vec!["k".to_string()]);
        assert!(owned[0].prior.present);
        assert_eq!(owned[0].prior.value, Some(json!("old")));
        assert_eq!(owned[0].wrote, json!("new"));
    }

    #[test]
    fn nested_recurse_yields_multiple_independent_leaves() {
        // user has permissions.otherKey; frag adds permissions.defaultMode + a new top key.
        let (merged, owned) = run(
            json!({"permissions": {"otherKey": true}}),
            json!({"permissions": {"defaultMode": "x"}, "model": "claude"}),
        );
        assert_eq!(
            merged,
            json!({"permissions": {"otherKey": true, "defaultMode": "x"}, "model": "claude"})
        );
        // two leaves: [permissions, defaultMode] (absent) and [model] (absent).
        assert_eq!(owned.len(), 2);
        let dm = owned
            .iter()
            .find(|k| k.path == vec!["permissions".to_string(), "defaultMode".to_string()])
            .expect("defaultMode leaf");
        assert!(!dm.prior.present);
        assert_eq!(dm.wrote, json!("x"));
        let model = owned
            .iter()
            .find(|k| k.path == vec!["model".to_string()])
            .expect("model leaf");
        assert!(!model.prior.present);
    }

    #[test]
    fn array_is_one_whole_array_leaf() {
        let (merged, owned) = run(
            json!({"permissions": {"allow": ["a", "b"]}}),
            json!({"permissions": {"allow": ["c"]}}),
        );
        assert_eq!(merged, json!({"permissions": {"allow": ["c"]}}));
        assert_eq!(owned.len(), 1, "array = ONE leaf (whole replace, no union)");
        assert_eq!(
            owned[0].path,
            vec!["permissions".to_string(), "allow".to_string()]
        );
        assert!(owned[0].prior.present);
        assert_eq!(owned[0].prior.value, Some(json!(["a", "b"])));
        assert_eq!(owned[0].wrote, json!(["c"]));
    }

    #[test]
    fn frag_object_replaces_user_scalar_as_one_leaf() {
        // user.permissions is a scalar; frag.permissions is an object → NOT both
        // objects → overwrite as one leaf prior.present=true.
        let (merged, owned) = run(
            json!({"permissions": "deny"}),
            json!({"permissions": {"defaultMode": "x"}}),
        );
        assert_eq!(merged, json!({"permissions": {"defaultMode": "x"}}));
        assert_eq!(owned.len(), 1);
        assert_eq!(owned[0].path, vec!["permissions".to_string()]);
        assert!(owned[0].prior.present);
        assert_eq!(owned[0].prior.value, Some(json!("deny")));
    }

    #[test]
    fn non_object_root_overwrite_root_leaf() {
        // both roots non-object-vs-object mismatch at the very top → root leaf path=[].
        let (merged, owned) = run(json!("old-root"), json!({"k": 1}));
        assert_eq!(merged, json!({"k": 1}));
        assert_eq!(owned.len(), 1);
        assert!(owned[0].path.is_empty(), "root leaf has empty path");
        assert!(owned[0].prior.present);
        assert_eq!(owned[0].prior.value, Some(json!("old-root")));
        assert_eq!(owned[0].wrote, json!({"k": 1}));
    }
}
```

2. **Run (expect FAIL — module `settings_merge` not registered)**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; export CARGO_NET_RETRY=10; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud/src-tauri && cargo test --lib settings_merge 2>&1 | tail -15
```

3. **Minimal impl** — register the module in `services/mod.rs` in alpha order: insert after `pub mod session_usage_gemini;` (~:22) and before `pub mod skill;` (~:23) — `settings_merge` (s-e-t) sorts after `session_usage_gemini` (s-e-s). Verify the exact neighbours by reading the file:
```rust
pub mod settings_merge;
```

4. **Run (expect PASS)**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; export CARGO_NET_RETRY=10; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud/src-tauri && cargo test --lib settings_merge::tests 2>&1 | tail -15
```

5. **Commit**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && cargo fmt && git add -A && git commit -m "feat(merge): settings_merge envelope + merge_with_snapshot forward pass

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## T6 — settings_merge.rs: `reverse_merge` + navigate_mut + apply_root_leaf + sort_json_keys_value (PURE, NO collapse)

### Files
- Modify: `src-tauri/src/services/settings_merge.rs` (add `reverse_merge`, `navigate_mut`, `apply_root_leaf`, `sort_json_keys_value`; extend test module)
- Test: `src-tauri/src/services/settings_merge.rs`

### Steps

1. **Write failing test** — append these tests to the `mod tests` in `settings_merge.rs` (before its closing `}`). They include the headline **M3b** (user empty object survives as `{}`), restore/leave/remove, version fail-closed, missing-file Ok, bad-disk leave:
```rust
    use std::fs;
    use tempfile::TempDir;

    fn write_disk(dir: &TempDir, v: &Value) -> std::path::PathBuf {
        let p = dir.path().join("settings.json");
        fs::write(&p, serde_json::to_vec_pretty(v).unwrap()).unwrap();
        p
    }
    fn env_json(keys: Vec<OwnedKey>) -> String {
        serde_json::to_string(&OwnedKeysEnvelope { v: OWNED_KEYS_VERSION, keys }).unwrap()
    }
    fn read_disk(p: &std::path::Path) -> Value {
        serde_json::from_slice(&fs::read(p).unwrap()).unwrap()
    }

    #[test]
    fn reverse_removes_inserted_leaf_when_still_ours() {
        let dir = TempDir::new().unwrap();
        // we inserted [model] (prior absent), wrote "claude"; disk still has it.
        let p = write_disk(&dir, &json!({"model": "claude", "userKept": 1}));
        let env = env_json(vec![OwnedKey {
            path: vec!["model".into()],
            prior: PriorLeaf { present: false, value: None },
            wrote: json!("claude"),
        }]);
        let mut w = Vec::new();
        super::reverse_merge(&p, Some(&env), &mut w).unwrap();
        assert_eq!(read_disk(&p), json!({"userKept": 1}), "our leaf removed, user key survives");
    }

    #[test]
    fn reverse_leaves_user_edited_leaf() {
        let dir = TempDir::new().unwrap();
        // we inserted [model]="claude", but user edited disk to "gpt" → cur!=wrote → leave.
        let p = write_disk(&dir, &json!({"model": "gpt"}));
        let env = env_json(vec![OwnedKey {
            path: vec!["model".into()],
            prior: PriorLeaf { present: false, value: None },
            wrote: json!("claude"),
        }]);
        let mut w = Vec::new();
        super::reverse_merge(&p, Some(&env), &mut w).unwrap();
        assert_eq!(read_disk(&p), json!({"model": "gpt"}), "user edit preserved (M1)");
    }

    #[test]
    fn reverse_restores_overwritten_leaf_to_prior() {
        let dir = TempDir::new().unwrap();
        // we overwrote [model] from "old" to "new"; disk still "new" → restore "old".
        let p = write_disk(&dir, &json!({"model": "new"}));
        let env = env_json(vec![OwnedKey {
            path: vec!["model".into()],
            prior: PriorLeaf { present: true, value: Some(json!("old")) },
            wrote: json!("new"),
        }]);
        let mut w = Vec::new();
        super::reverse_merge(&p, Some(&env), &mut w).unwrap();
        assert_eq!(read_disk(&p), json!({"model": "old"}), "prior restored (M7)");
    }

    #[test]
    fn reverse_user_empty_object_survives_as_empty_object() {
        // M3b (the adversarial-fix headline): user {"permissions":{}} + frag adds
        // [permissions, defaultMode] (prior absent). Detach removes defaultMode; the
        // user's `permissions` MUST survive as {} — NO collapse of created ancestors.
        let dir = TempDir::new().unwrap();
        let p = write_disk(&dir, &json!({"permissions": {"defaultMode": "x"}}));
        let env = env_json(vec![OwnedKey {
            path: vec!["permissions".into(), "defaultMode".into()],
            prior: PriorLeaf { present: false, value: None },
            wrote: json!("x"),
        }]);
        let mut w = Vec::new();
        super::reverse_merge(&p, Some(&env), &mut w).unwrap();
        assert_eq!(
            read_disk(&p),
            json!({"permissions": {}}),
            "user's permissions object MUST survive as {} (no collapse)"
        );
    }

    #[test]
    fn reverse_partial_nested_keeps_sibling_user_key() {
        // M3: frag {permissions:{defaultMode}} into user {permissions:{otherKey}};
        // reverse removes defaultMode, otherKey survives, permissions NOT removed.
        let dir = TempDir::new().unwrap();
        let p = write_disk(&dir, &json!({"permissions": {"otherKey": true, "defaultMode": "x"}}));
        let env = env_json(vec![OwnedKey {
            path: vec!["permissions".into(), "defaultMode".into()],
            prior: PriorLeaf { present: false, value: None },
            wrote: json!("x"),
        }]);
        let mut w = Vec::new();
        super::reverse_merge(&p, Some(&env), &mut w).unwrap();
        assert_eq!(read_disk(&p), json!({"permissions": {"otherKey": true}}));
    }

    #[test]
    fn reverse_whole_array_restores_prior_array() {
        // M5: we replaced [permissions, allow] from ["a","b"] to ["c"]; restore.
        let dir = TempDir::new().unwrap();
        let p = write_disk(&dir, &json!({"permissions": {"allow": ["c"]}}));
        let env = env_json(vec![OwnedKey {
            path: vec!["permissions".into(), "allow".into()],
            prior: PriorLeaf { present: true, value: Some(json!(["a", "b"])) },
            wrote: json!(["c"]),
        }]);
        let mut w = Vec::new();
        super::reverse_merge(&p, Some(&env), &mut w).unwrap();
        assert_eq!(read_disk(&p), json!({"permissions": {"allow": ["a", "b"]}}));
    }

    #[test]
    fn reverse_fail_closed_on_unknown_version_or_bad_input() {
        let dir = TempDir::new().unwrap();
        // M8a: version 99 → leave file untouched, Ok, warn.
        let p = write_disk(&dir, &json!({"model": "claude"}));
        let mut w = Vec::new();
        super::reverse_merge(&p, Some(r#"{"v":99,"keys":[]}"#), &mut w).unwrap();
        assert_eq!(read_disk(&p), json!({"model": "claude"}), "v99 leaves file");
        assert!(w.iter().any(|m| m.contains("version 99")));

        // M8b: not JSON → leave, Ok, warn.
        let mut w2 = Vec::new();
        super::reverse_merge(&p, Some("not json"), &mut w2).unwrap();
        assert_eq!(read_disk(&p), json!({"model": "claude"}));
        assert!(!w2.is_empty());

        // M8c: None / empty → Ok, no warn, untouched.
        let mut w3 = Vec::new();
        super::reverse_merge(&p, None, &mut w3).unwrap();
        super::reverse_merge(&p, Some(""), &mut w3).unwrap();
        assert!(w3.is_empty());
        assert_eq!(read_disk(&p), json!({"model": "claude"}));
    }

    #[test]
    fn reverse_missing_file_is_ok() {
        let dir = TempDir::new().unwrap();
        let p = dir.path().join("settings.json"); // does not exist
        let env = env_json(vec![OwnedKey {
            path: vec!["model".into()],
            prior: PriorLeaf { present: false, value: None },
            wrote: json!("claude"),
        }]);
        let mut w = Vec::new();
        super::reverse_merge(&p, Some(&env), &mut w).unwrap();
        assert!(!p.exists(), "missing file stays missing");
    }

    #[test]
    fn reverse_bad_disk_json_leaves_file() {
        let dir = TempDir::new().unwrap();
        let p = dir.path().join("settings.json");
        fs::write(&p, b"{ this is : not json").unwrap();
        let env = env_json(vec![OwnedKey {
            path: vec!["model".into()],
            prior: PriorLeaf { present: false, value: None },
            wrote: json!("claude"),
        }]);
        let mut w = Vec::new();
        super::reverse_merge(&p, Some(&env), &mut w).unwrap();
        assert_eq!(fs::read(&p).unwrap(), b"{ this is : not json", "byte-identical");
        assert!(w.iter().any(|m| m.contains("not valid JSON")));
    }

    #[test]
    fn sort_json_keys_value_sorts_recursively() {
        let v = json!({"b": 1, "a": {"z": 2, "y": 3}});
        let sorted = super::sort_json_keys_value(&v);
        assert_eq!(
            serde_json::to_string(&sorted).unwrap(),
            r#"{"a":{"y":3,"z":2},"b":1}"#
        );
    }
```

2. **Run (expect FAIL — `reverse_merge` / `sort_json_keys_value` not found)**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; export CARGO_NET_RETRY=10; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud/src-tauri && cargo test --lib settings_merge::tests::reverse 2>&1 | tail -15
```

3. **Minimal impl** — add `use std::path::Path;` and `use crate::error::AppError;` to the imports at the top of `settings_merge.rs`, then add these functions after `merge_with_snapshot`:
```rust
/// PURE teardown: a function of (owned_keys + current disk) ONLY — NO re-render,
/// NO env read. Per-leaf `cur==wrote` gate. There is deliberately NO
/// collapse_empty_created_ancestors (adversarial fix #1): emptied user objects
/// are left as `{}`. FAIL-CLOSED on unknown envelope version / unparseable
/// owned_keys / invalid on-disk JSON (warn, leave the file, return Ok).
pub fn reverse_merge(
    file_path: &Path,
    owned_json: Option<&str>,
    warnings: &mut Vec<String>,
) -> Result<(), AppError> {
    let Some(text) = owned_json else {
        return Ok(());
    };
    if text.trim().is_empty() {
        return Ok(());
    }
    let env: OwnedKeysEnvelope = match serde_json::from_str(text) {
        Ok(e) => e,
        Err(e) => {
            warnings.push(format!(
                "settings.json owned_keys unparseable, leaving untouched ({file_path:?}): {e}"
            ));
            return Ok(());
        }
    };
    if env.v != OWNED_KEYS_VERSION {
        warnings.push(format!(
            "settings.json owned_keys version {} unsupported (expected {OWNED_KEYS_VERSION}); leaving untouched ({file_path:?})",
            env.v
        ));
        return Ok(()); // FAIL-CLOSED
    }
    if !file_path.exists() {
        return Ok(());
    }
    let disk = std::fs::read(file_path).map_err(|e| AppError::io(file_path, e))?;
    let mut root: Value = match serde_json::from_slice(&disk) {
        Ok(v) => v,
        Err(e) => {
            warnings.push(format!(
                "settings.json on disk not valid JSON, leaving untouched ({file_path:?}): {e}"
            ));
            return Ok(());
        }
    };
    // Independent leaves; order is irrelevant for correctness now that collapse is
    // removed, but process deepest-first for determinism.
    let mut keys = env.keys.clone();
    keys.sort_by(|a, b| b.path.len().cmp(&a.path.len()));
    for k in &keys {
        if k.path.is_empty() {
            apply_root_leaf(&mut root, k);
            continue;
        }
        let (parents, leaf) = k.path.split_at(k.path.len() - 1);
        let leaf = &leaf[0];
        let Some(parent) = navigate_mut(&mut root, parents) else {
            continue;
        };
        let Some(pm) = parent.as_object_mut() else {
            continue;
        };
        match (k.prior.present, pm.get(leaf)) {
            // we inserted it and it is still ours → remove (leaves emptied user
            // objects as {} — NO collapse).
            (false, Some(cur)) if *cur == k.wrote => {
                pm.remove(leaf);
            }
            // user edited/removed our inserted leaf → leave it.
            (false, _) => {}
            // we overwrote it and it is still ours → restore the original value.
            (true, Some(cur)) if *cur == k.wrote => {
                pm.insert(leaf.clone(), k.prior.value.clone().unwrap_or(Value::Null));
            }
            // user re-edited → leave it.
            (true, _) => {}
        }
    }
    // NO collapse_empty_created_ancestors (removed per adversarial fix #1).
    let bytes = serde_json::to_vec_pretty(&sort_json_keys_value(&root))
        .map_err(|e| AppError::Message(format!("serialize settings.json: {e}")))?;
    crate::config::atomic_write(file_path, &bytes)?;
    Ok(())
}

fn navigate_mut<'a>(root: &'a mut Value, segs: &[String]) -> Option<&'a mut Value> {
    let mut cur = root;
    for s in segs {
        cur = cur.as_object_mut()?.get_mut(s)?;
    }
    Some(cur)
}

fn apply_root_leaf(root: &mut Value, k: &OwnedKey) {
    if *root == k.wrote {
        match (k.prior.present, &k.prior.value) {
            (true, Some(v)) => *root = v.clone(),
            _ => *root = Value::Object(serde_json::Map::new()),
        }
    }
}

/// Deterministic key-sorted clone of a JSON value (config.rs's `sort_json_keys`
/// is a private free fn; this mirrors its recursion shape locally).
pub fn sort_json_keys_value(value: &Value) -> Value {
    match value {
        Value::Object(map) => {
            let mut sorted = serde_json::Map::new();
            let mut keys: Vec<_> = map.keys().collect();
            keys.sort();
            for key in keys {
                sorted.insert(key.clone(), sort_json_keys_value(&map[key]));
            }
            Value::Object(sorted)
        }
        Value::Array(arr) => Value::Array(arr.iter().map(sort_json_keys_value).collect()),
        other => other.clone(),
    }
}
```

4. **Run (expect PASS)**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; export CARGO_NET_RETRY=10; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud/src-tauri && cargo test --lib settings_merge::tests 2>&1 | tail -15
```

5. **Commit**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && cargo fmt && git add -A && git commit -m "feat(merge): reverse_merge teardown (per-leaf, no collapse) + sort helper

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## T7 — `build_project_var_map` (precedence: spec.vars > provider env > process env; NO global profile leak)

### Files
- Modify: `src-tauri/src/services/project_apply.rs` (add `build_project_var_map` associated fn in `impl ProjectApplyService`, after `prune_orphan_project_channels` :317; add tests in the inline test module)
- Test: `src-tauri/src/services/project_apply.rs`

### Steps

1. **Write failing test** — append to the test `mod tests` in `project_apply.rs` (before its closing `}` at :1004). Mirrors the precedence test in `profile_vars.rs` but asserts the GLOBAL profile's vars do NOT leak:
```rust
    #[test]
    #[serial]
    fn build_project_var_map_precedence_spec_over_provider_over_process() {
        let home = TempHome::new();
        crate::settings::reload_settings().ok();
        std::env::set_var("ANTHROPIC_SHARED", "from_process");
        std::env::set_var("AGENTHUB_ONLY_PROCESS", "process_only");
        std::env::set_var("RANDOM_HOST_SECRET", "leak");

        let db = Arc::new(Database::memory().expect("db"));
        let app = AppType::Claude;

        let provider = crate::provider::Provider::with_id(
            "prov1".to_string(),
            "Prov 1".to_string(),
            serde_json::json!({
                "env": {
                    "ANTHROPIC_SHARED": "from_provider",
                    "ANTHROPIC_PROVIDER_KEY": "pk"
                }
            }),
            None,
        );
        db.save_provider(app.as_str(), &provider).expect("save provider");
        db.set_current_provider(app.as_str(), "prov1").expect("set current");

        let (mut proj, _canon) = project_at(home.home(), "varproj", ProfileContent::default());
        proj.spec.vars.insert(
            "ANTHROPIC_SHARED".to_string(),
            serde_json::Value::String("from_project".to_string()),
        );
        db.save_project(&proj).expect("save");

        let app_arc = AppType::Claude;
        let stored = db.get_project(&proj.id).unwrap().unwrap();
        let map =
            ProjectApplyService::build_project_var_map(&db, &app_arc, &stored).expect("build map");

        assert_eq!(map.get("ANTHROPIC_SHARED"), Some("from_project"), "project.spec.vars wins");
        assert_eq!(map.get("ANTHROPIC_PROVIDER_KEY"), Some("pk"), "provider env contributes");
        assert_eq!(map.get("AGENTHUB_ONLY_PROCESS"), Some("process_only"), "allowlisted process env");
        assert_eq!(map.get("RANDOM_HOST_SECRET"), None, "non-allowlisted env filtered");

        std::env::remove_var("ANTHROPIC_SHARED");
        std::env::remove_var("AGENTHUB_ONLY_PROCESS");
        std::env::remove_var("RANDOM_HOST_SECRET");
    }

    #[test]
    #[serial]
    fn build_project_var_map_does_not_leak_global_profile_vars() {
        // The global ACTIVE profile may carry spec.vars; build_project_var_map MUST
        // NOT include them (it never calls build_var_map). Only the PROJECT's own
        // spec.vars (+ provider env + allowlisted process env) feed the map.
        let home = TempHome::new();
        crate::settings::reload_settings().ok();
        let db = Arc::new(Database::memory().expect("db"));
        let app = AppType::Claude;

        // an active profile with a var that MUST NOT leak.
        let mut pvars = serde_json::Map::new();
        pvars.insert(
            "ANTHROPIC_PROFILE_ONLY".to_string(),
            serde_json::Value::String("LEAKED".to_string()),
        );
        let profile = crate::app_config::Profile {
            id: "local:claude:Active".into(),
            app_type: "claude".into(),
            name: "Active".into(),
            description: None,
            is_active: true,
            current_provider_id: None,
            spec: crate::app_config::ProfileSpec { content: Default::default(), vars: pvars },
            sort_index: 0,
            created_at: 0,
        };
        db.save_profile(&profile).expect("save profile");

        let (proj, _canon) = project_at(home.home(), "noleakproj", ProfileContent::default());
        db.save_project(&proj).expect("save");
        let stored = db.get_project(&proj.id).unwrap().unwrap();
        let map = ProjectApplyService::build_project_var_map(&db, &app, &stored).expect("map");
        assert_eq!(map.get("ANTHROPIC_PROFILE_ONLY"), None, "global profile vars must NOT leak");
    }
```

2. **Run (expect FAIL — no associated fn `build_project_var_map`)**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; export CARGO_NET_RETRY=10; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud/src-tauri && cargo test --lib build_project_var_map 2>&1 | tail -15
```

3. **Minimal impl** — in `impl ProjectApplyService` (after `prune_orphan_project_channels`, :317). Add `use indexmap::IndexMap;` only inside the fn via a fully-qualified path to avoid a top-of-file import churn — actually use a local `use`:
```rust
    /// Build the layered `${VAR}` map for a PROJECT (contract g). Layers low→high:
    /// 1. allowlisted process env (profile_vars::ENV_ALLOWLIST_PREFIXES),
    /// 2. active provider settings_config.env (get_effective_current_provider),
    /// 3. project.spec.vars (TOP). Does NOT call profile_vars::build_var_map — that
    /// would inject the GLOBAL active profile's vars, which must NOT leak into a
    /// project render. reverse_merge does NOT re-render, so a value change between
    /// apply and detach cannot defeat teardown (teardown is a pure fn of the
    /// stored snapshot).
    pub fn build_project_var_map(
        db: &crate::database::Database,
        app_type: &AppType,
        project: &crate::app_config::Project,
    ) -> Result<crate::services::profile_vars::VarMap, AppError> {
        use indexmap::IndexMap;
        let mut map: IndexMap<String, String> = IndexMap::new();

        // Layer 1: allowlisted process env.
        for (k, v) in std::env::vars() {
            if crate::services::profile_vars::ENV_ALLOWLIST_PREFIXES
                .iter()
                .any(|prefix| k.starts_with(prefix))
            {
                map.insert(k, v);
            }
        }

        // Layer 2: active provider env (if any).
        if let Some(provider_id) =
            crate::settings::get_effective_current_provider(db, app_type)?
        {
            if let Some(provider) = db.get_provider_by_id(&provider_id, app_type.as_str())? {
                if let Some(env_obj) = provider
                    .settings_config
                    .get("env")
                    .and_then(|v| v.as_object())
                {
                    for (k, v) in env_obj {
                        if let Some(value) = crate::services::profile_vars::coerce_value(v) {
                            map.insert(k.clone(), value);
                        }
                    }
                }
            }
        }

        // Layer 3: project spec.vars (highest precedence).
        for (k, v) in &project.spec.vars {
            if let Some(value) = crate::services::profile_vars::coerce_value(v) {
                map.insert(k.clone(), value);
            }
        }

        Ok(crate::services::profile_vars::VarMap::from_index_map(map))
    }
```

4. **Run (expect PASS)**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; export CARGO_NET_RETRY=10; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud/src-tauri && cargo test --lib build_project_var_map 2>&1 | tail -15
```

5. **Commit**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && cargo fmt && git add -A && git commit -m "feat(project): build_project_var_map (spec.vars>provider>env, no profile leak)

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## T8 — apply() settings_merge materialize block + `Self::row` 6-arg + 4 caller updates

### Files
- Modify: `src-tauri/src/services/project_apply.rs` (`Self::row` :276-294; the 4 content callers :130-136, :155-161, :183-189, :206-212; the temporary `owned_keys: None` added to `Self::row` in T2; new materialize block after the CLAUDE.md block :219, before `Ok(result)` :221; tests in the inline module)
- Test: `src-tauri/src/services/project_apply.rs`

### Steps

1. **Write failing test** — append to the test `mod tests` in `project_apply.rs`. Covers merge preserves unrelated user keys, bad fragment warn+untouched+no row, the recorded envelope is parseable, malformed disk skip (M11):
```rust
    fn merge_proj(home: &Path, sub: &str, frag: &str) -> (Project, std::path::PathBuf) {
        let (mut proj, canon) = project_at(home, sub, ProfileContent::default());
        proj.spec.dotfiles.settings = frag.into();
        (proj, canon)
    }

    #[test]
    #[serial]
    fn apply_merges_settings_and_preserves_unrelated_user_keys() {
        let home = TempHome::new();
        crate::settings::reload_settings().ok();
        let db = Arc::new(Database::memory().expect("db"));
        let state = AppState::new(db.clone());
        let (proj, canon) = merge_proj(home.home(), "merge-a", r#"{"model": "claude-x"}"#);
        db.save_project(&proj).expect("save");

        // user already has a settings.json with their OWN key.
        let target = canon.join(".claude").join("settings.json");
        std::fs::create_dir_all(target.parent().unwrap()).unwrap();
        std::fs::write(&target, r#"{"userKept": true}"#).unwrap();

        let res = ProjectApplyService::apply(&state, &proj.id).expect("apply");
        assert!(res.warnings.is_empty(), "no warnings: {:?}", res.warnings);

        let disk: serde_json::Value =
            serde_json::from_slice(&std::fs::read(&target).unwrap()).unwrap();
        assert_eq!(disk["model"], serde_json::json!("claude-x"), "frag merged");
        assert_eq!(disk["userKept"], serde_json::json!(true), "user key survives");

        // exactly one settings_merge row with a parseable v1 envelope, content_hash None.
        let chan = format!("project:{}", canon.to_string_lossy());
        let rows = db.get_manifest_for_channel(&chan).unwrap();
        let m: Vec<_> = rows.iter().filter(|r| r.kind == "settings_merge").collect();
        assert_eq!(m.len(), 1);
        assert_eq!(m[0].content_hash, None, "merge rows carry NO content_hash");
        let env: crate::services::settings_merge::OwnedKeysEnvelope =
            serde_json::from_str(m[0].owned_keys.as_deref().expect("owned_keys")).expect("parse env");
        assert_eq!(env.v, crate::services::settings_merge::OWNED_KEYS_VERSION);
        assert!(env.keys.iter().any(|k| k.path == vec!["model".to_string()]));
    }

    #[test]
    #[serial]
    fn apply_bad_fragment_warns_and_writes_no_file_no_row() {
        let home = TempHome::new();
        crate::settings::reload_settings().ok();
        let db = Arc::new(Database::memory().expect("db"));
        let state = AppState::new(db.clone());
        // fragment is not valid JSON even after render → warn, no write, no row.
        let (proj, canon) = merge_proj(home.home(), "merge-bad", r#"{ not: json"#);
        db.save_project(&proj).expect("save");
        let res = ProjectApplyService::apply(&state, &proj.id).expect("apply");
        assert!(res.warnings.iter().any(|w| w.contains("invalid JSON after render")));
        assert!(!canon.join(".claude").join("settings.json").exists(), "no file written");
        let chan = format!("project:{}", canon.to_string_lossy());
        assert!(
            !db.get_manifest_for_channel(&chan).unwrap().iter().any(|r| r.kind == "settings_merge"),
            "no settings_merge row for a bad fragment"
        );
    }

    #[test]
    #[serial]
    fn apply_malformed_disk_skips_merge_byte_identical_no_row() {
        // M11: disk settings.json is invalid JSON → skip the merge, leave bytes
        // identical, warn, NO row.
        let home = TempHome::new();
        crate::settings::reload_settings().ok();
        let db = Arc::new(Database::memory().expect("db"));
        let state = AppState::new(db.clone());
        let (proj, canon) = merge_proj(home.home(), "merge-mal", r#"{"model": "x"}"#);
        db.save_project(&proj).expect("save");
        let target = canon.join(".claude").join("settings.json");
        std::fs::create_dir_all(target.parent().unwrap()).unwrap();
        std::fs::write(&target, b"{ broken : json").unwrap();

        let res = ProjectApplyService::apply(&state, &proj.id).expect("apply");
        assert!(res.warnings.iter().any(|w| w.contains("not a JSON value")));
        assert_eq!(std::fs::read(&target).unwrap(), b"{ broken : json", "byte-identical");
        let chan = format!("project:{}", canon.to_string_lossy());
        assert!(
            !db.get_manifest_for_channel(&chan).unwrap().iter().any(|r| r.kind == "settings_merge"),
            "no row when disk skip"
        );
    }

    #[test]
    #[serial]
    fn apply_renders_vars_in_settings_fragment() {
        let home = TempHome::new();
        crate::settings::reload_settings().ok();
        let db = Arc::new(Database::memory().expect("db"));
        let state = AppState::new(db.clone());
        let (mut proj, canon) =
            merge_proj(home.home(), "merge-var", r#"{"model": "${MODEL_NAME}"}"#);
        proj.spec.vars.insert(
            "MODEL_NAME".to_string(),
            serde_json::Value::String("claude-from-var".to_string()),
        );
        db.save_project(&proj).expect("save");
        ProjectApplyService::apply(&state, &proj.id).expect("apply");
        let target = canon.join(".claude").join("settings.json");
        let disk: serde_json::Value =
            serde_json::from_slice(&std::fs::read(&target).unwrap()).unwrap();
        assert_eq!(disk["model"], serde_json::json!("claude-from-var"), "${VAR} rendered");
    }
```

2. **Run (expect FAIL — `dotfiles.settings` field missing AND no materialize block)**. NOTE: this task (T8 step 3) ADDS the `#[serde(default)] pub settings: String,` field to `ProjectDotfiles` PERMANENTLY (it is the field's owning task — needed so these materialize tests compile). T11 does NOT re-add it; T11 only adds the storage serde round-trip tests against the field T8 introduced. (The §5 order is T8-before-T11; no reorder is required, and T11 is a tests-only task with no production code.)
```bash
export PATH="$HOME/.cargo/bin:$PATH"; export CARGO_NET_RETRY=10; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud/src-tauri && cargo test --lib apply_merges_settings 2>&1 | tail -20
```

3. **Minimal impl**:
   - Add the `settings` field to `ProjectDotfiles` (`app_config.rs` :301-305) now (storage tests are T11):
```rust
pub struct ProjectDotfiles {
    /// Literal project-root CLAUDE.md content (NO ${VAR} rendering). Empty = none.
    #[serde(default)]
    pub claude_md: String,
    /// settings.json fragment (deep-MERGED into <project>/.claude/settings.json;
    /// supports ${VAR}). Empty = none. Arrays (permissions.allow/deny, hooks) are
    /// WHOLE-ARRAY replace (project wins; detach restores prior array). No union.
    #[serde(default)]
    pub settings: String,
}
```
   Adding this field is backward-compatible for the many `dotfiles: Default::default()` callers (a `String` defaults to empty). But there is exactly ONE explicit `ProjectDotfiles { claude_md: ... }` struct literal — `tests/project_apply_e2e.rs:81-83` — which MUST gain `settings: String::new(),` or it will fail to compile. Grep `ProjectDotfiles {` across `src-tauri/src/` AND `src-tauri/tests/` to confirm (only that one literal exists today); fix it now so the integration target compiles, then T13 adds the new e2e test.
   - `Self::row` (project_apply.rs :276-294) — 6-arg signature; REMOVE the temporary `owned_keys: None,` from T2 and wire the params:
```rust
    fn row(
        channel: &str,
        project_id: &str,
        target: &Path,
        kind: &str,
        content_hash: Option<&str>,
        owned_keys: Option<String>,
    ) -> ManifestEntry {
        ManifestEntry {
            id: 0,
            channel: channel.to_string(),
            profile_id: None,
            project_id: Some(project_id.to_string()),
            app_type: AppType::Claude.as_str().to_string(),
            target_path: target.to_string_lossy().to_string(),
            kind: kind.to_string(),
            content_hash: content_hash.map(|h| h.to_string()),
            owned_keys,
            created_at: chrono::Utc::now().timestamp(),
        }
    }
```
   - Update the 4 content callers to `(Some(&hash), None)`:
     - command (:130-136): `Self::row(&channel, &project.id, &target, "command", Some(&hash), None)`
     - agent (:155-161): `Self::row(&channel, &project.id, &target, "agent", Some(&hash), None)`
     - skill (:183-189): `Self::row(&channel, &project.id, &dest, "skill", Some(&hash), None)`
     - project_memory (:206-212): `Self::row(&channel, &project.id, &target, "project_memory", Some(&hash), None)`
   - Insert the materialize block AFTER the CLAUDE.md block (after :219) and BEFORE `Ok(result)` (:221):
```rust
        // ---- project settings.json (deep MERGE, ${VAR}; 4b-2, Claude only) ----
        // The pre-delete sweep already ran reverse_merge for our prior row
        // (contract d), so the on-disk settings.json is the USER baseline here.
        let settings_frag = &project.spec.dotfiles.settings;
        if !settings_frag.is_empty() {
            let target = base.settings_file();
            let var_map = Self::build_project_var_map(&state.db, &app, &project)?;
            let mut warns = Vec::new();
            let rendered = crate::services::profile_vars::substitute_vars(
                settings_frag,
                &var_map,
                /* json_escape = */ true,
                &mut warns,
            );
            for w in warns {
                result
                    .warnings
                    .push(format!("project settings.json render: {w}"));
            }
            match serde_json::from_str::<serde_json::Value>(&rendered) {
                Ok(frag) => {
                    // Load current disk as Option<Value> (None == skip-the-merge
                    // sentinel; NOT Value::Null — a valid on-disk `null` must not be
                    // misclassified, adversarial fix #3). absent file → Some({});
                    // present+valid → Some(v); present+invalid → None + warn.
                    let user_opt: Option<serde_json::Value> = if target.exists() {
                        match std::fs::read(&target)
                            .ok()
                            .and_then(|b| serde_json::from_slice(&b).ok())
                        {
                            Some(v) => Some(v),
                            None => {
                                result.warnings.push(format!(
                                    "project settings.json on disk is not a JSON value; skipping merge: {}",
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
                        let bytes = serde_json::to_vec_pretty(
                            &crate::services::settings_merge::sort_json_keys_value(&user),
                        )
                        .map_err(|e| {
                            AppError::Message(format!("serialize settings.json: {e}"))
                        })?;
                        atomic_write(&target, &bytes)?;
                        let env = crate::services::settings_merge::OwnedKeysEnvelope {
                            v: crate::services::settings_merge::OWNED_KEYS_VERSION,
                            keys: owned,
                        };
                        let owned_json = serde_json::to_string(&env).map_err(|e| {
                            AppError::Message(format!("serialize owned_keys: {e}"))
                        })?;
                        state.db.record_manifest_entry(&Self::row(
                            &channel,
                            &project.id,
                            &target,
                            "settings_merge",
                            None,
                            Some(owned_json),
                        ))?;
                    }
                }
                Err(e) => result.warnings.push(format!(
                    "project settings.json fragment invalid JSON after render, skipping: {e}"
                )),
            }
        }

        Ok(result)
```

4. **Run (expect PASS) + full-target compile** (catches any remaining `Self::row` arity mismatch in tests):
```bash
export PATH="$HOME/.cargo/bin:$PATH"; export CARGO_NET_RETRY=10; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud/src-tauri && cargo test --lib "apply_merges_settings apply_bad_fragment apply_malformed_disk apply_renders_vars" 2>&1 | tail -15 && cargo test --no-run --all-targets 2>&1 | tail -10
```

5. **Commit**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && cargo fmt && git add -A && git commit -m "feat(project): apply settings.json merge + Self::row 6-arg (content_hash/owned_keys)

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## T9 — CRITICAL apply pre-delete sweep arm (reverse_merge BEFORE the else) + idempotency / pre-reapply baseline

### Files
- Modify: `src-tauri/src/services/project_apply.rs` (pre-delete owned-row loop :88-107 — insert the `settings_merge` arm; tests in the inline module)
- Test: `src-tauri/src/services/project_apply.rs`

### Steps

1. **Write failing test** — append to the test `mod tests`. Covers M6: double-apply idempotency + the pre-reapply baseline (after a 2nd apply, the recorded envelope's prior for an OVERWRITTEN leaf must equal the TRUE original, not our 1st write):
```rust
    #[test]
    #[serial]
    fn reapply_settings_is_idempotent_and_prior_is_true_user_baseline() {
        let home = TempHome::new();
        crate::settings::reload_settings().ok();
        let db = Arc::new(Database::memory().expect("db"));
        let state = AppState::new(db.clone());
        // frag overwrites a key the USER already has → prior must be the user's value.
        let (proj, canon) = merge_proj(home.home(), "merge-reapply", r#"{"model": "ours"}"#);
        db.save_project(&proj).expect("save");
        let target = canon.join(".claude").join("settings.json");
        std::fs::create_dir_all(target.parent().unwrap()).unwrap();
        std::fs::write(&target, r#"{"model": "USER_ORIGINAL"}"#).unwrap();

        ProjectApplyService::apply(&state, &proj.id).expect("apply 1");
        ProjectApplyService::apply(&state, &proj.id).expect("apply 2");

        // exactly ONE settings_merge row (pre-delete reverse_merge ran first, then
        // re-merge recorded a fresh single row).
        let chan = format!("project:{}", canon.to_string_lossy());
        let rows = db.get_manifest_for_channel(&chan).unwrap();
        let m: Vec<_> = rows.iter().filter(|r| r.kind == "settings_merge").collect();
        assert_eq!(m.len(), 1, "double-apply stays idempotent (one settings_merge row)");

        // the recorded prior for [model] must be the TRUE user original — the
        // pre-delete reverse restored "USER_ORIGINAL" before the re-merge snapshotted.
        let env: crate::services::settings_merge::OwnedKeysEnvelope =
            serde_json::from_str(m[0].owned_keys.as_deref().unwrap()).unwrap();
        let model_leaf = env
            .keys
            .iter()
            .find(|k| k.path == vec!["model".to_string()])
            .expect("model leaf");
        assert!(model_leaf.prior.present);
        assert_eq!(
            model_leaf.prior.value,
            Some(serde_json::json!("USER_ORIGINAL")),
            "prior must be the TRUE user baseline, NOT our prior write (contract d)"
        );

        // disk reflects our value after re-apply.
        let disk: serde_json::Value =
            serde_json::from_slice(&std::fs::read(&target).unwrap()).unwrap();
        assert_eq!(disk["model"], serde_json::json!("ours"));
    }
```

2. **Run (expect FAIL — without the arm, the pre-delete else hits `remove_whole_file_if_owned(.., None)` which is a no-op for merge rows, so the prior on apply 2 would be "ours" (our 1st write), not "USER_ORIGINAL")**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; export CARGO_NET_RETRY=10; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud/src-tauri && cargo test --lib reapply_settings_is_idempotent 2>&1 | tail -20
```

3. **Minimal impl** — in the pre-delete owned-row loop (project_apply.rs :88-107), insert the `settings_merge` arm BETWEEN the `if r.kind == "skill"` block and the catch-all `else`:
```rust
        for r in &ours {
            if r.kind == "skill" {
                // dir-aware, content-hash-safe removal so re-apply cleans old skill copies.
                if let Some(parent) = Path::new(&r.target_path).parent() {
                    let dir_name = Path::new(&r.target_path)
                        .file_name()
                        .map(|n| n.to_string_lossy().to_string())
                        .unwrap_or_default();
                    crate::services::SkillService::remove_from_project_dir(&dir_name, parent, &app)
                        .map_err(|e| {
                            AppError::Message(format!("project skill remove failed: {e}"))
                        })?;
                }
            } else if r.kind == "settings_merge" {
                // CRITICAL (contract c): never let a merge row hit the else
                // (remove_whole_file_if_owned → whole-file delete of the user's
                // settings.json). This reverse IS the contract-(d) pre-reapply
                // ordering: undo our prior merge BEFORE the materialize block (after
                // the CLAUDE.md block, below) re-reads disk + re-merges, so our prior
                // write never becomes the new "user baseline".
                crate::services::settings_merge::reverse_merge(
                    std::path::Path::new(&r.target_path),
                    r.owned_keys.as_deref(),
                    &mut result.warnings,
                )?;
            } else {
                crate::services::profile_render::remove_whole_file_if_owned(
                    &r.target_path,
                    r.content_hash.as_deref(),
                )?;
            }
        }
```

4. **Run (expect PASS)**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; export CARGO_NET_RETRY=10; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud/src-tauri && cargo test --lib reapply_settings_is_idempotent 2>&1 | tail -15
```

5. **Commit**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && cargo fmt && git add -A && git commit -m "feat(project): apply pre-delete settings_merge reverse arm (contract c+d)

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## T10 — CRITICAL detach arm + M10 whole-file-deletion regression + M7 restore + M9 unrelated-key-survives

### Files
- Modify: `src-tauri/src/services/project_apply.rs` (detach loop :237-264 — insert the `settings_merge` arm; tests in the inline module)
- Test: `src-tauri/src/services/project_apply.rs`

### Steps

1. **Write failing test** — append the M10 regression (the headline), M7 restore, and M9:
```rust
    #[test]
    #[serial]
    fn m10_settings_file_is_never_whole_deleted_on_reapply_or_detach() {
        // M10 (CRITICAL): a bound project with a settings fragment + CLAUDE.md.
        // After apply: a settings_merge row exists. (a) re-apply: settings.json
        // STILL EXISTS with the user key. (b) detach: settings.json STILL EXISTS
        // (user keys reversed) — the merge row must NOT fall through the else and
        // get whole-file deleted.
        let home = TempHome::new();
        crate::settings::reload_settings().ok();
        let db = Arc::new(Database::memory().expect("db"));
        let state = AppState::new(db.clone());
        let (mut proj, canon) = merge_proj(home.home(), "m10", r#"{"model": "ours"}"#);
        proj.spec.dotfiles.claude_md = "# mem\n".into();
        db.save_project(&proj).expect("save");

        let target = canon.join(".claude").join("settings.json");
        std::fs::create_dir_all(target.parent().unwrap()).unwrap();
        std::fs::write(&target, r#"{"userKept": true}"#).unwrap();

        ProjectApplyService::apply(&state, &proj.id).expect("apply");
        let chan = format!("project:{}", canon.to_string_lossy());
        assert!(
            db.get_manifest_for_channel(&chan).unwrap().iter().any(|r| r.kind == "settings_merge"),
            "a settings_merge row must exist after apply"
        );

        // (a) re-apply: file STILL exists with the user key.
        ProjectApplyService::apply(&state, &proj.id).expect("re-apply");
        assert!(target.exists(), "settings.json must NOT be whole-deleted by re-apply");
        let disk_a: serde_json::Value =
            serde_json::from_slice(&std::fs::read(&target).unwrap()).unwrap();
        assert_eq!(disk_a["userKept"], serde_json::json!(true), "user key survives re-apply");

        // (b) detach: file STILL exists; our [model] leaf reversed; user key intact.
        ProjectApplyService::detach(&state, &proj.id).expect("detach");
        assert!(target.exists(), "settings.json must STILL EXIST after detach (M10)");
        let disk_b: serde_json::Value =
            serde_json::from_slice(&std::fs::read(&target).unwrap()).unwrap();
        assert_eq!(disk_b["userKept"], serde_json::json!(true), "user key survives detach");
        assert!(disk_b.get("model").is_none(), "our inserted leaf removed on detach (M7)");
    }

    #[test]
    #[serial]
    fn detach_restores_overwritten_user_key_and_keeps_new_user_key() {
        // M7 + M9: frag overwrote user's existing [model]; user later added a brand
        // new key never in the frag. Detach restores [model] to the user original
        // and leaves the brand-new key untouched.
        let home = TempHome::new();
        crate::settings::reload_settings().ok();
        let db = Arc::new(Database::memory().expect("db"));
        let state = AppState::new(db.clone());
        let (proj, canon) = merge_proj(home.home(), "m7m9", r#"{"model": "ours"}"#);
        db.save_project(&proj).expect("save");
        let target = canon.join(".claude").join("settings.json");
        std::fs::create_dir_all(target.parent().unwrap()).unwrap();
        std::fs::write(&target, r#"{"model": "USER_ORIGINAL"}"#).unwrap();

        ProjectApplyService::apply(&state, &proj.id).expect("apply");

        // user adds a brand-new key that was NEVER in our fragment (M9).
        let mut cur: serde_json::Value =
            serde_json::from_slice(&std::fs::read(&target).unwrap()).unwrap();
        cur["userBrandNew"] = serde_json::json!("added later");
        std::fs::write(&target, serde_json::to_vec_pretty(&cur).unwrap()).unwrap();

        ProjectApplyService::detach(&state, &proj.id).expect("detach");
        let disk: serde_json::Value =
            serde_json::from_slice(&std::fs::read(&target).unwrap()).unwrap();
        assert_eq!(disk["model"], serde_json::json!("USER_ORIGINAL"), "overwritten user key restored (M7)");
        assert_eq!(disk["userBrandNew"], serde_json::json!("added later"), "brand-new user key survives (M9)");

        // rows cleared.
        let chan = format!("project:{}", canon.to_string_lossy());
        assert_eq!(db.get_manifest_for_channel(&chan).unwrap().len(), 0, "rows cleared on detach");
    }
```

2. **Run (expect FAIL — M10(b) fails: WITHOUT the detach arm the merge row hits the else; `remove_whole_file_if_owned(.., None)` returns Ok(false), so the file actually survives the no-op delete — BUT the [model] leaf is NEVER reversed, so `disk_b.get("model")` is still "ours", and M7/M9 detach-restore fails)**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; export CARGO_NET_RETRY=10; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud/src-tauri && cargo test --lib "m10_settings_file detach_restores_overwritten" 2>&1 | tail -20
```

3. **Minimal impl** — in the detach loop (project_apply.rs :247-264), insert the `settings_merge` arm BETWEEN the `if r.kind == "skill"` block and the catch-all `else`:
```rust
            // owned-delete: only if on-disk hash == recorded hash.
            if r.kind == "skill" {
                // dir-aware, content-hash-safe removal (never remove_dir_all a real user dir)
                if let Some(parent) = Path::new(&r.target_path).parent() {
                    let dir_name = Path::new(&r.target_path)
                        .file_name()
                        .map(|n| n.to_string_lossy().to_string())
                        .unwrap_or_default();
                    crate::services::SkillService::remove_from_project_dir(&dir_name, parent, &app)
                        .map_err(|e| {
                            AppError::Message(format!("project skill remove failed: {e}"))
                        })?;
                }
            } else if r.kind == "settings_merge" {
                // CRITICAL (contract c): a merge row MUST NOT hit the else
                // (remove_whole_file_if_owned would attempt a whole-file delete of
                // the user's settings.json). Reverse our recorded leaves instead.
                crate::services::settings_merge::reverse_merge(
                    std::path::Path::new(&r.target_path),
                    r.owned_keys.as_deref(),
                    &mut result.warnings,
                )?;
            } else {
                crate::services::profile_render::remove_whole_file_if_owned(
                    &r.target_path,
                    r.content_hash.as_deref(),
                )?;
            }
```

4. **Run (expect PASS)**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; export CARGO_NET_RETRY=10; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud/src-tauri && cargo test --lib "m10_settings_file detach_restores_overwritten" 2>&1 | tail -15
```

5. **Commit**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && cargo fmt && git add -A && git commit -m "feat(project): detach settings_merge reverse arm + M10 regression guard

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## T11 — `ProjectSpec.dotfiles.settings` storage (serde round-trip; v17 blob without settings deserializes)

### Files
- Modify: `src-tauri/src/app_config.rs` (the `settings` field was added in T8 for compile order; this task adds its STORAGE tests). Tests go in the existing `#[cfg(test)] mod project_struct_tests` (module opens at :1379; append after its sibling `old_project_spec_json_without_dotfiles_still_deserializes` :1410, before the module's closing `}`).
- Optional: `src-tauri/src/database/dao/projects.rs` (DAO round-trip test — `save_project` / `get_project` already serialize `spec` whole, so no code change; add a test only if a project-DAO test module exists).
- Test: `src-tauri/src/app_config.rs`

### Steps

1. **Write failing test** — append to `mod project_struct_tests` in `app_config.rs`:
```rust
    #[test]
    fn old_project_spec_json_without_settings_still_deserializes() {
        // A v17/4b-1 spec blob has dotfiles.claudeMd but NO dotfiles.settings.
        // #[serde(default)] must let it deserialize with an empty settings string.
        let old = r#"{"content":{"skills":[],"commands":[],"agents":[],"mcp":[]},"vars":{},"dotfiles":{"claudeMd":"# m\n"}}"#;
        let spec: super::ProjectSpec = serde_json::from_str(old).expect("old spec must deserialize");
        assert_eq!(spec.dotfiles.claude_md, "# m\n");
        assert_eq!(spec.dotfiles.settings, "", "missing settings defaults to empty");
    }

    #[test]
    fn project_dotfiles_settings_round_trips_camelcase() {
        let mut spec = super::ProjectSpec::default();
        spec.dotfiles.settings = r#"{"model":"x"}"#.to_string();
        let json = serde_json::to_string(&spec).expect("serialize");
        // wire key is `settings` (rename_all=camelCase leaves single-word keys unchanged).
        assert!(json.contains(r#""settings":"{\"model\":\"x\"}""#), "got {json}");
        let back: super::ProjectSpec = serde_json::from_str(&json).expect("round-trip");
        assert_eq!(back.dotfiles.settings, r#"{"model":"x"}"#);
    }
```

2. **Run (expect FAIL if T8's field edit was somehow reverted; otherwise these PASS immediately because T8 already added the field)**. If executing strictly in order and T8 added the field, run to CONFIRM PASS — these tests pin the serde contract:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; export CARGO_NET_RETRY=10; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud/src-tauri && cargo test --lib "old_project_spec_json_without_settings project_dotfiles_settings_round_trips" 2>&1 | tail -15
```

3. **Minimal impl** — the `settings` field on `ProjectDotfiles` was added in T8. No additional production code is needed here (storage is serde-only, NO migration — the schema v18 bump is solely for `apply_manifest.owned_keys`). If for any reason the field is absent, add it now (same snippet as T8 step 3).

4. **Run (expect PASS)**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; export CARGO_NET_RETRY=10; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud/src-tauri && cargo test --lib project_struct_tests 2>&1 | tail -15
```

5. **Commit**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && cargo fmt && git add -A && git commit -m "test(project): ProjectDotfiles.settings serde round-trip + v17-blob compat

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## T12 — Frontend: projects.ts type + BindDialog textarea + seed-from-profile settings copy + 4 i18n + vitest

### Files
- Modify: `src/lib/api/projects.ts` (dotfiles type :8)
- Modify: `src/components/projects/ProjectBindDialog.tsx` (state :32; reset effect :42; onSave spec :62; new textarea after the claudeMd textarea :124)
- Modify: `src-tauri/src/commands/project.rs` (seed-from-profile: copy `settings.json` profile dotfile into `spec.dotfiles.settings` :148-154; add a Rust seed test)
- Modify: `src/i18n/locales/en.json`, `zh.json`, `ja.json`, `zh-TW.json` (`projects.settings` / `settingsPlaceholder` / `settingsHint`)
- Modify: `src/lib/api/projects.test.ts` (round-trip the settings field)
- Test: `src/lib/api/projects.test.ts` (vitest) + `src-tauri/src/commands/project.rs` (Rust seed test)

### Steps

1. **Write failing tests**:
   - Vitest — in `src/lib/api/projects.test.ts`, update the existing `save passes camelCase args` test's spec literals to include `settings: "{\"model\":\"x\"}"` and assert it round-trips. Replace the two `dotfiles: { claudeMd: "" }` occurrences (:15, :21) with `dotfiles: { claudeMd: "", settings: "{\"model\":\"x\"}" }` and add an assertion that the invoke payload carries `spec.dotfiles.settings === "{\"model\":\"x\"}"`.
   - Rust seed test — append to the test module in `commands/project.rs` (mirror `save_seeds_profile_claude_md_as_one_time_snapshot`):
```rust
    #[test]
    #[serial]
    fn save_seeds_profile_settings_json_as_one_time_snapshot() {
        let home = TempHome::new();
        crate::settings::reload_settings().ok();
        let db = std::sync::Arc::new(crate::database::Database::memory().expect("db"));
        let state = crate::store::AppState::new(db.clone());
        // seed a profile with a settings.json dotfile (rel_path == "settings.json").
        // Mirror save_seeds_profile_claude_md_as_one_time_snapshot exactly: there is
        // NO `seed_profile` helper in commands/project.rs — construct the Profile
        // literal inline and call db.save_profile.
        let prof = crate::app_config::Profile {
            id: "local:claude:Set".into(),
            app_type: "claude".into(),
            name: "Set".into(),
            description: None,
            is_active: false,
            current_provider_id: None,
            spec: ProfileSpec::default(),
            sort_index: 0,
            created_at: 0,
        };
        db.save_profile(&prof).expect("save profile");
        db.set_profile_dotfile("local:claude:Set", "settings.json", r#"{"model":"prof"}"#)
            .expect("seed settings dotfile");

        let root = home.dir.path().join("seedset");
        std::fs::create_dir_all(&root).expect("mkdir");
        let spec = crate::app_config::ProjectSpec::default();
        let created = run_save_logic(
            &state,
            None,
            "claude",
            root.to_str().unwrap(),
            None,
            spec,
            Some("local:claude:Set".to_string()),
        )
        .expect("save");
        assert_eq!(
            created.spec.dotfiles.settings, r#"{"model":"prof"}"#,
            "profile settings.json seeded into project dotfiles"
        );

        // editing the profile's settings.json afterwards must NOT change the project.
        db.set_profile_dotfile("local:claude:Set", "settings.json", r#"{"model":"CHANGED"}"#)
            .expect("edit");
        let reloaded = state.db.get_project(&created.id).unwrap().unwrap();
        assert_eq!(
            reloaded.spec.dotfiles.settings, r#"{"model":"prof"}"#,
            "snapshot, not a live link"
        );
    }
```
   (Match the real test module in `commands/project.rs`: it has `run_save_logic` (`pub(crate) fn run_save_logic` :120), the in-module `TempHome` guard, and the imports `use std::sync::Arc; use crate::database::Database; use crate::store::AppState; use crate::app_config::{ProfileContent, ProfileSpec};` plus the parent `use crate::app_config::{ManifestEntry, Project, ProjectSpec};`. There is NO `seed_profile` helper — the sibling `save_seeds_profile_claude_md_as_one_time_snapshot` (:300) constructs a `crate::app_config::Profile { .. }` literal inline and calls `db.save_profile(&prof)`, which is exactly what the test above does. The `run_save_logic` 7-arg call, `set_profile_dotfile`/`get_profile`, and snapshot-not-live-link assertions match the real signatures.)

2. **Run (expect FAIL)**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; export CARGO_NET_RETRY=10; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && ./node_modules/.bin/vitest run src/lib/api/projects.test.ts 2>&1 | tail -20; cd src-tauri && cargo test --lib save_seeds_profile_settings_json 2>&1 | tail -15
```

3. **Minimal impl**:
   - `projects.ts` :8 — `dotfiles: { claudeMd: string; settings: string };`
   - `ProjectBindDialog.tsx`:
     - state (:32): add `const [settings, setSettings] = useState("");`
     - reset effect (:42): add `setSettings(project?.spec.dotfiles?.settings ?? "");`
     - onSave spec (:62): `dotfiles: { claudeMd, settings },`
     - new textarea after the claudeMd textarea (after :124):
```tsx
        <label className="text-sm font-medium">{t("projects.settings")}</label>
        <p className="text-xs text-muted-foreground mt-1">{t("projects.settingsHint")}</p>
        <textarea
          className="mt-1 mb-3 w-full rounded-md border border-input bg-background px-3 py-2 font-mono text-xs focus:outline-none focus:ring-2 focus:ring-ring"
          value={settings}
          onChange={(e) => setSettings(e.target.value)}
          rows={6}
          spellCheck={false}
          aria-label={t("projects.settings")}
          placeholder={t("projects.settingsPlaceholder")}
        />
```
   - `commands/project.rs` seed (after the claude_md seed block, :154):
```rust
        // Seed the settings.json fragment from profile_dotfiles (rel_path=="settings.json")
        // only when the project has none yet — mirrors the claude_md empty-guard.
        // One-time snapshot, NOT a live link.
        if spec.dotfiles.settings.is_empty() {
            if let Some(df) = state.db.get_profile_dotfile(&pid, "settings.json")? {
                spec.dotfiles.settings = df.content;
            }
        }
```
   - i18n — in each of `en.json`, `zh.json`, `ja.json`, `zh-TW.json`, add three keys to the `projects` object (after `claudeMdHint`). English (translate the others appropriately):
```json
    "settings": "Project settings.json (deep-merged)",
    "settingsPlaceholder": "{\n  \"model\": \"${ANTHROPIC_MODEL}\"\n}",
    "settingsHint": "Deep-MERGED into <project>/.claude/settings.json. Supports ${VAR}. Arrays (permissions.allow/deny, hooks) are WHOLE-ARRAY replace — project wins; detach restores your prior array. No element union.",
```

4. **Run (expect PASS) + tsc**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; export CARGO_NET_RETRY=10; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && ./node_modules/.bin/tsc --noEmit 2>&1 | tail -10 && ./node_modules/.bin/vitest run src/lib/api/projects.test.ts 2>&1 | tail -15; cd src-tauri && cargo test --lib save_seeds_profile_settings_json 2>&1 | tail -10
```

5. **Commit**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && cargo fmt && git add -A && git commit -m "feat(ui): project settings.json textarea + seed-from-profile + 4-locale i18n

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## T13 — e2e (bind → apply → assert merged disk + envelope → detach → user keys survive) + FULL gate suite

### Files
- Modify: `src-tauri/src/lib.rs` (re-export the envelope type for the integration crate — `pub use services::{...}` block :57-65 exposes `ProjectApplyService`/`ProjectBase` but NOT `settings_merge`; `services` is NOT a top-level `pub mod`, so the integration crate can only see what's re-exported)
- Modify: `src-tauri/tests/project_apply_e2e.rs` (add an e2e test; reuse the file's existing `TempHome` (`home.dir.path()`, NO `reload_settings`) + flattened `use agenthub_lib::{AppState, AppType, Database, InstalledCommand, ProfileContent, Project, ProjectApplyService, ProjectBase, ProjectSpec}` imports. ALSO: the existing `project_apply_detach_lifecycle` test constructs a `ProjectDotfiles { claude_md: ... }` literal at :81-83 — this gains a `settings: String` field in T8, so that EXISTING literal must add `settings: String::new(),` to keep the crate compiling. Grep `ProjectDotfiles {` across `tests/` + `src/` when running T8/T13.)
- Test: `src-tauri/tests/project_apply_e2e.rs` (integration crate)

### Steps

1. **Write failing test** — the real harness is `TempHome` with `home.dir.path()` and NO `reload_settings` call; types come from the FLATTENED `agenthub_lib::{...}` imports. Add the envelope type to the test's `use` line: `use agenthub_lib::OwnedKeysEnvelope;` (re-exported in step 3). Then add:
```rust
#[test]
#[serial]
fn e2e_settings_merge_apply_then_detach_preserves_user_keys() {
    let home = TempHome::new();
    let db = Arc::new(Database::memory().expect("db"));
    let state = AppState::new(db.clone());

    // a project bound with a settings fragment that uses ${VAR} + a user file on disk.
    let root = home.dir.path().join("e2e-set");
    std::fs::create_dir_all(&root).unwrap();
    let canon = root.canonicalize().unwrap();
    let target = canon.join(".claude").join("settings.json");
    std::fs::create_dir_all(target.parent().unwrap()).unwrap();
    std::fs::write(&target, r#"{"userKept": 7, "model": "USER_ORIGINAL"}"#).unwrap();

    let mut spec = ProjectSpec::default();
    spec.dotfiles.settings =
        r#"{"model": "${MODEL_NAME}", "permissions": {"defaultMode": "ask"}}"#.into();
    spec.vars.insert(
        "MODEL_NAME".into(),
        serde_json::Value::String("claude-e2e".into()),
    );
    let proj = Project {
        id: "proj:e2e-set".into(),
        project_path: canon.to_string_lossy().to_string(),
        entered_path: root.to_string_lossy().to_string(),
        app_type: "claude".into(),
        name: Some("e2e".into()),
        spec,
        enabled: true,
        created_at: 1,
        updated_at: 1,
    };
    db.save_project(&proj).unwrap();

    // apply: ${VAR} rendered, frag merged, user keys preserved, one envelope row.
    ProjectApplyService::apply(&state, &proj.id).expect("apply");
    let merged: serde_json::Value =
        serde_json::from_slice(&std::fs::read(&target).unwrap()).unwrap();
    assert_eq!(merged["userKept"], serde_json::json!(7));
    assert_eq!(merged["model"], serde_json::json!("claude-e2e"));
    assert_eq!(merged["permissions"]["defaultMode"], serde_json::json!("ask"));

    let chan = format!("project:{}", canon.to_string_lossy());
    let rows = db.get_manifest_for_channel(&chan).unwrap();
    let env: OwnedKeysEnvelope = serde_json::from_str(
        rows.iter()
            .find(|r| r.kind == "settings_merge")
            .unwrap()
            .owned_keys
            .as_deref()
            .unwrap(),
    )
    .unwrap();
    assert_eq!(env.v, 1);

    // detach: user keys survive; our leaves reversed ([model] restored, defaultMode removed).
    ProjectApplyService::detach(&state, &proj.id).expect("detach");
    assert!(target.exists(), "settings.json survives detach");
    let after: serde_json::Value =
        serde_json::from_slice(&std::fs::read(&target).unwrap()).unwrap();
    assert_eq!(after["userKept"], serde_json::json!(7), "unrelated user key survives");
    assert_eq!(
        after["model"],
        serde_json::json!("USER_ORIGINAL"),
        "overwritten user key restored"
    );
    assert!(
        after.get("permissions").map_or(true, |p| p.get("defaultMode").is_none()),
        "our inserted defaultMode leaf removed"
    );
    assert_eq!(
        db.get_manifest_for_channel(&chan).unwrap().len(),
        0,
        "rows cleared"
    );
}
```
   (Confirmed against the real file: harness is `TempHome` with the `home.dir` field exposing `home.dir.path()`; the other e2e tests do NOT call `reload_settings` — omit it. `Project`/`ProjectSpec`/`ProjectDotfiles`/`ProjectApplyService` are all imported flattened from `agenthub_lib`.)

2. **Run (expect FAIL — `OwnedKeysEnvelope` is not re-exported, so the import does not resolve)**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; export CARGO_NET_RETRY=10; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud/src-tauri && cargo test --test project_apply_e2e e2e_settings_merge 2>&1 | tail -20
```

3. **Minimal impl** — re-export the envelope from `lib.rs` so the integration crate can parse it (no other production code change; all production code landed in T1–T12). Add to the `pub use services::{...}` block (:57-65), under the `project_apply::ProjectApplyService` line:
```rust
    project_apply::ProjectApplyService,
    project_paths::ProjectBase,
    settings_merge::OwnedKeysEnvelope,
```
   Match the e2e harness helper names so the test parses (no signature changes). `settings_merge` itself stays a non-pub-re-exported module; only the type the e2e asserts on is surfaced.

4. **Run the FULL gate suite (expect PASS; report exact counts)**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; export CARGO_NET_RETRY=10; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud/src-tauri && \
  cargo fmt --check && \
  cargo clippy --all-targets -- -D warnings 2>&1 | tail -15 && \
  cargo test 2>&1 | tail -30
cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && \
  ./node_modules/.bin/tsc --noEmit && \
  ./node_modules/.bin/vitest run 2>&1 | tail -15
```
   Expected: lib test count = **1551 baseline + the new 4b-2 tests** (T1: 2 new + 1 fixed assert; T2: extended; T3: 1; T4: 2; T5: 6; T6: 11; T7: 2; T8: 4; T9: 1; T10: 2; T11: 2; T12: 1 Rust seed; T13: 1 e2e in the integration crate). Confirm the printed `test result: ok. N passed` is `N > 1551` for the lib target and the e2e integration target also passes. Vitest stays at its baseline + 0 (the projects.test.ts edit modifies an existing test, not adds one — adjust the count claim to the actual printed number).

5. **Commit**:
```bash
export PATH="$HOME/.cargo/bin:$PATH"; cd /Users/fsm/project/MyProject/agentplugin/cc-switch-cloud && cargo fmt && git add -A && git commit -m "test(project): e2e settings.json merge + full gate suite (4b-2 complete)

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```
