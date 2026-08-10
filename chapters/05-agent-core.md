# 第五章 · Agent 核心：LangGraph 编排逻辑


> **本章目标**：
> 1. 深入理解 make_lead_agent 工厂函数的初始化流程
> 2. 掌握 ThreadState 的字段定义与状态管理机制
> 3. 了解 LangGraph 节点编排、中间件链与工具路由的实现细节

> **本章目标**：
> 1. 深入理解 Lead Agent 的 LangGraph 编排逻辑
> 2. 掌握中间件链的设计模式与扩展方法
> 3. 了解 Client SDK 的使用与 Agent 缓存机制

## 5.1 核心入口：make_lead_agent
在全局中的位置：[[03-architecture#3.6 请求处理流程]]
DeerFlow 的 Agent 核心入口是一个工厂函数：

```python
# 入口：packages/harness/deerflow/agents/lead_agent/agent.py
from deerflow.agents import make_lead_agent

agent = make_lead_agent(config)
```

这个函数完成：
1. 动态模型选择（支持 thinking / vision）
2. 工具加载（Sandbox + Builtin + MCP + Community + Subagent）
3. 系统 Prompt 生成（包含 skills、memory、subagent 指令）
4. 中间件链组装

---

📌 勘误：关于"工具加载"的分类

此处"Sandbox + Builtin + MCP + Community + Subagent"是按用途分的 5 类。当前代码（get_available_tools()，tools/tools.py）实际按加载机制分 4 类：

- Sandbox + Community → 配置驱动类（config.yaml 的 tools: 段，动态 import）
- Builtin + Subagent → 代码内置类（tools/builtins/，硬编码 + 条件追加）
- MCP → MCP 类（extensions_config.json 的 mcpServers）
- —（文档未提及） → ACP 类（config.yaml 的 acp_agents）

详见 5.6.1 的勘误说明。

---

## 5.2 ThreadState：Agent 状态定义

```python
# packages/harness/deerflow/agents/thread_state.py
from typing import Annotated, NotRequired, TypedDict
from langchain.agents import AgentState


class SandboxState(TypedDict):
    """沙箱状态"""
    sandbox_id: NotRequired[str | None]


class ThreadDataState(TypedDict):
    """线程数据路径状态"""
    workspace_path: NotRequired[str | None]
    uploads_path: NotRequired[str | None]
    outputs_path: NotRequired[str | None]


class ViewedImageData(TypedDict):
    """已查看图片数据"""
    base64: str          # Base64 编码的图片数据
    mime_type: str       # MIME 类型 (如 image/png)


def merge_artifacts(existing: list[str] | None, new: list[str] | None) -> list[str]:
    """Reducer for artifacts list - merges and deduplicates artifacts."""
    if existing is None:
        return new or []
    if new is None:
        return existing
    # Use dict.fromkeys to deduplicate while preserving order
    return list(dict.fromkeys(existing + new))


def merge_viewed_images(
    existing: dict[str, ViewedImageData] | None,
    new: dict[str, ViewedImageData] | None
) -> dict[str, ViewedImageData]:
    """Reducer for viewed_images dict - merges image dictionaries.

    Special case: If new is an empty dict {}, it clears the existing images.
    This allows middlewares to clear the viewed_images state after processing.
    """
    if existing is None:
        return new or {}
    if new is None:
        return existing
    # Special case: empty dict means clear all viewed images
    if len(new) == 0:
        return {}
    # Merge dictionaries, new values override existing ones for same keys
    return {**existing, **new}


class ThreadState(AgentState):
    """DeerFlow Agent 状态定义
    
    继承自 LangGraph 的 AgentState，扩展 DeerFlow 特有字段。
    使用 NotRequired 标记可选字段，使用 Annotated + reducer 处理合并逻辑。
    """
    # 沙箱信息
    sandbox: NotRequired[SandboxState | None]
    
    # 线程工作目录路径
    thread_data: NotRequired[ThreadDataState | None]
    
    # 自动生成的对话标题
    title: NotRequired[str | None]
    
    # 生成的文件列表（使用 reducer 自动去重合并）
    artifacts: Annotated[list[str], merge_artifacts]
    
    # 任务跟踪（plan 模式）
    todos: NotRequired[list | None]
    
    # 上传的文件列表
    uploaded_files: NotRequired[list[dict] | None]
    
    # Vision 模型图片数据（使用 reducer 合并/清除）
    # 格式: {image_path -> {base64, mime_type}}
    viewed_images: Annotated[dict[str, ViewedImageData], merge_viewed_images]
```

**自定义 Reducer 说明：**

| Reducer | 功能 | 特殊行为 |
|---------|------|----------|
| `merge_artifacts` | 合并 artifacts 列表并去重 | 使用 `dict.fromkeys` 保持顺序去重 |
| `merge_viewed_images` | 合并 viewed_images 字典 | 传入空 `{}` 时清空所有图片（用于处理后清理）|

## 5.3 运行时配置（configurable）

通过 `config.configurable` 注入运行时参数：

| 配置项 | 类型 | 说明 |
|--------|------|------|
| `thinking_enabled` | bool | 启用模型扩展思考 |
| `model_name` | str | 选择具体 LLM 模型 |
| `is_plan_mode` | bool | 启用 TodoList 中间件 |
| `subagent_enabled` | bool | 启用任务委托工具 |

```python
async def run_agent():
    # 使用示例
    config = RunnableConfig(
        configurable={
            "thinking_enabled": True,
            "model_name": "claude-sonnet-4-6",
            "subagent_enabled": True,
        }
    )
    result = await agent.ainvoke(input, config=config)
```

> **关键理解：configurable 不是配置文件，而是"每次调用时传的参数包"（一个字典）**
>
> 它的值按优先级来自三层：
>
> 1. **每次调用显式传参**（最高）：`client.chat("你好", model_name="gpt-4o")`
> 2. **创建客户端时的默认**（其次）：`DeerFlowClient(model_name=..., thinking_enabled=...)`，存于 `self._model_name` 等属性
> 3. **config.yaml 默认值**（兜底）
>
> 代码本质：`configurable["model_name"] = overrides.get("model_name", self._model_name)`（client.py:243）；
> Agent 端 `_get_runtime_config` 从 `config["configurable"]` 读取（agent.py:119）。
>
> **一句话：调用方每次把想要的参数装进这个字典，盖过默认值；不装就用默认（config.yaml）。**

## 5.4 LangGraph 工作流结构

> **⚠️ 注意**：自定义 State 的字段命名需避免与 LangGraph 内置字段冲突（如 `messages`、`is_last_step`），否则会导致不可预期的状态覆盖问题。

```
┌─────────────────────────────────────────────────────────────┐
│                    StateGraph<ThreadState>                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  START ──→  middleware_chain ──→  model_llm ──→  tools ──→ END
│                        ▲                  │
│                        │                  │
│                        └──────────────────┘
│                           (循环直到完成)
└─────────────────────────────────────────────────────────────┘
```

> ---
> 📌 **更详细版本:5.4 的编译后真实结构(修正补充)**
>
> 书 5.4 顶部 ASCII 图、5.4.1 的三节点表、5.4.2 的 should_continue,是**同一张真实图的三个逻辑侧面**。DeerFlow 并不手写 StateGraph,而是把 `create_agent(model, tools, middleware=…)`(`agent.py:907`)交给 LangChain 1.x 编译,产物如下:
>
> ```text
> START
>   │
>   ▼
> 【before_agent 段 · 首轮一次】图节点,顺序执行
>   ThreadData → Uploads → Sandbox → ToolProgress* → DynamicContext
>   → Todo*(plan) → LoopDetection* → TokenBudget* → TerminalResponse
>   │
>   ▼  ← 循环入口 loop_entry(每轮从这里重新开始)
> 【before_model 段 · 每轮】图节点,顺序执行
>   DurableContext → Summarization* → Todo* → ViewImage*(vision) → McpRouting*
>   │
>   ▼
> ┌────────────────────────────────────────────────┐
> │ model 节点(≈5.4.1 的 model_llm)                 │
> │   内部洋葱 wrap_model_call(运行时,外→内):       │
> │   InputSanitization → ToolOutputBudget → …     │
> │   → TerminalResponse → 【调 LLM】               │
> └────────────────────────────────────────────────┘
>   │
>   ▼
> 【after_model 段 · 每轮,逆序执行】图节点
>   SafetyFinishReason* → … → Title → … → DurableContext
>   │   └─ 链尾挂 条件边A
>   ▼  条件边A: 最后一条 AI 消息有未执行 tool_calls?
>   │       (≈5.4.2 的 should_continue,判定优先级: jump_to → 无AI消息
>   │        → 无tool_calls→退出 → 有pending→并行Send → 人工注入→回model)
>   │
>   ├─ 无 ──► 【after_agent 段 · 末轮一次,逆序】
>   │         TerminalResponse → … → Memory → … → Sandbox ──► END
>   │
>   └─ 有 ──► ┌────────────────────────────────────────────────┐
>             │ tools 节点(≈5.4.1 的 tools)                    │
>             │   并行 Send;内部洋葱 wrap_tool_call(外→内):     │
>             │   ToolOutputBudget → … → Clarification →【执行】│
>             └────────────────────────────────────────────────┘
>                 │
>                 ▼  条件边B
>                 ├─ 默认 ──► 回 loop_entry(重跑 before_model 段)──► 循环
>                 └─ 全部 return_direct ──► after_agent 段 ──► END
> ```
>
> **① 5.4.1 三节点 ↔ 真实挂载位**
>
> | 5.4.1 逻辑节点 | 真实编译产物 |
> |---|---|
> | `middleware_chain` | 拆成 4 段钩子节点链(before_agent / before_model / after_model / after_agent)+ 2 层运行时洋葱(wrap_model_call / wrap_tool_call,画在节点内部) |
> | `model_llm` | `model` 节点 |
> | `tools` | `tools` 节点(ToolNode) |
>
> **② 5.4.2 should_continue ↔ 条件边A**(语义相同,三处差异)
>
> | 5.4.2 | 真实实现 |
> |---|---|
> | 挂在 model_llm 直出 | 挂在 **after_model 链尾**(中间件改完消息才判定) |
> | 两分支 | 5 分支(见图中注释),工具用 `Send` **并行**执行 |
> | 回"model" | 回 **loop_entry**(第一个 before_model 节点),工具永远不直连 END |
>
> **③ 相对 5.4 原文的修正点**
> 1. 中间件**真实存在**(共 31 个 `AgentMiddleware` 子类),只是不是图上那一个盒子:每个中间件的钩子会被拆成多个小节点,围在 model 和 tools 周围——上面图里各段列的名字就是它们;
> 2. `tools ──→ END` 直连不存在,工具执行完必回模型再决策;
> 3. 5.5 的 `Middleware.process()` 接口是虚构的,真实靠 6 类钩子工作(如 ClarificationMiddleware 用 `wrap_tool_call` 拦截澄清请求并中断,等待用户回复)。
>
> **洋葱一句话**:`wrap_model_call/wrap_tool_call` = 运行时"函数套函数",`handler` 代表里面所有层加核心调用,列表第一个=最外层(`middleware/types.py:671`),去程回程都经过每层,故能双向处理(消毒/重试/拦截/审计)。
>
> ---

### 5.4.1 节点定义

DeerFlow 的核心节点：

| 节点 | 职责 |
|------|------|
| `middleware_chain` | 执行中间件链 |
| `model_llm` | 调用 LLM |
| `tools` | 执行工具调用 |

### 5.4.2 边与条件跳转

```python
from langgraph.graph import END

# 条件函数
def should_continue(state: ThreadState) -> str:
    """
    判断是否继续执行
    """
    last_message = state["messages"][-1]
    
    # 如果有工具调用 → 继续执行工具
    if hasattr(last_message, "tool_calls") and last_message.tool_calls:
        return "tools"
    
    # 如果是最终响应 → 结束
    return END

# 添加条件边
workflow.add_conditional_edges(
    "model_llm",
    should_continue,
    {
        "tools": "tools_node",
        END: END
    }
)
```

> ⚠️ **勘误:5.4.2 的代码只是示意,不是 DeerFlow 的真实代码**——它是照着 LangGraph 官方教程的通用画法写的,用来表达"模型没调用工具就结束,调用了就继续执行"这个循环。DeerFlow 实际直接用 LangChain 现成的 `create_agent` 工厂拼出循环,真实结构见 5.4 下方【更详细版本】框。

## 5.5 中间件链（Middleware Chain）

> **💡 最佳实践**：中间件的执行顺序至关重要。SecurityMiddleware 应尽可能靠前，SummarizationMiddleware 应放在消息生成之后。新增中间件时务必测试与其他中间件的交互。

中间件是 DeerFlow 请求处理的核心机制，**按顺序执行，不可跳过**。

### 5.5.1 完整中间件顺序（18 个）

```
┌─────────────────────────────────────────────────────────────────┐
│                    Middleware Chain (18 个)                      │
├─────────────────────────────────────────────────────────────────┤
│  【运行时基础中间件 - build_lead_runtime_middlewares】           │
│  1. ThreadDataMiddleware        初始化工作目录                    │
│  2. UploadsMiddleware           处理上传文件                      │
│  3. SandboxMiddleware           获取沙箱环境                      │
│  4. DanglingToolCallMiddleware  补全缺失的工具响应               │
│  5. LLMErrorHandlingMiddleware  LLM 错误处理与重试               │
│  6. GuardrailMiddleware         工具调用前授权（可选）           │
│  7. SandboxAuditMiddleware      沙箱命令安全审计                 │
│  8. ToolErrorHandlingMiddleware 工具错误处理                     │
│                                                                  │
│  【Lead Agent 专属中间件】                                       │
│  9. SummarizationMiddleware     上下文压缩（可选）               │
│ 10. TodoListMiddleware          任务跟踪（plan 模式可选）        │
│ 11. TokenUsageMiddleware        Token 使用统计（可选）           │
│ 12. TitleMiddleware             自动生成标题                     │
│ 13. MemoryMiddleware            异步记忆更新                     │
│ 14. ViewImageMiddleware         Vision 模型图片注入（可选）      │
│ 15. DeferredToolFilterMiddleware 延迟工具过滤（可选）            │
│ 16. SubagentLimitMiddleware     子任务数量限制（可选）           │
│ 17. LoopDetectionMiddleware     循环检测与阻断                   │
│ 18. ClarificationMiddleware     澄清请求拦截（必须最后）         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**注：** 基础中间件 8 个 + Lead Agent 专属 10 个 = 18 个（其中 6 个为可选配置，根据运行时配置动态加载）。

### 5.5.2 中间件接口

```python
class Middleware(ABC):
    """
    中间件基类
    """
    
    async def process(
        self,
        state: ThreadState,
        messages: List[BaseMessage]
    ) -> MiddlewareResult:
        """
        处理中间件逻辑
        返回：continue（继续）/ suspend（暂停）/ interrupt（中断）
        """
        raise NotImplementedError
```

### 5.5.3 核心中间件详解

#### ThreadDataMiddleware

```python
class ThreadDataMiddleware:
    """
    为每个 Thread 创建独立的工作目录
    """
    
    async def process(self, state: ThreadState) -> MiddlewareResult:
        thread_id = state.get("thread_id")
        
        # 创建目录结构
        thread_data = {
            "workspace": f".deer-flow/threads/{thread_id}/user-data/workspace",
            "uploads": f".deer-flow/threads/{thread_id}/user-data/uploads",
            "outputs": f".deer-flow/threads/{thread_id}/user-data/outputs",
        }
        
        # 确保目录存在
        for path in thread_data.values():
            Path(path).mkdir(parents=True, exist_ok=True)
        
        # 注入到 state
        state["thread_data"] = thread_data
        
        return MiddlewareResult(should_should_continue=True)
```

#### SandboxMiddleware

```python
class SandboxMiddleware:
    """
    获取沙箱环境
    """
    
    async def process(self, state: ThreadState) -> MiddlewareResult:
        # 获取沙箱提供者
        sandbox_provider = get_sandbox_provider()
        
        # 获取沙箱实例
        sandbox = await sandbox_provider.acquire()
        
        # 将沙箱信息注入 state
        state["sandbox"] = {
            "sandbox_id": sandbox.id,
            "sandbox_type": sandbox.type,
        }
        
        return MiddlewareResult(should_should_continue=True)
```

#### GuardrailMiddleware

```python
class GuardrailMiddleware:
    """
    工具调用前授权检查
    """
    
    def __init__(self, guardrail_provider: GuardrailProvider):
        self.guardrail = guardrail_provider
    
    async def process(
        self,
        state: ThreadState,
        tool_call: ToolCall
    ) -> MiddlewareResult:
        """
        在工具执行前进行授权检查
        """
        # 调用 Guardrail Provider 评估
        decision = await self.guardrail.evaluate(tool_call, state)
        
        if decision.allowed:
            return MiddlewareResult(should_should_continue=True)
        else:
            # 返回错误消息，中断执行
            return MiddlewareResult(
                should_continue=False,
                interrupt=True,
                error_message=decision.reason
            )
```

#### SummarizationMiddleware

```python
class SummarizationMiddleware:
    """
    当上下文接近 token 限制时，触发压缩
    """
    
    async def process(self, state: ThreadState) -> MiddlewareResult:
        messages = state["messages"]
        
        # 计算当前 token 数
        current_tokens = count_tokens(messages)
        
        # 检查是否需要压缩
        if current_tokens > self.threshold:
            # 触发上下文压缩
            compressed = await self.summarizer.compress(messages)
            state["messages"] = compressed
            
            return MiddlewareResult(
                should_continue=True,
                metadata={"compressed": True}
            )
        
        return MiddlewareResult(should_should_continue=True)
```

#### ClarificationMiddleware

```python
class ClarificationMiddleware:
    """
    拦截澄清请求，必须放在最后
    """
    
    async def process(
        self,
        state: ThreadState,
        last_message: AIMessage
    ) -> MiddlewareResult:
        """
        检查是否是澄清请求
        """
        if last_message.tool_calls:
            for tool_call in last_message.tool_calls:
                if tool_call.name == "ask_clarification":
                    # 拦截并中断
                    return MiddlewareResult(
                        should_continue=False,
                        interrupt=True,
                        goto=END  # 结束当前轮次
                    )
        
        return MiddlewareResult(should_should_continue=True)
```

### 5.5.4 新增中间件详解

#### ToolErrorHandlingMiddleware

```python
class ToolErrorHandlingMiddleware(AgentMiddleware[AgentState]):
    """Convert tool exceptions into error ToolMessages so the run can continue."""
    # 中文:将工具异常转换为错误 ToolMessage,让运行可以继续。

    def wrap_tool_call(
        self,
        request: ToolCallRequest,
        handler: Callable[[ToolCallRequest], ToolMessage | Command],
    ) -> ToolMessage | Command:
        try:
            return handler(request)
        except GraphBubbleUp:
            # Preserve LangGraph control-flow signals (interrupt/pause/resume).
            raise
        except Exception as exc:
            # 构建错误 ToolMessage，让 Agent 可以继续执行
            return self._build_error_message(request, exc)
```

**功能：**
- 捕获工具执行异常，避免整个 Agent 运行中断
- 将异常转换为包含错误信息的 `ToolMessage`
- 保留 LangGraph 控制流信号（interrupt/pause/resume）
- 错误信息截断至 500 字符，避免污染上下文

#### LoopDetectionMiddleware

```python
class LoopDetectionMiddleware(AgentMiddleware[AgentState]):
    """Detects and breaks repetitive tool call loops.
    中文:检测并打断重复的工具调用循环。

    Args:
        warn_threshold: 重复次数达到此值时注入警告。默认: 3。
        hard_limit: 重复次数达到此值时强制停止。默认: 5。
        window_size: 滑动窗口大小。默认: 20。
    """

    _DEFAULT_WARN_THRESHOLD = 3  # 3次警告
    _DEFAULT_HARD_LIMIT = 5      # 5次强制停止
```

**检测策略：**
1. 对每次模型响应的工具调用计算哈希（name + args）
2. 在滑动窗口中跟踪最近的工具调用哈希
3. **警告阶段（≥3次）：** 注入 "你正在重复调用工具，请总结结果" 的提示
4. **强制停止（≥5次）：** 清空 tool_calls，强制模型生成最终文本回答

**特殊处理：**
- `read_file`：按 200 行分桶，避免行号微小差异导致误判
- `write_file`/`str_replace`：对完整参数哈希，区分不同内容写入
- 警告以 `HumanMessage` 形式注入（避免 Anthropic 系统消息位置限制）

#### TokenUsageMiddleware

```python
class TokenUsageMiddleware(AgentMiddleware):
    """Logs token usage from model response usage_metadata."""
    # 中文:记录模型响应的 token 使用量(取自 usage_metadata)。

    def _log_usage(self, state: AgentState) -> None:
        messages = state.get("messages", [])
        if not messages:
            return None
        last = messages[-1]
        usage = getattr(last, "usage_metadata", None)
        if usage:
            logger.info(
                "LLM token usage: input=%s output=%s total=%s",
                usage.get("input_tokens", "?"),
                usage.get("output_tokens", "?"),
                usage.get("total_tokens", "?"),
            )
```

**功能：**
- 在 `after_model` 钩子中记录 Token 使用量
- 从 `usage_metadata` 提取 input/output/total tokens
- 仅在 `app_config.token_usage.enabled` 为 true 时加载

#### DeferredToolFilterMiddleware

```python
class DeferredToolFilterMiddleware(AgentMiddleware[AgentState]):
    """Remove deferred tools from request.tools before model binding.
    中文:在模型绑定前,从 request.tools 中移除延迟加载的工具。

    ToolNode still holds all tools (including deferred) for execution routing,
    but the LLM only sees active tool schemas — deferred tools are discoverable
    via tool_search at runtime.
    """

    def _filter_tools(self, request: ModelRequest) -> ModelRequest:
        registry = get_deferred_registry()
        if not registry:
            return request

        deferred_names = {e.name for e in registry.entries}
        active_tools = [t for t in request.tools 
                       if getattr(t, "name", None) not in deferred_names]
        return request.override(tools=active_tools)
```

**功能：**
- 在 `wrap_model_call` 中过滤延迟加载的工具 schema
- 减少发送给 LLM 的上下文 tokens
- 延迟工具可通过 `tool_search` 在运行时发现
- 仅在 `app_config.tool_search.enabled` 为 true 时加载

#### SandboxAuditMiddleware

```python
class SandboxAuditMiddleware(AgentMiddleware[ThreadState]):
    """Bash command security auditing middleware.
    中文:Bash 命令安全审计中间件。

    1. Command classification: regex + shlex 分析命令风险等级
    2. Audit log: 每个 bash 调用记录为结构化 JSON
    3. High-risk commands 被阻断，返回错误 ToolMessage
    4. Medium-risk commands 执行但附加警告
    """
```

**风险分类规则：**

| 等级 | 触发模式 | 处理方式 |
|------|----------|----------|
| **Block** | `rm -rf /`, `curl \| bash`, `mkfs`, `dd if=`, `base64 -d \|`, `/dev/tcp/`, fork bomb 等 | 阻断执行，返回错误消息 |
| **Warn** | `chmod 777`, `pip install`, `apt install`, `sudo`, `PATH=` 等 | 执行并附加警告提示 |
| **Pass** | 其他命令 | 正常执行 |

**输入校验：**
- 最大命令长度：10,000 字符（超出则阻断）
- 禁止 null 字节
- 支持复合命令解析（`&&`, `||`, `;`）

#### LLMErrorHandlingMiddleware

```python
class LLMErrorHandlingMiddleware(AgentMiddleware[AgentState]):
    """Retry transient LLM errors and surface graceful assistant messages.
    中文:重试临时性 LLM 错误,并呈现友好的助手消息。

    可重试错误：APITimeoutError, APIConnectionError, InternalServerError,
    状态码 408/409/425/429/500/502/503/504, 服务繁忙提示等
    """

    retry_max_attempts: int = 3
    retry_base_delay_ms: int = 1000
    retry_cap_delay_ms: int = 8000
```

**错误分类与处理：**

| 错误类型 | 识别模式 | 是否重试 | 用户提示 |
|----------|----------|----------|----------|
| Quota | `insufficient_quota`, `billing`, `credit`, `余额不足` | ❌ | "账户额度不足，请检查计费状态" |
| Auth | `authentication`, `unauthorized`, `invalid api key`, `未授权` | ❌ | "认证失败，请检查 API 密钥" |
| Busy | `server busy`, `overloaded`, `rate limit`, `服务繁忙` | ✅ | "服务暂时不可用，请稍后重试" |
| Transient | Timeout, ConnectionError, 5xx | ✅ | 指数退避重试 |

**重试机制：**
- 最大重试次数：3 次
- 退避策略：指数退避（1s → 2s → 4s），上限 8s
- 支持 `Retry-After` 响应头
- 发送 `llm_retry` 流式事件供前端展示

## 5.6 工具系统（Tools）

### 5.6.1 工具加载流程

```
get_available_tools()
       │
       ├── 1. Sandbox Tools (bash, ls, read, write, str_replace)
       ├── 2. Builtin Tools (present_files, ask_clarification, view_image)
       ├── 3. MCP Tools (从配置的 MCP Server 加载)
       ├── 4. Community Tools (tavily, jina_ai, firecrawl, image_search)
       └── 5. Subagent Tools (task delegation)
```


> 上图是简化视角，真实加载只有 4 个来源，且**沙箱与社区走同一条路**：
>
> ```
> get_available_tools() = 配置工具 + 内置工具 + MCP 工具 + ACP 工具
>   ├─ 配置工具：config.yaml tools: 段 → resolve_variable("模块:变量")
>   │    └─ 同时装载：沙箱（ls/read_file/glob/grep/write_file/str_replace/bash）
>   │                 社区（web_search/web_fetch/image_search …）
>   ├─ 内置工具：BUILTIN_TOOLS + 条件追加
>   │    └─ present_files / ask_clarification / review_skill_package
>   │       （+ view_image 仅 vision 模型；task 仅 subagent_enabled 等）
>   ├─ MCP 工具：启动时 initialize_mcp_tools() 缓存
>   └─ ACP 工具：config.acp_agents
> 之后：按 name 去重 → 授权过滤 → tool_search 延迟化 → create_agent 绑定
> ```
>
> 原文两处笔误：① 沙箱工具名应为 `read_file`/`write_file`（非 read/write），且漏了 `glob`、`grep`；② "Community Tools (tavily, jina_ai, firecrawl…)" 列的是**包名**，工具对外名是 `web_search`、`web_fetch` 等。

> **补充：MCP 工具的完整链路（配置 → 连接 → 缓存 → 取用）**
>
> 上图"3. MCP Tools（从配置的 MCP Server 加载）"分两个层面，容易混淆：
>
> - **你配置的是"服务器"，不是"工具"**：在 `extensions_config.json` 的 `mcpServers` 段添加的是服务器条目（command / args / env，即"怎么启动它"）。你写不出它的工具——工具长什么样，要连上服务器才知道。
> - **工具是连接后动态发现的**，流程如下：
>
> ```
> ① 配置：extensions_config.json → mcpServers 段添加服务器条目
> ② 启动或首次调用：initialize_mcp_tools()（deerflow/mcp/cache.py）
>    逐个连接启用的服务器，询问 tools/list
> ③ 发现：服务器返回它提供的工具（如 github_get_issue、github_create_issue …）
> ④ 缓存：工具存入内存缓存
> ⑤ 取用：get_available_tools() 组装时调 get_cached_mcp_tools()
>    从缓存取出；若配置有变（缓存过期）或尚未初始化，会自动重新初始化
> ```
>
> **对比沙箱/社区工具**：它们的 `use: 模块:变量` 直接指向本地代码函数，import 即得工具；
> MCP 工具指向**外部服务器**，必须"连接 → 询问"才能拿到工具，缓存只是避免每次重复连接。

### 5.6.2 工具注册表

> 本节为旧文档遗留，代码中不存在 `ToolRegistry` 类。
> DeerFlow 的工具就是 `list[BaseTool]`，加载后按 `name` 去重（tools/tools.py），
> 无注册表机制。请以 5.6.1 勘误框中的流程为准。

```python
class ToolRegistry:
    """
    工具注册表
    """
    
    def __init__(self):
        self._tools: Dict[str, BaseTool] = {}
    
    def register(self, name: str, tool: BaseTool):
        self._tools[name] = tool
    
    def get(self, name: str) -> Optional[BaseTool]:
        return self._tools.get(name)
    
    def list_all(self) -> List[BaseTool]:
        return list(self._tools.values())
```

### 5.6.3 内置工具

| 工具 | 功能 |
|------|------|
| `bash` | 执行 Shell 命令 |
| `ls` | 目录列表（树形，最大2层） |
| `read_file` | 读取文件内容（支持行范围） |
| `write_file` | 写入/追加文件 |
| `str_replace` | 字符串替换编辑 |
| `present_files` | 展示生成的文件 |
| `ask_clarification` | 请求用户澄清 |
| `view_image` | 查看图片 |

## 5.7 系统 Prompt 生成

```python
def apply_prompt_template(
    base_template: str,
    skills: List[Skill],
    memory_context: str,
    subagent_instructions: str,
    config: RunnableConfig
) -> str:
    """
    组装完整的系统 Prompt
    """
    parts = [base_template]
    
    # 1. Skills 指令
    if skills:
        skills_section = "\n\n## Available Skills\n"
        for skill in skills:
            skills_section += f"- **{skill.name}**: {skill.description}\n"
            skills_section += f"  Usage: {skill.usage}\n"
        parts.append(skills_section)
    
    # 2. Memory 上下文
    if memory_context:
        parts.append(f"\n\n## Memory Context\n{memory_context}\n")
    
    # 3. Subagent 指令
    if subagent_instructions:
        parts.append(f"\n\n## Subagent Delegation\n{subagent_instructions}\n")
    
    return "\n".join(parts)
```

## 5.8 Client SDK：DeerFlowClient

DeerFlow 提供嵌入式 Python 客户端 `DeerFlowClient`，无需启动 LangGraph Server 或 Gateway API 即可直接调用 Agent 能力。

### 5.8.1 基本使用

```python
from deerflow.client import DeerFlowClient

# 创建客户端
client = DeerFlowClient()

# 简单对话
response = client.chat("分析这篇论文", thread_id="my-thread")
print(response)

# 流式输出
for event in client.stream("你好"):
    print(event.type, event.data)
```

### 5.8.2 初始化配置

```python
client = DeerFlowClient(
    config_path="config.yaml",           # 配置文件路径
    checkpointer=checkpointer,           # LangGraph checkpointer（多轮对话必需）
    model_name="claude-sonnet-4-6",      # 覆盖默认模型
    thinking_enabled=True,               # 启用扩展思考
    subagent_enabled=True,               # 启用子任务委托
    plan_mode=True,                      # 启用 TodoList 计划模式
    agent_name="my-agent",               # 指定自定义 Agent
    available_skills={"research", "code"}, # 限定可用 Skills
    middlewares=[custom_middleware],     # 注入自定义中间件
)
```

**重要提示：**
- 多轮对话需要传入 `checkpointer`，否则每次调用都是独立的（`thread_id` 仅用于文件隔离）
- Agent 在首次调用时延迟创建，配置参数变更时会自动重建
- 调用 `reset_agent()` 可强制刷新 Agent（如 Skill 更新后）

### 5.8.3 chat() 方法

```python
def chat(self, message: str, *, thread_id: str | None = None, **kwargs) -> str:
    """Send a message and return the final text response.

    这是 stream() 的便利包装，只返回最后一个 AI 文本消息。
    如果一轮中产生多段文本，中间段落会被丢弃。
    """
```

**使用示例：**
```python
# 简单调用
reply = client.chat("解释量子计算")

# 覆盖配置参数
reply = client.chat(
    "解释量子计算",
    thread_id="session-123",
    model_name="gpt-4o",
    thinking_enabled=False,
    plan_mode=True,
)
```

### 5.8.4 stream() 方法

```python
def stream(
    self,
    message: str,
    *,
    thread_id: str | None = None,
    **kwargs,
) -> Generator[StreamEvent, None, None]:
    """Stream a conversation turn, yielding events incrementally.

    事件类型与 LangGraph SSE 协议对齐，支持 HTTP 流式和嵌入式模式切换。
    """
```

**事件类型：**

| 事件类型 | 数据格式 | 说明 |
|----------|----------|------|
| `values` | `{title, messages, artifacts}` | 完整状态快照 |
| `messages-tuple` | `{type, content, id, ...}` | 单条消息更新 |
| `custom` | `{...}` | 自定义事件 |
| `end` | `{usage: {input_tokens, output_tokens, total_tokens}}` | 流结束 |

**使用示例：**
```python
for event in client.stream("分析数据", thread_id="thread-1"):
    if event.type == "messages-tuple":
        data = event.data
        if data.get("type") == "ai":
            print(f"AI: {data.get('content', '')}")
        elif data.get("type") == "tool":
            print(f"Tool {data.get('name')}: {data.get('content', '')[:100]}...")
    elif event.type == "values":
        print(f"Artifacts: {event.data.get('artifacts', [])}")
    elif event.type == "end":
        usage = event.data.get("usage", {})
        print(f"Tokens: {usage.get('total_tokens', 0)}")
```

### 5.8.5 Agent 缓存机制

```python
def _ensure_agent(self, config: RunnableConfig):
    """Create (or recreate) the agent when config-dependent params change."""
    cfg = config.get("configurable", {})
    key = (
        cfg.get("model_name"),
        cfg.get("thinking_enabled"),
        cfg.get("is_plan_mode"),
        cfg.get("subagent_enabled"),
        self._agent_name,
        frozenset(self._available_skills) if self._available_skills else None,
    )

    if self._agent is not None and self._agent_config_key == key:
        return  # 使用缓存的 Agent

    # 重建 Agent
    self._agent = create_agent(...)
    self._agent_config_key = key
```

**缓存 key 包含：**
- `model_name`：模型变更需重建
- `thinking_enabled`：思考模式影响模型创建
- `is_plan_mode`：影响中间件链
- `subagent_enabled`：影响工具集和中间件
- `agent_name`：影响 system prompt 和 memory
- `available_skills`：影响 system prompt

### 5.8.6 其他 API

```python
# 配置查询
client.list_models()           # 列出可用模型
client.list_skills()           # 列出可用 Skills
client.get_model(name)         # 获取指定模型配置
client.get_mcp_config()        # 获取 MCP 配置
client.update_mcp_config({...}) # 更新 MCP 配置

# Memory 管理
client.get_memory()            # 获取记忆数据
client.export_memory()         # 导出记忆
client.import_memory(data)     # 导入记忆
client.reload_memory()         # 重载记忆
client.clear_memory()          # 清空记忆
client.create_memory_fact("内容", category="context")
client.delete_memory_fact(fact_id)
client.update_memory_fact(fact_id, content="新内容")

# 文件上传
client.upload_files(thread_id, ["/path/to/file.pdf", "/path/to/image.png"])
client.list_uploads(thread_id)
client.delete_upload(thread_id, "file.pdf")

# Artifacts
client.get_artifact(thread_id, "mnt/user-data/outputs/result.txt")
```

## 5.9 DeerFlow 二次开发：自定义 Lead Agent

> **🏢 企业级建议**：自定义 Agent 工厂时，建议保留原始 `make_lead_agent` 的调用链作为 fallback，并通过特性开关（feature flag）控制新旧 Agent 的切换，降低上线风险。

> 📌 **这句话的白话解读**（结合 DeerFlow 真实代码）：
>
> **一句话版本**：新工厂写好了先别急着替换，把老的 `make_lead_agent` 留着当备胎；用一个开关决定今天跑哪个；出问题就把开关拨回去，秒级回滚，不用重新发版。
>
> **逐词拆解**：
>
> | 词 | 意思 |
> |---|---|
> | 自定义 Agent 工厂 | 按 5.9.3 自己写一个 `make_custom_agent()`（换了编排方式、加了企业中间件），和 `make_lead_agent` 一样输入 config、输出编译好的图 |
> | 保留原始调用链作为 fallback | 新工厂上线后，旧的 `make_lead_agent` 调用链一条都不要删——它是"备胎"，新工厂出问题随时切回去 |
> | 特性开关（feature flag） | 配置文件里加一个开关，如 `agent_factory: "new"` / `"legacy"`，代码读开关决定调哪个工厂 |
> | 降低上线风险 | 新工厂先灰度（小流量试），出问题拨开关秒回滚——不用发版、不用改代码、不影响线上 |
>
> 伪代码示意：
>
> ```python
> def resolve_agent_factory():
>     if config.agent_factory == "custom":   # 特性开关
>         return make_custom_agent           # 新工厂（实验性）
>     return make_lead_agent                 # fallback：默认老工厂，永远保留
> ```
>
> **DeerFlow 自己就是这么做的（证据）**：
>
> ① worker 里有真实的"调用链回退"逻辑（`runtime/runs/worker.py`）：检查工厂签名认不认识 `app_config` 这个新参数，不认识就走旧调用方式传参；配套 `test_run_worker_rollback.py` 测试钉死回滚行为——工厂升级了参数，老工厂/老调用方式依然能跑。
>
> ② `resolve_agent_factory`（`app/gateway/services.py`）展示了更保守的默认设计：DeerFlow 里自定义 agent 根本不换工厂，全部复用同一个 `make_lead_agent`，只通过 `agent_name` 参数区分——"工厂永远只有一个、路由在工厂内部"，比 5.9 的建议还要稳。
>
> **完整图景**：这句话是在教你——换引擎可以，但别把旧引擎扔了；装个换挡杆（开关），随时能切回旧引擎。而 DeerFlow 的实际工程选择是——默认连引擎都不换，只换挡位（`agent_name` 参数）；真要换引擎，worker 也留着旧调用方式的回退通道。

### 5.9.1 扩展 ThreadState

```python
# 定制 ThreadState
class CustomThreadState(ThreadState):
    # 原有字段...
    
    # 新增：自定义字段
    project_id: Optional[str]        # 当前项目
    organization_id: Optional[str]   # 组织 ID
    user_role: Optional[str]         # 用户角色
    approval_queue: List[Approval]  # 待审批队列
    audit_context: AuditContext     # 审计上下文
```

### 5.9.2 添加企业中间件

```python
class CustomMiddlewareChain:
    """
    定制中间件链
    """
    
    @staticmethod
    def build() -> List[Middleware]:
        return [
            ThreadDataMiddleware(),           # 1. 原有
            RBACMiddleware(),                  # 2. 新增：权限检查
            UploadsMiddleware(),              # 3. 原有
            SandboxMiddleware(),              # 4. 原有
            ApprovalMiddleware(),             # 5. 新增：审批检查
            GuardrailMiddleware(),            # 6. 原有
            SummarizationMiddleware(),        # 7. 原有
            AuditLoggerMiddleware(),          # 8. 新增：审计日志
            TodoListMiddleware(),              # 9. 原有
            TitleMiddleware(),                # 10. 原有
            MemoryMiddleware(),               # 11. 原有
            ViewImageMiddleware(),            # 12. 原有
            SubagentLimitMiddleware(),        # 13. 原有
            ClarificationMiddleware(),        # 14. 原有
        ]
```

> 📌 **怎么写中间件:继承只是"报名",覆写钩子才是"干活"**(上面的 RBACMiddleware、ApprovalMiddleware、AuditLoggerMiddleware 就按这个模板写):
>
> **① 前提**:继承 `AgentMiddleware` 只是"报名",真正起作用的是你覆写了哪些"钩子方法"。基类所有钩子都是空实现(透传);LangGraph 编译时逐个检查——你覆写了哪个,哪个位置才生效。一个钩子都不覆写的中间件 = 白报名,编译后连节点都不会生成。
>
> **② 六个钩子 = 六个"执行插槽",想干什么覆写哪个**:
>
> | 你想要的 | 覆写 | 在哪执行 | 真实范本 |
> |---|---|---|---|
> | 模型调用前改状态(注入提醒/校验输入) | `before_model(state, runtime) -> dict \| None` | before_model 段,每轮循环入口 | TodoMiddleware(读 state["todos"],返回 {"messages": [提醒]}) |
> | 模型调用后收尾(生成标题/记 token) | `after_model(state, runtime) -> dict \| None` | after_model 段,每轮逆序 | TitleMiddleware(返回标题更新) |
> | 对话开始时初始化(建目录/拿沙箱) | `before_agent(state)` | before_agent 段,首轮一次 | ThreadDataMiddleware |
> | 对话结束时清理(写记忆) | `after_agent(state)` | after_agent 段,末轮一次 | MemoryMiddleware |
> | 包住模型调用(重试/消毒) | `wrap_model_call(request, handler)` | model 节点内部洋葱 | LLMErrorHandlingMiddleware |
> | 包住工具执行(拦截/审计/改结果) | `wrap_tool_call(request, handler)` | tools 节点内部洋葱 | GuardrailMiddleware |
>
> 注意签名规律:**返回 `dict` = 往状态里写更新;返回 `None` = 啥也不改**;`wrap_*` 的 `handler` = "里面那层 + 真正干活",不调 handler 就拦截成功了。
>
> **③ 从零写一个(模板)**:
>
> ```python
> class MyAuditMiddleware(AgentMiddleware):
>     """示例:①每轮模型调用前记录消息数;②拦截危险工具。"""
>
>     def before_model(self, state, runtime) -> dict | None:
>         n = len(state.get("messages", []))
>         print(f"[我的中间件] 本轮开始时已有 {n} 条消息")
>         return None                      # 只观察,不改状态
>
>     def wrap_tool_call(self, request, handler):
>         if request.tool_call.get("name") == "bash" and "rm -rf" in str(request.tool_call.get("args", {})):
>             return ToolMessage(content="拦截:禁止 rm -rf", tool_call_id=request.tool_call["id"])
>         return handler(request)          # 放行:调用里面真正执行
> ```
>
> **④ 它到底在哪执行?**:`before_model` 编译时被挂到 before_model 段的一个图节点上;`wrap_tool_call` 包在 tools 节点内部。图跑起来时,LangGraph 按 5.4 那张图的顺序自动调用它们——执行时机由钩子名决定,挂载位置由编译器决定,**你的代码里没有任何手动调用的地方**。
>
> **⑤ 怎么让它生效(三种方式)**:① 改代码:`build_middlewares(custom_middlewares=[MyAuditMiddleware()])`(插到 ClarificationMiddleware 之前);② 用客户端:`DeerFlowClient(middlewares=[...])`;③ 做成扩展:通过 `deerflow_extension_api` 的 `MiddlewareContributor.contribute_middlewares` + `MiddlewarePlacement` 声明位置(第三方插件官方路径,不用 fork 代码)。
>
> **一句话总结**:继承 = 报名,覆写钩子 = 干活,钩子名 = 执行时机,编译器 = 自动挂载;你只管写"想做的事",不用管"在哪调用"。

### 5.9.3 自定义 Agent 工厂

```python
def make_custom_agent(config: RunnableConfig) -> CompiledGraph:
    """
    定制 Agent 工厂
    """
    # 1. 构建中间件链
    middlewares = CustomMiddlewareChain.build()
    
    # 2. 创建 Agent
    workflow = StateGraph(CustomThreadState)
    
    # 3. 添加节点
    workflow.add_node("middlewares", run_middleware_chain(middlewares))
    workflow.add_node("model", model_node)
    workflow.add_node("tools", tools_node)
    
    # 4. 添加边
    workflow.add_edge(START, "middlewares")
    workflow.add_edge("middlewares", "model")
    workflow.add_conditional_edges("model", should_continue, {...})
    workflow.add_edge("tools", "model")
    
    # 5. 编译
    return workflow.compile()
```

## 5.10 小结

DeerFlow Agent 核心要点：

| 组件 | 核心机制 |
|------|----------|
| **状态管理** | ThreadState 扩展 LangGraph AgentState |
| **中间件链** | 14+ 个中间件按序执行，可插拔 |
| **工具系统** | Sandbox/Builtin/MCP/Community/Subagent 五类 |
| **配置驱动** | configurable 运行时注入 |
| **Prompt 组装** | 动态拼接 Skills/Memory/Subagent |

开发者可在此基础上：
- 扩展 ThreadState 添加自定义字段
- 在中间件链中添加自定义中间件
- 自定义工具集满足特定需求


## 本章小结

本章深入解析了 DeerFlow Agent 核心的实现细节：

1. **make_lead_agent**：工厂函数完成模型选择、工具加载、Prompt 生成、中间件链组装四大初始化步骤。
2. **ThreadState 管理**：TypedDict 定义的状态结构，支持 SandboxState、ThreadDataState 等扩展，通过 Annotated 实现状态归约。
3. **LangGraph 编排**：节点（Node）定义执行单元，边（Edge）定义流转规则，条件边实现动态分支（如 should_continue）。
4. **中间件链**：Middleware 在请求前/后执行，支持日志、限流、审计等横切关注点，通过 MiddlewareResult 控制流程。

> **⚠️ 注意**：should_continue 返回值必须与节点名称精确匹配（区分大小写），否则会导致 LangGraph 运行时报错。

---

**下一步**：阅读第六章，掌握 Skill 与 Tool 的抽象层级与注册机制。

