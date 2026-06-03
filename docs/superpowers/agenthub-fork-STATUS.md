# AgentHub Fork — 项目状态与续接手册（HANDOFF）

> **更新于 2026-06-02。** 这是「清空上下文后从这里继续」的权威文档。读完它 + git 历史 + 记忆文件即可无缝接续。

---

## 0. 一句话

把 cc-switch（Tauri 2 桌面应用，Rust + React）**fork** 成 **AgentHub**，逐步加入 agenthub 愿景的能力（profiles / 跨工具内容管理）。用户 Alba 选择了「fork + 用 cc-switch 的栈/风格二次开发」，放弃了原 Python agenthub CLI（退为蓝图）。

## 1. 仓库与位置

- **代码（fork）**：`/Users/fsm/project/MyProject/agentplugin/cc-switch-cloud`（origin `git@github.com:zgdd12345/cc-switch-cloud`，分支 `main`）。产品已改名 **AgentHub**。
- **文档/计划/规格（本仓）**：`/Users/fsm/project/MyProject/agentplugin/alba-agent-tool-hub`
  - 设计 spec：`docs/superpowers/specs/2026-06-01-agenthub-gui-ccswitch-fork-design.md`
  - 计划：`docs/superpowers/plans/2026-06-0X-*.md`（fork-foundation / inc2-commands / inc2.5-agents）
  - 本手册：`docs/superpowers/agenthub-fork-STATUS.md`
- **旧 Python agenthub CLI（仅作蓝图参考，不再运行）**：`/Users/fsm/project/MyProject/agentplugin/agenthub`（v0.4.1，171 测试）。
- **cc-switch 源码快照**（study 时用）：`/tmp/cc-switch-src`（临时，重启后可能没了；需要可从 `/tmp/ccsw.tar.gz` 重解，或直接读 cc-switch-cloud）。

## 2. 当前状态（main，全绿）

- `cargo test`（全量,含集成）：**全绿 0 failed**（lib 1513）；`clippy --all-targets -D warnings` 净；`cargo fmt --check` 净
- 前端 `vitest`：**314 passed**（57 文件）；`tsc --noEmit` 净
- 已交付并合并到 main 的增量:**1(fork)、2(Commands)、2.5(Agents)、3a(Profiles 核心+激活)、3b-1(dotfiles)、3b-2(`${VAR}`)、3b-3(CLAUDE.md)、3c(@tag)** —— **🎉 Profiles 旗舰(增量 3)全部收官。**
- DB schema 版本:**v16**(commands=v11,agents=v12,profiles=v13,apply_manifest.content_hash=v14,prompts.hidden=v15,skills.tags=v16)
- main HEAD:`c47df504`(3c merge)。`--all-targets` clippy 净。**下一支柱 = 增量 4(Projects)。**

## 3. ⚠️ 关键环境 GOTCHA（不看会浪费几小时）

1. **`cc` 被劫持**：全局 npm 包 `@wcldyx/claude-code-switcher` 在 `~/.nvm/.../bin/cc` 占用了 `cc`，遮蔽真编译器 → 仓库已有 **gitignored `.cargo/config.toml`** 强制 `/usr/bin/cc`，**保留它**。
2. **cargo 不在默认 PATH**：在 `~/.cargo/bin` → 所有 cargo 命令前 `export PATH="$HOME/.cargo/bin:$PATH"`。Rust 工具链 1.95（rust-toolchain.toml 钉定）。
3. **pnpm 11 预检会失败**（忽略 esbuild/msw 构建）→ 前端**直接调** `./node_modules/.bin/vitest run` 和 `./node_modules/.bin/tsc --noEmit`，**别用** `pnpm test:unit`/`pnpm typecheck`。
4. **网络慢**：cargo 下载超时就加 `CARGO_NET_RETRY=10 CARGO_HTTP_LOW_SPEED_LIMIT=0`。
5. Rust 测试隔离 HOME 用 `CC_SWITCH_TEST_HOME`（保留此环境变量名,勿改）；用 `#[serial]` 防并发。
6. **提交前先 `cargo fmt`**（多次发现实现 subagent 漏格式化）。tauri build 需 `dist/` → 干净 checkout 先 `./node_modules/.bin/vite build`。
7. 验证别用 `… | grep -c …` 收尾（grep 零匹配返回 1，会假报失败）；cargo test 真实退出码用 `> log 2>&1; echo $?` 捕获(管道会掩盖)。

