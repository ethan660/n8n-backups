# n8n Forum Daily Digest — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build an n8n workflow that scrapes community.n8n.io daily via Firecrawl, generates LLM summaries via DeepSeek, and writes to a Notion Database.

**Architecture:** Schedule Trigger → Firecrawl listing scrape → Code classification → Loop/Firecrawl detail scrape → Code merge comments → DeepSeek LLM summary → Code format → Notion write. Built via n8n MCP tools using the n8n Workflow SDK.

**Tech Stack:** n8n Workflow SDK (TypeScript), n8n MCP tools, Notion MCP, Firecrawl node, DeepSeek Chat Model node

## Global Constraints

- Do not hardcode secrets — use n8n credentials
- Code nodes must not issue HTTP requests — use Firecrawl/HTTP Request nodes
- jsonBody uses UTF-8 characters, not `\u{}` escapes
- AI summaries must be grounded in real scraped data, never fabricated
- Only scrape public pages from community.n8n.io, respect ToS
- Do not modify unrelated workflows or instance configuration
- Max 20 posts per day to prevent Firecrawl overage
- Schedule trigger: Cron `0 8 * * *`

---

### Task 1: Discover Required Nodes and Credentials

**Files:** None (exploration only)

**Interfaces:**
- Produces: node type IDs with discriminators, credential IDs, project ID

- [ ] **Step 1: Search for all required nodes**

```
search_nodes with queries: ["schedule trigger", "firecrawl", "code", "notion", "split in batches", "deepseek", "ai", "llm"]
```
Expected: Returns node type IDs with discriminators (resource/operation/mode) for Schedule Trigger, Firecrawl, Code, Notion, SplitInBatches, and DeepSeek/AI LLM nodes.

- [ ] **Step 2: List credentials for Firecrawl, Notion, and DeepSeek**

```
list_credentials with queries for "firecrawl", "notion", "deepseek"
```
Expected: Returns credential IDs for Firecrawl API, Notion integration, and DeepSeek API.

- [ ] **Step 3: Search for the project**

```
search_projects with query matching the user's target project
```
Expected: Returns project ID where the workflow will be created. If no specific project, use personal project.

---

### Task 2: Get SDK Reference and Best Practices

**Files:** None (reference only)

**Interfaces:**
- Produces: SDK patterns, expression syntax, design guidelines for workflow code

- [ ] **Step 1: Get SDK reference**

```
get_sdk_reference with sections: "patterns", "expressions", "guidelines", "design"
```
Expected: Returns complete SDK syntax for `workflow()`, `trigger()`, `node()`, `.add()`, `.to()`, `expr()`, and credential patterns.

- [ ] **Step 2: Get workflow best practices for relevant techniques**

```
get_workflow_best_practices with technique: "scheduling"
get_workflow_best_practices with technique: "data_persistence"
get_workflow_best_practices with technique: "content_generation"
```
Expected: Returns recommended nodes, patterns, and common pitfalls for each technique.

---

### Task 3: Get Node Type Definitions

**Files:** None (reference only)

**Interfaces:**
- Consumes: Node type IDs and discriminators from Task 1
- Produces: Exact parameter names and structures for all nodes

- [ ] **Step 1: Get type definitions for all nodes**

```
get_node_types with nodeIds containing all nodes from Task 1, including discriminators:
- schedule trigger (with schedule operation)
- firecrawl
- code
- notion (with databasePage operation)
- split in batches
- deepseek chat model
- basic llm chain or ai agent (for LLM summarization)
```
Expected: Returns full TypeScript parameter definitions with required/optional fields, displayOptions, and resource locator methods.

---

### Task 4: Create Notion Database

**Files:** None (Notion MCP)

**Interfaces:**
- Produces: Notion Database ID and data source ID for the daily digest database

- [ ] **Step 1: Create the Notion Database**

