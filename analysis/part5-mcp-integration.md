# oh-my-opencode 源码解析 - 第五部分：MCP集成系统

## MCP系统概述

MCP（Model Context Protocol）是 Claude Code 中引入的协议，用于集成外部服务。oh-my-opencode 实现了 MCP 集成，支持网络搜索、文档查询、代码搜索等多种外部服务。

## MCP 集成架构

### 内置 MCP 服务

项目内置了三个主要 MCP 服务：

1. **websearch**：基于 Exa AI 的网络搜索
2. **context7**：官方文档查询
3. **grep_app**：GitHub 代码搜索

### MCP 管理机制

```typescript
// 技能 MCP 管理器 (src/features/skill-mcp-manager/index.ts)
export class SkillMcpManager {
  private servers = new Map<string, McpServer>();
  private connections = new Map<string, McpConnection>();
  
  async connectSkillMcp(mcpName: string, sessionID: string) {
    if (!this.servers.has(mcpName)) {
      // 如果服务器不存在，创建新的服务器
      const server = await this.createMcpServer(mcpName);
      this.servers.set(mcpName, server);
    }
    
    const server = this.servers.get(mcpName)!;
    const connection = await server.connect(sessionID);
    this.connections.set(`${mcpName}:${sessionID}`, connection);
    
    return connection;
  }
  
  async execute(mcpName: string, toolName: string, arguments: any) {
    const server = this.servers.get(mcpName);
    if (!server) {
      throw new Error(`MCP server ${mcpName} not found`);
    }
    
    return await server.execute(toolName, arguments);
  }
}
```

## 内置 MCP 服务实现

### 1. Web Search (Exa AI)

```typescript
// Web Search MCP 实现
export const websearch_web_search_exa = {
  name: "websearch_web_search_exa",
  description: "使用 Exa AI 进行网络搜索",
  parameters: {
    type: "object",
    properties: {
      query: { type: "string", description: "搜索查询" },
      numResults: { type: "number", description: "返回结果数量" },
      livecrawl: { 
        type: "string", 
        enum: ["fallback", "preferred"], 
        description: "实时爬取模式" 
      },
      type: { 
        type: "string", 
        enum: ["auto", "fast", "deep"], 
        description: "搜索类型" 
      }
    },
    required: ["query"]
  },
  handler: async (args: WebSearchArgs, ctx: ToolContext) => {
    const response = await fetch("https://api.exa.ai/search", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Authorization": `Bearer ${getExaApiKey()}`
      },
      body: JSON.stringify({
        query: args.query,
        numResults: args.numResults || 8,
        type: args.type || "auto"
      })
    });
    
    const data = await response.json();
    return {
      results: data.results,
      query: args.query,
      timestamp: new Date().toISOString()
    };
  }
};
```

### 2. Context7 (官方文档查询)

