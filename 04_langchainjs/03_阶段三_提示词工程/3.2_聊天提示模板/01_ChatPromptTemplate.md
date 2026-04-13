---

聊天模板（ChatPromptTemplate）

```javascript
import { ChatPromptTemplate } from "@langchain/core/prompts";

// 多角色模板
const chatTemplate = ChatPromptTemplate.fromMessages([
  ["system", "你是{role}，擅长{skill}。"],
  ["human", "请帮我{task}"]
]);

// 格式化
const messages = await chatTemplate.formatMessages({
  role: "前端专家",
  skill: "React",
  task: "优化性能"
});

console.log(messages);
// [
//   SystemMessage { content: "你是前端专家，擅长React。" },
//   HumanMessage { content: "请帮我优化性能" }
// ]

// 直接调用
const chain = chatTemplate.pipe(model);
const result = await chain.invoke({
  role: "前端专家",
  skill: "React",
  task: "优化性能"
});
```

---
