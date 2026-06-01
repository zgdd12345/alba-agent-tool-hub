# agenthub Fork 基础（增量 1）Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把 cc-switch v3.16.0 fork 成 `AgentHub`：改名、迁移配置目录到 `~/.agenthub`、禁用自动更新、剥离所有赞助/联盟内容；产出一个能 build、能跑、现有测试全绿的干净品牌基线，作为后续四大支柱（commands/profiles/projects/source）增量的地基。

**Architecture:** 叠加式（方案 A）—— 本增量**不加新功能**，只做品牌/配置/卫生改造，最小化对 cc-switch 内核的改动，复用其全部既有能力（provider 切换、代理、MCP/skills/prompts、WebDAV 同步）与测试套件。SQLite 全程，不依赖 git。

**Tech Stack:** Tauri 2.8（Rust 1.95，MSRV 1.85）+ React 18 + TypeScript 5.3 + Vite + TailwindCSS + shadcn/Radix；pnpm 10.12.3 / Node 20；SQLite（rusqlite）。

**命名/选择（全计划一致）：** 显示名 `AgentHub` · bundle id `dev.agenthub.app` · crate `agenthub` / lib `agenthub_lib` · deeplink scheme `agenthub` · 配置目录 `~/.agenthub` · DB `agenthub.db` · updater **禁用（方案 B）** · 配置目录干净起步（不迁移旧数据） · fork 本地目录 `/Users/fsm/project/MyProject/agentplugin/agenthub-desktop`。

**参考来源：** 设计 spec `docs/superpowers/specs/2026-06-01-agenthub-gui-ccswitch-fork-design.md`；触点清单由 2026-06-01 分层 workflow 实测自 cc-switch 源码。

---

## 前置约定（每个任务通用）

- 所有命令在 fork 目录 `agenthub-desktop/` 下运行。
- 工具链：Node 20、pnpm 10.12.3、Rust 1.95（含 rustfmt+clippy）。
- 安全网：cc-switch 自带 ~273 前端测试 + Rust `src-tauri/tests/`（11 文件）。**本增量的验收标准 = 改造后这些测试仍全绿**，外加每个任务的 grep 完备性断言。
- 验证命令速查：
  - 前端类型检查：`pnpm typecheck`
  - 前端单测：`pnpm test:unit`
  - Rust 测试：`cargo test --manifest-path src-tauri/Cargo.toml`
  - Rust 格式/lint：`cargo fmt --check --manifest-path src-tauri/Cargo.toml` / `cargo clippy --manifest-path src-tauri/Cargo.toml -- -D warnings`
  - 跑应用：`pnpm tauri dev`

---

## 文件结构（本增量触及的文件）

- **品牌/清单**：`src-tauri/tauri.conf.json`、`src-tauri/tauri.windows.conf.json`、`src-tauri/Cargo.toml`、`src-tauri/src/main.rs`、`package.json`、`src-tauri/Info.plist`
- **运行时字符串**：`src-tauri/src/lib.rs`、`src-tauri/src/tray.rs`、`src-tauri/src/auto_launch.rs`、`src-tauri/src/panic_hook.rs`
- **配置目录/存储**：`src-tauri/src/config.rs`、`src-tauri/src/database/mod.rs`、`src-tauri/src/database/backup.rs`、`src-tauri/src/settings.rs`、`src-tauri/src/services/env_manager.rs`
- **updater**：`src-tauri/tauri.conf.json`、`src-tauri/src/lib.rs`、`src-tauri/capabilities/default.json`、`src/lib/updater.ts`、`src/contexts/UpdateContext.tsx`、`src/components/UpdateBadge.tsx`、`src/components/settings/AboutSection.tsx`、`package.json`
- **赞助剥离**：`README.md`、`README_ZH.md`、`README_JA.md`、`README_DE.md`、`.github/FUNDING.yml`、`src/config/*ProviderPresets.ts`（7 个）、`src/i18n/locales/*`、`src/components/providers/ProviderPresetSelector.tsx`、`src/components/providers/ApiKeySection.tsx`、`src/components/providers/ProviderCard.tsx`、`src-tauri/src/services/gemini_auth.rs`、provider 模型（Rust）

---

## Task 0：建立 fork 与绿色基线

