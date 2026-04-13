知道了。以下是 RunnableAssign、RunnablePick 和 RunnableItemgetter 的完整教程。

---

4.2 LCEL 链式语法 - 11 RunnableAssign（状态赋值）

核心概念

`RunnableAssign` 在链执行过程中动态添加或修改字段，同时保留原有数据。

```
输入：{ a: 1, b: 2 }
      ↓ RunnableAssign({ c: (x) => x.a + x.b })
输出：{ a: 1, b: 2, c: 3 }  // 原数据保留，新增 c
```

---

基础用法

```javascript
import { RunnableAssign } from "@langchain/core/runnables";

// 创建赋值器
const addFields = new RunnableAssign({
  mapping: {
    // 计算新字段
    fullName: (input) => `${input.firstName} ${input.lastName}`,

    // 异步计算
    userProfile: async (input) => {
      return await fetchUserProfile(input.userId);
    },

    // 固定值
    processedAt: () => new Date().toISOString()
  }
});

// 使用
const result = await addFields.invoke({
  firstName: "张",
  lastName: "三",
  userId: "U001"
});

console.log(result);
// {
//   firstName: "张",
//   lastName: "三",
//   userId: "U001",
//   fullName: "张三",
//   userProfile: { ... },
//   processedAt: "2026-04-13T18:26:00.000Z"
// }
```

---

简写形式：RunnablePassthrough.assign()

```javascript
import { RunnablePassthrough } from "@langchain/core/runnables";

// 更简洁的写法（推荐）
const chain = RunnableSequence.from([
  // 步骤1：基础处理
  (input) => ({ text: input.trim() }),

  // 步骤2：动态添加字段
  RunnablePassthrough.assign({
    wordCount: (input) => input.text.split(/\s+/).length,
    sentiment: (input) => analyzeSentiment(input.text),
    hash: (input) => generateHash(input.text)
  }),

  // 步骤3：使用新增字段
  (input) => `文本"${input.text}"有${input.wordCount}词，情感${input.sentiment}`
]);
```

---

实战：构建完整上下文

```javascript
import { RunnableSequence, RunnablePassthrough } from "@langchain/core/runnables";

const contextBuilder = RunnableSequence.from([
  // 输入：原始查询
  RunnablePassthrough.assign({
    // 添加时间上下文
    currentTime: () => new Date().toLocaleString('zh-CN'),
    currentDay: () => ['日','一','二','三','四','五','六'][new Date().getDay()]
  }),

  RunnablePassthrough.assign({
    // 添加用户上下文（基于 userId）
    userContext: async (input) => {
      return await getUserContext(input.userId);
    }
  }),

  RunnablePassthrough.assign({
    // 添加历史摘要（基于 userId）
    historySummary: async (input) => {
      return await summarizeHistory(input.userId, 5); // 最近5轮
    }
  }),

  RunnablePassthrough.assign({
    // 添加检索结果（基于查询）
    retrievedDocs: async (input) => {
      return await vectorStore.similaritySearch(input.query, 3);
    }
  }),

  // 最终组装提示
  (fullContext) => ({
    messages: buildMessages(fullContext),
    metadata: {
      userId: fullContext.userId,
      retrievedCount: fullContext.retrievedDocs.length
    }
  })
]);

// 使用
const result = await contextBuilder.invoke({
  query: "怎么退款？",
  userId: "U123"
});
```

---

4.2 LCEL 链式语法 - 12 RunnablePick（字段选择）

核心概念

`RunnablePick` 从对象中提取指定字段，过滤不需要的数据。

```
输入：{ a: 1, b: 2, c: 3, d: 4 }
      ↓ RunnablePick(["a", "c"])
输出：{ a: 1, c: 3 }
```

---

基础用法

```javascript
import { RunnablePick } from "@langchain/core/runnables";

// 创建选择器
const pickFields = new RunnablePick({
  keys: ["content", "metadata"]  // 只保留这两个字段
});

// 使用
const result = await pickFields.invoke({
  content: "重要内容",
  metadata: { source: "doc1", page: 5 },
  internalId: "abc123",      // 被过滤
  _debugInfo: {...}          // 被过滤
});

console.log(result);
// { content: "重要内容", metadata: { source: "doc1", page: 5 } }
```

