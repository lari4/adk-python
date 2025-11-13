# ADK Pipeline Quick Reference Guide

## 1. Pipeline Type Selection Matrix

| Agent Type | Use Case | Parent-Child Transfer | Peer Transfer | Iteration | Concurrency |
|---|---|---|---|---|---|
| **LlmAgent (SingleFlow)** | Single agent with tools | N/A | N/A | N/A | No |
| **LlmAgent (AutoFlow)** | Multi-agent delegation | Yes | Yes | No | No |
| **SequentialAgent** | Multi-step workflow | N/A | N/A | Sequential | No |
| **ParallelAgent** | Concurrent agents | N/A | N/A | No | Yes |
| **LoopAgent** | Iterative refinement | N/A | N/A | Yes (max_iter) | No |

## 2. Instruction Flow Decision Tree

```
Do you have static_instruction?
├─ YES
│  ├── static_instruction → system_instructions
│  └── instruction → user_content (after static)
│      └── Enables context caching optimization
├─ NO
│  └── instruction → system_instructions
│      └── instruction can be dynamic
│          ├── String literal
│          ├── InstructionProvider (callable)
│          │   └── Can access ctx.session.state
│          └── Variable substitution {variable_name}
```

## 3. Tool Execution Flow

```
Function Call from LLM
  ↓
[before_tool_callback?] → Can override tool result
  ├─ Returns content → Use instead of tool
  └─ Returns None → Continue
  ↓
[Tool Execution]
  ├─ Parallel if independent
  ├─ Sequential if dependent
  └─ Streaming if supported
  ↓
[Tool Success?]
  ├─ YES
  │  └── [after_tool_callback?] → Transform result
  └─ NO
     └── [on_tool_error_callback?] → Handle error
  ↓
[Return to LLM?]
  ├─ More function calls needed → Repeat
  ├─ Auth required → adk_request_credential
  ├─ Confirmation needed → adk_request_confirmation
  ├─ Output schema → Structured response
  └─ Complete → Return to user
```

## 4. Request Processor Order (CRITICAL)

Must be executed in this order for correct behavior:

