# n8n 论坛日报工作流 — 设计文档

> 创建日期：2026-07-02

## 概述

每天自动抓取 n8n 官方论坛（community.n8n.io）的最新讨论、热门话题、招聘信息和旧帖新论，通过 Firecrawl 获取内容、DeepSeek LLM 生成智能摘要，最终存入 Notion Database 形成可检索的论坛日报知识库。

## 目标

- 每天 8:00 自动执行
- 抓取 10 条核心帖子（5 最新 + 5 热门，限近 7 天内创建）
- 额外抓取招聘帖、旧帖新论（不占 10 条名额）
- 逐条抓取帖子评论区完整内容
- DeepSeek 一次性生成所有帖子的讨论摘要
- 写入 Notion Database，每天一条记录

## 节点管线

```
Schedule Trigger (每天 8:00)
    │
    ▼
Firecrawl #1 — 抓取 https://community.n8n.io/latest
    │
    ▼
Code #1 — 智能分类提取
    │
    ├── 📌 最新帖 ×5（近 7 天内创建）
    ├── 🔥 热门帖 ×5（近 7 天内创建，与最新去重）
    ├── 🔄 旧帖新论 ×N（创建 >7 天但有今日新回复）
    └── 💼 招聘帖 ×N（招聘分类，不与其他重复）
    │
    ▼
Loop / SplitInBatches — 遍历所有帖子 URL
    │
    ├── Firecrawl #2 — 逐条抓取帖子详情页评论区
    │
    ▼
Code #2 — 合并所有评论为一段文本
    │
    ▼
DeepSeek LLM — 一次性生成分类摘要
    │
    ▼
Code #3 — 格式化为 Notion 富文本
    │
    ▼
Notion — 创建 Database 记录
```

## 节点详细说明

### 1. Schedule Trigger
- **类型**：`n8n-nodes-base.scheduleTrigger`
- **配置**：Cron `0 8 * * *`（每天 8:00）

### 2. Firecrawl #1（列表页抓取）
- **类型**：Firecrawl 节点
- **URL**：`https://community.n8n.io/latest`
- **输出**：页面 Markdown 内容

### 3. Code #1（分类提取）
- **类型**：`n8n-nodes-base.code`（JavaScript）
- **输入**：Firecrawl 返回的 Markdown
- **逻辑**：
  1. 解析 Markdown，提取所有帖子信息（标题、URL、分类标签、创建日期、回复数、浏览量）
  2. 判断帖子创建时间是否在近 7 天内
  3. 识别招聘分类标签的帖子 → `💼 招聘帖`
  4. 剩余帖子中，创建 >7 天的 → `🔄 旧帖新论`
  5. 近 7 天内帖子：按回复数/浏览量排序取前 5 为热门，其余取最新 5 条
  6. 各分类间去重
- **输出**：`{ posts: [{ title, url, category, replies, views, createdDate, group }] }`

### 4. Loop + Firecrawl #2（评论抓取）
- **循环遍历** Code #1 输出的所有帖子 URL
- **Firecrawl 配置**：逐条抓取帖子详情页
- **输出**：每条帖子的完整页面内容（含评论区）

### 5. Code #2（评论合并）
- **类型**：`n8n-nodes-base.code`（JavaScript）
- **逻辑**：将所有帖子的评论内容拼成一段文本，按分类分组标注
- **输出**：`{ combinedText: "📌最新\n1. 标题:xxx\n评论:xxx\n\n🔥热门\n..." }`

### 6. DeepSeek LLM（摘要生成）
- **类型**：AI/LLM 节点，使用 DeepSeek 凭据
- **Prompt 要点**：
  - 对每类帖子生成讨论摘要
  - 提取核心观点、争议点、解决方案
  - 保持简洁，每条帖子 2-4 句摘要
  - 招聘帖：提取职位、要求、联系方式
- **输出**：结构化摘要文本

### 7. Code #3（Notion 格式化）
- **类型**：`n8n-nodes-base.code`（JavaScript）
- **逻辑**：将 DeepSeek 输出转为 Notion 富文本 blocks 格式
- **输出**：`{ title, date, contentBlocks, postCount }`

### 8. Notion（写入）
- **类型**：`n8n-nodes-base.notion`
- **操作**：在指定 Database 中创建新页面

## Notion Database Schema

| 属性名 | 类型 | 说明 |
|--------|------|------|
| `Name` | Title | `n8n 论坛日报 - YYYY-MM-DD` |
| `Date` | Date | 日报日期 |
| `Content` | Rich Text | 分类摘要（含标题、链接、讨论要点） |
| `Post Count` | Number | 总帖子数 |

## 输出格式示例

```
# n8n 论坛日报 - 2026-07-02

## 📌 最新帖子
1. **[如何优化 n8n 工作流性能？](url)** — 讨论要点：建议使用 SplitInBatches 减少内存占用，注意 Code 节点的执行模式选择...
2. ...

## 🔥 热门讨论
1. **[HTTP Request vs Webhook 最佳实践](url)** — 核心观点：Webhook 适合实时触发，HTTP Request 适合轮询。争议点在于...

## 🔄 旧帖新论
1. **[去年发布的 n8n 部署指南](url)** — 今日新回复：用户反馈 Docker Compose 在 ARM 架构上的兼容性问题...

## 💼 招聘信息
1. **[招聘 n8n 自动化工程师](url)** — 远程，要求熟悉 n8n 和 API 集成，联系邮箱...
```

## API 调用量估算

| 服务 | 每日调用量 | 备注 |
|------|-----------|------|
| Firecrawl | 11+N 次 | 1 次列表页 + (10+N) 次详情页 |
| DeepSeek | 1 次 | 一次性批量分析 |

## 错误处理

- Firecrawl 抓取失败：记录日志，跳过该帖子继续
- DeepSeek 调用失败：使用 Code #2 的原始文本作为降级内容直接写入 Notion
- Notion 写入失败：n8n 自带重试机制
- 页面解析异常：Code #1 返回空列表时，当天不创建记录，避免空数据污染

## 约束与边界

- 每日最多抓取 20 条帖子（10 核心 + 最多 10 额外），防止 Firecrawl 超额
- DeepSeek 单次调用的 token 上限需注意，评论合并时做截断保护
- 招聘分类的识别依赖 Discourse 分类标签名称的稳定性
