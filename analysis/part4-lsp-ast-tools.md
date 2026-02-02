# oh-my-opencode 源码解析 - 第四部分：LSP和AST-Grep工具系统

## LSP工具系统概述

LSP（Language Server Protocol）工具系统为 AI 代理提供了专业的代码分析和编辑能力，使代理能够像专业开发者一样理解和操作代码。

## LSP 工具列表

oh-my-opencode 提供了完整的 LSP 工具集：

1. **lsp_hover**：获取符号的类型信息、文档和签名
2. **lsp_goto_definition**：跳转到符号定义
3. **lsp_find_references**：查找符号的所有引用
4. **lsp_document_symbols**：获取文件中的所有符号
5. **lsp_workspace_symbols**：在工作区中搜索符号
6. **lsp_diagnostics**：获取错误和警告信息
7. **lsp_servers**：列出可用的 LSP 服务器
8. **lsp_prepare_rename**：准备重命名操作（验证重命名是否有效）
9. **lsp_rename**：在整个工作区重命名符号
10. **lsp_code_actions**：获取可用的快速修复和重构操作
11. **lsp_code_action_resolve**：解析并应用代码操作

## LSP 工具实现

### 工具注册机制

在 `src/tools/index.ts` 中注册所有 LSP 工具：

```typescript
// 示例 LSP 工具实现
export const lsp_goto_definition = {
  name: "lsp_goto_definition",
  description: "跳转到符号定义",
  parameters: {
    type: "object",
    properties: {
      filePath: { type: "string", description: "文件路径" },
      line: { type: "number", minimum: 1, description: "行号" },
      character: { type: "number", minimum: 0, description: "字符位置" }
    },
    required: ["filePath", "line", "character"]
  },
  handler: async (args: LspGotoDefinitionArgs, ctx: ToolContext) => {
    // 实际的 LSP 客户端实现
    const client = getLspClientForFile(args.filePath);
    const result = await client.gotoDefinition(args);
    return result;
  }
};
```

### LSP 客户端管理

LSP 客户端实现了完整的 LSP 协议：

```typescript
// LSP 客户端示例 (src/tools/lsp/client.ts)
export class LspClient {
  private connection: MessageConnection;
  private documents: TextDocuments<TextDocument>;
  
  async initialize(workspacePath: string) {
    // 初始化 LSP 连接
    const params: InitializeParams = {
      processId: process.pid,
      rootUri: URI.file(workspacePath).toString(),
      capabilities: this.getClientCapabilities(),
    };
    
    const result = await this.connection.sendRequest(InitializeRequest.type, params);
    this.capabilities = result.capabilities;
    
    // 通知服务器初始化完成
    await this.connection.sendNotification(InitializedNotification.type);
  }
  
  async gotoDefinition(params: TextDocumentPositionParams) {
    if (!this.capabilities.definitionProvider) {
      throw new Error("Language server does not support goto definition");
    }
    
    return this.connection.sendRequest(DefinitionRequest.type, params);
  }
  
  async findReferences(params: TextDocumentPositionParams) {
    if (!this.capabilities.referencesProvider) {
      throw new Error("Language server does not support find references");
    }
    
    const referenceParams: ReferenceParams = {
      ...params,
      context: { includeDeclaration: true }
    };
    
    return this.connection.sendRequest(ReferencesRequest.type, referenceParams);
  }
}
```

## AST-Grep 工具系统

### AST-Grep 概述

AST-Grep 是基于语法树的搜索和替换工具，提供比简单文本搜索更精确的代码分析能力。

### AST-Grep 工具实现

```typescript
// AST-Grep 搜索工具
export const ast_grep_search = {
  name: "ast_grep_search",
  description: "使用 AST 模式在代码库中搜索",
  parameters: {
    type: "object",
    properties: {
      pattern: { type: "string", description: "搜索模式" },
      lang: { type: "string", enum: SUPPORTED_LANGUAGES, description: "语言类型" },
      paths: { type: "array", items: { type: "string" }, description: "搜索路径" },
      globs: { type: "array", items: { type: "string" }, description: "文件匹配模式" },
      context: { type: "number", description: "上下文行数" }
    },
    required: ["pattern", "lang"]
  },
  handler: async (args: AstGrepSearchArgs, ctx: ToolContext) => {
    // 使用 @ast-grep/napi 库执行搜索
    const config = {
      pattern: args.pattern,
      language: args.lang,
      paths: args.paths || ['.'],
      include: args.globs
    };
    
    const astGrep = createAstGrep(config);
    const results = await astGrep.search();
    
    return {
      matches: results,
      count: results.length
    };
  }
};
```

### AST-Grep 模式语法

AST-Grep 支持强大的模式匹配语法：

```typescript
// 模式示例
// 1. 简单匹配
'console.log($MSG)' // 匹配 console.log 调用

// 2. 函数定义匹配
'export async function $NAME($$$) { $$$ }' // 匹配异步函数

// 3. 多节点匹配
'$$$\nimport $VAR from "$SOURCE"\n$$$' // 匹配导入语句

// 4. meta-variables 使用
// $VAR - 匹配单个 AST 节点
// $$$ - 匹配多个 AST 节点
```

## 工具集成与协调

### 工具优先级

在 OpenCode 中，oh-my-opencode 的工具会替换默认工具：

```typescript
// 在主插件中注册工具
tool: {
  ...builtinTools,           // 默认工具
  ...backgroundTools,        // 后台工具
  // LSP 工具覆盖默认实现
  lsp_hover: lspHoverTool,
  lsp_goto_definition: lspGotoDefinitionTool,
  // AST-Grep 工具
  ast_grep_search: astGrepSearchTool,
  ast_grep_replace: astGrepReplaceTool,
}
```

