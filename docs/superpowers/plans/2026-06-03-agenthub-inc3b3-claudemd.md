# agenthub 增量 3b-3 — CLAUDE.md 仲裁(经 PromptService 委派的隐藏 prompt 行)Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: superpowers:subagent-driven-development。每步 `- [ ]`。提交前 `cargo fmt` / `prettier --write`。

**Goal:** 让 profile 携带一份 **CLAUDE.md**(`~/.claude/CLAUDE.md`)。**CLAUDE.md 用字面文本、不渲染 `${VAR}`**(渲染会把密钥落进 DB `prompts.content`——3b-2 对抗审查的 CRITICAL)。profile **不直接写** CLAUDE.md,而是**驱动 PromptService**(它是该文件的唯一整文件写者):activate 时把 profile 的 CLAUDE.md 文本灌入一个**派生的隐藏 prompt 行** `__profile__:<profile_id>` 并 `enable_prompt` 它;deactivate 拉黑;delete 清理。schema 升 v15 加 `prompts.hidden`,UI 列表隐藏该行,而 PromptService 内部(单一启用扫描/备份/删除守卫)看得见它。

**Architecture(经对抗审查修正,务必照此):**
1. **唯一写者**:`~/.claude/CLAUDE.md` 只由 PromptService 写。profile 仅经隐藏行驱动它。CLAUDE.md 文本存 `profile_dotfiles`(rel_path=="CLAUDE.md",字面,activate step5 已 `continue` 跳过整文件渲染)。隐藏行 `content` = 该字面文本(**无 `${VAR}` 渲染**,故 `prompts.content` 永不含渲染密钥)。
2. **隐藏行**:id=`format!("__profile__:{}", profile_id)`,`hidden=true`,app_type=profile.app_type。**硬门 `AppType::Claude`**:仅 Claude profile 走 CLAUDE.md 生命周期(CLAUDE.md 是 Claude 专属;Codex profile 带 CLAUDE.md dotfile 必须 no-op,绝不把内容写进 `~/.codex/AGENTS.md`)。
3. **v15 `prompts.hidden`**:`get_prompts`(UI/首launch import)**过滤 hidden=0**;新增 `get_prompts_with_hidden` / `get_prompt_with_hidden`(内部用,看得见隐藏行)。**enable_prompt 的 backfill 扫描 + 单一启用 sweep、upsert_prompt 的 any_enabled、delete_prompt 守卫**全部改用 `_with_hidden`,否则隐藏行的 enabled 标志不会被清/被数 → 双启用或错误清空文件。
4. **clobber 防护**:enable_prompt phase-1 的 live-backfill,若当前启用行是 `__profile__:` 隐藏行 → **跳过 backfill**(profile 模板权威,绝不把 live 手改灌回模板;用户手改在切到该 profile 前已由 backfill 存为可见备份行,可恢复)。隐藏行**只能经 enable_prompt 启用**(走 sweep),**绝不**用 `upsert_prompt(enabled=true)`(那会绕过单一启用 sweep)。
5. **activate 时序**:CLAUDE.md 步(step 5b)放在 **set_active(step7)之后、settings 重建(step8)之前**——此时 INCOMING profile 已是 active,语义正确。
6. **空内容/清空**:前端 CLAUDE.md textarea 沿用 3b-1 的"非空 setDotfile / 空 deleteDotfile"模式。step 5b:`get_profile_dotfile("CLAUDE.md")` 为 None(或空)→ 若存在 `__profile__:` 隐藏行则禁用它(经 upsert_prompt(false),any_enabled 用 `_with_hidden` 决定是否清空文件;**有其它可见启用 prompt 时不清空**)。

**Tech Stack:** 同前。仓库 `/Users/fsm/project/MyProject/agentplugin/cc-switch-cloud`(从 main `d86a7d66` 开分支 `profiles-inc3b3`)。环境 gotcha 同前(cargo PATH、`.cargo/config.toml`、前端 `./node_modules/.bin/{vitest,tsc,prettier}`、慢网前缀、触磁盘测试 `CC_SWITCH_TEST_HOME`+`#[serial]`、提交前 fmt、最终门 `clippy --all-targets`、别用 grep -c 收尾、**判死代码须 grep `tests/`**)。

