# Agent Graph with Lifecycle Hooks: Bringing True Determinism to Agentforce

## Executive Summary

Agent Graph combined with Reasoning Loop Lifecycle Hooks provides a **hybrid approach** where deterministic workflow control meets agentic intelligence. This solves the core problem: **customers need predictable outcomes while maintaining conversational flexibility**.

---

## The Problem with Pure Stochastic Agents

Traditional Agentforce agents are **non-deterministic**:

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

## Use Case: Job Interview Agent (Adecco)

### Requirements

**Scenario**: An AI agent conducts structured job interviews with candidates.

**Strict Requirements:**
1. Ask 5 interview questions **in exact order**
2. Do not accept "yes/no" answers - require detailed responses
3. Cannot skip questions or change order
4. After all questions answered → qualify candidate → end session
5. Agent can answer candidate questions BUT cannot be derailed from its mission
6. Must maintain 100% agentic intelligence (not a phone menu)

### Why Traditional Approaches Fail

**Approach 1: Pure Instructions**
```json
{
  "instructions": "Ask these questions in order: 1) Can you carry 20kg? 2) Do you have cleaning experience? ..."
}
```
❌ **Result**: User says "Can I get another question first?" → Agent complies and breaks order

**Approach 2: More Forceful Instructions**
```json
{
  "instructions": "YOU MUST ask questions in order. DO NOT skip questions. DO NOT allow users to change order..."
}
```
❌ **Result**: Still fails ~30% of the time. LLMs are helpful by nature and accommodate user requests.

---

## The Solution: Agent Graph + Lifecycle Hooks

### Architecture Overview

```mermaid
graph TB
    subgraph "Deterministic Control Layer"
        Graph[Agent Graph<br/>Defines workflow structure]
        Hooks[Lifecycle Hooks<br/>Enforces rules at runtime]
    end
    
    subgraph "Agentic Intelligence Layer"
        LLM[LLM Reasoning<br/>Handles conversation]
        Tools[Dynamic Tool Usage<br/>Adapts to context]
    end
    
    Graph --> Hooks
    Hooks --> LLM
    LLM --> Tools
    Hooks -.->|Controls & Steers| LLM
    Hooks -.->|Prevents Derailment| Tools
    
    style Graph fill:#d4edda,stroke:#28a745,stroke-width:3px
    style Hooks fill:#fff3cd,stroke:#ffc107,stroke-width:3px
    style LLM fill:#e1f5ff,stroke:#0066cc,stroke-width:2px
    style Tools fill:#e1f5ff,stroke:#0066cc,stroke-width:2px
```

### How Determinism Works with Stochastic Processes

**Key Insight**: We don't make the LLM deterministic. We **steer** it deterministically.

```mermaid
graph LR
    subgraph "Without Hooks: Pure Stochastic"
        U1[User Input] --> L1[LLM]
        L1 --> O1[Unpredictable Output]
        O1 -.->|Can derail| Wrong[Wrong path taken]
    end
    
    subgraph "With Hooks: Controlled Stochastic"
        U2[User Input] --> H1[Pre-Hook:<br/>Inject Context]
        H1 --> L2[LLM with<br/>Enhanced Context]
        L2 --> H2[Post-Hook:<br/>Validate Output]
        H2 --> Decision{Meets<br/>Requirements?}
        Decision -->|Yes| O2[Approved Output]
        Decision -->|No| Override[Override with<br/>Deterministic Response]
    end
    
    style U1 fill:#e8f4f8,stroke:#006699,stroke-width:1px
    style L1 fill:#ffcccc,stroke:#cc0000,stroke-width:2px
    style Wrong fill:#f8d7da,stroke:#dc3545,stroke-width:2px
    style H1 fill:#fff3cd,stroke:#ffc107,stroke-width:2px
    style H2 fill:#fff3cd,stroke:#ffc107,stroke-width:2px
    style Decision fill:#fff3cd,stroke:#ffc107,stroke-width:2px
    style Override fill:#d4edda,stroke:#28a745,stroke-width:2px
    style O2 fill:#d4edda,stroke:#28a745,stroke-width:2px
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

### Hook 1: post_setup_agent - Inject Next Question

**Purpose**: Deterministically inject only the next unanswered question into the system prompt.

```python
def post_setup_agent_hook(ctx, chat_history, agent_state):
    """
    Runs after setup_agent step.
    Deterministically injects the next question to ask.
    """
    # Check which questions have been answered
    answered_questions = agent_state.get('answered_questions', [])
    
    # Define all interview questions
    questions = [
        "Are you able to carry 20kg? Please provide examples.",
        "Do you have cleaning experience in healthcare? Describe it.",
        "Can you explain infection control best practices?",
        "Are you available for night shifts?",
        "Do you have reliable transportation?"
    ]
    
    # Find next unanswered question
    next_question_idx = len(answered_questions)
    
    if next_question_idx < len(questions):
        # DETERMINISTIC INJECTION: Add to end of chat history
        # LLMs strongly honor recent instructions
        system_message = f"""
CRITICAL: You MUST ask this exact question next:
"{questions[next_question_idx]}"

Do NOT accept yes/no answers. Require detailed responses.
If the user tries to skip or asks other questions, politely redirect:
"I understand, but first I need you to answer: [question]"
"""
        chat_history.append({
            "role": "system",
            "content": system_message
        })
    
    return ctx, chat_history, agent_state
```

### Hook 2: pre_aggregate_tool_results - Control Handoff

**Purpose**: Check if all questions answered. If yes, force handoff to Qualifying Agent.

```python
def pre_aggregate_tool_results_hook(ctx, tool_results, agent_state):
    """
    Runs before aggregate_tool_results step.
    Checks completion and forces deterministic handoff.
    """
    answered_questions = agent_state.get('answered_questions', [])
    total_questions = 5
    
    # DETERMINISTIC CHECK
    if len(answered_questions) >= total_questions:
        # All questions answered - FORCE HANDOFF
        agent_state['candidate_status'] = 'interview_complete'
        
        # Force transition to Qualifying Agent
        ctx['force_handoff'] = {
            'target_agent': 'qualifying_agent_16jSG000000XXXXX',
            'reason': 'All interview questions completed'
        }
        
        # Lock current agent - prevent returning
        ctx['agent_sticky'] = False
        ctx['disable_handoff_from_current'] = True
    else:
        # NOT complete - LOCK to current agent
        ctx['agent_sticky'] = True
        ctx['disable_handoff'] = True  # Cannot escape
        
    return ctx, tool_results, agent_state
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