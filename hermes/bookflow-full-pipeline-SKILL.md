---
name: bookflow-full-pipeline
description: "Bookflow 一键全流程：全书汇总→优化升华→配套素材→小说配图→打包ZIP，适用于所有用户/项目"
triggers:
  - "跑全流程"
  - "一键全流程"
  - "全流程管线"
  - "打包下载"
  - "bookflow pipeline"
  - "热门数据"
  - "hot_tracks"
  - "更新热门"
  - "配图失败"
  - "generate_image"
---

# Bookflow 全流程管线

## 前置条件

1. bookflow 前后端在跑（`./dev.sh start`）
2. Docker Desktop 在跑（postgres 需要）
3. 如果是新项目（从 seed 立项），需要先跑 Step 0 前置准备
4. 如果项目已有正文（至少 1 章有 body），可以 `--skip-to 2` 跳过写正文

## ⚠️ 用户强调的正确全流程顺序

用户要求在跑一键全流程之前，必须先完成**账号复盘+定策略**。完整顺序是：

```
复盘分析（数据驱动）→ 制定策略（strategy）→ 生成选题 → 立项 → 写作全流程 → 打包发用户
```

**不要跳过复盘直接跑选题**。用户明确说过："再跑之前，复盘定策略，再到这些后续的流程"。

复盘流程见下方「账号复盘流程」章节，或参考 `references/weekly-review-workflow.md`。
如果该账号已有最近的复盘数据（`users.strategy` 非空），可以基于现有策略直接选题。

## 写作模式选择（重要）

**优先使用整章模式**（`ai-write-full/stream`），产出 1500-2000 字/章，节奏连贯。
逐beat模式（`ai-beats` + `ai-write/stream`）会导致每章 5000+ 字碎片拼接，仅在整章接口不可用时退回。
详见 `references/writing-mode-architecture.md`。

**QA 是必须环节**：写完正文后必须调用 `ai-publish-qa`，总分 <30 不打包。

## 从零开始的完整链路（新项目）

```
登录获取 session → ai-generate seeds → 创建 seed → POST /api/projects (seed_id) →
Step 0: README → 角色设定 → 大纲 → 手动创建 chapters →
Step 1: beats → write → save (每章) →
Step 2-6: 汇总 → 升华 → 配套 → 配图 → 打包
```

⚠️ 大纲生成后 **不会** 自动创建 chapters，必须手动解析大纲 artifact 中的 `### 第N章 标题` 然后逐个 POST 创建。
   - **已有正文的项目** — 直接从 Step 1（汇总）开始
   - **新建/空项目** — 必须先跑 Step 0（大纲→逐章写正文）再进后续步骤

## 获取必要参数

### 1. 获取用户 session

```bash
# 登录获取 session cookie
## 获取必要参数

### 1. 获取用户 session

```bash
# 登录获取 session cookie
curl -s http://localhost:3000/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"<email>","password":"<password>"}' \
  -D - | grep -i set-cookie
# 提取 bookflow_session=<token>
```

### 2. 获取项目 ID

```bash
curl -s http://localhost:3000/api/projects \
  -b 'bookflow_session=<token>' | python3 -c "
