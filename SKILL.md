---
name: lark-content-factory
version: 1.0.0
description: "飞书内容工厂：用飞书 CLI 驱动从选题到发布的内容生产全链路。当用户需要选题、写稿、审稿、排版、发布公众号文章，或一键跑完内容生产全流程时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 飞书内容工厂 (Lark Content Factory)

AI Agent 用飞书 CLI 跑完「选题 → 写稿 → 审稿 → 排版 → 发布」全流程。一条命令出一篇文章。

> **前置条件：** 需要安装 [飞书 CLI](https://github.com/larksuite/cli) 并完成 `lark-cli config init` 配置。

触发方式：`/content-factory`、"内容工厂"、"帮我写篇文章"、"今天选题"、"发公众号"

---

## 架构概览

```
飞书多维表格（选题库）          飞书云文档（稿件）           公众号草稿箱
    ┌──────────┐              ┌──────────┐              ┌──────────┐
    │  +topic  │──选题确认──▶ │  +draft  │──审核通过──▶ │ +publish │
    │ 热点抓取  │              │ AI写稿    │              │ 排版+发布 │
    │ AI筛选   │              │ 创建文档   │              │          │
    │ 写入表格  │              │ 更新状态   │              │          │
    └──────────┘              └──────────┘              └──────────┘
         │                         │                         │
         └─────────── +pipeline: 一键全流程 ─────────────────┘
```

**飞书能力深度绑定：**
- **多维表格 (base)** — 选题库管理、状态流转、数据追踪
- **云文档 (docs)** — 稿件创建、编辑、审阅
- **即时通讯 (im)** — 进度通知、审核提醒

---

## 配置：选题库多维表格

首次使用需创建选题库多维表格。表结构如下：

| 字段名 | 类型 | 说明 |
|--------|------|------|
| 标题 | 文本 | 选题标题 |
| 角度 | 文本 | 切入角度 / hook |
| 来源 | 文本 | 热点来源（微博/知乎/36kr等）|
| 热度 | 数字 | 热度评分（1-10）|
| 状态 | 单选 | 待选 / 已选 / 写稿中 / 待审 / 已审 / 已发布 |
| 文档链接 | URL | 飞书文档链接 |
| 发布链接 | URL | 公众号文章链接 |
| 创建时间 | 日期 | 自动 |
| 备注 | 文本 | AI 分析 / 人工批注 |

### 创建选题库

```bash
# 创建多维表格
lark-cli base +base-create --as user --name "内容工厂-选题库"

# 记录返回的 app_token，后续所有操作需要

# 获取默认表格
lark-cli base +table-list --as user --app-token <app_token>

# 创建字段（在默认表格上）
lark-cli base +field-create --as user --app-token <app_token> --table-id <table_id> \
  --name "角度" --type text
lark-cli base +field-create --as user --app-token <app_token> --table-id <table_id> \
  --name "来源" --type text
lark-cli base +field-create --as user --app-token <app_token> --table-id <table_id> \
  --name "热度" --type number
lark-cli base +field-create --as user --app-token <app_token> --table-id <table_id> \
  --name "状态" --type singleSelect \
  --data '{"property":{"options":[{"name":"待选"},{"name":"已选"},{"name":"写稿中"},{"name":"待审"},{"name":"已审"},{"name":"已发布"}]}}'
lark-cli base +field-create --as user --app-token <app_token> --table-id <table_id> \
  --name "文档链接" --type url
lark-cli base +field-create --as user --app-token <app_token> --table-id <table_id> \
  --name "发布链接" --type url
lark-cli base +field-create --as user --app-token <app_token> --table-id <table_id> \
  --name "备注" --type text
```

---

## 命令

### +topic：选题

从全网热点中筛选适合的选题，写入飞书多维表格。

**流程：**

1. **抓取热点** — 从微博热搜、知乎、36kr、Hacker News 等渠道获取当日热点
2. **AI 筛选** — 根据用户身份和内容定位，从热点中筛选 3-5 个最佳选题
3. **写入多维表格** — 每个选题作为一条记录写入选题库
4. **通知用户** — 发飞书消息告知今日推荐选题

```bash
# Step 1-2: 抓取热点 + AI 筛选（Agent 内完成）

# Step 3: 写入多维表格
lark-cli base +record-create --as user --app-token <app_token> --table-id <table_id> \
  --data '{"fields":{"标题":"<title>","角度":"<angle>","来源":"<source>","热度":<score>,"状态":"待选","备注":"<ai_analysis>"}}'

# Step 4: 通知用户
lark-cli im +messages-send --as bot --user-id <user_id> \
  --content "📋 今日选题推荐已更新，共 N 个选题，请在选题库中查看并选择"
```

**选题标准：**
- 冲击力：普通人看了会震撼吗？
- 故事性：能讲成一个故事吗？有人物、情节、转折？
- 相关性：跟目标读者直接相关吗？
- 信息密度：有足够数据和案例支撑吗？
- 时效性：是近两天的热点吗？

---

### +draft：写稿

从选题库中取选题，AI 生成文章，创建飞书云文档。

**流程：**

1. **读取选题** — 从多维表格读取状态为「已选」的记录
2. **更新状态** — 改为「写稿中」
3. **深度研究** — 围绕选题搜索补充资料、数据、案例
4. **AI 写稿** — 生成文章 markdown
5. **创建飞书文档** — 写入飞书云文档
6. **回填链接** — 更新选题状态为「待审」，回填文档链接

```bash
# Step 1: 读取「已选」的选题
lark-cli base +record-list --as user --app-token <app_token> --table-id <table_id> \
  --filter '{"conjunction":"and","conditions":[{"field_name":"状态","operator":"is","value":["已选"]}]}'

# Step 2: 更新状态为「写稿中」
lark-cli base +record-update --as user --app-token <app_token> --table-id <table_id> \
  --record-id <record_id> --data '{"fields":{"状态":"写稿中"}}'

# Step 3-4: 深度研究 + AI 写稿（Agent 内完成）

# Step 5: 创建飞书文档
lark-cli docs +create --as user --title "<article_title>" --markdown "<article_content>"
# 记录返回的 doc URL

# Step 6: 更新状态 + 回填文档链接
lark-cli base +record-update --as user --app-token <app_token> --table-id <table_id> \
  --record-id <record_id> \
  --data '{"fields":{"状态":"待审","文档链接":"<doc_url>"}}'
```

**写作标准：**
- 一篇文章一个主题，用故事串到底
- 短句轰炸，有节奏感，不要长篇大论
- 观点鲜明，站队明确，不两边讨好
- 论据充足，数据和案例撑住观点
- 说人话，砍掉英文术语和技术行话

---

### +review：审稿

读取飞书文档，AI 润色去 AI 味，更新文档。

**流程：**

1. **读取文档** — 获取飞书云文档内容
2. **AI 审稿** — 检查逻辑、润色文字、去 AI 味
3. **更新文档** — 将修改后的内容写回飞书文档
4. **更新状态** — 选题状态改为「已审」

```bash
# Step 1: 读取文档
lark-cli docs +fetch --as user --doc "<doc_url>"

# Step 2: AI 审稿润色（Agent 内完成）
# 去 AI 味标准：
# - 去掉"值得注意的是"、"总的来说"、"综上所述"等 AI 套话
# - 去掉过度整齐的排比句
# - 加入口语化表达、不完美的句式
# - 保留信息量，只改表达方式

# Step 3: 更新文档
lark-cli docs +update --as user --doc "<doc_url>" --mode overwrite --markdown "<revised_content>"

# Step 4: 更新状态
lark-cli base +record-update --as user --app-token <app_token> --table-id <table_id> \
  --record-id <record_id> --data '{"fields":{"状态":"已审"}}'
```

---

### +format：排版

将飞书文档转换为公众号 HTML 格式。

```bash
# Step 1: 读取已审文档
lark-cli docs +fetch --as user --doc "<doc_url>"

# Step 2: Markdown → 公众号 HTML
# 使用 baoyu-markdown-to-html skill 或内置转换逻辑
# 输出：带微信样式的 HTML 文件
```

---

### +publish：发布

将排版后的文章发布到公众号草稿箱。

```bash
# Step 1: 执行 +format 获取 HTML

# Step 2: 生成封面图

# Step 3: 发布到公众号草稿箱
# 使用 baoyu-post-to-wechat skill 或直接调用微信 API

# Step 4: 更新选题状态 + 回填发布链接
lark-cli base +record-update --as user --app-token <app_token> --table-id <table_id> \
  --record-id <record_id> \
  --data '{"fields":{"状态":"已发布","发布链接":"<publish_url>"}}'

# Step 5: 通知用户
lark-cli im +messages-send --as bot --user-id <user_id> \
  --content "✅ 文章已发布到公众号草稿箱：<title>"
```

---

### +pipeline：全流程

串联所有步骤，从选题到发布。

```
+topic → 用户选择 → +draft → +review → 用户确认 → +format → +publish
```

**两个人工确认节点：**
1. 选题后：等用户在多维表格中将状态改为「已选」
2. 审稿后：等用户确认定稿

---

## 权限

| 操作 | 所需 scope |
|------|-----------|
| 多维表格读写 | `bitable:app` |
| 云文档创建/编辑 | `docx:document`, `drive:drive` |
| 搜索文档 | `docs:doc:search_read` |
| 发送消息 | `im:message:send_as_bot` |

---

## 示例用法

```
用户: 帮我选几个今天的选题
Agent: [执行 +topic] 已从微博热搜、知乎、36kr 抓取热点，筛选出 3 个选题写入选题库。
       请在飞书多维表格中查看并将你想写的选题状态改为「已选」。

用户: 第二个选题不错，帮我写稿
Agent: [执行 +draft] 已完成深度研究和写稿，文章已创建为飞书文档：
       https://xxx.larkoffice.com/docx/xxx
       请审阅，确认后我帮你润色发布。

用户: 可以，帮我润色一下然后发公众号
Agent: [执行 +review → +format → +publish]
       ✅ 文章已润色、排版并发布到公众号草稿箱。
```
