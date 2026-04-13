# RunnableParallel 和 RunnableBranch 完整教程

---

## RunnableParallel（并行执行）

### 核心概念

`RunnableParallel` 同时执行多个分支，合并结果。适用于需要多个独立处理的任务。

```
顺序执行：A → B → C（总时间 = A+B+C）
并行执行：A、B、C 同时运行（总时间 = max(A,B,C)）
```

---

### 基础用法

```javascript
import { RunnableParallel } from "@langchain/core/runnables";
import { ChatOpenAI } from "@langchain/openai";
import { PromptTemplate } from "@langchain/core/prompts";

const model = new ChatOpenAI({ modelName: "gpt-4o-mini" });

// 并行执行多个分析
const parallelChain = RunnableParallel.from({
  // 分支1：情感分析
  sentiment: PromptTemplate
    .fromTemplate("判断以下文本的情感（positive/negative）：{text}")
    .pipe(model),

  // 分支2：关键词提取
  keywords: PromptTemplate
    .fromTemplate("提取以下文本的3个关键词：{text}")
    .pipe(model),

  // 分支3：摘要生成
  summary: PromptTemplate
    .fromTemplate("用一句话总结：{text}")
    .pipe(model)
});

// 执行（三个分支同时调用模型）
const result = await parallelChain.invoke({
  text: "这部电影太棒了，演员演技出色，剧情紧凑，强烈推荐！"
});

console.log(result);
// {
//   sentiment: AIMessage { content: "positive" },
//   keywords: AIMessage { content: "电影, 演技, 推荐" },
//   summary: AIMessage { content: "一部值得推荐的优秀电影" }
// }
```

---

### 结果处理与重组

```javascript
import { RunnableSequence } from "@langchain/core/runnables";

// 并行 + 后续合并处理
const chain = RunnableSequence.from([
  // 步骤1：并行分析
  RunnableParallel.from({
    sentiment: sentimentChain,
    keywords: keywordsChain,
    summary: summaryChain
  }),

  // 步骤2：合并结果（所有分支完成后执行）
  (parallelResults) => {
    const { sentiment, keywords, summary } = parallelResults;

    return {
      analysis: {
        mood: sentiment.content,
        keyPoints: keywords.content.split(", "),
        brief: summary.content
      },
      confidence: "high",
      timestamp: new Date().toISOString()
    };
  }
]);

const finalResult = await chain.invoke({ text: "用户反馈文本..." });
```

---

### 与透传结合（保留原始输入）

```javascript
import { RunnablePassthrough } from "@langchain/core/runnables";

const chain = RunnableSequence.from([
  // 并行处理，同时保留原始输入
  RunnableParallel.from({
    original: RunnablePassthrough(),  // 透传不变
    processed: (input) => input.text.toUpperCase(),
    analyzed: analysisChain
  }),

  // 合并：原始 + 处理结果 + 分析结果
  (input) => ({
    source: input.original,
    transformed: input.processed,
    insights: input.analyzed
  })
]);
```

---

## RunnableBranch（条件分支）

### 核心概念

`RunnableBranch` 根据条件选择不同的执行路径，实现 if/else 逻辑。

```
输入 → [条件判断] → 满足条件A？→ 执行分支A
                → 满足条件B？→ 执行分支B
                → 默认 → 执行默认分支
```

---

### 基础用法

```javascript
import { RunnableBranch } from "@langchain/core/runnables";
import { ChatOpenAI } from "@langchain/openai";

const model = new ChatOpenAI({ modelName: "gpt-4o-mini" });

// 条件分支链
const branchChain = new RunnableBranch({
  // 分支1：如果是技术问题
  [
    (input) => input.category === "tech",
    PromptTemplate
      .fromTemplate("作为技术专家，回答：{question}")
      .pipe(model)
  ],

  // 分支2：如果是销售问题
  [
    (input) => input.category === "sales",
    PromptTemplate
      .fromTemplate("作为销售顾问，回答：{question}")
      .pipe(model)
  ],

  // 默认分支：通用回答
  PromptTemplate
    .fromTemplate("作为通用助手，回答：{question}")
    .pipe(model)
]);

// 执行（自动选择分支）
const result1 = await branchChain.invoke({
  category: "tech",
  question: "什么是REST API？"
});
// 走技术专家分支

const result2 = await branchChain.invoke({
  category: "sales",
  question: "有优惠吗？"
});
// 走销售顾问分支

const result3 = await branchChain.invoke({
  category: "other",
  question: "你好"
});
// 走默认分支
```

