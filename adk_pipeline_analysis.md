# ADK Python Agent Workflow Pipelines - Comprehensive Analysis

## Executive Summary

The Agent Development Kit (ADK) is a sophisticated system for building multi-agent applications. The core architecture consists of **Agent Workflows**, **LLM Flows**, **Instruction Processors**, **Planners**, and **Evaluation Pipelines**. Each component orchestrates prompts, tool calls, and multi-agent delegation through a series of async processors that transform data at each stage.

---

## 1. OVERALL PROMPT FLOW ARCHITECTURE

### 1.1 End-to-End Prompt Journey

```
User Input
    ↓
InvocationContext Creation
    ↓
[Agent Invocation]
    ├── before_agent_callback (optional)
    ├── LLM Flow Execution
    │   ├── Request Preprocessing (9+ processors)
    │   ├── LLM Call
    │   ├── Response Postprocessing (2+ processors)
    │   └── Tool Execution Loop (if function calls)
    └── after_agent_callback (optional)
    ↓
Event Yielding
    ↓
[Optional] Agent Transfer
    ↓
Response Delivery to User
```

### 1.2 Request/Response Processing Stages

```
LLM REQUEST PIPELINE (9 stages)
├── 1. Basic Processor
│   └── Sets model, config, response_modalities
├── 2. Auth Preprocessor
│   └── Adds authentication handlers
├── 3. Request Confirmation Processor
│   └── Adds tool confirmation logic
├── 4. Instructions Processor
│   ├── Processes agent.instruction
│   ├── Processes global_instruction
│   └── Handles static_instruction
├── 5. Identity Processor
│   └── Sets agent identity/role
├── 6. Contents Processor
│   ├── Builds conversation history
│   └── Manages function call/response pairs
├── 7. Context Cache Processor
│   └── Sets up cache configuration
├── 8. NL Planning Processor
│   ├── Adds planning instructions (Plan-ReAct)
│   └── Marks thought content
└── 9. Code Execution Processor
    ├── Prepares code execution environment
    └── Optimizes data files
    
LLM RESPONSE PIPELINE (2 stages)
├── 1. NL Planning Response Processor
│   └── Processes planned response parts
└── 2. Code Execution Response Processor
    └── Executes code from response
```

---

## 2. FIVE CORE PIPELINE TYPES

### 2.1 SINGLE AGENT PIPELINE (SingleFlow)

**When Used:** Standard agent with tools but no sub-agents

**File:** `/src/google/adk/flows/llm_flows/single_flow.py`

**Flow Diagram:**
```
User Query
    ↓
[Request Processors] → LLM Request Built
    ↓
LLM.generate_content() → Model Response
    ↓
[Response Processors] → Process output
    ↓
[Function Call Handler?]
    ├─ YES → Execute tools → Tool Response
    │          ↓
    │       [LLM Loop] → Call LLM with tool results
    │          ↓
    │       [Next iteration check]
    │
    └─ NO → Return final response
```

**Key Components:**
```python
class SingleFlow(BaseLlmFlow):
    request_processors = [
        basic.request_processor,
        auth_preprocessor.request_processor,
        request_confirmation.request_processor,
        instructions.request_processor,
        identity.request_processor,
        contents.request_processor,
        context_cache_processor.request_processor,
        _nl_planning.request_processor,
        _code_execution.request_processor,
        _output_schema_processor.request_processor,
    ]
    response_processors = [
        _nl_planning.response_processor,
        _code_execution.response_processor,
    ]
```

**Example:**
```python
agent = LlmAgent(
    name="math_solver",
    instruction="You are a math expert",
    tools=[calculator_tool, python_interpreter],
)
# SingleFlow runs automatically with no sub-agents
```

---

### 2.2 MULTI-AGENT DELEGATION PIPELINE (AutoFlow)

**When Used:** Agent that can transfer control to sub-agents dynamically

**File:** `/src/google/adk/flows/llm_flows/auto_flow.py`

