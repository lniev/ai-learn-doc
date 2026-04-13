#### LCEL 链式语法 - 09 RunnableRetry（错误重试）

核心概念

`RunnableRetry` 自动重试失败的 Runnable，支持指数退避、条件判断等。

---

基础用法

```javascript
import { RunnableRetry } from "@langchain/core/runnables";
import { ChatOpenAI } from "@langchain/openai";

// 易失败的组件（如调用外部 API）
const flakyChain = new RunnableRetry({
	bound: unstableApiChain,

	// 重试配置
	maxAttempts: 3, // 最多3次
	initialRetryDelay: 1000, // 首次重试间隔1秒
	maxRetryDelay: 10000, // 最大间隔10秒
	backoffMultiplier: 2, // 指数退避（1s, 2s, 4s）

	// 条件重试：只对特定错误重试
	retryIf: error => {
		// 只重试网络错误，不重试业务错误
		return (
			error.code === "ECONNRESET" ||
			error.code === "ETIMEDOUT" ||
			error.status === 429
		); // 限流
	},

	// 重试回调
	onFailedAttempt: (error, attempt) => {
		console.log(`第${attempt}次失败: ${error.message}，准备重试...`);
	},
});
```

---

简写形式：.withRetry()

```javascript
// 更简洁的 .withRetry() 方法（推荐）
const reliableChain = unstableChain.withRetry({
	stopAfterAttempt: 3,
	backoffMultiplier: 2,
});

// 使用
const result = await reliableChain.invoke({ input: "test" });
// 失败自动重试，成功返回结果，最终失败抛出错误
```

---

与链组合

```javascript
import { RunnableSequence } from "@langchain/core/runnables";

const robustPipeline = RunnableSequence.from([
	// 步骤1：稳定操作（无需重试）
	validateInput,

	// 步骤2：易失败操作（加重试）
	unstableApiCall.withRetry({
		stopAfterAttempt: 3,
		retryIf: e => e.status >= 500, // 只重试服务器错误
	}),

	// 步骤3：稳定操作
	processResult,
]);
```

---

4.2 LCEL 链式语法 - 10 RunnableFallback（故障转移）

核心概念

`RunnableFallback` 当主 Runnable 失败时，自动切换到备用方案。

```
主方案失败 → 尝试备用1 → 尝试备用2 → ... → 最终兜底
```

---

基础用法

```javascript
import { RunnableFallback } from "@langchain/core/runnables";
import { ChatOpenAI } from "@langchain/openai";

// 主模型（GPT-4，贵但强）
const primaryModel = new ChatOpenAI({ modelName: "gpt-4o" });

// 备用模型（GPT-4o-mini，便宜）
const backupModel = new ChatOpenAI({ modelName: "gpt-4o-mini" });

// 最终兜底（本地模型）
const localModel = new ChatOpenAI({
	modelName: "gpt-4o",
	configuration: { baseURL: "http://localhost:8000/v1" },
});

// 构建故障转移链
const resilientChain = new RunnableFallback({
	runnable: primaryModel,
	fallbacks: [
		backupModel,
		localModel,
		// 最终兜底：返回静态响应
		new RunnableLambda({
			func: () => "服务暂时不可用，请稍后重试",
		}),
	],
});

// 使用：自动按优先级尝试
const result = await resilientChain.invoke("复杂问题");
// GPT-4 失败 → 用 GPT-4o-mini → 再失败 → 用本地模型 → 再失败 → 返回静态消息
```

---

简写形式：.withFallbacks()

```javascript
// 更简洁的 .withFallbacks() 方法（推荐）
const resilientChain = primaryModel.withFallbacks({
	fallbacks: [backupModel, localModel],
	// 可选：对特定错误才触发 fallback
	exceptionsToHandle: [Error, RateLimitError],
});
```

---

实战：多层级故障恢复

```javascript
import {
  RunnableSequence,
  RunnableFallback,
  RunnableLambda
} from "@langchain/core/runnables";

// 智能客服：多层次故障恢复
const customerServiceChain = RunnableSequence.from([
  // 步骤1：意图识别（带故障转移）
  intentClassifier.withFallbacks({
    fallbacks: [
      simpleKeywordClassifier,  // 备用：关键词匹配
      new RunnableLambda({
        func: () => ({ intent: "unknown", confidence: 0 })
      })
    ]
  }),

  // 步骤2：路由处理（带重试）
  new RunnableBranch({
    ["billing", billingHandler.withRetry({ stopAfterAttempt: 2 })],
    ["tech", techHandler.withRetry({ stopAfterAttempt: 2 })],
    ["unknown", humanEscalation]  // 未知意图转人工
  }),

  // 步骤3：格式化输出（带兜底）
  formatResponse.withFallbacks({
    fallbacks: [
      simpleFormat,  // 简化格式
      new RunnableLambda({
        func: (input) => ({
          reply: "抱歉，处理您的请求时出现问题，已转人工客服。",
          code: "ERROR_001"
        })
      })
    ]
  })
]);
```

---

组合：Retry + Fallback + Binding

```javascript
import { ChatOpenAI } from "@langchain/openai";

// 构建高可用模型调用
const robustModel = new ChatOpenAI({ modelName: "gpt-4o" })
	// 绑定参数：降低温度，更稳定
	.bind({ temperature: 0.2, maxRetries: 2 })
	// 添加重试逻辑
	.withRetry({
		stopAfterAttempt: 3,
		backoffMultiplier: 2,
	})
	// 添加故障转移
	.withFallbacks({
		fallbacks: [
			// 备用1：降级的 GPT-4o-mini
			new ChatOpenAI({ modelName: "gpt-4o-mini" }),
			// 备用2：本地模型
			new ChatOpenAI({
				modelName: "qwen-7b",
				configuration: { baseURL: "http://localhost:8000/v1" },
			}),
		],
	});

// 使用：多层保护，确保可用
const result = await robustModel.invoke("重要问题");
```

---

## 完整对比表

| 组件               | 核心功能 | 使用场景     | 简写方法           |
| ------------------ | -------- | ------------ | ------------------ |
| `RunnableBinding`  | 预设参数 | 创建专用版本 | `.bind()`          |
| `RunnableRetry`    | 失败重试 | 临时故障恢复 | `.withRetry()`     |
| `RunnableFallback` | 故障转移 | 主备切换     | `.withFallbacks()` |

---

## 学习检查清单

| 检查项                                         | 状态 |
| ---------------------------------------------- | ---- |
| 掌握 `.bind()` 预设固定参数                    | [ ]  |
| 掌握 `.withRetry()` 自动重试配置               | [ ]  |
| 掌握 `.withFallbacks()` 多级故障转移           | [ ]  |
| 能组合 Binding + Retry + Fallback 构建高可用链 | [ ]  |
| 理解条件重试（`retryIf`）和特定错误处理        | [ ]  |

---

> 💡 **下一步**：可以继续学习 `RunnableAssign`（状态赋值）或 `RunnablePick` / `RunnableItemgetter`（数据提取）