**复用锚点(已核实):** `services/prompt.rs`(enable_prompt 73-144、upsert_prompt 28-58、delete_prompt 60-71、import_from_file_on_first_launch 192)、`prompt_files.rs prompt_file_path`、`dao/prompts.rs`(get_prompts/get_prompt/save_prompt/delete_prompt + mapper)、`database/schema.rs`(CREATE TABLE prompts、apply_schema_migrations_on_conn、`add_column_if_missing`、migrate_v13_to_v14 模板)、`services/profile.rs`(activate step5 跳过 CLAUDE.md / step7 set_active / step8 重建、deactivate)、`commands/profile.rs delete_profile` + `dao/profiles.rs delete_profile`、`db.get_profile_dotfile(id,"CLAUDE.md")`、前端 ProfileEditDialog(3b-1 dotfile 文本框 + 3b-2 vars 编辑器,空则 deleteDotfile 模式)。

**安全(T7 对抗测试守护):** ① CLAUDE.md 唯一写者(PromptService);交织(activate P → UI 启用普通 prompt N → 再 activate P)无内容丢失/模板污染;② 用户手改 CLAUDE.md 在 profile 覆盖前已备份(复用现有 backfill,对**非 profile 行**);③ 隐藏行不进 Prompts UI 列表;enable sweep/any_enabled/delete guard 看得见隐藏行;④ CLAUDE.md 字面 → `prompts.content` 无渲染密钥;⑤ 非 Claude profile no-op;⑥ v15 幂等;⑦ deactivate 正确清空(含他行存在时不误清)、delete 清隐藏行无孤儿;⑧ 不回归 3a/3b-1/3b-2。

---

## 范围 / 不做

- **做**:v15 `prompts.hidden` + Prompt.hidden 字段(5 处字面 + DAO mapper);DAO `get_prompts`(过滤)+ `get_prompts_with_hidden`/`get_prompt_with_hidden`;PromptService 隐藏路由 + backfill 跳过守卫 + delete 守卫;ProfileService activate step5b(Claude-gated)+ deactivate CLAUDE.md 拉黑;delete_profile 隐藏行清理;前端 CLAUDE.md textarea(字面,空则删)+ i18n。
- **不做**:`${VAR}` 渲染 CLAUDE.md(故意字面);@tag/skills.tags(3c);项目通道(inc4);非 Claude 的 prompt-file 驱动(将来如需,按 app 映射 rel_path)。Prompts UI "由 profile 接管" 横幅(可选,默认不做)。

---

## T1:v15 迁移 prompts.hidden + Prompt.hidden 字段(含全部字面 + mapper)[Opus]

**Files:** Modify `database/mod.rs`(SCHEMA_VERSION 14→15)、`database/schema.rs`(CREATE TABLE prompts + migrate_v14_to_v15 + `14 =>` arm)、`database/tests.rs`;Modify `app_config.rs`(Prompt 结构 + 其内的字面)、`database/dao/prompts.rs`(mapper 读列 + save 写列)、`services/prompt.rs`(2 处 Prompt{} 字面)、`deeplink/prompt.rs`(1 处)、`app_config.rs`(legacy import 1 处)。