```typescript
// Context7 MCP 实现
export const context7_resolve_library_id = {
  name: "context7_resolve-library-id",
  description: "解析库名到 Context7 兼容的库 ID",
  parameters: {
    type: "object",
    properties: {
      query: { type: "string", description: "用户原始问题" },
      libraryName: { type: "string", description: "要搜索的库名" }
    },
    required: ["query", "libraryName"]
  },
  handler: async (args: Context7ResolveArgs, ctx: ToolContext) => {
    const response = await fetch("https://context7.com/api/resolve", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        query: args.query,
        library: args.libraryName
      })
    });
    
    const data = await response.json();
    return {
      libraries: data.libraries,
      selected: data.libraries[0], // 选择最相关的库
      reason: `Selected based on name similarity and documentation coverage`
    };
  }
};

export const context7_query_docs = {
  name: "context7_query-docs",
  description: "查询 Context7 的文档和代码示例",
  parameters: {
    type: "object",
    properties: {
      libraryId: { type: "string", description: "Context7 库 ID" },
      query: { type: "string", description: "查询问题" }
    },
    required: ["libraryId", "query"]
  },
  handler: async (args: Context7QueryArgs, ctx: ToolContext) => {
    const response = await fetch(`https://context7.com/api/query`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        library_id: args.libraryId,
        query: args.query
      })
    });
    
    const data = await response.json();
    return {
      results: data.results,
      library: args.libraryId,
      query: args.query,
      totalResults: data.results.length
    };
  }
};
```

### 3. Grep.app (GitHub 代码搜索)

```typescript
// Grep.app MCP 实现
export const grep_app_searchGitHub = {
  name: "grep_app_searchGitHub",
  description: "在 GitHub 仓库中搜索代码模式",
  parameters: {
    type: "object",
    properties: {
      query: { type: "string", description: "要搜索的代码模式" },
      matchCase: { type: "boolean", description: "是否区分大小写" },
      matchWholeWords: { type: "boolean", description: "是否匹配完整单词" },
      useRegexp: { type: "boolean", description: "是否使用正则表达式" },
      repo: { type: "string", description: "指定仓库" },
      path: { type: "string", description: "指定文件路径" },
      language: { type: "array", items: { type: "string" }, description: "编程语言过滤" }
    },
    required: ["query"]
  },
  handler: async (args: GrepAppArgs, ctx: ToolContext) => {
    // 构建搜索参数
    const searchParams = new URLSearchParams({
      q: args.query,
      case: args.matchCase ? "true" : "false",
      words: args.matchWholeWords ? "true" : "false",
      regex: args.useRegexp ? "true" : "false"
    });
    
    if (args.repo) searchParams.append('repo', args.repo);
    if (args.path) searchParams.append('path', args.path);
    if (args.language) args.language.forEach(lang => 
      searchParams.append('language', lang)
    );
    
    const response = await fetch(`https://grep.app/api/search?${searchParams.toString()}`, {
      headers: { "User-Agent": "oh-my-opencode/1.0" }
    });
    
    const data = await response.json();
    return {
      matches: data.data.matches,
      total: data.data.total,
      query: args.query,
      filters: {
        repo: args.repo,
        path: args.path,
        language: args.language
      }
    };
  }
};
```

## 技能嵌入 MCP

### 技能 MCP 支持

oh-my-opencode 实现了技能嵌入 MCP 的功能，允许技能自带 MCP 服务器：

```typescript
// 技能 MCP 工具 (src/tools/skill-mcp.ts)
export const skill_mcp = {
  name: "skill_mcp",
  description: "调用技能嵌入的 MCP 服务器操作",
  parameters: {
    type: "object",
    properties: {
      mcp_name: { type: "string", description: "MCP 服务器名称" },
      tool_name: { type: "string", description: "工具名称" },
      resource_name: { type: "string", description: "资源名称" },
      prompt_name: { type: "string", description: "提示名称" },
      arguments: { type: "string", description: "工具参数" },
      grep: { type: "string", description: "搜索参数" }
    },
    required: ["mcp_name"]
  },
  handler: async (args: SkillMcpArgs, ctx: ToolContext) => {
    const { mcp_name, tool_name, resource_name, prompt_name, arguments: argStr } = args;
    
    // 获取当前会话 ID
    const sessionID = ctx.sessionId || getMainSessionID();
    if (!sessionID) {
      throw new Error("No active session found for MCP execution");
    }
    
    // 连接到 MCP 服务器
    const connection = await skillMcpManager.connectSkillMcp(mcp_name, sessionID);
    
    // 准备工具参数
    let toolArgs = {};
    if (argStr) {
      try {
        toolArgs = JSON.parse(argStr);
      } catch (e) {
        toolArgs = { raw: argStr }; // 如果不是 JSON，作为原始参数
      }
    }
    
    // 执行 MCP 工具
    let result;
    if (tool_name) {
      result = await connection.executeTool(tool_name, toolArgs);
    } else if (resource_name) {
      result = await connection.getResource(resource_name);
    } else if (prompt_name) {
      result = await connection.getPrompt(prompt_name, toolArgs);
    } else {
      throw new Error("Must specify either tool_name, resource_name, or prompt_name");
    }
    
    return result;
  }
};
```

### Playwright 技能 MCP

Playwright 技能是一个很好的 MCP 集成示例：

```yaml
# Playwright 技能定义 (作为示例)
---
description: Browser automation skill
mcp:
  playwright:
    command: npx
    args: ["-y", "@anthropic-ai/mcp-playwright"]
---
```

## MCP 配置管理

### 系统 MCP 服务器注册

```typescript
// 获取系统 MCP 服务器名称 (src/features/claude-code-mcp-loader.ts)
export function getSystemMcpServerNames(): Set<string> {
  return new Set([
    "websearch",   // Exa AI 搜索
    "context7",    // 文档查询
    "grep_app",    // GitHub 代码搜索
    // 其他系统 MCP 服务器
  ]);
}
```

### MCP 使能/禁用控制

```typescript
// MCP 禁用配置管理
const disabledMcps = new Set(pluginConfig.disabled_mcps ?? []);

function isMcpEnabled(mcpName: string): boolean {
  return !disabledMcps.has(mcpName);
}

