# agenthub 增量 3b-1 — Dotfiles 第一片:manifest + settings.json 确定性重建 + statusline.sh Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: superpowers:subagent-driven-development。每步 `- [ ]`。环境 gotcha 见 Tech Stack;提交前 `cargo fmt`。

**Goal:** 让 profile 能携带 **settings.json 片段**(深合并到 provider 之上,绝不覆盖/丢 provider env/token)与 **statusline.sh**(整文件,拒绝覆盖用户已有文件),并由 **apply_manifest** 在切换/停用时安全移除上个 profile 写的**整文件**产物。**本片不含 `${VAR}` 渲染、CLAUDE.md、富前端编辑器(均 3b-2)**——片段在 3b-1 按字面写入。

**Architecture(经对抗审查修正,务必照此):**
1. **settings.json = 确定性重建,而非"片段 diff 撤销"。** 把 active profile 的 settings.json 片段织入**唯一的** effective-settings 构建器 `build_effective_settings_with_common_config`(provider.settings_config → common-config snippet → **active-profile 片段**,三层 `json_deep_merge`)。这样**每一次** settings.json 写入(switch / failover re-sync / sync_current_to_live)都完整重生该文件,profile 片段永不被 re-sync 丢弃,**切换时无需"移除上个 profile"步骤**(复用 3a "切换=全覆盖"不变量)。**settings.json 完全不进 manifest。**
2. **backfill 对称剥离(修复 CRITICAL token 污染)。** provider 切换的 backfill 会把 live settings.json 回灌进上个 provider 的 DB `settings_config`。现有代码已在回灌前剥离 common-config;**3b-1 须在同处同样剥离 active profile 的片段**(`json_deep_remove`,镜像 common-config 处理),否则 profile 的 statusLine/permissions(及 3b-2 起的渲染密钥)会被永久写进 provider 行。collision 边缘(provider 与片段设同一键)与 common-config 既有行为**一致**,接受并文档化。
3. **manifest 仅管整文件 dotfile(statusline.sh)。** 行记 `target_path` + `content_hash`;**不存 fragment_json**。切换时按**出向 profile id**(在 `set_active_profile` 改写前读取)移除其整文件产物中不在新集合者。**绝不删未纳管的用户文件**:写整文件前若目标已存在且无「我方所有权(manifest 行 + hash 匹配)」→ **拒绝 + 警告**(用户选择)。
4. **真正的 rel_path 校验器**(强制):拒绝绝对路径、拒任何 `..` 分量、`get_claude_config_dir().join(rel_path)` 后 canonical 断言 `starts_with` 配置根;每次整文件/片段写前调用。3a 那个是测试,不是校验器,不可复用。
5. **激活时序(关键)**:set_active_profile 改为**倒数第二步**,其后追加一次**显式** `write_live_with_common_config(当前 provider)` 作为最后一步——此时新 profile 已 active,settings.json 以新片段确定性重生。provider 缺失则跳过该写 + warn。
6. **deactivate 确定性拆除**:清 active flag 后,以**无 profile 层**重写 settings.json(provider+common)+ 按 manifest 移除本 profile 的整文件。**不用 diff 撤销。**

**Tech Stack:** 同 3a。仓库 `/Users/fsm/project/MyProject/agentplugin/cc-switch-cloud`(从 main `00446655` 开分支 `profiles-inc3b1`)。环境 gotcha:cargo 在 `~/.cargo/bin`(命令前 `export PATH="$HOME/.cargo/bin:$PATH"`);`.cargo/config.toml` 强制 `/usr/bin/cc`,勿删;前端用 `./node_modules/.bin/{vitest,tsc,prettier}`(非 pnpm);慢网 `CARGO_NET_RETRY=10 CARGO_HTTP_LOW_SPEED_LIMIT=0`;触磁盘测试用 `CC_SWITCH_TEST_HOME`+`#[serial]`(见 backup.rs:784),纯 DB 用 `Database::memory()`;**提交前 `cargo fmt`**;最终门 `cargo clippy --all-targets -- -D warnings`(项目预存 lint 已在 3a 清);验证别用 `grep -c` 收尾。

**复用锚点(已核实):** `json_deep_merge`(live.rs:99)、`json_deep_remove`(live.rs:117)、`json_is_subset`(live.rs:51);`build_effective_settings_with_common_config`(live.rs:483-509)+ `write_live_with_common_config`(live.rs:511-528);backfill 在 `switch_normal`(provider/mod.rs:1682-1703)经 `strip_common_config_from_live_settings`(live.rs:531-571);`atomic_write`/`write_text_file`(config.rs:179/187)、`get_claude_config_dir`(config.rs:37);`read_live_settings`(live.rs:976);3a 的 `ProfileService::activate/deactivate`(services/profile.rs)、profiles/profile_dotfiles/apply_manifest 表(schema v13)、dao/profiles.rs。