1. **basic** - Model, config setup
2. **auth_preprocessor** - Auth handlers
3. **request_confirmation** - Confirmation logic
4. **instructions** - Agent instructions (can read from processors 1-3)
5. **identity** - Agent identity
6. **contents** - Conversation history (uses instructions from #4)
7. **context_cache** - Cache setup
8. **nl_planning** - Planning instructions (reads instruction state from #4)
9. **code_execution** - Code prep (uses contents from #6)
10. **output_schema** - Final schema setup

Violating this order breaks functionality.

## 5. Agent State for Resumability

```python
# Savepoint pattern
if ctx.is_resumable:
    state = SequentialAgentState(
        current_sub_agent=sub_agent.name,
        times_looped=loop_count
    )
    ctx.set_agent_state(agent.name, agent_state=state)
    yield self._create_agent_state_event(ctx)

# Recovery pattern
agent_state = agent._load_agent_state(ctx, SequentialAgentState)
if agent_state:
    # Resume from saved state
    start_index = find_index(agent_state.current_sub_agent)
else:
    # Fresh start
    start_index = 0
```

## 6. Planner Selection Guide

| Planner | Use Case | Setup | Processing |
|---|---|---|---|
| **None** | Standard Q&A | Just instruction | Normal response |
| **PlanReActPlanner** | Complex tasks needing reasoning | Planning tags in instruction | Extract plan/reasoning/action |
| **BuiltInPlanner** | Model native thinking | ThinkingConfig with budget_tokens | Automatic, no post-processing |

## 7. Content Inclusion Options

```python
agent = LlmAgent(
    include_contents='default',  # Case A: Full history
    ...
)
# → Session history from current branch included
# → Good for conversational agents
# → More context for LLM

agent = LlmAgent(
    include_contents='none',  # Case B: Current turn only
    ...
)
# → Only last user message + instruction
# → Good for independent agents
# → Lower cost, less context
```

## 8. Branch Isolation Pattern

```
agent_tree:
    root (branch = None)
    ├── agent_a (branch = "root.agent_a")
    └── agent_b (branch = "root.agent_b")

Benefits:
  - agent_a doesn't see agent_b's history
  - Each sub-agent has clean context
  - Enables true delegation
  - No interference between peers
```

## 9. Live API Setup Checklist

```python
run_config = RunConfig(
    support_cfc=True,  # Enable live API
    response_modalities=[types.Modality.AUDIO],
    speech_config=types.SpeechConfig(...),
    save_live_blob=True,  # Cache audio
)

# Flow:
invocation_context.run_config = run_config
# → LlmFlow detects support_cfc=True
# → Uses run_live() instead of run_async()
# → Sets up LiveRequestQueue
# → Connects to model via live API
```

## 10. Callback Injection Points

```
Before Agent Run
  └─ before_agent_callback()
     └─ Can: Skip agent, modify state

Before LLM Call
  └─ before_model_callback()
     └─ Can: Modify LlmRequest, override response

LLM Call
  └─ LLM.generate_content()

After LLM Call
  └─ after_model_callback()
     └─ Can: Modify LlmResponse

Before Tool Call
  └─ before_tool_callback()
     └─ Can: Validate args, override result

Tool Execution
  └─ Tool.execute()

After Tool Call
  └─ after_tool_callback()
     └─ Can: Transform result

After Agent Run
  └─ after_agent_callback()
     └─ Can: Add additional content
```

## 11. Common Patterns

### Pattern 1: Self-Correcting Loop
```python
loop_agent = LoopAgent(
    name="corrector",
    max_iterations=3,
    sub_agents=[generate_agent, validate_agent]
)
# Generates → Validates → Generates → ...
```

### Pattern 2: Hierarchical Delegation
```python
root_agent = LlmAgent(name="dispatcher")
root_agent.sub_agents = [
    specialized_a,
    specialized_b,
    specialized_c
]
# Root transfers to specialists based on task
```

### Pattern 3: Parallel Consensus
```python
parallel_agent = ParallelAgent(
    name="consensus",
    sub_agents=[expert_a, expert_b, expert_c]
)
# All experts analyze in parallel
# Results merged for decision
```

### Pattern 4: Data Pipeline
```python
seq_agent = SequentialAgent(
    name="etl_pipeline",
    sub_agents=[
        extract_agent,
        transform_agent,
        load_agent,
        verify_agent
    ]
)
# Data flows through pipeline in order
```

## 12. State Variable Access Patterns

```python
# In instructions (with placeholders)
instruction = "You are helping {user_name} with {task_category}"
# Placeholders replaced at runtime

# In instruction provider (callable)
async def dynamic_instruction(ctx: ReadonlyContext) -> str:
    user_level = ctx.session.state.get('user_level', 'beginner')
    if user_level == 'expert':
        return "Use advanced technical terms..."
    else:
        return "Explain in simple terms..."

# In callbacks
def my_callback(callback_context: CallbackContext) -> Optional[types.Content]:
    user_name = callback_context.session.state.get('user_name')
    callback_context.state.set('previous_action', 'checked_orders')

# In tools
@tool_context.get_session_state('user_context')
def my_tool(tool_context: ToolContext):
    user_context = tool_context.session.state.get('user_context')
```

## 13. Event Streaming Semantics

```python
# Events yielded as soon as available
async for event in agent.run_async(ctx):
    # Process event immediately
    # Don't wait for entire flow to complete
    print(f"Event: {event}")

# Streaming captures:
├── LLM text chunks
├── Thinking/planning stages
├── Function call invocations
├── Tool execution results
├── State changes
└── Final response
```

## 14. Error Handling Patterns

```python
# Model Errors
on_model_error_callback: Optional[OnModelErrorCallback]
# Receives: CallbackContext, LlmRequest, Exception
# Returns: Optional[LlmResponse] to override

# Tool Errors
on_tool_error_callback: Optional[OnToolErrorCallback]
# Receives: BaseTool, args, ToolContext, Exception
# Returns: Optional[dict] tool result

# Agent Errors
try:
    async for event in agent.run_async(ctx):
        ...
except Exception as e:
    # Handle at application level
```

## 15. Performance Optimization Tips

### Reduce LLM Calls
```python
run_config = RunConfig(
    max_llm_calls=5  # Set limit
)
# Prevents runaway loops
```

### Enable Context Caching
```python
agent = LlmAgent(
    static_instruction="<unchanging content>",
    instruction="<dynamic content>",
)
# static_instruction enables caching optimization
```

### Parallelize Operations
```python
parallel_agent = ParallelAgent(
    sub_agents=[a, b, c]  # Run concurrently
)
```

### Reduce History Size
```python
agent = LlmAgent(
    include_contents='none'  # Only current turn
)
# Smaller requests, faster processing
```

### Batch Tool Calls
```python
# Multiple tools in single LLM turn
# Framework executes in parallel if independent
agent = LlmAgent(
    tools=[tool_a, tool_b, tool_c]
)
```

---

## Quick Troubleshooting

| Problem | Cause | Solution |
|---|---|---|
| Agent not transferring | AutoFlow not used | Use AutoFlow instead of SingleFlow |
| Sub-agents see peer history | Branch not isolated | Ensure branch parameter set correctly |
| Instruction not applied | Wrong processor order | Check processor list in flow |
| Tool not executing | Tool not in tools list | Add tool to agent.tools |
| State not persisting | ctx.is_resumable=False | Enable resumability config |
| LLM loops forever | No exit condition | Use max_llm_calls or output_schema |
| Audio not captured | save_live_blob=False | Set in run_config |

