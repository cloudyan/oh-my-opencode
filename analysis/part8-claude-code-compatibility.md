# oh-my-opencode 源码解析 - 第八部分：Claude Code 兼容层

## Claude Code 兼容层概述

Claude Code 兼容层是 oh-my-opencode 的一个重要特性，它允许 Claude Code 用户无缝迁移到 OpenCode 平台。该兼容层实现了 Claude Code 的核心功能，包括 Hooks、命令、技能、代理和 MCP。

## 兼容层架构设计

### 兼容层组件结构

```
Claude Code 兼容层
├── Hooks 兼容 (PreToolUse, PostToolUse, UserPromptSubmit, Stop)
├── 命令加载器 (Commands)
├── 技能加载器 (Skills) 
├── 代理加载器 (Agents)
├── MCP 加载器 (MCPs)
└── 设置解析器 (Settings)
```

### 主要实现文件

- `src/hooks/claude-code-hooks/index.ts` - Claude Code 钩子系统
- `src/features/claude-code-*` - 各种 Claude Code 加载器
- `src/features/claude-code-session-state.ts` - 会话状态管理

## Hooks 兼容实现

### Claude Code Hooks 系统

```typescript
// Claude Code Hooks 实现 (src/hooks/claude-code-hooks/index.ts)
export function createClaudeCodeHooksHook(
  ctx: PluginInput, 
  options: ClaudeCodeHooksOptions,
  contextCollector: ContextCollector
) {
  // 加载 Claude Code 设置
  const ccSettings = loadClaudeCodeSettings(ctx.directory);
  
  return {
    "chat.message": async (input: ChatMessageInput, output: ChatMessageOutput) => {
      if (options.disabledHooks) return;
      
      // 处理 UserPromptSubmit 类型的钩子
      const userPromptSubmitHooks = ccSettings.hooks?.UserPromptSubmit || [];
      for (const hook of userPromptSubmitHooks) {
        await executeClaudeCodeHook(hook, input, output, ctx);
      }
    },
    
    "tool.execute.before": async (input: ToolExecuteBeforeInput, output: ToolExecuteBeforeOutput) => {
      if (options.disabledHooks) return;
      
      // 处理 PreToolUse 类型的钩子
      const preToolUseHooks = ccSettings.hooks?.PreToolUse || [];
      for (const hook of preToolUseHooks) {
        if (shouldExecuteHook(hook.matcher, input.tool)) {
          await executeClaudeCodeHook(hook, input, output, ctx);
        }
      }
    },
    
    "tool.execute.after": async (input: ToolExecuteAfterInput, output: ToolExecuteAfterOutput) => {
      if (options.disabledHooks) return;
      
      // 处理 PostToolUse 类型的钩子
      const postToolUseHooks = ccSettings.hooks?.PostToolUse || [];
      for (const hook of postToolUseHooks) {
        if (shouldExecuteHook(hook.matcher, input.tool)) {
          await executeClaudeCodeHook(hook, input, output, ctx);
        }
      }
    },
    
    event: async (eventInput: EventInput) => {
      if (options.disabledHooks) return;
      
      // 处理 Stop 类型的钩子
      if (eventInput.event.type === "session.idle") {
        const stopHooks = ccSettings.hooks?.Stop || [];
        for (const hook of stopHooks) {
          await executeClaudeCodeHook(hook, eventInput, null, ctx);
        }
      }
    }
  };
}
```

### 钩子执行器

