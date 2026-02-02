# oh-my-opencode 源码解析 - 第三部分：代理系统设计

## 代理系统概述

代理系统是 oh-my-opencode 的核心组成部分，通过多个专门化代理协同工作，实现复杂任务的分工处理。每个代理都有特定的职责和专业领域。

## 代理系统架构

### 主要代理类型

1. **Sisyphus（主协调代理）**
   - 模型：`anthropic/claude-opus-4-5`
   - 职责：任务规划、委派和执行协调

2. **Oracle（架构师/调试专家）**
   - 模型：`openai/gpt-5.2`
   - 职责：架构设计、代码审查、复杂调试

3. **Librarian（知识管理专家）**
   - 模型：`anthropic/claude-sonnet-4-5` 或 `google/gemini-3-flash`
   - 职责：文档查询、多仓库分析、实现示例查找

4. **Explore（探索代理）**
   - 模型：`opencode/grok-code` 或其他轻量模型
   - 职责：快速代码库探索、模式匹配

5. **Frontend UI/UX Engineer（前端专家）**
   - 模型：`google/gemini-3-pro-high`
   - 职责：UI/UX 设计、前端开发

6. **Document Writer（文档专家）**
   - 模型：`google/gemini-3-flash`
   - 职责：技术文档编写

7. **Multimodal Looker（多模态专家）**
   - 模型：`google/gemini-3-flash`
   - 职责：图片、PDF 等多媒体内容分析

## 代理协调机制

### Sisyphus 协调逻辑

Sisyphus 作为主协调代理，实现以下协调模式：

```typescript
// 在 Sisyphus 代理内部逻辑中
if (taskRequiresSpecializedKnowledge(domain)) {
  // 根据任务领域委派给专门代理
  const specialistAgent = getSpecialistAgent(domain);
  return delegateTask(specialistAgent, task);
}

if (complexProblemDetected()) {
  // 在遇到复杂问题时调用 Oracle
  return consultOracle(problem);
}

if (documentationNeeded()) {
  // 需要文档时调用 Librarian
  return getDocumentationFromLibrarian(query);
}
```

### 后台代理管理

通过 BackgroundManager 实现代理的后台执行：

```typescript
// 创建后台代理任务
const backgroundManager = new BackgroundManager(ctx);

// 后台执行示例
const taskId = backgroundManager.createTask({
  agent: "explore",
  prompt: "Find all occurrences of authentication logic",
  priority: "high"
});

// 获取后台任务结果
const result = await backgroundManager.getResult(taskId);
```

## 代理配置系统

### 配置结构

代理配置通过 `oh-my-opencode.json` 文件管理：

```json
{
  "agents": {
    "explore": {
      "model": "anthropic/claude-haiku-4-5",
      "temperature": 0.5,
      "tools": {
        "include": ["read", "grep", "glob"]
      }
    },
    "frontend-ui-ux-engineer": {
      "disable": true
    }
  }
}
```

### 权限控制系统

代理的权限通过精细的配置进行控制：

```typescript
// 权限配置示例
{
  "permission": {
    "edit": "ask",           // 编辑文件需要确认
    "bash": "allow",         // 允许执行 bash 命令
    "webfetch": "deny"       // 禁止网络请求
  }
}
```

## 任务委派机制

### Sisyphus Task 工具

`sisyphus_task` 工具用于委派任务给专门代理：

```typescript
// 使用类别委派
sisyphus_task(category="visual", prompt="Create a responsive dashboard component")

// 直接指定代理
sisyphus_task(agent="oracle", prompt="Review this architecture")
```

### 类别系统

系统预定义了多个任务类别：

- `visual`：前端/UI/UX 任务，使用 `google/gemini-3-pro-preview`
- `business-logic`：后端逻辑，使用 `openai/gpt-5.2`

### 自定义类别

用户可以定义自己的类别：

```json
{
  "categories": {
    "data-science": {
      "model": "anthropic/claude-sonnet-4-5",
      "temperature": 0.2,
      "prompt_append": "Focus on data analysis and ML pipelines."
    }
  }
}
```

## 代理通信机制

### 工具调用模式

代理通过标准工具进行通信和协作：

```typescript
// call_omo_agent 工具用于调用其他代理
const result = await call_omo_agent({
  agent: "explore",
  prompt: "Find all usages of this function",
  run_in_background: true  // 后台运行
});
```

### 结果处理

后台任务的结果通过特定工具获取：

```typescript
// 获取后台任务结果
const backgroundResult = await background_output(task_id);

// 取消后台任务
await background_cancel(taskId);
```

## 代理优化策略

### 并行执行

系统支持多个代理并行执行：

```typescript
// 同时启动多个后台任务
const task1 = await call_omo_agent({ agent: "explore", prompt: "Find auth logic", run_in_background: true });
const task2 = await call_omo_agent({ agent: "librarian", prompt: "Get auth docs", run_in_background: true });
const task3 = await call_omo_agent({ agent: "oracle", prompt: "Review auth design", run_in_background: true });

// 等待所有任务完成
const [result1, result2, result3] = await Promise.all([
  background_output(task1.task_id),
  background_output(task2.task_id),
  background_output(task3.task_id)
]);
```

### 模型选择优化

根据任务类型选择最适合的模型：

- **复杂推理任务**：使用 GPT-5.2 或 Claude Opus
- **快速搜索任务**：使用 Claude Haiku 或 Grok
- **创意任务**：使用 Gemini Pro

## 错误处理和恢复

### 代理失败处理

当代理失败时，系统会尝试降级或重试：

```typescript
// 代理失败时的处理逻辑
if (agentCallFailed(agent, task)) {
  // 尝试其他代理
  const fallbackAgent = getFallbackAgent(agent);
  return await callAgent(fallbackAgent, task);
}
```

### 负载均衡

在使用多个账户时，系统会自动负载均衡：

```typescript
// 对于 Google 模型，支持多账户负载均衡
// 当一个账户达到速率限制时，自动切换到其他账户
```

## 性能优化

### 并发控制

通过后台任务配置控制并发数量：

```json
{
  "background_task": {
    "defaultConcurrency": 5,
    "providerConcurrency": {
      "anthropic": 3,
      "openai": 5,
      "google": 10
    },
    "modelConcurrency": {
      "anthropic/claude-opus-4-5": 2,
      "google/gemini-3-flash": 10
    }
  }
}
```

### 成本管理

- 对昂贵模型限制并发数
- 对快速模型允许更多并发
- 后台任务优先使用经济模型

代理系统通过专业分工和智能协调，实现了高效的任务处理能力。每个代理专注于其优势领域，通过统一的接口进行协作，形成了强大的开发助手团队。

下一节我们将深入分析 LSP 和 AST-Grep 工具的实现机制。