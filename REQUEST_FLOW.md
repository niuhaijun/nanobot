# Nanobot 请求处理流程详解

## 概述

本文档详细描述 Nanobot 系统处理一个请求的完整流程，从 HTTP 请求入口到最终响应的全过程。

## 请求处理总览

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Client    │────▶│ HTTPServer  │────▶│   Server    │────▶│   Runtime   │
│  (HTTP)     │     │  (MCP)      │     │  (路由)     │     │  (协调)     │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
                                                                │
                    ┌───────────────────────────────────────────┤
                    ▼                   ▼                   ▼
              ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
              │ ToolsService│     │ AgentsService│    │  Sampler    │
              │  (工具)     │     │   (Agent)   │     │  (采样)     │
              └─────────────┘     └─────────────┘     └─────────────┘
                    │                   │                   │
                    ▼                   ▼                   ▼
              ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
              │ MCP Client  │     │   LLM API   │     │   Tools     │
              │ (外部服务)  │     │  (大模型)   │     │  (内置)     │
              └─────────────┘     └─────────────┘     └─────────────┘
```

## 第一阶段：HTTP 入口

### 文件：`pkg/mcp/httpserver.go`

当客户端发送 HTTP POST 请求时，`HTTPServer.serveHTTP` 方法负责处理：

```go
func (h *HTTPServer) serveHTTP(rw http.ResponseWriter, req *http.Request) {
    ctx := h.baseCtx(rw, req)

    // 1. 从请求头或 URL 提取 Session ID
    sessionID := h.sessions.ExtractID(req)

    // 2. 设置审计日志
    auditLog := &util.AuditLog{
        TraceID:    h.traceID(),
        SessionID:  sessionID,
        RequestURI: req.RequestURI,
        UserAgent:  req.UserAgent(),
        RemoteAddr: req.RemoteAddr,
    }
    auditLog.RequestBody, _ = io.ReadAll(req.Body)

    var msg Message
    json.Unmarshal(auditLog.RequestBody, &msg)

    // 3. 处理已存在的 Session
    if sessionID != "" {
        streamingSession, ok, err := h.sessions.Acquire(ctx, h.MessageHandler, sessionID)
        if err != nil {
            // ...
        }
        response, err := streamingSession.Exchange(ctx, msg)
        // ...
        rw.Write(respBody)
        return
    }

    // 4. 处理新 Session（仅限 initialize 请求）
    if msg.Method == "initialize" {
        session, err := NewServerSession(h.ctx, h.MessageHandler, h.traceID, h.cfg)
        if err != nil {
            // ...
        }
        resp, err := session.Exchange(ctx, msg)
        if err != nil {
            // ...
        }

        // 存储新 Session
        h.sessions.Store(ctx, session.ID(), session)

        // 返回 Session ID 给客户端
        rw.Header().Set("Mcp-Session-Id", session.ID())
        rw.Write(respBody)
    }
}
```

### Session 管理

Session 管理器维护两个存储：
1. **内存缓存** (`liveSessions`)：活跃的 Session
2. **数据库**：持久化存储

```go
// pkg/session/manager.go
func (m *Manager) Acquire(ctx context.Context, server mcp.MessageHandler, id string) (ret *mcp.ServerSession, found bool, retErr error) {
    // 优先从内存缓存查找
    m.liveSessionsLock.Lock()
    live, ok := m.liveSessions[id]
    if ok {
        live.count++
        m.liveSessionsLock.Unlock()
        return live.session, true, nil
    }
    m.liveSessionsLock.Unlock()

    // 从数据库加载
    serverSession, ok, err := m.loadSessionFromDatabase(ctx, server, id)
    if err != nil {
        return nil, false, err
    }

    // 添加到内存缓存
    m.liveSessions[id] = liveSession{session: serverSession, count: 1}
    return serverSession, true, nil
}
```

## 第二阶段：MCP 协议路由

### 文件：`pkg/server/server.go`

`Server` 结构体维护一个处理器映射表，根据方法名路由请求：

```go
func (s *Server) init() {
    s.handlers = []handler{
        handle("initialize", s.handleInitialize),
        handle("tools/list", s.handleListTools),
        handle("tools/call", s.handleCallTool),
        handle("prompts/list", s.handleListPrompts),
        handle("prompts/get", s.handleGetPrompt),
        handle("resources/list", s.handleListResources),
        handle("resources/read", s.handleReadResource),
        handle("resources/templates/list", s.handleListResourceTemplates),
        handle("resources/templates/get", s.handleGetResourceTemplate),
        handle("completion/complete", s.handleComplete),
        handle("logging/setLevel", s.handleSetLevel),
        handle("ping", s.handlePing),
    }
}
```

### 工具调用流程

当收到 `tools/call` 请求时：

```go
func (s *Server) handleCallTool(ctx context.Context, msg mcp.Message, payload mcp.CallToolRequest) error {
    // 1. 获取工具映射配置
    toolMappings, err := s.data.ToolMapping(ctx)
    toolMapping, ok := toolMappings[payload.Name]

    // 2. 如果未找到，尝试刷新配置后重试
    if !ok {
        s.data.Refresh(ctx, false)
        toolMappings, err = s.data.ToolMapping(ctx)
        toolMapping, ok = toolMappings[payload.Name]
        if !ok {
            return msg.ReplyError(ctx, -32602, "tool not found: "+payload.Name, nil)
        }
    }

    // 3. 调用 Runtime 执行工具
    result, err := s.runtime.Call(ctx, toolMapping.MCPServer, toolMapping.TargetName, payload.Arguments, tools.CallOptions{
        ProgressToken: msg.ProgressToken(),
    })
    if err != nil {
        // 错误处理
    }

    // 4. 返回结果
    return msg.Reply(ctx, mcp.CallToolResult{
        Content: []mcp.Content{},
        IsError: false,
    })
}
```

## 第三阶段：Runtime 协调

### 文件：`pkg/runtime/runtime.go`

`Runtime` 是核心协调器，整合了所有服务：

```go
func NewRuntime(cfg llm.Config, opts ...Options) (*Runtime, error) {
    // 1. 创建 LLM 客户端
    completer := llm.NewClient(cfg)

    // 2. 创建工具服务
    registry := tools.NewToolsService(
        cfg.PanelAgentKey,
        cfg.MCPExternalURL,
        cfg.MCPExternalToken,
        cfg.DisableMCPExternal,
        cfg.Sessions,
    )

    // 3. 创建 Agent 服务
    agentsService := agents.New(completer, registry)

    // 4. 创建采样器
    sampler := sampling.NewSampler(agentsService)

    // 5. 设置循环依赖
    registry.SetSampler(sampler)

    // 6. 注册内置服务器
    registry.AddServer("nanobot.meta", metaServer)
    registry.AddServer("nanobot.agent", agentServer)
    registry.AddServer("nanobot.system", systemServer)
    // ...

    return &Runtime{
        toolsService:  registry,
        agentsService: agentsService,
        sampler:       sampler,
    }, nil
}
```

## 第四阶段：工具服务

### 文件：`pkg/tools/service.go`

`ToolsService.Call` 方法决定工具调用策略：

```go
func (s *Service) Call(ctx context.Context, server, tool string, args any, opts ...CallOptions) (*types.CallResult, error) {
    // 1. 检查是否为 Agent 调用
    if _, ok := config.Agents[server]; ok && tool != types.AgentTool+server {
        return s.sampleCall(ctx, server, args, opts...)
    }

    // 2. 获取或创建 MCP 客户端
    c, err := s.GetClient(ctx, server)
    if err != nil {
        return nil, err
    }

    // 3. 通过 MCP 客户端调用工具
    mcpCallResult, err := c.Call(ctx, tool, args, mcp.CallOption{
        Timeout:       300 * time.Second,
        ProgressToken: opts.ProgressToken,
    })
    if err != nil {
        return nil, err
    }

    return &types.CallResult{
        Content: mcpCallResult.Content,
        IsError: mcpCallResult.IsError,
    }, nil
}
```

### Agent 调用（采样模式）

```go
func (s *Service) sampleCall(ctx context.Context, server string, args any, opts ...CallOptions) (*types.CallResult, error) {
    // 1. 创建采样请求
    req := &sampling.Request{
        Agent:   server,
        Input:   args,
        Sampler: s.sampler,
    }

    // 2. 执行采样
    resp, err := s.sampler.Sample(ctx, req)
    if err != nil {
        return nil, err
    }

    return &types.CallResult{
        Content: []types.Content{
            {Type: "text", Text: resp.Output},
        },
    }, nil
}
```

## 第五阶段：MCP 客户端

### 文件：`pkg/mcp/client.go`

`Client.Call` 方法向外部 MCP 服务器发送调用请求：

```go
func (c *Client) Call(ctx context.Context, tool string, args any, opts ...CallOption) (result *CallToolResult, err error) {
    err = c.Session.Exchange(ctx, "tools/call", struct {
        Name      string         `json:"name"`
        Arguments any            `json:"arguments,omitempty"`
        Meta      map[string]any `json:"_meta,omitempty"`
    }{
        Name:      tool,
        Arguments: args,
        Meta:      opt.Meta,
    }, result, ExchangeOption{ProgressToken: opt.ProgressToken})
    return
}
```

## 第六阶段：Session 消息交换

### 文件：`pkg/mcp/serversession.go`

`ServerSession.Exchange` 处理请求-响应模式：

```go
func (s *ServerSession) Exchange(ctx context.Context, msg Message) (Message, error) {
    // 1. 预处理消息（处理 initialize 特殊逻辑）
    isInit, err := s.session.preInit(&msg)
    if err != nil {
        return Message{}, err
    }

    // 2. 通过 wire 发送消息并等待响应
    resp, err := s.wire.exchange(ctx, msg)
    if err != nil {
        return Message{}, err
    }

    // 3. 后处理响应
    if isInit {
        s.session.postInit(&resp)
    }

    return resp, nil
}
```

### serverWire 实现

```go
func (s *serverWire) exchange(ctx context.Context, msg Message) (Message, error) {
    // 1. 创建通道等待响应
    ch := s.pending.WaitFor(msg.ID)

    // 2. 异步发送消息
    go func() {
        s.handler(ctx, msg)
    }()

    // 3. 等待响应
    select {
    case m, ok := <-ch:
        if !ok {
            return Message{}, errors.New("channel closed")
        }
        return m, nil
    case <-ctx.Done():
        return Message{}, ctx.Err()
    }
}
```

## 第七阶段：Agent 执行

### 文件：`pkg/agents/run.go`

`Agents.run` 是 Agent 的核心执行循环：

```go
func (a *Agents) run(ctx context.Context, config types.Config, run *types.Execution, prev *types.Execution, opts []types.CompletionOptions) error {
    // 1. 填充请求参数
    completionRequest, toolMapping, err := a.populateRequest(ctx, config, run, prev, opts)
    if err != nil {
        return err
    }

    // 2. 处理历史记录压缩（如果上下文过长）
    completionRequest, resp, err := a.handleCompaction(ctx, config, completionRequest, opts)
    if err != nil {
        return err
    }

    // 3. 执行 before 钩子
    completionRequest, resp, err = a.runBefore(ctx, config, completionRequest)
    if err != nil {
        return err
    }

    // 4. 处理 UI 动作（如果需要用户交互）
    modifiedRequest, resp, err := a.handleUIAction(ctx, config, completionRequest, opts)
    if err != nil {
        return err
    }

    // 5. 调用 LLM 获取完成结果
    resp, err = a.completer.Complete(ctx, modifiedRequest, opts...)
    if err != nil {
        return err
    }

    // 6. 执行 after 钩子
    resp, err = a.runAfter(ctx, config, completionRequest, resp)
    if err != nil {
        return err
    }

    // 7. 保存响应
    run.Response = resp
    return nil
}
```

## 完整请求处理序列图

```
Client          HTTPServer       ServerSession    Server          Runtime          ToolsService     MCPClient
  │                 │                │              │                │                  │               │
  │──POST /mcp─────▶│                │              │                │                  │               │
  │                 │                │              │                │                  │               │
  │                 │──serveHTTP()───┤              │                │                  │               │
  │                 │                │              │                │                  │               │
  │                 │                │──Exchange()──│                │                  │               │
  │                 │                │              │                │                  │               │
  │                 │                │              │──handleCallTool()──             │               │
  │                 │                │              │                │                  │               │
  │                 │                │              │───────────────Call()────────────▶│               │
  │                 │                │              │                │                  │               │
  │                 │                │              │                │──GetClient()────▶│               │
  │                 │                │              │                │                  │               │
  │                 │                │              │                │                  │──Call()──────▶│
  │                 │                │              │                │                  │               │
  │                 │                │              │                │                  │◀──Result──────│
  │                 │                │              │                │◀──Result─────────│               │
  │                 │                │              │◀──Result───────│                  │               │
  │                 │                │◀──Response───│                │                  │               │
  │◀──HTTP Response─│                 │              │                │                  │               │
