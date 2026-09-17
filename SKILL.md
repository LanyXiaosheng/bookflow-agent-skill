---
name: Bookflow 全流程
description: >-
  use this when the user asks to run Bookflow / 番茄短篇生产：选题、立项、写章、汇总升华、配套配图、QA、打包，或排查 Bookflow API/管线踩坑
---

# Bookflow 全流程

把 Hermes 在 Bookflow 上练出来的调用纪律，变成可复用的 API 管线。AI 加速草稿；选题拍板、敏感操作、最终发布由人确认。

## 何时用

- 「跑 Bookflow / 全流程 / 打包下载」
- 选题 → 立项 → 写正文 → 汇总升华 → 配套配图 → QA → ZIP
- 排查大纲不建章、session 过期、章节 idx 错位、人名分裂等

## 环境

| 场景 | API Base | 说明 |
|------|----------|------|
| 本地 Mac 开发 | `http://localhost:3000/api` | `./dev.sh start`；旧栈常是 Docker Postgres `:5433` |
| 线上（若已部署） | 以当前环境为准（如同源 `/api` 或单独 API 主机） | 先 `GET /api/auth/me` 验登录；不要假定仍是本地 Postgres |

鉴权：`POST /api/auth/login` body `{"email","password"}` → cookie（常见名 `bookflow_session`，也可用 curl cookie jar）。每批开跑前用 `GET /api/auth/me` 验一次，401 就重登。

**禁止**：把账号密码、API Key、番茄 cookie 写进 skill 或提交到仓库。

## 正确业务顺序（不要跳）

```
账号复盘 / 定策略 → 选题（用户确认）→ 立项 → README → 角色 → 大纲
→ 按大纲建齐章节 → 整章写正文（优先）→ 章级/全书 QA → 汇总 → 升华 → 配套 → 配图 → 打包
```

有近期策略可直接选题；**不要跳过复盘就盲选题**，除非用户明确说跳过。

选题结果用表格给用户看（标题、四维分、热度、点评 + 你的推荐），**等用户拍板再立项**。

## 依赖链（跳步会 409）

```
readme → character_setup → outline → 手动创建 chapters → 正文(body)
→ book_summary → book_polished → side_dishes / publish
ai-story-image 依赖 readme
ai-publish 依赖 outline + 正文
```

## API 速查

### Auth / Seeds / Projects

- `POST /api/auth/login` body `{"email","password"}` → Set-Cookie
- `GET /api/seeds` · `POST /api/seeds` body **必填** `title` + `track` + `score`
  - `title`：约 1–25 字（超限会 400）
  - `score`：**四维** `{title_ctr, conflict, tagfit, novelty}` 各 1–10（旧七维已废）
  - `total_score` = 四维和（满分 40）；tier：`≥30` greenlight / `22–29` backlog / `<22` reject
  - 短时同 `(user,title,track)` 可能 409
- `POST /api/seeds/ai-generate` body `{"track":"..."}`（track 空 → 400）
- `POST /api/projects` body `{"seed_id":"<uuid>"}` → status 常为 `writing`（不是 `/from-seed/:id`）
- `GET /api/projects` · `GET /api/projects/:id` · `POST /api/projects/:id/transition`

### Artifacts / 流式生成（Accept: `text/event-stream`，必须吃完流）

- `POST /api/projects/:id/ai-readme/stream`
- `POST /api/projects/:id/ai-character-setup/stream`
- `POST /api/projects/:id/ai-outline/stream` → **只存 artifact，不自动建章**
- `POST /api/projects/:id/ai-book-summary/stream`（至少 1 章有 body）
- `POST /api/projects/:id/ai-book-polish/stream`
- `POST /api/projects/:id/ai-side-dishes/stream`
- `POST /api/projects/:id/ai-publish/stream`
- `POST /api/projects/:id/ai-story-image` **非流式** body `{"author_name","show_author","size":"2:3","quality":"high"}`
- `POST /api/projects/:id/ai-publish-qa`

### Chapters

- `POST /api/projects/:id/chapters` body `{"title"}` — 从大纲解析 `### 第N章 标题` 后按序建齐
- `GET /api/projects/:id/chapters` · `PUT /api/chapters/:id`

**写正文（优先整章）**

```
POST /api/chapters/:id/ai-write-full/stream
Body: {} 或 {"force":true} 或 {"extra_notes":"<针对 QA 的修改指令>"}
```

目标约 1500–2000 字/章。若 404，多半是旧 binary，需重建后端。

**旧模式（仅接口不可用时退回）**

1. `POST /api/chapters/:id/ai-beats`
2. 对每个 beat：`POST /api/chapters/:id/ai-write/stream` body **必须**带 `beat` + `prev_tail`（空 body → 422）
3. 拼接后 `PUT` 保存

章级 QA：`POST /api/chapters/:id/ai-qa` → `total_score` / `verdict`（pass≥35 / revise 30–34 / reject<30）。reject 用 `extra_notes` 定向重写，不要同 prompt 盲跑。

全书发布 QA：`POST /api/projects/:id/ai-publish-qa`；习惯上总分偏低（如 <30）不打包。

## SSE 消费要点

- timeout 建议 ≥ 600s（汇总/长章）
- 吃到 `event: done`；遇 `event: error` 停并报错
- cookie 隔几小时易失效，分批跑前验 `auth/me`

## 数据层三连坑（废稿级，必须防）

1. **重跑 setup/大纲 → 新 artifact 版本 + 人名换一套**  
   setup 确认后不要为了「验证落库」重跑。已写正文后更不能重跑设定。发现人名分裂：只留一版 artifact，推倒重写正文。

2. **中途删/插章节 → `idx` 与大纲「第N章」错位**  
   建章一次按大纲建齐；清理空章必须在写正文前做完，并核对 `idx/title` 与大纲一一对应。

3. **QA 高分 ≠ 数据层正确**  
   章级 QA 不查跨章人名、不查 idx。重写前先拉全章列表看开头人名/章号。

## 打包 ZIP（对齐前端）

通常含：配套素材 txt、优化升华 txt、封面 png（常裁 3:4）。txt 带 BOM（`\uFEFF`）便于 Windows 记事本。文件名去掉 `\ / : * ? " < > |`。

## 执行纪律

1. 先体检：服务可达、登录有效、项目状态、已有 artifact/章节全景
2. 分批跑：选题确认 → setup → 建章核对 → 写正文+QA → 后处理
3. 每批核实再下一批；不要一口气盲跑后才发现废稿
4. 不自动发布、签约、付款；发布动作由用户在平台完成

## 本地运维提示（开发机）

- 启动：`./dev.sh start`（需 Docker 若依赖本地库）
- 无 `/api/health` 时，用登录或其他 API 有无 HTTP 响应判断进程是否在
- AI 配置可能被 DB 覆盖 `.env`：改配置后确认实际生效源
- 中文 token 效率高：靠 `max_tokens` 物理兜底控字数，别只靠 prompt

## 交付检查

- [ ] 用户已确认选题（或明确授权你选）
- [ ] setup 只有一版且人名稳定
- [ ] 章节 idx 与大纲对齐、无空章/重复章
- [ ] 正文字数大致达标，关键章 QA 过关
- [ ] 汇总/升华/配套/配图齐全再打包
- [ ] 未把密钥写进产物或日志

## 来源

提炼自 Mac Hermes：`~/.hermes/skills/bookflow/bookflow-full-pipeline` 与 `software-development/bookflow-ops`，以及桌面 `bookflow-skill` 人机协作边界。具体主机、端口、库类型以当前部署为准。