import sys, json
for p in json.loads(sys.stdin.read()):
    print(f\"{p['id']} | {p['title']}\")
"
```

### 3. 获取作者名

从 project 的 author 字段或用户配置中获取。常见作者名列表见 memory。

## 选题生成 + 立项（全流程起点）

### AI 生成选题

```
POST /api/seeds/ai-generate
Content-Type: application/json
Body: {"track":"打脸逆袭"}  # track 必填
```

返回 `{track, candidates: [{title, score:{title_ctr,conflict,tagfit,novelty}, why_buy, type, recommend_reason, heat}]}`

### 创建 Seed（手动/从 AI 结果中选）

```
POST /api/seeds
Content-Type: application/json
Body: {"title":"...","track":"打脸逆袭","score":{"title_ctr":9,"conflict":9,"tagfit":9,"novelty":5}}
```

score 必填，4 维各 1-10。返回含 total_score 和 tier（greenlight/backlog/reject）。

### 从 Seed 立项创建 Project

```
POST /api/projects
Content-Type: application/json
Body: {"seed_id":"<uuid>"}
```

返回新 project（status=writing）。⚠️ 不是 `/api/projects/from-seed/:id`，是标准 POST /api/projects 带 seed_id。

### 其他 Seed API

- `GET /api/seeds` — 列出当前用户所有 seeds
- `DELETE /api/seeds/:id` — 删除 seed（已被项目引用则 409）
- `POST /api/seeds/ai-score` — 对已有 seed 重新 AI 评分
- `POST /api/seeds/ai-recommend-track` — 推荐赛道（需 `{primaries:[], plots:[]}`）
- `GET /api/seeds/ai-drafts` — 查看 AI 生成的选题历史

## Step 0: AI 生成选题（在跑全流程之前）

当用户要求「推荐选题然后跑全流程」时，先生成选题再立项：

### 生成选题

```bash
# 需要指定 track（赛道），否则 400
curl -s -X POST http://localhost:3000/api/seeds/ai-generate \
  -b 'bookflow_session=<token>' \
  -H 'Content-Type: application/json' \
  -d '{"track":"打脸逆袭"}' --max-time 120
```

返回 JSON：`{track, candidates: [{title, score:{title_ctr,conflict,tagfit,novelty}, why_buy, type, recommend_reason, heat}]}`

评分满分 40（4维×10）。≥30 greenlight, 22-29 backlog, <22 reject。

### 选赛道策略

根据用户 `strategy` jsonb 里的 `category_performance` + `proven_formula` + `banned_tracks` 来选赛道：
1. 优先选 CTR 高 + hot_tracks 里有大量爆款的赛道
2. 避开 banned_tracks
3. 可以交叉组合（如「暗恋」嫁接「打脸逆袭」）

### 立项（seed → project）

用户确认选题后，先入库再立项：
```bash
# 创建 seed（必须带 score 字段，否则 400）
curl -s -X POST http://localhost:3000/api/seeds \
  -b 'bookflow_session=<token>' \
  -H 'Content-Type: application/json' \
  -d '{"title":"<选中标题>","track":"<赛道>","score":{"title_ctr":9,"conflict":8,"tagfit":7,"novelty":7}}'
# 返回 seed with id, total_score, tier

# 从 seed 立项（POST /api/projects，body 带 seed_id）
curl -s -X POST http://localhost:3000/api/projects \
  -b 'bookflow_session=<token>' \
  -H 'Content-Type: application/json' \
  -d '{"seed_id":"<seed_uuid>"}'
# 返回 project with id, status=writing
```

⚠️ **注意**：路径不是 `/api/projects/from-seed/:id`，而是 `POST /api/projects` + body `{"seed_id":"..."}`。

### Seeds 相关 API 汇总

| 路径 | 方法 | 用途 |
|------|------|------|
| /api/seeds | GET | 列出当前用户所有 seeds |
| /api/seeds | POST | 创建 seed（body: {title, track}） |
| /api/seeds/:id | DELETE | 删除 seed |
| /api/seeds/ai-generate | POST | AI 批量生成选题（body: {track}） |
| /api/seeds/ai-score | POST | AI 评分单个选题 |
| /api/seeds/ai-recommend-track | POST | AI 推荐赛道（body: {primaries, plots}） |
| /api/seeds/ai-backfill-heat | POST | 回填热度 |
| /api/seeds/ai-drafts | GET | 列出 AI 生成的草稿历史 |

## 全流程步骤（按顺序执行）

所有 AI 步骤都是 SSE 流式接口，需要完整消费流。

### Step 0（新项目必须）: 生成大纲 + 逐章写正文

新建的项目没有内容，必须先生成大纲和正文，后续汇总/升华才有素材可用。

```
# 生成大纲（SSE 流式，会自动创建章节）
POST /api/projects/:id/ai-outline/stream
Accept: text/event-stream

# 获取章节列表
GET /api/projects/:id/chapters
# 返回 [{id, title, position, ...}]

# 逐章写正文（SSE 流式，每章 2-5 分钟）
POST /api/chapters/:chapter_id/ai-write/stream
Accept: text/event-stream
```

流程：大纲生成完 → 拿章节列表 → 循环调每章的 ai-write/stream → 全部写完后才能进入汇总。

⚠️ 10 章正文预计 20-50 分钟（取决于 AI 响应速度），建议写成脚本后台跑。

其他可选前置步骤（非必须但可提高质量）：
## 全流程步骤（按顺序执行）

所有 AI 步骤都是 SSE 流式接口，需要完整消费流。

⚠️ **前置依赖链（必须按顺序，跳步会 409）：**
README → Character Setup → Outline → 手动创建 Chapters → 逐章写正文 → 汇总/升华/配套/配图

### Step 0a: 生成 README（如项目刚从 seed 立项）

```
POST /api/projects/:id/ai-readme/stream
Accept: text/event-stream
```

- SSE 流式，生成包含 hook/主角/核心冲突的 README artifact
- 新建项目必须先跑这一步

### Step 0b: 角色设定

```
POST /api/projects/:id/ai-character-setup/stream
Accept: text/event-stream
```

- SSE 流式，依赖 README artifact
- 生成完整角色表、关系图、感情线、命名约束

### Step 0c: 生成大纲

```
POST /api/projects/:id/ai-outline/stream
Accept: text/event-stream
```

- SSE 流式，依赖 README + character_setup artifacts
- ⚠️ 大纲只存为 artifact，**不会自动创建 chapters**
- 需要从大纲内容解析章节标题（`### 第N章 标题`），然后手动建章

### Step 0d: 创建章节（手动）

```
POST /api/projects/:id/chapters
Content-Type: application/json
Body: {"title":"章节标题"}
```

- 从大纲 artifact 中 regex 提取：`re.findall(r'###\s+第(\d+)章\s+(.*)', outline_content)`
- 逐个 POST 创建，每次只需 title 字段

### Step 0e: 逐章写正文

```
POST /api/chapters/:chapter_id/ai-write/stream
Accept: text/event-stream
```

- SSE 流式，每章 2-3 分钟
- 依赖大纲 + 角色设定 artifact
## 全流程步骤（按顺序执行）

所有 AI 步骤都是 SSE 流式接口，需要完整消费流。

### Step 0: 前置准备（README + 角色设定 + 大纲 + 创建章节）

大纲依赖 README 和角色设定，必须先生成这两项：

```
POST /api/projects/:id/ai-readme/stream        → artifact: readme
POST /api/projects/:id/ai-character-setup/stream → artifact: character_setup
POST /api/projects/:id/ai-outline/stream        → artifact: outline
```

大纲只存为 artifact，**不会自动创建 chapters**。需要从大纲中解析章节标题，然后逐个创建：

```
POST /api/projects/:id/chapters  Body: {"title": "第1章标题"}
```

### Step 1: 写正文（整章一口气写，推荐）

**新接口（推荐）——整章一次性写完，替代逐 beat 分段：**

```
POST /api/chapters/:id/ai-write-full/stream
Accept: text/event-stream
Body: {}
```

- 一次性产出整章 1500-2000 字
- 自动读取大纲 excerpt + 角色设定（截断800字）+ 上一章结尾衔接
- 写完后自动保存到 chapter.body（SSE 流 done 事件触发保存）
- max_tokens=600，物理兜底不超标

**旧接口（保留，前端手动可用）——逐 beat 分段：**

```
对每一章:
  1. POST /api/chapters/:id/ai-beats → {"beats": [{"id":"b1","label":"...","note":"..."},...]}
  2. 对每个 beat:
     POST /api/chapters/:id/ai-write/stream
     Body: {"beat": {"id":"b1","label":"...","note":"..."}, "prev_tail": "上段尾部200字"}
  3. 拼接后: PUT /api/chapters/:id  Body: {"title": "章节标题", "body": "完整正文"}
```

### Step 1.5: QA 抽检（可选但推荐）

```
POST /api/chapters/:id/ai-qa
```

返回 JSON：total_score (5-50), verdict (pass/revise/reject), kill_reasons, quick_fix。
建议对第1章和中间章做 QA，≥30 分继续，<30 考虑重写。

### Step 2: 全书汇总
- **max_tokens 设为 1200**（物理截断兜底，prompt 约束不可靠）

#### 模式B：逐 beat 分段写（旧模式，前端仍在用）

```
对每一章:
  1. POST /api/chapters/:id/ai-beats → {"beats": [...]}
  2. 对每个 beat:
     POST /api/chapters/:id/ai-write/stream
     Body: {"beat": {...}, "prev_tail": "上段尾部200字"}
  3. PUT /api/chapters/:id  Body: {"title": "...", "body": "拼接正文"}
```

⚠️ 逐beat模式字数容易超标（每beat 1300-1800字，全章5000+）。优先用模式A。

### Step 1.5: 章级 QA（可选但推荐）

写完每章后可调 QA 检查：
```
POST /api/chapters/:id/ai-qa
Body: {}
→ 返回 publish_qa_check 的 JSON（total_score, verdict, kill_reasons 等）
```
- ≥ 35 → pass
- 30-34 → revise（建议修改）
- < 30 → reject（重写）

### Step 2: 全书汇总

旧流程（逐beat）会导致每章 5000+ 字碎片拼装感强，读者体验差。
新流程产出 1500-2000 字/章，一口气连贯写完。

```
对每一章:
  POST /api/chapters/:id/ai-write-full/stream
  Body: {}  (或 {"force": true} 强制覆盖已有正文)
  Accept: text/event-stream
```

内部逻辑：
- 自动读取 outline artifact 中本章描述
- 自动读取 character_setup（截断800字）
- 自动读取上一章最后300字做衔接
- prompt 在 `ai_prompts/chapter_write_full.system.md`

写完后立即做章级 QA：
```
POST /api/chapters/:id/ai-qa
→ 返回 publish_qa 格式的 JSON（total_score, verdict, kill_reasons 等）
```

⚠️ 旧接口 `ai-write/stream`（逐beat）仍然保留给前端手动用，但全流程脚本应用新接口。
⚠️ 如果调新接口返回 404，说明跑的是旧 binary，需要 `./dev.sh stop && cargo build && ./dev.sh start`。

### Step 1.5: 章级 QA（写完每章后）

```
POST /api/chapters/:id/ai-qa
```

- 返回 5 维评分（首句冲击力/留人率/节奏密度/AI味/标题匹配，各1-10）
- verdict: pass(≥35) / revise(30-34) / reject(<30)
- reject → 重写该章一次（最多重试1次）
- revise → 标记但继续

### Step 2: 全书汇总

```
POST /api/projects/:id/ai-book-summary/stream
Accept: text/event-stream
```

- 依赖 Step 1 的正文存在（至少1章有 body）
- 409 = "先生成至少一章正文，再汇总"

### Step 3: 优化升华

```
POST /api/projects/:id/ai-book-polish/stream
Accept: text/event-stream
```

- 依赖 Step 2 的 book_summary artifact

### Step 4: 配套素材

```
POST /api/projects/:id/ai-side-dishes/stream
Accept: text/event-stream
```

- 生成标签、简介、卖点等配套内容

### Step 5: 小说配图

```
POST /api/projects/:id/ai-story-image
Content-Type: application/json
Body: {"author_name":"<作者名>","show_author":true,"size":"2:3","quality":"high"}
```

- **非流式**，同步请求，但耗时较长（30s-120s）
- 返回 JSON 包含 artifact（content 是 JSON 字符串，内含 data_url base64 图片）
- 依赖 README artifact 存在

### Step 6: 打包 ZIP（模拟前端一键打包）

前端打包内容：
1. `{标题}-配套素材.txt` — side_dishes artifact 内容，markdown 转纯文本，带 BOM
2. `{标题}-优化升华.txt` — book_polished artifact 内容，markdown 转纯文本，带 BOM
3. `{标题}-{作者名}-3x4.png` — 封面图 3:4 裁剪

**3:4 裁剪逻辑（模拟前端 canvas）：**
- 原图 2:3（竖版），比 3:4 更窄
- 走「保留全宽，按 3:4 裁掉底部」分支
- `crop_h = min(src_h, round(src_w / (3/4)))`
- 水平居中 `sx = (src_w - crop_w) / 2`，顶部对齐 `sy = 0`

```python
from PIL import Image
from io import BytesIO
import base64

def crop_cover_3x4(data_url):
    b64 = data_url.split(",", 1)[1]
    img = Image.open(BytesIO(base64.b64decode(b64)))
    w, h = img.size
    ratio = 3 / 4
    if w / h > ratio:
        crop_h, crop_w = h, round(h * ratio)
    else:
        crop_w, crop_h = w, min(h, round(w / ratio))
    sx = round((w - crop_w) / 2)
    cropped = img.crop((sx, 0, sx + crop_w, crop_h))
    buf = BytesIO()
    cropped.save(buf, format='PNG')
    return buf.getvalue()
```

## 完整脚本模板（从 seed 立项到打包）

```python
#!/usr/bin/env python3
"""bookflow 全流程: 选题→立项→README→角色→大纲→建章→正文→汇总→升华→配套→配图→打包"""
import requests, json, sys, re, base64, zipfile, os, time
from PIL import Image
from io import BytesIO

BASE = "http://localhost:3000/api"
COOKIE = {"bookflow_session": "<SESSION_TOKEN>"}
AUTHOR_NAME = "<作者名>"

def consume_sse(url, timeout=600):
    resp = requests.post(url, json={}, cookies=COOKIE,
                         headers={"Accept": "text/event-stream"},
                         stream=True, timeout=timeout)
    if resp.status_code != 200:
        print(f"  ❌ {resp.status_code}: {resp.text[:300]}")
        return None
    full_text = ""
    for line in resp.iter_lines(decode_unicode=True):
        if not line: continue
        if line.startswith("event: error"): return None
        if line.startswith("event: done"): break
        if line.startswith("data: "):
            data = line[6:]
            try:
                full_text += json.loads(data).get("text", "")
            except json.JSONDecodeError:
                full_text += data
    return full_text

# === Phase 1: 选题 + 立项 ===
TRACK = "<赛道>"
print("选题生成...")
seeds_resp = requests.post(f"{BASE}/seeds/ai-generate", json={"track": TRACK}, cookies=COOKIE, timeout=120)
candidates = seeds_resp.json()["candidates"]
# 选最高分的
best = max(candidates, key=lambda c: sum(c["score"].values()))
print(f"  选中: {best['title']} (score={sum(best['score'].values())})")

# 创建 seed
seed = requests.post(f"{BASE}/seeds", json={"title": best["title"], "track": TRACK, "score": best["score"]}, cookies=COOKIE).json()
# 从 seed 立项
project = requests.post(f"{BASE}/projects", json={"seed_id": seed["id"]}, cookies=COOKIE).json()
PROJECT_ID = project["id"]
print(f"  立项: {PROJECT_ID}")

# === Phase 2: 内容生成链 ===
for step_name, path in [
    ("README", f"projects/{PROJECT_ID}/ai-readme/stream"),
    ("角色设定", f"projects/{PROJECT_ID}/ai-character-setup/stream"),
    ("大纲", f"projects/{PROJECT_ID}/ai-outline/stream"),
]:
    print(f"{step_name}...", end=" ", flush=True)
    r = consume_sse(f"{BASE}/{path}")
    if r: print(f"✅ ({len(r)} 字)")
    else: print("❌"); sys.exit(1)

# 从大纲解析章节并创建
time.sleep(1)
arts = requests.get(f"{BASE}/projects/{PROJECT_ID}/artifacts", cookies=COOKIE).json()
outline = next((a["content"] for a in arts if a["kind"] == "outline"), "")
chapter_titles = re.findall(r'###\s+第\d+章\s+(.*)', outline)
print(f"创建 {len(chapter_titles)} 章...")
for title in chapter_titles:
    requests.post(f"{BASE}/projects/{PROJECT_ID}/chapters", json={"title": title.strip()}, cookies=COOKIE)

# 逐章写正文
chapters = requests.get(f"{BASE}/projects/{PROJECT_ID}/chapters", cookies=COOKIE).json()
print(f"写正文 ({len(chapters)} 章)...")
for i, ch in enumerate(chapters, 1):
    print(f"  [{i}/{len(chapters)}] {ch.get('title','')}...", end=" ", flush=True)
    r = consume_sse(f"{BASE}/chapters/{ch['id']}/ai-write/stream", timeout=300)
    print(f"✅ ({len(r)} 字)" if r else "❌")

# === Phase 3: 汇总 + 打磨 ===
for step_name, path in [
    ("全书汇总", f"projects/{PROJECT_ID}/ai-book-summary/stream"),
    ("优化升华", f"projects/{PROJECT_ID}/ai-book-polish/stream"),
    ("配套素材", f"projects/{PROJECT_ID}/ai-side-dishes/stream"),
]:
    print(f"{step_name}...", end=" ", flush=True)
    r = consume_sse(f"{BASE}/{path}")
    print(f"✅ ({len(r)} 字)" if r else "❌")

# 配图
print("配图...", end=" ", flush=True)
resp = requests.post(f"{BASE}/projects/{PROJECT_ID}/ai-story-image",
    json={"author_name": AUTHOR_NAME, "show_author": True, "size": "2:3", "quality": "high"},
    cookies=COOKIE, timeout=300)
print("✅" if resp.status_code == 200 else f"⚠️ {resp.status_code}")

print("\n🎉 全流程完成！")
```

## 展示选题结果给用户的格式

生成选题后，以表格形式展示给用户确认再继续：
- 列出所有候选标题、总分、各维度分数、热度、AI点评
- 给出自己的判断（哪个最优、为什么、有什么风险）
- **等用户拍板**再立项跑后续流程，不要自动选

## Pitfalls

0. **Health 端点是 404** — bookflow 后端没有 /api/health，判断服务是否在跑用 `curl -sv http://localhost:3000/api/auth/login` 看是否有 HTTP 响应即可（哪怕 405 也说明在跑）。
## Pitfalls

0. **写正文需要 beat 参数** — `POST /api/chapters/:id/ai-write/stream` 必须带 `{"beat": {"id":"b1","label":"...","note":"..."}, "prev_tail": "..."}` body。空 body 发过去报 422（`missing field beat`）。正确流程：先 `POST /api/chapters/:id/ai-beats` 拿到 beats 数组，再逐个 beat 调 write/stream。
0b. **大纲不会自动建章节** — `ai-outline/stream` 只存 artifact，不创建 chapters 记录。必须从 outline artifact 解析章节标题后手动 `POST /api/projects/:id/chapters` 逐个创建。
## Pitfalls

0. **max_tokens 与中文字数换算** — Claude 中文 tokenizer 效率极高：1 token ≈ 2-3 中文字。设 max_tokens=4000 会产出 6000-8000 字，设 1200 产出 3700 字，设 600 产出 1700-2100 字。目标每章 1500-2000 字对应 max_tokens=600。永远通过 max_tokens 物理截断兜底，prompt 字数约束对 Claude 几乎无效。
1. **Docker 未启动** — `./dev.sh start` 会报 docker.sock 错误。先 `open -a Docker` 等就绪。
2. **后端卡死** — 认证接口超时但 health 通过 = 数据库锁。需要 `./dev.sh stop && ./dev.sh start`。
3. **配图 API 余额不足** — 多米异步接口报 400（"API账户余额不足或没有权限"），代码已加 fallback 到 OpenAI 同步接口（用 duomiapi_key 鉴权）。关键：fallback 用的是 `duomiapi_key` 而非主 `api_key`（主 key 是 Anthropic 的）。
4. **配图接口是异步的** — 多米路径：POST 提交→返回 task_id→轮询 query_task→下载图片。但如果第一步就失败（400），直接走 OpenAI 同步 `/v1/images/generations`（同一个 key 同一个 base_url）。
5. **Session 过期** — 401 时重新 login 获取新 token。
6. **SSE 超时** — 全书汇总（10章）可能需要 3-5 分钟，timeout 设 600s。
7. **Pillow 依赖** — 打包裁剪需要 `pip install Pillow`（系统 python3.9 已有）。
8. **文件名安全** — 中文标题直接用，但要去掉 `\/:*?"<>|` 等特殊字符。
9. **BOM 头** — 前端打包的 txt 带 `\uFEFF` BOM，Windows 记事本兼容性需要。
10. **dev.sh start 超时** — cargo build 增量编译慢时 dev.sh 可能超过 60s，分两步：先 `cd api && cargo build`（给 300s），再 `./dev.sh start`。
11. **max_tokens 与中文字数关系** — Claude 中文 tokenizer 效率极高：1 token ≈ 2-3 中文字。设 800 tokens 会产出 1300-2000 字，设 4000 tokens 会产出 5000-8000 字。**要控制番茄短篇字数（1500-2000字/章），整章写作 max_tokens 必须设 1200 左右**。不能依赖 prompt 约束，模型不遵守字数指令。
12. **整章写作 vs 逐beat写作** — 整章一口气写（ai-write-full/stream）产出连贯性远好于逐beat拼装。逐beat模式每段独立起承转合，拼起来碎片感强。番茄短篇推荐用整章模式。
13. **角色设定过长导致字数膨胀** — 5000字角色设定注入prompt后模型会"觉得需要展开写"。新版整章写作已截断为800字。
14. **大纲不自动建章** — ai-outline/stream 只保存 artifact，不创建 chapters 记录。必须手动从大纲提取标题后逐个 POST /api/projects/:id/chapters。
15. **章节标题含引号** — JSON body 里标题含双引号会导致创建失败。用 python json.dumps 构造 body 或手动转义。
15. **generate_image fallback 鉴权** — fallback 到 OpenAI 同步接口时必须用 `duomiapi_key` 而不是主 `api_key`。
16. **番茄 API 不需要 msToken/a_bogus** — 只靠 cookie 中的 sessionid 就能调通。
15. **generate_image fallback 鉴权** — fallback 到 OpenAI 同步接口时必须用 `duomiapi_key` 而不是主 `api_key`（主 key 是 Anthropic 的）。代码在 ai.rs generate_image() 中已修复：`fallback_cfg.api_key = cfg.duomiapi_key.clone()`。
16. **番茄 API 不需要 msToken/a_bogus** — 只靠 cookie 中的 sessionid 就能调通，简化自动化脚本。
17. **health 端点是 404** — 后端没有 /api/health 路由，但只要端口有响应即表示服务在跑。用 `curl -sv` 看 HTTP 状态判断。

## ⚠️⚠️ 数据层错乱三连坑（2026-07-08 血泪，全书废稿级）

跑全流程时中途**重跑设定步骤 / 删章节**会悄悄污染数据层，QA 分数完全反映不出来，最后整本废稿。三个坑必须一起防：

**坑A · 重跑 setup/大纲会生成新 artifact 版本，AI 可能换一套人名 → 全书角色分裂**
- artifact 是**版本化**的：重跑 `ai-readme/character-setup/outline` stream 不覆盖旧版，而是插一条**新版本**，`latest_all` 取最新。
- 致命后果：character_setup 重跑时 opus 会**自己另起一套人名**（本场第一次是「盛棠序/霍砚白/程荻宁」，重跑变成「柏棠/蒋渡/钱穗」）。`ai-write-full` 读的是 latest character_setup + latest outline，于是第1章用旧名、第2章起用新名，**主角全书改名换姓 = 废稿**。
- 铁律：**setup 步骤（readme/character/outline）确认好一版就不要为了"验证落库"或任何理由重跑**。真要重跑，重跑后必须核对人名与已写章节一致；发现分裂 → 删掉多余 artifact 版本，只留一版，且必须早于任何正文写作。

**坑B · 中途删 chapters 会让 idx 与大纲错位 → 每章内容整体移位一格**
- `ai-write-full` 靠 `chapter.idx` 去大纲 `extract_chapter_outline(outline, idx)` 抓「第N章」段落。idx 是建章顺序自增的。
- 如果建了空章、或删掉中间某章再补，**idx 与大纲「第N章」不再一一对应**，写出来的第4章其实是大纲第5章的内容（本场真实症状：正文开头 `# 第5章…` 出现在 idx=4 的章里，且出现重复的第10章）。
- 铁律：**建章一次到位、按大纲章序建齐 N 章，中途不删不插**。要清理空章/重复章，必须在**写正文之前**清完，清完立刻核对 `SELECT idx,title FROM chapters ORDER BY idx` 与大纲逐条对齐。

**坑C · 报 QA 分数≠稿子没问题——分数高但人名错/内容错位照样废**
- QA（`ai-qa`）只评单章文本质量（钩子/冲突/节奏/AI味），**不检测跨章人名一致性、不检测 idx 错位**。坑A/坑B 造成的错乱 QA 全绿也照过。
- 铁律：单章重写打转前，先**拉全章节全景核对底层数据**，别在表层反复重写：
```bash
docker exec bookflow-postgres psql -U bookflow -d bookflow_dev -tA -c \
  "SELECT idx, title, left(body,60), char_length(body) FROM chapters WHERE project_id='<PID>' ORDER BY idx;"
# 逐章看：idx 连不连续？title 对不对应大纲？正文开头人名/章号对不对？
```
发现数据层坏了（人名分裂/idx 错位/重复章/空章）→ **推倒重建**：删多余 artifact 版本只留一版 → 删光章节 → 用统一设定按 idx 1..N 干净重跑，别单章缝补。

## extra_notes 定向重写救 QA-reject 章（验证有效，2026-07-08）

QA reject（<30）的章别盲目重跑（同 prompt 再跑一次还是崩）。用写作端点的 `extra_notes` 字段喂**针对性重写要求**，一次救回（本场第9章 25→41、第10章 27→42 pass）：
```bash
curl -s -b "$CK" -X POST "http://localhost:3000/api/chapters/<cid>/ai-write-full/stream" \
  -H "Content-Type: application/json" \
  -d '{"extra_notes":"<针对 QA kill_reasons 的具体修改指令>"}'
```
番茄短篇高潮/结局章最常见的 reject 原因 + 对应 extra_notes 处方：
- **高潮章「太文艺/不够爽」**：删内心独白，加男主付出惨痛代价（事业崩/当众卑微）+ 女主强势碾压的名场面，增加外部冲突。
- **结局章「太开放/追妻力度不够」**：番茄读者要明确解气或甜宠收束，不要留白；男主要够卑微（如楼下守N天）、女主端着但给机会；去掉突兀的「N个月后」时间跳跃。
- 处方来源：先 `POST /api/chapters/:id/ai-qa` 拿 `kill_reasons`/`quick_fix`，照着写 extra_notes。重写后必须再 QA 确认 pass 才算数。

## 正文字数控制 + QA 质量关

详见 `references/word-count-qa-control.md` 和 `references/writing-mode-architecture.md`。
番茄爆款写作公式参考：`references/fanqie-explosive-formula.md`。

**核心数字**：番茄短篇每章 1700-2000 字，全书 ~18000 字。超过 2200 字/章 = 超标。

**QA 接口**：`POST /api/projects/:id/ai-publish-qa`（5 维评分，≥35/50 才能发布）。全流程应在写完正文后、汇总前调用 QA 做质量关。

**字数超标修复**：已在 chapter_write.system.md 和 ai.rs user prompt 中加入硬性字数约束（400-600 字/beat）。如果仍超标，根因多半是角色设定过长（被截断到 1500 字仍嫌多）或 beat note 过于详细。

## 选题评分系统 V2

当前评分被改为4维体系（title_ctr/conflict/tagfit/novelty，各1-10），强制对标 hot_tracks 真实爆款数据。
详见 `references/seed-scoring-v2.md`。

## 写作质量标准

详见 `references/writing-quality-standards.md`。
包含：字数标准、爆款范本分析、字数失控根因、整章写作方案、QA维度、读者画像。

API 路由完整参考见 `references/api-routes-cheatsheet.md`。

⚠️ **seeds 表 score 是独立列（非 jsonb）**：score_title, score_opening 等是 integer 列，不是一个 jsonb 字段。改评分维度需要写 migration 加新列。

## 热门数据更新（hot_tracks.md）

用户发送番茄 App 热门截图时，OCR 提取 → 结构化 → 写入 `hot_tracks.md`。
详见 `references/hot-tracks-update.md`。

关键：用 macOS Vision framework（swift）做中文 OCR，分段处理超长截图。

## 选题评分系统 V2（2026-06-30）

旧评分系统（8维×5分=40满分，AI自评全高分）已淘汰。新系统设计：
- **4维评分**：title_ctr / conflict / tagfit / novelty（各1-10，满分40）
- **立项阈值**：≥30 greenlight, 22-29 backlog, <22 reject
- **强制对标**：评分 prompt 注入 hot_tracks 数据，必须点名相似爆款做 benchmark
- **硬规则扣分**：标题超25字扣分、无数字/时间锚点扣分、跟爆款撞车 novelty 上限3
- **生成时分差规则**：5个候选中至少2个 total ≤25，不允许全 greenlight

详见 `.hermes/plans/seed-scoring-v2.md` 和 `references/seed-scoring-v2-design.md`。

改动文件清单：domain/lib.rs (Score结构体) + ai.rs (score_seed/generate_seeds/apply_hard_rules) + seed_scorer.system.md + seed_generator.system.md + 前端适配。

## 每周复盘流程

详见 `references/weekly-review-workflow.md`。
包含：数据抓取→录入fanqie_stats→更新users.strategy→更新hot_tracks.md→前端展示。

## 工作分工原则（用户强制要求）

- **咕咕嘎（Hermes）**: 需求设计、数据分析、复盘、沟通协调、运维操作、热门数据整理、数据库直接操作、脚本跑管线、hotfix（改一个数字/一行 prompt 级别的紧急修复）
- **Claude Code**: 所有功能代码改动（后端Rust + 前端React）。**用户明确要求：功能开发必须让 Claude Code 来做，Hermes 不要自己直接改代码**。
- 例外：ai.rs 的 max_tokens 调整、prompt 文件微调属于 hotfix，可以直接改。新增函数/路由/接口属于功能开发，交给 Claude Code。
- 派活方式: 
  1. 写需求文档到 `.hermes/plans/<feature>.md`
  2. tmux 启动 Claude Code: `tmux new-session -d -s bookflow-fix && tmux send-keys -t bookflow-fix "cd <project> && claude" Enter`
  3. 等它就绪后发任务: `tmux send-keys -t bookflow-fix "<prompt>" Enter`
  4. Claude Code 需要审批 shell 命令时: `tmux send-keys -t bookflow-fix Enter`（选 Yes）
  5. 选 "don't ask again for similar commands" 用 `tmux send-keys -t bookflow-fix Down Enter`
- 如果 Claude Code 连接失败（502），参考 `references/claude-code-gateway-fix.md`
- 如果 gateway 无法从 Hermes 内部重启，告知用户手动执行 `hermes gateway restart`

## 多用户批量跑

bookflow 多用户共用同一后端，每个用户有独立 session：
- 惜箬、川香鸡腿堡、花旁读经书、渡轮之上、骑驴写爽文闯番茄...
- 每个用户 login 获取各自 session，然后跑各自项目的全流程
- 可以写循环脚本，遍历用户列表依次处理
- **多用户邮箱格式不统一** — 需先查 DB：`SELECT email FROM users WHERE display_name = '花旁读经书'`
- 大部分用户密码是 `123456`（主账号除外）

## 选题评分注意事项（2026-06-30）

当前 AI 自评分数虚高（31-33/40 全 greenlight），**不能依赖评分做选题决策**。
在评分系统优化前，选题质量判断应该：
1. 手动对标 hot_tracks.md 里同赛道的真实爆款阅读量
2. 看标题是否符合已验证的爆款公式（假离婚、白月光对线、全家悔疯等）
3. 确认赛道是否在 hot_tracks 数据中有高阅读量支撑

## 账号复盘流程（周复盘）

每周对每个账号执行一次数据复盘：

1. **抓取数据** — 用番茄 cookie 调用 list + single_common API 获取所有作品最新数据
2. **录入数据库** — DELETE + INSERT 到 `fanqie_stats` 表（user_id 关联）
3. **更新策略** — 将复盘结论写入 `users.strategy` jsonb 字段
4. **同步到文件** — 更新 `my_track_analysis.md`（选题 prompt 自动注入）

关键 API（只需 cookie，不需要 msToken/a_bogus）：
- 列表: `GET /api/author/short_article/list/v0/?page_count=20&page_index=N`
- 单篇: `GET /api/author/sa_stats/single_common/v0/?book_id=X`
- 总览: `GET /api/author/sa_stats/common/v0/`

数据库结构：
- `fanqie_stats` 表: book_id, title, read_count, show_count, click_rate, digg_count, comment_count, shelf_count, douyin_pay_rate
- `users.strategy` jsonb: account_summary, proven_formula, banned_tracks, key_insights, next_actions, suggested_topics

参考: `references/claude-code-gateway-fix.md` — Claude Code 连接问题排查

## 周复盘流程（数据录入+分析）

每周执行一次，对每个账号:

### 1. 抓取最新数据

```bash
# 列表API（翻页抓全部）
GET /api/author/short_article/list/v0/?page_count=20&page_index={N}&status=0&time_sort=0&pack_type=1
# 逐篇详细数据
GET /api/author/sa_stats/single_common/v0/?book_id={book_id}
# 总体数据
GET /api/author/sa_stats/common/v0/
```

注意：番茄API只需cookie（sessionid），不需要msToken/a_bogus。

### 2. 录入数据库

```sql
-- 先删旧数据再插入（fanqie_stats 无 unique on book_id）
DELETE FROM fanqie_stats WHERE user_id = '<user_uuid>';
INSERT INTO fanqie_stats (user_id, book_id, title, read_count, show_count, click_rate, 
  digg_count, comment_count, shelf_count, douyin_pay_rate, recorded_at)
VALUES (...);
```

### 3. 复盘分析维度

- **爆款率**: 阅读>1万的篇数 / 总篇数（目标>15%）
- **CTR分布**: 高(>30%) / 中(15-30%) / 低(<5%) 各多少篇
- **赛道表现**: 哪些赛道贡献阅读，哪些全军覆没
- **标题公式**: 高CTR标题的共性模式
- **付费率**: 付费率>20%的作品特征
- **头部集中度**: TOP3占全部阅读的百分比

### 4. 输出建议

- 继续/加码的赛道
- 立即停止的赛道
- 下一批选题方向（具体标题参考）
- 标题公式模板
