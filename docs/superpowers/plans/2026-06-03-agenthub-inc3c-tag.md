# agenthub 增量 3c — @tag 支持(Profiles 旗舰收官)Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: superpowers:subagent-driven-development。每步 `- [ ]`。提交前 `cargo fmt` / `prettier --write`。

**Goal:** 让 profile 的内容列表(skills/commands/agents/mcp)支持 **`@tagname`** 选择器:激活时 `@tag` 展开成该内容类型中**标签含 tagname 的全部项**,与字面名取并集后翻 enable-flag。补齐 `skills.tags` 列(v16);commands/agents/mcp 已有 tags 列。**完成后 Profiles 旗舰(增量 3)收官。**

**Architecture(经对抗审查修正):**
1. **skills.tags(v16)**:`TEXT NOT NULL DEFAULT '[]'`(同 mcp_servers.tags,never NULL,解析简单)。`InstalledSkill.tags: Vec<String>`(`#[serde(default)]`,同 commands/agents)。DAO row-mapper 按 **`String`** 读(`let tags_str: String = row.get(idx)?; serde_json::from_str(&tags_str).unwrap_or_default()`,mcp 模式,非 commands/agents 的 `Option<String>+parse_tags` 可空模式)。**5 处 `InstalledSkill{}` 字面**都要加 tags 字段。
2. **@tag 解析(内存,激活期)**:profile spec.content 的每个条目,**前缀 `@` 永远是 tag 选择器**(无转义;本仓库无 skill.directory/command.name/agent.name/mcp.id 以 @ 开头);tag 名 = `@` 之后部分。激活 step 3 的每个 flag-flip 循环里,先把 spec 列表解析成 `want` 集:字面条目原样;`@tag` 条目展开成「已载入行中 tags 含该 tag 的 key 集」;并集。**大小写敏感精确匹配**(与现有字面名匹配一致)。空 `@`/未知 tag → 零展开 + 警告(**可区分**字面 miss 与 tag miss),绝不报错。复用 activate 已载入的全部行(skills/commands/agents/mcp),**不用 json_each**。
3. **skills 标签编辑**:skills 现无编辑对话框(仅安装/导入 + toggle)。加 `update_skill_tags(id, tags)` Tauri 命令(纯元数据,**不触发 SkillService 落盘**)+ SkillCard 标签输入。否则 skills 的 @tag 在 UI 不可达。
4. **前端 @tag 可发现性**:profile 内容列表已是逗号文本框,`@tag` 直接作为字符串条目;仅更新 placeholder/help i18n 说明可填 `@tagname`。

**HIGH 数据丢失防护(对抗审查)**:`update_skill`(skill.rs:1090)从已载入 `skill` 重建——**必须 `tags: skill.tags.clone()`**,否则每次更新清空用户标签(且 @tag 依赖标签)。

**Tech Stack:** 同前。仓库 `/Users/fsm/project/MyProject/agentplugin/cc-switch-cloud`(从 main `13e1118d` 开分支 `profiles-inc3c`)。环境 gotcha 同前(cargo PATH、`.cargo/config.toml`、前端 `./node_modules/.bin/{vitest,tsc,prettier}`、慢网前缀、触磁盘测试 `CC_SWITCH_TEST_HOME`+`#[serial]`、提交前 fmt、最终门 `clippy --all-targets`、判死代码须 grep `tests/`、别用 grep -c 收尾)。

**复用锚点(已核实):** commands/agents `parse_tags`(dao/commands.rs:151)+ `serde_json::to_string(&tags)` save;mcp `let tags_str: String = row.get(6)?; from_str.unwrap_or_default()`(dao/mcp.rs:29-37);`add_column_if_missing` + `migrate_v14_to_v15` 模板(schema.rs);activate step 3 的 4 个 flag-flip 循环(services/profile.rs);`InstalledSkill` + dao/skills.rs;create/update_command 的 `tags: Vec<String>` 参数 + 前端逗号分割 UI(CommandEditDialog);ProfileEditDialog 的 `stringToList = split(",").map(trim).filter(Boolean)`。

