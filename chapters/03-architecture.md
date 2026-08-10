# 第三章 · 架构总览


> **本章目标**：
> 1. 掌握 DeerFlow 的分层架构（Client → Nginx → Gateway → LangGraph）
> 2. 理解 Gateway API 与 LangGraph Server 的职责边界
> 3. 了解 ThreadState 的状态流转与 AgentState 扩展机制

> **本章目标**：
> 1. 掌握 DeerFlow 的分层系统架构与各层职责
> 2. 理解 ThreadState 与 AgentState 的状态流转
> 3. 了解中间件链、Skill 加载与 Gateway 的协作机制

## 3.1 系统架构图

DeerFlow 采用典型的分层架构，通过 Nginx 统一入口：

```
┌──────────────────────────────────────────────────────────────────┐
│                        Client (Browser)                           │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│                    Nginx (Port 2026)                               │
│                   Unified Reverse Proxy Entry Point                 │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  /api/langgraph/*  →  LangGraph Server (2024)              │  │
│  │  /api/*            →  Gateway API (8001)                   │  │
│  │  /*                →  Frontend (3000)                       │  │
│  └────────────────────────────────────────────────────────────┘  │
└────────────────────────────┬─────────────────────────────────────┘
                             │
     ┌───────────────────────┼───────────────────────┐
     │                       │                       │
     ▼                       ▼                       ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ LangGraph Server│ │   Gateway API   │ │    Frontend     │
│   (Port 2024)   │ │   (Port 8001)   │ │   (Port 3000)   │
│                 │ │                 │ │                 │
│  Agent Runtime  │ │  Models API     │ │  Next.js App    │
│  Thread Mgmt    │ │  MCP Config     │ │  React UI       │
│  SSE Streaming  │ │  Skills Mgmt    │ │  Chat Interface │
│  Checkpointing  │ │  File Uploads   │ │                 │
│  Progressive    │ │  Artifacts      │ │                 │
│  Skill Loading  │ │                 │ │                 │
└────────┬────────┘ └────────┬────────┘ └─────────────────┘
         │                   │
         │     ┌─────────────┘
         │     │
         ▼     ▼
┌──────────────────────────────────────────────────────────────────┐
│                     Shared Configuration                           │
│  ┌────────────────────────┐  ┌────────────────────────────────┐ │
│  │      config.yaml       │  │   extensions_config.json       │ │
│  │  Models                │  │   MCP Servers                  │ │
│  │  Tools                 │  │   Skills State                 │ │
│  │  Sandbox               │  │                                │ │
│  │  Summarization         │  │                                │ │
│  └────────────────────────┘  └────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

## 3.2 核心组件详解

> **🏢 企业级建议**：Gateway API 作为统一入口，建议在其前面再部署一层企业级 API 网关（如 Kong、Apigee）以处理 SSO、WAF、DDoS 防护等更高级别的安全需求。

### 3.2.1 LangGraph Server（Agent 运行时）

**职责：**
- Agent 创建与配置
- Thread 状态管理
- 中间件链执行
- Tool 执行编排
- SSE 流式响应
- **渐进式 Skill 加载** — 按需动态加载 Skill，降低启动开销

**入口点：**
```text
packages/harness/deerflow/agents/lead_agent/agent.py:make_lead_agent
```

**配置文件：** `langgraph.json`

```json
{
  "agent": {
    "type": "agent",
    "path": "deerflow.agents:make_lead_agent"
  }
}
```

### 3.2.2 Gateway API

FastAPI 应用，提供非 Agent 操作的 REST 端点。

**路由划分：**

| 路由 | 端点 | 职责 |
|------|------|------|
| `models.py` | `/api/models` | 模型列表与详情 |
| `mcp.py` | `/api/mcp` | MCP Server 配置 |
| `skills.py` | `/api/skills` | Skills 管理 |
| `uploads.py` | `/api/threads/{id}/uploads` | 文件上传 |
| `threads.py` | `/api/threads/{id}` | Thread 数据清理 |
| `artifacts.py` | `/api/threads/{id}/artifacts` | 文件服务 |
| `suggestions.py` | `/api/threads/{id}/suggestions` | 后续建议生成 |

### 3.2.3 Frontend

Next.js 应用，提供 React UI。

### 前端架构

```
┌─────────────────────────────────────────────────────────────┐
│                     Next.js Frontend (Port 3000)              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐   │
│  │   App Router    │  │   API Routes    │  │  Middleware│   │
│  │  (app/)         │  │  (app/api/)     │  │            │   │
│  ├─────────────────┤  ├─────────────────┤  └─────────────┘   │
│  │                 │  │                 │                    │
│  │ • /chat/[id]    │  │ • /api/models   │                    │
│  │ • /projects     │  │ • /api/threads    │                    │
│  │ • /settings     │  │ • /api/skills     │                    │
│  │ • /artifacts    │  │ • /api/uploads    │                    │
│  │                 │  │                 │                    │
│  └────────┬────────┘  └────────┬────────┘                    │
│           │                    │                             │
│           ▼                    ▼                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                    React Components                      │ │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐    │ │
│  │  │ Chat UI    │  │ ThreadList │  │  Settings  │    │ │
│  │  │            │  │            │  │            │    │ │
│  │  │ - Message  │  │ - Search   │  │ - Models   │    │ │
│  │  │   Stream   │  │ - Filter   │  │ - Theme    │    │ │
│  │  │ - Input    │  │ - Sort     │  │ - Lang     │    │ │
│  │  │ - Toolbar  │  │            │  │            │    │ │
│  │  └────────────┘  └────────────┘  └────────────┘    │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │              SSE 流式消息处理 (EventSource)                │ │
│  │                                                          │ │
│  │  Gateway/LangGraph ──→ Server-Sent Events ──→ React State│ │
│  │  (实时推送 Agent 输出)                                     │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**核心页面：**