```typescript
// Claude Code 钩子执行器
async function executeClaudeCodeHook(
  hook: ClaudeCodeHook,
  input: any,
  output: any | null,
  ctx: PluginInput
): Promise<void> {
  for (const hookItem of hook.hooks || []) {
    switch (hookItem.type) {
      case "command":
        await executeCommandHook(hookItem, input, ctx);
        break;
      case "template":
        await executeTemplateHook(hookItem, input, output, ctx);
        break;
      case "message":
        await executeMessageHook(hookItem, ctx);
        break;
      default:
        console.warn(`Unknown hook type: ${(hookItem as any).type}`);
    }
  }
}

async function executeCommandHook(
  hook: ClaudeCodeCommandHook,
  input: any,
  ctx: PluginInput
): Promise<void> {
  // 将钩子中的变量替换为实际值
  const commandTemplate = hook.command;
  const command = replaceVariables(commandTemplate, {
    FILE: input.args?.filePath || "",
    TOOL: input.tool || "",
    RESULT: input.result || "",
    SESSION_DIR: ctx.directory
  });
  
  try {
    const result = await executeShellCommand(command, ctx.directory);
    
    // 如果钩子需要结果，可以将结果注入到上下文中
    if (hook.capture) {
      // 将命令结果注入到当前会话
      await ctx.client.session.message({
        path: { id: input.sessionID },
        body: {
          parts: [{
            type: "text",
            text: `Hook command result: ${result}`
          }]
        }
      });
    }
  } catch (error) {
    console.error(`Hook command failed: ${command}`, error);
  }
}

function shouldExecuteHook(matcher: string, toolName: string): boolean {
  if (!matcher) return true;
  
  // 支持正则表达式匹配
  try {
    const regex = new RegExp(matcher);
    return regex.test(toolName);
  } catch {
    // 如果不是正则，使用简单字符串包含匹配
    return toolName.includes(matcher);
  }
}
```

## 命令加载器

### Claude Code 命令系统

```typescript
// 命令加载器 (src/features/claude-code-command-loader/index.ts)
export function discoverCommandsSync(): ClaudeCodeCommand[] {
  const possibleDirs = [
    path.join(os.homedir(), '.claude', 'commands'),
    path.join(process.cwd(), '.claude', 'commands'),
    path.join(os.homedir(), '.config', 'opencode', 'command'),
    path.join(process.cwd(), '.opencode', 'command')
  ];
  
  const commands: ClaudeCodeCommand[] = [];
  
  for (const dir of possibleDirs) {
    if (!fs.existsSync(dir)) continue;
    
    const files = fs.readdirSync(dir);
    for (const file of files) {
      if (!file.endsWith('.md')) continue;
      
      const filePath = path.join(dir, file);
      try {
        const content = fs.readFileSync(filePath, 'utf-8');
        const command = parseClaudeCodeCommand(content, filePath);
        if (command) {
          commands.push(command);
        }
      } catch (error) {
        console.error(`Failed to parse command file ${filePath}:`, error);
      }
    }
  }
  
  return commands;
}

function parseClaudeCodeCommand(content: string, filePath: string): ClaudeCodeCommand | null {
  // 解析 Markdown 文件，提取 frontmatter 和内容
  const { data, content: commandContent } = matter(content);
  
  // 从 frontmatter 中提取元数据
  const commandName = data.name || path.basename(filePath, '.md');
  const description = data.description || extractDescription(commandContent);
  
  return {
    name: commandName,
    description,
    content: commandContent,
    filePath,
    metadata: data
  };
}

function extractDescription(content: string): string {
  // 从 Markdown 内容中提取描述
  const lines = content.split('\n');
  for (const line of lines) {
    const trimmed = line.trim();
    if (trimmed.startsWith('#')) continue; // 跳过标题行
    if (trimmed) return trimmed.substring(0, 100) + (trimmed.length > 100 ? '...' : '');
  }
  return 'No description provided';
}
```

### Slash Command 工具