**安全/正确性(T6 对抗测试守护):** ① 纯字面 spec 与 3a/3b 行为**完全一致**(无回归);② @tag 展开正确(并集去重、未匹配项不丢)、未知/空 @tag 警告非报错、大小写敏感;③ skills.tags 全读写点更新、`update_skill` 不清空标签、v16 幂等 + backfill;④ tags 解析容错(null/空/坏 JSON → 空 Vec 不 panic);⑤ 无新内容类型/项目通道。

---

## 范围 / 不做

- **做**:skills.tags(v16)+ InstalledSkill.tags + DAO 全读写 + update_skill_tags 命令;@tag 解析接入 activate 的 4 个循环;前端 skills 标签编辑器 + @tag placeholder 提示 + i18n。
- **不做**:新内容类型;项目通道(inc4);@tag 转义/前缀逃逸(文档化为已知限制——名字不以 @ 开头);大小写不敏感(故意敏感);富标签 UX(自动补全/标签页/在 commands/agents/mcp 列表显示标签)。
- **已知限制(文档化)**:字面项名以 `@` 开头将被当作 tag 选择器(本仓库无此情况;在安装/导入处或 help 文案注明)。

---

## T1:v16 迁移 + skills.tags 列 [Opus]

**Files:** Modify `database/mod.rs`(SCHEMA_VERSION 15→16)、`database/schema.rs`(CREATE TABLE skills + migrate_v15_to_v16 + `15 =>` arm)、`database/tests.rs`。

- **- [ ] Step 1 (TDD):** tests.rs 加 `skills_has_tags_after_migrate`:内存 conn → create_tables_on_conn → **在 skills 插一行(v15 形,无 tags)** → `set_user_version(&conn, 15)` → `apply_schema_migrations_on_conn` → 断言 `get_user_version==16` 且 `PRAGMA table_info(skills)` 含 `tags`;**并 backfill 断言**:`get_installed_skill(那行)` 的 tags == `vec![]`(对抗 LOW:旧行迁移后默认 '[]')。经版本门 runner(勿二次 ALTER)。跑确认 FAIL。
  > 注:若 create_tables 已建 v16 形(含 tags),则插行须显式不含 tags 列;或先 set_user_version(15) 再插。实现者按现有迁移测试惯例处理(参考 prompts_has_hidden_after_migrate)。
- **- [ ] Step 2:** mod.rs:52 `SCHEMA_VERSION = 16`。
- **- [ ] Step 3:** schema.rs CREATE TABLE skills 加 `tags TEXT NOT NULL DEFAULT '[]'`(全新 v16 == v15+迁移)。
- **- [ ] Step 4:** 加 `fn migrate_v15_to_v16(conn) -> Result<(), AppError> { if Self::table_exists(conn, "skills")? { Self::add_column_if_missing(conn, "skills", "tags", "TEXT NOT NULL DEFAULT '[]'")?; } log::info!("v15 -> v16 migration done: skills.tags"); Ok(()) }`(镜像 migrate_v14_to_v15 的 table_exists 守卫)。`apply_schema_migrations_on_conn` 在 `14 =>` arm 后、`_ =>` 前插 `15 => { log; Self::migrate_v15_to_v16(conn)?; Self::set_user_version(conn, 16)?; }`。
- **- [ ] Step 5:** `cargo test --lib database::`(全绿,含新测试)+ `cargo fmt`。
- **- [ ] Step 6:** commit `feat(db): skills.tags column (schema v16)`。

## T2:InstalledSkill.tags + skills DAO 全读写 + 5 处字面 [Sonnet]

**Files:** Modify `app_config.rs`(InstalledSkill)、`database/dao/skills.rs`(读/写/可能新增 update helper)、`services/skill.rs`(**全部 5 处 InstalledSkill 字面**)。蓝本:commands/agents 的 tags 字段 + mcp 的 String 读法。