## 4. 工作方法（每个增量都照此）

**纪律弧**：①（用户说 workflow 时）**study workflow** 摸清要镜像/复用的 cc-switch 机制 → ② **writing-plans** 写 bite-sized、无占位、TDD 的计划（存 plans/，提交）→ 用户审 → ③ **subagent-driven-development** 逐任务执行 → ④ **finishing-a-development-branch** 合并。

**分层模型（用户明确要求，已入全局 CLAUDE.md）**：困难/正确性关键（迁移、service 安全、激活流水线、总评审、对抗审查）用 **Opus**；中等（DAO、Tauri 命令、前端）用 **Sonnet**;机械抽取用 **Haiku**。study workflow 也分层。

**每任务**：派实现 subagent（给全文，不让它读计划文件）→ 我独立确定性核实（grep + 复跑测试）+ spec/质量评审（关键任务 Opus，含对抗式安全评审）→ 修 → 标记完成。每增量末 Opus 总评审 + 合并后复验。

**用户偏好**：分支收尾一贯选「本地 `--no-ff` 合并到 main」。

## 5. 已交付增量明细

- **增量 1 — fork 基础**（merge 9ee7cf00 + 修 e164ee68）：cc-switch→AgentHub 改名（identity `dev.agenthub.app`/scheme `agenthub`/crate `agenthub`+`agenthub_lib`/tray/log + 用户可见 i18n/UI）；配置目录迁 `~/.agenthub` + `agenthub.db`（3 处绕过点，无脑裂）；禁用自动更新（方案 B）；剥离全部赞助/联盟（providers 保留）。**功能性 `cc-switch` 标识符有意保留**（proxy 错误码、codex 旧数据检测、WebDAV 远端名 `cc-switch-sync`、localStorage 键、SQL 导出头、`CC_SWITCH_TEST_HOME`、`SkillStorageLocation="cc_switch"` enum）。修了一个上游日期 flaky 测试（usage_rollup 用本地日）。
- **增量 2 — Commands（仅 Claude）**（merge ef70c418）：`commands` 表 v11 + `CommandService`（内容存 DB,启用时**原子直写** `~/.claude/commands/<name>.md`,**安全 reconcile**:只遍历 DB 行、绝不枚举删用户文件,name 校验防穿越)+ 7 Tauri 命令 + Commands tab/面板/编辑对话框 + i18n。设计=内容存 DB + 直写(非软链)。
- **增量 2.5 — Agents（仅 Claude）**（merge cc6334c4）：commands 的精确克隆 → `agents` 表 v12 + `AgentService`（→ `~/.claude/agents/<name>.md`，同安全语义）+ 7 Tauri 命令 + **填充了原占位 Agents 视图**。顺手修 WebDAV 同步白名单漏洞（加 `commands`+`agents`）。
- **增量 3a — Profiles 核心 + 激活（旗舰）**（merge `00446655`，分支已删）：`profiles`/`profile_dotfiles`/`apply_manifest` 表（schema **v13**；后两表建好但 3a 无 Rust 读写，留 3b）+ `Profile/ProfileSpec/ProfileContent` + profiles DAO（CRUD + `set/clear_active` 单行不变量事务）+ **`ProfileService::activate/deactivate`**（无状态 unit struct）+ 8 Tauri 命令 + WebDAV 接线 + 前端 Profiles 视图（api/hook/Panel/EditDialog + App.tsx five-touch + Layers nav + 四语 i18n）。**激活 = 纯 flag 翻转 + reconcile**（①复用 `ProviderService::switch`,provider 缺失则 warn+跳过；②按字面名全覆盖 4 类 enable-flag 到 spec;③4 个 reconciler **恒最后无条件**跑;④`set_active_profile` 最后）。**关键设计修正(超越原 study)：apply_manifest 在 3a 冗余——reconciler 已安全收敛,manifest 推迟到 3b 给 dotfiles 用。** **安全 blocker 修复**：`skill.rs` `sync_to_app_dir`/`remove_from_app`/reconcile-disable 分支全部硬化为「绝不 `remove_dir_all` 用户真实目录」(软链/内容哈希判别 + skip+warn)。三视角 Opus 总评审全 SOUND。
  - **3a 遗留 minor backlog(非阻断,记 3b 顺带)**：(1) `set_active_profile` 目标行不存在时仍提交(清空全部)——调用方 `activate` 已校验,可加 affected-rows 守卫;(2) skill 内容哈希 `compute_dir_hash` 跳过隐藏文件 → copy-mode 受管拷贝里用户新增的 `.dotfile` 可能在 disable 时被删(极窄;默认 Auto 走软链不受影响);(3) Profiles nav 按钮只在默认 app 分支(按计划,profiles 为 claude-only);(4) profiles 跨设备同步可致 `current_provider_id` 悬挂 FK——激活期 missing-provider 守卫已优雅降级。