**Files:**
- Create: 新仓 `/Users/fsm/project/MyProject/agentplugin/agenthub-desktop/`（cc-switch v3.16.0 源码全量）

- [ ] **Step 1: 取得 cc-switch 源码作为 fork**

在 GitHub 上 fork `farion1231/cc-switch` 到你的账号（保留历史，便于以后选择性 pull 上游），然后：
```bash
git clone git@github.com:<your-org>/cc-switch.git /Users/fsm/project/MyProject/agentplugin/agenthub-desktop
cd /Users/fsm/project/MyProject/agentplugin/agenthub-desktop
git remote add upstream https://github.com/farion1231/cc-switch.git
git checkout -b fork-foundation
```

- [ ] **Step 2: 装依赖**

```bash
cd /Users/fsm/project/MyProject/agentplugin/agenthub-desktop
pnpm install --frozen-lockfile
```
Expected: 安装成功，无报错。

- [ ] **Step 3: 验证基线绿色（改造前）**

```bash
pnpm typecheck
pnpm test:unit
cargo test --manifest-path src-tauri/Cargo.toml
```
Expected: 三者全部通过（这是后续每步的对照基线）。若 Rust 测试因本机缺系统依赖失败，记录下来并以前端测试 + `cargo build` 为准。

- [ ] **Step 4: 提交基线**

```bash
git add -A && git commit -m "chore: import cc-switch v3.16.0 as AgentHub fork baseline"
```

---

## Task 1：品牌 — 清单与标识（需要具体新值的项）

**Files:**
- Modify: `src-tauri/tauri.conf.json`、`src-tauri/tauri.windows.conf.json`、`src-tauri/Cargo.toml`、`src-tauri/src/main.rs`、`package.json`、`src-tauri/Info.plist`

- [ ] **Step 1: 改 `src-tauri/tauri.conf.json`**

- `productName`: `"CC Switch"` → `"AgentHub"`
- `identifier`: `"com.ccswitch.desktop"` → `"dev.agenthub.app"`
- `plugins.deep-link.desktop.schemes` 数组元素: `"ccswitch"` → `"agenthub"`

（`plugins.updater` 与 `bundle.createUpdaterArtifacts` 在 Task 4 处理。）

- [ ] **Step 2: 改 `src-tauri/tauri.windows.conf.json`**

`app.windows[0].title`: `"CC Switch"` → `"AgentHub"`

- [ ] **Step 3: 改 `src-tauri/Cargo.toml`**

- `[package] name`: `"cc-switch"` → `"agenthub"`
- `[package] repository`: → `"https://github.com/<your-org>/agenthub-desktop"`
- `[lib] name`: `"cc_switch_lib"` → `"agenthub_lib"`

- [ ] **Step 4: 改 `src-tauri/src/main.rs`**

```rust
// line 21
agenthub_lib::run();   // was: cc_switch_lib::run();
```

- [ ] **Step 5: 改 `package.json`**

`"name": "cc-switch"` → `"name": "agenthub"`

- [ ] **Step 6: 改 `src-tauri/Info.plist`**

- `CFBundleURLName` string: `"CC Switch Deep Link"` → `"AgentHub Deep Link"`
- `CFBundleURLSchemes` 数组元素: `"ccswitch"` → `"agenthub"`

- [ ] **Step 7: 验证编译与类型**

```bash
cargo build --manifest-path src-tauri/Cargo.toml
pnpm typecheck
```
Expected: 编译通过（lib 改名后 main.rs 调用一致）；typecheck 通过。

- [ ] **Step 8: 提交**

```bash
git add src-tauri/tauri.conf.json src-tauri/tauri.windows.conf.json src-tauri/Cargo.toml src-tauri/src/main.rs package.json src-tauri/Info.plist
git commit -m "refactor(brand): rename product/identifier/scheme/crate to AgentHub"
```

---

## Task 2：品牌 — 运行时字符串与对应测试

**Files:**
- Modify: `src-tauri/src/lib.rs`、`src-tauri/src/tray.rs`、`src-tauri/src/auto_launch.rs`、`src-tauri/src/panic_hook.rs`

- [ ] **Step 1: 改 `src-tauri/src/lib.rs`**

