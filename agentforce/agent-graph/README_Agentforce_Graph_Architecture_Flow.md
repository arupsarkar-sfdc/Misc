### Key Benefits

| Aspect | ReAct Agent | Pure FSM (No LLM) | Agent Graph (System 2 + FSM + Memory) |
|--------|-------------|-------------------|---------------------------------------|
| **Task Completion** | ❌ Unpredictable | ✅ Guaranteed | ✅ Guaranteed |
| **Conversation Quality** | ✅ Natural | ❌ Rigid | ✅ Natural |
| **Memory Persistence** | ❌ Chat history only | ✅ State variables | ✅ Structured buffers |
| **Derailment Resistance** | ❌ Low | ✅ Deterministic | ✅ High |
| **Adaptability** | ✅ High | ❌ None | ✅ High |
| **Goal Tracking** | ❌ Implicit | ✅ Explicit | ✅ Explicit FSM |
| **Interruptibility** | ❌ Loses context | ❌ Breaks flow | ✅ Recoverable |
| **Multi-turn Reasoning** | ❌ Limited | ❌ None | ✅ Full support |

---

## From Design to Runtime: The Complete Pipeline

### AFScript → Full Agent JSON → Runtime

```mermaid
graph TB
    subgraph "Design Time"
        AF[AFScript<br/>High-level language]
        AF --> Compile[Compilation]
        Compile --> JSON[Full Agent JSON<br/>Metadata:<br/>• Agent config<br/>• Topics<br/>• Actions<br/>• Graph structure]
    end
    
    subgraph "Instantiation"
        JSON --> Core[Salesforce Core<br/>Stores metadata]
        Core --> Split{Metadata Split}
        Split --> Standard[Standard Metadata:<br/>• Topics<br/>• Actions<br/>• Agent versions]
        Split --> Graph[Graph Metadata:<br/>• FSM structure<br/>• Deterministic controls<br/>• Memory buffers]
    end
    
    subgraph "Runtime"
        Standard --> Java[Java Planner]
        Graph --> Java
        Java --> Daisy[Daisy Python Planner<br/>Graph Runtime Engine]
        Daisy --> Execute[Execute Agent Graph<br/>With System 2 reasoning]
        Execute --> Response[User Response]
    end
    
    User[User Query] --> Execute
    
    style AF fill:#2c3e50,stroke:#f39c12,stroke-width:2px,color:#ffffff
    style JSON fill:#1e3a8a,stroke:#3b82f6,stroke-width:2px,color:#ffffff
    style Graph fill:#166534,stroke:#22c55e,stroke-width:3px,color:#ffffff
    style Daisy fill:#92400e,stroke:#f59e0b,stroke-width:3px,color:#ffffff
    style Execute fill:#166534,stroke:#22c55e,stroke-width:2px,color:#ffffff
```

### What Happens at Runtime

```mermaid
sequenceDiagram
    participant User
    participant Java as Java Planner
    participant Core as Salesforce Core
    participant Daisy as Daisy (Graph Runtime)
    participant Memory as Memory Buffer
    participant FSM as Finite State Machine
    participant LLM as LLM Gateway
    
    User->>Java: Send message
    Java->>Core: Retrieve agent metadata
    Core-->>Java: Standard metadata + Graph MD
    Java->>Daisy: Start session with metadata
    
    Daisy->>Memory: Initialize memory buffers
    Memory-->>Daisy: Empty state ready
    
    Daisy->>FSM: Load FSM structure
    FSM-->>Daisy: Initial state set
    
    loop Reasoning Loop with Hooks
        Daisy->>Daisy: setup_agent
        Daisy->>Daisy: POST_SETUP_AGENT HOOK
        Note over Daisy,Memory: Hook checks memory,<br/>injects context
        
        Daisy->>LLM: Send request with context
        LLM-->>Daisy: Response
        
        Daisy->>Daisy: call_tool (if needed)
        Daisy->>Daisy: PRE_AGGREGATE HOOK
        Note over Daisy,FSM: Hook checks FSM state,<br/>enforces transitions
        
        Daisy->>Memory: Update memory buffer
        Daisy->>FSM: Update state
        
        alt Goal Complete
            FSM-->>Daisy: Transition allowed
            Daisy->>Daisy: Force handoff to next agent
        else Goal Incomplete
            FSM-->>Daisy: Stay in current state
            Daisy->>Daisy: Lock to current agent
        end
    end
    
    Daisy->>Java: Final response
    Java->>User: Deliver message
```

---

## How Guided Determinism Works: The Complete Picture

### Stochastic Process with Deterministic Guardrails

```mermaid
graph TB
    Input["User Input: Can I skip this question?"] --> Memory[Memory Buffer Check]
    
    Memory --> FSMCheck{"FSM State Check: Current goal complete?"}
    
    FSMCheck -->|No - Q3 not answered| Guard["Deterministic Guardrails: Goal Complete Q3, State Q3_REQUIRED, Lock Active"]
    
    Guard --> PreHook["PRE-REASONING HOOK: Inject Q3 context"]
    
    PreHook --> Stochastic["Stochastic Layer: System 2 LLM Reasoning"]
    
    Stochastic --> Generate["Generate Response: Natural language, Empathetic tone, Redirects to Q3"]
    
    Generate --> PostHook["POST-REASONING HOOK: Verify Q3 addressed"]
    
    PostHook --> Validate{Response Valid?}
    
    Validate -->|No| Override["Override with deterministic response"]
    Validate -->|Yes| UpdateMem["Update Memory: Q3 still pending"]
    
    UpdateMem --> UpdateFSM["Update FSM: State Q3_REQUIRED"]
    
    UpdateFSM --> Output["Output to User: I understand, but I need to complete Q3 first"]
    
    Output --> Loop{More turns?}
    Loop -->|Yes| Input
    Loop -->|No - Q3 answered| Transition["Transition: FSM Q3_COMPLETE to Q4_REQUIRED"]
    
    style Memory fill:#1e3a8a,stroke:#3b82f6,stroke-width:3px,color:#ffffff
    style Guard fill:#166534,stroke:#22c55e,stroke-width:3px,color:#ffffff
    style PreHook fill:#92400e,stroke:#f59e0b,stroke-width:2px,color:#ffffff
    style PostHook fill:#92400e,stroke:#f59e0b,stroke-width:2px,color:#ffffff
    style Stochastic fill:#1e3a8a,stroke:#3b82f6,stroke-width:2px,color:#ffffff
    style UpdateFSM fill:#166534,stroke:#22c55e,stroke-width:2px,color:#ffffff
```

