# Bookflow Agent Skill

Agent skill for driving the **Bookflow / 番茄短篇** production platform end-to-end over HTTP/SSE. Same `SKILL.md` is meant for Hermes, Grok (Grok Bot / Cursor), Claude Code, and OpenAI Codex.

这是一套 **Agent Skill**：让 AI 代理按真实 Bookflow 平台 API 跑完整短篇生产管线——选题 → 立项 → 写章 → 汇总升华 → 配套配图 → QA → 打包。AI 加速草稿；**选题拍板、敏感操作、最终发布由人确认**。

入口始终是仓库根目录的 [`SKILL.md`](SKILL.md)。不要把本仓库当成独立虚构后端；代理应对着正在运行的 Bookflow 实例发 HTTP / SSE。

## 这是什么

Bookflow 是番茄短篇生产平台（选题、项目、章节、artifacts、配图、打包）。本 skill 把「怎么调这些 API、顺序是什么、哪些坑会废稿」写成代理可执行的纪律。

正确业务顺序（不要跳）：

```
账号复盘 / 定策略 → 选题（用户确认）→ 立项 → README → 角色 → 大纲
→ 按大纲建齐章节 → 整章写正文（优先）→ 章级/全书 QA → 汇总 → 升华 → 配套 → 配图 → 打包
```

跳过依赖会 409：`readme → character_setup → outline → 手动创建 chapters → 正文(body) → book_summary → book_polished → side_dishes / publish`。大纲接口**只存 artifact，不自动建章**。

路由与请求体以 [`SKILL.md`](SKILL.md) 和 [`api-routes-cheatsheet.md`](api-routes-cheatsheet.md) 为准，不要发明新端点。

## 来源

提炼自 **Hermes** 训练：那次训练是 **直接调用 Bookflow 平台 API**（登录 cookie、SSE 流式生成、建章、QA、打包），不是另一套虚构栈。

Hermes 原文作为参考保存在 [`hermes/`](hermes/)：

| 文件 | 原名 | 用途 |
|------|------|------|
| [`hermes/bookflow-full-pipeline-SKILL.md`](hermes/bookflow-full-pipeline-SKILL.md) | `hermes-bookflow-full-pipeline-SKILL.md` | 全流程管线原配方 |
| [`hermes/bookflow-ops-SKILL.md`](hermes/bookflow-ops-SKILL.md) | `hermes-bookflow-ops-SKILL.md` | 运维、踩坑、本地开发机细节 |

日常跑生产请用根目录 `SKILL.md`（已蒸馏、跨平台）。需要完整 curl 示例、本地 Docker/运维坑、或原始逐步配方时再翻 `hermes/`。

[`desktop-human-led-SKILL.md`](desktop-human-led-SKILL.md) 补充人机边界：AI 出候选与草稿，人做题材判断、人物取舍、核验和发布决定。

## 兼容性

同一份 `SKILL.md` 供下列代理使用：加载 skill 后，用 **HTTP + SSE** 调 Bookflow，方式相同。

| 平台 | 怎么用 |
|------|--------|
| **Hermes** | 安装/复制到 Hermes skills 目录（例如 `~/.hermes/skills/bookflow/`），让代理在「跑 Bookflow / 全流程 / 打包」时加载 |
| **Grok Bot / Cursor** | 作为 workflow skill 安装；用户提到 Bookflow、番茄短篇、选题立项写章打包时调用 |
| **Claude Code** | 按 `SKILL.md` 约定放到项目 skill（如 `.claude/skills/bookflow/SKILL.md`）或用户级 skills 目录 |
| **OpenAI Codex** | 把 `SKILL.md`（及需要时的 cheatsheet）放到/附到代理能读到的位置后开跑 |

各平台差异主要是 **skill 怎么被发现**；一旦加载，调用纪律相同：先 `GET /api/auth/me` 验登录，按依赖链分批跑，SSE 吃到 `event: done`。

本地开发常见 API Base：`http://localhost:3000/api`（`./dev.sh start`）。线上以当前部署为准（同源 `/api` 或单独 API 主机）。不要假定仍是本地 Docker Postgres。

## 各平台用法（简）

### Hermes

1. 将本仓库（至少 `SKILL.md`）复制到 Hermes skills 目录，例如 `~/.hermes/skills/bookflow/`。
2. 需要对照原配方时同时放入 `hermes/` 下两份原文。
3. 用户说「跑 Bookflow / 全流程 / 打包下载」时加载本 skill，对 Bookflow 发 HTTP/SSE。
4. Hermes 终端可能拦截 `curl \| python3` 或未确认的 POST；被拦就如实让用户放行，不要当成功。详见 `hermes/bookflow-ops-SKILL.md`。

### Grok Bot / Cursor

1. 将本仓库作为 **workflow skill** 安装（Cursor：项目或用户 skills，保证代理能读到根目录 `SKILL.md`）。
2. 用户要求 Bookflow / 番茄短篇生产时调用本 skill，不要另编一套流程。
3. 用环境里的 HTTP 能力调 Bookflow；cookie 用本地 jar / 运行时会话，**不要写进 skill 文件**。

### Claude Code

1. **项目 skill**：复制为 `<repo>/.claude/skills/bookflow/SKILL.md`（可同时带上 cheatsheet）。
2. **用户 skill**：放到 Claude Code 用户 skills 目录，使跨项目可用。
3. 触发描述见 `SKILL.md` front matter：`use this when the user asks to run Bookflow / 番茄短篇生产…`

### OpenAI Codex

1. 把 `SKILL.md` 放到 Codex 能读的工作区，或在任务里 **attach** 该文件。
2. 复杂管线可同时附上 `api-routes-cheatsheet.md`。
3. Codex 按 skill 纪律调用同一套 Bookflow HTTP/SSE；主机、端口、登录态由用户当前环境提供。

## 安全

- **仓库不含密钥。** 不要把账号密码、API Key、番茄 cookie、session 写进 skill、日志或提交。登录信息只存在本地环境或用户当面提供。
- 本仓库 `.gitignore` 已忽略 `.env`、`*.zip`、cookie 文件等。
- **禁止自动发布到番茄。** 不自动发布、签约、付款或提交敏感身份资料。发布由用户在平台确认后完成。
- 选题结果用表格给用户看（标题、四维分、热度、点评 + 推荐），**等用户拍板再立项**，除非用户明确授权代选。

Hermes 原文里曾出现过本地账号示例；收录时已换成占位符。运行时向用户要凭据，不要把真实密码写回本仓库。

## 仓库结构

```
README.md
LICENSE
.gitignore
SKILL.md                      # 入口：跨平台蒸馏版
api-routes-cheatsheet.md      # 路由速查（与 SKILL.md 一致，勿发明端点）
desktop-human-led-SKILL.md    # 人机协作边界
hermes/
  bookflow-full-pipeline-SKILL.md
  bookflow-ops-SKILL.md
```

## 代理执行要点（详见 SKILL.md）

1. 先体检：服务可达、登录有效、项目状态、已有 artifact/章节全景。
2. 分批跑：选题确认 → setup → 建章核对 → 写正文+QA → 后处理；每批核实再下一批。
3. 正文优先 `POST /api/chapters/:id/ai-write-full/stream`；整章接口 404 再退回 beats 模式。
4. SSE：`Accept: text/event-stream`，timeout 建议 ≥ 600s，吃到 `event: done`；`event: error` 则停。
5. 防废稿：setup/大纲确认后不要为「验证落库」重跑（会换人名）；建章一次按大纲建齐，避免 `idx` 错位；QA 高分不等于数据层正确。

License: [MIT](LICENSE)