- line 114: `if !url_str.starts_with("ccswitch://")` → `"agenthub://"`
- line 1465: `if url_str.starts_with("ccswitch://")` → `"agenthub://"`
- line 822: `.tooltip("CC Switch")` → `.tooltip("AgentHub")`
- line 767: `"applications/cc-switch-handler.desktop"` → `"applications/agenthub-handler.desktop"`
- lines 303/315: `"cc-switch.log"` → `"agenthub.log"`
- line 325: `Some("cc-switch".into())` → `Some("agenthub".into())`

- [ ] **Step 2: 改 `src-tauri/src/tray.rs`**

- line 81: `pub const TRAY_ID: &str = "cc-switch";` → `"agenthub"`
- line 714: `"https://ccswitch.io"` → `"https://github.com/<your-org>/agenthub-desktop"`（暂无品牌域名，先指向仓库）
- line 883: 测试 `assert_eq!(TRAY_ID, "cc-switch")` → `assert_eq!(TRAY_ID, "agenthub")`

- [ ] **Step 3: 改 `src-tauri/src/auto_launch.rs`**

- line 20: `let app_name = "CC Switch";` → `"AgentHub"`
- 测试 fixtures（lines ~79/83/91/96）里 `"/Applications/CC Switch.app/Contents/MacOS/CC Switch"` 等路径 → 用 `"AgentHub"` 同步替换（保持测试与新名一致）

- [ ] **Step 4: 改 `src-tauri/src/panic_hook.rs`**

- line 25: `.join(".cc-switch")` → `.join(".agenthub")`（注：此为兜底；运行时由 lib.rs:290 注入正确值，但需与 Task 3 一致）
- line 195: 测试断言 path contains `".cc-switch"` → `".agenthub"`

- [ ] **Step 5: 跑相关 Rust 测试**

```bash
cargo test --manifest-path src-tauri/Cargo.toml tray auto_launch panic_hook
```
Expected: TRAY_ID、auto_launch 路径、panic_hook 目录断言均以新值通过。

- [ ] **Step 6: 提交**

```bash
git add src-tauri/src/lib.rs src-tauri/src/tray.rs src-tauri/src/auto_launch.rs src-tauri/src/panic_hook.rs
git commit -m "refactor(brand): rebrand runtime strings (scheme/tray/log/autolaunch) + tests"
```

---

## Task 3：配置目录迁移 `~/.cc-switch` → `~/.agenthub`（含 3 个绕过点）

> 关键：`get_app_config_dir()` 是 SSOT，但有 **3 处硬编码绕过它**，必须一并改，否则 settings/env-backup/panic 会留在旧目录。

**Files:**
- Modify: `src-tauri/src/config.rs`、`src-tauri/src/database/mod.rs`、`src-tauri/src/database/backup.rs`、`src-tauri/src/settings.rs`、`src-tauri/src/services/env_manager.rs`、`src-tauri/src/lib.rs`

- [ ] **Step 1: 改主解析器 `src-tauri/src/config.rs`**

- line 95: `let default_dir = get_home_dir().join(".cc-switch");` → `.join(".agenthub")`
- line 89 文档注释 `(~/.cc-switch)` → `(~/.agenthub)`；line 16 注释里 `.cc-switch/cc-switch.db` → `.agenthub/agenthub.db`
- lines 101-120 的 `#[cfg(windows)]` v3.10.3 legacy 块：**整块删除**（干净起步，不保留旧 Windows HOME 迁移 hack；其内含 `.cc-switch` 与 `cc-switch.db` 两处字面量）。删除后该分支直接 `return default_dir;`
- **保留** `CC_SWITCH_TEST_HOME`（line 23）不改名（仅测试可见，改名牵涉 ~30 处，v1 不值得）。

- [ ] **Step 2: 改 DB 文件名（两处必须同步）**

- `src-tauri/src/database/mod.rs` line 96: `.join("cc-switch.db")` → `.join("agenthub.db")`（line 94 注释同步）
- `src-tauri/src/database/backup.rs` line 299: `.join("cc-switch.db")` → `.join("agenthub.db")`（必须与 mod.rs 一致，否则备份静默 no-op）

- [ ] **Step 3: 改 3 个绕过中央解析器的硬编码点**

- `src-tauri/src/settings.rs` line 409: `get_home_dir().join(".cc-switch").join("settings.json")` → `.join(".agenthub").join("settings.json")`（line 217 注释同步）
- `src-tauri/src/services/env_manager.rs` line 71: `home.join(".cc-switch").join("backups")` → `.join(".agenthub").join("backups")`
- （`panic_hook.rs:25` 已在 Task 2 改过。）