**安全(承载属性,T8 对抗测试守护):** ① provider env/token 经片段合并后仍在;切换后**上个 provider 的 DB settings_config 字节不变**(无片段污染);② re-sync/failover 仍重生片段;③ 用户已有 statusline.sh 不被覆盖(拒绝+warn);④ 切换只删出向 profile 写的整文件,绝不删用户文件;⑤ rel_path 穿越被拒,绝不写出 `~/.claude` 外;⑥ deactivate 拆除后 provider 自身键存活。

---

## 范围 / 不做

- **做**:v14(`apply_manifest` 加 `content_hash`);profile_dotfiles + manifest(整文件)DAO;settings.json 确定性重建 + backfill 剥离;statusline.sh 整文件(拒覆盖 + 校验器 + manifest);activate/deactivate 接线;set/get/delete dotfile + get manifest 的 Tauri 命令;ProfileEditDialog **最小**新增(settings.json / statusline 两个原始文本框)。
- **不做(3b-2)**:`${VAR}` 替换引擎与优先级;CLAUDE.md 仲裁(prompt_id + 内联自动建隐藏 prompt 行);富 dotfile/vars 编辑器与 warnings UX。**不做(3c)**:@tag/skills.tags。**不做(inc4)**:项目通道。片段在 3b-1 **按字面**写(含字面 `${...}` 文本——3b-2 才替换)。

---

## T1:v14 迁移 — apply_manifest 加 content_hash [Opus]

**Files:** Modify `database/mod.rs`(SCHEMA_VERSION 13→14)、`database/schema.rs`(create_tables apply_manifest DDL + `migrate_v13_to_v14` + `13 =>` arm)、`database/tests.rs`。

- **- [ ] Step 1 (TDD):** tests.rs 加 `apply_manifest_has_content_hash_after_migrate`:内存 conn → `create_tables_on_conn` → `set_user_version(&conn, 13)` → `apply_schema_migrations_on_conn` → 用 `PRAGMA table_info(apply_manifest)` 断言存在列 `content_hash`,且 `get_user_version == 14`。跑确认 FAIL。**勿**直接二次调用 migrate(ADD COLUMN 非幂等);**只经版本门 runner 驱动**。
- **- [ ] Step 2:** mod.rs:52 `SCHEMA_VERSION = 14`。
- **- [ ] Step 3:** create_tables_on_conn 的 `apply_manifest` DDL 末尾加列 `content_hash TEXT`(使全新 v14 == v13+迁移)。
- **- [ ] Step 4:** 加 `fn migrate_v13_to_v14(conn) -> Result<(), AppError>`:`conn.execute("ALTER TABLE apply_manifest ADD COLUMN content_hash TEXT", [])`(map_err)+ `log::info!`。仿 `migrate_v12_to_v13`。
- **- [ ] Step 5:** `apply_schema_migrations_on_conn` 在 `12 =>` arm 后、`_ =>` 前插 `13 => { log; Self::migrate_v13_to_v14(conn)?; Self::set_user_version(conn, 14)?; }`。
- **- [ ] Step 6:** `cargo test --lib database::`(绿)+ `cargo fmt`。
- **- [ ] Step 7:** commit `feat(db): apply_manifest.content_hash (schema v14)`。

## T2:profile_dotfiles + manifest DAO + 模型 [Sonnet]

**Files:** Create `database/dao/profile_dotfiles.rs`、`database/dao/manifest.rs`;Modify `database/dao/mod.rs`、`app_config.rs`(模型)。

- **- [ ] Step 1:** `app_config.rs` 加:
  - `ProfileDotfile { profile_id: String, rel_path: String, content: String }`(serde camelCase)。
  - `ManifestEntry { id: i64, channel: String, profile_id: Option<String>, app_type: String, target_path: String, kind: String, content_hash: Option<String>, created_at: i64 }`(camelCase)。`kind` 值域 3b-1 仅 `"whole_file"`(3b-2 加 `"prompt"`)。