| 路由 | 页面 | 功能 |
|------|------|------|
| `/chat/[id]` | 对话页 | 消息流、工具调用展示、文件上传 |
| `/projects` | 项目管理 | 多 Agent 协作项目列表 |
| `/settings` | 设置页 | 模型选择、主题切换、语言设置 |
| `/artifacts` | 产物管理 | 查看/下载 Agent 生成的文件 |

**SSE 流式通信：**

前端通过 EventSource 连接到 LangGraph Server 的 SSE 端点，实时接收 Agent 的输出：

```typescript
// app/chat/hooks/useAgentStream.ts
const eventSource = new EventSource(`/api/langgraph/threads/${threadId}/runs/stream`);

eventSource.onmessage = (event) => {
  const data = JSON.parse(event.data);
  // 处理消息类型：message/tool_call/artifact/error
  if (data.type === 'message') {
    appendMessage(data.content);
  } else if (data.type === 'tool_call') {
    showToolExecution(data.tool_name, data.status);
  }
};
```

**状态管理：**

- **React Context**：Thread 级别状态（messages、artifacts、loading）
- **SWR**：服务端数据缓存（model list、thread list、skill registry）
- **Zustand**：全局 UI 状态（theme、sidebar collapsed）

## 3.3 Agent 架构详解

```
┌───────────────────────────────────────────────────────────────────┐
│                      make_lead_agent(config)                       │
└─────────────────────────────┬─────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────────┐
│                         Middleware Chain                           │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ 1. ThreadDataMiddleware    - 初始化 workspace/uploads/outputs │  │
│  │ 2. UploadsMiddleware      - 处理上传文件                    │  │
│  │ 3. SandboxMiddleware      - 获取沙箱环境                    │  │
│  │ 4. SummarizationMiddleware - 上下文压缩（启用时）           │  │
│  │ 5. TitleMiddleware        - 自动生成标题                    │  │
│  │ 6. TodoListMiddleware     - 任务跟踪（plan_mode 时）        │  │
│  │ 7. ViewImageMiddleware    - Vision 模型支持                 │  │
│  │ 8. ClarificationMiddleware - 处理澄清请求                   │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────┬─────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────────┐
│                            Agent Core                              │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐ │
│  │      Model       │  │      Tools       │  │   System Prompt  │ │
│  │   (from config)  │  │ (configured +    │  │   (with skills)  │ │
│  │                  │  │  MCP + builtin)  │  │                  │ │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘ │
└───────────────────────────────────────────────────────────────────┘
```

## 3.4 ThreadState 与 AgentState

`ThreadState` 扩展了 LangGraph 的 `AgentState`：

```python
class ThreadState(AgentState):
    # 来自 AgentState 的核心状态
    messages: list[BaseMessage]

    # DeerFlow 扩展字段
    sandbox: dict              # 沙箱环境信息
    artifacts: list[str]       # 生成的文件路径
    thread_data: dict          # {workspace, uploads, outputs} 路径
    title: str | None          # 自动生成的对话标题
    todos: list[dict]          # 任务跟踪（plan 模式）
    viewed_images: dict        # Vision 模型图片数据
```

## 3.5 Sandbox 系统架构

> **⚠️ 注意**：Local Sandbox 虽然部署简单，但在多租户场景下存在容器逃逸风险。生产环境务必使用 K8s Provisioner 或专用沙箱服务。

