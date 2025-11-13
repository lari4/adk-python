# ADK Python Agent Workflow Pipelines - Executive Summary

## Overview

The Agent Development Kit (ADK) implements a sophisticated, multi-layered architecture for orchestrating LLM-based agents. This analysis provides complete documentation of how prompts flow through the system, different pipeline types, key components, and data flow patterns.

## Key Findings

### 1. Prompt Flow Architecture

Prompts flow through a **9-stage request pipeline** before LLM invocation:

```
Request Processors (9 stages):
1. Basic - Model/config setup
2. Auth - Authentication handlers
3. Confirmation - Tool confirmation
4. Instructions - Agent instructions
5. Identity - Agent identity
6. Contents - Conversation history
7. Context Cache - Cache configuration
8. NL Planning - Planning instructions
9. Code Execution - Code execution prep

LLM Call ↓

Response Processors (2 stages):
1. NL Planning - Process planned responses
2. Code Execution - Execute code blocks
```

**Critical Finding:** The order of request processors is fixed and interdependent. Violating this order breaks functionality.

### 2. Five Core Pipeline Types

#### A. Single Agent Pipeline (SingleFlow)
- **Use:** Standard agent with tools but no sub-agents
- **Characteristics:** Straightforward request → LLM → tools → response
- **File:** `/src/google/adk/flows/llm_flows/single_flow.py`

#### B. Multi-Agent Delegation (AutoFlow)
- **Use:** Dynamic delegation between agents
- **Characteristics:** Agent can transfer to sub-agents, peers, or parent
- **Special:** Uses `transfer_to_agent()` function call
- **File:** `/src/google/adk/flows/llm_flows/auto_flow.py`

#### C. Sequential Agent Pipeline
- **Use:** Execute agents one after another in fixed order
- **Characteristics:** ETL-like processing, state tracking, resumability
- **Iteration:** No, runs once per sequence
- **File:** `/src/google/adk/agents/sequential_agent.py`

#### D. Parallel Agent Pipeline
- **Use:** Execute multiple agents concurrently
- **Characteristics:** Async task groups, event merging, backpressure support
- **Concurrency:** Full async parallelization
- **File:** `/src/google/adk/agents/parallel_agent.py`

#### E. Loop Agent Pipeline
- **Use:** Iterate through agents until completion or max iterations
- **Characteristics:** Configurable max iterations, escalation support
- **Iteration:** Yes, configurable max_iterations
- **File:** `/src/google/adk/agents/loop_agent.py`

### 3. Instruction Processing System

Instructions flow through a **three-tier system**:

```
Instruction Types:
├── static_instruction (types.ContentUnion)
│   └── Never processed, sent literally
│   └── Enables context caching
│   └── Always first in system instructions
│
├── instruction (str or InstructionProvider callable)
│   └── Goes to system instructions if no static_instruction
│   └── Goes to user_content if static_instruction present
│   └── Can be dynamic with placeholder substitution
│   └── Can access session state via InstructionProvider
│
└── global_instruction (deprecated)
    └── Applies to all agents in tree
    └── Only in root agent
```

**Key Pattern:** For context caching optimization:
- Use `static_instruction` for unchanging content
- Use `instruction` for dynamic, user-specific content
- This leverages model provider caching mechanisms

### 4. Planner Architecture

Three planner types handle different reasoning patterns:

#### None (Default)
- Standard Q&A without explicit planning
- Normal response processing

#### Plan-ReAct Planner
- Uses structured tagging system
- Tags: `/*PLANNING*/`, `/*ACTION*/`, `/*REASONING*/`, `/*FINAL_ANSWER*/`
- Extracts and filters response parts
- Good for complex multi-step tasks
- Works without model's native thinking

#### Built-In Planner
- Uses model's native thinking features (extended thinking)
- Applies `ThinkingConfig` with budget_tokens
- Automatic thought generation
- No post-processing needed
- Model provider dependent

### 5. Tool Execution Pipeline