```typescript
// Slash Command 工具 (src/tools/slashcommand.ts)
export const slashcommand = {
  name: "slashcommand",
  description: "执行内置命令或加载技能",
  parameters: {
    type: "object",
    properties: {
      command: { type: "string", description: "命令名称" }
    },
    required: ["command"]
  },
  handler: async (args: SlashCommandArgs, ctx: ToolContext) => {
    const commandText = args.command.trim();
    const commandName = commandText.split(' ')[0].substring(1); // 移除 '/' 前缀
    const commandArgs = commandText.substring(commandName.length + 2); // 获取参数
    
    // 检查是否是内置命令
    const builtinCommand = findBuiltinCommand(commandName);
    if (builtinCommand) {
      return await builtinCommand.handler(commandArgs, ctx);
    }
    
    // 检查 Claude Code 命令
    const ccCommand = findClaudeCodeCommand(commandName, ctx.skills || []);
    if (ccCommand) {
      return await executeClaudeCodeCommand(ccCommand, commandArgs, ctx);
    }
    
    // 检查技能
    const skill = findSkillByName(commandName, ctx.skills || []);
    if (skill) {
      return await handleSkillCommand(skill, commandArgs, ctx);
    }
    
    throw new Error(`Command '${commandName}' not found`);
  }
};

// 自动 Slash Command 钩子 (src/hooks/auto-slash-command/index.ts)
export function createAutoSlashCommandHook(options?: AutoSlashCommandHookOptions) {
  return {
    "chat.message": async (input: ChatMessageInput, output: ChatMessageOutput) => {
      const messageText = extractTextFromMessage(output);
      
      // 检查是否包含命令模式（如 /command 或 @skill）
      const commandMatches = messageText.match(/\/([a-zA-Z0-9_-]+)/g);
      const skillMatches = messageText.match(/@([a-zA-Z0-9_-]+)/g);
      
      if (commandMatches || skillMatches) {
        // 自动执行命令
        for (const match of commandMatches || []) {
          const command = match.substring(1);
          await executeAutoCommand(command, input.sessionID);
        }
      }
    }
  };
}
```

## 技能加载器

### Claude Code 技能系统

```typescript
// Claude Code 技能加载器 (src/features/claude-code-skill-loader/index.ts)
export async function discoverClaudeSkills(baseDir: string): Promise<ClaudeCodeSkill[]> {
  const skillsDir = path.join(baseDir, '.claude', 'skills');
  if (!fs.existsSync(skillsDir)) {
    return [];
  }
  
  const skills: ClaudeCodeSkill[] = [];
  const dirs = fs.readdirSync(skillsDir, { withFileTypes: true });
  
  for (const dir of dirs) {
    if (!dir.isDirectory()) continue;
    
    const skillDir = path.join(skillsDir, dir.name);
    const skillFile = path.join(skillDir, 'SKILL.md');
    
    if (!fs.existsSync(skillFile)) continue;
    
    try {
      const content = fs.readFileSync(skillFile, 'utf-8');
      const skill = parseClaudeSkill(content, skillDir);
      if (skill) {
        skills.push(skill);
      }
    } catch (error) {
      console.error(`Failed to parse skill ${dir.name}:`, error);
    }
  }
  
  return skills;
}

function parseClaudeSkill(content: string, skillDir: string): ClaudeCodeSkill | null {
  const { data: frontmatter, content: skillContent } = matter(content);
  
  // 解析 MCP 配置（如果存在）
  let mcpConfig = null;
  if (frontmatter.mcp) {
    mcpConfig = parseMcpConfig(frontmatter.mcp, skillDir);
  }
  
  return {
    name: frontmatter.name || path.basename(skillDir),
    description: frontmatter.description || extractDescription(skillContent),
    instructions: skillContent,
    mcpConfig,
    directory: skillDir,
    metadata: frontmatter
  };
}

function parseMcpConfig(mcpData: any, skillDir: string): McpConfig | null {
  if (!mcpData || typeof mcpData !== 'object') return null;
  
  const parsedConfig: McpConfig = {};
  
  for (const [mcpName, mcpSpec] of Object.entries(mcpData)) {
    if (typeof mcpSpec === 'object' && mcpSpec.command) {
      parsedConfig[mcpName] = {
        command: mcpSpec.command,
        args: mcpSpec.args || [],
        env: resolveEnvVars(mcpSpec.env || {}, skillDir),
        port: mcpSpec.port
      };
    }
  }
  
  return parsedConfig;
}

function resolveEnvVars(env: Record<string, string>, skillDir: string): Record<string, string> {
  const resolved = { ...env };
  
  for (const [key, value] of Object.entries(env)) {
    // 支持 ${VAR} 语法
    if (typeof value === 'string' && value.includes('${')) {
      resolved[key] = value.replace(/\$\{([^}]+)\}/g, (match, varName) => {
        return process.env[varName] || match;
      });
    }
  }
  
  // 添加技能目录到环境变量
  resolved.SKILL_DIR = skillDir;
  
  return resolved;
}
```

## MCP 加载器

### Claude Code MCP 系统

