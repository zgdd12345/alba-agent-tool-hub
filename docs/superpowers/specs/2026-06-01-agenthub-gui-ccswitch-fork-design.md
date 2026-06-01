---
title: agenthub GUI — 基于 cc-switch fork 的二次开发设计
date: 2026-06-01
status: DRAFT — 待 Alba 审阅
owner: Alba
schema_version: 1
supersedes: 无（与 2026-05-15 的 agenthub CLI 设计并存；CLI 退为蓝图/参考）
---

# agenthub GUI 设计文档（cc-switch fork）

## §0 背景与目标

Alba 偏好 [cc-switch](https://github.com/farion1231/cc-switch) 的桌面 GUI 风格，并希望在它的基础上二次开发，得到一个**既能切换 AI 供应商、又能管理自己的扩展内容（skills/commands/agents/mcp/plugins）**的统一桌面应用。

### 0.1 已核实的基础事实（2026-06-01）

- **agenthub 已是工作版 Python CLI**，位于兄弟仓 `/Users/fsm/project/MyProject/agentplugin/agenthub`：git tags 到 **v0.4.1**（HEAD `1067395`），46 个源文件，五大支柱齐全，**171 个测试在其 `.venv` 全通过**。`pyproject`/README 版本串（0.1.0 / v0.1.2）已过期，以 git tags + 测试为准。
- 本仓 `alba-agent-tool-hub` 是该项目的**设计/文档与内容主目录**。
- **cc-switch v3.16.0**：Tauri 2（Rust ~114K 行 + React 18/TS ~72K 行），MIT；自称 "All-in-One Manager for Claude Code, Codex, Gemini CLI, OpenCode, OpenClaw & Hermes"。单一真相源是 **SQLite**（`~/.cc-switch/cc-switch.db`，`SCHEMA_VERSION=10`，~15 张表）；切换 = 把 provider 配置**合并写入**各工具原生配置文件；多机同步用 **WebDAV 整库快照**；内置 LLM 代理转发 + 用量统计 + failover；由 ~25 个 API relay 联盟赞助商资助。

### 0.2 关键决策（来自 2026-06-01 头脑风暴）

1. **能力诉求 = 两者都要**：供应商一键切换（cc-switch 强项）+ 用 cc-switch 的风格管理自己的内容（agenthub 强项）。
2. **技术栈 = 完整 Tauri 桌面应用**（与 cc-switch 同栈，视觉最接近）。
3. **做法 = fork cc-switch 二次开发**（不另起炉灶）。
4. **存储 = SQLite 全程**，放弃 agenthub 原本「git 可 diff 纯文本内容仓」哲学。
5. **Python agenthub = 蓝图/参考**，其设计与 171 测试作为 Rust 重写的验收基准；停用 Python 运行时。
6. **集成方式 = 叠加式（方案 A）**：profile 作为「启用状态快照」叠加在 cc-switch 现有 provider/内容模型之上，**最小化内核改动**。
7. **保留 cc-switch 的代理/relay/用量/failover 子系统**。
8. **整个工具不依赖 git**：内容在 SQLite、源拉取走 HTTP、多机同步沿用 WebDAV 快照。
9. **UI = 布局 ③（cc-switch 原貌 + 新增 tab）**：最小改动，加 Profiles/Projects/Commands 三个 tab。
10. **v1 范围 = agenthub 四大独有能力全要**：Profiles+dotfile（旗舰）、项目/全局双通道、Source ingestion、Slash commands（仅 Claude）。

## §1 整体架构

```
fork(cc-switch)  ──>  agenthub (GUI)
├── 保留：provider 切换 / 代理转发 / 用量统计 / failover / MCP / skills / prompts / agents / WebDAV 同步
├── 改造：品牌、配置目录、updater、剥离赞助商
└── 新增（叠加式，原生 Rust + SQLite）：
    ① Profiles + dotfile 管理（旗舰）
    ② 项目级 / 全局 双通道
    ③ Source ingestion（HTTP，无 git）
    ④ Slash commands（仅 Claude）
```

- **单一真相源**：SQLite（沿用 cc-switch 模型，配置目录随品牌改为 `~/.agenthub/`；注意与已退役的 Python 工具旧目录区分，实现时确认无冲突或换用独立目录名）。
- **设备本地状态**：沿用 cc-switch 的 `settings.json`（不进同步库），承载「激活的 profile / provider」与「cwd→project 映射」。
- **混合写入模型**（落地到各工具原生位置）：
  - **软链 SYMLINK**（agenthub 式）：skills / commands / agents → `~/.<tool>/{skills,commands,agents}/`；复用并扩展 cc-switch 的 skill 软链分发器；由 `apply_manifest` 跟踪。
  - **合并写入 MERGE**（cc-switch 式）：provider 配置 / MCP / `settings.json` 键 → 复用 `config.rs::atomic_write` + 结构保留合并，只改受管键、保留外来键。
  - **整文件渲染 RENDER**（profile 拥有）：CLAUDE.md / statusline.sh 等 dotfile → `${VAR}` 替换后整文件 `atomic_write`；由 `apply_manifest` 跟踪。

## §2 数据模型（SQLite，叠加式）

**复用** cc-switch 现有表：`providers (id, app_type)` 含 `is_current`、`skills`、`mcp_servers`（含 `tags` + 各家 enable 标志）、`prompts`、agents、proxy 相关表。**新增**：

1. **`commands`**（新内容类型）：`(id, name, content, description, tags, <各家 enable 标志>, <source 列>)`。v1 仅 Claude 落地。
2. **`profiles`**：`(id, app_type, name, description, is_active, current_provider_id, sort_index, spec JSON)`。**按工具**（per-tool）。
   - `spec`（JSON）= include 列表（skills/commands/agents/mcp），**支持字面名 + `@tag`**，与 agenthub `profile.toml` 一一对应；激活时解析 `@tag`。（plugins 见 §11，v1 不纳入。）
   - `current_provider_id`：该 profile 钉定的 provider（switch-mode 工具）。
3. **`profile_dotfiles`**：`(profile_id, rel_path, content)` — CLAUDE.md / settings.json / statusline.sh 等，内容含 `${VAR}`。
   - **`${VAR}` 取值来源**：新增 `variables` 键值表（或复用 cc-switch 的 `settings` 表）。默认随 SQLite 库同步（与 cc-switch 现有把密钥存进库的行为一致）；若 Alba 要求密钥**不同步**，则改存设备本地 `settings.json` 的专用节段——此为实现时可选项。激活时缺值立即报错（§3.1）。
4. **Tags 列**：给 `skills`/`commands`/agents/`prompts`/plugins 补 `tags` 列（`mcp_servers` 已有），使 `@tag` 解析为 DB 查询。
5. **Source 跟踪列**（折叠 agenthub 的 sidecar+lock 到内容表）：`source_type / source_url / source_path / source_ref / source_ref_type / locked_sha / content_hash / imported_at / last_updated`（扩展 cc-switch `skills` 已有的 repo_owner/branch/content_hash）。
6. **`projects`**：`(id, name, description, created_at)` + 每 (project_id, app_type) 的 `spec JSON`（同 profile 的 include 列表，**不含 dotfiles**）。cwd→project 映射放**设备本地** `settings.json`（不同步），对齐 agenthub「项目配置同步、本机映射不同步」。
7. **`apply_manifest`**（关键安全表，cc-switch 无）：`(id, channel[global|project], project_id, app_type, target_path, kind[symlink|render], created_at)` — 记录每次 apply 建了什么，**只回收自己建的**（移植 agenthub 的 manifest-driven UNLINK）。

**Schema 版本**：沿用 cc-switch 的数字 `SCHEMA_VERSION` + 编号迁移 + 升级前自动备份 `.db`；新增表通过新迁移引入。

## §3 Profile 激活流水线（旗舰核心）

激活某 (app_type, profile) 时，六阶段执行：

1. **解析**：读 `profile.spec` → 展开 `@tag` 得最终内容集；解析 `${VAR}`（**缺值立即报错**）。
2. **计划**：期望状态 vs 现场状态 → 计划项。**GUI 弹确认对话框**展示语义 plan（可展开文件级 / 内容 diff = agenthub 三层 dry-run）。
3. **预检**：扫目标位置的「非本应用管理」文件 → **撞车则在任何写入前 abort**（不写一个字节）。
4. **应用**：按 §1 混合写入模型落地（provider 切换复用 cc-switch `live.rs`；翻各家 enable 标志到 profile 集合；软链 skills/commands/agents；渲染 dotfiles；渲染 MCP 复用 cc-switch 现有 renderer）。**先写 `apply_manifest` 行**，再动文件。
5. **回收**：`apply_manifest` 里「上次建、本次不在集合」的软链/渲染件 → UNLINK。
6. **收尾**：置 `profiles.is_active`；失败按 cc-switch 现有 rollback 回退；Toast 报告。

### 3.1 安全保证（移植自 agenthub）

- **预检撞车检查**：写入前确认目标无冲突。
- **Manifest-driven UNLINK**：只回收 `apply_manifest` 记录的条目；用户预存文件永不被碰；首次 apply 无 manifest → UNLINK 集合为空。
- **原子写入**：复用 `config.rs::atomic_write`（temp + fsync + rename + 权限保留）。
- **Plan-before-mutate**：GUI 确认对话框默认先展示 plan。
- **不静默销毁**：覆盖前备份。

### 3.2 一致性（解决叠加式的双入口问题）

**激活 profile = 唯一真相源**。用户手动开关某 skill/MCP/command 时，**直接编辑激活 profile 的 `spec` 并即时重跑流水线** —— 所见即所应用，零漂移。想做实验就新建一个 profile。

### 3.3 全局 / 项目 双通道（物理正交）

同一条流水线，只换目标根：全局 profile → `~/.<tool>/`；项目 → `<project>/.<tool>/`。`apply_manifest` 行用 `channel` 区分；**全局 apply 永不碰项目目录，反之亦然**。

## §4 项目级 / 全局双通道

- `projects` 表（同步）持每 (project, app_type) 的 include 列表（同 profile，不含 dotfile）。
- **桌面 GUI 适配**：无 cwd 概念 → 用**文件夹选择器**选项目根 → 映射到 project id（映射存设备本地 `settings.json`）。
- 激活项目 = 跑激活流水线，目标根为项目目录，`channel=project`。
- **孤儿检测**（doctor 视图）：列出「映射的文件夹已不存在」「文件夹有映射但 project 缺失」等情况，提示 unlink 或新建。

## §5 Source Ingestion（无 git）

- **拉取**：HTTP 下载仓库 archive（tarball/zip）到指定 ref（复用 cc-switch 既有 GitHub skill 下载路径），无需 git 二进制。
- **锁定**：记 `content_hash` + 解析到的 commit sha。
- **更新**：比 `content_hash`；上游变了 → **先备份当前本地副本 → 再覆盖**；更新 `locked_sha`/`last_updated`。
- **本地改动保护**：本地 hash ≠ lock 时，更新前警告「你改过，更新会覆盖，旧版已备份至…」，给三选项：**覆盖（带备份）/ 跳过 / detach（停止跟踪、保留为原创）**。
- **明确不做**：三方合并、autostash+rebase 回放、临时 git 工作树（这些是 agenthub CLI 的 git 方案，本设计放弃）。

## §6 Slash Commands（v1 仅 Claude）

- `commands` 表 + 仿 skills 的「Commands」面板（列表 / 新建 / 从磁盘收编 / 从上游导入 / Claude enable / tags）。
- 落点：`~/.claude/commands/`（项目通道 `<project>/.claude/commands/`），软链分发。
- Codex/OpenCode 等的 command 支持 → v2（待这些工具支持用户级独立 command 后再做）。

## §7 UI / 导航（布局 ③：最小改动）

- **保留 cc-switch 现有布局**：顶部 per-app 分段 pill（Claude/Codex/Gemini/OpenCode/OpenClaw/Hermes）+ 横向视图 tab 行。
- **新增 tab**：`Profiles` / `Projects` / `Commands`，与既有 `Providers / MCP / Skills / Agents / Prompts / Proxy / Usage / Settings` 并列。
- **完全复用 cc-switch 视觉语言**：shadcn/Radix 组件、Tailwind token、玻璃拟态、暗色模式、Framer Motion、lobehub 图标、i18n。
- Profiles tab：当前 app 的 profile 列表，激活态高亮，「Activate」+ 组成摘要（provider · N skills · N mcp · dotfiles），编辑即改 spec 并重应用（§3.2）。

## §8 Fork 卫生与品牌

| 项 | 处置 |
|---|---|
| 品牌 | 改 app 名 / 包标识 / deeplink scheme / 窗口标题 / 仓库名；产品名沿用 **`agenthub`** |
| 配置目录 | `~/.cc-switch/` → `~/.agenthub/`（确认与旧 Python 工具目录无冲突） |
| 赞助商 | 剥离 README ~25 个赞助商 + 应用内赞助位；清除 provider 预设里的 `partnerPromotionKey` 促销码（预设本身保留） |
| 自动更新 | 替换 updater 公钥 + 端点为自有，或个人工具直接关闭 |
| 许可证 | 保留 MIT + 对 cc-switch 的署名（MIT 强制） |
| 代理子系统 | **保留** |

## §9 测试策略

- **Rust**：新增 DAO（profiles/projects/commands/source 列/apply_manifest）单元测试 + **激活流水线集成测试**；移植 agenthub spec §10.4 必测安全场景为验收基准：干净 apply、切换拔旧建新、**预检撞车写前 abort**、中途崩溃 doctor、`${VAR}` 缺失报错、项目 apply 不污染全局。
- **前端**：vitest 测 Profiles/Projects/Commands 面板。
- **行为对照**：agenthub 的 171 个 Python 测试作为 Rust 重写的行为参照表。
- 沿用 cc-switch 既有 CI / typecheck / lint。

## §10 实现顺序（最小改动优先；详细分解见实现计划）

1. **fork + 改名 agenthub + 剥赞助/促销码 + 换/关 updater + 改配置目录**，保留代理；先让它 build、能跑。
2. **`commands` 类型 + Commands tab（仅 Claude）** — 最简单的新支柱，先跑通「新内容类型 + 软链 + manifest」模式。
3. **`profiles` + 激活流水线 + apply_manifest + Profiles tab** — 旗舰（确认对话框 / 预检 / 混合写入 / manifest UNLINK / §3.2 一致性）。
4. **`projects` + Projects tab** — 复用激活流水线，目标根换项目目录（正交）。
5. **Source ingestion** — HTTP 拉取 + 备份后覆盖 + detach。

每步可独立交付。

## §11 明确不做（v1 范围外 / v2 backlog）

- agenthub 原「git 可 diff 纯文本内容仓 + git 同步 + 工具/内容双仓分离」哲学（已用 SQLite + WebDAV 取代）。
- Source 更新的三方合并 / autostash+rebase / 临时工作树（用「备份+覆盖+detach」取代）。
- Codex / OpenCode / 其它工具的 slash command（v1 仅 Claude）。
- **完整 plugin 管理支柱**（agenthub Pillar 4：whole-plugin 安装/版本/启用，marketplace + npm）：v1 不纳入；profile spec 的 include 列表暂不含 plugins。v1 沿用 cc-switch 既有的最小 plugin 处理。
- 真正的 profile 继承（沿用「无继承」；如需可后加 conf.d 片段）。
- 把 profile 抬为内核中心对象（方案 B）/ Profile 中心首页（布局 ①②）—— 保留为后续 UI 演进选项。
- 保留 Python agenthub 运行时（已退为蓝图）。

## §12 风险与缓解

- **Rust/Tauri 学习曲线**（Alba 是 Python 开发者）：以「最小改动 + 叠加式 + 复用 cc-switch 原语」压低门槛；分步交付，第 1、2 步先建立信心。
- **追上游成本**（cc-switch 月度更新）：叠加式尽量不动内核（`AppType` 枚举、provider/proxy 核心、schema 迁移冲突最痛）；新增能力集中在新表 + 新 tab + 新 DAO，降低合并面。
- **品牌/赞助剥离遗漏**：第 1 步设清单逐项核对（README、应用内、预设促销码、updater key/endpoint）。
- **Source ingestion 边界**：无 git 的「备份+覆盖」语义简单但会丢未 detach 的本地改动 —— 已用「更新前警告 + 备份 + detach」缓解。
- **混合写入与 cc-switch 既有写入并存**：profile 的软链/渲染 与 cc-switch 的 provider/MCP 合并写入同时作用于 `~/.<tool>/`，需保证 manifest 跟踪范围与 cc-switch 的「受管键」范围不重叠冲突；实现时在第 3 步重点验证。

## §13 决策摘要表

| # | 维度 | 选择 |
|---|---|---|
| 1 | 基础 | fork cc-switch 二次开发 |
| 2 | 集成方式 | 叠加式（方案 A），最小化内核改动 |
| 3 | 存储 | SQLite 全程（放弃 git 文件哲学） |
| 4 | Python agenthub | 蓝图/参考，Rust 重写 |
| 5 | git 依赖 | 无（HTTP 拉源 + WebDAV 同步） |
| 6 | 代理子系统 | 保留 |
| 7 | UI 布局 | ③ cc-switch 原貌 + 新增 tab |
| 8 | v1 能力 | Profiles+dotfile / 项目-全局双通道 / Source ingestion / Commands(仅 Claude) |
| 9 | Profile 作用域 | per-tool；激活 profile = 唯一真相源，手动开关即编辑并重应用 |
| 10 | Commands 工具范围 | v1 仅 Claude |
| 11 | Source 更新语义 | 备份 + 覆盖 + detach（无三方合并） |
| 12 | 项目身份 | 文件夹选择器 + 设备本地映射 |
| 13 | 安全 | 预检撞车 + manifest UNLINK + 原子写 + plan 确认 |
| 14 | 品牌 | 产品名 agenthub；剥赞助；换/关 updater；保留 MIT 署名 |
