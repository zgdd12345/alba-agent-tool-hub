# AgentHub Increment 4b-1 — Project CLAUDE.md (literal, whole-file)

REQUIRED-SUB-SKILL: superpowers:test-driven-development

## Goal

Bind a real project directory to an own, **literal** project-root `CLAUDE.md` and materialize it on apply. A bound project's `spec` now persists a `dotfiles.claude_md` string (serde-only, no schema change). On apply, when `claude_md` is non-empty, AgentHub hash-gated-writes it to `<project-root>/CLAUDE.md` (ROOT level, NOT `.claude/`) and records an `apply_manifest` row `kind="project_memory"`. Teardown (apply pre-delete sweep + detach) reuses the EXISTING 4a whole-file else-arm (`remove_whole_file_if_owned`) — no loop restructure, no new dispatch arm. Seed-from-profile copies the profile's literal `CLAUDE.md` snapshot once. The frontend BindDialog gains a CLAUDE.md textarea (carried through the already-whole-spec save path), with i18n in all 4 locales.

Project-root `CLAUDE.md` is **additive** memory in Claude Code (it walks ancestors up to and including `~/.claude/CLAUDE.md`); the project file augments, not replaces, the global one. This is documented in code comments and the UI hint.

## Architecture

- **Storage (no schema change)**: `ProjectSpec` gains `dotfiles: ProjectDotfiles` where `ProjectDotfiles { claude_md: String }`. Both `#[serde(default)]`. Stored inside the existing `projects.spec` JSON blob (DAO at `projects.rs:89` already does `serde_json::to_string(&p.spec)`), so all existing v17 rows deserialize unchanged via `serde_json::from_str(..).unwrap_or_default()` at `projects.rs:135`. Schema stays **v17**, no migration.
- **Path**: `ProjectBase::memory_file(&self, app) = self.root().join(app.project_memory_filename())` → `<root>/CLAUDE.md` for Claude. Inherits the 4a `resolve` HOME-collapse + symlink gate (root is already canonical + gated).
- **Write helper**: a private `write_project_whole_file(abs_path, content, prior_owned_hash)` in `project_apply.rs` that mirrors `profile_render::render_whole_file`'s ownership contract but is base-agnostic (no `validate_rel_path`, no `~/.claude` hardcode, no `channel="global"` stamping). Reuses `profile_render::content_hash` + `config::atomic_write`. The filename is a compile-time constant (`CLAUDE.md`), and the abs path is past `ProjectBase::resolve`'s gate, so no traversal validation is needed.
- **Materialize**: a new block in `ProjectApplyService::apply` AFTER the skills block. When `project.spec.dotfiles.claude_md` is non-empty → `write_project_whole_file(base.memory_file(&app), claude_md, prior_hash)` then (on `Ok(Some(hash))`) `record_manifest_entry(Self::row(channel, project.id, target, "project_memory", &hash))`. `Self::row` is UNCHANGED (no `owned_keys` param — v17 has no such column). Empty `claude_md` → write nothing; the pre-delete sweep already removed any prior `project_memory` file.
- **Teardown (reuse)**: `project_memory` is a whole-file kind, so it falls into the existing `else` arm in BOTH the apply pre-delete loop (`project_apply.rs:101-106`) and `detach` (`project_apply.rs:231-236`), which calls `remove_whole_file_if_owned(&r.target_path, r.content_hash.as_deref())`. NO new arm.
- **Seed**: `run_save_logic` (`commands/project.rs:120`) extends its existing one-time profile snapshot to ALSO copy the profile's `CLAUDE.md` dotfile (read via `db.get_profile_dotfile(profile_id, "CLAUDE.md")`) into `spec.dotfiles.claude_md`, guarded by emptiness (one-time snapshot, not a live link).
- **Frontend**: BindDialog adds a `<textarea>` for `claude_md`; `spec.dotfiles.claude_md` rides the already-whole-spec `project_save` invoke (api/hook need only the type field). `project_memory` manifest rows appear automatically in the applied-files list.

## Tech Stack

- Rust 1.95, crate `agenthub_lib` / bin `agenthub`. `serde`, `serde_json`, `rusqlite`, `sha2` (via `profile_render::content_hash`), `tempfile`, `serial_test`.
- Frontend: React 18 + TypeScript, `react-i18next`, `@tanstack/react-query`, Vitest, `tsc --noEmit`.
- Repo: `/Users/fsm/project/MyProject/agentplugin/cc-switch-cloud`, branch `main`, HEAD `2e26f866`, schema **v17**.

### Env gotchas (apply to EVERY run/commit command)
- `cargo` lives at `~/.cargo/bin` → prefix every cargo command with `export PATH="$HOME/.cargo/bin:$PATH";`. Slow net → also export `CARGO_NET_RETRY=10`.
- Frontend tools run DIRECTLY from `./node_modules/.bin` (NOT pnpm): `./node_modules/.bin/tsc --noEmit`, `./node_modules/.bin/vitest run`.
- HOME-isolation Rust tests use `#[serial]` + the in-module `TempHome` guard (sets HOME/USERPROFILE/CC_SWITCH_TEST_HOME) and call `crate::settings::reload_settings().ok();` before touching the db.
- `cargo fmt` before EVERY commit; gate is `cargo fmt --check`.
- Clippy gate is `--all-targets` (lints tests too): `cargo clippy --all-targets -- -D warnings`.
- Full `cargo test` (NOT `--lib`) — the integration crate `tests/project_apply_e2e.rs` constructs `ProjectSpec` directly and must compile.

### Baseline (HEAD `2e26f866`)
- lib unit tests: **1540** passing. Vitest: **316** passing. 4b-1 ADDS tests (it never removes any).

---

## T1 — `ProjectSpec.dotfiles` + `ProjectDotfiles` struct (serde-only, no schema change)

### Files
- Modify: `src-tauri/src/app_config.rs` (add `ProjectDotfiles` near `ProjectSpec` ~:293-301; add `dotfiles` field to `ProjectSpec`; new test in the `project_struct_tests` module ~:1358-1388)
- Test: `src-tauri/src/app_config.rs` (inline `#[cfg(test)] mod project_struct_tests`)