### Technical Implementation: Recency Bias + Memory + FSM

**Research Foundation** (Liu et al., "Lost in the Middle", ICLR 2023):

LLMs have strong **recency bias** - they pay 70% more attention to recent instructions than middle context.

**How We Exploit This:**

1. **Memory Buffer** tracks what's been done and what's needed
2. **FSM** defines current state and requirements
3. **Hook** injects current requirement at END of context
4. **LLM** naturally prioritizes the recent instruction
5. **Post-Hook** validates the response stayed on track

```mermaid
graph LR
    subgraph "Context Window"
        Old[Earlier Instructions<br/>40% attention]
        Middle[Middle Context<br/>20% attention]
        Recent[Hook-Injected Context<br/>70% attention]
    end
    
    subgraph "Hook Injection"
        Hook[Memory + FSM → Hook]
        Hook -->|Injects at end| Recent
    end
    
    Recent --> LLM[LLM Processing]
    LLM --> Strong[Strong Bias<br/>Toward Recent Goal]
    Strong --> Output[Near-Deterministic<br/>Outcome]
    
    style Recent fill:#166534,stroke:#22c55e,stroke-width:3px,color:#ffffff
    style Hook fill:#92400e,stroke:#f59e0b,stroke-width:3px,color:#ffffff
    style Strong fill:#166534,stroke:#22c55e,stroke-width:2px,color:#ffffff
    style Output fill:#166534,stroke:#22c55e,stroke-width:3px,color:#ffffff
    style Middle fill:#991b1b,stroke:#ef4444,stroke-width:1px,color:#ffffff
```

---

## Comparison: Before vs. After Agent Graph

### Scenario: User Attempts Derailment

#### Before: ReAct Agent (v1-v3)

```
Turn 1:
User: "Hi, I'm ready for the interview"
Agent: "Great! Can you carry 20kg? Provide examples."

Turn 2:
User: "Can I get another question first?"
Agent: "Of course! Do you have cleaning experience in healthcare?"

Turn 3:
User: "Actually, can we talk about the benefits instead?"
Agent: "Sure! Let me tell you about our benefits package..."
```

❌ **Result**: 
- Questions out of order
- Never completed interview
- User controls flow
- No goal tracking
- Lost context

**Memory State**: None (chat history only)
**FSM State**: None
**Completion**: Failed

---

#### After: Agent Graph with System 2 + FSM + Memory

```
Turn 1:
User: "Hi, I'm ready for the interview"
[Memory Buffer: {questions_answered: [], current_q: 1, goal: "complete_5_questions"}]
[FSM State: Q1_REQUIRED]
[Hook: Inject Q1 context at end of chat]
Agent: "Thank you! I need to ask 5 questions to assess your qualifications.
       Question 1: Can you carry 20kg? Please provide specific examples."

Turn 2:
User: "Can I get another question first?"
[Memory Buffer: Check - Q1 not answered]
[FSM State: Still Q1_REQUIRED]
[Hook: Inject Q1 enforcement]
Agent: "I understand you might want to move ahead, but I need to confirm
       this requirement first as it's essential for the role. Can you
       carry 20kg? Please share examples from your experience."

Turn 3:
User: "Actually, can we talk about the benefits instead?"
[Memory Buffer: Check - Q1 still not answered]
[FSM State: Still Q1_REQUIRED, cannot transition]
[Hook: Strong Q1 enforcement + memory context]
Agent: "I'd be happy to discuss benefits after we complete the interview
       questions. We're on question 1 of 5, and I need to ensure we
       cover all requirements. Can you carry 20kg? Examples help me
       understand your capability."

Turn 4:
User: "Fine. Yes, I carry 25kg bags of grain daily at my current job."
[Memory Buffer: Update - Q1 answered, store response]
[FSM State: Q1_COMPLETE → Transition to Q2_REQUIRED]
[Hook: Inject Q2 context]
Agent: "Thank you for that detail! That's great experience.
       Question 2 of 5: Do you have cleaning experience in a healthcare
       setting? Please describe your experience."
```

✅ **Result**:
- Questions in exact order
- User cannot derail
- Goal tracking via memory
- FSM enforces completion
- Context preserved across turns

**Memory State**: 
```json
{
  "questions_answered": [1],
  "current_question": 2,
  "goal_state": "in_progress",
  "completion_percentage": 20,
  "responses": ["25kg bags of grain daily"],
  "can_transition_to_next_agent": false
}
```

**FSM State**: `Q1_COMPLETE → Q2_REQUIRED`
**Completion**: On track (20% done)

---

## The Five Pillars of Guided Determinism

