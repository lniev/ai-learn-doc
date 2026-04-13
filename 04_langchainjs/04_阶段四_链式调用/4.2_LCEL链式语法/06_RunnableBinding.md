4.2 LCEL 链式语法 - 08 RunnableBinding（参数绑定）

核心概念

`RunnableBinding` 为 Runnable 组件预设部分参数，创建专用版本，无需每次传入。

```
原始链：chain.invoke({ topic: "AI", style: "简洁", language: "中文" })
绑定后：boundChain.invoke({ topic: "AI" })  // style 和 language 已固定
```

---

基础用法

```javascript
import { RunnableBinding } from "@langchain/core/runnables";
import { ChatOpenAI } from "@langchain/openai";
import { PromptTemplate } from "@langchain/core/prompts";

// 原始链
const chain = PromptTemplate.fromTemplate(
	"用{style}风格，将{topic}介绍给{audience}",
).pipe(new ChatOpenAI());

// 绑定固定参数
const techForKids = new RunnableBinding({
	bound: chain,
	kwargs: {
		style: "生动有趣",
		audience: "小学生",
	},
});

// 只需传入剩余参数
const result = await techForKids.invoke({ topic: "人工智能" });
// 等效于：chain.invoke({ topic: "AI", style: "生动有趣", audience: "小学生" })
```

---

简写形式：.bind()

```javascript
// 更简洁的 .bind() 方法（推荐）
const techForKids = chain.bind({
	style: "生动有趣",
	audience: "小学生",
});

// 模型参数绑定
const creativeModel = model.bind({
	temperature: 0.9,
	maxTokens: 500,
});

const preciseModel = model.bind({
	temperature: 0.1,
	maxTokens: 200,
});
```

---

实战：多角色绑定

```javascript
import { ChatPromptTemplate } from "@langchain/core/prompts";

const basePrompt = ChatPromptTemplate.fromMessages([
	["system", "你是{role}，专长：{expertise}。回答限制：{constraints}"],
	["human", "{question}"],
]).pipe(model);

// 创建不同角色的专用链
const doctorChain = basePrompt.bind({
	role: "全科医生",
	expertise: "常见病诊断",
	constraints: "不提供处方，建议就医",
});

const lawyerChain = basePrompt.bind({
	role: "知识产权律师",
	expertise: "商标专利",
	constraints: "不替代正式法律咨询",
});

const teacherChain = basePrompt.bind({
	role: "高中数学老师",
	expertise: "代数几何",
	constraints: "用高考考点讲解，每步说明原理",
});

// 使用
const answer1 = await doctorChain.invoke({ question: "头痛怎么办？" });
const answer2 = await lawyerChain.invoke({ question: "怎么申请专利？" });
const answer3 = await teacherChain.invoke({ question: "什么是导数？" });
```

---