- **- [ ] Step 2 (TDD):** 两个 DAO 各自 `Database::memory()` round-trip 测试。
- **- [ ] Step 3:** `dao/profile_dotfiles.rs`:`get_profile_dotfiles(profile_id) -> Vec<ProfileDotfile>`、`get_profile_dotfile(profile_id, rel_path) -> Option<ProfileDotfile>`、`set_profile_dotfile(profile_id, rel_path, content)`(INSERT OR REPLACE,PK=(profile_id,rel_path))、`delete_profile_dotfile(profile_id, rel_path) -> bool`、`delete_all_profile_dotfiles(profile_id)`。
- **- [ ] Step 4:** `dao/manifest.rs`:`record_manifest_entry(&ManifestEntry) -> i64`(忽略入参 id,AUTOINCREMENT;`created_at` 用 `unixepoch()` 或调用方传)、`get_manifest_for_profile(profile_id, app_type) -> Vec<ManifestEntry>`、`delete_manifest_entries(&[i64])`、`clear_manifest_for_profile(profile_id, app_type)`。
- **- [ ] Step 5:** `dao/mod.rs` 加 `mod profile_dotfiles; mod manifest;`。
- **- [ ] Step 6:** `cargo test --lib database::`(绿)+ `cargo fmt`。clippy 若报新 pub 方法 dead_code(消费在 T3-T6),**可接受**,勿加 allow。
- **- [ ] Step 7:** commit `feat(db): profile_dotfiles + manifest DAO + models`。

## T3:settings.json 片段织入 effective-settings + backfill 剥离 [Opus] —— 皇冠

**Files:** Modify `services/provider/live.rs`(effective-settings 构建器 + backfill 剥离)。先**完整读** `build_effective_settings_with_common_config`(483-509)、`write_live_with_common_config`(511-528)、`strip_common_config_from_live_settings`(531-571)、`switch_normal` backfill(provider/mod.rs:1682-1703)。

**前向(写):** 在 `build_effective_settings_with_common_config` 内、common-config 合并之后,追加第 3 层:取 active profile 的 settings.json 片段并深合并。
```rust
// 在 effective_settings 应用完 common-config 之后：
if let Some(profile) = db.get_active_profile(app_type.as_str())? {
    if let Some(df) = db.get_profile_dotfile(&profile.id, "settings.json")? {
        match serde_json::from_str::<serde_json::Value>(&df.content) {
            Ok(frag) => json_deep_merge(&mut effective_settings, &frag),
            Err(e) => log::warn!("profile {} settings.json 片段非法 JSON，跳过: {e}", profile.id),
        }
    }
}
```
（3b-1 按字面合并；3b-2 会在 `from_str` 前对 `df.content` 做 `${VAR}` 替换。仅 `AppType::Claude` 适用——其余 app 跳过,与 common-config 的 app 分支一致。）

**反向(backfill 剥离):** 在 backfill 回灌前剥离 active profile 片段,**镜像 common-config 的剥离**。在 `strip_common_config_from_live_settings`(剥 common 之后)或紧邻 backfill `save_provider` 之前:
```rust
if let Some(profile) = db.get_active_profile(app_type.as_str())? {
    if let Some(df) = db.get_profile_dotfile(&profile.id, "settings.json")? {
        if let Ok(frag) = serde_json::from_str::<serde_json::Value>(&df.content) {
            json_deep_remove(&mut stripped, &frag); // 镜像 remove_common_config_from_settings
        }
    }
}
```
> collision 边缘(provider 与片段设同一键 → 剥离会移除 provider 的)与 common-config 既有行为一致,接受并在代码注释说明。

- **- [ ] Step 1 (TDD,触磁盘 `CC_SWITCH_TEST_HOME`+`#[serial]`):** 写下列对抗测试,先 RED:
  - `profile_settings_fragment_merges_over_provider_keeps_env`:seed provider(env.ANTHROPIC_AUTH_TOKEN=tok)+ active profile 片段 `{"statusLine":{"x":1}}`;`write_live_with_common_config` 后读 `~/.claude/settings.json`:含 `env.ANTHROPIC_AUTH_TOKEN==tok` **且** `statusLine.x==1`。
  - `provider_settings_config_not_polluted_by_fragment_on_backfill`(**CRITICAL**):provider P 为 current + active profile 片段 `{"statusLine":{"x":1}}` 已落盘;切到 provider Q(`ProviderService::switch`);断言 **P 的 DB `settings_config` 与原始字节一致**(无 statusLine 残留)。
  - `resync_reproduces_fragment`:落盘后,直接再调 `write_live_with_common_config`(模拟 re-sync),settings.json 仍含 `statusLine.x==1`(证明确定性重建)。