- **- [ ] Step 1:** `Prompt` 结构加 `#[serde(default)] pub hidden: bool`(serde default 覆盖 JSON 反序列化的旧数据)。
- **- [ ] Step 2 (TDD):** tests.rs 加 `prompts_has_hidden_after_migrate`:内存 conn → create_tables → `set_user_version(13)`? 不,从 14:`set_user_version(&conn,14)` → `apply_schema_migrations_on_conn` → 断言 `get_user_version==15` 且 `PRAGMA table_info(prompts)` 含 `hidden`。**经版本门 runner 驱动**(勿二次 ALTER)。跑确认 FAIL。
- **- [ ] Step 3:** mod.rs:52 `SCHEMA_VERSION = 15`。
- **- [ ] Step 4:** schema.rs CREATE TABLE prompts 末尾加 `hidden BOOLEAN NOT NULL DEFAULT 0`(使全新 v15 == v14+迁移)。
- **- [ ] Step 5:** 加 `fn migrate_v14_to_v15(conn) -> Result<(), AppError> { Self::add_column_if_missing(conn, "prompts", "hidden", "BOOLEAN NOT NULL DEFAULT 0")?; log::info!("v14 -> v15 migration done: prompts.hidden"); Ok(()) }`;`apply_schema_migrations_on_conn` 在 `13 =>` arm 后、`_ =>` 前插 `14 => { log; Self::migrate_v14_to_v15(conn)?; Self::set_user_version(conn, 15)?; }`。
- **- [ ] Step 6:** `dao/prompts.rs`:`save_prompt` 的 INSERT 加 `hidden` 列;**所有** row→Prompt mapper(get_prompts 等)读 `hidden`(注意列索引)。
- **- [ ] Step 7:** 给**全部 5 处** `Prompt{...}` 字面加 `hidden: false`:`services/prompt.rs`(backup_prompt ~103、import_from_file ~158、import_from_file_on_first_launch ~223)、`deeplink/prompt.rs`(~64)、`app_config.rs`(legacy JSON auto-import ~896)。grep `Prompt {` 全仓确认无遗漏(否则编译失败)。
- **- [ ] Step 8:** `cargo build` + `cargo test --lib database:: prompt`(绿)+ `cargo fmt`。
- **- [ ] Step 9:** commit `feat(db): prompts.hidden column (schema v15) + Prompt.hidden field`。

## T2:DAO hidden-aware 读方法 + 调用点过滤 [Opus]

**Files:** Modify `database/dao/prompts.rs`。

- **- [ ] Step 1:** `get_prompts(app_type)` 改为 **`WHERE app_type=?1 AND hidden=0`**(UI 安全:隐藏行永不出现在列表/首launch import 幂等检查)。
- **- [ ] Step 2:** 新增 `get_prompts_with_hidden(app_type) -> Result<IndexMap<String,Prompt>, AppError>`(无 hidden 过滤,内部用)+ `get_prompt_with_hidden(app_type, id) -> Result<Option<Prompt>, AppError>`(单行,看得见隐藏)。三个方法共用同一 mapper(读 hidden 列)。
- **- [ ] Step 2.5 (TDD):** `Database::memory()` 测试:插一个 hidden=1 行 + 一个 hidden=0 行;`get_prompts` 只返回可见行;`get_prompts_with_hidden` 两者都返回;`get_prompt_with_hidden(hidden_id)` 返回 Some。
- **- [ ] Step 3:** `cargo test --lib database::`(绿)+ `clippy --lib`(新方法将由 T3 消费,瞬时 dead_code 可接受)+ `cargo fmt`。
- **- [ ] Step 4:** commit `feat(db): get_prompts filters hidden=0; add get_prompts_with_hidden / get_prompt_with_hidden`。

## T3:PromptService 隐藏路由 + backfill 跳过 + delete 守卫 [Opus]

**Files:** Modify `services/prompt.rs`。**改动点(精确)**:enable_prompt(73-144)、upsert_prompt(28-58)、delete_prompt(60-71)。**注意:`import_from_file_on_first_launch`(192)与 `PromptService::get_prompts`(25,喂 UI)保持用过滤版 `get_prompts`(不变)。**

- **- [ ] Step 1 (enable_prompt):**
  - phase-1 backfill 扫描(~79)`get_prompts` → **`get_prompts_with_hidden`**(以便当前启用行可能是隐藏 profile 行被识别)。
  - 找到 enabled_id 后(~82),**若 `enabled_id.starts_with("__profile__:")` → 跳过 backfill**(模板权威;不把 live 灌回隐藏行)。否则走原 backfill。
  - 单一启用 sweep(~124)`get_prompts` → **`get_prompts_with_hidden`**(使隐藏行的 enabled 标志也被清/被设;启用普通 prompt 时隐藏行会被关掉)。