```typescript
// Claude Code MCP 加载器 (src/features/claude-code-mcp-loader/index.ts)
export function loadClaudeCodeMcpConfig(baseDir: string): McpServerConfig[] {
  const possibleFiles = [
    path.join(baseDir, '.claude', '.mcp.json'),
    path.join(baseDir, '.mcp.json'),
    path.join(baseDir, '.claude', '.mcp.jsonc')
  ];
  
  const configs: McpServerConfig[] = [];
  
  for (const file of possibleFiles) {
    if (!fs.existsSync(file)) continue;
    
    try {
      const content = fs.readFileSync(file, 'utf-8');
      const config = file.endsWith('.jsonc') ? parseJsonc(content) : JSON.parse(content);
      
      if (config.servers && Array.isArray(config.servers)) {
        configs.push(...config.servers);
      } else if (config.server) {
        configs.push(config.server);
      }
    } catch (error) {
      console.error(`Failed to load MCP config from ${file}:`, error);
    }
  }
  
  return configs;
}

export function getSystemMcpServerNames(): Set<string> {
  return new Set([
    "websearch",   // Exa AI 搜索
    "context7",    // 文档查询  
    "grep_app",    // GitHub 代码搜索
    // 可能还有其他系统 MCP 服务器
  ]);
}
```

### MCP 服务器管理

```typescript
// MCP 服务器管理 (src/features/skill-mcp-manager/index.ts)
export class ClaudeCodeMcpManager {
  private servers: Map<string, ChildProcess> = new Map();
  private connections: Map<string, any> = new Map();
  
  async startMcpServer(config: McpServerConfig): Promise<void> {
    const { command, args, env = {}, port } = config;
    
    // 设置 MCP 相关环境变量
    const mcpEnv = {
      ...process.env,
      ...env,
      MCP_PORT: port?.toString() || getRandomPort().toString(),
      MCP_SERVER_NAME: config.name
    };
    
    const serverProcess = spawn(command, args, {
      env: mcpEnv,
      cwd: config.cwd || process.cwd(),
      stdio: ['pipe', 'pipe', 'pipe']
    });
    
    // 保存服务器引用
    this.servers.set(config.name, serverProcess);
    
    // 监听服务器输出
    serverProcess.stdout?.on('data', (data) => {
      console.log(`MCP ${config.name} stdout:`, data.toString());
    });
    
    serverProcess.stderr?.on('data', (data) => {
      console.error(`MCP ${config.name} stderr:`, data.toString());
    });
    
    serverProcess.on('error', (error) => {
      console.error(`MCP ${config.name} error:`, error);
    });
    
    serverProcess.on('close', (code) => {
      console.log(`MCP ${config.name} closed with code ${code}`);
      this.servers.delete(config.name);
    });
  }
  
  async executeMcpTool(
    serverName: string, 
    toolName: string, 
    args: any
  ): Promise<any> {
    const server = this.servers.get(serverName);
    if (!server) {
      throw new Error(`MCP server ${serverName} not found`);
    }
    
    // 通过标准输入输出与 MCP 服务器通信
    // 这里需要实现 MCP 协议的具体细节
    
    return new Promise((resolve, reject) => {
      const request = {
        jsonrpc: "2.0",
        method: "tools/" + toolName,
        params: args,
        id: generateRequestId()
      };
      
      server.stdin?.write(JSON.stringify(request) + '\n');
      
      // 监听响应
      server.stdout?.on('data', (data) => {
        try {
          const response = JSON.parse(data.toString());
          if (response.id === request.id) {
            if (response.error) {
              reject(new Error(response.error.message));
            } else {
              resolve(response.result);
            }
          }
        } catch (e) {
          reject(e);
        }
      });
    });
  }
}
```

## 设置加载与解析

### Claude Code 设置系统