- **- [ ] Step 2:** 实现前向 + 反向。
- **- [ ] Step 3:** `cargo test --lib`(无回归;provider/live 既有测试仍绿)+ `clippy --lib -- -D warnings` + `cargo fmt`。
- **- [ ] Step 4:** commit `feat(provider): thread active-profile settings.json fragment into effective settings (+ backfill strip)`。

## T4:整文件 dotfile 渲染 — statusline.sh(拒覆盖 + 路径校验 + manifest)[Opus]

**Files:** Create `services/profile_render.rs`(或在 `services/profile.rs` 内新模块);Modify `services/mod.rs`。复用 `config.rs` `atomic_write`/`write_text_file`/`get_claude_config_dir`、`compute`?(用 `sha2` 现有依赖或既有 hash helper——先查 `skill.rs:830 compute_dir_hash` 所用,整文件用同 crate 的 SHA-256)。

- **- [ ] Step 1:** `validate_rel_path(rel_path) -> Result<PathBuf, AppError>`:拒空、拒以 `/` 开头(绝对)、拒任何 `Component::ParentDir`(`..`)或 `RootDir`/前缀;`let p = get_claude_config_dir().join(rel_path)`;`let base = get_claude_config_dir()`;断言 `p.starts_with(&base)`(对 `..` 已拒,无需 canonical 实体存在)。返回 `p`。
- **- [ ] Step 2:** `content_hash(bytes) -> String`(SHA-256 hex,复用项目现有 sha2)。
- **- [ ] Step 3 (TDD,FS `#[serial]`):**
  - `whole_file_writes_and_records_manifest`:profile 含 dotfile `statusline.sh`="echo hi";渲染后文件存在、内容对、有一条 manifest `kind=whole_file,target_path=.../statusline.sh,content_hash=...`。
  - `whole_file_refuses_to_overwrite_unowned_user_file`(**安全**):预置用户 `~/.claude/statusline.sh`="USER"(无 manifest);渲染 profile 的 statusline → 文件**仍为 "USER"**,返回 warning,**无** manifest 行(未接管)。
  - `whole_file_overwrites_own_managed_file`:先渲染(建 manifest+hash),改 profile 内容再渲染 → 文件更新、hash 更新(因有匹配所有权行)。
  - `rel_path_traversal_rejected`:dotfile rel_path=`../evil.sh` 或 `/etc/x` → 不写任何 `~/.claude` 外文件,返回 Err/warn。
- **- [ ] Step 4:** 实现 `render_whole_file(db, app, profile_id, rel_path, content) -> Result<Option<ManifestEntry>, AppError(or warning)>`:`validate_rel_path`;若目标存在 → 查本 profile 的 manifest 是否有该 `target_path` 且**磁盘 hash == 记录 hash**(我方所有权未被用户改);否则(存在但非我方/被改)→ **跳过 + warn**(返回 None + warning,不写不记);可写则 `write_text_file` + 返回新 manifest 行(由调用方 `record_manifest_entry`)。**绝不删未记录文件。**
- **- [ ] Step 5:** `cargo test --lib services::`(绿)+ `clippy --lib` + `fmt`。
- **- [ ] Step 6:** commit `feat(profile): whole-file dotfile renderer (statusline.sh) with refuse-overwrite + path validator + manifest`。

## T5:接线 activate/deactivate(确定性重建 + 整文件 manifest 生命周期)[Opus]

**Files:** Modify `services/profile.rs`(activate/deactivate)。

**activate 新时序**(在 3a 的 load→provider→flag→reconcile 之后):
1. **记出向 profile**:`let outgoing = db.get_active_profile(app_type.as_str())?` —— 在改 active 之前读,用于移除其整文件。
2. (3a 步骤照旧:provider switch / flag flip / 4 reconciler)
3. **整文件渲染**:遍历 incoming profile 的 profile_dotfiles 中 `rel_path != "settings.json"`(settings.json 由确定性重建处理,不走整文件)且非 CLAUDE.md(3b-2)→ 对每个调 `render_whole_file`,成功者 `record_manifest_entry`,warning 收进 `ActivateResult.warnings`。
4. **移除出向整文件**:若 `outgoing` 存在且 `outgoing.id != incoming.id`:取 `get_manifest_for_profile(outgoing.id, app)` 中 `kind=whole_file` 的行,对**不在 incoming 新写集合**的 `target_path`:仅当磁盘 hash == 记录 hash(我方且未被用户改)才 `delete_file` + `delete_manifest_entries`;否则跳过 + warn(用户改过,不删)。
5. `set_active_profile(app, incoming.id)`(**倒数第二**)。
6. **最后:确定性重写 settings.json**:`if let Some(p)=db.get_current_provider(app)? { write_live_with_common_config(db, app, &p)?; } else { warnings.push("无当前 provider，settings.json 未应用 profile 片段") }`。此时 active=incoming,片段为新。
7. `Ok(ActivateResult{warnings})`。