- **- [ ] Step 2 (upsert_prompt):** any_enabled 判定(~44-45)`get_prompts` → **`get_prompts_with_hidden`**。**保持现有顺序**:先 `save_prompt`(37)再 `get_prompts_with_hidden` 重查(45)——这样刚禁用的隐藏行已落库为 disabled,被正确计入 → 该 profile 的 CLAUDE.md 是唯一启用且被禁用时,文件被正确清空。
- **- [ ] Step 3 (delete_prompt):** 守卫(~62)`get_prompts` → **`get_prompts_with_hidden`**(否则删隐藏行时守卫看不到、或漏判 enabled)。
- **- [ ] Step 4 (硬规则,文档化):** 隐藏行**只经 enable_prompt 启用**;禁止 `upsert_prompt(enabled=true)` 启用隐藏行(会绕过 sweep 破坏单一启用)。在代码注释写明。
- **- [ ] Step 5 (TDD,触磁盘 `#[serial]`):**
  - `enable_normal_prompt_disables_hidden_profile_row`:DB 有一个 enabled 的 `__profile__:p` 隐藏行(CLAUDE.md=profile 文本);`enable_prompt(N)` 后:隐藏行 enabled=false(经 `_with_hidden` 查证)、文件=N 内容、恰好一行 enabled。
  - `enable_hidden_row_skips_backfill_does_not_capture_user_edit`:`__profile__:p` 启用中,用户手改 live CLAUDE.md;再 `enable_prompt("__profile__:p")` → 隐藏行 content **不**被 live 手改污染(仍是模板内容)。
  - `disable_lone_hidden_row_blanks_file`:仅隐藏行 enabled;`upsert_prompt(hidden,enabled=false)` → 文件=""。
  - `disable_hidden_row_keeps_file_when_visible_prompt_enabled`:隐藏行 + 一个可见 enabled prompt;禁用隐藏行 → 文件不被清空(any_enabled 经 `_with_hidden` 仍 true)。
- **- [ ] Step 6:** `cargo test --lib services::prompt`(绿)+ `cargo test --lib`(无回归)+ `clippy --lib -- -D warnings` + `cargo fmt`。
- **- [ ] Step 7:** commit `feat(prompt): hidden-row aware enable/upsert/delete (single-enabled sweep + backfill skip)`。

## T4:ProfileService activate step5b + deactivate(CLAUDE.md,Claude-gated)[Opus]

**Files:** Modify `services/profile.rs`(activate、deactivate)。

- **- [ ] Step 1 (activate step 5b):** 在 set_active(step7)之后、settings 重建(step8)之前,加 CLAUDE.md 步,**整步 `if matches!(app_type, AppType::Claude)` 硬门**:
  - `let hidden_id = format!("__profile__:{}", profile_id);`
  - `match state.db.get_profile_dotfile(profile_id, "CLAUDE.md")?`:
    - `Some(df) if !df.content.trim().is_empty()` → 构造/更新隐藏行 `Prompt{ id: hidden_id, app_type:"claude", name: format!("(profile) {}", profile.name), content: df.content(字面), description: Some("AgentHub profile-managed CLAUDE.md (hidden)"), enabled:false(由 enable_prompt 置), hidden:true, created_at/updated_at: now }`,`save_prompt` 落库,然后 `PromptService::enable_prompt(state, AppType::Claude, &hidden_id)?`(走 sweep + backfill-skip)。
    - `None`(或空)→ 若 `get_prompt_with_hidden("claude", &hidden_id)?` 存在,`PromptService::upsert_prompt(state, AppType::Claude, &hidden_id, {该行 enabled=false})?`(禁用;any_enabled 决定是否清空)。无隐藏行则 no-op。
  - **warn 校验**:enable 后,`get_prompts_with_hidden("claude")` 中 enabled 行 id 若 != hidden_id → `result.warnings.push("CLAUDE.md not driven by active profile (another prompt enabled)")`。
