# agenthub 增量 3b-2 — `${VAR}` 模板引擎(settings.json + statusline)+ 前端 vars 编辑器 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: superpowers:subagent-driven-development。每步 `- [ ]`。提交前 `cargo fmt` / `prettier --write`。

**Goal:** 给 profile 的 dotfile 内容加 `${VAR}` 模板替换——应用于 **settings.json 片段** 与 **statusline.sh**(**不含 CLAUDE.md**,推迟到 3b-3)。值来源优先级 **profile.spec.vars > 激活 provider 的 settings_config.env > 进程环境(前缀白名单)**。前端加 vars 键值编辑器 + profile manifest 查看。顺带清掉 3b-1 遗留 minor。

**Architecture(经对抗审查修正):**
1. **纯函数 `${VAR}` 引擎**(`services/profile_vars.rs`):`build_var_map`(一次性按优先级合并三源)+ `substitute_vars`(单次左到右扫描 `${NAME}` / `${NAME:-default}`,NAME=`[A-Za-z_][A-Za-z0-9_]*`,**无递归**——输出不再扫描,未知变量保留字面 `${NAME}` + 警告,绝不 panic)。
2. **三个前向渲染点共用同一张 var map**(每次 activate/写一次构建,保证确定性):
   - **settings.json 片段**:在唯一的 `build_effective_settings_with_common_config` 内,对 `df.content` 在 `serde_json::from_str` **之前**做文本级替换,`json_escape=true`(值经 `serde_json` 转义后再拼接,避免引号/反斜杠破坏 JSON;`${NAME:-default}` 的 default 分支**同样转义**)。
   - **statusline.sh**:在 activate 整文件渲染前对 `df.content` 替换,`json_escape=false`(shell 文本)。
3. **backfill 对称剥离须用「重渲染后」的片段(CRITICAL no-bleed)**:3b-1 的 backfill 按字面片段 `json_deep_remove` 剥离;片段渲染后磁盘值已变,字面剥离会漏 → 渲染值(含密钥)回灌进 provider DB。修复:backfill 处用**同一张 var map 重新渲染出向 profile 的片段**,再 `strip_fragment_restoring_provider_owned`。
4. **缺口核实(对抗 HIGH)**:`switch_normal` 仅在 `current_id != id` 时 backfill。**同 provider 切换**(两 profile 指同一 provider)→ 不 backfill → provider 未被改 → 无 bleed(加测试证实)。**proxy 热切换**提前返回 → 不 backfill 也不写 live(proxy 自管)→ 无 bleed(文档化为既有限制)。须在 T3 验证这两条确为"无 bleed"而非"漏剥离"。
5. **密钥不新增落库**:settings.json 仅到磁盘(现状不可避免);statusline 仅存 `content_hash`(哈希非明文);`profile_dotfiles` 只存**未渲染模板**;`apply_manifest` 不存渲染文本。

**Tech Stack:** 同前。仓库 `/Users/fsm/project/MyProject/agentplugin/cc-switch-cloud`(从 main `7c4f2bab` 开分支 `profiles-inc3b2`)。环境 gotcha 同 3b-1(cargo PATH、`.cargo/config.toml`、前端用 `./node_modules/.bin/{vitest,tsc,prettier}`、慢网前缀、触磁盘测试 `CC_SWITCH_TEST_HOME`+`#[serial]`、提交前 fmt、最终门 `clippy --all-targets`、别用 grep -c 收尾)。

**复用锚点(已核实):** `build_effective_settings_with_common_config` + `strip_fragment_restoring_provider_owned` + backfill 调用点(provider/live.rs)、`profile_render.rs`(render_whole_file/validate_rel_path/content_hash)、`ProfileService::activate/deactivate`(services/profile.rs,activate step5 整文件循环 / step8 settings 重建 / current_provider 用 `get_effective_current_provider`)、`get_provider_by_id`、`Profile.spec.vars`(serde_json::Map)、`indexmap`(已依赖)。

**安全(T6 对抗测试守护):** ① 渲染片段切换后**上个 provider DB settings_config 字节不变**(含同 provider 切换);② 渲染密钥不入 manifest/任何新落库点;③ `${VAR}` 无递归/注入,JSON 上下文转义正确,未知变量保留字面+警告;④ 不回归 3b-1(settings 确定性重建、re-sync 重生渲染片段、provider-key restore-on-collision、statusline 拒覆盖)与 3a 不变量。