### Steps

1. **Write failing test** — append to `mod project_struct_tests` (after the existing `project_spec_serde_camelcase_roundtrip`, before the closing `}` at :1388):
```rust
    #[test]
    fn old_project_spec_json_without_dotfiles_still_deserializes() {
        // An OLD v17 spec blob (pre-4b-1) has NO "dotfiles" key. #[serde(default)]
        // must let it deserialize with an empty claude_md — proving NO migration is needed.
        let old = r#"{"content":{"skills":[],"commands":["c"],"agents":[],"mcp":[]},"vars":{}}"#;
        let spec: super::ProjectSpec = serde_json::from_str(old).expect("old spec must deserialize");
        assert_eq!(spec.content.commands, vec!["c"]);
        assert_eq!(spec.dotfiles.claude_md, "", "missing dotfiles defaults to empty");

        // Round-trip a populated dotfiles set.
        let mut s2 = super::ProjectSpec::default();
        s2.dotfiles.claude_md = "# Project memory\nbe terse".into();
        let json = serde_json::to_string(&s2).expect("serialize");
        assert!(json.contains("\"claudeMd\""), "camelCase key expected: {json}");
        let back: super::ProjectSpec = serde_json::from_str(&json).expect("round-trip");
        assert_eq!(back.dotfiles.claude_md, "# Project memory\nbe terse");
    }
```

2. **Run (expect FAIL — `dotfiles` field does not exist)**:
```
export PATH="$HOME/.cargo/bin:$PATH"; cargo test -p agenthub --lib project_struct_tests
```
Expected: `error[E0609]: no field 'dotfiles' on type 'ProjectSpec'` (and `claudeMd` assertion unreachable). FAIL.

3. **Minimal impl** — in `app_config.rs`, insert `ProjectDotfiles` immediately before `ProjectSpec` (~:293), and add the field. Replace the existing `ProjectSpec` block at :295-301 with:
```rust
/// Device-local project dotfiles persisted inside the `projects.spec` JSON blob
/// (serde-only — NO schema change). 4b-1 ships `claude_md` (a LITERAL project-root
/// CLAUDE.md). Reserved room for a future `settings`/`mcp` field lands in 4b-2/4b-3.
#[derive(Debug, Clone, Default, Serialize, Deserialize)]
// REQUIRED (load-bearing): rename_all=camelCase makes `claude_md` serialize as the wire
// key `claudeMd`, which MUST match the frontend ProjectSpec.dotfiles.claudeMd type and the
// on-disk projects.spec JSON blob. Removing this attr silently diverges DB/TS from Rust.
#[serde(rename_all = "camelCase")]
pub struct ProjectDotfiles {
    /// Literal project-root CLAUDE.md content (NO ${VAR} rendering). Empty = none.
    #[serde(default)]
    pub claude_md: String,
}

/// Project spec: own content set (same JSON shape as ProfileSpec) + reserved vars for 4b
/// + device-local dotfiles (4b-1). All fields `#[serde(default)]` so every existing v17
/// `projects.spec` blob deserializes unchanged (no migration, schema stays v17).
#[allow(dead_code)] // wired in Task 3 DAO
#[derive(Debug, Clone, Default, Serialize, Deserialize)]
pub struct ProjectSpec {
    #[serde(default)]
    pub content: ProfileContent, // REUSE existing struct (skills/commands/agents/mcp)
    #[serde(default)]
    pub vars: serde_json::Map<String, serde_json::Value>, // reserved for 4b ${VAR} (NOT used in 4b-1)
    #[serde(default)]
    pub dotfiles: ProjectDotfiles, // 4b-1: literal project-root CLAUDE.md
}
```

4. **Fix every `ProjectSpec { .. }` / `Project { .. }` struct literal** (the recurring lesson — they spell out fields, so the new field must be added). The literals are at: `src/database/dao/projects.rs:163`, `src/services/project_apply.rs:379`, `tests/project_apply_e2e.rs:73`. (The literals at `app_config.rs:1364` and `commands/project.rs` use `ProjectSpec::default()` — unaffected.) Add `dotfiles: Default::default(),` to each:

`src/database/dao/projects.rs` — after `vars: serde_json::Map::new(),` (:170):
```rust
                vars: serde_json::Map::new(),
                dotfiles: Default::default(),
            },
```
`src/services/project_apply.rs` — after `vars: serde_json::Map::new(),` (:381) inside `project_at`:
```rust
                vars: serde_json::Map::new(),
                dotfiles: Default::default(),
            },
```
`tests/project_apply_e2e.rs` — after `vars: serde_json::Map::new(),` (:80):
```rust
            vars: serde_json::Map::new(),
            dotfiles: Default::default(),
        },
```

5. **Run (expect PASS) + confirm the integration crate compiles**:
```
export PATH="$HOME/.cargo/bin:$PATH"; cargo test -p agenthub --lib project_struct_tests && cargo test --no-run --all-targets
```
Expected: `old_project_spec_json_without_dotfiles_still_deserializes ... ok`; `cargo test --no-run` builds all targets (incl. `project_apply_e2e`) with no `missing field dotfiles` errors.

6. **Commit**:
```
export PATH="$HOME/.cargo/bin:$PATH"; cargo fmt
git add -A && git commit -m "feat(projects): add ProjectSpec.dotfiles (serde-only, no schema change)"
```

---

## T2 — `ProjectBase::memory_file` (project-root CLAUDE.md path)

### Files
- Modify: `src-tauri/src/services/project_paths.rs` (add `memory_file` method to `impl ProjectBase` ~:32-43; new `#[serial]` test in the inline `mod tests` ~:273+)
- Test: `src-tauri/src/services/project_paths.rs` (inline `#[cfg(test)] mod tests`)

### Steps