```
notion-create-database with:
  title: "n8n 论坛日报"
  schema: CREATE TABLE ("Name" TITLE, "Date" DATE, "Content" RICH_TEXT, "Post Count" NUMBER)
```
Expected: Returns Database ID and data source ID. The `Name` property will hold "n8n 论坛日报 - YYYY-MM-DD", `Date` holds the report date, `Content` holds the rich-text summary, `Post Count` holds the total number of posts.

- [ ] **Step 2: Verify database creation**

```
notion-fetch with the database URL
```
Expected: Returns database schema confirming all 4 properties exist with correct types.

---

### Task 5: Build Workflow Code — Part 1 (Trigger + Data Sources)

**Files:**
- Create: workflow code (in-memory before create_workflow_from_code)

**Interfaces:**
- Produces: Validated workflow code for Schedule Trigger, Firecrawl listing scrape, and Code #1 classification node

- [ ] **Step 1: Write the Schedule Trigger node**

Using SDK patterns from Task 2:
```typescript
const schedule = workflow().addNode({
  name: 'Schedule Trigger',
  type: 'n8n-nodes-base.scheduleTrigger',
  typeVersion: 1,
  position: [250, 300],
  parameters: {
    rule: {
      interval: [{ field: 'cronExpression', expression: '0 8 * * *' }]
    }
  }
});
```

- [ ] **Step 2: Write the Firecrawl listing scrape node**

Using Firecrawl type definition from Task 3:
```typescript
const firecrawlList = schedule.addNode({
  name: 'Firecrawl List',
  type: 'n8n-nodes-base.firecrawl',
  typeVersion: 1,
  position: [450, 300],
  parameters: {
    resource: 'scrape',
    url: 'https://community.n8n.io/latest',
    formats: ['markdown'],
    options: {}
  }
}, { credential: { id: '<firecrawl-credential-id>', name: 'firecrawlApi' } });
```

- [ ] **Step 3: Write Code #1 classification node**

```typescript
const code1 = firecrawlList.addNode({
  name: 'Code Classify',
  type: 'n8n-nodes-base.code',
  typeVersion: 2,
  position: [650, 300],
  parameters: {
    jsCode: `
const markdown = $input.first().json.markdown || $input.first().json.data?.markdown || '';
const sevenDaysAgo = new Date();
sevenDaysAgo.setDate(sevenDaysAgo.getDate() - 7);

// Extract posts from markdown - Discourse /latest page format
// Each topic is typically: ## [Title](url) with category/replies/views metadata
const topicRegex = /###?\\s+\\[([^\\]]+)\\]\\(([^)]+)\\)[\\s\\S]*?(?=###?\\s+\\[|$)/g;
const posts = [];
let match;
while ((match = topicRegex.exec(markdown)) !== null) {
  const title = match[1].trim();
  const url = match[2].trim();
  const block = match[0];
  
  // Extract metadata: category, replies, views, created date
  const categoryMatch = block.match(/(?:category|分类)[:\\s]*([^\\n,]+)/i);
  const repliesMatch = block.match(/(\\d+)\\s*(?:replies|回复|reply)/i);
  const viewsMatch = block.match(/(\\d+)\\s*(?:views|浏览|view)/i);
  const dateMatch = block.match(/(\\d{4}-\\d{2}-\\d{2}|\\w{3}\\s+\\d{1,2})/);
  
  const category = categoryMatch ? categoryMatch[1].trim() : '';
  const replies = repliesMatch ? parseInt(repliesMatch[1]) : 0;
  const views = viewsMatch ? parseInt(viewsMatch[1]) : 0;
  
  let createdDate = null;
  if (dateMatch) {
    createdDate = new Date(dateMatch[1]);
    if (isNaN(createdDate.getTime())) createdDate = null;
  }
  
  const isJobs = /job|招聘|hire|hiring/i.test(category);
  const isRecent = createdDate && createdDate >= sevenDaysAgo;
  
  let group = '';
  if (isJobs) {
    group = 'jobs';
  } else if (!isRecent && createdDate) {
    group = 'revived';
  } else if (!isRecent && !createdDate) {
    group = 'revived'; // unknown date, treat as old
  } else {
    group = 'normal'; // will be split into latest/hot later
  }
  
  posts.push({ json: { title, url, category, replies, views, createdDate: createdDate ? createdDate.toISOString().split('T')[0] : 'unknown', group } });
}