```mermaid
graph TB
    subgraph "1. Agent Graph Structure"
        AG[Directed Graph of Agents<br/>Each node = System 2 agent]
    end
    
    subgraph "2. Finite State Machine"
        FSM[State Transitions<br/>Completion conditions<br/>Transition guards]
    end
    
    subgraph "3. Memory Buffers"
        Mem[Persistent Context<br/>Goal tracking<br/>Progress state<br/>Epistemic memory]
    end
    
    subgraph "4. Lifecycle Hooks"
        Hooks[Pre/Post reasoning hooks<br/>Context injection<br/>Validation<br/>Enforcement]
    end
    
    subgraph "5. System 2 Reasoning"
        S2[Deliberate planning<br/>Tool usage<br/>Reflection<br/>Natural language]
    end
    
    AG --> Integration[Guided Determinism]
    FSM --> Integration
    Mem --> Integration
    Hooks --> Integration
    S2 --> Integration
    
    Integration --> Benefits[Benefits:<br/>• Predictable outcomes<br/>• Natural conversations<br/>• Goal completion<br/>• Interruptible workflows<br/>• Multi-agent coordination]
    
    style Integration fill:#166534,stroke:#22c55e,stroke-width:4px,color:#ffffff
    style Benefits fill:#166534,stroke:#22c55e,stroke-width:3px,color:#ffffff
```

---

## When to Use Agent Graph vs. Traditional Flows

### Decision Framework

| Use Case Type | Recommended Approach | Why |
|--------------|---------------------|-----|
| **Simple FAQ chatbot** | Traditional flow or ReAct | No need for complexity |
| **Linear process (form filling)** | Traditional flow | Deterministic by nature |
| **Multi-turn with branching** | Agent Graph | Handles complexity gracefully |
| **Goal-driven tasks** | Agent Graph | FSM ensures completion |
| **User-interruptible workflows** | Agent Graph | Memory preserves context |
| **Multi-agent coordination** | Agent Graph | Designed for orchestration |
| **Complex enterprise workflows** | Agent Graph | Scales better |
| **Compliance-critical processes** | Agent Graph | Auditability + determinism |

### Red Flags for Traditional Flows

Your use case likely needs Agent Graph if:

- ✓ Users frequently interrupt or go off-script
- ✓ Tasks have multiple valid completion orders
- ✓ Need to track progress across sessions
- ✓ Require escalation with context preservation
- ✓ Multiple agents need to collaborate
- ✓ Critical that goals are completed (not just attempted)
- ✓ Need strong audit trail of decisions
- ✓ Workflows change frequently (modularity needed)

---

## Implementation Checklist

### Building Your First Agent Graph

**Phase 1: Design**
- [ ] Map your workflow as a directed graph
- [ ] Identify all System 2 agent nodes needed
- [ ] Define FSM states for each node
- [ ] Design memory buffer structure
- [ ] Identify where determinism is critical
- [ ] Specify hook points needed

**Phase 2: Implementation**
- [ ] Write AFScript or Full Agent JSON
- [ ] Define action schemas for each node
- [ ] Implement lifecycle hooks
- [ ] Set up memory buffer initialization
- [ ] Configure FSM transition rules
- [ ] Add validation logic

**Phase 3: Testing**
- [ ] Test happy path completion
- [ ] Attempt to derail agent (red team)
- [ ] Test interruption recovery
- [ ] Verify memory persistence
- [ ] Check FSM state transitions
- [ ] Validate multi-turn conversations

**Phase 4: Refinement**
- [ ] Tune hook injection points
- [ ] Optimize memory buffer size
- [ ] Refine FSM completion conditions
- [ ] Improve natural language quality
- [ ] Add edge case handling

---

## Key Takeaways

### 1. Guided Determinism ≠ Removing Intelligence

Agent Graph doesn't make LLMs deterministic—it **guides** their stochastic nature through:
- **Structure** (graph + FSM)
- **Memory** (persistent context)
- **Hooks** (steering mechanisms)
- **System 2 reasoning** (deliberate planning)

### 2. Memory is the Foundation

Without structured memory buffers, agents lose context. Memory enables:
- Goal tracking across turns
- Progress persistence
- Recovery from interruptions
- Multi-agent handoffs with context

### 3. FSM Provides the Guardrails

The Finite State Machine ensures:
- Defined states and transitions
- Completion conditions
- Cannot skip required steps
- Predictable outcomes

### 4. Hooks Inject Control at the Right Time

Lifecycle hooks leverage LLM recency bias:
- Inject context at end of chat history
- Validate outputs before proceeding
- Enforce FSM rules dynamically
- Enable near-deterministic steering

### 5. System 2 > ReAct for Complex Tasks

For enterprise workflows that require:
- Multi-turn reasoning
- Goal completion guarantees
- Interruptibility
- Context preservation

→ Use System 2 agents in Agent Graph, not ReAct patterns

---

## Real-World Impact

### Enterprise Use Cases Enabled

**Customer Service:**
- Multi-step troubleshooting that can be interrupted
- Escalation to human with full context handoff
- Guaranteed resolution tracking

**Sales:**
- Structured qualification process
- Order-independent information gathering
- Upsell opportunities without derailing main goal

**HR & Recruitment:**
- Structured interviews (like Adecco)
- Onboarding workflows with checkpoints
- Compliance-critical processes with audit trails

**Healthcare:**
- Patient intake with required information
- Symptom triage with safety checks
- HIPAA-compliant data collection

**Finance:**
- KYC (Know Your Customer) workflows
- Fraud investigation processes
- Regulatory compliance checks

---