**deactivate 确定性拆除**(替换 3a 的"仅清 flag"):
1. `let cur = db.get_active_profile(app)?`;`db.clear_active_profile(app)?`。
2. 若 `cur` 存在:移除其 `kind=whole_file` manifest 行所记文件(同 hash 守卫:仅删我方未被改者)+ `clear_manifest_for_profile(cur.id, app)`。
3. **重写 settings.json 无 profile 层**:active 已清,`write_live_with_common_config(当前 provider)` 自然只含 provider+common(片段层因 `get_active_profile==None` 跳过)。
> 文档化:3b 起 deactivate 会拆除 dotfile(偏离 3a "无 teardown"),用**确定性重建**而非 diff,无 #1/#3 撤销风险。

- **- [ ] Step 1 (TDD,FS `#[serial]`):**
  - `activate_applies_settings_fragment_and_statusline`:profile 含 settings 片段 + statusline;activate 后两者落盘 + manifest 有 statusline 行;settings.json 含 provider env + 片段。
  - `switch_removes_outgoing_statusline_not_incoming`:P 写 statusline,切到 Q(Q 无 statusline)→ statusline 被删(我方 hash 匹配);切到含不同 statusline 的 R → 旧删新写。
  - `switch_does_not_delete_user_modified_statusline`:P 写 statusline 后用户手改其内容;切走 → **不删**(hash 不匹配)+ warn。
  - `deactivate_tears_down_dotfiles_keeps_provider_keys`:activate P(片段+statusline)→ deactivate → statusline 删除、settings.json 回到 provider+common(无片段)、**provider env/token 仍在**。
  - `activate_missing_provider_warns_no_settings_write`:profile 无 current provider → settings 片段跳过 + warn,不崩。
- **- [ ] Step 2:** 实现;保持 3a 既有不变量(reconciler 仍在 provider/flag 之后;绝不删用户内容文件)。
- **- [ ] Step 3:** `cargo test --lib services::profile`(全绿,含 3a 既有 10 测试无回归)+ `clippy --lib` + `fmt`。
- **- [ ] Step 4:** commit `feat(profile): wire dotfile render + manifest lifecycle into activate/deactivate (deterministic settings rebuild)`。

## T6:Tauri 命令 — dotfiles + manifest 读 [Sonnet]

**Files:** Modify `commands/profile.rs`、`lib.rs`(generate_handler)。

- **- [ ] Step 1:** 加 `#[tauri::command]`(State<AppState> 直委派):`set_profile_dotfile(id, relPath, content)`(校验 rel_path 后 `db.set_profile_dotfile`)、`get_profile_dotfiles(id) -> Vec<ProfileDotfile>`、`delete_profile_dotfile(id, relPath) -> bool`、`get_profile_manifest(id, app) -> Vec<ManifestEntry>`(只读)。
- **- [ ] Step 2:** `lib.rs` generate_handler 的 "// Profile management" 块追加这 4 个。无新 `.manage()`。
- **- [ ] Step 3:** `cargo build` + `cargo test --lib`(无回归)+ `clippy --lib -- -D warnings`(此刻 T2 的 DAO 应已全被消费,须净)+ `fmt`。
- **- [ ] Step 4:** commit `feat(profile): tauri commands for profile dotfiles + manifest read`。

## T7:前端最小新增 — ProfileEditDialog 的 settings.json / statusline 文本框 [Sonnet]

**Files:** Modify `src/lib/api/profiles.ts`、`src/hooks/useProfiles.ts`、`src/components/profiles/ProfileEditDialog.tsx`、`src/i18n/locales/{en,zh,zh-TW,ja}.json`;Modify `tests/components/ProfilesPanel.test.tsx`(或新增 dialog 测试)。