---

## 范围 / 不做

- **做**:`profile_vars` 引擎;渲染应用于 settings.json 片段 + statusline.sh;backfill 重渲染剥离 + 缺口核实;前端 vars 键值编辑器 + `get_profile_manifest` 包装/hook/展示;清 3b-1 minor(validate_rel_path canonicalize 硬化;删/`#[cfg(test)]` 死代码 `ConfigService::sync_current_providers_to_live`+`sync_claude_live`;proxy 接管不带片段的文档注释)。
- **不做(3b-3)**:**CLAUDE.md 仲裁**(隐藏 prompt 行 / v15 / prompts.hidden / inline 文本)——对抗发现渲染密钥会落 DB,单独增量再设计。**不做(3c)**:@tag/skills.tags。**不做(inc4)**:项目通道。`${VAR}` **不**应用于 CLAUDE.md(3b-2 不渲染 CLAUDE.md)。
- **既有限制(文档化,不修)**:proxy 接管模式 live 写不带 profile 片段(proxy 自管 live);var 在 activate 与 switch 之间变化的 stale 边缘(key-结构剥离 + 警告兜底)。

---

## T1:`profile_vars` 引擎(扫描器 + 优先级 map + 白名单)[Opus]

**Files:** Create `src-tauri/src/services/profile_vars.rs`;Modify `src-tauri/src/services/mod.rs`(`pub mod profile_vars;`)。

- **- [ ] Step 1:** 定义:
```rust
/// 进程环境只读这些前缀（避免泄漏任意宿主环境进渲染产物）。
pub const ENV_ALLOWLIST_PREFIXES: &[&str] = &["ANTHROPIC_", "AGENTHUB_", "CLAUDE_"];

#[derive(Debug, Clone, Default)]
pub struct VarMap(indexmap::IndexMap<String, String>);
impl VarMap { pub fn get(&self, k: &str) -> Option<&str> { self.0.get(k).map(|s| s.as_str()) } }
```
- **- [ ] Step 2:** `pub fn build_var_map(db: &Database, app_type: &AppType, profile: &Profile) -> Result<VarMap, AppError>`:按**从低到高**插入(高覆盖低):
  1. 进程环境:`std::env::vars()` 仅保留 key 以 `ENV_ALLOWLIST_PREFIXES` 任一前缀开头者。
  2. 激活 provider env:`crate::settings::get_effective_current_provider(db, app_type)? → db.get_provider_by_id(&id, app_type.as_str())?`;读 `provider.settings_config.get("env").and_then(|v| v.as_object())`,对每个 `(k, Value)` 用 `coerce_value`(String 原样;number/bool `to_string`;其他 → 跳过 + 可忽略)。
  3. `profile.spec.vars`:同 `coerce_value`。
  返回 `VarMap`。**注意 provider 用入参隐含的"当前有效 provider",profile 用入参 `&profile`**(调用方传正确的 active profile)。
- **- [ ] Step 3 (TDD):** `pub fn substitute_vars(template: &str, vars: &VarMap, json_escape: bool, warnings: &mut Vec<String>) -> String`:
  - 单次左到右扫描;遇 `${` 起始,解析 `NAME`(`[A-Za-z_][A-Za-z0-9_]*`),可选 `:-default`(到匹配的 `}`;default 文本里**不**支持嵌套 `${}`,遇内层 `${` 视为字面)。
  - 解析值:`vars.get(NAME)`;无则若有 `:-default` 用 default(**空串也算"未设"→ 用 default**,shell `:-` 语义),否则保留字面 `${...}` + `warnings.push(format!("unknown var: {NAME}"))`。
  - **无递归**:解析出的值/ default **不再扫描**。
  - `json_escape=true` 时:把要拼接的字符串(值或 default)用 `serde_json::to_string(&Value::String(s))` 取**去掉首尾引号**的转义形式再拼接(default 分支**同样转义**)。`json_escape=false` 原样拼接。
  - 非 `${NAME...}` 的 `$`、未闭合 `${` → 原样输出。
  TDD 单测:`${A}` 替换、`${X:-def}` 用 default、空串用 default、未知保留+警告、`a"b` 值在 json_escape 下转义、值含 `${Y}` 不二次替换、`profile>provider>process` 优先级、白名单过滤进程环境。