- **- [ ] Step 1:** `InstalledSkill` 加 `#[serde(default)] pub tags: Vec<String>`(同 InstalledCommand.tags 风格)。
- **- [ ] Step 2:** `dao/skills.rs`:
  - `get_all_installed_skills` / `get_installed_skill` 的 SELECT 加 `tags` 列;row-mapper 按 **String** 读:`let tags_str: String = row.get(<idx>)?; let tags = serde_json::from_str(&tags_str).unwrap_or_default();`(因列 NOT NULL DEFAULT '[]',用 mcp 模式,非 Option)。
  - skill 保存路径(save_skill / 等 upsert)的 INSERT/REPLACE 加 `tags` 列,值 `serde_json::to_string(&skill.tags)`。
  - 加 `update_skill_tags(&self, id: &str, tags: &[String]) -> Result<(), AppError>`(`UPDATE skills SET tags=?2 WHERE id=?1`,tags JSON)。供 T4 用。
- **- [ ] Step 3(改全部 5 处 `InstalledSkill{}` 字面,否则编译失败 + HIGH 数据丢失):**
  - `services/skill.rs:748`(install)→ `tags: vec![]`。
  - **`services/skill.rs:1090`(update,从已载入 `skill` 重建)→ `tags: skill.tags.clone()`**(**绝不 `vec![]`**——否则每次更新清空用户标签)。
  - `services/skill.rs:1524`(import)→ `tags: vec![]`。
  - `services/skill.rs:3099`(import)→ `tags: vec![]`。
  - `services/skill.rs:2712`(测试)→ `tags: vec![]`。
  > 先 `grep -rn "InstalledSkill {" src` 复核全部站点;行号可能偏移,以 grep 为准。
- **- [ ] Step 4 (TDD):** `dao/skills.rs` 加 round-trip 测试(harness 从 `dao/commands.rs` 的测试拷贝:`Database::memory()`):save 一个带 tags 的 skill → get 读回 tags 一致;`update_skill_tags` 改 tags → 读回新值;旧无 tags 行(模拟)→ 空 Vec。
- **- [ ] Step 5:** `cargo build`(编译过 = 5 字面齐)+ `cargo test --lib`(skills 既有测试 + WebDAV 导出无回归)+ `clippy --lib -- -D warnings` + `cargo fmt`。
- **- [ ] Step 6:** commit `feat(skill): InstalledSkill.tags + dao read/write/update_skill_tags`。

## T3:@tag 解析接入 ProfileService::activate [Opus] —— 收官皇冠

**Files:** Modify `services/profile.rs`(activate step 3 的 4 个 flag-flip 循环)。

**解析辅助(放 profile.rs 或 profile_vars 旁,纯函数):**
```rust
/// 把 spec 列表解析为 want 集 + 收集未匹配/未知警告。
/// items: (key, tags) 列表（key=该类型的标识：skill.directory / command.name / agent.name / mcp.id）。
/// 返回 (want: HashSet<String>, warnings: Vec<String>)。
fn resolve_selectors(
    type_label: &str,                      // "command"/"agent"/"skill"/"mcp"，用于警告文案
    spec: &[String],
    items: &[(String, Vec<String>)],       // (key, tags)
) -> (std::collections::HashSet<String>, Vec<String>) {
    use std::collections::HashSet;
    let mut want: HashSet<String> = HashSet::new();
    let mut warnings = Vec::new();
    for entry in spec {
        if let Some(tag) = entry.strip_prefix('@') {
            // @tag 选择器（前缀 @ 永远是选择器；空 tag 也走这里）
            let matched: Vec<&String> = items.iter()
                .filter(|(_, tags)| tags.iter().any(|t| t == tag))   // 大小写敏感精确
                .map(|(k, _)| k).collect();
            if matched.is_empty() {
                warnings.push(format!("tag matched no {type_label}s: @{tag}"));
            } else {
                for k in matched { want.insert(k.clone()); }
            }
        } else {
            // 字面：仅当存在该 key 才计入；否则警告（区别于 tag-miss）
            if items.iter().any(|(k, _)| k == entry) {
                want.insert(entry.clone());
            } else {
                warnings.push(format!("{type_label} not found: {entry}"));
            }
        }
    }
    (want, warnings)
}
```
（注:3a 原循环可能是「遍历 DB 行,flag = (key ∈ want)」并对未匹配字面名 warn。改为:先调 `resolve_selectors` 得到 want + warnings,push warnings 进 result;再用 want 翻 flag。**纯字面 spec → want 与 3a 完全一致,无回归**。)