- [ ] **Step 4: 改迁移入口与用户可见文案 `src-tauri/src/lib.rs`**

- lines 346-348：`let db_path = app_config_dir.join("cc-switch.db");` → `.join("agenthub.db")`（`json_path` 仍为 `config.json` 不变）
- lines 1789（中）/1803（英）错误对话框里 `cc-switch.db` → `agenthub.db`；line 205 注释 `~/.cc-switch/crash.log` → `~/.agenthub/crash.log`
- **决定：干净起步** —— 不从 `~/.cc-switch` 拷数据（这是 fork，可接受全新开始）。如日后想迁移，在此块前加一次性 `~/.cc-switch → ~/.agenthub` 拷贝。

- [ ] **Step 5: SQL 导出头 —— 不要动魔术常量（向后兼容）**

`src-tauri/src/database/backup.rs` 的 `CC_SWITCH_SQL_EXPORT_HEADER`（line 16，值 `"-- CC Switch SQLite 导出"`）**保持原样**：已导出的 .sql 备份以此开头，`validate_cc_switch_sql_export`（166-177）据此校验，改了会导致旧备份导入失败。仅把 line 175 面向用户的英文报错 `"...exported by CC Switch are supported."` 改为 `"...exported by AgentHub..."`（不影响校验）。

- [ ] **Step 6: 完备性断言 + 跑应用**

```bash
# 除「SQL 导出头常量」与「CC_SWITCH_TEST_HOME」外，src 里不应再有残留
grep -rn "\.cc-switch\|cc-switch\.db" src-tauri/src/ | grep -v "CC_SWITCH_TEST_HOME"
```
Expected: 仅剩 `database/backup.rs` 里 `CC_SWITCH_SQL_EXPORT_HEADER` 那一处（有意保留）；其余为 0 命中。
```bash
cargo test --manifest-path src-tauri/Cargo.toml
pnpm tauri dev   # 启动后确认在 ~/.agenthub/ 下创建 agenthub.db 与 settings.json
ls -la ~/.agenthub/
```
Expected: Rust 测试通过；`~/.agenthub/agenthub.db` 被创建；不在 `~/.cc-switch` 写入。

- [ ] **Step 7: 提交**

```bash
git add src-tauri/src/config.rs src-tauri/src/database/mod.rs src-tauri/src/database/backup.rs src-tauri/src/settings.rs src-tauri/src/services/env_manager.rs src-tauri/src/lib.rs
git commit -m "refactor(config): relocate config dir to ~/.agenthub + agenthub.db (incl. 3 bypass sites)"
```

---

## Task 4：禁用自动更新（方案 B）

> 个人工具、暂无发布签名设施 → 禁用 updater。日后要自动更新再走方案 A（自建 minisign 密钥 + 自有 endpoint）。

**Files:**
- Modify: `src-tauri/tauri.conf.json`、`src-tauri/src/lib.rs`、`src-tauri/capabilities/default.json`、`src-tauri/src/commands/misc.rs`、`package.json`
- Delete: `src/lib/updater.ts`
- Modify: `src/contexts/UpdateContext.tsx`、`src/components/UpdateBadge.tsx`、`src/components/settings/AboutSection.tsx`

- [ ] **Step 1: 删 updater 配置 `src-tauri/tauri.conf.json`**

- 删除整个 `plugins.updater` 对象（lines 62-67，含 `pubkey` 与 `endpoints`）。
- `bundle.createUpdaterArtifacts`（line 39）: `true` → `false`（禁用时跳过签名工件，免去构建时 `TAURI_SIGNING_PRIVATE_KEY`）。

- [ ] **Step 2: 删插件注册 `src-tauri/src/lib.rs`**

删除 lines 295-301 的 `#[cfg(desktop)]` updater 插件注册块（`tauri_plugin_updater::Builder::new().build()`）。

- [ ] **Step 3: 删能力 `src-tauri/capabilities/default.json`**

删除 line 11 `"updater:default",`。`"process:allow-restart"`（line 19）仅被 updater 的 relaunch 用到，本计划删 updater 后一并删除该行（若 clippy/编译报有其它用处则保留）。

