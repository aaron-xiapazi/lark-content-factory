# 飞书内容工厂 (Lark Content Factory)

> AI Agent 用飞书 CLI 跑完「选题 → 写稿 → 审稿 → 排版 → 发布」全流程。一条命令出一篇文章。

## 这是什么？

一个基于 [飞书 CLI](https://github.com/larksuite/cli) 的 Skill，让 AI Agent 帮你完成内容生产的全部环节：

```
你：帮我选几个今天的选题
AI：✅ 已从微博热搜、知乎、36kr 抓取热点，筛选出 3 个选题写入多维表格。

你：第二个不错，帮我写稿
AI：✅ 已完成深度研究和写稿，文章已创建为飞书文档。

你：可以，润色一下发公众号
AI：✅ 文章已润色、排版并发布到公众号草稿箱。
```

**不是 demo，是每天在跑的真实生产系统。**

## 架构

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

**深度使用飞书三大能力：**
- 📊 **多维表格** — 选题库管理、状态流转、数据追踪
- 📝 **云文档** — 稿件创建、编辑、审阅
- 💬 **即时通讯** — 进度通知、审核提醒

## 5 个子命令

| 命令 | 功能 | 飞书能力 |
|------|------|----------|
| `+topic` | 抓取热点 → AI筛选 → 写入多维表格 | base (多维表格) |
| `+draft` | 读选题 → 深度研究 → AI写稿 → 创建文档 | base + docs |
| `+review` | 读文档 → AI润色去AI味 → 更新文档 | docs |
| `+format` | 飞书文档 → 公众号HTML排版 | docs |
| `+publish` | 发布到公众号草稿箱 → 更新状态 → 通知 | base + docs + im |
| `+pipeline` | 一键全流程：选题到发布 | 全部 |

## 快速开始

### 1. 安装飞书 CLI

```bash
npm install -g @larksuite/cli
lark-cli config init
```

### 2. 安装 Skill

将 `SKILL.md` 放入你的 Claude Code skills 目录：

```bash
# 方式一：克隆仓库
git clone https://github.com/aaron-xiapazi/lark-content-factory.git
cp -r lark-content-factory ~/.claude/skills/

# 方式二：直接下载
mkdir -p ~/.claude/skills/lark-content-factory
curl -o ~/.claude/skills/lark-content-factory/SKILL.md \
  https://raw.githubusercontent.com/aaron-xiapazi/lark-content-factory/main/SKILL.md
```

### 3. 初始化选题库

首次使用时，Skill 会自动创建飞书多维表格作为选题库，包含以下字段：

| 字段 | 说明 |
|------|------|
| 标题 | 选题标题 |
| 角度 | 切入角度 / hook |
| 来源 | 热点来源 |
| 热度 | 1-10 评分 |
| 状态 | 待选→已选→写稿中→待审→已审→已发布 |
| 文档链接 | 飞书文档 URL |
| 发布链接 | 公众号文章 URL |

### 4. 开始使用

```
/content-factory    # 或直接说"帮我写篇文章"
```

## 工作流详解

### 选题 (+topic)

1. 从微博热搜、知乎、36kr、Hacker News 等渠道抓取热点
2. AI 根据你的内容定位筛选 3-5 个最佳选题
3. 写入飞书多维表格，附带 AI 分析和热度评分
4. 飞书消息通知你查看

### 写稿 (+draft)

1. 从多维表格读取你选中的选题
2. 围绕选题进行深度研究，补充数据和案例
3. AI 生成文章（短句轰炸 + 观点鲜明 + 论据充足）
4. 创建飞书云文档，自动更新选题状态

### 审稿 (+review)

1. 读取飞书文档内容
2. AI 润色 + 去 AI 味（去掉"值得注意的是"等套话，加入口语化表达）
3. 更新飞书文档
4. 更新选题状态为「已审」

### 排版 + 发布 (+format → +publish)

1. 飞书文档 → 公众号 HTML 排版
2. 自动生成封面图
3. 发布到公众号草稿箱
4. 更新多维表格状态，飞书消息通知

## 所需权限

| scope | 用途 |
|-------|------|
| `bitable:app` | 读写多维表格 |
| `docx:document` | 创建/编辑云文档 |
| `drive:drive` | 云空间操作 |
| `im:message:send_as_bot` | 发送通知消息 |

## 为什么做这个？

作为一个每天都要产出内容的创作者，我发现内容生产流程中有大量重复工作：

- 每天花 30 分钟刷热点找选题
- 写完稿子还要手动复制到公众号后台排版
- 状态管理靠脑子记，经常忘了哪篇到哪个阶段

现在这些全交给 AI Agent，我只需要在两个关键节点做决策：**选哪个题** 和 **定不定稿**。

## License

MIT