```
┌───────────────────────────────────────────────────────────────────┐
│                        Sandbox Architecture                       │
└───────────────────────────────────────────────────────────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
         ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐
│     Local       │  │     Docker      │  │   Provisioner       │
│   Executor      │  │   Container     │  │   (K8s Pod)         │
│                 │  │                 │  │                     │
│ - 直接本地执行   │  │ - 容器隔离      │  │ - K8s Pod 隔离      │
│ - 无额外开销    │  │ - 镜像预拉取    │  │ - 按需创建/销毁     │
│ - 开发/调试用   │  │ - 资源限制      │  │ - 生产环境推荐      │
└─────────────────┘  └─────────────────┘  └─────────────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
         ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐
│  Code Interpreter│  │  Web Fetch      │  │   Skill Executor    │
│  (Python/JS)    │  │  + Search       │  │                     │
└─────────────────┘  └─────────────────┘  └─────────────────────┘
```

## 3.6 请求处理流程

```
用户请求
    │
    ▼
Nginx (2026)
    │
    ├─→ /api/langgraph/* → LangGraph Server
    │                         │
    │                         ▼
    │                      Agent Runtime
    │                         │
    │                         ├─→ Middleware Chain
    │                         ├─→ Progressive Skill Loading（按需加载）
    │                         ├─→ Model (LLM)
    │                         ├─→ Tools
    │                         ├─→ Sub-Agents
    │                         └─→ Sandbox
    │
    ├─→ /api/* → Gateway API
    │              │
    │              ▼
    │           REST Endpoints
    │
    └─→ /* → Frontend (Static)
```

> **🔎 用真实代码走一遍这张图**
>
>
> ## 一、改造后的图（每个框 = 真实代码）
>
> ```text
> 用户在浏览器聊天框输入消息
>     │  前端聊天组件 → POST /api/threads/{id}/runs/stream
>     ▼
> Nginx (2026)
>     │  location /api/langgraph/ { rewrite ^/api/langgraph/(.*) /api/$1 break; }   ← nginx.conf:80
>     ▼
> Gateway API (8001) — FastAPI 内嵌 LangGraph 运行时（没有独立 2024 进程）
>     │
>     ├─ stream_run(thread_runs.py:846)  POST /{thread_id}/runs/stream
>     │     ├─ start_run(services.py:1050)   ← 建 RunRecord、校验线程权限
>     │     │     ├─ resolve_agent_factory(services.py:1128) → make_lead_agent
>     │     │     ├─ build_run_config(services.py:536)       ← thread_id/agent_name/递归上限
>     │     │     └─ 后台启动 run_agent worker
>     │     ├─ run_agent(worker.py:534)     ← 真正的"Agent Runtime"
>     │     │     ├─ _build_runtime_context(worker.py:367)   ← 模拟 CLI 的 context 注入
>     │     │     ├─ agent = agent_factory(config)           ← make_lead_agent(agent.py:640)
>     │     │     │     └─ _make_lead_agent(agent.py:671)
>     │     │     │           ├─ create_chat_model(...)      ← Model (LLM)
>     │     │     │           ├─ get_available_tools(...)    ← Tools (tools.py)
>     │     │     │           ├─ build_middlewares(...)      ← Middleware Chain (agent.py:373)
>     │     │     │           └─ apply_prompt_template(...)  ← System Prompt (含 skills)
>     │     │     └─ agent.astream(...)     ← 图执行，每步 bridge.publish 事件
>     │     └─ sse_consumer(services.py:1325) ← 从 bridge 订阅 → format_sse → SSE 回前端
>     ▼
> 前端 EventSource 收到 message/tool_call/end 事件并渲染
> ```
>
> ## 二、例子 1：用户发"帮我看看当前目录里有什么文件"
>
> 1. **前端**发 `POST /api/threads/{thread_id}/runs/stream`（浏览器里走 nginx 2026）
> 2. **Nginx** `nginx.conf:80-81`：`/api/langgraph/*` → 重写为 `/api/*` → 代理到 Gateway 8001
> 3. **Gateway 路由** `thread_runs.py:846 stream_run`：调用 `start_run` 后立刻返回 `StreamingResponse(media_type="text/event-stream")`，把 HTTP 连接挂起
> 4. **start_run** `services.py:1050`：校验 `thread_id`、模型白名单、线程归属权限（`check_access`），然后 `resolve_agent_factory` 拿到 `make_lead_agent`，后台启动 worker——**请求在路由器就返回了，agent 在后台跑**
> 5. **run_agent** `worker.py:534`：
>    - 注入运行时上下文（thread_id/run_id/app_config，`worker.py:367`）
>    - `agent = make_lead_agent(config)` → 现场构建图：解析模型 → 组装工具 → 装配中间件链 → 生成系统提示词
>    - `agent.astream(...)` 开始执行，每个输出事件 `bridge.publish(run_id, "messages", chunk)`
> 6. **sse_consumer** `services.py:1325`：`bridge.subscribe(run_id)` 拿到事件，`format_sse` 转成 `event: messages\ndata: {...}` 帧推给浏览器
> 7. 前端收到事件流，逐条渲染。如果中途**关掉页面**：`sse_consumer` 的 `finally` 块（services.py:1376-1382）按 `on_disconnect` 决定取消任务（`run_mgr.cancel`）还是让它跑完
>
> ## 三、例子 2：模型决定调用沙箱里的 bash 工具
>
> 当模型在图执行中说"我要调用 `bash` 工具"（例如列目录）：
>
> - **中间件链**先拦一道：20+ 个中间件按 `build_middlewares`（agent.py:373）的装配顺序执行——`SandboxMiddleware`（`sandbox/middleware.py`，懒初始化：默认 `lazy_init=True`，**第一次工具调用才获取沙箱**，同一 thread 复用，应用关闭才释放）
> - **工具清单**由 `get_available_tools`（tools.py）按 `config.yaml` 决定：测试 `test_local_bash_tool_loading.py` 里就有"默认 local sandbox 隐藏 bash、AIO 沙箱保留 bash"的行为；`subagent_enabled=True` 时还会加 `task`/`task_status`
> - 工具执行结果写回状态 → 模型继续下一轮，直到输出最终答案
>
> ## 四、例子 3：用户输入 `/skill-name`（渐进式 Skill 加载的真实落点）
>
> 图里那句"Progressive Skill Loading（按需加载）"其实拆成三个真实机制：
>
> 1. **包根懒加载**：`deerflow.agents` 的 `__getattr__`（agents/__init__.py:15）只在 LangGraph 解析 `deerflow.agents:make_lead_agent` 时才 import 整棵工具图并预热 skills 缓存——**进程启动时不拉全量**
> 2. **SkillActivationMiddleware**（agent.py:390 处装配）：用户消息以 `/skill-name` 开头时，确定性加载完整 SKILL.md
> 3. **SkillToolPolicyMiddleware**（agent.py:399 处）：技能激活后，运行时才把该技能允许的工具暴露给模型