- **- [ ] Step 1:** 在 activate step 3 的每个类型循环里接入:
  - commands:`items` = `get_all_installed_commands()` 映射 `(c.name, c.tags)`;`resolve_selectors("command", &spec.commands, &items)` → want;`for c: set_command_enabled(&c.id, want.contains(&c.name))`。
  - agents:同构(`(a.name, a.tags)`)。
  - skills:`(skill.directory, skill.tags)`;翻 per-app flag `set_enabled_for(&app_type, want.contains(&skill.directory))`。
  - mcp:`(server.id, server.tags)`;**借用注意**:先用**不可变** `servers.values()` 构造 `items` 与 want(借用结束),**再** `servers.values_mut()` 翻 flag(避免借用冲突)。
  - 所有类型的 warnings 收进 `result.warnings`。
- **- [ ] Step 2 (TDD,集成,`#[serial]` 触磁盘按需):**
  - `activate_literal_only_unchanged`:纯字面 spec(无 @)→ 行为与既有一致(commands [a,b] → a,b enabled;c disabled)。
  - `activate_tag_expands`:commands a(tags=[x]) b(tags=[x]) c(tags=[]);spec commands=["@x"] → a,b enabled、c disabled。
  - `activate_mixed_literal_and_tag`:spec=["c","@x"] → a,b(tag)+c(字面)enabled。
  - `activate_unknown_tag_warns_no_error`:spec=["@nope"] → 无 enabled、result.warnings 含 "tag matched no commands: @nope"、Ok。
  - `activate_tag_case_sensitive`:item tag "X",spec "@x" → 不匹配(零展开 + warn)。
  - `activate_tag_for_skills_and_mcp_per_app`:skills 用 directory tags、mcp 用 id tags 展开,仅翻本 app flag。
- **- [ ] Step 3:** `cargo test --lib services::profile`(全绿,含 3a/3b 无回归)+ `cargo test --lib`(无回归)+ `clippy --lib -- -D warnings` + `cargo fmt`。
- **- [ ] Step 4:** commit `feat(profile): @tag selector resolution in activation (literal + @tag union)`。

## T4:update_skill_tags Tauri 命令 + 注册 [Sonnet]

**Files:** Modify `commands/skill.rs`、`lib.rs`(generate_handler)。蓝本:update_command/update_agent(`tags: Vec<String>` 直参)。

- **- [ ] Step 1:** 加 `#[tauri::command] pub fn update_skill_tags(id: String, tags: Vec<String>, state: ...) -> Result<(), String>`:`state.db.update_skill_tags(&id, &tags).map_err(|e| e.to_string())`(纯元数据;**不**调 SkillService 落盘——tags 不影响磁盘 skill 内容)。State 取法对齐现有 skill 命令(skill 命令用 AppState 还是 SkillServiceState?以现有为准)。
- **- [ ] Step 2:** `lib.rs` generate_handler 在 skill 命令区(toggle_skill_app 附近)加 `update_skill_tags`。
- **- [ ] Step 3:** `cargo build` + `cargo test --lib`(无回归)+ `clippy --lib -- -D warnings` + `cargo fmt`。
- **- [ ] Step 4:** commit `feat(skill): update_skill_tags tauri command`。