---

简写形式

```javascript
import { RunnableSequence } from "@langchain/core/runnables";

const chain = RunnableSequence.from([
  // 复杂处理，产生大量中间字段
  complexProcessor,

  // 只输出需要的字段给用户
  (input) => ({
    answer: input.generatedContent,
    sources: input.retrievedDocs.map(d => d.metadata.source),
    confidence: input.confidenceScore
  })

  // 或使用 RunnablePick（如果是过滤已有字段）
  // new RunnablePick({ keys: ["answer", "sources", "confidence"] })
]);
```

---

4.2 LCEL 链式语法 - 13 RunnableItemgetter（嵌套提取）

核心概念

`RunnableItemgetter` 从嵌套结构中提取特定路径的数据。

```
输入：{
  data: {
    results: [{ content: "A" }, { content: "B" }]
  }
}
      ↓ RunnableItemgetter(["data", "results", 0, "content"])
输出："A"
```

---

基础用法

```javascript
import { RunnableItemgetter } from "@langchain/core/runnables";

// 提取嵌套字段
const getFirstResult = new RunnableItemgetter({
  keys: ["response", "data", "items", 0, "text"]
});

// 使用
const result = await getFirstResult.invoke({
  response: {
    data: {
      items: [
        { text: "第一条结果", score: 0.95 },
        { text: "第二条结果", score: 0.87 }
      ],
      total: 100
    },
    status: "ok"
  }
});

console.log(result);  // "第一条结果"
```

---

简写形式：箭头函数

```javascript
// 更灵活的简写（推荐）
const chain = RunnableSequence.from([
  // 复杂 API 调用
  callExternalApi,

  // 提取嵌套结果
  (apiResponse) => apiResponse.data.results[0].content,

  // 继续处理
  processContent
]);
```

---

实战：API 响应处理管道

```javascript
import {
  RunnableSequence,
  RunnableAssign,
  RunnablePick
} from "@langchain/core/runnables";

const apiPipeline = RunnableSequence.from([
  // 步骤1：调用 API
  async (params) => {
    const response = await fetch('/api/search', {
      method: 'POST',
      body: JSON.stringify(params)
    });
    return await response.json();
  },

  // 步骤2：提取并转换数据
  (apiResult) => ({
    // 使用 Itemgetter 逻辑提取嵌套数据
    results: apiResult.data.hits.map(hit => ({
      title: hit._source.title,
      content: hit._source.content,
      score: hit._score
    })),
    total: apiResult.data.total.value,
    took: apiResult.took
  }),

  // 步骤3：添加处理元数据
  RunnablePassthrough.assign({
    processedAt: () => new Date().toISOString(),
    resultCount: (input) => input.results.length
  }),

  // 步骤4：过滤输出字段
  (input) => ({
    items: input.results,
    meta: {
      total: input.total,
      returned: input.resultCount,
      responseTime: input.took,
      timestamp: input.processedAt
    }
  })
]);

// 使用
const searchResult = await apiPipeline.invoke({
  query: "人工智能",
  size: 10
});
```

---

组合实战：完整数据处理流