Tools flow through a **callback-wrapped execution pipeline**:

```
LLM Function Call
  ↓
before_tool_callback (can override)
  ↓
Tool Execution (parallel if independent)
  ↓
Success?
  ├─ Yes → after_tool_callback (can transform)
  └─ No → on_tool_error_callback (can handle error)
  ↓
Special Handling:
├─ adk_request_credential (auth needed)
├─ adk_request_confirmation (confirmation needed)
└─ Integration with next LLM turn
```

### 6. State Management

Two parallel state tracking systems:

#### Agent States (Resumability)
```python
agent_states: dict[str, dict[str, Any]]
# Tracks per-agent execution state
# Example: SequentialAgentState.current_sub_agent
# Enables resumable execution from checkpoints
```

#### Session State (Variables)
```python
session.state: dict[str, Any]
# User/context variables
# Accessible in: instructions, callbacks, tools
# Used for dynamic behavior per user/session
```

### 7. Branch Isolation

Multi-agent systems use **branch-based context isolation**:

```
agent_tree:
    root (branch = None)
    ├── agent_a (branch = "root.agent_a")
    │   └── See only own events
    └── agent_b (branch = "root.agent_b")
        └── See only own events
        └── Don't see agent_a's history
```

**Benefit:** Sub-agents get clean context without peer interference.

### 8. Live API Pipeline (Real-time)

Real-time audio/video flow uses **bidirectional streaming**:

```
LiveRequestQueue ← User Input (audio/content)
  ↓
[Send to Model]
  ├── send_realtime() - Audio chunks
  ├── send_content() - Text content
  └── Activity markers

[Receive from Model]
  ├── Audio responses
  ├── Function calls
  ├── Transcriptions
  ├── Session resumption handles
  └── Control events

AudioCacheManager
  ├── Cache input/output audio
  └── Flush on control events
```

### 9. Evaluation Pipeline

Comprehensive testing framework:

```
EvalSet (Test Cases)
  ├── query: str
  ├── reference: str (expected)
  └── expected_tool_use: str
  ↓
[Multiple Runs]
  ├── Run 1-N: Execute agent
  ├── Capture trajectory
  ├── Record metrics
  ↓
[Metric Evaluation]
  ├── Tool Trajectory Score
  ├── Response Evaluation Score
  ├── Response Match Score
  └── Safety Score
  ↓
[Comparison vs Criteria]
  └── Pass/Fail determination
```

## Data Flow Examples

### Example 1: Simple Tool Use
```
User: "What's the weather?"
→ LlmAgent with weather_tool
→ [Preprocessing] → LLM call
→ LLM returns: function_call(weather_api)
→ [Tool Execution] → weather_api()
→ Tool returns: {temp: 72, condition: sunny}
→ [LLM continues] → Summarizes result
→ Final response to user
```

### Example 2: Multi-Agent Delegation
```
User: "Order status and billing?"
→ Root dispatcher agent
→ LLM decides: transfer_to_agent(orders_agent)
→ [Agent Transfer] → New context, branch="dispatcher.orders_agent"
→ orders_agent processes query
→ Returns: "Order #123 is shipped"
→ User continues with billing
```

### Example 3: Plan-ReAct Analysis
```
User: "Analyze CSV data"
→ Agent with PlanReActPlanner
→ LLM outputs:
   /*PLANNING*/ 1. Load CSV 2. Analyze...
   /*ACTION*/ csv.load(file)
   /*REASONING*/ Data has 1000 rows...
   /*FINAL_ANSWER*/ Report summary...
→ [Response Processing] → Extracts parts
→ [Code Execution] → Runs code blocks
→ Final report to user
```

## Key Components

### Critical Files