- **- [ ] Step 1:** `profiles.ts` 加 `ProfileDotfile` 类型 + `profilesApi.{getDotfiles(id), setDotfile(id,relPath,content), deleteDotfile(id,relPath)}`(invoke 名对应 T6)。
- **- [ ] Step 2:** `useProfiles.ts` 加 `useProfileDotfiles(id)` query(key `["profiles","dotfiles",id]`)+ `useSetProfileDotfile`/`useDeleteProfileDotfile`(成功失效该 key)。
- **- [ ] Step 3:** `ProfileEditDialog.tsx` 加两个折叠/区块:**settings.json 片段**(多行 textarea,占位提示"JSON 片段,深合并到 provider 之上")与 **statusline.sh**(多行 textarea)。打开编辑时载入 `getDotfiles`;保存时对非空者 `setDotfile(id, "settings.json"/"statusline.sh", content)`,空则 `deleteDotfile`。**不做** `${VAR}` 提示/CLAUDE.md(3b-2)。
- **- [ ] Step 4:** i18n 四语加键:`profiles.settingsFragment`、`profiles.settingsFragmentHint`、`profiles.statusline`、`profiles.statuslineHint`、`profiles.dotfilesSection`(真翻译)。
- **- [ ] Step 5:** 测试:dialog 渲染两个文本框、保存调用 setDotfile。
- **- [ ] Step 6:** `./node_modules/.bin/tsc --noEmit`(净)+ `vitest run`(绿)+ `prettier --write`。
- **- [ ] Step 7:** commit `feat(profile): minimal settings.json/statusline editors in ProfileEditDialog (literal, no \${VAR} yet)`。

## T8:对抗式安全测试汇总 + 全绿 [Opus]

**Files:** Modify `services/profile.rs` / `services/provider/live.rs` 测试模块(补齐/确认)。

- **- [ ] Step 1:** 确认/补齐承载安全测试均在且绿:`provider_settings_config_not_polluted_by_fragment_on_backfill`(T3)、`resync_reproduces_fragment`(T3)、`whole_file_refuses_to_overwrite_unowned_user_file`(T4)、`rel_path_traversal_rejected`(T4)、`switch_does_not_delete_user_modified_statusline`(T5)、`deactivate_tears_down_dotfiles_keeps_provider_keys`(T5)、`activate_missing_provider_warns_no_settings_write`(T5)。若有缺口补测试。
- **- [ ] Step 2:** 全套:`cargo fmt --check` + `cargo clippy --all-targets -- -D warnings` + `cargo test`(全量,含集成)+ `tsc --noEmit` + `vitest run`。全绿。
- **- [ ] Step 3:** 独立确认:`profile_dotfiles`/`manifest` DAO 已被消费(无 dead_code);settings.json 仍只由 effective-settings 单写路径产出(grep 无第二处 settings.json 写)。
- **- [ ] Step 4:** Opus 总评审 `main..profiles-inc3b1`:① settings.json 确定性重建无 provider 污染(backfill 剥离生效)② 整文件拒覆盖 + 路径校验 + manifest 只删我方 ③ deactivate 确定性拆除保 provider 键 ④ 时序(set_active 倒二、最后显式 settings 写)⑤ 无 3b-2/3c scope 泄漏(无 `${VAR}`、无 CLAUDE.md prompt 逻辑、无 @tag)⑥ 无回归。
- **- [ ] Step 5:** finishing-a-development-branch:本地 `--no-ff` 合并 main,合并后复验全绿,删分支。

---

## 自查 / 一致性

- **spec 覆盖**:对应增量 3 愿景的 dotfiles(settings.json + statusline)之安全落地与撤销;`${VAR}`/CLAUDE.md 在 3b-2。
- **类型一致**:ProfileDotfile/ManifestEntry(Rust↔TS camelCase);命令名 `set_profile_dotfile`/`get_profile_dotfiles`/`delete_profile_dotfile`/`get_profile_manifest` ↔ profilesApi ↔ query key。
- **占位符**:无;每步给实体代码或精确锚点。
- **schema 序列**:profiles=v13(3a)→ apply_manifest.content_hash=**v14**(3b-1)。
- **对抗修复落点**:CRITICAL(backfill 污染)→ T3 反向剥离 + 时序;CRITICAL(第二写不可见)→ T3 织入单写路径 + T5 最后显式重写;HIGH(deep_remove collision)→ 确定性重建,前向不 remove,反向镜像 common-config 并文档化;HIGH(statusline 覆盖)→ T4 拒覆盖;MEDIUM(路径校验)→ T4 validate_rel_path;MEDIUM(deactivate)→ T5 确定性拆除;LOW(manifest 出向识别)→ T5 先读 outgoing;LOW(v14 幂等)→ T1 版本门 + PRAGMA 断言。
- **不做**:见范围。