```

## 关键数据结构

### Message（JSON-RPC 消息）

```go
type Message struct {
    JSONRPC string          `json:"jsonrpc"`
    ID      any             `json:"id,omitempty"`
    Method  string          `json:"method,omitempty"`
    Params  any             `json:"params,omitempty"`
    Result  any             `json:"result,omitempty"`
    Error   *Error          `json:"error,omitempty"`
}
```

### ServerSession（服务端会话）

```go
type ServerSession struct {
    session  *Session
    wire     *serverWire
    handlers map[string]Handler
    mu       sync.RWMutex
}
```

### ToolMapping（工具映射配置）

```go
type ToolMapping struct {
    MCPServer   string `json:"mcp_server"`
    TargetName  string `json:"target_name"`
    InputSchema string `json:"input_schema"`
}
```

## 错误处理机制

1. **工具未找到**：刷新配置后重试一次
2. **Session 未找到**：尝试从数据库加载
3. **LLM 调用失败**：执行 after 钩子后返回错误
4. **MCP 客户端调用超时**：默认 300 秒超时

## 内置工具服务器

系统在初始化时注册了以下内置服务器：

| 服务器名 | 说明 |
|---------|------|
| `nanobot.meta` | 元信息查询 |
| `nanobot.agent` | Agent 控制 |
| `nanobot.system` | 系统操作 |
| `nanobot.file` | 文件操作 |
| `nanobot.web` | Web 操作 |
| `nanobot.git` | Git 操作 |

## 配置与扩展

工具映射配置通过 `data.ToolMapping()` 获取，支持动态刷新。外部 MCP 服务器通过 `MCPExternalURL` 和 `MCPExternalToken` 配置。
