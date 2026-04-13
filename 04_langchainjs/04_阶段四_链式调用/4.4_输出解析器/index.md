知道了。以下是 4.4 输出解析器（Output Parsers） 的完整教程。

---

4.4 输出解析器（Output Parsers）

核心概念

输出解析器 = 将模型生成的非结构化文本转换为结构化数据。

```
模型输出："positive"  →  StringOutputParser  →  "positive"
模型输出："{\"name\":\"张三\"}"  →  JsonOutputParser  →  { name: "张三" }
模型输出：多行文本  →  StructuredOutputParser  →  { field1: "...", field2: "..." }
```

---

基础解析器

1. StringOutputParser（最常用）

```javascript
import { StringOutputParser } from "@langchain/core/output_parsers";

// 将 AIMessage 转换为纯字符串
const parser = new StringOutputParser();

const result = await parser.invoke(aiMessage);
// 等效于：aiMessage.content
```

链中使用：

```javascript
const chain = prompt.pipe(model).pipe(new StringOutputParser());

const text = await chain.invoke({ topic: "AI" });
// 直接得到字符串，无需 .content
```

---

2. JsonOutputParser（JSON 解析）

```javascript
import { JsonOutputParser } from "@langchain/core/output_parsers";

const parser = new JsonOutputParser();

// 模型必须输出合法 JSON
const prompt = PromptTemplate.fromTemplate(`
分析以下产品，返回 JSON 格式：
{{
  "name": "产品名称",
  "price": 价格数字,
  "category": "类别"
}}

产品描述：{description}
`);

const chain = prompt.pipe(model).pipe(parser);

const result = await chain.invoke({
  description: "iPhone 15 Pro，售价8999元，旗舰手机"
});
console.log(result);
// { name: "iPhone 15 Pro", price: 8999, category: "旗舰手机" }
```

---

3. StructuredOutputParser（结构化字段）

```javascript
import { StructuredOutputParser, ResponseSchema } from "@langchain/core/output_parsers";

// 定义输出模式
const parser = StructuredOutputParser.fromNamesAndDescriptions({
  answer: "问题的答案",
  confidence: "置信度评分（0-100）",
  sources: "参考来源列表"
});

// 获取格式指令（注入提示词）
const formatInstructions = parser.getFormatInstructions();

const prompt = PromptTemplate.fromTemplate(`
回答用户问题。
{format_instructions}

问题：{question}
`);

const chain = prompt.pipe(model).pipe(parser);

const result = await chain.invoke({
  question: "什么是机器学习？",
  format_instructions: formatInstructions
});
console.log(result);
// {
//   answer: "机器学习是...",
//   confidence: "95",
//   sources: ["维基百科", "教材第3章"]
// }
```

---

4. CommaSeparatedListOutputParser（列表解析）

```javascript
import { CommaSeparatedListOutputParser } from "@langchain/core/output_parsers";

const parser = new CommaSeparatedListOutputParser();

const prompt = PromptTemplate.fromTemplate(`
列出{topic}的5个要点，用逗号分隔。
`);

const chain = prompt.pipe(model).pipe(parser);

const result = await chain.invoke({ topic: "健康生活习惯" });
console.log(result);
// ["早睡早起", "均衡饮食", "适量运动", "多喝水", "保持心情愉快"]
```

---

高级解析器

5. RegexParser（正则解析）

```javascript
import { RegexParser } from "@langchain/core/output_parsers";

// 从文本提取特定模式
const parser = new RegexParser(
  /答案：(.+?)\n置信度：(\d+)/,  // 正则
  ["answer", "confidence"],       // 捕获组命名
  "no_answer"                     // 默认值
);

const text = `
思考过程...
答案：这是正确答案
置信度：95
`;

const result = parser.parse(text);
console.log(result);
// { answer: "这是正确答案", confidence: "95" }
```

---

6. Custom Output Parser（自定义）

