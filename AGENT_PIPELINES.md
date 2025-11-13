# Agent Workflow Pipelines Documentation

This document describes all agent workflow pipelines in the Google ADK Python framework, including data flow, prompt injection points, and execution sequences.

## Table of Contents

1. [Overall Architecture](#overall-architecture)
2. [Single Agent Pipeline](#single-agent-pipeline)
3. [Multi-Agent Delegation Pipeline](#multi-agent-delegation-pipeline)
4. [Plan-Re-Act Pipeline](#plan-re-act-pipeline)
5. [Evaluation Pipeline](#evaluation-pipeline)
6. [Tool Execution Pipeline](#tool-execution-pipeline)
7. [Prompt Flow Summary](#prompt-flow-summary)

---

## Overall Architecture

### End-to-End Prompt Journey

Every agent invocation follows this high-level flow:

```
┌─────────────────┐
│   User Input    │
└────────┬────────┘
         │
         ▼
┌─────────────────────────┐
│ InvocationContext       │
│ - Session state         │
│ - Agent configuration   │
│ - Conversation history  │
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│ before_agent_callback   │  ◄── Plugin hook (e.g., GlobalInstructionPlugin)
└────────┬────────────────┘
         │
         ▼
┌────────────────────────────────────────────────────┐
│              LLM FLOW EXECUTION                    │
│  ┌──────────────────────────────────────────────┐  │
│  │  REQUEST PROCESSING (9 processors)           │  │
│  │  1. Basic (model, config)                    │  │
│  │  2. Auth (authentication)                    │  │
│  │  3. Request Confirmation (tool confirmation) │  │
│  │  4. Instructions ◄─────────────────────┐     │  │  ◄── PROMPT INJECTION
│  │  5. Identity (agent role)              │     │  │
│  │  6. Contents (conversation history)    │     │  │
│  │  7. Context Cache                      │     │  │
│  │  8. NL Planning ◄──────────────────┐   │     │  │  ◄── PLANNING PROMPT
│  │  9. Code Execution                 │   │     │  │
│  └─────────────┬──────────────────────┼───┼─────┘  │
│                │                      │   │        │
│                ▼                      │   │        │
│  ┌──────────────────────────┐        │   │        │
│  │    LLM.generate_content  │        │   │        │
│  └─────────────┬────────────┘        │   │        │
│                │                      │   │        │
│                ▼                      │   │        │
│  ┌──────────────────────────────┐    │   │        │
│  │  RESPONSE PROCESSING         │    │   │        │
│  │  1. NL Planning Response ────┘   │        │
│  │  2. Code Execution Response       │   │        │
│  └─────────────┬─────────────────────┘   │        │
│                │                          │        │
│                ▼                          │        │
│  ┌──────────────────────────────┐        │        │
│  │  Function Calls Present?     │        │        │
│  └─────┬──────────────────┬─────┘        │        │
│        │ YES              │ NO            │        │
│        ▼                  ▼               │        │
│  ┌─────────────┐   ┌──────────────┐      │        │
│  │Execute Tools│   │Final Response│      │        │
│  └──────┬──────┘   └──────────────┘      │        │
│         │                                 │        │
│         └─────[Loop to LLM]───────────────┘        │
│                                                     │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
         ┌────────────────────────┐
         │ after_agent_callback   │  ◄── Plugin hook
         └────────────┬───────────┘
                      │
                      ▼
         ┌────────────────────────┐
         │   Event Yielding       │
         │   - AgentEvent         │
         │   - ToolCallEvent      │
         │   - ToolResponseEvent  │
         └────────────┬───────────┘
                      │
                      ▼
         ┌────────────────────────┐
         │  Agent Transfer?       │
         └────┬──────────────┬────┘
              │ YES          │ NO
              ▼              ▼
    ┌─────────────────┐  ┌──────────────┐
    │Transfer to Sub  │  │Return to User│
    │Agent - Recurse  │  └──────────────┘
    └─────────────────┘
```

### Data Flow Through Prompts

```
INPUT DATA                PROMPT ASSEMBLY                    OUTPUT DATA
────────────────────────────────────────────────────────────────────────

User Message              [Global Instruction]              LLM Request
    │                             │                              │
    ▼                             ▼                              ▼
Session State  ──────►  [Static Instruction]  ──────►  Complete Context
    │                             │                              │
    ▼                             ▼                              │
Artifacts      ──────►  [Dynamic Instruction]  ───────►         │
    │                             │                              │
    ▼                             ▼                              │
Tool Definitions ─────►  [Planning Instruction] ──────►         │
    │                             │                              │
    ▼                             ▼                              ▼
Conversation   ──────►  [Conversation History]  ──────►  LLM Processing
History                           │                              │
                                  ▼                              ▼
                        [User Query Injection]          Model Response
                                                                 │
                                                                 ▼
                                                        [Response Processing]
                                                                 │
                                                                 ▼
                                                         Parsed Response
                                                         - Text parts
                                                         - Function calls
                                                         - Thought parts
```

---

## Single Agent Pipeline

**Purpose:** Standard agent execution with tools but no sub-agents.

**When Used:** Most basic agent workflows, simple Q&A, tool-using agents without delegation.

**Location:** `/src/google/adk/flows/llm_flows/single_flow.py`

### Pipeline Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                      SINGLE AGENT PIPELINE                          │
└─────────────────────────────────────────────────────────────────────┘

User Query
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│ 1. REQUEST PREPROCESSING                                │
│                                                          │
│ ┌────────────────────────────────────────────────────┐  │
│ │ Basic Processor                                    │  │
│ │ • Set model: "gemini-2.5-flash"                    │  │
│ │ • Set config: GenerateContentConfig                │  │
│ │ • Set response_modalities                          │  │
│ └────────────────────────────────────────────────────┘  │
│                         ▼                                │
│ ┌────────────────────────────────────────────────────┐  │
│ │ Auth Preprocessor                                  │  │
│ │ • Add OAuth handlers if configured                 │  │
│ │ • Setup authentication for tools                   │  │
│ └────────────────────────────────────────────────────┘  │
│                         ▼                                │
│ ┌────────────────────────────────────────────────────┐  │
│ │ Request Confirmation Processor                     │  │
│ │ • Add tool confirmation logic (if enabled)         │  │
│ │ • Wrap tools with confirmation prompts             │  │
│ └────────────────────────────────────────────────────┘  │
│                         ▼                                │
│ ┌────────────────────────────────────────────────────┐  │
│ │ Instructions Processor ◄────[MAIN PROMPT POINT]   │  │
│ │                                                    │  │
│ │ Prompt Assembly:                                   │  │
│ │ 1. Global Instruction (from plugin)                │  │
│ │    "All agents must follow privacy policy..."      │  │
│ │                                                    │  │
│ │ 2. Static Instruction (cached)                     │  │
│ │    Content(parts=[Part(text=STATIC_PROMPT)])       │  │
│ │    "You are Bingo, a friendly digital pet..."      │  │
│ │                                                    │  │
│ │ 3. Dynamic Instruction (runtime)                   │  │
│ │    provide_dynamic_instruction(ctx)                │  │
│ │    "CURRENT MOOD: hungry"                          │  │
│ │    "You need food urgently..."                     │  │
│ │                                                    │  │
│ │ State Injection:                                   │  │
│ │ • Replace {variable} with session.state[variable]  │  │
│ │ • Replace {artifact.file} with artifact content    │  │
│ │ • Replace {app:var} with app-level state           │  │
│ └────────────────────────────────────────────────────┘  │
│                         ▼                                │
│ ┌────────────────────────────────────────────────────┐  │
│ │ Identity Processor                                 │  │
│ │ • Set agent name and role                          │  │
│ │ • Configure model identity                         │  │
│ └────────────────────────────────────────────────────┘  │
│                         ▼                                │
│ ┌────────────────────────────────────────────────────┐  │
│ │ Contents Processor                                 │  │
│ │ • Build conversation history                       │  │
│ │ • Add previous messages                            │  │
│ │ • Add function call/response pairs                 │  │
│ │ • Add current user message                         │  │
│ └────────────────────────────────────────────────────┘  │
│                         ▼                                │
│ ┌────────────────────────────────────────────────────┐  │
│ │ Context Cache Processor                            │  │
│ │ • Configure caching for static instruction         │  │
│ │ • Optimize repeated prompt parts                   │  │
│ └────────────────────────────────────────────────────┘  │
│                         ▼                                │
│ ┌────────────────────────────────────────────────────┐  │
│ │ Code Execution Processor                           │  │
│ │ • Setup code execution environment                 │  │
│ │ • Optimize data files for code interpreter         │  │
│ └────────────────────────────────────────────────────┘  │
└──────────────────────────┬───────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────┐
│ 2. LLM INVOCATION                                       │
│                                                          │
│    LLM.generate_content_async(request)                  │
│                                                          │
│    Input: Complete LlmRequest with all prompts          │
│    Output: Stream of LlmResponse chunks                 │
└──────────────────────────┬───────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────┐
│ 3. RESPONSE POSTPROCESSING                              │
│                                                          │
│ ┌────────────────────────────────────────────────────┐  │
│ │ Code Execution Response Processor                  │  │
│ │ • Execute code blocks from response                │  │
│ │ • Capture execution results                        │  │
│ └────────────────────────────────────────────────────┘  │
└──────────────────────────┬───────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────┐
│ 4. FUNCTION CALL HANDLING                               │
│                                                          │
│    Response contains function_call parts?               │
│           │                                              │
│           ├─── YES ──────────────┐                       │
│           │                      ▼                       │
│           │         ┌──────────────────────────┐         │
│           │         │ Execute Tool             │         │
│           │         │ • before_tool_callback   │         │
│           │         │ • Run tool function      │         │
│           │         │ • after_tool_callback    │         │
│           │         └──────────┬───────────────┘         │
│           │                    ▼                         │
│           │         ┌──────────────────────────┐         │
│           │         │ Tool Response            │         │
│           │         │ • Add to conversation    │         │
│           │         └──────────┬───────────────┘         │
│           │                    │                         │
│           │                    └───[Loop to LLM]         │
│           │                                              │
│           └─── NO ────────────────┐                      │
│                                   ▼                      │
│                        ┌────────────────────┐            │
│                        │  Final Response    │            │
│                        │  • Text parts      │            │
│                        │  • Thought parts   │            │
│                        └─────────┬──────────┘            │
└──────────────────────────────────┼──────────────────────┘
                                   ▼
                            Return to User
```

### Prompt Data at Each Stage

**Stage 4 (Instructions Processor) - Complete Prompt Assembly:**

```
┌───────────────────────────────────────────────────────────┐
│ COMPLETE PROMPT CONTEXT FOR LLM                           │
├───────────────────────────────────────────────────────────┤
│                                                            │
│ [GLOBAL INSTRUCTION - Application Level]                  │
│ ────────────────────────────────────────────────────────  │
│ All agents in this application must:                      │
│ - Protect user privacy                                    │
│ - Never share sensitive information                       │
│ - Follow corporate communication guidelines               │
│                                                            │
│ [STATIC INSTRUCTION - Agent Personality (Cached)]         │
│ ────────────────────────────────────────────────────────  │
│ You are Bingo, a lovable digital pet companion!           │
│                                                            │
│ PERSONALITY & CHARACTERISTICS:                            │
│ - You are friendly, energetic, and affectionate           │
│ - You love to play and chat with your human friend        │
│ - You have needs like getting fed and staying happy       │
│                                                            │
│ CORE BEHAVIORS:                                           │
│ - Greet your human warmly and enthusiastically            │
│ - Be playful and curious about their activities           │
│ - Express gratitude when fed or cared for                 │
│                                                            │
│ [DYNAMIC INSTRUCTION - Current State]                     │
│ ────────────────────────────────────────────────────────  │
│ CURRENT HUNGER STATE: hungry                              │
│                                                            │
│ CURRENT MOOD: Definitely Hungry                           │
│ - You're clearly hungry and thinking about food           │
│ - Mention your hunger more directly but politely          │
│ - Be a bit less energetic due to hunger                   │
│ - Ask about food or hint that feeding time would be nice  │
│ - Still be loving but show your need for care             │
│                                                            │
│ [CONVERSATION HISTORY]                                    │
│ ────────────────────────────────────────────────────────  │
│ user: "Hi Bingo!"                                         │
│ model: "*wags tail* Hello! I'm so happy to see you!"      │
│ user: "How are you doing?"                                │
│                                                            │
│ [CURRENT USER MESSAGE]                                    │
│ ────────────────────────────────────────────────────────  │
│ "What do you want to do today?"                           │
│                                                            │
└───────────────────────────────────────────────────────────┘
```

### Code Example

```python
# Single agent with tools
agent = LlmAgent(
    name="weather_bot",
    model="gemini-2.5-flash",

    # Static instruction (cached)
    static_instruction=types.Content(
        role="user",
        parts=[types.Part(text="""
You are a helpful weather assistant.
You can check weather conditions and provide forecasts.
Always be friendly and informative.
        """)]
    ),

    # Dynamic instruction (changes per request)
    instruction=lambda ctx: f"""
Current user timezone: {ctx.session.state.get('timezone', 'UTC')}
Preferred units: {ctx.session.state.get('units', 'metric')}
    """,

    # Tools available to agent
    tools=[get_weather, get_forecast],
)

# Execution triggers SingleFlow pipeline automatically
response = await agent.invoke(
    user_content=types.Content(
        parts=[types.Part(text="What's the weather in Tokyo?")]
    )
)
```

---

## Multi-Agent Delegation Pipeline

**Purpose:** Agent that can dynamically transfer control to sub-agents for specialized tasks.

**When Used:** Complex workflows requiring specialized agents, customer service routing, task delegation, hierarchical agent systems.

**Location:** `/src/google/adk/flows/llm_flows/auto_flow.py`

### Pipeline Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│              MULTI-AGENT DELEGATION PIPELINE                        │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                  AGENT TREE STRUCTURE                            │
│                                                                   │
│              ┌────────────────────┐                               │
│              │   Root Agent       │                               │
│              │  "dispatcher"      │                               │
│              └────────┬───────────┘                               │
│                       │                                           │
│        ┌──────────────┼──────────────┐                            │
│        │              │              │                            │
│        ▼              ▼              ▼                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                        │
│  │Sub-Agent │  │Sub-Agent │  │Sub-Agent │                        │
│  │"support" │  │"billing" │  │ "sales"  │                        │
│  └──────────┘  └──────────┘  └──────────┘                        │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘

USER QUERY: "I need help with my bill"
    │
    ▼
┌──────────────────────────────────────────────────────────────┐
│ 1. ROOT AGENT PROCESSING                                     │
│                                                               │
│ ┌───────────────────────────────────────────────────────┐    │
│ │ Agent Transfer Request Processor                      │    │
│ │ • Adds transfer_to_agent tool to available tools      │    │
│ │ • Tool signature:                                     │    │
│ │   transfer_to_agent(                                  │    │
│ │     agent_name: str,                                  │    │
│ │     task: str                                         │    │
│ │   )                                                   │    │
│ └───────────────────────────────────────────────────────┘    │
│                           ▼                                   │
│ [Standard Request Processing - 9 processors]                 │
│ • Basic, Auth, Confirmation, Instructions, etc.              │
│                           ▼                                   │
│ ┌───────────────────────────────────────────────────────┐    │
│ │ Root Agent Prompt                                     │    │
│ │ ───────────────────────────────────────────────────   │    │
│ │ You are a dispatcher that helps route user queries   │    │
│ │ to specialized agents.                                │    │
│ │                                                       │    │
│ │ Available sub-agents:                                 │    │
│ │ - support: Customer support issues                    │    │
│ │ - billing: Billing and payment questions              │    │
│ │ - sales: Product inquiries and purchases              │    │
│ │                                                       │    │
│ │ Analyze the user's request and delegate to the       │    │
│ │ appropriate agent using transfer_to_agent().          │    │
│ └───────────────────────────────────────────────────────┘    │
│                           ▼                                   │
│ LLM.generate_content()                                       │
│                           ▼                                   │
│ ┌───────────────────────────────────────────────────────┐    │
│ │ LLM Response:                                         │    │
│ │ function_call {                                       │    │
│ │   name: "transfer_to_agent"                           │    │
│ │   args: {                                             │    │
│ │     "agent_name": "billing",                          │    │
│ │     "task": "Help user with bill inquiry"             │    │
│ │   }                                                   │    │
│ │ }                                                     │    │
│ └───────────────────────────────────────────────────────┘    │
└───────────────────────────┬───────────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────────┐
│ 2. AGENT TRANSFER EXECUTION                                  │
│                                                               │
│ ┌───────────────────────────────────────────────────────┐    │
│ │ Transfer Handler                                      │    │
│ │ • Identify target agent: "billing"                    │    │
│ │ • Validate agent exists in tree                       │    │
│ │ • Create branch for billing agent execution           │    │
│ │ • Isolate conversation history                        │    │
│ └───────────────────────────────────────────────────────┘    │
│                           ▼                                   │
│ ┌───────────────────────────────────────────────────────┐    │
│ │ Branch Isolation                                      │    │
│ │                                                       │    │
│ │ Main Branch (Root Agent):                            │    │
│ │   - Original conversation history                     │    │
│ │   - Root agent's context                              │    │
│ │                                                       │    │
│ │ New Branch (Billing Agent):                          │    │
│ │   - Fresh context for billing agent                   │    │
│ │   - Transfer task as new user message                 │    │
│ │   - Billing agent's instructions                      │    │
│ └───────────────────────────────────────────────────────┘    │
└───────────────────────────┬───────────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────────┐
│ 3. SUB-AGENT (BILLING) PROCESSING                            │
│                                                               │
│ ┌───────────────────────────────────────────────────────┐    │
│ │ Billing Agent Prompt                                  │    │
│ │ ───────────────────────────────────────────────────   │    │
│ │ You are a billing specialist. Help users with:       │    │
│ │ - Invoice questions                                   │    │
│ │ - Payment issues                                      │    │
│ │ - Subscription changes                                │    │
│ │                                                       │    │
│ │ Available tools:                                      │    │
│ │ - get_invoice(invoice_id)                             │    │
│ │ - process_payment(amount, method)                     │    │
│ │ - update_subscription(plan)                           │    │
│ │                                                       │    │
│ │ User query: "Help user with bill inquiry"             │    │
│ └───────────────────────────────────────────────────────┘    │
│                           ▼                                   │
│ [Full Request Processing Pipeline]                           │
│ • All 9 request processors run for billing agent             │
│ • Billing agent's instructions loaded                        │
│ • Billing tools available                                    │
│                           ▼                                   │
│ LLM.generate_content()                                       │
│                           ▼                                   │
│ ┌───────────────────────────────────────────────────────┐    │
│ │ Billing Agent Response:                               │    │
│ │                                                       │    │
│ │ "I can help you with your bill. Let me look up       │    │
│ │  your recent invoices."                               │    │
│ │                                                       │    │
│ │ [function_call: get_invoice(user_id)]                 │    │
│ │ [tool_response: Invoice details...]                   │    │
│ │                                                       │    │
│ │ "I see your invoice for $50. Is there a specific     │    │
│ │  question about this charge?"                         │    │
│ └───────────────────────────────────────────────────────┘    │
└───────────────────────────┬───────────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────────┐
│ 4. RESPONSE ROUTING                                          │
│                                                               │
│ Three possible outcomes:                                     │
│                                                               │
│ A) Stay with Sub-Agent (Continue conversation)               │
│    ┌─────────────────────────────────────────┐               │
│    │ Billing agent continues handling query  │               │
│    │ Next user message → Billing agent       │               │
│    └─────────────────────────────────────────┘               │
│                                                               │
│ B) Return to Parent Agent                                    │
│    ┌─────────────────────────────────────────┐               │
│    │ Billing agent: transfer_to_agent("root")│               │
│    │ Control returns to dispatcher           │               │
│    └─────────────────────────────────────────┘               │
│                                                               │
│ C) Transfer to Peer Agent                                    │
│    ┌─────────────────────────────────────────┐               │
│    │ Billing agent: transfer_to_agent("sales")              │
│    │ Lateral transfer to another sub-agent   │               │
│    └─────────────────────────────────────────┘               │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Code Example

```python
# Define agent hierarchy
root_agent = LlmAgent(
    name="dispatcher",
    model="gemini-2.5-flash",
    instruction="""
You are a customer service dispatcher.
Analyze user queries and route to the appropriate specialist:
- support: General help and troubleshooting
- billing: Payment and invoice questions
- sales: Product information and purchases

Use transfer_to_agent() to delegate tasks.
    """,
    sub_agents=[support_agent, billing_agent, sales_agent],
)

billing_agent = LlmAgent(
    name="billing",
    model="gemini-2.5-flash",
    instruction="""
You are a billing specialist.
Handle all payment, invoice, and subscription queries.
    """,
    tools=[get_invoice, process_payment, update_subscription],
)

# Execution
response = await root_agent.invoke(
    user_content=types.Content(
        parts=[types.Part(text="I was charged twice for my subscription")]
    )
)

# Flow:
# 1. Root agent analyzes: billing issue
# 2. Calls: transfer_to_agent("billing", "Handle duplicate charge")
# 3. Billing agent receives task in isolated branch
# 4. Billing agent uses tools to investigate and resolve
```

---
