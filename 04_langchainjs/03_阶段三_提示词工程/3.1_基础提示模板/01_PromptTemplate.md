 模板格式化

核心概念

模板格式化 = 预定义文本结构 + 动态变量注入，实现提示词的标准化和复用。

```
模板: "你好，{name}！今天是{day}。"
      ↓ 注入 {name: "张三", day: "周一"}
结果: "你好，张三！今天是周一。"
```

---

基础模板（PromptTemplate）

```javascript
import { PromptTemplate } from "@langchain/core/prompts";

// 1. 创建模板
const template = PromptTemplate.fromTemplate(
  "请用一句话介绍{topic}，要求{style}风格。"
);

// 2. 格式化（注入变量）
const formatted = await template.format({
  topic: "区块链",
  style: "幽默"
});

console.log(formatted);
// 输出: 请用一句话介绍区块链，要求幽默风格。
```

---

模板 + 模型调用

```javascript
import { PromptTemplate } from "@langchain/core/prompts";
import { ChatOpenAI } from "@langchain/openai";

const model = new ChatOpenAI({ modelName: "gpt-4o-mini" });

// 创建模板
const template = PromptTemplate.fromTemplate(
  "将以下{text}翻译成{language}："
);

// 方式1：先格式化，再调用
const prompt = await template.format({
  text: "Hello World",
  language: "中文"
});
const result1 = await model.invoke(prompt);

// 方式2：用 LCEL 链式（推荐）
const chain = template.pipe(model);
const result2 = await chain.invoke({
  text: "Hello World",
  language: "中文"
});
```