```javascript
import {
  RunnableSequence,
  RunnableParallel,
  RunnablePassthrough,
  RunnablePick
} from "@langchain/core/runnables";

// 智能文档处理系统
const documentProcessor = RunnableSequence.from([
  // 步骤1：并行提取多种信息
  RunnableParallel.from({
    raw: RunnablePassthrough(),

    extracted: (doc) => ({
      title: extractTitle(doc),
      author: extractAuthor(doc),
      date: extractDate(doc)
    }),

    analyzed: (doc) => ({
      sentiment: analyzeSentiment(doc.content),
      keywords: extractKeywords(doc.content),
      summary: summarize(doc.content)
    })
  }),

  // 步骤2：合并结果
  (parallelResults) => ({
    ...parallelResults.raw,
    metadata: parallelResults.extracted,
    analysis: parallelResults.analyzed
  }),

  // 步骤3：添加派生字段
  RunnablePassthrough.assign({
    searchIndex: (doc) => createSearchIndex(doc),
    qualityScore: (doc) => calculateQuality(doc)
  }),

  // 步骤4：格式化输出（只选必要字段）
  (doc) => ({
    id: doc.id,
    title: doc.metadata.title,
    summary: doc.analysis.summary,
    keywords: doc.analysis.keywords,
    score: doc.qualityScore,
    indexed: !!doc.searchIndex
  })
]);

// 使用
const processed = await documentProcessor.invoke({
  id: "doc-001",
  content: "这是一篇关于人工智能的长文档...",
  source: "upload"
});
```

---

## 对比总结

| 组件 | 功能 | 输入 | 输出 | 使用场景 |
|------|------|------|------|----------|
| `RunnableAssign` | 添加字段 | `{a:1}` | `{a:1, b:2}` | 扩展上下文 |
| `RunnablePick` | 选择字段 | `{a:1, b:2, c:3}` | `{a:1, c:3}` | 过滤输出 |
| `RunnableItemgetter` | 嵌套提取 | `{data:{items:[...]}}` | 深层值 | API响应解析 |

---

## 学习检查清单

| 检查项 | 状态 |
|--------|------|
| 掌握 `RunnablePassthrough.assign()` 动态添加字段 | [ ] |
| 掌握字段选择和嵌套数据提取 | [ ] |
| 能组合 Assign + Pick 构建数据处理管道 | [ ] |
| 理解简写形式（箭头函数）与显式类的选择 | [ ] |

---
知道了。以下是 RunnableConfigurable 和 RunnableGenerator 的完整教程。

---

4.2 LCEL 链式语法 - 14 RunnableConfigurable（动态配置）

核心概念

`RunnableConfigurable` 允许在运行时动态切换配置，实现同一链的不同行为模式。

```
同一条链，不同配置：
- 开发模式：详细日志、本地模型
- 生产模式：精简输出、云端模型
- 调试模式：中间结果、延迟模拟
```

---

基础用法

```javascript
import { RunnableConfigurable } from "@langchain/core/runnables";
import { ChatOpenAI } from "@langchain/openai";

// 创建可配置链
const configurableChain = new RunnableConfigurable({
  // 默认配置
  default: {
    model: new ChatOpenAI({ modelName: "gpt-4o-mini", temperature: 0.7 }),
    verbose: false,
    maxRetries: 2
  },

  // 预设配置方案
  alternatives: {
    // 高质量模式
    highQuality: {
      model: new ChatOpenAI({ modelName: "gpt-4o", temperature: 0.2 }),
      verbose: true,
      maxRetries: 5
    },

    // 快速模式
    fast: {
      model: new ChatOpenAI({ modelName: "gpt-4o-mini", temperature: 0.9 }),
      verbose: false,
      maxRetries: 1
    },

    // 本地模式
    local: {
      model: new ChatOpenAI({
        modelName: "qwen-7b",
        configuration: { baseURL: "http://localhost:8000/v1" }
      }),
      verbose: true,
      maxRetries: 0
    }
  }
});

// 使用默认配置
const result1 = await configurableChain.invoke("你好");

// 切换到高质量模式
const result2 = await configurableChain.withConfig("highQuality").invoke("复杂问题");

// 切换到快速模式
const result3 = await configurableChain.withConfig("fast").invoke("简单问题");
```

---

简写形式：.bind() + 运行时配置

```javascript
import { RunnableSequence } from "@langchain/core/runnables";

// 更灵活的方式：运行时通过 config 参数切换
const baseChain = RunnableSequence.from([
  (input) => input,
  model,
  parser
]);

// 执行时指定配置
const result = await baseChain.invoke(
  { question: "你好" },
  {
    configurable: {
      modelName: "gpt-4o",      // 动态切换模型
      temperature: 0.2,          // 动态调整参数
      userId: "U123",            // 传递上下文
      sessionId: "S456"
    },
    callbacks: [customHandler],  // 动态添加回调
    tags: ["production"],        // 动态标签
    metadata: { region: "CN" }   // 动态元数据
  }
);
```