## T5:前端 skills 标签编辑器 + @tag 可发现性 + i18n [Sonnet]

**Files:** Modify `src/lib/api/skills.ts`(InstalledSkill.tags + updateSkillTags)、skills hook、`SkillCard`(或 skills 面板,标签输入)、`ProfileEditDialog`(placeholder)、`src/i18n/locales/{en,zh,zh-TW,ja}.json`;测试。

- **- [ ] Step 1:** `skills.ts`:`InstalledSkill` 加 `tags?: string[]`;`skillsApi.updateSkillTags(id, tags)` → invoke `update_skill_tags`。
- **- [ ] Step 2:** SkillCard(或 skills 列表项)加一个标签输入(逗号分割,同 commands/mcp 的 tags 输入模式),保存调 `updateSkillTags`;成功后失效 skills query。
- **- [ ] Step 3(@tag 可发现性):** ProfileEditDialog 的 4 个内容列表 placeholder/help i18n 更新,说明条目可填字面名或 `@tagname`(例 `core, @backend`)。**无需改解析逻辑**(逗号文本框已把 `@core` 当字符串)。
- **- [ ] Step 4:** i18n 四语:skills 标签输入相关键(如 `skills.tags`/`skills.tagsPlaceholder`)+ profiles 4 个 placeholder 更新(真翻译)。
- **- [ ] Step 5:** vitest:SkillCard 标签输入保存调 updateSkillTags;tsc 净。
- **- [ ] Step 6:** `tsc --noEmit` + `vitest run`(绿)+ `prettier --write`。
- **- [ ] Step 7:** commit `feat(skill): skill tags editor + @tag discoverability hints + i18n`。

## T6:全绿 + Opus 总评审 + 收官 [Opus]

- **- [ ] Step 1:** 全套:`cargo fmt --check` + `cargo clippy --all-targets -- -D warnings` + `cargo test`(全量,含集成)+ `tsc --noEmit` + `vitest run`。全绿。
- **- [ ] Step 2:** 独立确认:`grep "InstalledSkill {"` 全部含 tags;skill.rs:1090 用 `skill.tags.clone()`(非 vec![]);纯字面 spec 解析与 3a 一致(resolve_selectors 对无 @ 列表返回与字面集相同);v16 幂等(经 runner);skills.tags row 读为 String。
- **- [ ] Step 3:** Opus 总评审 `main..profiles-inc3c`:① @tag 解析正确(并集去重、未匹配不丢、未知/空 @tag 警告非报错、大小写敏感)② 纯字面无回归 ③ skills.tags 全读写点 + update_skill 不清空 ④ v16 幂等 + backfill ⑤ tags 解析容错 ⑥ 不回归 3a/3b ⑦ 无新内容类型/项目通道泄漏。
- **- [ ] Step 4:** finishing-a-development-branch:本地 `--no-ff` 合并 main,合并后复验全绿,删分支。**至此 Profiles 旗舰(增量 3 = 3a+3b-1+3b-2+3b-3+3c)收官。**

---

## 自查 / 一致性

- **spec 覆盖**:增量 3 愿景的「按 @tag 选内容」;收官 Profiles。
- **类型/命名一致**:`InstalledSkill.tags`(Rust↔TS);`update_skill_tags` ↔ `skillsApi.updateSkillTags`;`resolve_selectors` 用于 4 类型;tags 存 JSON-array TEXT(skills/mcp NOT NULL '[]')。
- **占位符**:无;每步实体代码或精确锚点(含 5 处 skill 字面 + 解析函数)。
- **schema**:skills.tags=**v16**(v14=content_hash,v15=prompts.hidden,v16=skills.tags)。
- **对抗修复落点**:update_skill 清空(HIGH)→ T2 step3 `:1090 skill.tags.clone()`;5 字面→T2 step3;row String 读→T2 step2;迁移 backfill→T1 step1;mcp 借用→T3 step1;@-literal 限制→文档化。
- **不做**:见范围。
