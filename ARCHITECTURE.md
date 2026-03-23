# Nanobot 系统架构文档

## 项目概述

Nanobot 是一个用 Go 编写的 **MCP (Model Context Protocol) 主机服务**，它将 MCP 服务器与大语言模型结合，为 AI 代理提供工具调用、资源访问和提示管理能力。

## 技术栈

| 层级 | 技术 |
|------|------|
| 后端 | Go 1.26.0 |
| 前端 | Svelte 5 + TypeScript |
| HTTP | 标准库 (net/http) |
| 数据库 | GORM (SQLite/MySQL/PostgreSQL) |
| CLI | Cobra |
| 日志 | slog (Go 1.21+) |
| LLM | OpenAI Responses/Completions API, Anthropic Messages API |

## 项目结构

```
/Users/bytedance/Downloads/workspace-go/nanobot/
├── main.go                    # 应用入口点
├── go.mod                     # Go 模块定义
├── Makefile                   # 构建配置
├── api/                       # API 处理相关
│   └── routes.go             # API 路由定义
├── pkg/                       # 核心代码包
│   ├── agents/               # 代理执行引擎
│   ├── api/                  # HTTP API 路由
│   ├── cli/                  # CLI 命令行
│   ├── config/               # 配置加载
│   ├── hooks/                # 生命周期钩子
│   ├── llm/                  # LLM 客户端抽象
│   ├── logging/              # 日志系统
│   ├── mcp/                  # MCP 协议实现
│   ├── sampling/             # MCP 采样处理
│   ├── schema/               # Schema 定义
│   ├── servers/              # MCP 服务器实现
│   ├── session/              # 会话管理
│   ├── supervisor/           # 进程监管
│   ├── tools/                # 工具服务
│   └── types/                # 核心类型定义
├── ui/                        # 前端 UI (Svelte 5)
└── builtin/                   # 内置配置
    ├── agents/               # 内置代理定义
    └── servers/              # 内置 MCP 服务器
```

## 核心模块

| 模块 | 职责 |
|------|------|
| pkg/runtime | 核心运行时初始化，协调 LLM、工具、代理、采样组件 |
| pkg/server | HTTP MCP 协议处理器，处理 initialize、tools/call 等端点 |
| pkg/agents | 代理执行引擎，处理工具映射、请求填充、对话压缩 |
| pkg/tools | 工具注册中心，管理 MCP 服务器连接和工具发现 |
| pkg/mcp | MCP 协议类型定义 (InitializeRequest, ServerCapabilities, Content, Tool, Resource, Prompt) |
| pkg/llm | LLM 抽象层，支持 OpenAI 和 Anthropic |
| pkg/session | 数据库会话存储，支持父子会话关系 |
| pkg/servers | 内置 MCP 服务器实现 |

## 系统架构图

```
┌─────────────────────────────────────────┐
│           CLI (pkg/cli)                 │
│              Cobra 命令行               │
└─────────────────┬───────────────────────┘
                  ▼
┌─────────────────────────────────────────┐
│        Server (pkg/server)              │
│   HTTP MCP 端点: /mcp/v1/*              │
└─────────────────┬───────────────────────┘
                  ▼
┌─────────────────────────────────────────┐
│       Runtime (pkg/runtime)             │
│   初始化 LLM、Tools、Agents、Sampling    │
└─────────────────┬───────────────────────┘
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
┌───────────┐ ┌────────┐ ┌──────────┐
│ LLM Client│ │  Tools │ │ Sampling │
└─────┬─────┘ └───┬────┘ └──────────┘
      │           │
      │     ┌─────┴─────┐
      │     ▼           ▼
      │  ┌────────┐ ┌──────────────┐
      │  │ Agents │ │ MCP Servers   │
      │  └───┬────┘ └───────┬──────┘
      │      │              │
      │      └──────┬───────┘
      │             ▼
      │     ┌──────────────┐
      │     │  MCP Client  │
      │     └──────┬───────┘
      │            ▼
      ▼    ┌─────────────────┐
            │ External MCP Servers │
            └─────────────────┘
```

## 内置 MCP 服务器

| 服务器 | 位置 | 工具 |
|--------|------|------|
| system | pkg/servers/system/ | bash, read, write, edit, glob, grep, webFetch, todoWrite, askUserQuestion |
| agent | pkg/servers/agent/ | agent 代理调用 |
| meta | pkg/servers/meta/ | 元操作工具 |
| workflows | pkg/servers/workflows/ | 工作流工具 |
| artifacts | pkg/servers/artifacts/ | 工件工具 |
| skills | pkg/servers/skills/ | 技能工具 |

## 配置结构

```yaml
agents:
  - name: shopping-assistant
    instructions: "你是购物助手..."
    model: anthropic
    tools:
      - server:brave-web-search  # server:tool 格式
      - server:filesystem
    hooks:
      on_config: my-hook
      on_request: request-logger

mcpServers:
  brave-web-search:
    command: npx
    args: ["-y", "@modelcontextprotocol/server-brave-web-search"]
    env:
      BRAVE_API_KEY: "${BRAVE_API_KEY}"

prompts:
  - name: analyze-product
    description: 分析产品信息
    arguments:
      - name: url
        description: 产品 URL

hooks:
  - name: my-hook
    on_config: ./hooks/config.ts
```

## 请求处理流程

1. **CLI 启动** → cmd.Main(cli.New()) → serve 命令
2. **HTTP 请求** → /mcp/v1/* 端点
3. **Session 创建** → 从数据库加载或新建会话
4. **MCP 处理** → server.handleMCPRequest() 路由到相应处理器
5. **运行时协调** → runtime.HandleRequest() 调用 LLM + Tools
6. **工具执行** → tools.service.CallTool() 委托给 MCP 服务器
7. **响应返回** → 序列化 MCP 协议格式返回

## 关键架构特点

1. **协议层分离** - MCP 协议实现 (pkg/mcp/) 与业务逻辑分离
2. **工具抽象** - 通过 server:tool 引用格式解耦工具定义与使用
3. **可扩展钩子** - 支持 config/request/response 生命周期的代码注入
4. **会话继承** - 支持父子会话，可进行上下文传递
5. **多 LLM 支持** - 通过动态路由支持 OpenAI/Anthropic
6. **内置工具集** - 提供系统级工具 (bash, file ops, search) 作为内置 MCP 服务器