---

实战：多租户配置隔离

```javascript
import { RunnableSequence, RunnableConfigurable } from "@langchain/core/runnables";

// 为不同租户预设配置
const tenantAwareChain = new RunnableConfigurable({
  default: {
    model: "gpt-4o-mini",
    rateLimit: 100,      // 默认 100 RPM
    dataRegion: "US"
  },

  alternatives: {
    enterprise: {
      model: "gpt-4o",
      rateLimit: 1000,   // 企业 1000 RPM
      dataRegion: "EU",  // 欧盟数据驻留
      auditLog: true
    },

    startup: {
      model: "gpt-4o-mini",
      rateLimit: 50,     // 初创 50 RPM
      dataRegion: "US",
      costCap: 100       // 月度成本上限
    }
  }
});

// 根据租户ID动态选择配置
async function processForTenant(tenantId, input) {
  const tenant = await getTenantConfig(tenantId);
  const configName = tenant.plan; // 'enterprise' | 'startup' | 'default'

  return await tenantAwareChain
    .withConfig(configName)
    .invoke(input, {
      configurable: { tenantId, userId: tenant.adminId }
    });
}
```

---

4.2 LCEL 链式语法 - 15 RunnableGenerator（流式生成器）

核心概念

`RunnableGenerator` 将异步生成器函数封装为 Runnable，实现自定义流式处理。

```
普通函数：返回单个结果
生成器函数：yield 多个中间结果（流式）
```

---

基础用法

```javascript
import { RunnableGenerator } from "@langchain/core/runnables";

// 创建生成器 Runnable
const tokenStream = new RunnableGenerator({
  // 异步生成器函数
  func: async function* (input) {
    const text = input.text;
    const words = text.split(" ");

    for (const word of words) {
      // 模拟处理每个词
      await delay(100);

      // yield 中间结果
      yield {
        word: word,
        processed: word.toUpperCase(),
        progress: `${words.indexOf(word) + 1}/${words.length}`
      };
    }

    // 最终汇总
    yield {
      complete: true,
      totalWords: words.length,
      allWords: words.map(w => w.toUpperCase())
    };
  }
});

// 消费流
for await (const chunk of await tokenStream.stream({ text: "hello world test" })) {
  console.log(chunk);
}
// { word: "hello", processed: "HELLO", progress: "1/3" }
// { word: "world", processed: "WORLD", progress: "2/3" }
// { word: "test", processed: "TEST", progress: "3/3" }
// { complete: true, totalWords: 3, allWords: ["HELLO", "WORLD", "TEST"] }
```

---

简写形式：直接 async function

```javascript
import { RunnableSequence } from "@langchain/core/runnables";

// 直接在链中使用生成器
const streamingChain = RunnableSequence.from([
  // 步骤1：格式化
  (input) => ({ prompt: `回答：${input.question}` }),

  // 步骤2：流式生成（自定义逻辑）
  async function* (input) {
    const model = new ChatOpenAI({ streaming: true });
    const stream = await model.stream(input.prompt);

    let fullText = "";

    for await (const chunk of stream) {
      fullText += chunk.content;

      // 实时 yield 进度
      yield {
        type: "token",
        content: chunk.content,
        accumulated: fullText
      };
    }

    // 最终 yield 完整结果
    yield {
      type: "complete",
      fullText: fullText,
      wordCount: fullText.split(/\s+/).length
    };
  }
]);

// 消费
for await (const event of await streamingChain.stream({ question: "你好" })) {
  if (event.type === "token") {
    process.stdout.write(event.content);
  } else if (event.type === "complete") {
    console.log(`\n\n完成！共 ${event.wordCount} 词`);
  }
}
```

---

实战：SSE 服务器推送