**Flow Diagram:**
```
User Query to Agent A
    ↓
[Preprocessing] → LLM Request
    ↓
LLM Response (with transfer_to_agent function call?)
    ├─ YES → Agent Transfer Decision
    │   ├─ Transfer to Sub-agent B
    │   │   ↓
    │   │   [Agent B Execution]
    │   │   ├─ Sub-agent processes query
    │   │   ├─ Returns response
    │   │   ↓
    │   │   [Back to Agent A?]
    │   │   └─ Depends on parent-child relationship
    │   │
    │   └─ Transfer to Peer Agent C
    │       ↓
    │       [Agent C Execution]
    │       ↓
    │       [Response handling]
    │
    └─ NO → Direct response
```

**Key Features:**
- Parent → Sub-agent transfer (hierarchical)
- Sub-agent → Parent transfer (escalation)
- Sub-agent → Peer transfer (lateral delegation)
- Agent tree structure management
- Branch-based isolation of conversation history

**Agent Tree Example:**
```python
root_agent = LlmAgent(name="dispatcher", ...)
customer_service = LlmAgent(name="support", ...)
billing = LlmAgent(name="billing", ...)
root_agent.sub_agents = [customer_service, billing]

# AutoFlow enables transfers between these agents
```

**Code Excerpt:**
```python
class AutoFlow(SingleFlow):
    def __init__(self):
        super().__init__()
        self.request_processors += [agent_transfer.request_processor]
```

---

### 2.3 SEQUENTIAL AGENT PIPELINE (SequentialAgent)

**When Used:** Execute agents in a fixed sequence

**File:** `/src/google/adk/agents/sequential_agent.py`

**Flow Diagram:**
```
[Agent 1]
    ↓
[Agent 2]
    ↓
[Agent 3]
    ↓
[Completion]
```

**Key Characteristics:**
- Agents run one after another
- Each agent receives the full context
- State tracked via `SequentialAgentState`
- Resumable execution with checkpointing
- Task completion signals for live API

**Example:**
```python
seq_agent = SequentialAgent(
    name="data_pipeline",
    sub_agents=[
        extract_agent,      # Step 1: Extract data
        transform_agent,    # Step 2: Transform
        load_agent,         # Step 3: Load
    ]
)
# Runs: extract → transform → load
```

**State Management:**
```python
class SequentialAgentState(BaseAgentState):
    current_sub_agent: str = ''  # Which agent is running
```

---

### 2.4 PARALLEL AGENT PIPELINE (ParallelAgent)

**When Used:** Execute multiple agents concurrently

**File:** `/src/google/adk/agents/parallel_agent.py`

**Flow Diagram:**
```
              [Agent 1]
              /         \
User Query <─ [Agent 2]  ├─> Merge events → Response
              \         /
              [Agent 3]
```

**Key Features:**
- Async task groups for concurrent execution
- Branch isolation for each sub-agent
- Event queue-based merging
- Backpressure support (wait for event consumption)
- Python 3.9+ compatibility

**Implementation Details:**
```python
async def _merge_agent_run(
    agent_runs: list[AsyncGenerator[Event, None]],
) -> AsyncGenerator[Event, None]:
    """Merges events from parallel agents sequentially."""
    # Uses asyncio.TaskGroup for concurrent execution
    # Each agent processes in parallel
    # Events merged in order to downstream
```

---

### 2.5 LOOP AGENT PIPELINE (LoopAgent)

**When Used:** Iterate through agents until completion or max iterations

**File:** `/src/google/adk/agents/loop_agent.py`

**Flow Diagram:**
```
Iteration 1:
    [Agent A] → [Agent B] → [Agent C]
        ↓
Iteration 2:
    [Agent A] → [Agent B] → [Agent C]
        ↓
Continue until:
    - max_iterations reached, OR
    - Agent escalates
```

**Key Characteristics:**
- Configurable max iterations
- Escalation support for early exit
- State tracking per iteration
- Sub-agent state reset between loops
- Resumable execution

**Example:**
```python
loop_agent = LoopAgent(
    name="retry_handler",
    max_iterations=3,
    sub_agents=[validation_agent, correction_agent]
)
# Runs validation → correction loop up to 3 times
```

---

## 3. INSTRUCTION PROCESSORS IN DETAIL

### 3.1 How Instructions Flow Through the System

**File:** `/src/google/adk/flows/llm_flows/instructions.py`