```typescript
// Claude Code 设置加载器
export function loadClaudeCodeSettings(directory: string): ClaudeCodeSettings {
  const possiblePaths = [
    path.join(os.homedir(), '.claude', 'settings.json'),
    path.join(directory, '.claude', 'settings.json'),
    path.join(directory, '.claude', 'settings.local.json')
  ];
  
  let settings: ClaudeCodeSettings = {
    hooks: {}
  };
  
  for (const settingsPath of possiblePaths) {
    if (fs.existsSync(settingsPath)) {
      try {
        const content = fs.readFileSync(settingsPath, 'utf-8');
        const fileSettings = JSON.parse(content);
        
        // 深度合并设置
        settings = deepMerge(settings, fileSettings);
      } catch (error) {
        console.error(`Failed to load settings from ${settingsPath}:`, error);
      }
    }
  }
  
  return settings;
}

// 深度合并函数
function deepMerge(target: any, source: any): any {
  const result = { ...target };
  
  for (const key in source) {
    if (source.hasOwnProperty(key)) {
      if (
        typeof source[key] === 'object' && 
        source[key] !== null && 
        !Array.isArray(source[key]) &&
        typeof target[key] === 'object' && 
        target[key] !== null &&
        !Array.isArray(target[key])
      ) {
        result[key] = deepMerge(target[key], source[key]);
      } else {
        result[key] = source[key];
      }
    }
  }
  
  return result;
}
```

## 会话状态管理

### Claude Code 会话状态

```typescript
// Claude Code 会话状态管理 (src/features/claude-code-session-state.ts)
let mainSessionID: string | undefined;

export function setMainSession(sessionID: string | undefined): void {
  mainSessionID = sessionID;
}

export function getMainSessionID(): string | undefined {
  return mainSessionID;
}

// 会话创建时的处理
// 在主插件的 event 处理器中：
if (event.type === "session.created") {
  const sessionInfo = props?.info as
    | { id?: string; title?: string; parentID?: string }
    | undefined;
  if (!sessionInfo?.parentID) {
    // 这是主会话（非子会话）
    setMainSession(sessionInfo?.id);
  }
}

// 会话删除时的处理
if (event.type === "session.deleted") {
  const sessionInfo = props?.info as { id?: string } | undefined;
  if (sessionInfo?.id === getMainSessionID()) {
    setMainSession(undefined);
  }
  if (sessionInfo?.id) {
    // 断开与此会话相关的 MCP 连接
    await skillMcpManager.disconnectSession(sessionInfo.id);
  }
}
```

## 兼容性开关

### 功能启用/禁用控制

```typescript
// Claude Code 功能兼容性开关
export function applyClaudeCodeCompatibility(
  config: OhMyOpenCodeConfig,
  ctx: PluginInput
): ClaudeCodeEnabledFeatures {
  const ccConfig = config.claude_code || {};
  
  return {
    mcp: ccConfig.mcp !== false,
    commands: ccConfig.commands !== false, 
    skills: ccConfig.skills !== false,
    agents: ccConfig.agents !== false,
    hooks: ccConfig.hooks !== false,
    plugins: ccConfig.plugins !== false,
    plugins_override: ccConfig.plugins_override || {}
  };
}

// 在主插件中根据兼容性开关决定是否加载功能
const includeClaudeSkills = pluginConfig.claude_code?.skills !== false;
const [userSkills, globalSkills, projectSkills, opencodeProjectSkills] = await Promise.all([
  includeClaudeSkills ? discoverUserClaudeSkills() : Promise.resolve([]),
  discoverOpencodeGlobalSkills(),
  includeClaudeSkills ? discoverProjectClaudeSkills() : Promise.resolve([]),
  discoverOpencodeProjectSkills(),
]);
```

## 特殊处理场景

### 覆盖与冲突处理

```typescript
// 处理功能覆盖和冲突
export function resolveFeatureConflicts(
  config: OhMyOpenCodeConfig,
  builtinFeatures: BuiltinFeatures
): ResolvedFeatures {
  const resolved: ResolvedFeatures = {
    ...builtinFeatures,
    hooks: { ...builtinFeatures.hooks },
    skills: [...builtinFeatures.skills],
    mcps: [...builtinFeatures.mcps]
  };
  
  // 应用用户覆盖
  if (config.disabled_hooks) {
    for (const hookName of config.disabled_hooks) {
      delete resolved.hooks[hookName];
    }
  }
  
  if (config.disabled_skills) {
    resolved.skills = resolved.skills.filter(
      skill => !config.disabled_skills?.includes(skill.name)
    );
  }
  
  if (config.disabled_mcps) {
    resolved.mcps = resolved.mcps.filter(
      mcp => !config.disabled_mcps?.includes(mcp.name)
    );
  }
  
  // 检查 Claude Code 插件覆盖
  const pluginOverrides = config.claude_code?.plugins_override || {};
  for (const [pluginName, enabled] of Object.entries(pluginOverrides)) {
    if (!enabled) {
      // 禁用特定 Claude Code 插件
      resolved.disabledPlugins.add(pluginName);
    }
  }
  
  return resolved;
}
```

