# oh-my-opencode 源码解析 - 第六部分：技能系统和扩展机制

## 技能系统概述

技能系统是 oh-my-opencode 的重要扩展机制，允许开发者添加特定领域的功能。技能可以包含文档说明、MCP 配置、提示模板等，为 AI 代理提供专业能力。

## 技能系统架构

### 技能定义结构

技能系统支持多种来源的技能：

1. **内置技能**：项目自带的技能（如 playwright）
2. **用户技能**：`~/.claude/skills/` 目录下的技能
3. **项目技能**：`./.claude/skills/` 目录下的技能
4. **OpenCode 全局技能**：`~/.config/opencode/command/` 目录下的技能
5. **项目 OpenCode 技能**：`./.opencode/command/` 目录下的技能

### 技能加载器

```typescript
// 技能加载器 (src/features/opencode-skill-loader/index.ts)
export async function discoverUserClaudeSkills(): Promise<Skill[]> {
  const skillsDir = path.join(os.homedir(), '.claude', 'skills');
  return await discoverSkillsInDirectory(skillsDir);
}

export async function discoverProjectClaudeSkills(): Promise<Skill[]> {
  const skillsDir = path.join(process.cwd(), '.claude', 'skills');
  return await discoverSkillsInDirectory(skillsDir);
}

export async function discoverOpencodeGlobalSkills(): Promise<Skill[]> {
  const commandsDir = path.join(os.homedir(), '.config', 'opencode', 'command');
  return await discoverCommandsInDirectory(commandsDir);
}

export async function discoverOpencodeProjectSkills(): Promise<Skill[]> {
  const commandsDir = path.join(process.cwd(), '.opencode', 'command');
  return await discoverCommandsInDirectory(commandsDir);
}

// 合并技能
export function mergeSkills(...skillGroups: Skill[][]): Skill[] {
  const skillMap = new Map<string, Skill>();
  
  for (const group of skillGroups) {
    for (const skill of group) {
      // 防止重复技能
      if (!skillMap.has(skill.name)) {
        skillMap.set(skill.name, skill);
      }
    }
  }
  
  return Array.from(skillMap.values());
}
```

## 内置技能实现

### Playwright 技能

Playwright 技能是项目内置的主要技能，用于浏览器自动化：

```typescript
// 内置技能创建 (src/features/builtin-skills/index.ts)
export function createBuiltinSkills(): Skill[] {
  return [
    createPlaywrightSkill(),
    createGitMasterSkill(),
    // 其他内置技能
  ];
}

function createPlaywrightSkill(): Skill {
  return {
    name: "playwright",
    description: "浏览器自动化、网络浏览、信息收集、网页抓取、测试、截图等",
    handler: async (args, ctx) => {
      // Playwright 技能处理逻辑
      return await handlePlaywrightSkill(args, ctx);
    },
    mcpConfig: {
      playwright: {
        command: "npx",
        args: ["-y", "@anthropic-ai/mcp-playwright"]
      }
    }
  };
}
```

### 技能 MCP 管理

```typescript
// 技能 MCP 管理器 (src/features/skill-mcp-manager/index.ts)
export class SkillMcpManager {
  private servers = new Map<string, McpServer>();
  private connections = new Map<string, McpConnection>();
  private sessionSkills = new Map<string, Set<string>>(); // 会话-技能映射
  
  async connectSkillMcp(mcpName: string, sessionID: string) {
    if (!this.servers.has(mcpName)) {
      // 创建 MCP 服务器
      const server = await this.createMcpServer(mcpName);
      this.servers.set(mcpName, server);
    }
    
    const server = this.servers.get(mcpName)!;
    const connection = await server.connect(sessionID);
    this.connections.set(`${mcpName}:${sessionID}`, connection);
    
    // 记录会话使用的技能
    if (!this.sessionSkills.has(sessionID)) {
      this.sessionSkills.set(sessionID, new Set());
    }
    this.sessionSkills.get(sessionID)!.add(mcpName);
    
    return connection;
  }
  
  async disconnectSession(sessionID: string) {
    const skills = this.sessionSkills.get(sessionID) || new Set();
    for (const skill of skills) {
      const key = `${skill}:${sessionID}`;
      const connection = this.connections.get(key);
      if (connection) {
        await connection.disconnect();
        this.connections.delete(key);
      }
    }
    this.sessionSkills.delete(sessionID);
  }
}
```

## 技能工具实现

### Skill 工具