| Component | File | Purpose |
|---|---|---|
| Base Flow | `/flows/llm_flows/base_llm_flow.py` | Core request/response loop |
| Single Flow | `/flows/llm_flows/single_flow.py` | 9-stage processor chain |
| Auto Flow | `/flows/llm_flows/auto_flow.py` | Agent transfer support |
| Base Agent | `/agents/base_agent.py` | Agent lifecycle management |
| Invocation Context | `/agents/invocation_context.py` | Execution state container |
| Instructions | `/flows/llm_flows/instructions.py` | Instruction injection |
| Functions | `/flows/llm_flows/functions.py` | Tool execution |
| Contents | `/flows/llm_flows/contents.py` | History management |
| Planners | `/planners/*.py` | Planning strategies |
| Evaluation | `/evaluation/agent_evaluator.py` | Testing framework |

### Processor Chain

```python
SingleFlow request_processors = [
    basic,                      # Model setup
    auth_preprocessor,          # Auth handlers
    request_confirmation,       # Confirmation logic
    instructions,               # Instruction injection
    identity,                   # Agent identity
    contents,                   # History building
    context_cache,              # Cache setup
    _nl_planning,               # Planning instructions
    _code_execution,            # Code execution prep
    _output_schema,             # Schema validation
]

SingleFlow response_processors = [
    _nl_planning,               # Plan response processing
    _code_execution,            # Code execution
]
```

## Design Patterns

### 1. Async Generator Pattern
All processing uses async generators for streaming:
```python
async def _run_one_step_async(self, ctx):
    # Yields events as they become available
    # Enables real-time feedback
    # Supports backpressure
```

### 2. Processor Chain Pattern
Pipeline built from composable processors:
```python
class SingleFlow(BaseLlmFlow):
    def __init__(self):
        self.request_processors = [...]  # Ordered list
        self.response_processors = [...]  # Ordered list
```

### 3. Context Propagation
Single InvocationContext flows through entire pipeline:
- All processors receive same context
- Modifications accumulate
- Sub-agent contexts derived from parent

### 4. State Serialization
Agent states can be saved/restored for resumability:
```python
ctx.set_agent_state(agent.name, state)  # Save
agent_state = agent._load_agent_state(ctx, StateType)  # Load
```

## Performance Considerations

### Context Caching
- **static_instruction**: Unchanging, cacheable by model provider
- **instruction**: Dynamic, not cacheable
- **contents**: History, implicitly cached

### Cost Management
- `max_llm_calls`: Limit total LLM invocations
- `include_contents`: Choose 'default' (full history) or 'none' (current turn)
- Tool parallelization: Execute independent tools concurrently

### Optimization Opportunities
- Use ParallelAgent for concurrent operations
- Batch multiple tool calls
- Minimize conversation history with `include_contents='none'`
- Use context caching with static_instruction

## Security/Safety Features

### Authentication
- `before_tool_callback`: Validate before execution
- `adk_request_credential`: Framework-generated auth requests

### Confirmation
- `before_tool_callback`: Pre-execution validation
- `adk_request_confirmation`: User confirmation flow

### Tool Sandboxing
- Code execution in isolated environment
- Tool argument validation against schema
- Error handling with `on_tool_error_callback`

## Conclusion

The ADK implements a remarkably comprehensive agent orchestration system with:

1. **Modular architecture**: Each processor handles specific concern
2. **Flexible agent types**: Single, sequential, parallel, looping, delegating
3. **Extensible callbacks**: Hooks at agent, model, and tool levels
4. **State management**: Resumable execution with checkpointing
5. **Real-time support**: Live API with audio/video and session resumption
6. **Planning integration**: Both native thinking and Plan-ReAct patterns
7. **Testing framework**: Comprehensive evaluation with metrics
8. **Performance optimization**: Caching, parallelization, cost control

All built on async/await patterns for responsiveness and event streaming for real-time feedback.

## Documents Included

1. **adk_pipeline_analysis.md** - Comprehensive 15-section analysis with detailed code examples
2. **adk_quick_reference.md** - Quick lookup guide with matrices, decision trees, and patterns
3. **PIPELINE_ANALYSIS_SUMMARY.md** - This document, executive overview

---

*Generated from thorough analysis of ADK Python codebase*
*Last Updated: 2025-11-13*