- **- [ ] Step 4:** `pub fn render_with_profile_vars(db, app_type, profile, template, json_escape, &mut warnings) -> Result<String, AppError>` 便捷封装(build_var_map + substitute_vars)。`coerce_value` 私有。
- **- [ ] Step 5:** `cargo test --lib services::profile_vars`(绿)+ `clippy --lib`(新 pub 由 T2/T3 消费;若 dead_code 告警可接受,勿加 allow)+ `cargo fmt`。
- **- [ ] Step 6:** commit `feat(profile): ${VAR} substitution engine (precedence + allowlist + json-escape)`。

## T2:在三个前向点应用渲染(settings.json + statusline)[Opus]

**Files:** Modify `src-tauri/src/services/provider/live.rs`(settings 片段渲染)、`src-tauri/src/services/profile.rs`(activate step5 statusline 渲染)。

- **- [ ] Step 1:** 先完整读 `build_effective_settings_with_common_config`(片段 `from_str` 点)与 activate step5 整文件循环。
- **- [ ] Step 2 (settings.json 前向):** 在 `build_effective_settings_with_common_config` 取到 active profile 的 `settings.json` 片段 `df.content` 后、`serde_json::from_str` **之前**:
```rust
let mut warns = Vec::new();
let rendered = crate::services::profile_vars::render_with_profile_vars(
    db, app_type, &profile, &df.content, /*json_escape=*/ true, &mut warns)?;
match serde_json::from_str::<serde_json::Value>(&rendered) {
    Ok(frag) => json_deep_merge(&mut effective_settings, &frag),
    Err(e) => log::warn!("profile {} settings.json fragment invalid JSON after render, skipping: {e}", profile.id),
}
```
（此函数被 switch/re-sync/failover 共用,profile 取 `db.get_active_profile`——activate step8 时 active 已是 INCOMING,re-sync 时是当前 active,均正确。warns 此处仅 log,不外冒。）
- **- [ ] Step 3 (statusline 前向):** activate step5 循环里,对每个非 `settings.json`/`CLAUDE.md` 的 dotfile,在调 `render_whole_file` 前渲染:
```rust
let rendered = crate::services::profile_vars::render_with_profile_vars(
    state.db.as_ref(), &app_type, &profile, &df.content, /*json_escape=*/ false, &mut result.warnings)?;
// 然后 render_whole_file(..., &rendered, prior_owned_hash)
```
（`profile` 是 INCOMING(step1 已加载)。content_hash 现在是渲染后字节的哈希——所有权检测仍成立:vars 不变时重激活渲染同字节。）
- **- [ ] Step 4 (TDD,触磁盘 `#[serial]`):**
  - `activate_renders_settings_fragment_var`:profile vars `{"GREETING":"hi"}` + 片段 `{"statusLine":{"msg":"${GREETING}"}}` → settings.json 含 `statusLine.msg=="hi"`。
  - `activate_renders_statusline_var`:vars `{"NAME":"alba"}` + statusline `echo ${NAME}` → 磁盘 statusline.sh == `echo alba`。
  - `provider_env_resolves_in_fragment`:provider env.ANTHROPIC_MODEL=foo,片段 `{"model":"${ANTHROPIC_MODEL}"}` → settings.json `model=="foo"`(profile vars 缺该键时取 provider env)。
  - `profile_var_overrides_provider_env`:provider env.X=prov,profile vars X=prof,片段引用 `${X}` → "prof"。
  - `unknown_var_kept_literal_with_warning`:片段引用 `${MISSING}` → 字面保留 + ActivateResult.warnings 含提示(注:settings 前向的 warns 仅 log;此断言针对 statusline 前向,其 warns 进 result)。
- **- [ ] Step 5:** `cargo test --lib services::`(绿,无回归)+ `clippy --lib -- -D warnings` + `cargo fmt`。
- **- [ ] Step 6:** commit `feat(profile): render ${VAR} in settings.json fragment + statusline on activate`。