---

### 嵌套分支与复杂逻辑

```javascript
import { RunnableBranch, RunnableSequence } from "@langchain/core/runnables";

// 分类器（决定走哪个分支）
const classifierChain = PromptTemplate
  .fromTemplate(`
分析用户问题类型，只返回一个标签：tech | sales | billing | other
问题：{question}
标签：
`)
  .pipe(model)
  .pipe((res) => res.content.trim());

// 主链：分类 + 分支处理
const mainChain = RunnableSequence.from([
  // 步骤1：分类
  (input) => ({ question: input.question }),
  classifierChain,

  // 步骤2：根据分类结果分支处理
  (category) => ({ category, question: input.question }),
  new RunnableBranch({
    ["tech", techSupportChain],
    ["sales", salesChain],
    ["billing", billingChain],
    [true, generalChain]  // 默认分支（条件写 true）
  })
]);

const result = await mainChain.invoke({ question: "怎么退款？" });
// 分类为 "billing" → 走 billingChain
```

---

### 分支 + 并行组合实战

```javascript
import {
  RunnableSequence,
  RunnableParallel,
  RunnableBranch
} from "@langchain/core/runnables";

// 智能客服系统
const smartSupportChain = RunnableSequence.from([
  // 步骤1：意图识别（并行多个检测器）
  RunnableParallel.from({
    isUrgent: urgencyDetector,      // 紧急度检测
    sentiment: sentimentAnalyzer,  // 情感分析
    category: intentClassifier,     // 意图分类
    language: languageDetector      // 语言检测
  }),

  // 步骤2：根据综合判断选择处理策略
  (analysis) => {
    const { isUrgent, sentiment, category, language } = analysis;

    // 动态选择分支条件
    if (isUrgent.score > 0.8) return { ...analysis, route: "urgent" };
    if (sentiment.negative) return { ...analysis, route: "care" };
    return { ...analysis, route: category.value };
  },

  // 步骤3：分支处理
  new RunnableBranch({
    ["urgent", urgentEscalationChain],   // 紧急：立即升级
    ["care", empathyCareChain],          // 负面：安抚关怀
    ["tech", techSupportChain],          // 技术：专业解答
    ["billing", billingChain],           // 账单：财务处理
    [true, generalInquiryChain]          // 默认：通用处理
  }),

  // 步骤4：后处理（格式化、记录日志）
  (response) => ({
    answer: response.content,
    handledBy: response._route,
    timestamp: new Date().toISOString()
  })
]);

const result = await smartSupportChain.invoke({
  message: "我的账户被锁定了，急需用钱！"
});
// 检测为 urgent → 走紧急升级流程
```

---

## 对比总结

| 组件 | 执行方式 | 适用场景 |
|------|----------|----------|
| `RunnableSequence` | 顺序执行 | 有依赖关系的步骤 |
| `RunnableParallel` | 并行执行 | 独立任务同时处理 |
| `RunnableBranch` | 条件分支 | 根据输入动态选择路径 |
| `RunnablePassthrough` | 透传 | 保留原始数据 |

---

## 学习检查清单

- [ ] 掌握 `RunnableParallel.from({})` 并行执行多个分支
- [ ] 能合并并行结果并后续处理
- [ ] 掌握 `RunnableBranch` 的条件判断语法
- [ ] 能嵌套分支实现复杂逻辑
- [ ] 能组合 Sequence + Parallel + Branch 构建完整工作流

---

> 💡 **下一步**：可以继续学习 `RunnableLambda`（自定义函数封装）或 `RunnableMap`（批量映射处理）