- **增量 3b-1 — Profile dotfiles 第一片(settings.json + statusline + manifest)**（merge `7c4f2bab`，分支已删）：`apply_manifest.content_hash`(schema **v14**,迁移用幂等 `add_column_if_missing`)+ `profile_dotfiles`/`manifest` DAO + 模型。**settings.json = 确定性重建**:active profile 的 settings.json 片段织入**唯一** effective-settings 构建器(`build_effective_settings_with_common_config`,provider→common→profile 三层 `json_deep_merge`),每次写/re-sync 都完整重生,**不进 manifest、无 diff 撤销**;backfill 用 `strip_fragment_restoring_provider_owned` **对称剥离**片段(差值覆盖时**恢复 provider 原值**,修复了 provider-token 污染 CRITICAL + 评审发现的 provider-key 耐久性 important)。`statusline.sh` 整文件渲染(`profile_render.rs`:真路径校验器拒 `..`/绝对、**拒覆盖未纳管用户文件**、内容哈希所有权、manifest 只删我方未改文件)。activate 新时序:先读 outgoing → 渲染整文件 + manifest → 移除 outgoing 整文件 → `set_active` 倒二 → **最后显式重写 settings.json**(经 `get_effective_current_provider`)。deactivate = 确定性拆除(无 profile 层重写 + 删本 profile 整文件,保 provider 键)。4 Tauri 命令 + 前端 ProfileEditDialog 两个 dotfile 文本框 + 四语。片段**字面**用(`${VAR}` 在 3b-2)。三视角 Opus 总评审:safety 修 2 important 后全绿,consistency/interaction SOUND。
  - **3b-1 遗留 minor(记 3b-2)**:(1) `validate_rel_path` 未 canonicalize,`~/.claude` 下预置软链子目录理论上可绕(被 render 所有权门缓解,超出 3b-1 威胁模型);(2) 死代码 `ConfigService::sync_current_providers_to_live`(无生产调用者,可删/`#[cfg(test)]`);(3) proxy 接管模式的 live 写不带 profile 片段(该模式 proxy 自管 live,可接受,文档化);(4) `get_profile_manifest` 命令无前端包装(3b-2 用);(5) `setDotfile` 已修为 `Promise<void>`。
