知道了。以下是 阶段四：链式调用 中 4.1 基础链（LLM Chain） 的完整教程。

---

4.1 基础链（LLM Chain）

核心概念

链（Chain）= 组件的有序组合，实现"输入 → 处理 → 输出"的自动化流程。

```
基础链结构：
输入变量 → [PromptTemplate] → 格式化提示 → [LLM] → 生成结果 → 输出
```

---

传统方式（了解即可，已废弃）

```javascript
import { LLMChain } from "langchain/chains";  // 旧版，不推荐

const chain = new LLMChain({
  llm: model,
  prompt: promptTemplate
});

const result = await chain.call({ input: "你好" });  // 旧 API
```

> ⚠️ 注意：`LLMChain` 和 `.call()` 已废弃，推荐使用 LCEL 语法。

---