## 3.7 中间件链详解

中间件是 DeerFlow 的请求处理链，每个中间件负责特定功能。DeerFlow 2.0 共包含 **18 个中间件**（详见第五章 5.5.1 节），按职责分为两大类：

| 类别 | 中间件 | 职责 |
|------|--------|------|
| **运行时基础** | ThreadDataMiddleware | 初始化每个 Thread 的工作目录 |
| | UploadsMiddleware | 处理用户上传的文件 |
| | SandboxMiddleware | 获取沙箱环境 |
| | DanglingToolCallMiddleware | 补全缺失的工具响应 |
| | LLMErrorHandlingMiddleware | LLM 错误处理与重试 |
| | GuardrailMiddleware | 工具调用前授权（可选） |
| | SandboxAuditMiddleware | 沙箱命令安全审计 |
| | ToolErrorHandlingMiddleware | 工具错误处理 |
| **Lead Agent 专属** | SummarizationMiddleware | 上下文压缩（可选） |
| | TodoListMiddleware | 任务跟踪（plan 模式可选） |
| | TokenUsageMiddleware | Token 使用统计（可选） |
| | TitleMiddleware | 自动生成标题 |
| | MemoryMiddleware | 异步记忆更新 |
| | ViewImageMiddleware | Vision 模型图片注入（可选） |
| | DeferredToolFilterMiddleware | 延迟工具过滤（可选） |
| | SubagentLimitMiddleware | 子任务数量限制（可选） |
| | LoopDetectionMiddleware | 循环检测与阻断 |
| | ClarificationMiddleware | 澄清请求拦截（必须最后） |

> 完整中间件接口定义、参数说明和源码解析，请参阅 **5.5.1 完整中间件顺序** 与 **5.5.3 核心中间件详解**。

## 3.8 配置体系

DeerFlow 使用双配置文件：

### config.yaml
主配置文件，定义模型、工具、沙箱等核心设置。

```yaml
models:
  - name: gpt-4
    display_name: GPT-4
    use: langchain_openai:ChatOpenAI
    model: gpt-4
    api_key: $OPENAI_API_KEY

sandbox:
  use: deerflow.community.aio_sandbox:AioSandboxProvider
  provisioner_url: https://...
```

