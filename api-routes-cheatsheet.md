# Bookflow API Routes Cheatsheet

## Auth
```
POST /api/auth/login       Body: {"email":"...","password":"..."}  → Set-Cookie: bookflow_session=<token>
```

## Seeds (选题)
```
GET  /api/seeds                                    → 列出所有 seeds
POST /api/seeds                                    Body: {"title":"...","track":"...","score":{"title_ctr":N,"conflict":N,"tagfit":N,"novelty":N}}
DELETE /api/seeds/:id
POST /api/seeds/ai-generate                        Body: {"track":"打脸逆袭"}  → {"track":"...","candidates":[...]}
POST /api/seeds/ai-score                           Body: 待确认
POST /api/seeds/ai-recommend-track                 Body: {"primaries":["..."],"plots":["..."]}
POST /api/seeds/ai-backfill-heat
GET  /api/seeds/ai-drafts                          → 历史 AI 生成记录
```

## Projects
```
GET  /api/projects                                 → 列出项目 (可选 ?status=writing)
GET  /api/projects/:id                             → 项目详情
POST /api/projects                                 Body: {"seed_id":"<uuid>"}  → 从 seed 立项
DELETE /api/projects/:id
POST /api/projects/:id/transition                  Body: {"status":"ready"}
```

## Artifacts (AI 生成产物)
```
GET  /api/projects/:id/artifacts                   → 列出所有 artifacts
```
Artifact kinds: readme, character_setup, outline, book_summary, book_polished, side_dishes, story_image, publish_post

## AI Streams (SSE, Accept: text/event-stream)
```
POST /api/projects/:id/ai-readme/stream            → readme artifact
POST /api/projects/:id/ai-character-setup/stream   → character_setup artifact
POST /api/projects/:id/ai-outline/stream           → outline artifact (不自动建章!)
POST /api/projects/:id/ai-book-summary/stream      → book_summary (需要正文存在)
POST /api/projects/:id/ai-book-polish/stream       → book_polished (需要 book_summary)
POST /api/projects/:id/ai-side-dishes/stream       → side_dishes
POST /api/projects/:id/ai-blurb/stream
POST /api/projects/:id/ai-publish/stream           → 发布稿 (需要大纲+正文)
```

## AI Non-stream
```
POST /api/projects/:id/ai-story-image              Body: {"author_name":"...","show_author":true,"size":"2:3","quality":"high"}
POST /api/projects/:id/ai-publish-qa
```

## Chapters
```
GET  /api/projects/:id/chapters                    → 章节列表
POST /api/projects/:id/chapters                    Body: {"title":"章标题"}
PUT  /api/chapters/:id                             Body: {"title":"...","body":"正文内容"}
```

## Chapter AI (写正文的两步)
```
POST /api/chapters/:id/ai-beats                    Body: {}  → {"beats":[{"id":"b1","label":"...","note":"..."},...]}
POST /api/chapters/:id/ai-write/stream             Body: {"beat":{"id":"b1","label":"...","note":"..."},"prev_tail":"上段尾200字"}
POST /api/chapters/:id/ai-write                    (非流式版，同样需要 beat)
```

## 依赖链
```
ai-outline        需要 readme + character_setup 存在
ai-book-summary   需要至少1章有 body
ai-book-polish    需要 book_summary 存在
ai-publish        需要 outline + 正文节选
```

## Seeds API 评分结构 (Score V2)
```json
{"title_ctr": 1-10, "conflict": 1-10, "tagfit": 1-10, "novelty": 1-10}
```
total_score = sum of 4 fields (max 40)
Tier: ≥30 greenlight, 22-29 backlog, <22 reject