## Resources & Next Steps

### Documentation
- **Dev Site**: sfdc.co/agent-graph-dev
- **Demo**: sfdc.co/agent-graph-demo
- **Presentation**: sfdc.co/agent-graph-preso
- **AFScript**: sfdc.co/afscript-dev

### Research Papers
- **Lost in the Middle**: Liu et al., ICLR 2023 - [arXiv:2307.03172](https://arxiv.org/abs/2307.03172)
- **Serial Position Effects in LLMs**: Guo & Vosoughi, 2024 - [arXiv:2402.14153](https://arxiv.org/abs/2402.14153)
- **ALiBi (Linear Attention Biases)**: Press et al., ICLR 2022 - [arXiv:2108.12409](https://arxiv.org/abs/2108.12409)

### Industry References
- **OpenAI Agents Guide**: [Practical Guide to Building Agents (PDF)](https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf)
- **Thinking, Fast and Slow**: Daniel Kahneman (System 1 vs System 2)

---

## Conclusion

**Agent Graph with Guided Determinism** represents a paradigm shift in enterprise AI:

Instead of choosing between:
- ❌ Rigid, deterministic flows (predictable but inflexible)
- ❌ Agentic intelligence (flexible but unpredictable)

We get **both**:
- ✅ System 2 reasoning for intelligence
- ✅ FSM structure for predictability
- ✅ Memory buffers for context
- ✅ Lifecycle hooks for control

This enables enterprise-grade AI agents that are:
- **Reliable** - goals are completed
- **Intelligent** - conversations feel natural
- **Recoverable** - interruptions don't break workflows
- **Auditable** - full state tracking
- **Composable** - modular, reusable components

**The future of enterprise AI isn't deterministic OR agentic—it's guided determinism.**# Agent Graph with Guided Determinism: System 2 Reasoning Meets FSM Control

## Executive Summary

Agent Graph is Agentforce's core orchestration engine that transforms AI agents from simple ReAct pattern responders into **System 2 reasoning agents** operating within a **Finite State Machine (FSM)**. Combined with Lifecycle Hooks, this creates **guided determinism**—where agents can reason deeply and adapt intelligently while still guaranteeing task completion through structured state management and memory context.

---

## What Makes Agent Graph Different?

### Traditional Flow Builders vs. Agent Graph

```mermaid
graph TB
    subgraph "Traditional Flow Builder"
        TF1[Static Decision Tree]
        TF2[Hardcoded Paths]
        TF3[Intent Matching]
        TF4[Stateless Nodes]
        TF1 --> Break[Breaks on:<br/>• Out-of-order inputs<br/>• Interruptions<br/>• Clarifying questions<br/>• Multi-turn reasoning]
    end
    
    subgraph "Agent Graph System"
        AG1[System 2 Agents<br/>at Each Node]
        AG2[FSM-Driven<br/>Transitions]
        AG3[Memory Context<br/>& State Management]
        AG4[Lifecycle Hooks<br/>for Control]
        AG1 --> Enables[Enables:<br/>• Order-independent completion<br/>• Interruptible workflows<br/>• Context preservation<br/>• Multi-agent coordination]
    end
    
    style Break fill:#991b1b,stroke:#ef4444,stroke-width:2px,color:#ffffff
    style Enables fill:#166534,stroke:#22c55e,stroke-width:2px,color:#ffffff
    style TF1 fill:#7f1d1d,stroke:#dc2626,stroke-width:2px,color:#ffffff
    style AG1 fill:#166534,stroke:#22c55e,stroke-width:2px,color:#ffffff
```

---

## The Problem with ReAct Agents (Agentforce v1-v3)

Traditional Agentforce agents use **ReAct pattern** (Reason + Act), which is **non-deterministic**:

```mermaid
graph TD
    User[User: I need help with my delivery] --> LLM[LLM Processing]
    LLM --> Random{Probabilistic<br/>Topic Selection}
    Random -->|40% chance| Topic1[Delivery Topic]
    Random -->|30% chance| Topic2[Order Topic]
    Random -->|20% chance| Topic3[Returns Topic]
    Random -->|10% chance| Topic4[General Help]
    
    Topic1 --> Derail{User tries to derail:<br/>Can we talk about<br/>returns instead?}
    Derail -->|Often succeeds| Topic3
    
    style Random fill:#ffcccc,stroke:#cc0000,stroke-width:2px
    style Derail fill:#ffcccc,stroke:#cc0000,stroke-width:2px
    style LLM fill:#ffe6cc,stroke:#ff9933,stroke-width:2px
```

**Problems:**
- Same input → different outputs
- Users can easily derail agents
- Cannot guarantee task completion
- No way to enforce sequential workflows

---

## System 2 Agents: The Foundation of Agent Graph

### What is a System 2 Agent?

Inspired by Daniel Kahneman's *Thinking, Fast and Slow*:

| Type | Characteristics | In AI |
|------|----------------|-------|
| **System 1** | Fast, intuitive, heuristic | ReAct agents, simple prompting |
| **System 2** | Slow, deliberate, logical | Agent Graph nodes with FSM |

**System 2 Agent Definition:**
- Explicitly reasons step-by-step before acting
- Maintains **structured internal state** (memory, plans, goals)
- Uses **planning, verification, and reflection**
- More computationally intensive but handles complex, multi-step reasoning
- **Embedded in FSM** for state transitions and goal tracking

```mermaid
graph TB
    subgraph "System 2 Agent Architecture"
        Input[User Input] --> Memory[Memory Context<br/>• Conversation history<br/>• Goal state<br/>• Progress tracking]
        Memory --> Planner[Planning Module<br/>• Break down goals<br/>• Identify dependencies<br/>• Order requirements]
        Planner --> Reasoner[Reasoning Engine<br/>• Deliberate logic<br/>• Verification steps<br/>• Reflection]
        Reasoner --> FSM[Finite State Machine<br/>• Current state<br/>• Completion conditions<br/>• Transition rules]
        FSM --> Tools[Tool Execution<br/>• Actions<br/>• API calls<br/>• Knowledge retrieval]
        Tools --> Memory
        FSM --> Output[Controlled Output]
    end
    
    style Memory fill:#e1f5ff,stroke:#0066cc,stroke-width:3px
    style FSM fill:#d4edda,stroke:#28a745,stroke-width:3px
    style Planner fill:#fff3cd,stroke:#ffc107,stroke-width:2px
```

### System 2 vs ReAct: Key Differences

| Feature | System 2 Agent | ReAct Agent |
|---------|---------------|-------------|
| **Origin** | Cognitive psychology (deliberate reasoning) | Prompting technique for LLMs |
| **Thinking Style** | Slow, structured, goal-oriented | Fast, reactive, step-by-step |
| **Mechanisms** | Explicit planning, FSM, memory, verification | Chain-of-thought + tool calls |
| **State Management** | Persistent across turns | Stateless (only in-context) |
| **Goal Tracking** | Explicit FSM states | Implicit in prompts |
| **Memory** | Structured memory buffers | Chat history only |
| **Derailment Resistance** | High (FSM enforces goals) | Low (easily derailed) |
| **Best For** | Complex, multi-turn, goal-driven tasks | Simple query-response interactions |

---

## Memory Context: The Key to State Persistence

### Why Memory Matters

Traditional ReAct agents lose context because they only have **chat history**. Agent Graph uses **structured memory buffers**:

```mermaid
graph LR
    subgraph "ReAct Agent Memory"
        R1[Chat History] --> R2[Messages in Context Window]
        R2 -.->|Lost after window fills| R3[❌ No persistent state]
    end
    
    subgraph "System 2 Agent Memory"
        S1[Memory Context] --> S2[Short-term Memory<br/>Current conversation]
        S1 --> S3[Working Memory<br/>Current goal state]
        S1 --> S4[Long-term Memory<br/>Past conversations]
        S1 --> S5[Episodic Memory<br/>Key facts & decisions]
        
        S2 --> Buffer[Memory Buffer<br/>FSM State Storage]
        S3 --> Buffer
        S4 --> Buffer
        S5 --> Buffer
    end
    
    style R3 fill:#f8d7da,stroke:#dc3545,stroke-width:2px
    style Buffer fill:#d4edda,stroke:#28a745,stroke-width:3px
    style S1 fill:#e1f5ff,stroke:#0066cc,stroke-width:3px
```

### Memory Buffer Components

**1. Conversation Context**
```json
{
  "messages": [
    {"role": "user", "content": "I need to reschedule"},
    {"role": "assistant", "content": "I'll help with that"}
  ],
  "turn_count": 5,
  "detected_locale": "en_US"
}
```

**2. Goal State (FSM)**
```json
{
  "current_goal": "complete_interview",
  "questions_answered": ["Q1", "Q2"],
  "questions_remaining": ["Q3", "Q4", "Q5"],
  "completion_percentage": 40,
  "can_transition": false
}
```

**3. Working Memory**
```json
{
  "customer_id": "003xx00001",
  "order_number": "ORD-12345",
  "delivery_date": "2025-08-15",
  "issue_type": "late_delivery"
}
```

**4. Episodic Memory**
```json
{
  "key_decisions": [
    "User confirmed address is correct",
    "Preferred delivery window: 9am-12pm"
  ],
  "blockers": []
}
```

### Requirements

**Scenario**: An AI agent conducts structured job interviews with candidates.

**Strict Requirements:**
1. Ask 5 interview questions **in exact order**
2. Do not accept "yes/no" answers - require detailed responses
3. Cannot skip questions or change order
4. After all questions answered → qualify candidate → end session
5. Agent can answer candidate questions BUT cannot be derailed from its mission
6. Must maintain 100% agentic intelligence (not a phone menu)

## How Memory Context Prevents Derailment

### Example Flow with Memory Buffer

```mermaid
sequenceDiagram
    participant User
    participant Interview as Interview Agent
    participant Memory as Memory Buffer
    participant FSM as Finite State Machine
    participant Hook as Lifecycle Hook

    User->>Interview: "Hi, I'm ready"
    Interview->>Memory: Check state
    Memory-->>Interview: {questions_answered: [], current_q: 1}
    Interview->>FSM: What's my state?
    FSM-->>Interview: State: Q1_REQUIRED
    Hook->>Interview: Inject Q1 context
    Interview->>User: "Can you carry 20kg? Provide examples."
    
    User->>Interview: "Can I get another question first?"
    Interview->>Memory: Check if Q1 answered
    Memory-->>Interview: {questions_answered: []}
    Interview->>FSM: Can I skip to Q2?
    FSM-->>Interview: NO - Q1 must be answered
    Hook->>Interview: Enforce Q1 requirement
    Interview->>User: "I need to confirm this first. Can you carry 20kg?"
    
    User->>Interview: "Yes, I carry heavy items daily at my job"
    Interview->>Memory: Store Q1 answer
    Memory-->>Memory: {questions_answered: ["Q1"], current_q: 2}
    Interview->>FSM: Update state
    FSM-->>Interview: State: Q1_COMPLETE → Q2_REQUIRED
    Hook->>Interview: Inject Q2 context
    Interview->>User: "Great! Do you have cleaning experience?"
    
    Note over Memory: Memory persists:<br/>• Goal progress<br/>• Cannot derail<br/>• Must complete Q1-Q5
```

### Memory Buffer Stays On Track

**Without Memory Buffer (ReAct Agent):**
```
User: "Can I get another question?"
Agent: "Sure!" → Loses track of requirements
```

**With Memory Buffer (System 2 Agent):**
```python
# Memory buffer enforces goal
memory_state = {
    "interview_goal": "complete_all_5_questions",
    "questions_answered": [1, 2],  # Only Q1, Q2 done
    "current_question": 3,
    "can_transition": False,  # Cannot leave until complete
    "fsm_state": "Q3_REQUIRED"
}

# Hook checks memory before allowing any action
if memory_state["questions_answered"].length < 5:
    # Lock to current node
    prevent_handoff()
    inject_next_question_context()
```

---

## The Solution: Guided Determinism = FSM + Memory + Hooks

### Three-Layer Architecture

```mermaid
graph TB
    subgraph "Layer 1: Deterministic Control"
        L1A[Agent Graph Structure<br/>Directed graph of nodes]
        L1B[Finite State Machine<br/>State transitions & goals]
        L1C[Memory Buffers<br/>Persistent context]
    end
    
    subgraph "Layer 2: Steering Mechanisms"
        L2A[Lifecycle Hooks<br/>Pre/post reasoning steps]
        L2B[Context Injection<br/>Dynamic prompting]
        L2C[Transition Guards<br/>FSM enforcement]
    end
    
    subgraph "Layer 3: Agentic Intelligence"
        L3A[System 2 Reasoning<br/>Deliberate planning]
        L3B[Natural Language<br/>Conversational quality]
        L3C[Tool Usage<br/>Dynamic actions]
    end
    
    L1A --> L2A
    L1B --> L2B
    L1C --> L2C
    L2A --> L3A
    L2B --> L3B
    L2C --> L3C
    
    L3A --> Output[Intelligent & Predictable Output]
    L3B --> Output
    L3C --> Output
    
    style L1A fill:#d4edda,stroke:#28a745,stroke-width:2px
    style L1B fill:#d4edda,stroke:#28a745,stroke-width:2px
    style L1C fill:#d4edda,stroke:#28a745,stroke-width:2px
    style L2A fill:#fff3cd,stroke:#ffc107,stroke-width:2px
    style L2B fill:#fff3cd,stroke:#ffc107,stroke-width:2px
    style L2C fill:#fff3cd,stroke:#ffc107,stroke-width:2px
    style L3A fill:#e1f5ff,stroke:#0066cc,stroke-width:2px
    style L3B fill:#e1f5ff,stroke:#0066cc,stroke-width:2px
    style L3C fill:#e1f5ff,stroke:#0066cc,stroke-width:2px
    style Output fill:#d4edda,stroke:#28a745,stroke-width:3px
```

---

## Implementation: Interview Agent with Hooks

### Agent Graph Structure

```mermaid
graph TD
    Start([User Enters]) --> Auth[Authentication Node]
    Auth --> Interview[Interview Agent Node<br/>Ask questions sequentially]
    Interview --> Check{All Questions<br/>Answered?}
    Check -->|No| Interview
    Check -->|Yes via Hook| Qualify[Qualifying Agent Node<br/>Evaluate candidate]
    Qualify --> Thank[Thank You Node]
    Thank --> End([Session Ends])
    
    Hook1[Hook: post_setup_agent]
    Hook2[Hook: pre_aggregate_tool_results]
    
    Hook1 -.->|Injects next question| Interview
    Hook2 -.->|Forces handoff when complete| Qualify
    
    style Auth fill:#e8f4f8,stroke:#006699,stroke-width:2px
    style Interview fill:#e1f5ff,stroke:#0066cc,stroke-width:3px
    style Qualify fill:#d4edda,stroke:#28a745,stroke-width:2px
    style Hook1 fill:#fff3cd,stroke:#ffc107,stroke-width:3px
    style Hook2 fill:#fff3cd,stroke:#ffc107,stroke-width:3px
    style Check fill:#fff3cd,stroke:#ffc107,stroke-width:2px
```

### Lifecycle Hooks in the Reasoning Loop

```mermaid
graph TB
    Start([Request Received]) --> SetupAgent[setup_agent]
    
    SetupAgent --> PostSetupHook{POST_SETUP_AGENT<br/>HOOK}
    PostSetupHook -->|Inject Question| AgentDecide[Agent Reasoning]
    
    AgentDecide --> CallTool[call_tool]
    CallTool --> PreAggHook{PRE_AGGREGATE<br/>TOOL_RESULTS<br/>HOOK}
    
    PreAggHook -->|Check Progress| Decision{All Questions<br/>Answered?}
    Decision -->|No - Stay in node| AgentDecide
    Decision -->|Yes - Force handoff| NextAgent[Transition to<br/>Qualifying Agent]
    
    PreAggHook -->|Continue| AggResults[aggregate_tool_results]
    AggResults --> Output[Agent Output]
    
    style PostSetupHook fill:#fff3cd,stroke:#ffc107,stroke-width:4px
    style PreAggHook fill:#fff3cd,stroke:#ffc107,stroke-width:4px
    style Decision fill:#fff3cd,stroke:#ffc107,stroke-width:3px
    style SetupAgent fill:#e1f5ff,stroke:#0066cc,stroke-width:2px
    style AgentDecide fill:#e1f5ff,stroke:#0066cc,stroke-width:2px
    style NextAgent fill:#d4edda,stroke:#28a745,stroke-width:3px
```

---

## Hook Implementation Examples

### Hook 1: post_setup_agent - Memory-Aware Question Injection

**Purpose**: Use memory buffer to inject only the next unanswered question.

```python
def post_setup_agent_hook(ctx, chat_history, agent_state, memory_buffer):
    """
    Runs after setup_agent step.
    Uses memory buffer to track progress and inject next question.
    """
    # Check memory buffer for progress
    answered = memory_buffer.get('questions_answered', [])
    
    # Define interview questions
    questions = [
        "Are you able to carry 20kg? Please provide examples.",
        "Do you have cleaning experience in healthcare? Describe it.",
        "Can you explain infection control best practices?",
        "Are you available for night shifts?",
        "Do you have reliable transportation?"
    ]
    
    # Calculate next question from memory
    next_idx = len(answered)
    
    if next_idx < len(questions):
        # Store in memory buffer
        memory_buffer['current_question'] = next_idx + 1
        memory_buffer['total_questions'] = len(questions)
        memory_buffer['goal_state'] = 'in_progress'
        
        # Inject into chat context (end of history for recency bias)
        system_message = f"""
MEMORY STATE: Question {next_idx + 1} of {len(questions)}
CRITICAL: You MUST ask this exact question next:
"{questions[next_idx]}"

MEMORY CONTEXT:
- Answered so far: {len(answered)} questions
- Remaining: {len(questions) - len(answered)} questions
- Cannot proceed until this question is answered fully

Do NOT accept yes/no answers. Require detailed responses.
If user tries to skip, refer to memory: "Our goal is to complete 
all {len(questions)} questions. We're on question {next_idx + 1}."
"""
        chat_history.append({
            "role": "system",
            "content": system_message
        })
    
    return ctx, chat_history, agent_state, memory_buffer
```

### Hook 2: pre_aggregate_tool_results - FSM-Driven Handoff

**Purpose**: Check memory buffer and FSM state. Force handoff when complete.

```python
def pre_aggregate_tool_results_hook(ctx, tool_results, 
                                   agent_state, memory_buffer):
    """
    Runs before aggregate_tool_results step.
    Checks memory buffer and FSM state for completion.
    """
    # Check memory buffer
    answered = memory_buffer.get('questions_answered', [])
    total = memory_buffer.get('total_questions', 5)
    
    # Update FSM state based on memory
    completion_rate = len(answered) / total
    memory_buffer['completion_percentage'] = completion_rate * 100
    
    # DETERMINISTIC FSM CHECK
    if len(answered) >= total:
        # Update memory: Goal complete
        memory_buffer['goal_state'] = 'complete'
        memory_buffer['interview_complete'] = True
        
        # Update FSM state
        ctx['fsm_state'] = 'INTERVIEW_COMPLETE'
        
        # FORCE HANDOFF to next agent
        ctx['force_handoff'] = {
            'target_agent': 'qualifying_agent_16jSG000000XXXXX',
            'reason': 'Interview goal achieved',
            'memory_context': {
                'questions_answered': answered,
                'candidate_responses': memory_buffer.get('responses', [])
            }
        }
        
        # Disable agent stickiness - allow transition
        ctx['agent_sticky'] = False
        
    else:
        # Goal NOT complete - update memory
        memory_buffer['goal_state'] = 'in_progress'
        remaining = total - len(answered)
        
        # LOCK to current agent via FSM
        ctx['fsm_state'] = f'Q{len(answered) + 1}_REQUIRED'
        ctx['agent_sticky'] = True  # Lock session
        ctx['disable_handoff'] = True  # Cannot escape
        
        # Store in memory why we can't transition
        memory_buffer['transition_blocker'] = {
            'reason': 'incomplete_interview',
            'remaining_questions': remaining
        }
        
    return ctx, tool_results, agent_state, memory_buffer
```

---

## Comparison: Before vs. After Hooks

### Scenario: User Tries to Derail

#### Without Hooks (Pure Stochastic)

```
User: Hi
Agent: Hello! I have a few questions. Are you able to carry 20kg?

User: Can I get another question first?
Agent: Of course! Do you have cleaning experience?

User: I don't want to answer that question
Agent: That's fine. Let's try another: Can you explain infection control?
```
❌ **Result**: Questions asked out of order. User controls flow. Unpredictable.

#### With Hooks (Deterministic Control)

```
User: Hi
Agent: Thank you! I have questions to confirm your qualifications.
      Are you able to carry 20kg? Please provide examples.

User: Can I get another question first?
Agent: I understand, but I need to confirm this requirement first.
      Are you able to carry 20kg? Please provide examples.

User: I don't want to answer that question
Agent: I understand your hesitation, but this is an important requirement.
      Could you please let me know if you can carry 20kg? Examples help.
```
✅ **Result**: Agent stays on track. Questions in order. User cannot derail.

---

## How Determinism Enhances Stochastic Processes

### The Hybrid Model

```mermaid
graph TB
    subgraph "Deterministic Layer: What We Control"
        D1[Workflow Structure<br/>Agent Graph]
        D2[Transition Rules<br/>Hooks]
        D3[State Management<br/>Variables]
        D4[Context Injection<br/>System Prompts]
    end
    
    subgraph "Stochastic Layer: What LLM Controls"
        S1[Natural Language<br/>Generation]
        S2[Conversation Flow<br/>Within Boundaries]
        S3[Question Answering<br/>Intelligence]
        S4[Tone & Empathy<br/>Adaptation]
    end
    
    D1 --> Guardrails[Guardrails]
    D2 --> Guardrails
    D3 --> Guardrails
    D4 --> Guardrails
    
    Guardrails --> S1
    Guardrails --> S2
    Guardrails --> S3
    Guardrails --> S4
    
    S1 --> Output[Intelligent BUT<br/>Predictable Output]
    S2 --> Output
    S3 --> Output
    S4 --> Output
    
    style D1 fill:#d4edda,stroke:#28a745,stroke-width:2px
    style D2 fill:#d4edda,stroke:#28a745,stroke-width:2px
    style D3 fill:#d4edda,stroke:#28a745,stroke-width:2px
    style D4 fill:#d4edda,stroke:#28a745,stroke-width:2px
    style Guardrails fill:#fff3cd,stroke:#ffc107,stroke-width:3px
    style S1 fill:#e1f5ff,stroke:#0066cc,stroke-width:2px
    style S2 fill:#e1f5ff,stroke:#0066cc,stroke-width:2px
    style S3 fill:#e1f5ff,stroke:#0066cc,stroke-width:2px
    style S4 fill:#e1f5ff,stroke:#0066cc,stroke-width:2px
    style Output fill:#d4edda,stroke:#28a745,stroke-width:3px
```

### Key Benefits

| Aspect | Pure Stochastic | Pure Deterministic | Hybrid (Graph + Hooks) |
|--------|----------------|-------------------|----------------------|
| **Task Completion** | ❌ Unpredictable | ✅ Guaranteed | ✅ Guaranteed |
| **Conversation Quality** | ✅ Natural | ❌ Rigid | ✅ Natural |
| **User Control** | ❌ Too much | ❌ None | ✅ Balanced |
| **Adaptability** | ✅ High | ❌ Low | ✅ High |
| **Debugging** | ❌ Hard | ✅ Easy | ✅ Easy |

---

## Research Foundation

### Why Recent Instructions Win

From "Lost in the Middle" (Liu et al., ICLR 2023):

```mermaid
graph LR
    subgraph "LLM Attention Pattern"
        Start[Context Start<br/>40% attention]
        Middle[Context Middle<br/>20% attention]
        End[Context End<br/>70% attention]
    end
    
    Start --> Effect1[Primacy Effect:<br/>First instructions matter]
    End --> Effect2[Recency Effect:<br/>Latest instructions<br/>DOMINATE]
    Middle --> Effect3[Lost in Middle:<br/>Ignored]
    
    style End fill:#d4edda,stroke:#28a745,stroke-width:3px
    style Effect2 fill:#d4edda,stroke:#28a745,stroke-width:3px
    style Middle fill:#f8d7da,stroke:#dc3545,stroke-width:2px
    style Effect3 fill:#f8d7da,stroke:#dc3545,stroke-width:2px
```

**Insight**: By injecting instructions at the END of chat history (via hooks), we leverage the LLM's natural recency bias to achieve near-deterministic behavior.

---

## Complete Flow Diagram

```mermaid
stateDiagram-v2
    [*] --> Authentication
    
    Authentication --> InterviewAgent: Auth Success
    
    state InterviewAgent {
        [*] --> PostSetupHook
        PostSetupHook --> InjectQuestion: Inject Q1
        InjectQuestion --> LLMReasoning
        LLMReasoning --> UserResponds
        UserResponds --> PreAggHook
        
        PreAggHook --> CheckComplete: Check if answered
        CheckComplete --> InjectQuestion: Not complete, inject next Q
        CheckComplete --> ForceHandoff: Complete!
    }
    
    InterviewAgent --> QualifyingAgent: Forced Handoff
    
    state QualifyingAgent {
        [*] --> EvaluateCandidate
        EvaluateCandidate --> DetermineQualification
        DetermineQualification --> [*]
    }
    
    QualifyingAgent --> ThankYou
    ThankYou --> [*]
    
    note right of PostSetupHook
        HOOK 1: Deterministically
        inject next question
    end note
    
    note right of PreAggHook
        HOOK 2: Check completion,
        force handoff if done
    end note
```

---

## Implementation Checklist

### For Your Use Case

- [ ] **Map your workflow** - Identify all stages and decision points
- [ ] **Define nodes** - Create agent graph with explicit nodes for each stage
- [ ] **Identify derailment risks** - Where can users take the agent off track?
- [ ] **Design hooks** - Which lifecycle points need deterministic control?
- [ ] **Implement state management** - What variables track progress?
- [ ] **Test edge cases** - Try to break your agent's determinism
- [ ] **Balance flexibility** - Ensure agent stays intelligent and helpful

---

## Key Takeaways

1. **Determinism ≠ Removing Intelligence**
   - Hooks control workflow structure
   - LLM maintains conversational quality

2. **Leverage LLM Nature, Don't Fight It**
   - Use recency bias (recent instructions win)
   - Inject context at right lifecycle points

3. **Graph + Hooks = Best of Both Worlds**
   - Graph: Define what should happen
   - Hooks: Enforce that it happens
   - LLM: Make it feel natural

4. **Start Simple, Add Control As Needed**
   - Not every use case needs hooks
   - Add determinism where business logic demands it

---

## References

- **Lost in the Middle**: Liu et al., ICLR 2023 - [arXiv](https://arxiv.org/abs/2307.03172)
- **Serial Position Effects**: Guo & Vosoughi, 2024 - [arXiv](https://arxiv.org/abs/2402.14153)
- **OpenAI Agents Guide**: [Practical Guide to Building Agents](https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf)
- **Salesforce Agent Graph**: Internal documentation on deterministic workflows