```
Agent Definition
    ├── instruction (str or callable)
    ├── global_instruction (str - deprecated)
    └── static_instruction (types.ContentUnion)
    ↓
_InstructionsLlmRequestProcessor
    ├── Process dynamic instruction
    │   ├── Resolve if callable (InstructionProvider)
    │   ├── Inject session state variables
    │   └── Handle placeholder substitution ({variable_name})
    ├── Process static instruction
    │   └── Add raw without processing
    └── Process global instruction
        └── Apply to all agents in tree
    ↓
LLM Request
    ├── system_instructions (if no static_instruction)
    └── user_content (if static_instruction present)
```

### 3.2 Instruction Processing Logic

**Case 1: No static_instruction**
```
instruction → system_instructions in LLM request
```

**Case 2: With static_instruction**
```
static_instruction → system_instructions
instruction → appended to user content (after static)
```

This enables **context caching optimization** where static content remains unchanged across invocations.

### 3.3 Instruction Provider Pattern

```python
InstructionProvider = Callable[
    [ReadonlyContext], 
    Union[str, Awaitable[str]]
]

# Example: Dynamic instruction that changes per context
async def get_instruction(ctx: ReadonlyContext) -> str:
    user_name = ctx.session.state.get('user_name', 'Guest')
    return f"You are a helpful assistant for {user_name}"

agent = LlmAgent(
    instruction=get_instruction,
    ...
)
```

---

## 4. PLANNER TYPES AND PLANNING PIPELINES

### 4.1 Base Planner Interface

**File:** `/src/google/adk/planners/base_planner.py`

```python
class BasePlanner(ABC):
    def build_planning_instruction(
        self,
        readonly_context: ReadonlyContext,
        llm_request: LlmRequest,
    ) -> Optional[str]:
        """Builds system instruction for planning phase."""

    def process_planning_response(
        self,
        callback_context: CallbackContext,
        response_parts: List[types.Part],
    ) -> Optional[List[types.Part]]:
        """Processes and filters response parts after planning."""
```

### 4.2 Plan-ReAct Planner

**File:** `/src/google/adk/planners/plan_re_act_planner.py`

**When Used:** Force structured reasoning without model's native thinking features

**Flow:**
```
Planning Phase:
    1. Agent creates a PLAN (tagged with /*PLANNING*/)
    2. Agent executes ACTIONS (tagged with /*ACTION*/)
    3. Agent reasons about results (tagged with /*REASONING*/)
    4. Agent revises plan if needed (/*REPLANNING*/)
    5. Agent provides FINAL_ANSWER

Instruction Injection:
    LLM receives tags and format requirements
    ↓
Response Processing:
    - Splits response by tags
    - Marks thoughts (planning/reasoning)
    - Returns actionable parts
    - Stops at first function call
```

**Example Instruction:**
```
Follow this process:
(1) come up with a plan in natural language text
(2) use tools to execute the plan with reasoning interleaved
(3) return final answer

Format:
- Planning: /*PLANNING*/...
- Actions: /*ACTION*/...
- Reasoning: /*REASONING*/...
- Final: /*FINAL_ANSWER*/...
```

### 4.3 Built-In Planner

**File:** `/src/google/adk/planners/built_in_planner.py`

**When Used:** Use model's native thinking features (like extended thinking)

**Flow:**
```
thinking_config applied to GenerateContentConfig
    ↓
LLM processes with native thinking
    ↓
Thought content automatically created
    ↓
No post-processing needed
```

**Usage:**
```python
from google.genai import types
from google.adk.planners import BuiltInPlanner

agent = LlmAgent(
    name="thinker",
    planner=BuiltInPlanner(
        thinking_config=types.ThinkingConfig(
            budget_tokens=5000  # Thinking budget
        )
    ),
    ...
)
```

### 4.4 NL Planning Pipeline

**File:** `/src/google/adk/flows/llm_flows/_nl_planning.py`

**Request Stage:**
```python
class _NlPlanningRequestProcessor:
    async def run_async(self, ...):
        planner = _get_planner(invocation_context)
        
        if isinstance(planner, BuiltInPlanner):
            planner.apply_thinking_config(llm_request)
        elif isinstance(planner, PlanReActPlanner):
            planning_instruction = planner.build_planning_instruction(...)
            llm_request.append_instructions([planning_instruction])
            _remove_thought_from_request(llm_request)
```

**Response Stage:**
```python
class _NlPlanningResponse:
    async def run_async(self, ...):
        processed_parts = planner.process_planning_response(
            callback_context, 
            llm_response.content.parts
        )
        llm_response.content.parts = processed_parts
```