## 性能优化

### 延迟加载与缓存

```typescript
// Claude Code 功能的延迟加载
class ClaudeCodeFeatureLoader {
  private settingsCache: Map<string, ClaudeCodeSettings> = new Map();
  private commandsCache: ClaudeCodeCommand[] | null = null;
  private skillsCache: ClaudeCodeSkill[] | null = null;
  
  async loadSettings(directory: string): Promise<ClaudeCodeSettings> {
    const cacheKey = directory;
    if (this.settingsCache.has(cacheKey)) {
      return this.settingsCache.get(cacheKey)!;
    }
    
    const settings = loadClaudeCodeSettings(directory);
    this.settingsCache.set(cacheKey, settings);
    return settings;
  }
  
  async loadCommands(): Promise<ClaudeCodeCommand[]> {
    if (this.commandsCache) {
      return this.commandsCache;
    }
    
    this.commandsCache = discoverCommandsSync();
    return this.commandsCache;
  }
  
  async loadSkills(directory: string): Promise<ClaudeCodeSkill[]> {
    if (this.skillsCache) {
      return this.skillsCache;
    }
    
    const [userSkills, projectSkills] = await Promise.all([
      this.loadClaudeSkills(path.join(os.homedir(), '.claude')),
      this.loadClaudeSkills(path.join(directory, '.claude'))
    ]);
    
    this.skillsCache = [...userSkills, ...projectSkills];
    return this.skillsCache;
  }
  
  private async loadClaudeSkills(baseDir: string): Promise<ClaudeCodeSkill[]> {
    return await discoverClaudeSkills(baseDir);
  }
}
```

## 错误处理与恢复

### 兼容性错误处理

```typescript
// Claude Code 兼容性错误处理
export class ClaudeCodeCompatibilityErrorHandler {
  static handleSettingError(error: Error, filePath: string): void {
    console.error(`Error loading Claude Code settings from ${filePath}:`, error);
    
    // 记录错误但不中断插件加载
    // 可能的错误处理策略：
    // 1. 使用默认设置
    // 2. 跳过有问题的设置
    // 3. 生成错误报告
  }
  
  static handleHookError(error: Error, hookConfig: any): void {
    console.error('Error executing Claude Code hook:', error);
    console.log('Hook config:', hookConfig);
    
    // 记录失败的钩子，可以提供诊断信息
    // 但不应中断主要流程
  }
  
  static reportCompatibilityIssues(settings: ClaudeCodeSettings): string[] {
    const issues: string[] = [];
    
    // 检查不兼容的设置
    if (settings.experimental?.unsupported_feature) {
      issues.push("Using unsupported experimental feature 'unsupported_feature'");
    }
    
    // 检查过时的钩子配置
    if (settings.hooks?.["legacy-hook-type"]) {
      issues.push("Using deprecated hook type 'legacy-hook-type'");
    }
    
    return issues;
  }
}
```

Claude Code 兼容层通过全面实现 Claude Code 的核心功能，为现有用户提供了无缝的迁移路径。该兼容层不仅保持了原有功能的完整性，还通过 oh-my-opencode 的增强功能提供了更好的性能和体验。

通过精心设计的架构，兼容层实现了功能的模块化处理，确保不同组件间的解耦和可维护性，同时提供了灵活的配置选项来满足不同用户的迁移需求。

## 总结

oh-my-opencode 的 Claude Code 兼容层是整个项目架构中的重要组成部分，它体现了项目"渐进式增强"的设计理念。通过兼容层，新用户可以快速上手，而 Claude Code 老用户则可以无缝迁移到 OpenCode 平台，同时享受 oh-my-opencode 提供的增强功能。