// 在技能注册时考虑 MCP 禁用设置
const builtinSkills = createBuiltinSkills().filter((skill) => {
  if (disabledSkills.has(skill.name as never)) return false;
  if (skill.mcpConfig) {
    for (const mcpName of Object.keys(skill.mcpConfig)) {
      if (systemMcpNames.has(mcpName) || disabledMcps.has(mcpName)) {
        return false; // 如果 MCP 被禁用，则不注册此技能
      }
    }
  }
  return true;
});
```

## MCP 工具的执行流程

### 工具执行钩子

MCP 工具的执行也遵循标准的钩子系统：

```typescript
// 在工具执行前后钩子中处理 MCP 相关逻辑
"tool.execute.before": async (input, output) => {
  await claudeCodeHooks["tool.execute.before"](input, output);
  // ... 其他钩子处理
  
  // 特殊处理: 限制某些工具的嵌套调用
  if (input.tool === "task") {
    const args = output.args as Record<string, unknown>;
    const subagentType = args.subagent_type as string;
    const isExploreOrLibrarian = ["explore", "librarian"].includes(subagentType);

    // 防止无限递归调用
    args.tools = {
      ...(args.tools as Record<string, boolean> | undefined),
      sisyphus_task: false, // 禁用嵌套任务
      ...(isExploreOrLibrarian ? { call_omo_agent: false } : {}),
    };
  }
}
```

## MCP 安全机制

### 访问控制

```typescript
// MCP 访问权限控制示例
class SecureMcpManager {
  private allowedDomains = new Set([
    'exa.ai',
    'context7.com', 
    'grep.app',
    // 其他受信任的域名
  ]);
  
  async validateAndExecute(mcpName: string, operation: string, params: any) {
    // 验证 MCP 名称
    if (!this.isAllowedMcp(mcpName)) {
      throw new Error(`MCP ${mcpName} is not allowed`);
    }
    
    // 验证操作参数
    if (!this.validateParams(operation, params)) {
      throw new Error(`Invalid parameters for operation ${operation}`);
    }
    
    // 执行操作
    return await this.executeMcp(mcpName, operation, params);
  }
  
  private isAllowedMcp(mcpName: string): boolean {
    // 检查是否为系统允许的 MCP
    return this.systemMcps.has(mcpName) || this.userAllowedMcps.has(mcpName);
  }
}
```

## 故障恢复机制

### MCP 服务器重连

```typescript
// MCP 连接恢复机制
export class ResilientMcpConnection {
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 3;
  
  async executeWithRecovery(
    toolName: string, 
    args: any,
    retryCount: number = 3
  ): Promise<any> {
    for (let attempt = 0; attempt < retryCount; attempt++) {
      try {
        return await this.execute(toolName, args);
      } catch (error) {
        if (attempt === retryCount - 1) {
          // 最后一次尝试失败，尝试重新连接
          await this.reconnect();
          if (this.reconnectAttempts < this.maxReconnectAttempts) {
            this.reconnectAttempts++;
            // 重试执行
            return await this.execute(toolName, args);
          }
          throw error;
        }
        
        // 等待后重试
        await new Promise(resolve => setTimeout(resolve, 1000 * (attempt + 1)));
      }
    }
  }
  
  private async reconnect() {
    this.disconnect();
    await this.connect();
    this.reconnectAttempts = 0; // 重置重连计数
  }
}
```

## 性能优化

### MCP 请求缓存

```typescript
// MCP 结果缓存机制
class McpResultCache {
  private cache = new Map<string, { result: any, timestamp: number }>();
  private readonly TTL = 5 * 60 * 1000; // 5分钟TTL
  
  async getCachedOrExecute(
    mcpName: string,
    toolName: string, 
    args: any,
    executor: () => Promise<any>
  ): Promise<any> {
    const cacheKey = this.generateCacheKey(mcpName, toolName, args);
    
    // 检查缓存
    const cached = this.cache.get(cacheKey);
    if (cached && Date.now() - cached.timestamp < this.TTL) {
      return cached.result;
    }
    
    // 执行并缓存结果
    const result = await executor();
    this.cache.set(cacheKey, { result, timestamp: Date.now() });
    
    return result;
  }
  
  private generateCacheKey(mcpName: string, toolName: string, args: any): string {
    return `${mcpName}:${toolName}:${JSON.stringify(args)}`;
  }
}
```

## 扩展性设计

### 自定义 MCP 服务

开发者可以创建自定义 MCP 服务：

```typescript
// 自定义 MCP 服务示例
export class CustomMcpServer implements McpServer {
  async initialize() {
    // 初始化 MCP 服务器
  }
  
  async getTools() {
    return {
      tools: [{
        name: "custom_tool",
        description: "自定义工具",
        inputSchema: {
          type: "object",
          properties: {
            param1: { type: "string" }
          },
          required: ["param1"]
        }
      }]
    };
  }
  
  async execute(toolName: string, arguments: any) {
    switch (toolName) {
      case "custom_tool":
        return await this.handleCustomTool(arguments);
      default:
        throw new Error(`Unknown tool: ${toolName}`);
    }
  }
  
  private async handleCustomTool(args: any) {
    // 处理自定义工具逻辑
    return { result: "custom result" };
  }
}
```

MCP 集成系统为 oh-my-opencode 提供了强大的外部服务集成能力，使 AI 代理能够访问网络资源、官方文档和公共代码库。通过标准化的接口和灵活的配置，该系统既保证了功能的强大性，又确保了安全性和可扩展性。

下一节我们将探讨技能系统和扩展机制的设计与实现。