---

## 5. TOOL EXECUTION PIPELINE

### 5.1 Function Calling Flow

**File:** `/src/google/adk/flows/llm_flows/functions.py`

```
LLM Response with function_call
    ↓
[Tool Validation & Preprocessing]
    ├── Check function exists in tools_dict
    ├── Validate arguments against schema
    ├── Run before_tool_callback
    └── Check for required auth
    ↓
[Tool Execution]
    ├── Parallel execution if multiple calls
    ├── Sequential execution if dependencies
    ├── Handle streaming tools
    └── Handle long-running tools
    ↓
[Tool Response Handling]
    ├── Run after_tool_callback
    ├── Capture output
    ├── Handle errors (on_tool_error_callback)
    └── Create FunctionResponse events
    ↓
[Next Step Determination]
    ├─ More function calls? → Call again
    ├─ Auth needed? → Auth flow
    ├─ Confirmation needed? → Confirmation flow
    ├─ Output schema? → Structured response
    └─ Final response? → Return to user
```

### 5.2 Function Call Tracking

```python
def populate_client_function_call_id(model_response_event: Event):
    """Generate ADK internal IDs for tracking."""
    # Prefix: 'adk-' + UUID
    # Used for matching function calls to responses
    # Stripped before sending back to model

def get_long_running_function_calls(
    function_calls: list[types.FunctionCall],
    tools_dict: dict[str, BaseTool],
) -> set[str]:
    """Identify async/long-running tools."""
    # These may take time and trigger pausing
```

### 5.3 Special Function Calls

```python
REQUEST_EUC_FUNCTION_CALL_NAME = 'adk_request_credential'
REQUEST_CONFIRMATION_FUNCTION_CALL_NAME = 'adk_request_confirmation'

# Generated by framework when:
# 1. Tool requires authentication
# 2. Tool needs user confirmation before execution
```

---

## 6. CONTENT BUILDING AND HISTORY MANAGEMENT

### 6.1 Contents Processor

**File:** `/src/google/adk/flows/llm_flows/contents.py`

```
Session Events (all agent interactions)
    ↓
[Branch Filtering]
    └── Isolate events for current branch
         (prevents peer agents from seeing history)
    ↓
[Conversation History Building]
    ├── include_contents='default'
    │   └── Full history from branch
    ├── include_contents='none'
    │   └── Only current turn
    └── [Current turn determination]
        └── Last user message + agent responses
    ↓
[Function Call/Response Pairing]
    ├── Sync responses
    │   └── Immediately follow calls
    ├── Async responses
    │   ├── May come later
    │   ├── Merged into single response event
    │   └── Rearranged in history
    └── Latest response special handling
    ↓
LLM Request Contents
```

### 6.2 Branch Isolation

```
Agent Tree:
    root_agent
    ├── agent_a
    └── agent_b
    
Branch format: "root_agent.agent_a"

When agent_b runs:
    - Sees only events from its branch
    - Does not see agent_a's history
    - Each sub-agent has isolated context
    - Enables clean delegation
```

### 6.3 Event History

```python
class InvocationContext:
    session: Session  # Contains all events
    branch: Optional[str]  # Current branch identifier
    
# Methods:
def _get_events(
    current_invocation: bool = False,
    current_branch: bool = False,
) -> list[Event]:
    """Filter events by invocation and branch."""
```

---

## 7. STATE MANAGEMENT ACROSS PIPELINES

### 7.1 InvocationContext Structure

**File:** `/src/google/adk/agents/invocation_context.py`

```python
class InvocationContext(BaseModel):
    # Core identity
    invocation_id: str
    agent: BaseAgent
    branch: Optional[str]
    
    # Execution tracking
    agent_states: dict[str, dict[str, Any]]
    end_of_agents: dict[str, bool]
    end_invocation: bool
    
    # Communication
    live_request_queue: Optional[LiveRequestQueue]
    active_streaming_tools: Optional[dict[str, ActiveStreamingTool]]
    
    # Audio/Real-time
    input_realtime_cache: Optional[list[RealtimeCacheEntry]]
    output_realtime_cache: Optional[list[RealtimeCacheEntry]]
    
    # Configuration
    run_config: Optional[RunConfig]
    resumability_config: Optional[ResumabilityConfig]
```