```javascript
import { BaseOutputParser } from "@langchain/core/output_parsers";

class BooleanOutputParser extends BaseOutputParser<boolean> {
  lc_namespace = ["custom", "parsers"];

  parse(text: string): boolean {
    const normalized = text.trim().toLowerCase();
    if (normalized === "true" || normalized === "yes" || normalized === "是") {
      return true;
    }
    if (normalized === "false" || normalized === "no" || normalized === "否") {
      return false;
    }
    throw new Error(`无法解析为布尔值: ${text}`);
  }

  getFormatInstructions(): string {
    return '输出 "true" 或 "false"';
  }
}

// 使用
const parser = new BooleanOutputParser();
const result = await parser.parse("  YES  ");  // true
```

---

输出解析器与模型绑定

OpenAI 函数调用（结构化输出）

```javascript
import { ChatOpenAI } from "@langchain/openai";
import { z } from "zod";
import { zodToJsonSchema } from "zod-to-json-schema";

// 定义 Zod 模式
const AnswerSchema = z.object({
  answer: z.string().describe("详细答案"),
  confidence: z.number().min(0).max(100).describe("置信度"),
  relatedTopics: z.array(z.string()).describe("相关主题")
});

// 绑定到模型
const model = new ChatOpenAI({
  modelName: "gpt-4o-mini"
}).bind({
  functions: [{
    name: "answer_question",
    description: "回答问题",
    parameters: zodToJsonSchema(AnswerSchema)
  }],
  function_call: { name: "answer_question" }
});

// 解析函数调用结果
const result = await model.invoke("什么是量子计算？");
const parsed = JSON.parse(result.additional_kwargs.function_call.arguments);
console.log(parsed);
// { answer: "...", confidence: 92, relatedTopics: ["...", "..."] }
```

---

新版：withStructuredOutput（推荐）

```javascript
import { ChatOpenAI } from "@langchain/openai";
import { z } from "zod";

const model = new ChatOpenAI({ modelName: "gpt-4o-mini" });

// 直接绑定结构化输出
const structuredModel = model.withStructuredOutput(
  z.object({
    name: z.string(),
    age: z.number(),
    hobbies: z.array(z.string())
  }),
  {
    name: "person_extractor"
  }
);

// 直接得到解析后的对象
const result = await structuredModel.invoke("张三，25岁，喜欢篮球和编程");
console.log(result);
// { name: "张三", age: 25, hobbies: ["篮球", "编程"] }
```

---

完整实战：多阶段解析

```javascript
import { RunnableSequence } from "@langchain/core/runnables";

// 复杂解析管道
const analysisPipeline = RunnableSequence.from([
  // 阶段1：原始文本生成
  PromptTemplate.fromTemplate(`
    分析以下用户反馈，从三个维度评价：
    1. 情感（positive/negative/neutral）
    2. 紧急程度（1-10）
    3. 涉及部门（技术/产品/客服/其他）

    反馈：{feedback}
  `).pipe(model),

  // 阶段2：结构化解析
  StructuredOutputParser.fromNamesAndDescriptions({
    sentiment: "情感倾向",
    urgency: "紧急程度数字",
    department: "责任部门",
    summary: "一句话总结"
  }),

  // 阶段3：数据转换
  (parsed) => ({
    ...parsed,
    urgencyLevel: parsed.urgency >= 8 ? "high" :
                  parsed.urgency >= 5 ? "medium" : "low",
    needsEscalation: parsed.urgency >= 8 && parsed.sentiment === "negative"
  })
]);

const result = await analysisPipeline.invoke({
  feedback: "系统崩溃了，损失惨重，必须立即解决！"
});
console.log(result);
// {
//   sentiment: "negative",
//   urgency: "10",
//   department: "技术",
//   summary: "系统崩溃导致损失",
//   urgencyLevel: "high",
//   needsEscalation: true
// }
```

---

学习检查清单

- 掌握 `StringOutputParser` 基础字符串解析
- 掌握 `JsonOutputParser` JSON 数据解析
- 掌握 `StructuredOutputParser` 多字段结构化
- 掌握 `withStructuredOutput` 函数调用方式（推荐）
- 能自定义 `BaseOutputParser` 实现特殊解析
- 能组合解析器构建多阶段处理管道

---
