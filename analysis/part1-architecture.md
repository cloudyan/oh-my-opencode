# oh-my-opencode 源码解析 - 第一部分：项目架构与入口

## 项目概述

oh-my-opencode 是一个为 OpenCode 平台设计的增强插件，旨在提供类似 Claude Code 的功能，包括专门化 AI 代理、LSP 工具、MCP 集成等。项目核心目标是实现"像人类工程师一样编码"的代理系统。

## 项目入口文件分析

### src/index.ts - 主入口

```typescript
import type { Plugin } from "@opencode-ai/plugin";
import {
  createTodoContinuationEnforcer,
  createContextWindowMonitorHook,
  createSessionRecoveryHook,
  createSessionNotification,
  createCommentCheckerHooks,
  createToolOutputTruncatorHook,
  createDirectoryAgentsInjectorHook,
  createDirectoryReadmeInjectorHook,
  createEmptyTaskResponseDetectorHook,
  createThinkModeHook,
  createClaudeCodeHooksHook,
  createAnthropicContextWindowLimitRecoveryHook,
  createPreemptiveCompactionHook,
  createCompactionContextInjector,
  createRulesInjectorHook,
  createBackgroundNotificationHook,
  createAutoUpdateCheckerHook,
  createKeywordDetectorHook,
  createAgentUsageReminderHook,
  createNonInteractiveEnvHook,
  createInteractiveBashSessionHook,
  createEmptyMessageSanitizerHook,
  createThinkingBlockValidatorHook,
  createRalphLoopHook,
  createAutoSlashCommandHook,
  createEditErrorRecoveryHook,
  createTaskResumeInfoHook,
  createStartWorkHook,
  createSisyphusOrchestratorHook,
  createPrometheusMdOnlyHook,
} from "./hooks";
```

### 核心初始化流程

1. **配置加载**：
```typescript
const pluginConfig = loadPluginConfig(ctx.directory, ctx);
const disabledHooks = new Set(pluginConfig.disabled_hooks ?? []);
const isHookEnabled = (hookName: HookName) => !disabledHooks.has(hookName);
```

2. **组件条件创建**：
```typescript
const todoContinuationEnforcer = isHookEnabled("todo-continuation-enforcer")
  ? createTodoContinuationEnforcer(ctx, { backgroundManager })
  : null;
```

3. **插件对象返回**：
- 工具 (tool) - 包含各种开发工具
- 事件处理器 (event) - 处理系统事件
- 工具执行前后处理器 - 拦截工具调用

### 架构设计特点

1. **模块化设计**：功能按模块分离，便于维护
2. **可配置性**：通过配置文件控制功能启用/禁用
3. **事件驱动**：基于事件的组件通信
4. **插件化**：遵循 OpenCode 插件规范

### 关键组件关系图

```
[User Input] 
    ↓
[Chat Message Hook] → [Keyword Detector] → [Context Injector]
    ↓
[Tool Execute Before] → [Various Hooks]
    ↓
[Tool Execution]
    ↓
[Tool Execute After] → [Output Processors]
    ↓
[Event System] → [Session Management, Recovery, etc.]
```

### 核心概念

1. **PluginInput/Output**：OpenCode 插件系统的基础接口
2. **Hook System**：拦截和处理系统事件的机制
3. **Context Management**：上下文信息的收集和注入
4. **Background Management**：后台任务管理

下一节我们将深入分析钩子系统的设计和实现。