- **- [ ] Step 2 (deactivate):** 在现有拆除步骤中,若 `cur`(原 active)存在且为 Claude,禁用其隐藏行:`get_prompt_with_hidden("claude", &format!("__profile__:{}", cur.id))?` 若 Some → `upsert_prompt(enabled=false)`(经 `_with_hidden` 的 any_enabled 决定清空)。
- **- [ ] Step 3 (TDD,`#[serial]`):**
  - `activate_claude_profile_writes_claude_md_via_hidden_row`:profile dotfile CLAUDE.md="hello"; activate → `~/.claude/CLAUDE.md`=="hello"; `get_prompts("claude")`(UI)**不**含隐藏行;`get_prompts_with_hidden` 含且 enabled。
  - `non_claude_profile_with_claude_md_is_noop`:codex profile 带 CLAUDE.md dotfile;activate → **不**创建隐藏行、**不**写 `~/.codex/AGENTS.md`(grep 无)。
  - `switch_claude_profiles_repoints_claude_md`:A(CLAUDE.md="A")→B(CLAUDE.md="B");文件=="B";A 隐藏行 disabled、B 隐藏行 enabled。
  - `deactivate_blanks_claude_md`:activate A(CLAUDE.md)→ deactivate → 文件=""(无其它启用 prompt)。
  - `reactivate_same_profile_claude_md_no_clobber`:activate A;用户手改 live;re-activate A → 文件==A 模板(模板权威),A 隐藏行 content 未被污染。
  - `interleave_enable_normal_prompt_then_reactivate`:activate A → UI `enable_prompt(N)` → re-activate A → 文件==A 模板、恰一行 enabled、N 的 content 未丢(其行仍在)。
- **- [ ] Step 4:** `cargo test --lib services::profile`(全绿,含 3a/3b-1/3b-2 无回归)+ `clippy --lib -- -D warnings` + `cargo fmt`。
- **- [ ] Step 5:** commit `feat(profile): activate/deactivate CLAUDE.md via hidden prompt row (Claude-gated, literal)`。

## T5:delete_profile 隐藏行清理 [Sonnet]

**Files:** Modify `commands/profile.rs delete_profile`(或 services 层),复用 `dao/profiles.rs delete_profile`。

- **- [ ] Step 1:** delete_profile 改为(**不调用完整 deactivate**,避免再入/非 Claude 报错):
  - `let Some(profile) = state.db.get_profile(&id)? else { return state.db.delete_profile(&id).map(|_| true) ...}`(None 安全:直接删返回)。
  - 若该 profile 是 Claude 且为当前 active(`get_active_profile("claude") == Some(id)`):`let hidden_id = format!("__profile__:{}", id); if get_prompt_with_hidden("claude",&hidden_id)?.is_some() { upsert_prompt(state, Claude, &hidden_id, {enabled=false}) }`(Claude-gated;拉黑→经 any_enabled 清空文件)。
  - `if profile.app_type=="claude" { state.db.delete_prompt? }`—— 但 delete_prompt 守卫拒删 enabled 行,故须先确保上一步已禁用;用 `db.delete_prompt("claude", &hidden_id)`(DAO 层直删,绕过 service 守卫,因隐藏行是我方管理)清隐藏行,**无孤儿**。
  - 最后 `state.db.delete_profile(&id)`(CASCADE profile_dotfiles/manifest)。
- **- [ ] Step 2 (TDD,`#[serial]`):** `delete_active_claude_profile_blanks_and_removes_hidden_row`:activate A(CLAUDE.md)→ delete A → 文件=""、`get_prompt_with_hidden("claude","__profile__:A")`==None、profile 没了。`delete_nonexistent_profile_is_ok`(None 路径)。
- **- [ ] Step 3:** `cargo test --lib`(无回归)+ `clippy --lib -- -D warnings` + `cargo fmt`。
- **- [ ] Step 4:** commit `feat(profile): delete_profile cleans up hidden CLAUDE.md prompt row`。

## T6:前端 CLAUDE.md textarea(字面,空则删)+ i18n [Sonnet]