1. **Write failing test** — append inside `mod tests` (after `refuses_when_dotdir_symlinks_to_claude`, before the closing `}` at :373):
```rust
    #[test]
    #[serial]
    fn memory_file_is_root_level_not_under_dotdir() {
        let home = TempHome::new();
        let proj = home.home().join("work").join("memrepo");
        std::fs::create_dir_all(&proj).expect("mkdir proj");
        let base = ProjectBase::resolve(proj.to_str().unwrap(), &AppType::Claude).expect("resolve");
        // CLAUDE.md lives at the project ROOT, NOT under .claude/
        assert_eq!(base.memory_file(&AppType::Claude), base.root().join("CLAUDE.md"));
        assert_ne!(
            base.memory_file(&AppType::Claude),
            base.dotdir().join("CLAUDE.md"),
            "memory_file must use root(), not dotdir()"
        );
    }
```

2. **Run (expect FAIL — no `memory_file` method)**:
```
export PATH="$HOME/.cargo/bin:$PATH"; cargo test -p agenthub --lib services::project_paths::tests::memory_file_is_root_level_not_under_dotdir
```
Expected: `error[E0599]: no method named 'memory_file' found for ... ProjectBase`. FAIL.

3. **Minimal impl** — add to `impl ProjectBase` (after `channel`, before `resolve`, ~:43):
```rust
    /// Project-root memory file for `app` (4b-1: <root>/CLAUDE.md for Claude).
    /// ROOT level, NOT under dotdir() — Claude Code reads project-root CLAUDE.md
    /// ADDITIVELY with the global ~/.claude/CLAUDE.md (ancestor walk-up).
    pub fn memory_file(&self, app: &AppType) -> PathBuf {
        self.root.join(app.project_memory_filename())
    }
```

4. **Run (expect PASS)**:
```
export PATH="$HOME/.cargo/bin:$PATH"; cargo test -p agenthub --lib services::project_paths::tests::memory_file_is_root_level_not_under_dotdir
```
Expected: `memory_file_is_root_level_not_under_dotdir ... ok`.

5. **Commit**:
```
export PATH="$HOME/.cargo/bin:$PATH"; cargo fmt
git add -A && git commit -m "feat(projects): add ProjectBase::memory_file (project-root CLAUDE.md path)"
```

---

## T3 — `write_project_whole_file` hash-gated write helper

### Files
- Modify: `src-tauri/src/services/project_apply.rs` (add private fn `write_project_whole_file` in the file scope below `impl ProjectApplyService` ~:290; add imports `std::path::Path` already present — add nothing new; 3 new `#[serial]` tests in the inline `mod tests` ~:680+)
- Test: `src-tauri/src/services/project_apply.rs` (inline `#[cfg(test)] mod tests`)

### Steps

1. **Write failing tests** — append inside `mod tests` (before the closing `}` at :730). These exercise write-when-absent, skip-on-user-edit, and overwrite-when-owned:
```rust
    #[test]
    #[serial]
    fn write_project_whole_file_writes_when_absent() {
        let _home = TempHome::new();
        let dir = TempDir::new().expect("tmp");
        let target = dir.path().join("CLAUDE.md");
        // absent → write, returns Some(hash) of the content.
        let h = super::write_project_whole_file(&target, "# memory", None).expect("write");
        assert_eq!(std::fs::read_to_string(&target).unwrap(), "# memory");
        assert_eq!(
            h,
            Some(crate::services::profile_render::content_hash(b"# memory"))
        );
    }

    #[test]
    #[serial]
    fn write_project_whole_file_skips_user_edited_file() {
        let _home = TempHome::new();
        let dir = TempDir::new().expect("tmp");
        let target = dir.path().join("CLAUDE.md");
        // file exists with content we do NOT own (prior hash None) → skip + Ok(None).
        std::fs::write(&target, "USER WROTE THIS").expect("seed");
        let r = super::write_project_whole_file(&target, "MANAGED", None).expect("skip");
        assert_eq!(r, None, "must skip an unmanaged file");
        assert_eq!(std::fs::read_to_string(&target).unwrap(), "USER WROTE THIS");

        // file exists, prior hash present but disk hash differs (user edited) → skip.
        let prior = crate::services::profile_render::content_hash(b"OLD MANAGED");
        let r2 = super::write_project_whole_file(&target, "MANAGED", Some(&prior)).expect("skip2");
        assert_eq!(r2, None, "disk_hash != prior_owned_hash must skip");
        assert_eq!(std::fs::read_to_string(&target).unwrap(), "USER WROTE THIS");
    }

    #[test]
    #[serial]
    fn write_project_whole_file_overwrites_when_owned() {
        let _home = TempHome::new();
        let dir = TempDir::new().expect("tmp");
        let target = dir.path().join("CLAUDE.md");
        std::fs::write(&target, "OLD MANAGED").expect("seed");
        // disk_hash == prior_owned_hash → we own it → overwrite, return new hash.
        let prior = crate::services::profile_render::content_hash(b"OLD MANAGED");
        let h = super::write_project_whole_file(&target, "NEW MANAGED", Some(&prior)).expect("write");
        assert_eq!(std::fs::read_to_string(&target).unwrap(), "NEW MANAGED");
        assert_eq!(
            h,
            Some(crate::services::profile_render::content_hash(b"NEW MANAGED"))
        );
    }
```

2. **Run (expect FAIL — no `write_project_whole_file`)**:
```
export PATH="$HOME/.cargo/bin:$PATH"; cargo test -p agenthub --lib services::project_apply::tests::write_project_whole_file
```
Expected: `error[E0425]: cannot find function 'write_project_whole_file' in module 'super'` (3 tests fail to compile). FAIL.