- **增量 3b-2 — `${VAR}` 模板引擎(settings.json + statusline)+ vars 编辑器**（merge `d86a7d66`，分支已删）：`services/profile_vars.rs`(`build_var_map` 优先级 **profile.spec.vars > 激活 provider settings_config.env > 进程env[白名单 `ANTHROPIC_`/`AGENTHUB_`/`CLAUDE_`]** + `substitute_vars`:`${NAME}`/`${NAME:-default}`、**无递归**、json-escape、未知保留字面+警告)。渲染应用于 **settings.json 片段**(`build_effective_settings_with_common_config` 内 `from_str` 前,json_escape=true)与 **statusline.sh**(activate,json_escape=false);三处共用一张 map(确定性)。**backfill 用「重渲染后」的出向片段剥离**(`strip_fragment_restoring_provider_owned`)——修复渲染密钥回灌 provider 的 CRITICAL(评审中 T2 抓到并 revert 验证)。前端:vars 键值编辑器(写 `spec.vars`)+ 只读 "Applied files"(manifest)视图 + `get_profile_manifest` 包装。清 3b-1 minor:`validate_rel_path` 加 canonicalize 软链逃逸防护;**注意**:T4 误删 `ConfigService::sync_*` 簇(集成测试 `import_export_sync.rs` 在用,非死代码)→ 已恢复(教训:判死代码须 grep `tests/`)。**`${VAR}` 不渲染 CLAUDE.md;无 schema 变更。** 三视角 Opus 总评审全 **SOUND**(0 blocker)。
  - **3b-2 遗留 minor(记 3b-3)**:进程env(最低层)值跨会话变动可让 settings.json 片段里引用的 process-env 值在 backfill 重渲染时失配(窄;已加代码注释,建议片段用 spec.vars/provider.env 这两层 DB 稳定源)。
- **增量 3b-3 — CLAUDE.md 仲裁(经 PromptService 隐藏 prompt 行)**（merge `13e1118d`，分支已删）：profile 携带**字面** CLAUDE.md(存 profile_dotfiles rel_path=="CLAUDE.md",**不渲染 `${VAR}`** → 渲染密钥不入 DB)。profile **驱动 PromptService**(`~/.claude/CLAUDE.md` 唯一写者),经派生隐藏行 `__profile__:<id>` + `enable_prompt`。schema **v15** 加 `prompts.hidden`:`get_prompts` 过滤(UI/import 不见隐藏行)、新增 `get_prompts_with_hidden`/`get_prompt_with_hidden`;**enable 单一启用 sweep / backfill 扫描 / upsert any_enabled / delete 守卫全改用 `_with_hidden`**;enable_prompt **对 `__profile__:` 启用行跳过 backfill**(模板权威,用户手改不灌回模板)。activate step5b(set_active 后、settings 重建前,**硬门 AppType::Claude**)建/更新隐藏行并 enable;**切到无 CLAUDE.md 的 profile 时拆除出向隐藏行**(评审 important 修复:否则旧 CLAUDE.md 残留);deactivate 拉黑;delete_profile 清隐藏行(None 安全、无孤儿)。upsert_prompt 拒对 `__profile__:` 行 enabled=true(防御性)。前端 ProfileEditDialog CLAUDE.md textarea(空则删)+ 四语。三视角 Opus 总评审:safety/consistency SOUND,regression 修 1 important 后全绿。
  - **3b-3 遗留 minor(非阻断)**:(1) `__profile__:` 保留前缀仅在 upsert(enabled=true)拦截,未在所有命令层校验(shipped UI 不可达);(2) CLAUDE.md textarea 未 app-gate(与 settings/statusline 同款,后端硬门兜底,inert);(3) delete_profile 用 active_profile 表代理隐藏行 enabled 态(理论边缘)。