// Split normal posts into latest and hot
const normalPosts = posts.filter(p => p.json.group === 'normal');
const hotPosts = [...normalPosts].sort((a, b) => b.json.replies - a.json.replies).slice(0, 5);
const hotUrls = new Set(hotPosts.map(p => p.json.url));
const latestPosts = normalPosts.filter(p => !hotUrls.has(p.json.url)).slice(0, 5);

// Assign final groups
const result = [];
for (const p of hotPosts) { p.json.group = 'hot'; result.push(p); }
for (const p of latestPosts) { p.json.group = 'latest'; result.push(p); }
for (const p of posts) {
  if (p.json.group === 'revived' || p.json.group === 'jobs') result.push(p);
}

return result.slice(0, 20);
`
  }
});
```

- [ ] **Step 4: Validate this partial pipeline**

```
validate_workflow with code containing Schedule Trigger → Firecrawl List → Code Classify
```
Expected: No validation errors. Fix any issues before proceeding.

---

### Task 6: Build Workflow Code — Part 2 (Loop + Detail Scrape + Merge)

**Files:**
- Modify: workflow code (add to Part 1)

**Interfaces:**
- Consumes: Code #1 output `{ posts: [{title, url, category, group}] }`
- Produces: Loop with Firecrawl detail scrape, Code #2 merge node

- [ ] **Step 1: Write SplitInBatches loop node**

```typescript
const split = code1.addNode({
  name: 'Loop Posts',
  type: 'n8n-nodes-base.splitInBatches',
  typeVersion: 3,
  position: [850, 300],
  parameters: {
    batchSize: 1,
    options: {}
  }
});
```

- [ ] **Step 2: Write Firecrawl detail scrape inside loop**

```typescript
const firecrawlDetail = split.addNode({
  name: 'Firecrawl Detail',
  type: 'n8n-nodes-base.firecrawl',
  typeVersion: 1,
  position: [1050, 300],
  parameters: {
    resource: 'scrape',
    url: '={{ $json.url }}',
    formats: ['markdown'],
    options: {}
  }
}, { credential: { id: '<firecrawl-credential-id>', name: 'firecrawlApi' } });
```

Note: connect to split's output 1 (loop output).

- [ ] **Step 3: Write Code #2 merge node (after loop aggregation)**

```typescript
const code2 = split.addNode({
  name: 'Code Merge',
  type: 'n8n-nodes-base.code',
  typeVersion: 2,
  position: [1250, 300],
  parameters: {
    jsCode: `
const items = $input.all();
const groups = { latest: [], hot: [], revived: [], jobs: [] };
const groupLabels = {
  latest: '📌 最新帖子',
  hot: '🔥 热门讨论',
  revived: '🔄 旧帖新论',
  jobs: '💼 招聘信息'
};

