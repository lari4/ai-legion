# AI Legion - Agent Pipelines Documentation

This document describes all agent pipelines and workflows in the AI Legion application, showing how data flows through the system and which prompts are used at each stage.

## Table of Contents

1. [Main Execution Pipeline](#main-execution-pipeline)
2. [Memory Management Pipeline](#memory-management-pipeline)
3. [Action Execution Pipeline](#action-execution-pipeline)
4. [Message Bus Pipeline](#message-bus-pipeline)
5. [Web Content Pipeline](#web-content-pipeline)
6. [Module Initialization Pipeline](#module-initialization-pipeline)

---

## Main Execution Pipeline

**Purpose:** The core agent loop that processes incoming messages and makes autonomous decisions.

**Entry Points:**
- `src/main.ts:60` - `agent.start()` initiates the agent
- `src/agent.ts:28` - `Agent.start()` method begins execution

### Pipeline Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        AGENT STARTUP                                │
│                     (src/agent.ts:28-54)                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Subscribe to Message Bus                                       │
│     └─> Filter messages by targetAgentIds                          │
│     └─> Append matching messages to memory                         │
│                                                                     │
│  2. Start Periodic Action Loop (every 1000ms)                      │
│     └─> Calls takeAction()                                         │
│                                                                     │
│  3. [Optional] Start Heartbeat Loop                                │
│     └─> Disabled by default                                        │
│                                                                     │
└──────────────────────┬──────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    ACTION CYCLE LOOP                                │
│                  (src/agent.ts:56-89)                               │
│                  Runs every 1000ms                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Step 1: Retrieve Events from Memory                               │
│  ┌───────────────────────────────────────┐                         │
│  │ events = await memory.retrieve()      │                         │
│  └───────────────────────────────────────┘                         │
│                       │                                             │
│                       ▼                                             │
│  Step 2: Check if Action Needed                                    │
│  ┌───────────────────────────────────────────────────┐             │
│  │ Is last event a "decision"?                       │             │
│  │   YES → EXIT (prevent duplicate actions)          │             │
│  │   NO  → Continue                                  │             │
│  └───────────────────────────────────────────────────┘             │
│                       │                                             │
│                       ▼                                             │
│  Step 3: Make Decision via LLM                                     │
│  ┌───────────────────────────────────────────────────┐             │
│  │ actionText = await makeDecision(events)           │             │
│  │                                                   │             │
│  │ PROMPTS USED:                                     │             │
│  │ • Core System Prompt (always present)             │             │
│  │ • Goals Management Pinned Prompt                  │             │
│  │ • Notes Management Pinned Prompt                  │             │
│  │ • [Any module pinned messages]                    │             │
│  │                                                   │             │
│  │ DATA SENT TO LLM:                                 │             │
│  │ • All events converted to OpenAI messages         │             │
│  │ • System message with agent instructions          │             │
│  │ • User messages from message bus                  │             │
│  │ • Previous assistant decisions                    │             │
│  └───────────────────────────────────────────────────┘             │
│                       │                                             │
│                       ▼                                             │
│  Step 4: Store Decision in Memory                                  │
│  ┌───────────────────────────────────────────────────┐             │
│  │ events = await memory.append({                    │             │
│  │   type: "decision",                               │             │
│  │   actionText: actionText                          │             │
│  │ })                                                │             │
│  │                                                   │             │
│  │ NOTE: May trigger memory summarization            │             │
│  └───────────────────────────────────────────────────┘             │
│                       │                                             │
│                       ▼                                             │
│  Step 5: Parse Action Text                                         │
│  ┌───────────────────────────────────────────────────┐             │
│  │ result = parseAction(actions, actionText)         │             │
│  │                                                   │             │
│  │ Extracts:                                         │             │
│  │ • Action name                                     │             │
│  │ • Thoughts                                        │             │
│  │ • Parameters                                      │             │
│  └───────────────────────────────────────────────────┘             │
│                       │                                             │
│                       ▼                                             │
│  Step 6: Valid Action?                                             │
│  ┌───────────────────────────────────────────────────┐             │
│  │ Is result an error?                               │             │
│  │   YES → Send error message to agent               │◄────┐       │
│  │         (Uses Error Message Template)             │     │       │
│  │   NO  → Continue to execution                     │     │       │
│  └───────────────────────────────────────────────────┘     │       │
│                       │                                    │       │
│                       NO                                   │       │
│                       ▼                                    │       │
│  Step 7: Execute Action                                   │       │
│  ┌───────────────────────────────────────────────────┐    │       │
│  │ await actionHandler.handle(agentId, action)       │    │       │
│  │                                                   │    │       │
│  │ • Looks up module context                         │    │       │
│  │ • Calls action.execute()                          │    │       │
│  │ • Action may send messages back to agent          │────┘       │
│  └───────────────────────────────────────────────────┘             │
│                       │                                             │
│                       ▼                                             │
│  Step 8: [Optional] Delay                                          │
│  ┌───────────────────────────────────────────────────┐             │
│  │ If AGENT_DELAY env var set:                       │             │
│  │   await sleep(delay)                              │             │
│  └───────────────────────────────────────────────────┘             │
│                       │                                             │
│                       ▼                                             │
│                  LOOP BACK TO STEP 1                                │
│                  (after 1000ms)                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Data Flow Through Pipeline

**Input Data:**
- Messages from message bus (user input, inter-agent messages, system messages)
- Previous agent decisions and memory

**Data Transformations:**

1. **Events → OpenAI Messages** (`src/make-decision.ts:26-49`)
   - Adds headers to messages:
     - Agent-to-agent: `"--- MESSAGE FROM AGENTNAME ---"`
     - Errors: `"--- ERROR ---"`
     - System messages: No header

2. **LLM Response → Action Text** (`src/make-decision.ts:22`)
   - Extracts raw text from completion

3. **Action Text → Parsed Action** (`src/parse-action.ts:19-114`)
   - Converts YAML-like format to structured Action object
   - Example transformation:
     ```
     Input (Action Text):
     action: addGoal
     thoughts: |
       I need to add this to my goals
     goal: Complete the documentation

     Output (Action Object):
     {
       actionDef: AddGoalActionDefinition,
       thoughts: "I need to add this to my goals",
       parameters: { goal: "Complete the documentation" }
     }
     ```

**Output Data:**
- Action execution results
- Messages sent to message bus
- Updated memory state

### Prompts Used

**Primary Prompts (Always Active):**

1. **Core System Prompt** (`src/module/definitions/core.ts:9-80`)
   - Defines agent identity and action format
   - Lists all available actions
   - See: [PROMPTS_DOCUMENTATION.md - Section 1](#)

2. **Goals Management Pinned Prompt** (`src/module/definitions/goals.ts:18-32`)
   - Shows current goals and their status
   - Instructs on goal lifecycle
   - See: [PROMPTS_DOCUMENTATION.md - Section 2](#)

3. **Notes Management Pinned Prompt** (`src/module/definitions/notes.ts:10-21`)
   - Shows current note titles
   - Instructs on note management
   - See: [PROMPTS_DOCUMENTATION.md - Section 3](#)

**Conditional Prompts:**

4. **Heartbeat Prompt** (`src/agent.ts:48-49`) - Only if enabled
   - Triggers periodic activity check
   - See: [PROMPTS_DOCUMENTATION.md - Section 6](#)

### Key Decision Points

1. **Line 61**: Should agent take action?
   - Check: Is last event a decision?
   - If YES: Skip this cycle (prevents loops)
   - If NO: Proceed with decision-making

2. **Line 72**: Is parsed action valid?
   - Check: Did parseAction return error?
   - If YES: Send error message to agent
   - If NO: Execute the action

3. **Line 66-69**: Does memory need summarization?
   - Check: Context window approaching limit?
   - Handled internally by memory.append()
   - If YES: Triggers Memory Management Pipeline

### Error Handling

**Errors trigger feedback to agent:**
- Unknown action → Error message with available actions list
- Invalid parameters → Error message with usage text
- Missing parameters → Error message with required parameters
- Execution failure → Error message with failure details

**All errors use the Error Message Header:**
```
--- ERROR ---

[Error details]
```

### Performance Characteristics

- **Loop Frequency:** 1000ms (1 second between cycles)
- **Optional Delay:** Configurable via AGENT_DELAY environment variable
- **Async Operations:** All I/O operations are async (memory, LLM calls, actions)
- **Serialization:** TaskQueue ensures operations don't overlap