- **增量 3c — @tag 选择器(Profiles 旗舰收官)**（merge `c47df504`，分支已删）：profile spec.content 的条目前缀 `@` = tag 选择器,激活时展开成「该内容类型中 tags 含此 tag 的全部项」并与字面名取并集翻 enable-flag。`skills.tags` 列(**v16**,`TEXT NOT NULL DEFAULT '[]'`,幂等 `add_column_if_missing` + table_exists 守卫)+ `InstalledSkill.tags`(全仓 16 处 `InstalledSkill{}` 字面含 tests/skill_sync.rs 均补;update 站点 `skill.tags.clone()` 防清空)+ `update_skill_tags` DAO/命令/前端编辑器。解析用纯函数 `resolve_selectors`(services/profile.rs):**大小写敏感、内存解析(非 json_each)**、字面项行为与 3a 完全一致(零回归)、未知/空 @tag → 零展开 + **可区分**警告非报错、并集去重(HashSet)、skills/mcp 仅翻本 app flag(mcp 先不可变借用构 want 再 values_mut)。前端 UnifiedSkillsPanel 加标签输入 + ProfileEditDialog 4 列表 placeholder 提示 @tagname + 四语。三视角 Opus 总评审全 **SOUND**(0 blocker;2 minor:v16 迁移测试未走真·无列 ALTER 路径[helper 已验证]、缺字面∩@tag 同项去重测试[HashSet 可证])。**修了一处回归:T2 仅 grep src/,漏了 tests/skill_sync.rs 的 4 处 InstalledSkill 字面 → `--all-targets` 编译失败 → 已补(同 3b-2 教训:判改点须含 tests/)。**

**已确立模式**：新「Claude-only 单文件内容类型」= 表(id/name/content/description/tags/enabled_claude/installed_at) + DAO + Service(直写+安全 reconcile) + 7 Tauri 命令 + 前端 tab(api/hook/Panel/EditDialog) + i18n。commands 与 agents 是两份范本。

## 6. 路线图

- **增量 3 — Profiles（旗舰）✅ 已收官**：3a + 3b-1 + 3b-2 + 3b-3 + 3c 全部交付并合并(3c = `c47df504`)。profile = per-tool 命名配置,绑 provider + 内容集(skills/commands/agents/mcp,按字面名或 `@tag`)+ dotfiles(settings.json 片段确定性重建 / statusline.sh / CLAUDE.md 经 prompt 子系统 / `${VAR}` 模板),激活=切 provider + 翻 flag + reconcile + dotfile 渲染 + manifest 安全增删,active=唯一真相源。**下一步 = 增量 4(Projects)。**
- **增量 4 — Projects**：按目录绑定不同内容（`<project>/.<tool>/`），与全局通道正交。
- **增量 5 — Source ingestion**：从上游 git 仓拉 skills/commands/agents，**不用 git**（HTTP archive 下载 + 备份后覆盖 + detach）。

## 7. 增量 3（Profiles）—— study 结论（**3a 已交付**；下方为 3b/3c 仍有效的依据）

> ⚠ **3a 已实现并合并**（见 §5）。下面 study 中关于「apply_manifest 在激活时记录并安全移除」的设想已被**修正**：3a 证明 reconciler 已能安全收敛 4 类内容,manifest 在 3a 冗余 → **manifest 的实际写入/删除推迟到 3b**(给无 DB-flag 的 dotfiles 用,届时 4 类内容也一并纳入 manifest)。其余 study 结论(REUSE 映射、settings.json `json_deep_merge` 冲突、`${VAR}` 来源、@tag/skills.tags 缺口、WebDAV 分类)对 **3b/3c 仍然有效**。3a 详细计划见 `docs/superpowers/plans/2026-06-02-agenthub-inc3a-profiles.md`。

**愿景**：profile（per-tool、命名）绑定 (a) 该工具的一个 provider，(b) 启用的内容集（skills/commands/agents/mcp，按字面名或 @tag），(c) dotfiles（CLAUDE.md/settings.json/statusline.sh，含 `${VAR}`）。**激活 profile** = 切 provider + 把内容启用标志翻成 profile 的集合并落地 + 渲染 dotfiles + 在 **apply_manifest** 记录所写，以便下次切换**安全移除上次所建**（绝不碰用户文件）。一致性规则：激活 profile = 唯一真相源（激活态下手动开关即编辑该 profile）。v1 = **仅全局通道**（项目通道是增量 4）。