### 7.2 Agent State Management

```python
# Example: SequentialAgentState
class SequentialAgentState(BaseAgentState):
    current_sub_agent: str = ''  # Which sub-agent is running

# Saving state
ctx.set_agent_state(agent.name, agent_state=my_state)

# Loading state
agent_state = agent._load_agent_state(ctx, StateType)

# Usage in resumable flows
if agent_state:
    # Resume from where we left off
    start_index = find_agent_index(agent_state.current_sub_agent)
else:
    # First run
    start_index = 0
```

### 7.3 Session State for Variables

```python
# Accessing session state
user_name = ctx.session.state.get('user_name', 'Guest')

# In instruction provider
async def instruction_provider(ctx: ReadonlyContext) -> str:
    user_name = ctx.session.state.get('user_name')
    return f"Greet {user_name} warmly"
```

---

## 8. CALLBACK SYSTEM

### 8.1 Four Callback Types

**Agent Callbacks:**
```python
before_agent_callback: Optional[BeforeAgentCallback]
# Called before agent execution
# Can override agent run or modify state

after_agent_callback: Optional[AfterAgentCallback]
# Called after agent execution
# Can add additional content

def before_agent(callback_context: CallbackContext) -> Optional[types.Content]:
    # Return content to skip agent and use this instead
    pass
```

**Model Callbacks:**
```python
before_model_callback: Optional[BeforeModelCallback]
# Called before LLM invocation
# Inspect/modify LlmRequest

after_model_callback: Optional[AfterModelCallback]
# Called after LLM returns
# Inspect/modify LlmResponse

on_model_error_callback: Optional[OnModelErrorCallback]
# Handle model errors gracefully
```

**Tool Callbacks:**
```python
before_tool_callback: Optional[BeforeToolCallback]
# Intercept tool calls, validate arguments

after_tool_callback: Optional[AfterToolCallback]
# Process tool results, transform output

on_tool_error_callback: Optional[OnToolErrorCallback]
# Handle tool execution errors
```

### 8.2 Callback Context

```python
class CallbackContext:
    invocation_context: InvocationContext
    session: Session
    state: PersistentState  # Mutable state for callbacks
    _event_actions: EventActions  # Actions to apply
```

---

## 9. EVALUATION PIPELINE

### 9.1 Agent Evaluation Flow

**File:** `/src/google/adk/evaluation/agent_evaluator.py`

```
EvalSet (Test Cases)
    ├── query: str
    ├── reference: str (expected)
    └── expected_tool_use: Optional[str]
    ↓
[For each test case, multiple runs]
    ├── Run 1: Execute agent against query
    │   ├── Agent invocation
    │   ├── Capture trajectory
    │   └── Record events
    ├── Run 2: Execute agent again
    │   └── Repeated execution
    └── Average metrics
    ↓
[Metric Evaluation]
    ├── Tool Trajectory Score
    │   └── Evaluates tool use accuracy
    ├── Response Evaluation Score
    │   └── LLM-as-judge evaluation
    ├── Response Match Score
    │   └── String/semantic matching
    └── Safety Evaluation
        └── Safety guideline compliance
    ↓
[Criteria Comparison]
    ├── Actual score vs threshold
    └── Pass/Fail determination
    ↓
Results Report
```

### 9.2 Evaluation Metrics

```python
class PrebuiltMetrics(Enum):
    TOOL_TRAJECTORY_AVG_SCORE = "tool_trajectory_score"
    RESPONSE_EVALUATION_SCORE = "response_evaluation_score"
    RESPONSE_MATCH_SCORE = "response_match_score"
    SAFETY_V1 = "safety_score"
```

### 9.3 Evaluation Configuration

```python
@dataclass
class EvalConfig:
    criteria: dict[str, BaseCriterion]  # Metric thresholds
    
class BaseCriterion:
    threshold: float  # Minimum acceptable score
```

---

## 10. LIVE API PIPELINE (Real-time Audio/Video)

### 10.1 Live Connection Flow