- [ ] **Step 4: 删前端 updater 代码**

- 删除文件 `src/lib/updater.ts`。
- `src/contexts/UpdateContext.tsx`、`src/components/UpdateBadge.tsx`、`src/components/settings/AboutSection.tsx`：移除对 `checkForUpdate`/`getCurrentVersion`/`relaunchApp` 的 import 与调用，以及「检查更新」UI（AboutSection 里仅保留版本号展示，去掉更新按钮/徽章）。
- `package.json`：从 deps 移除 `@tauri-apps/plugin-updater`（以及 `@tauri-apps/plugin-process`，若无其它使用）。
- `src-tauri/Cargo.toml`：移除 `tauri-plugin-updater` 依赖（若 `tauri-plugin-process` 无其它使用一并移除）。

- [ ] **Step 5: 重指「打开发布页」命令 `src-tauri/src/commands/misc.rs`**

line 59 `"https://github.com/farion1231/cc-switch/releases/latest"` → `"https://github.com/<your-org>/agenthub-desktop/releases/latest"`（这是浏览器打开发布页，独立于自动更新）。

- [ ] **Step 6: 验证无悬空引用 + 构建**

```bash
pnpm install                 # 同步移除的依赖
pnpm typecheck               # 确认无 updater/process 悬空 import
cargo build --manifest-path src-tauri/Cargo.toml
pnpm test:unit
```
Expected: 全部通过；启动 `pnpm tauri dev` 不再尝试检查更新。

- [ ] **Step 7: 提交**

```bash
git add -A
git commit -m "chore(updater): disable auto-updater (option B); repoint releases link"
```

---

## Task 5：剥离赞助 / 联盟内容

> 原则：**保留所有 provider 预设条目**，只去掉推广链接/促销 key/isPartner 标志与赞助 UI。

**Files:**
- Modify: `README.md`、`README_ZH.md`、`README_JA.md`、`README_DE.md`
- Delete/Modify: `.github/FUNDING.yml`
- Modify: `src/config/claudeProviderPresets.ts` 及其余 6 个 `src/config/*ProviderPresets.ts`
- Modify: `src/i18n/locales/{en,zh,zh-TW,ja}/*`（含 `providerForm.partnerPromotion`）
- Modify: `src/components/providers/ProviderPresetSelector.tsx`、`src/components/providers/ApiKeySection.tsx`、`src/components/providers/ProviderCard.tsx`
- Modify: `src-tauri/src/services/gemini_auth.rs` 及 provider 模型（Rust）

- [ ] **Step 1: 删 README 赞助段**

四个 README（`README.md`/`README_ZH.md`/`README_JA.md`/`README_DE.md`）各删除赞助段（约 lines 20-151，从 `## ❤️Sponsor`/`## ❤️赞助商`/`## ❤️スポンサー` 到该段 `</details>` 结束，止于下一个 `## Why...`/`## 为什么...`）。

- [ ] **Step 2: 删 `.github/FUNDING.yml`**

```bash
git rm .github/FUNDING.yml
```
（或清空其 `custom:` 列表。）

- [ ] **Step 3: 清 7 个 provider 预设文件的联盟字段**

对每个 `src/config/*ProviderPresets.ts`（claude/claudeDesktop/codex/gemini/hermes/openclaw/opencode/universal —— 实际以 `ls src/config/*ProviderPresets.ts` 为准）：
- 删除 `ProviderPreset` 接口里的 `partnerPromotionKey?: string;` 与 `isPartner?: boolean;` 字段声明。
- 删除每个条目里的 `isPartner: true,` 与 `partnerPromotionKey: "...",` 行。
- 把带推广参数的 `websiteUrl`/`apiKeyUrl` 去掉查询串（例：`"https://www.shengsuanyun.com/?from=CH_4HHXMRYF"` → `"https://www.shengsuanyun.com"`）。**保留 provider 条目本身**。

- [ ] **Step 4: 清 i18n 推广文案**

`src/i18n/locales/{en,zh,zh-TW,ja}` 各删除 `providerForm.partnerPromotion` 对象（约 27 条推广字符串/语言）。

- [ ] **Step 5: 中和应用内赞助 UI**