**study 关键结论（REUSE vs MISSING）**：
- **provider 切换 = REUSE** `ProviderService::switch`（src-tauri/src/services/provider/mod.rs；处理 proxy 接管热切换分支、backfill 保留手改、设设备本地 settings.json `current_provider_<app>` + DB `is_current`、写原生配置、MCP 同步）。激活的「设 provider」步直接调它。
- **内容落地多数 REUSE**：skills（`services/skill.rs` `sync_to_app_dir`/`sync_to_app` 软链+reconcile）、commands（`services/command.rs` 直写+reconcile）、agents（`services/agent.rs` 同）、mcp（`services/mcp.rs` `sync_all_enabled` 合并写；**Claude MCP 在 `~/.claude.json` 不在 settings.json**；Gemini MCP 在 `~/.gemini/settings.json`）、prompts（`prompt.rs` 是现有 CLAUDE.md/AGENTS.md 整文件写入者）。
- **最尖锐冲突**：Claude `settings.json` 现在是 provider blob **整文件覆盖**（`services/config.rs` via `sanitize_claude_settings_for_live`）。profile 的 settings.json 层**必须**在 provider 写之后用 `json_deep_merge`(live.rs:99)**深合并片段**、并记录片段以便 `json_deep_remove`(live.rs:117) 撤销——**绝不整文件覆盖**。statusline.sh 无人写（profile 可干净接管）。CLAUDE.md 被 PromptService 整文件占用 → 需仲裁（建议 profile 驱动 prompt 子系统）。
- **apply_manifest 必须新建且 DEVICE-LOCAL（不同步）**：因一致性规则会重写 DB 启用标志,reconciler 无法重建上个 profile 建了什么 → 必须显式记录每件磁盘产物（软链/写入文件/合并片段/翻的标志）。类比 `proxy_live_backup`（已在 SYNC_SKIP/PRESERVE）。
- **`${VAR}` 引擎缺失**：无密钥库/模板引擎。值主源 = 激活 provider 的 `settings_config.env`（REUSE，已含 ANTHROPIC_AUTH_TOKEN 等）；需建替换函数 + profile-vars 表 + 优先级。安全：manifest 别存渲染后的密钥,存模板 id+hash。
- **@tag**：`mcp_servers`/`commands`/`agents` 已有 `tags` 列;**`skills` 表缺 `tags` 列**(3c 加)。解析用 `json_each`。
- **WebDAV**：新表自动进整库快照导出;`profiles`/`profile_dotfiles` 加进 `should_trigger_for_table` 同步白名单;`apply_manifest` 加进 `SYNC_SKIP_TABLES`+`SYNC_PRESERVE_TABLES`(设备本地)。
- **前端**:同 Commands/Agents 的 App.tsx 接线模式(View union/VALID_VIEWS/renderContent/toolbar/header)加 Profiles 视图(全局,不限 app);加 active-profile 徽章。

**新表 DDL（schema v13，study 提议）**：
```sql
profiles(id TEXT PK, app_type TEXT NOT NULL, name TEXT NOT NULL, description TEXT,
  is_active BOOLEAN DEFAULT 0, current_provider_id TEXT, spec TEXT NOT NULL DEFAULT '{}',
  sort_index INTEGER, created_at INTEGER, UNIQUE(app_type,name),
  FOREIGN KEY(current_provider_id,app_type) REFERENCES providers(id,app_type) ON DELETE SET NULL)
profile_dotfiles(profile_id TEXT, rel_path TEXT, content TEXT, PRIMARY KEY(profile_id,rel_path),
  FOREIGN KEY(profile_id) REFERENCES profiles(id) ON DELETE CASCADE)
apply_manifest(id INTEGER PK AUTOINCREMENT, channel TEXT DEFAULT 'global', profile_id TEXT,
  app_type TEXT, target_path TEXT, kind TEXT, created_at INTEGER,
  FOREIGN KEY(profile_id) REFERENCES profiles(id) ON DELETE CASCADE)
```
profiles.spec JSON 形如 `{content:{skills:[],commands:[],agents:[],mcp:[]}, vars:{}}`（3a 按字面名;@tag 在 3c）。