```typescript
// Skill 工具实现 (src/tools/skill.ts)
export const skill = {
  name: "skill",
  description: "加载技能以获取特定任务的详细说明",
  parameters: {
    type: "object",
    properties: {
      name: { 
        type: "string", 
        description: "要加载的技能名称" 
      }
    },
    required: ["name"]
  },
  handler: async (args: SkillArgs, ctx: ToolContext) => {
    const { name } = args;
    
    // 查找技能
    const skill = findSkillByName(name, ctx.skills || []);
    if (!skill) {
      throw new Error(`Skill '${name}' not found`);
    }
    
    // 如果技能有 MCP 配置，连接 MCP 服务器
    if (skill.mcpConfig) {
      const sessionID = ctx.sessionId || getMainSessionID();
      if (sessionID) {
        for (const [mcpName, mcpConfig] of Object.entries(skill.mcpConfig)) {
          await skillMcpManager.connectSkillMcp(mcpName, sessionID);
        }
      }
    }
    
    // 返回技能详细信息
    return {
      name: skill.name,
      description: skill.description,
      instructions: skill.instructions || "No specific instructions provided",
      mcpConfig: skill.mcpConfig ? "MCP configuration available" : undefined,
      available: true
    };
  }
};

function findSkillByName(name: string, allSkills: Skill[]): Skill | undefined {
  return allSkills.find(skill => 
    skill.name.toLowerCase() === name.toLowerCase()
  );
}
```

### Skill MCP 工具

```typescript
// Skill MCP 工具 (src/tools/skill-mcp.ts)
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
      arguments: { type: "string", description: "工具参数(JSON字符串)" },
      grep: { type: "string", description: "搜索参数" }
    },
    required: ["mcp_name"]
  },
  handler: async (args: SkillMcpArgs, ctx: ToolContext) => {
    const { 
      mcp_name, 
      tool_name, 
      resource_name, 
      prompt_name, 
      arguments: rawArgs 
    } = args;
    
    const sessionID = ctx.sessionId || getMainSessionID();
    if (!sessionID) {
      throw new Error("No active session for MCP execution");
    }
    
    // 解析参数
    let parsedArgs = {};
    if (rawArgs) {
      try {
        parsedArgs = JSON.parse(rawArgs);
      } catch (e) {
        // 如果参数不是 JSON，则保留原始字符串
        parsedArgs = { raw: rawArgs };
      }
    }
    
    // 执行 MCP 操作
    let result;
    if (tool_name) {
      result = await skillMcpManager.execute(mcp_name, tool_name, parsedArgs);
    } else if (resource_name) {
      result = await skillMcpManager.getResource(mcp_name, resource_name);
    } else if (prompt_name) {
      result = await skillMcpManager.getPrompt(mcp_name, prompt_name, parsedArgs);
    } else {
      throw new Error("Must specify tool_name, resource_name, or prompt_name");
    }
    
    return result;
  }
};
```

## 技能发现与注册机制

### 技能发现流程

```typescript
// 主插件中的技能注册流程 (src/index.ts)
const disabledSkills = new Set(pluginConfig.disabled_skills ?? []);
const systemMcpNames = getSystemMcpServerNames();

// 过滤禁用的技能
const builtinSkills = createBuiltinSkills().filter((skill) => {
  // 检查技能是否被禁用
  if (disabledSkills.has(skill.name as never)) return false;
  
  // 检查 MCP 配置是否与系统 MCP 冲突
  if (skill.mcpConfig) {
    for (const mcpName of Object.keys(skill.mcpConfig)) {
      if (systemMcpNames.has(mcpName)) return false;
    }
  }
  return true;
});

// 并行发现不同来源的技能
const includeClaudeSkills = pluginConfig.claude_code?.skills !== false;
const [userSkills, globalSkills, projectSkills, opencodeProjectSkills] = await Promise.all([
  includeClaudeSkills ? discoverUserClaudeSkills() : Promise.resolve([]),
  discoverOpencodeGlobalSkills(),
  includeClaudeSkills ? discoverProjectClaudeSkills() : Promise.resolve([]),
  discoverOpencodeProjectSkills(),
]);

// 合并所有技能
const mergedSkills = mergeSkills(
  builtinSkills,
  pluginConfig.skills, // 用户配置的自定义技能
  userSkills,
  globalSkills,
  projectSkills,
  opencodeProjectSkills
);
```

## 技能文件格式

### 技能定义文件 (SKILL.md)

```markdown
---
# 技能元数据
name: "example-skill"
description: "这是一个示例技能"
version: "1.0.0"
author: "Developer"

# MCP 配置（可选）
mcp:
  example-mcp:
    command: "node"
    args: ["./mcp-server.js"]
    env:
      API_KEY: "${EXAMPLE_API_KEY}"
---

# 示例技能

这是一个示例技能的说明文档。

## 功能
- 功能1
- 功能2

## 使用方法
要使用此技能，请按以下方式操作：
1. 第一步
2. 第二步
```

### 命令文件格式

```markdown
---
name: "example-command"
description: "示例命令"
---

# 示例命令

这是一个示例命令的实现。
```

## 技能执行流程

### 会话生命周期管理

```typescript
// 在事件处理中管理技能连接 (src/index.ts)
event: async (input) => {
  // ... 其他事件处理
  
  if (event.type === "session.created") {
    const sessionInfo = props?.info as
      | { id?: string; title?: string; parentID?: string }
      | undefined;
    if (!sessionInfo?.parentID) {
      setMainSession(sessionInfo?.id);
    }
  }

  if (event.type === "session.deleted") {
    const sessionInfo = props?.info as { id?: string } | undefined;
    if (sessionInfo?.id === getMainSessionID()) {
      setMainSession(undefined);
    }
    if (sessionInfo?.id) {
      // 断开会话相关的 MCP 连接
      await skillMcpManager.disconnectSession(sessionInfo.id);
    }
  }
}
```