```javascript
import { RunnableGenerator } from "@langchain/core/runnables";
import { ChatOpenAI } from "@langchain/openai";

// 构建 SSE 流生成器
const sseGenerator = new RunnableGenerator({
  func: async function* (input) {
    const { prompt, sessionId } = input;

    // 发送开始事件
    yield {
      event: "start",
      data: JSON.stringify({ sessionId, timestamp: Date.now() })
    };

    // 调用模型流
    const model = new ChatOpenAI({ streaming: true });
    const stream = await model.stream(prompt);

    let buffer = "";

    for await (const chunk of stream) {
      buffer += chunk.content;

      // 按句子分割发送（更友好的体验）
      const sentences = buffer.match(/[^。！？.!?]+[。！？.!?]+/g) || [];

      if (sentences.length > 0) {
        buffer = buffer.slice(sentences.join("").length);

        for (const sentence of sentences) {
          yield {
            event: "message",
            data: JSON.stringify({
              content: sentence,
              sessionId
            })
          };
        }
      }
    }

    // 发送剩余内容
    if (buffer.trim()) {
      yield {
        event: "message",
        data: JSON.stringify({ content: buffer, sessionId })
      };
    }

    // 发送结束事件
    yield {
      event: "end",
      data: JSON.stringify({
        sessionId,
        completed: true,
        timestamp: Date.now()
      })
    };
  }
});

// Express 中使用
app.post("/chat", async (req, res) => {
  res.setHeader("Content-Type", "text/event-stream");
  res.setHeader("Cache-Control", "no-cache");

  const stream = await sseGenerator.stream({
    prompt: req.body.message,
    sessionId: req.body.sessionId
  });

  for await (const event of stream) {
    res.write(`event: ${event.event}\n`);
    res.write(`data: ${event.data}\n\n`);
  }

  res.end();
});
```

---

组合实战：完整流式 RAG 系统

```javascript
import {
  RunnableSequence,
  RunnableGenerator,
  RunnablePassthrough
} from "@langchain/core/runnables";

const streamingRagChain = RunnableSequence.from([
  // 步骤1：检索（非流式）
  async (input) => {
    const docs = await retriever.getRelevantDocuments(input.question);
    return { ...input, context: formatDocs(docs) };
  },

  // 步骤2：构建提示
  (input) => ({
    prompt: `基于以下上下文回答：\n${input.context}\n\n问题：${input.question}`
  }),

  // 步骤3：流式生成（自定义生成器）
  async function* (input) {
    // 发送检索完成事件
    yield { stage: "retrieval", status: "complete", docCount: input.context.length };

    // 流式生成
    const stream = await model.stream(input.prompt);
    let fullAnswer = "";

    for await (const chunk of stream) {
      fullAnswer += chunk.content;
      yield { stage: "generation", token: chunk.content, partial: fullAnswer };
    }

    // 发送完成事件
    yield {
      stage: "complete",
      answer: fullAnswer,
      sources: extractSources(input.context)
    };
  }
]);

// 前端消费
const events = await streamingRagChain.stream({ question: "什么是RAG？" });

for await (const event of events) {
  switch (event.stage) {
    case "retrieval":
      showStatus(`检索完成，找到 ${event.docCount} 篇文档`);
      break;
    case "generation":
      appendText(event.token);  // 实时显示
      break;
    case "complete":
      showSources(event.sources);  // 显示来源
      break;
  }
}
```

---

## 对比总结

| 组件 | 核心能力 | 使用场景 |
|------|----------|----------|
| `RunnableConfigurable` | 运行时切换配置 | 多环境、多租户 |
| `RunnableGenerator` | 自定义流式输出 | SSE、实时进度、分阶段处理 |

---

## 学习检查清单

| 检查项 | 状态 |
|--------|------|
| 掌握 `withConfig()` 动态切换预设配置 | [ ] |
| 掌握运行时 `config` 参数传递 | [ ] |
| 能用 `async function*` 创建自定义流 | [ ] |
| 能组合生成器实现 SSE 服务器推送 | [ ] |
| 能构建分阶段的流式 RAG 系统 | [ ] |

---