## T3:backfill 重渲染剥离(no-bleed)+ 缺口核实 [Opus] —— 皇冠

**Files:** Modify `src-tauri/src/services/provider/live.rs`(backfill 剥离点);可能 `provider/mod.rs`(核实 switch 路径)。

- **- [ ] Step 1:** 完整读 backfill 剥离点(`strip_common_config_from_live_settings` 内加载 active profile + `df.content` + `from_str` + `strip_fragment_restoring_provider_owned` 的那段)与 `switch_normal`(`current_id != id` 守卫、proxy 热切换提前返回)。
- **- [ ] Step 2(重渲染剥离):** 在 backfill 剥离点,对出向 active profile 的 `df.content` **先重渲染再剥离**(用与前向同一引擎):
```rust
let mut ignored = Vec::new();
let rendered = crate::services::profile_vars::render_with_profile_vars(
    db, app_type, &profile, &df.content, /*json_escape=*/ true, &mut ignored)?;
if let Ok(frag) = serde_json::from_str::<serde_json::Value>(&rendered) {
    strip_fragment_restoring_provider_owned(&mut backfill_settings, &frag, &provider.settings_config);
}
```
（active profile 在 backfill 时仍是 OUTGOING——正确,正是磁盘上那份片段的来源。）
- **- [ ] Step 3 (TDD,对抗,`#[serial]`):**
  - `rendered_fragment_not_polluting_provider_on_backfill`(**CRITICAL**):provider P 当前 + active profile 片段 `{"env":{"X":"${ANTHROPIC_AUTH_TOKEN}"}}`、provider/vars 提供 `ANTHROPIC_AUTH_TOKEN=sk-x`;落盘(live 含 `env.X="sk-x"`);切到 provider Q;断言 **P 的 settings_config 与原始字节一致**(无 `env.X`、无 `sk-x` 残留)。
  - `same_provider_switch_no_bleed`:profile A、B **指向同一 provider P**,各带不同片段;activate A 后 activate B;断言 P 的 settings_config 不变 + live settings.json 是 B 的片段(证实同 provider 切换不 backfill、step8 前向重建生效)。若发现同 provider 路径确有 bleed(backfill 实际运行了),则补剥离;否则文档化"同 provider 不 backfill 故无 bleed"。
  - 保留 3b-1 的 `provider_settings_config_not_polluted_by_fragment_on_backfill`(字面片段)与 `resync_reproduces_fragment` 仍绿。
- **- [ ] Step 4(proxy 文档):** 在 backfill/switch 处加注释:proxy 热切换提前返回,不 backfill 也不写 live(proxy 自管 live),故 profile 片段在该模式不应用、也不污染 provider——既有限制。
- **- [ ] Step 5:** `cargo test --lib services::provider`(绿)+ `cargo test --lib`(无回归)+ `clippy --lib -- -D warnings` + `cargo fmt`。
- **- [ ] Step 6:** commit `fix(provider): re-render outgoing profile fragment before backfill strip (rendered no-bleed)`。

## T4:清 3b-1 遗留 minor [Sonnet]

**Files:** Modify `src-tauri/src/services/profile_render.rs`(validate_rel_path)、`src-tauri/src/services/config.rs`(死代码)。

- **- [ ] Step 1(路径硬化):** `validate_rel_path` 在词法校验(已拒 `..`/绝对/前缀 + `starts_with` base)后,**额外**:对已存在的父目录做 `canonicalize` 并复核 `starts_with(base.canonicalize())`(若父目录不存在则跳过 canonical,词法校验已足够);加测试 `validate_rel_path_rejects_symlinked_escape`(在 base 下建一个指向外部的软链子目录,`sub/x.sh` 应被拒)。
- **- [ ] Step 2(死代码):** 确认 `ConfigService::sync_current_providers_to_live` 与 `sync_claude_live` 无生产调用者(grep);若确死,删除或 `#[cfg(test)]`-gate(取不破坏现有测试者)。`clippy --lib -- -D warnings` 须净。
- **- [ ] Step 3(文档):** 若 T3 未加,在 proxy 接管 live 写处加注释:不携带 profile dotfile 片段(proxy 自管)。
- **- [ ] Step 4:** `cargo test --lib`(无回归)+ `clippy --lib -- -D warnings` + `cargo fmt`。
- **- [ ] Step 5:** commit `chore(profile): harden validate_rel_path (canonicalize) + drop dead ConfigService live writers`。