### 技能工具执行前处理

```typescript
"tool.execute.before": async (input, output) => {
  // ... 其他处理
  
  // 处理技能工具调用
  if (input.tool === "skill") {
    const args = output.args as { name: string };
    const skill = findSkillByName(args.name, mergedSkills);
    
    if (skill && skill.mcpConfig) {
      // 在工具执行前连接 MCP 服务器
      const sessionID = input.sessionID || getMainSessionID();
      if (sessionID) {
        for (const [mcpName, _] of Object.entries(skill.mcpConfig)) {
          await skillMcpManager.connectSkillMcp(mcpName, sessionID);
        }
      }
    }
  }
}
```

## 配置与控制

### 技能禁用配置

```typescript
// 配置示例
{
  "disabled_skills": ["playwright", "git-master"],
  "claude_code": {
    "skills": false  // 完全禁用 Claude Code 技能系统
  }
}
```

### 运行时技能管理

```typescript
// 动态技能管理
class SkillManager {
  private skills: Map<string, Skill> = new Map();
  private disabledSkills: Set<string>;
  
  constructor(config: PluginConfig) {
    this.disabledSkills = new Set(config.disabled_skills || []);
  }
  
  addSkill(skill: Skill): void {
    if (this.disabledSkills.has(skill.name)) {
      console.log(`Skill ${skill.name} is disabled, skipping registration`);
      return;
    }
    
    this.skills.set(skill.name, skill);
  }
  
  getSkill(name: string): Skill | undefined {
    return this.skills.get(name);
  }
  
  executeSkill(name: string, args: any, ctx: SkillContext): Promise<any> {
    const skill = this.getSkill(name);
    if (!skill) {
      throw new Error(`Skill ${name} not found`);
    }
    
    return skill.handler(args, ctx);
  }
}
```

## 扩展机制

### 自定义技能开发

开发者可以创建自定义技能：

```typescript
// 自定义技能示例
export const customSkill: Skill = {
  name: "custom-analysis",
  description: "代码分析和质量检查技能",
  handler: async (args: CustomAnalysisArgs, ctx: SkillContext) => {
    const { filePath, rules } = args;
    
    // 执行自定义分析逻辑
    const fileContent = await ctx.tools.read(filePath);
    const analysisResults = await performCustomAnalysis(fileContent, rules);
    
    return {
      results: analysisResults,
      summary: generateSummary(analysisResults),
      recommendations: getRecommendations(analysisResults)
    };
  },
  mcpConfig: {
    // 如果需要外部服务，定义 MCP 配置
    customAnalyzer: {
      command: "node",
      args: ["./analyzer-mcp.js"]
    }
  }
};
```

### 技能热加载

```typescript
// 技能热加载机制
export class HotReloadSkillManager {
  private watcher: FSWatcher | null = null;
  
  async enableHotReload(skillsPath: string) {
    this.watcher = chokidar.watch([
      path.join(skillsPath, "**/*.md"),
      path.join(skillsPath, "**/*.mdx")
    ], {
      persistent: true,
      ignoreInitial: true
    });
    
    this.watcher.on('add', (filePath) => this.reloadSkill(filePath));
    this.watcher.on('change', (filePath) => this.reloadSkill(filePath));
    this.watcher.on('unlink', (filePath) => this.removeSkill(filePath));
  }
  
  private async reloadSkill(filePath: string) {
    try {
      const skill = await this.loadSkillFromFile(filePath);
      this.skillManager.addSkill(skill);
      console.log(`Skill reloaded: ${skill.name}`);
    } catch (error) {
      console.error(`Failed to reload skill from ${filePath}:`, error);
    }
  }
}
```

## 技能安全机制

### 权限控制

```typescript
// 技能权限验证
export async function validateSkillExecution(
  skillName: string, 
  args: any, 
  userPermissions: UserPermissions
): Promise<boolean> {
  // 检查技能是否存在
  const skill = findSkillByName(skillName, availableSkills);
  if (!skill) {
    return false;
  }
  
  // 检查权限配置
  const requiredPermissions = skill.requiredPermissions || [];
  for (const perm of requiredPermissions) {
    if (!userPermissions[perm]) {
      throw new Error(`Permission denied: ${perm}`);
    }
  }
  
  // 验证参数安全性
  if (containsUnsafePath(args)) {
    throw new Error("Unsafe file path detected in skill arguments");
  }
  
  return true;
}

function containsUnsafePath(args: any): boolean {
  // 检查是否存在路径遍历等不安全路径
  const jsonString = JSON.stringify(args);
  return jsonString.includes('../') || jsonString.includes('..\\');
}
```

技能系统通过灵活的扩展机制和标准化的接口设计，为 oh-my-opencode 提供了强大的功能扩展能力。开发者可以轻松添加新技能，为 AI 代理提供特定领域的专业知识和工具能力。

下一节我们将分析配置管理系统和动态加载机制的实现。