**Files:** Modify `src/components/profiles/ProfileEditDialog.tsx`、`src/i18n/locales/{en,zh,zh-TW,ja}.json`;测试。

- **- [ ] Step 1:** ProfileEditDialog 的 dotfiles 区加第三个 textarea **CLAUDE.md**(rel_path "CLAUDE.md"),与 settings.json/statusline 同款:编辑态载入 `getDotfiles` 预填;保存时**非空 → `setDotfile(id,"CLAUDE.md",text)`;空 → `deleteDotfile(id,"CLAUDE.md")`**(沿用 3b-1 模式,使清空生效)。**无 `${VAR}` 提示**(CLAUDE.md 字面)。
- **- [ ] Step 2:** i18n 四语加 `profiles.claudeMd`、`profiles.claudeMdHint`("Literal CLAUDE.md for this profile; drives ~/.claude/CLAUDE.md via the prompt system when active. No \${VAR} substitution.")(真翻译)。
- **- [ ] Step 3:** vitest:CLAUDE.md textarea 渲染、保存非空调 setDotfile、清空调 deleteDotfile。
- **- [ ] Step 4:** `tsc --noEmit`(净)+ `vitest run`(绿)+ `prettier --write`。
- **- [ ] Step 5:** commit `feat(profile): CLAUDE.md textarea (literal) in ProfileEditDialog + i18n`。

## T7:全绿 + Opus 总评审 + 收尾 [Opus]

- **- [ ] Step 1:** 全套:`cargo fmt --check` + `cargo clippy --all-targets -- -D warnings` + `cargo test`(全量,含集成)+ `tsc --noEmit` + `vitest run`。全绿。**特别核实集成测试编译**(prompts 改动 + 5 处字面)。
- **- [ ] Step 2:** 独立确认:CLAUDE.md 字面(grep:无对 CLAUDE.md 内容的 `${VAR}`/substitute 调用);隐藏行不进 `get_prompts`(过滤版);非 Claude no-op;v15 幂等(经 runner)。grep `Prompt {` 全仓均含 hidden。
- **- [ ] Step 3:** Opus 总评审 `main..profiles-inc3b3`:① 唯一写者 + 交织无 clobber + 模板权威 ② 隐藏行不泄漏 UI、内部 `_with_hidden` 全覆盖(enable sweep/backfill/any_enabled/delete)③ 非 Claude 硬门 ④ delete 无孤儿、None 安全 ⑤ 字面无密钥 ⑥ v15 幂等 ⑦ 不回归 3a/3b-1/3b-2(尤其 3b-2 ${VAR} settings/statusline、no-bleed)⑧ 无 @tag/项目通道泄漏。
- **- [ ] Step 4:** finishing-a-development-branch:本地 `--no-ff` 合并 main,合并后复验全绿,删分支。

---

## 自查 / 一致性

- **spec 覆盖**:增量 3 愿景的 CLAUDE.md dotfile(经 prompt 子系统仲裁)。`${VAR}` 故意不用于 CLAUDE.md(安全)。
- **类型/命名一致**:`Prompt.hidden`(Rust↔serde default);`get_prompts`(过滤)/`get_prompts_with_hidden`/`get_prompt_with_hidden`;隐藏行 id `__profile__:<id>`;前端 setDotfile/deleteDotfile("CLAUDE.md")。
- **占位符**:无;每步实体代码或精确锚点(含 5 处 Prompt{} 字面位置)。
- **schema**:prompts.hidden=**v15**(v13→v14=content_hash,v14→v15=hidden)。
- **对抗修复落点**:非 Claude 硬门→T4 step1;5 处字面+mapper→T1 step6/7;_with_hidden 全覆盖(enable sweep/backfill/any_enabled/delete)→T2/T3;backfill 跳过隐藏行→T3 step1;upsert save-before-scan→T3 step2;delete 不调 deactivate + None 安全→T5;清空=前端 deleteDotfile→T6。
- **不做**:见范围。CLAUDE.md 字面(无 ${VAR});隐藏行只经 enable_prompt 启用。