3. **Minimal impl** — add a free function at file scope in `project_apply.rs`, immediately AFTER the closing `}` of `impl ProjectApplyService` (~:290, before `#[cfg(test)]`):
```rust
/// Hash-gated whole-file write for a project dotfile (4b-1). Mirrors
/// `profile_render::render_whole_file`'s OWNERSHIP contract but is base-agnostic:
/// it does NOT use `validate_rel_path` / `~/.claude` and does NOT stamp a manifest
/// row (the caller records via `Self::row`). The abs path is already past
/// `ProjectBase::resolve`'s HOME/symlink gate and the filename is a compile-time
/// constant ("CLAUDE.md"), so there is no traversal risk.
///
/// - exists AND (prior_owned_hash is None OR disk_hash != prior_owned_hash):
///   skip + warn, return Ok(None) (never overwrite a user-edited/unmanaged file).
/// - absent, OR disk_hash == prior_owned_hash: atomic_write the content, return
///   Ok(Some(sha256(content))).
#[allow(dead_code)] // consumed by apply's CLAUDE.md block (Task 4)
fn write_project_whole_file(
    abs_path: &Path,
    content: &str,
    prior_owned_hash: Option<&str>,
) -> Result<Option<String>, AppError> {
    if abs_path.exists() {
        let disk = std::fs::read(abs_path).map_err(|e| AppError::io(abs_path, e))?;
        let disk_hash = crate::services::profile_render::content_hash(&disk);
        let owned = matches!(prior_owned_hash, Some(h) if h == disk_hash);
        if !owned {
            log::warn!(
                "拒绝覆盖未托管/被用户编辑的项目文件: {}",
                abs_path.display()
            );
            return Ok(None);
        }
    }
    atomic_write(abs_path, content.as_bytes())?;
    Ok(Some(crate::services::profile_render::content_hash(
        content.as_bytes(),
    )))
}
```

4. **Run (expect PASS)**:
```
export PATH="$HOME/.cargo/bin:$PATH"; cargo test -p agenthub --lib services::project_apply::tests::write_project_whole_file
```
Expected: 3 tests `... ok` (`writes_when_absent`, `skips_user_edited_file`, `overwrites_when_owned`).

5. **Commit**:
```
export PATH="$HOME/.cargo/bin:$PATH"; cargo fmt
git add -A && git commit -m "feat(projects): add write_project_whole_file hash-gated helper"
```

---

## T4 — apply: materialize CLAUDE.md + record `project_memory` (+ idempotent re-apply / user-edit safety)

### Files
- Modify: `src-tauri/src/services/project_apply.rs` (add the CLAUDE.md block in `apply` after the skills loop ~:191, before `Ok(result)`; 3 new `#[serial]` tests in inline `mod tests`)
- Test: `src-tauri/src/services/project_apply.rs` (inline `#[cfg(test)] mod tests`)

### Steps

1. **Write failing tests** — append inside `mod tests` (before the closing `}`). The `project_at` helper takes only `content`, so these build the `Project` from it then set `spec.dotfiles.claude_md` before saving:
```rust
    #[test]
    #[serial]
    fn apply_writes_claude_md_at_root_and_records_project_memory() {
        let home = TempHome::new();
        crate::settings::reload_settings().ok();
        let db = Arc::new(Database::memory().expect("db"));
        let state = AppState::new(db.clone());
        let (mut proj, canon) = project_at(home.home(), "memroot", ProfileContent::default());
        proj.spec.dotfiles.claude_md = "# Project memory\nbe terse\n".into();
        db.save_project(&proj).expect("save");

        let res = ProjectApplyService::apply(&state, &proj.id).expect("apply");
        assert!(res.warnings.is_empty(), "no warnings: {:?}", res.warnings);

        // CLAUDE.md materialized at the project ROOT — NOT under .claude/.
        let root_file = canon.join("CLAUDE.md");
        assert_eq!(
            std::fs::read_to_string(&root_file).unwrap(),
            "# Project memory\nbe terse\n"
        );
        assert!(
            !canon.join(".claude").join("CLAUDE.md").exists(),
            "must NOT be written under .claude/"
        );

        // one project_memory manifest row on this channel, hash = sha256(content).
        let chan = format!("project:{}", canon.to_string_lossy());
        let rows = db.get_manifest_for_channel(&chan).unwrap();
        let mem: Vec<_> = rows.iter().filter(|r| r.kind == "project_memory").collect();
        assert_eq!(mem.len(), 1, "exactly one project_memory row");
        assert_eq!(mem[0].project_id.as_deref(), Some(proj.id.as_str()));
        assert_eq!(mem[0].target_path, root_file.to_string_lossy());
        assert_eq!(
            mem[0].content_hash.as_deref(),
            Some(crate::services::profile_render::content_hash(b"# Project memory\nbe terse\n").as_str())
        );
    }

    #[test]
    #[serial]
    fn apply_empty_claude_md_writes_no_file_and_no_row() {
        let home = TempHome::new();
        crate::settings::reload_settings().ok();
        let db = Arc::new(Database::memory().expect("db"));
        let state = AppState::new(db.clone());
        let (proj, canon) = project_at(home.home(), "memnone", ProfileContent::default());
        // dotfiles.claude_md left empty by Default
        db.save_project(&proj).expect("save");
        ProjectApplyService::apply(&state, &proj.id).expect("apply");

        assert!(!canon.join("CLAUDE.md").exists(), "empty claude_md → no file");
        let chan = format!("project:{}", canon.to_string_lossy());
        let rows = db.get_manifest_for_channel(&chan).unwrap();
        assert!(
            !rows.iter().any(|r| r.kind == "project_memory"),
            "no project_memory row for empty claude_md"
        );
    }

    #[test]
    #[serial]
    fn reapply_is_idempotent_and_user_edited_claude_md_survives() {
        let home = TempHome::new();
        crate::settings::reload_settings().ok();
        let db = Arc::new(Database::memory().expect("db"));
        let state = AppState::new(db.clone());
        let (mut proj, canon) = project_at(home.home(), "memreapply", ProfileContent::default());
        proj.spec.dotfiles.claude_md = "# v1\n".into();
        db.save_project(&proj).expect("save");
        ProjectApplyService::apply(&state, &proj.id).expect("apply 1");

        let root_file = canon.join("CLAUDE.md");
        let chan = format!("project:{}", canon.to_string_lossy());

        // clean re-apply with NEW owned content: pre-delete removes the owned v1
        // (hash matches), then the new write lands → still exactly one row.
        proj.spec.dotfiles.claude_md = "# v2\n".into();
        db.save_project(&proj).expect("save v2");
        ProjectApplyService::apply(&state, &proj.id).expect("apply 2");
        assert_eq!(std::fs::read_to_string(&root_file).unwrap(), "# v2\n");
        let mem = db
            .get_manifest_for_channel(&chan)
            .unwrap()
            .into_iter()
            .filter(|r| r.kind == "project_memory")
            .count();
        assert_eq!(mem, 1, "re-apply stays idempotent (one row)");

        // USER edits CLAUDE.md → next apply must NOT clobber it: pre-delete skips
        // (hash mismatch leaves the file), and the new write also skips (file
        // exists with a non-matching prior hash) → user edit preserved + warn.
        std::fs::write(&root_file, "# USER OWNS THIS NOW\n").expect("user edit");
        ProjectApplyService::apply(&state, &proj.id).expect("apply 3");
        assert_eq!(
            std::fs::read_to_string(&root_file).unwrap(),
            "# USER OWNS THIS NOW\n",
            "user-edited CLAUDE.md must survive re-apply"
        );
    }
```