### 工具输出截断

为了管理上下文窗口，系统实现了输出截断功能：

```typescript
// 工具输出截断钩子
export function createToolOutputTruncatorHook(ctx: PluginInput, options?: ToolOutputTruncatorOptions) {
  return {
    "tool.execute.after": async (input, output) => {
      const toolName = input.tool;
      const result = output.result;
      
      // 检查是否需要截断
      if (shouldTruncateToolOutput(toolName)) {
        const truncated = truncateOutput(result, {
          remainingTokens: getRemainingContextWindow(ctx),
          threshold: options?.threshold || 0.5
        });
        
        output.result = truncated;
      }
    }
  };
}
```

## 特定语言支持

### 语言服务器配置

系统支持多种语言的 LSP 服务器：

```typescript
// 语言服务器配置示例
const lspServers = {
  typescript: {
    command: ["typescript-language-server", "--stdio"],
    extensions: [".ts", ".tsx"],
    priority: 10
  },
  python: {
    command: ["pylsp"],
    extensions: [".py"],
    priority: 5
  },
  rust: {
    command: ["rust-analyzer"],
    extensions: [".rs"],
    priority: 8
  }
};
```

### 语言检测

系统自动检测文件类型并选择相应的 LSP 服务器：

```typescript
function getLspServerForFile(filePath: string) {
  const extension = path.extname(filePath);
  
  for (const [lang, config] of Object.entries(lspServers)) {
    if (config.extensions.includes(extension)) {
      return getOrCreateServer(lang, config);
    }
  }
  
  // 默认服务器
  return getDefaultServer();
}
```

## 性能优化

### 服务器生命周期管理

LSP 服务器采用懒加载和缓存策略：

```typescript
class LspServerManager {
  private servers = new Map<string, LspClient>();
  private pendingInitializations = new Map<string, Promise<LspClient>>();
  
  async getServerForFile(filePath: string, config: LspServerConfig): Promise<LspClient> {
    const key = config.command.join(' ');
    
    // 如果服务器已存在，直接返回
    if (this.servers.has(key)) {
      return this.servers.get(key)!;
    }
    
    // 如果正在初始化，等待初始化完成
    if (this.pendingInitializations.has(key)) {
      return this.pendingInitializations.get(key)!;
    }
    
    // 初始化新服务器
    const initPromise = this.initializeServer(config, path.dirname(filePath));
    this.pendingInitializations.set(key, initPromise);
    
    const server = await initPromise;
    this.servers.set(key, server);
    this.pendingInitializations.delete(key);
    
    return server;
  }
}
```

### 智能缓存

对常见查询结果进行缓存：

```typescript
// 示例：符号定义缓存
const definitionCache = new Map<string, Location[]>();
const cacheKey = `${filePath}:${line}:${character}`;

if (definitionCache.has(cacheKey)) {
  return definitionCache.get(cacheKey)!;
}

const result = await lspClient.gotoDefinition(params);
definitionCache.set(cacheKey, result);

// 设置过期时间
setTimeout(() => definitionCache.delete(cacheKey), 5 * 60 * 1000); // 5分钟
```

## 错误处理

### LSP 服务器错误恢复

当 LSP 服务器出现错误时，系统会尝试恢复：

```typescript
async function robustLspRequest<T>(
  requestFn: () => Promise<T>,
  retryCount = 3
): Promise<T> {
  for (let i = 0; i < retryCount; i++) {
    try {
      return await requestFn();
    } catch (error) {
      if (i === retryCount - 1) {
        throw error; // 最后一次尝试失败，抛出错误
      }
      
      // 重启 LSP 服务器
      await restartServer();
      await delay(1000); // 等待 1 秒后重试
    }
  }
  
  throw new Error("Should not reach here");
}
```

## 实际应用场景

### 代码重构场景

通过 LSP 工具实现安全的代码重构：

```typescript
// 重命名函数示例
async function renameFunction(oldName: string, newName: string, workspace: string) {
  // 1. 查找所有引用
  const references = await lsp_find_references({
    filePath: getFunctionDefinitionFile(oldName),
    line: getFunctionDefinitionLine(oldName),
    character: getFunctionDefinitionChar(oldName)
  });
  
  // 2. 执行重命名
  await lsp_rename({
    filePath: getFunctionDefinitionFile(oldName),
    line: getFunctionDefinitionLine(oldName),
    character: getFunctionDefinitionChar(oldName),
    newName: newName
  });
  
  // 3. 验证重构结果
  const newReferences = await lsp_find_references({
    filePath: getFunctionDefinitionFile(newName),
    line: getFunctionDefinitionLine(newName),
    character: getFunctionDefinitionChar(newName)
  });
  
  return {
    success: newReferences.length === references.length,
    updatedFiles: newReferences.map(ref => ref.uri)
  };
}
```

### 代码理解场景

利用 AST-Grep 进行代码模式分析：

```typescript
// 查找所有异步函数
const asyncFunctions = await ast_grep_search({
  pattern: 'async function $NAME($$$) { $$$ }',
  lang: 'typescript',
  paths: ['.']
});

// 查找所有 useEffect 调用
const useEffectCalls = await ast_grep_search({
  pattern: 'useEffect($$$)',
  lang: 'tsx',
  paths: ['.']
});
```

LSP 和 AST-Grep 工具系统为 AI 代理提供了专业的代码分析能力，使它们能够像经验丰富的开发者一样理解和操作代码。这些工具不仅提高了代码操作的准确性，还大大增强了代理的开发能力。

下一节我们将分析 MCP（Model Context Protocol）集成系统的实现。