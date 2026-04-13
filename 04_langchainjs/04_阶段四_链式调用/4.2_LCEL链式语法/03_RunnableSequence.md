知道了。以下是 4.2 LCEL 链式语法 中 03 RunnableSequence 的完整教程。

---

4.2 LCEL 链式语法 - 03 RunnableSequence

核心概念

`RunnableSequence` 是 LCEL 中显式构建顺序执行链的类，等效于 `.pipe()` 链式调用，但更灵活。

```
.pipe() 语法：隐式序列
prompt.pipe(model).pipe(parser)

RunnableSequence：显式序列
RunnableSequence.from([prompt, model, parser])
```

---

基础用法

```javascript
import { RunnableSequence } from "@langchain/core/runnables";
import { ChatOpenAI } from "@langchain/openai";
import { PromptTemplate } from "@langchain/core/prompts";
import { StringOutputParser } from "@langchain/core/output_parsers";

// 组件
const prompt = PromptTemplate.fromTemplate("解释{topic}，用小学生能听懂的话");
const model = new ChatOpenAI({ modelName: "gpt-4o-mini" });
const parser = new StringOutputParser();

// 显式构建序列
const chain = RunnableSequence.from([
	prompt, // 步骤1：格式化提示
	model, // 步骤2：调用模型
	parser, // 步骤3：解析输出
]);

// 执行
const result = await chain.invoke({ topic: "区块链" });
console.log(result); // 纯字符串
```

---

带数据转换的序列

```javascript
import {
	RunnableSequence,
	RunnablePassthrough,
} from "@langchain/core/runnables";

const chain = RunnableSequence.from([
	// 步骤1：注入额外上下文
	input => ({
		...input,
		timestamp: new Date().toISOString(),
		userLevel: "premium",
	}),

	// 步骤2：提示模板
	ChatPromptTemplate.fromMessages([
		["system", "当前时间：{timestamp}，用户等级：{userLevel}"],
		["human", "{question}"],
	]),

	// 步骤3：模型
	new ChatOpenAI(),

	// 步骤4：提取内容并包装
	response => ({
		answer: response.content,
		meta: {
			model: "gpt-4o-mini",
			timestamp: new Date().toISOString(),
		},
	}),
]);

const result = await chain.invoke({ question: "你好" });
console.log(result);
// {
//   answer: "你好！有什么可以帮您？",
//   meta: { model: "gpt-4o-mini", timestamp: "2026-04-13..." }
// }
```

---

RunnablePassthrough 透传

```javascript
import { RunnablePassthrough } from "@langchain/core/runnables";

// 场景：保留原始输入，同时添加新字段
const chain = RunnableSequence.from([
	// 透传原始输入，并添加 processed 字段
	RunnablePassthrough.assign({
		processed: input => input.raw.toUpperCase(),
		length: input => input.raw.length,
	}),

	// 使用转换后的数据
	input =>
		`原始：${input.raw}，处理后：${input.processed}，长度：${input.length}`,
]);

const result = await chain.invoke({ raw: "hello" });
console.log(result); // "原始：hello，处理后：HELLO，长度：5"
```

---

序列的错误处理与重试

```javascript
import { RunnableSequence } from "@langchain/core/runnables";

const chain = RunnableSequence.from(
	[
		{
			// 带配置的步骤
			runnable: prompt,
			name: "format_prompt",
		},
		{
			runnable: model,
			name: "call_model",
			// 步骤级重试配置
			retry: {
				maxRetries: 3,
				onFailedAttempt: error => console.log("Retrying...", error.message),
			},
		},
		parser,
	],
	{
		// 全局配置
		name: "my_chain",
		tags: ["production", "v1.0"],
	},
);
```

---

序列与 .pipe() 对比

场景 推荐方式 原因
简单线性链 `.pipe()` 简洁直观
需要中间数据转换 `RunnableSequence` 可插入函数步骤
需要透传原始输入 `RunnablePassthrough` 保留上下文
复杂条件分支 `RunnableSequence` + 函数 灵活控制流
需要详细配置（重试、日志） `RunnableSequence` 支持步骤级配置

---

实战：多步骤数据处理链

```javascript
import { RunnableSequence } from "@langchain/core/runnables";

// 复杂业务链：查询 → 分析 → 报告 → 格式化
const businessChain = RunnableSequence.from([
	// 步骤1：接收原始查询，标准化
	rawQuery => ({
		query: rawQuery.trim().toLowerCase(),
		id: generateId(),
	}),

	// 步骤2：检索相关数据（模拟）
	async input => {
		const data = await fetchData(input.query);
		return { ...input, data };
	},

	// 步骤3：分析数据
	analysisPrompt.pipe(analysisModel),

	// 步骤4：生成报告
	analysisResult => ({
		analysis: analysisResult.content,
		timestamp: new Date().toISOString(),
	}),

	// 步骤5：格式化输出
	reportTemplate.pipe(formatModel).pipe(outputParser),
]);

const finalReport = await businessChain.invoke("Q3 销售数据");
```

---

学习检查清单

- 掌握 `RunnableSequence.from([...])` 显式构建链
- 理解 `.pipe()` 与 `RunnableSequence` 的等价关系
- 会用 `RunnablePassthrough.assign()` 透传并扩展数据
- 能在序列中插入自定义函数步骤
- 了解步骤级配置（重试、命名、标签）

---