```
LiveRequestQueue (bidirectional communication)
    ↑                        ↓
User Audio Input    LLM Audio Output
    ↓                        ↑
[Send to Model]        [Receive from Model]
    ├── send_realtime()     ├── Audio chunks
    ├── send_content()      ├── Transcriptions
    └── Blobs               ├── Function calls
                            └── Events
                            
LLM Connection Management:
    ├── Session Resumption (reconnect with handle)
    ├── Activity Start/End markers
    ├── Control Events (turn_complete, interrupted)
    └── Error Recovery
```

### 10.2 Live Event Handling

```python
async def _send_to_model(self, llm_connection):
    while True:
        live_request = await live_request_queue.get()
        
        if live_request.blob:
            await llm_connection.send_realtime(live_request.blob)
        if live_request.content:
            await llm_connection.send_content(live_request.content)
        if live_request.close:
            await llm_connection.close()

async def _receive_from_model(self, llm_connection):
    async for llm_response in llm_connection.receive():
        if llm_response.live_session_resumption_update:
            # Save resumption handle
            ctx.live_session_resumption_handle = handle
        
        # Process audio/events
        yield event
```

### 10.3 Audio Caching

```python
class AudioCacheManager:
    def cache_audio(
        invocation_context,
        blob: types.Blob,
        cache_type: str  # 'input' or 'output'
    ):
        # Store audio chunks for later flushing

    async def flush_caches(
        invocation_context,
        flush_user_audio: bool,
        flush_model_audio: bool,
    ) -> list[Event]:
        # Write to artifact service when ready
```

---

## 11. DATA FLOW EXAMPLES

### Example 1: Simple Q&A with Tools

```
User: "What is the weather in NYC?"
    ↓
Agent: weather_assistant
    ├── instruction: "You are a weather expert"
    ├── tools: [weather_api]
    ↓
[Request Processing]
    ├── Contents: [User message]
    ├── Instructions: [System instruction]
    ├── Tools: [weather_api function]
    ↓
LLM Call
    ↓
Response: "I'll check the weather for you..."
Function Call: weather_api(location="NYC")
    ↓
[Tool Execution]
    ├── weather_api returns: {temp: 72°F, condition: sunny}
    ↓
[Contents Updated with Tool Response]
    ├── User message
    ├── Agent function call
    ├── Tool result
    ↓
LLM Call (2nd turn)
    ↓
Response: "The weather in NYC is sunny, 72°F"
No function calls
    ↓
Final Response to User
```

### Example 2: Multi-Agent Delegation

```
User: "I need to check my order status and billing"
    ↓
Root Agent: customer_service_dispatcher
    ├── Can transfer to: [orders_agent, billing_agent]
    ↓
[Request Processing]
    ├── AutoFlow with agent_transfer processor
    ↓
LLM Response:
    "Transfer to orders_agent for order status"
Function Call: transfer_to_agent(agent_name="orders_agent")
    ↓
[Agent Transfer]
    ├── New context created
    ├── Branch: "dispatcher.orders_agent"
    ├── orders_agent executes with same user query
    ↓
orders_agent Response:
    "Your order #12345 is shipped"
    ↓
[Back to dispatcher?]
    └── User can continue with billing question
    ├── Dispatcher may suggest billing_agent
    ├── Or user can ask original dispatcher
    ↓
[Cycle continues based on user needs]
```

### Example 3: Plan-ReAct with Tools

```
User: "Analyze this CSV data and generate a report"
    ↓
Agent: data_analyst
    ├── planner: PlanReActPlanner()
    ├── tools: [csv_loader, python_interpreter]
    ├── code_executor: BuiltInCodeExecutor()
    ↓
[Request Processing]
    ├── Instructions include planning tags
    ├── Code execution prep
    ↓
LLM Call
    ↓
Response:
    /*PLANNING*/
    1. Load the CSV file
    2. Analyze columns and statistics
    3. Generate visualizations
    4. Create summary report
    /*ACTION*/
    csv.load(file)
    [code block execution]
    /*REASONING*/
    The data shows 10,000 records with 5 columns...
    /*ACTION*/
    [visualization code]
    /*FINAL_ANSWER*/
    Here is the comprehensive report...
    ↓
[Response Processing]
    ├── Extracts planning/reasoning parts
    ├── Marks as thoughts
    ├── Executes code blocks
    ├── Captures results
    ↓
Final Report to User
```

---

## 12. IMPLEMENTATION ARCHITECTURE

### 12.1 Request/Response Processor Chain