## T5:前端 — vars 键值编辑器 + manifest 查看 [Sonnet]

**Files:** Modify `src/lib/api/profiles.ts`、`src/hooks/useProfiles.ts`、`src/components/profiles/ProfileEditDialog.tsx`、`src/i18n/locales/{en,zh,zh-TW,ja}.json`;测试。

- **- [ ] Step 1:** `profiles.ts` 加 `getManifest(id, app)` → `invoke("get_profile_manifest",{id, app})` 返回 `ManifestEntry[]`(定义 TS 类型 `ManifestEntry{ id; channel; profileId?; appType; targetPath; kind; contentHash?; createdAt }`)。
- **- [ ] Step 2:** `useProfiles.ts` 加 `useProfileManifest(id, app)` query(key `["profiles","manifest",id]`)。
- **- [ ] Step 3:** `ProfileEditDialog.tsx` 加 **Variables 区块**:键值对列表编辑(增/删行,key+value 输入)绑定 `spec.vars`(保存时并入 `update_profile` 的 spec);并在 dotfiles 区下方加只读 **"Applied files"** 小节,用 `useProfileManifest` 展示该 profile 写过的整文件 target_path(编辑态)。**不加 CLAUDE.md 文本框**(3b-3)。
- **- [ ] Step 4:** i18n 四语加键:`profiles.variables`、`profiles.variablesHint`、`profiles.varKey`、`profiles.varValue`、`profiles.addVar`、`profiles.appliedFiles`(真翻译)。
- **- [ ] Step 5:** vitest:vars 编辑器增删行 + 保存把 vars 写进 spec;manifest 小节渲染。
- **- [ ] Step 6:** `./node_modules/.bin/tsc --noEmit`(净)+ `vitest run`(绿)+ `prettier --write`。
- **- [ ] Step 7:** commit `feat(profile): vars key/value editor + applied-files (manifest) view in ProfileEditDialog`。

## T6:全绿 + Opus 总评审 + 收尾 [Opus]

- **- [ ] Step 1:** 全套:`cargo fmt --check` + `cargo clippy --all-targets -- -D warnings` + `cargo test`(全量) + `tsc --noEmit` + `vitest run`。全绿。
- **- [ ] Step 2:** 独立确认:渲染密钥不入 manifest(grep manifest 写入处无渲染文本,只 content_hash);profile_dotfiles 仍存未渲染模板;CLAUDE.md 未被渲染(activate step5 仍 `continue` 跳过 CLAUDE.md;无 prompts.hidden / v15)。
- **- [ ] Step 3:** Opus 总评审 `main..profiles-inc3b2`:① 渲染片段 no-bleed(含同 provider 切换)② `${VAR}` 无递归/注入 + JSON 转义正确 + 优先级 + 白名单 ③ 密钥不新增落库 ④ 不回归 3b-1/3a ⑤ 无 CLAUDE.md/@tag scope 泄漏。
- **- [ ] Step 4:** finishing-a-development-branch:本地 `--no-ff` 合并 main,合并后复验全绿,删分支。

---

## 自查 / 一致性

- **spec 覆盖**:增量 3 愿景的 `${VAR}` 模板(settings.json + statusline);CLAUDE.md 仲裁明确移到 3b-3。
- **类型/命名一致**:`profile_vars::{build_var_map,substitute_vars,render_with_profile_vars}`;`get_profile_manifest` ↔ `profilesApi.getManifest` ↔ `useProfileManifest`;`ManifestEntry`(Rust↔TS camelCase)。
- **占位符**:无;每步实体代码或精确锚点。
- **schema**:**无变更**(无 v15;CLAUDE.md/prompts.hidden 移 3b-3)。
- **对抗修复落点**:渲染片段 bleed → T3 重渲染剥离 + 同 provider/proxy 核实;JSON 转义 + no-recursion + `:-` 语义 → T1;map 构建点一致(get_active_profile)→ T2;路径 canonicalize + 死代码 → T4。CLAUDE.md 两个 CRITICAL → 整体移 3b-3。
- **不做**:见范围。