2. **Run (expect FAIL — apply does not write CLAUDE.md yet)**:
```
export PATH="$HOME/.cargo/bin:$PATH"; cargo test -p agenthub --lib services::project_apply::tests::apply_writes_claude_md_at_root_and_records_project_memory services::project_apply::tests::reapply_is_idempotent_and_user_edited_claude_md_survives
```
Expected: `apply_writes_claude_md_at_root_and_records_project_memory` FAILS at `read_to_string(&root_file)` (No such file or directory); `reapply...` FAILS at the v2 assertion. (`apply_empty_claude_md_writes_no_file_and_no_row` passes already — empty is the current behavior.) FAIL.

3. **Minimal impl** — in `ProjectApplyService::apply`, insert AFTER the skills loop (after :191's closing `}` of the `for s in skills.values()` block, before `Ok(result)` at :193):
```rust
        // ---- project memory (CLAUDE.md, literal, whole-file; 4b-1) ----
        // Whole-file kind. Teardown is handled by the EXISTING else-arm above
        // (pre-delete) and in detach (remove_whole_file_if_owned) — NO new arm.
        // The prior project_memory file (if any) was already owned-deleted by the
        // pre-delete sweep, so a non-empty write here is a clean (re)materialize and
        // an empty claude_md correctly leaves nothing behind.
        let claude_md = &project.spec.dotfiles.claude_md;
        if !claude_md.is_empty() {
            let target = base.memory_file(&app);
            // prior_owned_hash is None: the pre-delete sweep already removed our
            // previously-owned file, so the only reason `target` still exists is a
            // user-created/edited file we must NOT clobber (write helper skips+warns).
            if let Some(hash) = write_project_whole_file(&target, claude_md, None)? {
                state.db.record_manifest_entry(&Self::row(
                    &channel,
                    &project.id,
                    &target,
                    "project_memory",
                    &hash,
                ))?;
            } else {
                result.warnings.push(format!(
                    "skipped project CLAUDE.md (user-edited/unmanaged file present): {}",
                    target.display()
                ));
            }
        }

```

4. **Run (expect PASS)**:
```
export PATH="$HOME/.cargo/bin:$PATH"; cargo test -p agenthub --lib services::project_apply::tests::apply_writes_claude_md_at_root_and_records_project_memory services::project_apply::tests::apply_empty_claude_md_writes_no_file_and_no_row services::project_apply::tests::reapply_is_idempotent_and_user_edited_claude_md_survives
```
Expected: 3 tests `... ok`. (Note `write_project_whole_file` is now used in production — its `#[allow(dead_code)]` is now satisfied; clippy in T8 confirms no dead-code warning. You may drop the `#[allow(dead_code)]` line now or in T8 — T8's clippy gate will flag it if stale.)

5. **Commit**:
```
export PATH="$HOME/.cargo/bin:$PATH"; cargo fmt
git add -A && git commit -m "feat(projects): materialize literal project CLAUDE.md on apply (kind=project_memory)"
```

---

## T5 — detach / pre-delete reuse verification (whole-file else-arm handles `project_memory`)

This task adds NO production code — it proves the EXISTING 4a else-arm (`remove_whole_file_if_owned`) tears down `project_memory` in both detach and the apply pre-delete sweep, with NO new dispatch arm.

### Files
- Modify: `src-tauri/src/services/project_apply.rs` (2 new `#[serial]` tests in inline `mod tests`; NO change to `apply`/`detach`/`row`)
- Test: `src-tauri/src/services/project_apply.rs` (inline `#[cfg(test)] mod tests`)

### Steps

1. **Write failing tests** — append inside `mod tests`:
```rust
    #[test]
    #[serial]
    fn detach_removes_owned_claude_md_via_existing_else_arm() {
        let home = TempHome::new();
        crate::settings::reload_settings().ok();
        let db = Arc::new(Database::memory().expect("db"));
        let state = AppState::new(db.clone());
        let (mut proj, canon) = project_at(home.home(), "memdetach", ProfileContent::default());
        proj.spec.dotfiles.claude_md = "# owned memory\n".into();
        db.save_project(&proj).expect("save");
        ProjectApplyService::apply(&state, &proj.id).expect("apply");

        let root_file = canon.join("CLAUDE.md");
        assert!(root_file.is_file(), "applied first");

        // detach: the project_memory row falls into the existing ELSE arm
        // (remove_whole_file_if_owned) — owned (hash matches) → removed.
        ProjectApplyService::detach(&state, &proj.id).expect("detach");
        assert!(!root_file.exists(), "owned CLAUDE.md removed on detach");
        let chan = format!("project:{}", canon.to_string_lossy());
        assert_eq!(db.get_manifest_for_channel(&chan).unwrap().len(), 0, "rows cleared");
    }

    #[test]
    #[serial]
    fn detach_preserves_user_edited_claude_md() {
        let home = TempHome::new();
        crate::settings::reload_settings().ok();
        let db = Arc::new(Database::memory().expect("db"));
        let state = AppState::new(db.clone());
        let (mut proj, canon) = project_at(home.home(), "memdetach2", ProfileContent::default());
        proj.spec.dotfiles.claude_md = "# owned\n".into();
        db.save_project(&proj).expect("save");
        ProjectApplyService::apply(&state, &proj.id).expect("apply");

        // user edits the materialized CLAUDE.md → hash no longer matches the row →
        // remove_whole_file_if_owned must skip+warn → file survives detach.
        let root_file = canon.join("CLAUDE.md");
        std::fs::write(&root_file, "# USER EDITED\n").expect("edit");
        ProjectApplyService::detach(&state, &proj.id).expect("detach");
        assert_eq!(
            std::fs::read_to_string(&root_file).unwrap(),
            "# USER EDITED\n",
            "user-edited CLAUDE.md must NOT be deleted on detach"
        );
    }
```

2. **Run (expect PASS immediately — reuse, not new code)**:
```
export PATH="$HOME/.cargo/bin:$PATH"; cargo test -p agenthub --lib services::project_apply::tests::detach_removes_owned_claude_md_via_existing_else_arm services::project_apply::tests::detach_preserves_user_edited_claude_md
```
Expected: both `... ok`. This is the verification gate proving the else-arm at `project_apply.rs:231-236` already hash-gates an arbitrary whole-file `target_path`. If either fails, STOP — it would mean a new arm is wrongly required (it is not in 4b-1).

3. **Commit**:
```
export PATH="$HOME/.cargo/bin:$PATH"; cargo fmt
git add -A && git commit -m "test(projects): verify project_memory teardown reuses 4a whole-file else-arm"
```

---

## T6 — seed-from-profile: snapshot the profile's CLAUDE.md into `dotfiles.claude_md`

### Files
- Modify: `src-tauri/src/commands/project.rs` (extend the seed block in `run_save_logic` ~:136-146; 1 new `#[serial]` test in inline `mod tests` ~:226+)
- Test: `src-tauri/src/commands/project.rs` (inline `#[cfg(test)] mod tests`)

### Steps

1. **Write failing test** — append inside `mod tests` (after `save_seed_from_profile_is_a_snapshot_not_a_live_link`, before `save_refuses_home_collapse_path`):
```rust
    #[test]
    #[serial]
    fn save_seeds_profile_claude_md_as_one_time_snapshot() {
        let home = TempHome::new();
        crate::settings::reload_settings().ok();
        let db = Arc::new(Database::memory().expect("db"));
        let state = AppState::new(db.clone());

        let prof = crate::app_config::Profile {
            id: "local:claude:Mem".into(),
            app_type: "claude".into(),
            name: "Mem".into(),
            description: None,
            is_active: false,
            current_provider_id: None,
            spec: ProfileSpec::default(),
            sort_index: 0,
            created_at: 0,
        };
        db.save_profile(&prof).expect("save profile");
        // the profile's literal CLAUDE.md lives in profile_dotfiles (3b-3).
        db.set_profile_dotfile("local:claude:Mem", "CLAUDE.md", "# from profile\n")
            .expect("set dotfile");

        let root = home.dir.path().join("seedmem");
        std::fs::create_dir_all(&root).expect("mkdir");
        let created = run_save_logic(
            &state,
            None,
            "claude",
            root.to_str().unwrap(),
            Some("Seeded".into()),
            ProjectSpec::default(),
            Some("local:claude:Mem".into()),
        )
        .expect("save");
        assert_eq!(
            created.spec.dotfiles.claude_md, "# from profile\n",
            "profile CLAUDE.md seeded into project dotfiles"
        );

        // EDIT the profile's CLAUDE.md afterwards — must NOT change the bound project.
        db.set_profile_dotfile("local:claude:Mem", "CLAUDE.md", "# CHANGED\n")
            .expect("update dotfile");
        let reloaded = db.get_project(&created.id).unwrap().unwrap();
        assert_eq!(
            reloaded.spec.dotfiles.claude_md, "# from profile\n",
            "snapshot, not a live link"
        );
    }
```

2. **Run (expect FAIL — seed copies content only, not claude_md)**:
```
export PATH="$HOME/.cargo/bin:$PATH"; cargo test -p agenthub --lib commands::project::tests::save_seeds_profile_claude_md_as_one_time_snapshot
```
Expected: assertion fails — `created.spec.dotfiles.claude_md` is `""` not `"# from profile\n"`. FAIL.

3. **Minimal impl** — in `run_save_logic`, extend the existing seed block (:136-146). Replace it with:
```rust
    // One-time seed-from-profile snapshot (decision #3): only when the project
    // has no own content yet AND a profile id is given. COPY spec.content + the
    // profile's literal CLAUDE.md (a one-time snapshot, NOT a live link — a later
    // profile edit must not change the bound project).
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
        // Seed the literal CLAUDE.md from profile_dotfiles (rel_path=="CLAUDE.md", 3b-3)
        // only when the project has none yet — mirrors the content empty-guard.
        if spec.dotfiles.claude_md.is_empty() {
            if let Some(df) = state.db.get_profile_dotfile(&pid, "CLAUDE.md")? {
                spec.dotfiles.claude_md = df.content; // snapshot, not a live link
            }
        }
    }
```

4. **Run (expect PASS — and the existing snapshot test still passes)**:
```
export PATH="$HOME/.cargo/bin:$PATH"; cargo test -p agenthub --lib commands::project::tests
```
Expected: `save_seeds_profile_claude_md_as_one_time_snapshot ... ok`, `save_seed_from_profile_is_a_snapshot_not_a_live_link ... ok`, `save_refuses_home_collapse_path ... ok`.

5. **Commit**:
```
export PATH="$HOME/.cargo/bin:$PATH"; cargo fmt
git add -A && git commit -m "feat(projects): seed profile CLAUDE.md snapshot into project dotfiles"
```

---

## T7 — Frontend: BindDialog CLAUDE.md textarea + spec type + i18n (4 locales)

### Files
- Modify: `src/lib/api/projects.ts` (add `dotfiles.claudeMd` to the `ProjectSpec` interface ~:4-7)
- Modify: `src/components/projects/ProjectBindDialog.tsx` (claudeMd state + prefill + onSave spec + textarea UI)
- Modify: `src/i18n/locales/{en,zh,zh-TW,ja}.json` (3 keys each in the `projects` block)
- Test: `src/lib/api/__tests__/projects.spec.ts` (new vitest file — type-shape + serialization through `projectsApi.save`)

> Splitting note: this touches 6 files but is one cohesive frontend slice (type + dialog + 4 locales + 1 test). The TS type + the test are written first (TDD), then the dialog + locales are mechanical wiring that the type change forces.

### Steps

1. **Write failing test** — create `src/lib/api/__tests__/projects.spec.ts`:
```ts
import { describe, it, expect, vi, beforeEach } from "vitest";

const invokeMock = vi.fn();
vi.mock("@tauri-apps/api/core", () => ({ invoke: (...a: unknown[]) => invokeMock(...a) }));

import { projectsApi, type ProjectSpec } from "../projects";

describe("projectsApi project CLAUDE.md (4b-1)", () => {
  beforeEach(() => invokeMock.mockReset());

  it("ProjectSpec carries dotfiles.claudeMd", () => {
    const spec: ProjectSpec = {
      content: { skills: [], commands: [], agents: [], mcp: [] },
      vars: {},
      dotfiles: { claudeMd: "# memory" },
    };
    expect(spec.dotfiles.claudeMd).toBe("# memory");
  });

  it("save() forwards spec (incl. dotfiles.claudeMd) to project_save", async () => {
    invokeMock.mockResolvedValueOnce({ id: "proj:1" });
    const spec: ProjectSpec = {
      content: { skills: [], commands: [], agents: [], mcp: [] },
      vars: {},
      dotfiles: { claudeMd: "# project rules" },
    };
    await projectsApi.save(null, "claude", "/abs/repo", "Repo", spec, null);
    expect(invokeMock).toHaveBeenCalledWith("project_save", {
      id: null,
      app: "claude",
      enteredPath: "/abs/repo",
      name: "Repo",
      spec,
      seedFromProfileId: null,
    });
    expect(invokeMock.mock.calls[0][1].spec.dotfiles.claudeMd).toBe("# project rules");
  });
});
```

2. **Run (expect FAIL — `dotfiles` not on `ProjectSpec`)**:
```
./node_modules/.bin/vitest run src/lib/api/__tests__/projects.spec.ts
```
Expected: TS/type error `Object literal may only specify known properties, and 'dotfiles' does not exist in type 'ProjectSpec'`. FAIL.

3. **Minimal impl — type** — in `src/lib/api/projects.ts`, replace the `ProjectSpec` interface (:4-7):
```ts
export interface ProjectSpec {
  content: { skills: string[]; commands: string[]; agents: string[]; mcp: string[] };
  vars: Record<string, unknown>;
  // 4b-1: device-local project dotfiles. claudeMd = literal project-root CLAUDE.md.
  dotfiles: { claudeMd: string };
}
```

4. **Run (expect PASS for the api test)**:
```
./node_modules/.bin/vitest run src/lib/api/__tests__/projects.spec.ts
```
Expected: 2 tests pass.

5. **Wire the dialog** — in `src/components/projects/ProjectBindDialog.tsx`:

(a) add state after `const [agents, setAgents] = useState("");` (:31):
```tsx
  const [claudeMd, setClaudeMd] = useState("");
```
(b) add prefill in the `useEffect`, after `setAgents(...)` (:40):
```tsx
    setClaudeMd(project?.spec.dotfiles?.claudeMd ?? "");
```
(c) carry it in `onSave`'s `spec` literal — replace the spec object (:52-60):
```tsx
    const spec: ProjectSpec = {
      content: {
        skills: splitList(skills),
        commands: splitList(commands),
        agents: splitList(agents),
        mcp: [],
      },
      vars: {},
      dotfiles: { claudeMd },
    };
```
(d) add the textarea after the agents `<Input>` (:109), mirroring the ProfileEditDialog CLAUDE.md pattern (literal, monospace, additive-memory hint):
```tsx
        <label className="text-sm font-medium">{t("projects.claudeMd")}</label>
        <p className="text-xs text-muted-foreground mt-1">{t("projects.claudeMdHint")}</p>
        <textarea
          className="mt-1 mb-3 w-full rounded-md border border-input bg-background px-3 py-2 font-mono text-xs focus:outline-none focus:ring-2 focus:ring-ring"
          value={claudeMd}
          onChange={(e) => setClaudeMd(e.target.value)}
          rows={6}
          spellCheck={false}
          aria-label={t("projects.claudeMd")}
          placeholder={t("projects.claudeMdPlaceholder")}
        />
```

6. **Add i18n keys (all 4 locales)** — in each `src/i18n/locales/{en,zh,zh-TW,ja}.json`, locate the top-level `"projects"` object (NOT the `"profiles"` object — they are distinct; the existing `claudeMd`/`claudeMdHint` keys live under `profiles`, so there is no collision). Insert the 3 new keys inside the `"projects"` block, after that block's own `"agentsPlaceholder"` key and before that block's own `"appliedFiles"` key. Do NOT trust absolute line numbers — grep each file for the `"projects"` object's `"agentsPlaceholder"` and insert immediately after it. (Reference only: the `projects` block opens near en:2789, zh:2775, ja:2775, zh-TW:2720 — verify by reading, as line numbers drift.)

en.json:
```json
    "claudeMd": "Project CLAUDE.md (additive memory)",
    "claudeMdPlaceholder": "# Project rules\nLiteral text — no ${VAR} rendering.",
    "claudeMdHint": "Materialized to <project>/CLAUDE.md (project root). Claude Code reads it ADDITIVELY with your global ~/.claude/CLAUDE.md — it augments, not replaces.",
```
zh.json:
```json
    "claudeMd": "项目 CLAUDE.md（叠加记忆）",
    "claudeMdPlaceholder": "# 项目规则\n纯文本字面量——不做 ${VAR} 渲染。",
    "claudeMdHint": "写入 <项目>/CLAUDE.md（项目根目录）。Claude Code 会与全局 ~/.claude/CLAUDE.md 叠加读取——是补充而非替换。",
```
zh-TW.json:
```json
    "claudeMd": "專案 CLAUDE.md（疊加記憶）",
    "claudeMdPlaceholder": "# 專案規則\n純文字字面值——不做 ${VAR} 渲染。",
    "claudeMdHint": "寫入 <專案>/CLAUDE.md（專案根目錄）。Claude Code 會與全域 ~/.claude/CLAUDE.md 疊加讀取——是補充而非取代。",
```
ja.json:
```json
    "claudeMd": "プロジェクト CLAUDE.md（追加メモリ）",
    "claudeMdPlaceholder": "# プロジェクトルール\nリテラルテキスト — ${VAR} 展開なし。",
    "claudeMdHint": "<プロジェクト>/CLAUDE.md（プロジェクトルート）に出力されます。Claude Code はグローバルの ~/.claude/CLAUDE.md と追加的に読み込みます — 置換ではなく補完です。",
```

7. **Run (expect PASS — full frontend gate)**:
```
./node_modules/.bin/tsc --noEmit && ./node_modules/.bin/vitest run
```
Expected: `tsc` clean (0 errors); vitest **318** passing (baseline 316 + the 2 new tests).

8. **Commit**:
```
git add -A && git commit -m "feat(projects-ui): add project CLAUDE.md textarea + dotfiles type + i18n (4 locales)"
```

---

## T8 — E2E + full-suite verification + manual smoke

### Files
- Modify: `src-tauri/tests/project_apply_e2e.rs` (extend `project_apply_detach_lifecycle` to cover CLAUDE.md at root through the public API)
- Test: `src-tauri/tests/project_apply_e2e.rs` (integration crate, public `agenthub_lib` API)

### Steps

1. **Write failing test additions** — extend `project_apply_detach_lifecycle`. First set `claude_md` on the spec literal (the `dotfiles: Default::default()` added in T1 becomes a populated value); replace `dotfiles: Default::default(),` (:81 from T1) with the populated form, then add assertions. After the `db.save_project(&proj).unwrap();` line (:86), the spec must already carry the CLAUDE.md; change the `spec` literal's dotfiles to:
```rust
            vars: serde_json::Map::new(),
            dotfiles: agenthub_lib::ProjectDotfiles {
                claude_md: "# e2e project memory\n".into(),
            },
        },
```
Then append, after the existing `assert!(user.is_file(), "user file untouched");` (:102), still inside the test:
```rust
    // (re-apply to re-materialize after detach removed it) — apply once more to assert
    // CLAUDE.md materializes at ROOT, then a fresh detach removes it.
    ProjectApplyService::apply(&state, "proj:e2e").unwrap();
    let claude_md = canon.join("CLAUDE.md");
    assert_eq!(
        std::fs::read_to_string(&claude_md).unwrap(),
        "# e2e project memory\n",
        "project CLAUDE.md materialized at ROOT"
    );
    assert!(
        !canon.join(".claude").join("CLAUDE.md").exists(),
        "never under .claude/"
    );
    ProjectApplyService::detach(&state, "proj:e2e").unwrap();
    assert!(!claude_md.exists(), "owned CLAUDE.md removed on detach");
```
This requires re-exporting `ProjectDotfiles`. In `src-tauri/src/lib.rs`, extend the `app_config` re-export (:39-42) to include `ProjectDotfiles`:
```rust
pub use app_config::{
    AppType, InstalledCommand, InstalledSkill, McpApps, McpServer, MultiAppConfig, ProfileContent,
    Project, ProjectDotfiles, ProjectSpec, SkillApps,
};
```

2. **Run (expect FAIL before the lib.rs re-export, then exercises the new path)**:
```
export PATH="$HOME/.cargo/bin:$PATH"; cargo test --test project_apply_e2e
```
Expected (before the re-export): `error[E0432]: unresolved import 'agenthub_lib::ProjectDotfiles'`. After adding the re-export it builds and the new assertions pass. FAIL → PASS.

3. **Run the FULL gate suite (this is the verification gate — confirm every number)**:
```
export PATH="$HOME/.cargo/bin:$PATH" CARGO_NET_RETRY=10
cargo fmt
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test
./node_modules/.bin/tsc --noEmit
./node_modules/.bin/vitest run
```
Expected:
- `cargo fmt --check`: clean (exit 0, no diff).
- `cargo clippy --all-targets -- -D warnings`: 0 warnings. In particular, no `dead_code` warning for `write_project_whole_file` (now used by `apply`) — if clippy flags it, remove its `#[allow(dead_code)]` attribute (T4 step 4 note) and re-run.
- `cargo test`: lib **exactly 1551** passing = baseline 1540 + 11 new lib tests (T1:1, T2:1, T3:3, T4:3, T5:2, T6:1); integration `project_apply_e2e` 1 test passing (extended, not added). 0 failures. Any deviation from 1551 = a test was miscounted or accidentally dropped — investigate, do not accept as "close enough".
- `tsc --noEmit`: 0 errors.
- `vitest run`: **318** passing (baseline 316 + 2 new in `projects.spec.ts`).

4. **Commit**:
```
git add -A && git commit -m "test(projects): e2e project CLAUDE.md materialize+detach; re-export ProjectDotfiles"
```

5. **Manual smoke note** (document; run after `pnpm tauri dev` or the packaged build):
- Bind a real dir (e.g. `~/work/smoke-repo`), set the CLAUDE.md textarea to `# smoke memory`, Save.
- Click Apply → confirm `~/work/smoke-repo/CLAUDE.md` appears at the ROOT (NOT under `.claude/`) with exactly `# smoke memory`; the applied-files list shows a `project_memory` row at that path.
- Click Detach → confirm `~/work/smoke-repo/CLAUDE.md` is gone.
- Re-apply, then manually edit `CLAUDE.md` on disk, then Detach again → confirm the user-edited file SURVIVES (skip+warn) and re-apply does NOT clobber it.

---

## Out of scope (explicit — belongs to 4b-2 / 4b-3, do NOT build here)
- NO `owned_keys` column / schema v18; `Self::row` stays 5-arg (no owned_keys param).
- NO `merge_with_snapshot` / `reverse_merge` / `json_deep_merge` promotion / `build_project_var_map`.
- NO `settings_file` / `mcp_file` ProjectBase helpers; NO settings.json / .mcp.json.
- NO `${VAR}` rendering — project CLAUDE.md is LITERAL.
- NO apply/detach loop restructure / new dispatch arm — `project_memory` reuses the existing whole-file else-arm (`remove_whole_file_if_owned`).
- NO non-Claude tools (4c). apply already hard-errors non-Claude (unchanged).