**3a 范围（下一步要做的）**：profiles/profile_dotfiles/apply_manifest 表（v13）+ ProfileService 激活流水线（调 `ProviderService::switch` + 按 spec 翻 skills/commands/agents/mcp 启用并跑各自现有 reconciler + apply_manifest 安全增删）+ Profiles tab/面板（建/激活/编辑 profile、active 徽章）+ 「激活=唯一真相源」一致性。**不含** dotfile 渲染/`${VAR}`（3b）、@tag（3c）。激活流水线 + apply_manifest 安全语义是重点 → 用 **Opus**。

**study workflow run id**：`wf_7e36f97a-3b0`（4 份报告已读;若要原文可重跑或查 transcript）。

## 8. 清空上下文后如何继续（当前里程碑:Profiles 旗舰已收官,下一步 = 增量 4 Projects）

1. 新会话自动加载记忆 `agenthub-ccswitch-direction.md`（含 Profiles 收官摘要 + 环境 gotcha + 下一步)。
2. 读本手册:**§3 环境 gotcha(必看)、§4 工作方法、§5 已交付明细、§6 路线图**。(§7 是增量 3 的 study,Profiles 已全做完,仅供回溯。)
3. **说「开始增量 4」** 即可,据既定纪律弧推进:
   - ① **计划预备 workflow**(核实当前代码锚点 + Opus 设计 + Opus 对抗式查漏,分层模型)
   - ② **writing-plans** 写 bite-sized/TDD/无占位计划(存 `docs/superpowers/plans/`,提交)→ 交用户审
   - ③ **subagent-driven** 分层批次执行:每任务 实现 subagent →(关键任务)spec + 对抗式安全评审 /(次要)spec + 质量评审 → 修;Opus 用于迁移/激活/安全核心,Sonnet 用于 DAO/命令/前端,事实抽取用 Sonnet/Haiku
   - ④ **finishing-a-development-branch**:本地 `--no-ff` 合并 main + 合并后复验全绿 + 删分支 + 还原 pnpm 噪声 + 刷新本手册/记忆。
4. **增量 4 = Projects**:按目录绑定 profile/内容集(`<project>/.<tool>/`),与全局通道正交。**`apply_manifest` 的 `channel` 列已为 `project:<path>` 预留**(3a 建,目前恒 'global')。增量 5 = Source ingestion(HTTP 拉取,不用 git,备份+替换+detach)。
5. **复发教训(务必)**:改某 struct 字段 / 删 pub API 时,**必须 grep `tests/`**——集成测试是独立 crate,只 grep `src/` 会让 `--lib` 通过但 `--all-targets`/全量 `cargo test` 编译失败(3b-2 的 ConfigService、3c 的 InstalledSkill 各踩一次)。**验证门用 `cargo clippy --all-targets -- -D warnings` + 全量 `cargo test`(非 `--lib`,否则集成测试不编译)+ 前端 `./node_modules/.bin/{tsc --noEmit, vitest run}`。**
6. 可选:`pnpm tauri dev`(cc-switch-cloud,注意 §3 PATH/cc gotcha)验收完整 Profiles(provider + 内容 + @tag + dotfiles + `${VAR}` + CLAUDE.md)。

## 9. 待办/已记的小项（非阻断,可在后续增量顺带）

- **Profiles minors**:(3b-2)进程env 跨会话变动可致 settings.json 片段 backfill 重渲染失配(窄,已注释,建议片段用 spec.vars/provider.env);(3b-3)`__profile__:` 保留前缀仅在 upsert(enabled=true)拦截、未全命令层校验;CLAUDE.md textarea 未 app-gate(后端硬门兜底);delete_profile 用 active_profile 表代理隐藏行 enabled 态;(3c)v16 迁移测试未走真·无列 ALTER 路径(helper 已验证);缺「字面∩@tag 同项」去重测试(HashSet 可证)。
- 旧:agents/commands service 硬化 nice-to-have(符号链接删除回归测试、import 重名 stem、name 长度上限);AgentsPanel 未用 `currentApp?` prop;若干 cc-switch 文档注释仍提旧名(cosmetic,功能性标识符有意保留,见 §5)。