for (const item of items) {
  const data = item.json;
  const group = data.group || 'latest';
  const title = data.title || 'Untitled';
  const url = data.url || '';
  const markdown = data.markdown || data.data?.markdown || '';
  
  // Extract comments section from the detail page markdown
  // Discourse detail pages have post content followed by replies
  const comments = markdown.replace(/^#[^\\n]*\\n/gm, '').trim();
  
  if (groups[group]) {
    groups[group].push({ title, url, comments: comments.substring(0, 3000) });
  }
}

// Build combined text for LLM
let combinedText = '';
for (const [group, posts] of Object.entries(groups)) {
  if (posts.length === 0) continue;
  combinedText += '\\n## ' + groupLabels[group] + '\\n\\n';
  posts.forEach((p, i) => {
    combinedText += (i + 1) + '. **' + p.title + '**\\n';
    combinedText += '   URL: ' + p.url + '\\n';
    combinedText += '   讨论内容:\\n' + p.comments.split('\\n').map(l => '   ' + l).join('\\n') + '\\n\\n';
  });
}

return [{
  json: {
    combinedText: combinedText.substring(0, 50000),
    postCount: items.length,
    groups: Object.fromEntries(Object.entries(groups).map(([k, v]) => [k, v.length]))
  }
}];
`
  }
});
```

Note: connect to split's output 0 (done/aggregate output).

- [ ] **Step 5: Validate this extended pipeline**

```
validate_workflow with code containing all nodes so far
```
Expected: No validation errors.

---

### Task 7: Build Workflow Code — Part 3 (DeepSeek LLM + Format + Notion Write)

**Files:**
- Modify: workflow code (add to Part 2)

**Interfaces:**
- Consumes: Code #2 output `{ combinedText, postCount, groups }`
- Produces: DeepSeek LLM, Code #3 format, Notion write nodes

- [ ] **Step 1: Write DeepSeek LLM chain node**

```typescript
const llm = code2.addNode({
  name: 'DeepSeek Summary',
  type: '@n8n/n8n-nodes-langchain.chainLlm',
  typeVersion: 1,
  position: [1450, 300],
  parameters: {
    prompt: `你是一个论坛日报编辑。请根据以下论坛帖子的讨论内容，生成一份中文日报摘要。

要求：
1. 按四个板块输出：📌 最新帖子、🔥 热门讨论、🔄 旧帖新论、💼 招聘信息
2. 每个帖子用 2-4 句话概括讨论要点（核心观点、争议点、解决方案）
3. 招聘帖子：提取职位名称、要求、联系方式
4. 保持 Markdown 格式，标题和链接可点击
5. 如果某个板块没有内容，标注「今日无」

格式示例：
## 📌 最新帖子
1. **[帖子标题](URL)** — 讨论要点：...

## 🔥 热门讨论
1. **[帖子标题](URL)** — 核心观点：...

以下是论坛帖子内容：

{{ $json.combinedText }}`
  }
}, { credential: { id: '<deepseek-credential-id>', name: 'deepseekApi' } });
```

- [ ] **Step 2: Validate DeepSeek node config**

```
validate_node_config with the DeepSeek chainLlm node config
```
Expected: No validation errors. If the node type or credential key is wrong, adjust based on error messages.

- [ ] **Step 3: Write Code #3 format node**

```typescript
const code3 = llm.addNode({
  name: 'Code Format',
  type: 'n8n-nodes-base.code',
  typeVersion: 2,
  position: [1650, 300],
  parameters: {
    jsCode: `
const llmOutput = $input.first().json.text || $input.first().json.output || '';
const today = new Date().toISOString().split('T')[0];
const title = 'n8n 论坛日报 - ' + today;

// Get post count from upstream Code Merge
const postCount = $('Code Merge').first().json.postCount || 0;

return [{
  json: {
    title: title,
    date: today,
    summary: llmOutput,
    postCount: postCount
  }
}];
`
  }
});
```

- [ ] **Step 4: Write Notion create page node**

```typescript
const notion = code3.addNode({
  name: 'Notion Write',
  type: 'n8n-nodes-base.notion',
  typeVersion: 1,
  position: [1850, 300],
  parameters: {
    resource: 'databasePage',
    operation: 'create',
    databaseId: '<notion-database-id>',
    properties: {
      title: '={{ $json.title }}',
      date: '={{ $json.date }}',
      content: '={{ $json.summary }}',
      postCount: '={{ $json.postCount }}'
    }
  }
}, { credential: { id: '<notion-credential-id>', name: 'notionApi' } });
```

- [ ] **Step 5: Validate the full workflow**

```
validate_workflow with the complete code
```
Expected: No validation errors. Fix any issues.

---

### Task 8: Test Workflow with Pin Data

**Files:** None (testing only)

**Interfaces:**
- Consumes: Complete validated workflow ID
- Produces: Test results confirming pipeline correctness

- [ ] **Step 1: Create the workflow in n8n**

```
create_workflow_from_code with the validated code, description: "Daily n8n forum digest: scrapes community.n8n.io, generates DeepSeek summaries, writes to Notion"
```
Expected: Returns workflow ID.

- [ ] **Step 2: Prepare test pin data**

```
prepare_test_pin_data with the workflow ID
```
Expected: Returns JSON schemas for nodes that need pin data (Schedule Trigger, Firecrawl nodes, Notion).

- [ ] **Step 3: Generate realistic pin data for the listing page**

For Firecrawl List node, provide simulated markdown matching Discourse /latest format:
```json
{
  "Firecrawl List": [{
    "json": {
      "markdown": "### [How to optimize n8n workflow performance?](https://community.n8n.io/t/123)\ncategory: Questions | 15 replies | 200 views | Jul 1\n\n### [Best practices for error handling](https://community.n8n.io/t/456)\ncategory: Tutorials | 42 replies | 500 views | Jun 28\n\n### [Hiring: n8n Automation Engineer](https://community.n8n.io/t/789)\ncategory: Jobs | 5 replies | 100 views | Jul 1\n\n### [Deploying n8n on AWS - 2025 guide](https://community.n8n.io/t/101)\ncategory: Guides | 3 replies | 50 views | Jan 15"
    }
  }],
  "Firecrawl Detail": [{
    "json": {
      "markdown": "# How to optimize n8n workflow performance?\n\nI'm running into performance issues with large datasets. Has anyone found good optimization strategies?\n\nReply 1: Try using SplitInBatches to reduce memory usage.\nReply 2: Also check your Code node execution mode - per-item is faster for simple transforms.\nReply 3: I had the same issue and switching to PostgreSQL helped a lot."
    }
  }]
}
```

- [ ] **Step 4: Run test with pin data**

```
test_workflow with the workflow ID and pin data
```
Expected: Execution completes, check that Code Classify correctly categorizes posts, Code Merge produces combined text, and pipeline reaches Notion node.

- [ ] **Step 5: Verify test execution output**

```
get_execution with includeData: true, check each node's output
```
Expected: Code Classify shows correct groups (hot/latest/revived/jobs), Code Merge shows combinedText, DeepSeek shows summary text, Code Format shows title/date/summary.

---

### Task 9: Finalize and Publish

**Files:** None

**Interfaces:**
- Consumes: Tested workflow ID
- Produces: Published (active) workflow

- [ ] **Step 1: Run end-to-end test with real Firecrawl scrape**

```
execute_workflow with executionMode: "manual" (no pin data)
```
Expected: Workflow executes with real Firecrawl and DeepSeek calls, Notion record created.

- [ ] **Step 2: Verify Notion record**

```
notion-fetch with the database URL, then notion-query-data-sources to check the new record
```
Expected: New record with correct title "n8n 论坛日报 - YYYY-MM-DD", Date set, Content filled with summary text in 4 sections, Post Count > 0.

- [ ] **Step 3: Publish the workflow**

```
publish_workflow with the workflow ID
```
Expected: Workflow activated, will run daily at 8:00.

---

### Task 10: Add Error Handling (Post-Publish Refinement)

**Files:**
- Modify: workflow (via update_workflow)

**Interfaces:**
- Consumes: Published workflow ID
- Produces: Workflow with error handling configured

- [ ] **Step 1: Set continueOnFail for Firecrawl Detail node**

```
update_workflow with setNodeSettings: Firecrawl Detail → onError: "continueErrorOutput"
```
This ensures one failed post scrape doesn't block the entire Loop.

- [ ] **Step 2: Add error branch for DeepSeek fallback**

If DeepSeek fails, the workflow should write raw comments to Notion instead. Add an If node after DeepSeek to check for errors, and a fallback path that writes Code Merge output directly.

- [ ] **Step 3: Verify error handling**

Test by temporarily using an invalid DeepSeek credential to confirm the fallback path works.