### extensions_config.json
扩展配置文件，管理 MCP Servers 和 Skills。

```json
{
  "mcp_servers": [
    {
      "name": "filesystem",
      "command": ["npx", "-y", "@modelcontextprotocol/server-filesystem"],
      "args": ["/tmp"]
    }
  ],
  "skills": {
    "deer-flow-skills/recursive-summarizer": {
      "enabled": true
    }
  }
}
```

## 3.9 渐进式 Skill 加载的架构位置

Progressive Skill Loading 是 DeerFlow 2.0 引入的核心特性，它在架构层面解决了 Skill 过多导致的启动延迟问题。

### 3.9.1 架构定位

```
┌───────────────────────────────────────────────────────────────────┐
│                    Progressive Skill Loading                      │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐           │
│  │  Skill      │    │  Lazy       │    │  On-Demand  │           │
│  │  Registry   │───→│  Loader     │───→│  Executor   │           │
│  │             │    │             │    │             │           │
│  │ - ClawHub   │    │ - 解析      │    │ - 初始化    │           │
│  │ - Local     │    │   依赖树    │    │ - 注入      │           │
│  │ - Builtin   │    │ - 按需拉取  │    │   工具集    │           │
│  └─────────────┘    └─────────────┘    └─────────────┘           │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

**在 LangGraph Server 中的位置：**
- 位于 **Middleware Chain 之后、Model 之前**
- 拦截用户请求中的 Skill 引用（如 `@skill_name`）
- 动态解析 Skill 依赖树，按需加载到当前 Thread 的上下文中

### 3.9.2 加载时机

| 触发时机 | 行为 | 性能影响 |
|----------|------|----------|
| Thread 创建 | 加载 `extensions_config.json` 中 `enabled: true` 的 Skill | 基础开销 |
| 用户显式引用 | 解析 `@skill_name`，懒加载对应 Skill | 首次延迟 |
| Skill 依赖解析 | 递归加载被引用的子 Skill | 级联加载 |
| 热更新 | 检测到 Skill 版本变更时重新加载 | 运行时更新 |

### 3.9.3 与其他组件的协作

**与 Middleware Chain 的协作：**
```
Middleware Chain
    │
    ├─→ ThreadDataMiddleware（初始化目录）
    ├─→ UploadsMiddleware（处理上传）
    ├─→ SandboxMiddleware（获取沙箱）
    ├─→ ProgressiveSkillMiddleware ← 新增：解析并加载 Skill
    │      │
    │      ▼
    │   Skill Registry → Lazy Loader → Tool Injection
    │
    └─→ SummarizationMiddleware（上下文压缩）
```

**与 Gateway API 的协作：**
- Gateway 的 `/api/skills` 端点提供 Skill 元数据查询
- Progressive Loader 通过内部 API 获取 Skill 的依赖关系和加载策略

### 3.9.4 设计优势

1. **启动加速**：Agent 启动时无需加载全部 Skill，仅加载必需的基础集合
2. **内存优化**：未使用的 Skill 不占用运行时内存
3. **依赖自治**：Skill 声明自己的依赖，Loader 自动解析依赖树
4. **热插拔**：支持运行时更新 Skill 而不重启 Agent

## 3.10 小结

DeerFlow 的架构设计体现了以下原则：

| 原则 | 体现 |
|------|------|
| **分层解耦** | Nginx → Gateway/LangGraph → Services |
| **中间件编排** | 请求经过可插拔的中间件链 |
| **配置驱动** | 双配置文件机制 |
| **可扩展性** | MCP Server、Custom Skills 支持 |
| **安全隔离** | Sandbox 多层架构 |

理解这些架构设计，是后续深入源码的前提。


## 本章小结

本章梳理了 DeerFlow 的系统架构与模块交互：

1. **分层网关**：Nginx (2026) → Gateway API (8001) / LangGraph Server (2024) / Frontend (3000)，统一入口、职责分离。
2. **Gateway API**：FastAPI 应用，负责非 Agent 操作（认证、模型列表、租户管理），与 LangGraph Server 解耦。
3. **LangGraph Server**：核心 Agent 编排服务，处理对话流、工具调用、状态管理，通过 ThreadState 维护会话上下文。
4. **ThreadState 扩展**：在 LangGraph AgentState 基础上扩展了 sandbox、artifacts、thread_data 等企业级字段。

> **🏢 企业级建议**：生产环境中建议为 Gateway API 和 LangGraph Server 配置独立的负载均衡与健康检查。

---

**下一步**：阅读第四章，熟悉 DeerFlow 的 monorepo 目录结构与项目组织方式。