- `src/components/providers/ProviderPresetSelector.tsx`：移除合作伙伴星标徽章渲染。
- `src/components/providers/ApiKeySection.tsx`：移除促销文案框。
- `src/components/providers/ProviderCard.tsx`：移除 partner 星标 emoji。

- [ ] **Step 6: 修复 `gemini_auth.rs` 的功能性用法**

`src-tauri/src/services/gemini_auth.rs` 用 `partnerPromotionKey` 区分 PackyCode 鉴权流程。改为按 provider 的稳定标识（id 或 name，如包含 "packycode"）判断，**或**移除该特例分支（确认 PackyCode 走通用流程仍可用）。同步移除 provider 模型（Rust）里为序列化而 re-expose 的 partner 字段，保持与 TS 端一致。

- [ ] **Step 7: 完备性断言 + 测试**

```bash
grep -rni "partnerPromotionKey\|isPartner\|partnerPromotion" src/ src-tauri/src/   # 期望 0 命中
grep -rni "minimax\|packycode\|aigocode\|aicodemirror\|from=CH_\|invitecode\|?aff=\|&aff=" src/ README*.md  # 期望仅剩无害的非推广文本（核对）
pnpm typecheck
pnpm test:unit
cargo test --manifest-path src-tauri/Cargo.toml
```
Expected: partner 相关标识 0 命中；typecheck/前端测试/Rust 测试通过。

- [ ] **Step 8: 提交**

```bash
git add -A
git commit -m "chore(sponsors): strip all sponsor/affiliate content; keep provider presets"
```

---

## Task 6：最终全绿验证

**Files:** 无（纯验证；如有遗漏 lint/format 再小步修）

- [ ] **Step 1: 全套检查**

```bash
pnpm install --frozen-lockfile
pnpm typecheck
pnpm test:unit
cargo fmt --check --manifest-path src-tauri/Cargo.toml
cargo clippy --manifest-path src-tauri/Cargo.toml -- -D warnings
cargo test --manifest-path src-tauri/Cargo.toml
```
Expected: 全部通过（clippy 零警告）。如 fmt/clippy 报问题，按提示小步修复后 `git commit -m "style: fmt/clippy"`。

- [ ] **Step 2: 残留品牌扫描**

```bash
grep -rni "cc-switch\|cc switch\|ccswitch" src/ src-tauri/src/ \
  | grep -v "CC_SWITCH_TEST_HOME" \
  | grep -v "CC_SWITCH_SQL_EXPORT_HEADER" \
  | grep -v "导出"
```
Expected: 仅剩有意保留项（测试 env 变量、SQL 导出头常量及其注释）；其余 0 命中。

- [ ] **Step 3: 端到端冒烟**

```bash
pnpm tauri dev
```
确认：窗口标题/托盘显示 `AgentHub`；在 `~/.agenthub/` 创建 `agenthub.db`+`settings.json`；provider 切换仍可用（写入各工具原生配置）；无更新检查；无赞助 UI。

- [ ] **Step 4: 合并基线分支**

```bash
git checkout main && git merge --no-ff fork-foundation -m "feat: AgentHub fork foundation (rebrand + ~/.agenthub + no updater + no sponsors)"
```

---

## 自查（spec 覆盖 / 占位符 / 类型一致）

- **spec 覆盖**：本计划对应 spec §8（Fork 卫生与品牌）+ §1 配置目录/§10 步骤 1。spec §10 的步骤 2-5（commands / profiles+pipeline / projects / source ingestion）是**后续独立计划**，不在本增量。
- **占位符**：除有意的 `<your-org>` 仓库占位（实现者填自己的 GitHub org）外，无 TBD/TODO；每个改动给了精确文件、行/键、当前值、目标值。
- **命名一致**：全计划统一 `AgentHub`(显示) / `agenthub`(crate/npm/scheme/dir) / `agenthub_lib`(lib) / `dev.agenthub.app`(id) / `agenthub.db`(DB)；lib 改名（Task 1）与 main.rs 调用（Task 1 Step 4）及测试 `use` 一致。

---

## 后续增量（各自独立计划，待本增量绿色后再写）

2. `commands` 类型 + Commands tab（仅 Claude）
3. `profiles` + 激活流水线 + `apply_manifest` + Profiles tab（旗舰）
4. `projects` + Projects tab
5. Source ingestion（HTTP + 备份覆盖 + detach，无 git）
