# 4.2 LCEL 链式语法 - RunnableLambda 与 RunnableMap

---

## 目录

1. [RunnableLambda（自定义函数封装）](#1-runnablelambda自定义函数封装)
   - [核心概念](#11-核心概念)
   - [基础用法](#12-基础用法)
   - [与链组合](#13-与链组合)
   - [异步函数与错误处理](#14-异步函数与错误处理)
   - [简写形式](#15-简写形式推荐)
2. [RunnableMap（批量映射处理）](#2-runnablemap批量映射处理)
   - [核心概念](#21-核心概念)
   - [基础用法](#22-基础用法)
   - [与链组合](#23-与链组合批量翻译示例)
   - [实战：文档批量处理管道](#24-实战文档批量处理管道)
3. [batch vs RunnableMap 对比](#3-batch-vs-runnablemap-对比)
4. [完整实战：智能客服批量回复](#4-完整实战智能客服批量回复)
5. [学习检查清单](#5-学习检查清单)

---

## 1. RunnableLambda（自定义函数封装）

### 1.1 核心概念

`RunnableLambda` 将普通 JavaScript 函数封装为 LCEL 兼容的 Runnable 组件，可在链中使用。

| 类型 | 说明 |
|------|------|
| 普通函数 | `(input) => output` |
| RunnableLambda | 可 `.invoke()`、`.batch()`、`.stream()` 的函数组件 |

---

### 1.2 基础用法

```javascript
import { RunnableLambda } from "@langchain/core/runnables";

// 封装普通函数
const addTimestamp = new RunnableLambda({
  func: (input) => ({
    ...input,
    timestamp: new Date().toISOString()
  })
});

// 作为链组件使用
const result = await addTimestamp.invoke({ data: "test" });
console.log(result);
// { data: "test", timestamp: "2026-04-13T18:18:00.000Z" }
```

---

### 1.3 与链组合

```javascript
import { RunnableSequence } from "@langchain/core/runnables";

const chain = RunnableSequence.from([
  // 步骤1：格式化输入
  (input) => ({ text: input.trim().toLowerCase() }),

  // 步骤2：自定义处理（RunnableLambda）
  new RunnableLambda({
    func: async (input) => {
      // 复杂异步逻辑
      const wordCount = input.text.split(/\s+/).length;
      const charCount = input.text.length;

      return {
        ...input,
        stats: { wordCount, charCount }
      };
    }
  }),

  // 步骤3：生成报告
  (input) => `文本"${input.text}"包含${input.stats.wordCount}个单词，${input.stats.charCount}个字符`
]);

const result = await chain.invoke("  Hello World  ");
console.log(result);
// "文本"hello world"包含2个单词，11个字符"
```

---

### 1.4 异步函数与错误处理

```javascript
const safeApiCall = new RunnableLambda({
  func: async (input) => {
    try {
      const response = await fetch(input.url);
      return await response.json();
    } catch (error) {
      return { error: error.message, fallback: true };
    }
  },

  // 配置
  name: "api_fetcher",
  tags: ["external", "api"]
});

// 批量执行（自动并发）
const results = await safeApiCall.batch([
  { url: "https://api.example.com/data1" },
  { url: "https://api.example.com/data2" },
  { url: "invalid-url" }  // 错误会被捕获
]);
```

---

### 1.5 简写形式（推荐）

```javascript
import { RunnableSequence } from "@langchain/core/runnables";

// 直接传入函数，自动包装为 RunnableLambda
const chain = RunnableSequence.from([
  (input) => input * 2,           // 自动包装
  (input) => input + 1,           // 自动包装
  async (input) => {              // 异步自动支持
    await delay(100);
    return input * 10;
  }
]);

const result = await chain.invoke(5);  // (5*2+1)*10 = 110
```

---

## 2. RunnableMap（批量映射处理）

### 2.1 核心概念

`RunnableMap` 对输入数组的每个元素应用同一个处理逻辑，返回结果数组。

```
输入：[item1, item2, item3]
      ↓ RunnableMap(runnable)
输出：[result1, result2, result3]
```

---

### 2.2 基础用法

```javascript
import { RunnableMap } from "@langchain/core/runnables";
import { ChatOpenAI } from "@langchain/openai";

const model = new ChatOpenAI({ modelName: "gpt-4o-mini" });

// 创建映射处理器
const summarizer = new RunnableMap({
  bound: PromptTemplate
    .fromTemplate("一句话总结：{text}")
    .pipe(model)
});

// 批量处理多个文本
const texts = [
  "人工智能是计算机科学的一个分支...",
  "区块链是一种分布式账本技术...",
  "量子计算利用量子力学原理..."
];

const summaries = await summarizer.invoke(texts);
console.log(summaries);
// [AIMessage, AIMessage, AIMessage]
```

---

### 2.3 与链组合：批量翻译示例

```javascript
import { RunnableSequence, RunnableMap } from "@langchain/core/runnables";

// 翻译单条文本的链
const translateChain = PromptTemplate
  .fromTemplate("将以下{text}翻译成{targetLang}")
  .pipe(model)
  .pipe((res) => res.content);

// 批量翻译处理器
const batchTranslator = new RunnableSequence([
  // 输入：{ texts: [], targetLang: "" }

  // 步骤1：为每个文本添加目标语言
  (input) => input.texts.map(text => ({
    text,
    targetLang: input.targetLang
  })),

  // 步骤2：批量映射翻译
  new RunnableMap({ bound: translateChain }),

  // 步骤3：重组结果
  (results) => ({
    translations: results,
    count: results.length,
    targetLang: input.targetLang
  })
]);

const result = await batchTranslator.invoke({
  texts: ["Hello", "World", "How are you"],
  targetLang: "中文"
});

console.log(result);
// {
//   translations: ["你好", "世界", "你好吗"],
//   count: 3,
//   targetLang: "中文"
// }
```

---

### 2.4 实战：文档批量处理管道

```javascript
import {
  RunnableSequence,
  RunnableParallel,
  RunnableMap
} from "@langchain/core/runnables";

// 文档处理管道
const documentPipeline = RunnableSequence.from([
  // 输入：Document[]（多个文档）

  // 步骤1：并行提取多种信息
  new RunnableMap({
    bound: RunnableParallel.from({
      // 每个文档的摘要
      summary: docSummaryChain,
      // 每个文档的关键词
      keywords: keywordChain,
      // 每个文档的情感
      sentiment: sentimentChain
    })
  }),

  // 步骤2：聚合分析
  (docResults) => ({
    documents: docResults,
    totalCount: docResults.length,
    avgSentiment: calculateAverage(docResults.map(d => d.sentiment)),
    allKeywords: [...new Set(docResults.flatMap(d => d.keywords))]
  }),

  // 步骤3：生成综合报告
  reportTemplate.pipe(model)
]);

// 使用
const docs = [
  { pageContent: "公司Q3财报..." },
  { pageContent: "产品发布会回顾..." },
  { pageContent: "客户投诉处理..." }
];

const report = await documentPipeline.invoke(docs);
```

---

## 3. batch vs RunnableMap 对比

| 特性 | `runnable.batch()` | `RunnableMap` |
|------|-------------------|---------------|
| 调用方式 | 实例方法 | 独立组件 |
| 输入 | 参数数组 | 需手动映射 |
| 输出控制 | 直接返回 | 可后续处理 |
| 适用场景 | 简单批量 | 复杂管道中的批量步骤 |

```javascript
// 简单批量：直接用 batch
const results = await chain.batch([input1, input2, input3]);

// 管道中的批量：用 RunnableMap
const pipeline = RunnableSequence.from([
  (input) => input.items,           // 提取数组
  new RunnableMap({ bound: processor }),  // 批量处理
  (results) => aggregate(results)   // 聚合
]);
```

---

## 4. 完整实战：智能客服批量回复

```javascript
import {
  RunnableSequence,
  RunnableParallel,
  RunnableMap,
  RunnableBranch
} from "@langchain/core/runnables";

// 批量客服回复系统
const batchReplySystem = RunnableSequence.from([
  // 输入：用户消息数组 [{userId, message, priority}, ...]

  // 步骤1：按优先级分类
  (messages) => ({
    urgent: messages.filter(m => m.priority === 'high'),
    normal: messages.filter(m => m.priority === 'normal'),
    low: messages.filter(m => m.priority === 'low')
  }),

  // 步骤2：并行处理不同优先级（批量映射）
  RunnableParallel.from({
    // 紧急：快速响应模板
    urgent: new RunnableMap({
      bound: urgentReplyChain
    }),
    // 普通：标准处理
    normal: new RunnableMap({
      bound: standardReplyChain
    }),
    // 低优：简单回复
    low: new RunnableMap({
      bound: simpleReplyChain
    })
  }),

  // 步骤3：合并并按原顺序排序
  (categorized) => {
    const all = [
      ...categorized.urgent.map((r, i) => ({ ...r, originalIndex: i, priority: 'high' })),
      ...categorized.normal.map((r, i) => ({ ...r, originalIndex: i, priority: 'normal' })),
      ...categorized.low.map((r, i) => ({ ...r, originalIndex: i, priority: 'low' }))
    ];
    return all.sort((a, b) => a.originalIndex - b.originalIndex);
  },

  // 步骤4：添加发送时间戳
  new RunnableLambda({
    func: (replies) => replies.map(r => ({
      ...r,
      sentAt: new Date().toISOString()
    }))
  })
]);

// 使用
const userMessages = [
  { userId: "U001", message: "系统崩溃了！", priority: "high" },
  { userId: "U002", message: "怎么改密码？", priority: "normal" },
  { userId: "U003", message: "建议增加深色模式", priority: "low" }
];

const replies = await batchReplySystem.invoke(userMessages);
```

---

## 5. 学习检查清单

- [ ] 掌握 `RunnableLambda` 封装自定义函数
- [ ] 理解简写形式（直接传入函数）
- [ ] 掌握 `RunnableMap` 批量处理数组
- [ ] 能区分 `batch()` 和 `RunnableMap` 的使用场景
- [ ] 能组合 Lambda + Map + Parallel + Branch 构建复杂管道

---