```python
class BaseLlmRequestProcessor(ABC):
    async def run_async(
        self,
        invocation_context: InvocationContext,
        llm_request: LlmRequest
    ) -> AsyncGenerator[Event, None]:
        """Process and potentially yield events."""

class BaseLlmResponseProcessor(ABC):
    async def run_async(
        self,
        invocation_context: InvocationContext,
        llm_response: LlmResponse
    ) -> AsyncGenerator[Event, None]:
        """Process response and potentially yield events."""
```

### 12.2 Flow Composition

```python
class SingleFlow(BaseLlmFlow):
    def __init__(self):
        super().__init__()
        self.request_processors = [...]  # 9 processors
        self.response_processors = [...]  # 2 processors
    
    async def run_async(self, invocation_context):
        while True:
            # One loop iteration = one LLM turn
            async for event in self._run_one_step_async(ctx):
                yield event
            
            # Check if done
            if last_event.is_final_response():
                break

class AutoFlow(SingleFlow):
    def __init__(self):
        super().__init__()
        self.request_processors += [agent_transfer.request_processor]
        # Adds agent transfer capability on top of SingleFlow
```

---

## 13. PROMPT INJECTION POINTS

### Where Prompts Enter the System

1. **User Message** → InvocationContext.user_content
2. **Agent Instruction** → _InstructionsLlmRequestProcessor
3. **Global Instruction** → _InstructionsLlmRequestProcessor
4. **Planning Instruction** → _NlPlanningRequestProcessor
5. **Tool Descriptions** → Each tool's schema
6. **System Instructions** → SetContentConfig.system_instructions
7. **Context Cache** → Static content for caching
8. **Callback Injections** → before_model_callback returns content

### Where Responses Are Processed

1. **Initial Response** → LlmResponse from model
2. **NL Planning Processing** → Extract plan/reasoning/action
3. **Code Execution** → Execute code blocks
4. **Output Schema** → Validate structured output
5. **Function Call Handling** → Extract and execute tools
6. **Tool Result Integration** → Add to request for next turn

---

## 14. KEY DESIGN PATTERNS

### 14.1 Async Generator Pattern

```python
async def _run_one_step_async(self, ctx):
    """Yielding events allows streaming responses."""
    
    # Preprocess
    async for event in self._preprocess_async(ctx):
        yield event  # Immediate feedback
    
    # Call LLM
    async for llm_response in self._call_llm_async(ctx):
        # Postprocess
        async for event in self._postprocess_async(...):
            yield event  # Stream results as ready
```

### 14.2 Context Propagation

- Single InvocationContext flows through entire pipeline
- All processors receive same context
- Modifications accumulate through stages
- Sub-agent contexts derived from parent

### 14.3 State Serialization

```python
# AgentState saved and restored for resumability
agent_state = SequentialAgentState(
    current_sub_agent="agent_2"
)
ctx.set_agent_state("seq_agent", agent_state)

# On resume
agent_state = agent._load_agent_state(ctx, SequentialAgentState)
if agent_state.current_sub_agent == "agent_2":
    # Resume from agent 2
```

---

## 15. PERFORMANCE CONSIDERATIONS

### Context Caching
- **static_instruction**: Never changes, cacheable
- **instruction**: Dynamic, not cacheable
- **contents**: Conversation history, implicitly cached by model

### Parallel Processing
- **ParallelAgent**: Tasks run concurrently
- **Tool Calls**: Multiple tools can execute in parallel
- **Sub-agent Isolation**: Branches run independently

### Cost Management
```python
run_config.max_llm_calls = 10  # Limit total calls
# Tracks via _InvocationCostManager
```

---

## CONCLUSION

The ADK Python system implements a sophisticated multi-layered pipeline for agent orchestration:

1. **Modular Processors**: Each stage handles specific concerns (instructions, contents, tools, planning)
2. **Flexible Agent Types**: Single, sequential, parallel, looping, and delegating agents
3. **Extensible Callbacks**: Hooks at agent, model, and tool levels
4. **State Management**: Resumable execution with agent state tracking
5. **Real-time Support**: Live API with audio caching and session resumption
6. **Planning Integration**: Native thinking and Plan-ReAct patterns
7. **Evaluation Framework**: Comprehensive testing and benchmarking

The entire system is built on async/await patterns for responsiveness and uses event streaming for real-time feedback.

