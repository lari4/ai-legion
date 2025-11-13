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

---

## Memory Management Pipeline

**Purpose:** Manages the agent's context window with automatic summarization to prevent overflow and maintain relevant historical context.

**Entry Points:**
- `src/memory/memory.ts:23` - `append(event)` - Add new event to memory
- `src/memory/memory.ts:39` - `retrieve()` - Get all events for LLM

### Pipeline Overview

The Memory Management Pipeline has three main sub-pipelines:
1. **Retrieve Events** - Assembles current context for LLM
2. **Append Events** - Adds new events and manages errors
3. **Summarization** - Compresses old events when context fills up

### Sub-Pipeline 1: Retrieve Events

**Purpose:** Constructs the full context window for the LLM by combining the introduction with stored events.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    RETRIEVE EVENTS PIPELINE                         │
│                   (src/memory/memory.ts:39-53)                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Step 1: Get Introduction                                          │
│  ┌───────────────────────────────────────────────────┐             │
│  │ const intro = await this.getIntroduction()        │             │
│  │                                                   │             │
│  │ Combines all module pinned messages:              │             │
│  │ • Core system prompt                              │             │
│  │ • Goals list with current status                  │             │
│  │ • Notes list with titles                          │             │
│  │ • Any other module pinned messages                │             │
│  │                                                   │             │
│  │ Format:                                           │             │
│  │   --- CORE ---                                    │             │
│  │   [Core system prompt]                            │             │
│  │                                                   │             │
│  │   --- GOALS ---                                   │             │
│  │   [Goals pinned message]                          │             │
│  │                                                   │             │
│  │   --- NOTES ---                                   │             │
│  │   [Notes pinned message]                          │             │
│  └───────────────────────────────────────────────────┘             │
│                       │                                             │
│                       ▼                                             │
│  Step 2: Load Stored Events                                        │
│  ┌───────────────────────────────────────────────────┐             │
│  │ let events = await this.store.get(EVENTS_KEY)     │             │
│  │                                                   │             │
│  │ If empty (first run):                             │             │
│  │   events = [{                                     │             │
│  │     type: "decision",                             │             │
│  │     actionText: "noop"                            │             │
│  │   }]                                              │             │
│  └───────────────────────────────────────────────────┘             │
│                       │                                             │
│                       ▼                                             │
│  Step 3: First-Time Initialization?                                │
│  ┌───────────────────────────────────────────────────┐             │
│  │ if (!this.hasRetrievedOnce)                       │             │
│  │   events = await this.summarize(                  │             │
│  │     [intro, ...events]                            │             │
│  │   )                                               │             │
│  │   await this.store.set(                           │             │
│  │     EVENTS_KEY,                                   │             │
│  │     events.slice(1)                               │             │
│  │   )                                               │             │
│  │   this.hasRetrievedOnce = true                    │             │
│  └───────────────────────────────────────────────────┘             │
│                       │                                             │
│                       ▼                                             │
│  Step 4: Return Combined Events                                    │
│  ┌───────────────────────────────────────────────────┐             │
│  │ return [intro, ...events]                         │             │
│  │                                                   │             │
│  │ Structure:                                        │             │
│  │ [0] Introduction (always regenerated)             │             │
│  │ [1] Event | Summary                               │             │
│  │ [2] Event | Summary                               │             │
│  │ ...                                               │             │
│  │ [n] Latest event                                  │             │
│  └───────────────────────────────────────────────────┘             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Prompts Used:**
- **Core System Prompt** (in introduction)
- **Goals Pinned Prompt** (in introduction)
- **Notes Pinned Prompt** (in introduction)
- All module pinned messages (in introduction)

**Key Points:**
- Introduction is **always regenerated** on each retrieval (never stored)
- This ensures pinned messages reflect current state (goals, notes)
- First retrieval triggers summarization to establish baseline
- Events array excludes introduction when stored

### Sub-Pipeline 2: Append Events

**Purpose:** Adds new events to memory and manages error cleanup to prevent agents from repeating mistakes.

```
┌─────────────────────────────────────────────────────────────────────┐
│                     APPEND EVENT PIPELINE                           │
│                   (src/memory/memory.ts:23-37)                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Input: new event (decision or message)                            │
│                       │                                             │
│                       ▼                                             │
│  Step 1: Log Event                                                 │
│  ┌───────────────────────────────────────────────────┐             │
│  │ printEvent(event)                                 │             │
│  │ // Prints to console for debugging                │             │
│  └───────────────────────────────────────────────────┘             │
│                       │                                             │
│                       ▼                                             │
│  Step 2: Retrieve Current Events                                   │
│  ┌───────────────────────────────────────────────────┐             │
│  │ let events = await this.retrieve()                │             │
│  │ // Gets [intro, ...stored events]                 │             │
│  └───────────────────────────────────────────────────┘             │
│                       │                                             │
│                       ▼                                             │
│  Step 3: Error Cleanup Decision                                    │
│  ┌────────────────────────────────────────────────────────┐        │
│  │ Is this an "ok" message?                               │        │
│  │                                                        │        │
│  │ if (event.type === "message" &&                        │        │
│  │     event.message.type === "ok")                       │        │
│  │                                                        │        │
│  │   YES → events = removeErrors(events) ──────────┐      │        │
│  │   NO  → Skip cleanup                            │      │        │
│  └─────────────────────────────────────────────────┼──────┘        │
│                       │                            │               │
│                       │                            ▼               │
│                       │        ┌──────────────────────────────┐    │
│                       │        │  REMOVE ERRORS               │    │
│                       │        │  (src/memory/memory.ts:76-96)│    │
│                       │        ├──────────────────────────────┤    │
│                       │        │                              │    │
│                       │        │  Logic:                      │    │
│                       │        │  • Find last error message   │    │
│                       │        │  • Find decision before it   │    │
│                       │        │  • Remove both from events   │    │
│                       │        │                              │    │
│                       │        │  Result:                     │    │
│                       │        │  • Agent won't see the       │    │
│                       │        │    failed attempt            │    │
│                       │        │  • Prevents repeating same   │    │
│                       │        │    mistake                   │    │
│                       │        └──────────────────────────────┘    │
│                       │                            │               │
│                       │                            │               │
│                       ▼◄───────────────────────────┘               │
│  Step 4: Add New Event                                             │
│  ┌───────────────────────────────────────────────────┐             │
│  │ events = [...events, event]                       │             │
│  └───────────────────────────────────────────────────┘             │
│                       │                                             │
│                       ▼                                             │
│  Step 5: Summarize if Needed                                       │
│  ┌───────────────────────────────────────────────────┐             │
│  │ events = await this.summarize(events)             │             │
│  │                                                   │             │
│  │ Triggers summarization if context too large       │             │
│  │ (See Sub-Pipeline 3 for details)                  │             │
│  └───────────────────────────────────────────────────┘             │
│                       │                                             │
│                       ▼                                             │
│  Step 6: Persist to Storage                                        │
│  ┌───────────────────────────────────────────────────┐             │
│  │ await this.store.set(                             │             │
│  │   EVENTS_KEY,                                     │             │
│  │   events.slice(1)  // Exclude intro              │             │
│  │ )                                                 │             │
│  └───────────────────────────────────────────────────┘             │
│                       │                                             │
│                       ▼                                             │
│  Return: Updated events array                                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Key Decision:**
- **"ok" message detection** triggers error cleanup
  - Removes the last error message
  - Removes the decision that caused it
  - Prevents agent from seeing failed attempts
  - Helps agent avoid repeating mistakes

### Sub-Pipeline 3: Summarization

**Purpose:** Compresses old events when the context window approaches its token limit, preserving essential information while freeing space.

```
┌─────────────────────────────────────────────────────────────────────┐
│                  SUMMARIZATION PIPELINE                             │
│                 (src/memory/memory.ts:101-186)                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Input: events array (may be too large)                            │
│                       │                                             │
│                       ▼                                             │
│  Step 1: Calculate Token Counts                                    │
│  ┌────────────────────────────────────────────────────┐            │
│  │ const eventTokens = events.map(toOpenAiMessage)    │            │
│  │                           .map(countTokens)        │            │
│  │                                                    │            │
│  │ const cumulativeTokens =                           │            │
│  │   eventTokens.reduce((acc, tokens) => [           │            │
│  │     ...acc,                                        │            │
│  │     tokens + (last(acc) || 0)                      │            │
│  │   ], [])                                           │            │
│  │                                                    │            │
│  │ Example:                                           │            │
│  │   eventTokens:      [100, 200, 150, 300]          │            │
│  │   cumulativeTokens: [100, 300, 450, 750]          │            │
│  └────────────────────────────────────────────────────┘            │
│                       │                                             │
│                       ▼                                             │
│  Step 2: Check Threshold                                           │
│  ┌─────────────────────────────────────────────────────┐           │
│  │ const threshold = contextWindow * 0.75              │           │
│  │ const totalTokens = last(cumulativeTokens)          │           │
│  │                                                     │           │
│  │ Is totalTokens > threshold?                         │           │
│  │   NO  → Return events unchanged ──────────┐         │           │
│  │   YES → Continue to summarization         │         │           │
│  └───────────────────────────────────────────┼─────────┘           │
│                       │                      │                     │
│                       ▼                      │                     │
│  Step 3: Find Truncation Point              │                     │
│  ┌──────────────────────────────────────────┐│                     │
│  │ Calculate how many tokens over threshold ││                     │
│  │ overrun = totalTokens - threshold        ││                     │
│  │                                          ││                     │
│  │ Find index where cumulative ≈ overrun    ││                     │
│  │ (Binary search in cumulativeTokens)      ││                     │
│  │                                          ││                     │
│  │ Constraints:                             ││                     │
│  │ • Must leave at least 3 events           ││                     │
│  │ • Must summarize at least 50% of events  ││                     │
│  │   (excluding introduction)               ││                     │
│  └──────────────────────────────────────────┘│                     │
│                       │                      │                     │
│                       ▼                      │                     │
│  Step 4: Calculate Summary Word Limit       │                     │
│  ┌──────────────────────────────────────────┐│                     │
│  │ summaryWordLimit =                       ││                     │
│  │   Math.floor(threshold / 6)              ││                     │
│  │                                          ││                     │
│  │ Example:                                 ││                     │
│  │   threshold = 3000 tokens                ││                     │
│  │   summaryWordLimit = 500 words           ││                     │
│  └──────────────────────────────────────────┘│                     │
│                       │                      │                     │
│                       ▼                      │                     │
│  Step 5: Generate Summary via LLM           │                     │
│  ┌───────────────────────────────────────────────────┐             │
│  │ const eventsToSummarize =                         │             │
│  │   events.slice(1, truncationIndex + 1)            │             │
│  │   // Excludes intro, includes old events          │             │
│  │                                                   │             │
│  │ const summaryContent = await makeDecision([       │             │
│  │   ...eventsToSummarize,                           │             │
│  │   {                                               │             │
│  │     type: "message",                              │             │
│  │     message: {                                    │             │
│  │       type: "ok",                                 │             │
│  │       content: [MEMORY SUMMARIZATION PROMPT]      │             │
│  │     }                                             │             │
│  │   }                                               │             │
│  │ ])                                                │             │
│  │                                                   │             │
│  │ PROMPT USED:                                      │             │
│  │ "Write a summary in ${summaryWordLimit} words    │             │
│  │  or less of what has happened since (but not     │             │
│  │  including) the introductory message.            │             │
│  │  Include key information that you learned...     │             │
│  │  Use the second person voice...                  │             │
│  │  In this particular instance, your response      │             │
│  │  should just be raw text, not formatted as       │             │
│  │  an action."                                     │             │
│  │                                                   │             │
│  │ DATA SENT TO LLM:                                 │             │
│  │ • Events being summarized (old events)            │             │
│  │ • Summarization instruction prompt                │             │
│  │ • Word limit constraint                           │             │
│  │                                                   │             │
│  │ EXPECTED RESPONSE:                                │             │
│  │ Raw text summary (not action-formatted)           │             │
│  └───────────────────────────────────────────────────┘             │
│                       │                                             │
│                       ▼                                             │
│  Step 6: Create Summary Event                                      │
│  ┌───────────────────────────────────────────────────┐             │
│  │ const summaryText =                               │             │
│  │   "Several events are omitted here to free up"   │             │
│  │   "space in your context window, summarized"     │             │
│  │   "as follows:\n\n" + summaryContent             │             │
│  │                                                   │             │
│  │ const summaryEvent = {                            │             │
│  │   type: "message",                                │             │
│  │   message: {                                      │             │
│  │     type: "spontaneous",                          │             │
│  │     source: { type: "user" },                     │             │
│  │     targetAgentIds: [this.agentId],               │             │
│  │     content: summaryText                          │             │
│  │   }                                               │             │
│  │ }                                                 │             │
│  └───────────────────────────────────────────────────┘             │
│                       │                                             │
│                       ▼                                             │
│  Step 7: Validate Token Savings                                    │
│  ┌──────────────────────────────────────────────────────┐          │
│  │ const summarizedEvents = [                           │          │
│  │   events[0],              // Introduction            │          │
│  │   summaryEvent,           // New summary             │          │
│  │   ...events.slice(truncationIndex + 1) // Recent     │          │
│  │ ]                                                    │          │
│  │                                                      │          │
│  │ const newTotal = countTokens(summarizedEvents)       │          │
│  │ const oldTotal = countTokens(events)                 │          │
│  │                                                      │          │
│  │ Is newTotal < oldTotal?                              │          │
│  │   YES → Return summarizedEvents                      │          │
│  │   NO  → Return original events (summary too long)    │          │
│  └──────────────────────────────────────────────────────┘          │
│                       │                      │                     │
│                       ▼                      ▼                     │
│           Summarized Events        Original Events                │
│           (space saved)            (summary didn't help)           │
│                       └──────────────┴───────────────────┐         │
│                                                          │         │
│  Output: Events array (possibly with summary)           │         │
│                                                          ▼         │
└──────────────────────────────────────────────────────────────────────┘
                                                           │
                                                           │
                    ┌──────────────────────────────────────┘
                    ▼
           Events returned to caller
           (Main Execution Pipeline)
```

### Prompts Used

**Primary Prompt:**

**Memory Summarization Prompt** (`src/memory/memory.ts:154-156`)
- Instructs agent to compress history
- Specifies word limit
- Requests second-person voice
- Emphasizes information density
- Asks for raw text (not action format)
- See: [PROMPTS_DOCUMENTATION.md - Section 4](#)

### Data Transformations

1. **Events → Token Counts**
   - Event → OpenAI message format
   - Message → Token count
   - Individual → Cumulative sums

2. **Events → Summary Text**
   - Multiple events → Single LLM prompt
   - LLM response → Summary string
   - Summary string → Summary event

3. **Old Events → Summary Event**
   - N old events replaced by 1 summary event
   - Frees context window space
   - Preserves essential information

### Key Decision Points

1. **Line 26**: Is this an "ok" message?
   - Determines if error cleanup should run
   - "ok" means previous action succeeded
   - Removes failed attempts from context

2. **Line 107**: Is context window over threshold?
   - Threshold: 75% of context window
   - If NO: Skip summarization
   - If YES: Proceed with compression

3. **Line 170**: Does summary save tokens?
   - Compare new total vs old total
   - If YES: Use summarized events
   - If NO: Keep original events (summary was too verbose)

### Token Budget Management

**Context Window Allocation:**
- **Threshold Trigger:** 75% of context window
- **Compression Target:** Reduce to below threshold
- **Summary Word Limit:** threshold / 6 (approximately)
- **Minimum Events:** At least 3 events must remain
- **Summarization Coverage:** At least 50% of events (excluding intro)

**Example (4000 token context window):**
- Trigger threshold: 3000 tokens (75%)
- Summary word limit: 500 words
- If current total: 3500 tokens
- Overrun: 500 tokens
- Find events totaling ~500 tokens to summarize

### Error Recovery Strategy

**Problem:** Agents may repeat failed actions if they see errors in context.

**Solution:** Remove error pairs when action succeeds:
1. When "ok" message arrives (action succeeded)
2. Find last error message in events
3. Find the decision immediately before that error
4. Remove both from events array
5. Agent's context no longer shows the failed attempt

**Example:**
```
Before "ok" message:
[intro, decision1, message1, decision2_FAILED, error2, decision3, ok3]
                                      ↑         ↑
                                      Remove these

After "ok" message:
[intro, decision1, message1, decision3, ok3]
```

### Performance Characteristics

- **Token Counting:** Uses tiktoken library (accurate for GPT models)
- **Binary Search:** Efficient truncation point calculation (O(log n))
- **Parallel Processing:** None (summarization is sequential)
- **Storage:** Events persisted to file system after each append
- **Cache:** Introduction regenerated on each retrieval (not cached)

---

## Action Execution Pipeline

**Purpose:** Discovers, validates, parses, and executes agent actions based on LLM output.

**Entry Points:**
- `src/parse-action.ts:19` - `parseAction()` - Parse action text from LLM
- `src/action-handler.ts:12` - `handle()` - Execute parsed action

### Pipeline Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ACTION EXECUTION PIPELINE                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Input: Action text from LLM response                              │
│  Example:                                                          │
│    action: addGoal                                                 │
│    thoughts: |                                                     │
│      I need to track this task                                     │
│    goal: Complete the documentation                                │
│                       │                                             │
│                       ▼                                             │
│  ┌──────────────────────────────────────────────────────┐          │
│  │         PHASE 1: PARSING                             │          │
│  │      (src/parse-action.ts:19-114)                    │          │
│  ├──────────────────────────────────────────────────────┤          │
│  │                                                      │          │
│  │  Step 1: Validate Format                            │          │
│  │  ┌────────────────────────────────────────┐         │          │
│  │  │ Check first line format                │         │          │
│  │  │ Must start with non-whitespace         │         │          │
│  │  │   YES → Continue                       │         │          │
│  │  │   NO  → Return error                   │         │          │
│  │  └────────────────────────────────────────┘         │          │
│  │                  │                                   │          │
│  │                  ▼                                   │          │
│  │  Step 2: Transform to JSON                          │          │
│  │  ┌────────────────────────────────────────┐         │          │
│  │  │ • Prepend "name: " to text             │         │          │
│  │  │ • Replace MULTILINE_DELIMITER with     │         │          │
│  │  │   proper multiline format              │         │          │
│  │  │ • Split on newlines                    │         │          │
│  │  │ • Extract key:value pairs              │         │          │
│  │  │ • Build JSON object                    │         │          │
│  │  └────────────────────────────────────────┘         │          │
│  │                  │                                   │          │
│  │                  ▼                                   │          │
│  │  Step 3: Parse JSON                                 │          │
│  │  ┌────────────────────────────────────────┐         │          │
│  │  │ Extract:                               │         │          │
│  │  │ • action (or name): string             │         │          │
│  │  │ • thoughts: string (optional)          │         │          │
│  │  │ • ...parameters: Record<string, any>   │         │          │
│  │  └────────────────────────────────────────┘         │          │
│  │                  │                                   │          │
│  │                  ▼                                   │          │
│  │  Step 4: Action Exists?                             │          │
│  │  ┌─────────────────────────────────────────────┐    │          │
│  │  │ actionDef = dictionary.get(actionName)     │    │          │
│  │  │                                            │    │          │
│  │  │ if (!actionDef)                            │    │          │
│  │  │   Return ERROR:                            │    │          │
│  │  │   "Unknown action `${name}`.               │    │          │
│  │  │    Available actions: ..."                 │    │          │
│  │  └─────────────────────────────────────────────┘    │          │
│  │                  │                                   │          │
│  │                  ▼                                   │          │
│  │  Step 5: Validate Required Parameters              │          │
│  │  ┌─────────────────────────────────────────────┐    │          │
│  │  │ For each required parameter:               │    │          │
│  │  │   if (!(param in parameters))              │    │          │
│  │  │     Return ERROR:                          │    │          │
│  │  │     "Missing required parameter: ${param}" │    │          │
│  │  │     + usage text                           │    │          │
│  │  └─────────────────────────────────────────────┘    │          │
│  │                  │                                   │          │
│  │                  ▼                                   │          │
│  │  Step 6: Validate No Extra Parameters               │          │
│  │  ┌─────────────────────────────────────────────┐    │          │
│  │  │ For each provided parameter:               │    │          │
│  │  │   if (param not in actionDef)              │    │          │
│  │  │     Return ERROR:                          │    │          │
│  │  │     "Unexpected parameter: ${param}"       │    │          │
│  │  │     + usage text                           │    │          │
│  │  └─────────────────────────────────────────────┘    │          │
│  │                  │                                   │          │
│  │                  ▼                                   │          │
│  │  Step 7: Return Success                             │          │
│  │  ┌─────────────────────────────────────────────┐    │          │
│  │  │ return {                                   │    │          │
│  │  │   success: true,                           │    │          │
│  │  │   action: {                                │    │          │
│  │  │     actionDef,                             │    │          │
│  │  │     thoughts,                              │    │          │
│  │  │     parameters                             │    │          │
│  │  │   }                                        │    │          │
│  │  │ }                                          │    │          │
│  │  └─────────────────────────────────────────────┘    │          │
│  └──────────────────────────────────────────────────────┘          │
│                       │                                             │
│                       ▼                                             │
│  ┌──────────────────────────────────────────────────────┐          │
│  │         PHASE 2: EXECUTION                           │          │
│  │      (src/action-handler.ts:12-18)                   │          │
│  ├──────────────────────────────────────────────────────┤          │
│  │                                                      │          │
│  │  Step 1: Lookup Module Context                      │          │
│  │  ┌────────────────────────────────────────┐         │          │
│  │  │ const moduleInstance =                 │         │          │
│  │  │   actionToModuleMap.get(actionName)    │         │          │
│  │  │                                        │         │          │
│  │  │ const context =                        │         │          │
│  │  │   moduleInstance.context               │         │          │
│  │  │                                        │         │          │
│  │  │ Context includes:                      │         │          │
│  │  │ • agentId                              │         │          │
│  │  │ • allAgentIds                          │         │          │
│  │  │ • actionDictionary                     │         │          │
│  │  │ • state (module-specific)              │         │          │
│  │  └────────────────────────────────────────┘         │          │
│  │                  │                                   │          │
│  │                  ▼                                   │          │
│  │  Step 2: Execute Action                             │          │
│  │  ┌────────────────────────────────────────┐         │          │
│  │  │ await actionDef.execute({              │         │          │
│  │  │   parameters,                          │         │          │
│  │  │   context,                             │         │          │
│  │  │   sendMessage: (msg) => {              │         │          │
│  │  │     messageBus.send(msg)               │         │          │
│  │  │   }                                    │         │          │
│  │  │ })                                     │         │          │
│  │  │                                        │         │          │
│  │  │ Action may:                            │         │          │
│  │  │ • Read/write module state              │         │          │
│  │  │ • Call external APIs                   │         │          │
│  │  │ • Execute system commands              │         │          │
│  │  │ • Send messages to agent               │         │          │
│  │  └────────────────────────────────────────┘         │          │
│  └──────────────────────────────────────────────────────┘          │
│                       │                                             │
│                       ▼                                             │
│  Output: Messages sent to agent via message bus                   │
│  • "ok" message (success)                                          │
│  • "error" message (failure)                                       │
│  • "agentToAgent" message (inter-agent communication)              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Example Action Execution Flow

**Input (Action Text):**
```
action: addGoal
thoughts: |
  I should track this task as a goal
goal: Document all agent pipelines
```

**Parsing Phase:**
1. Validated format ✓
2. Transformed to JSON:
   ```json
   {
     "action": "addGoal",
     "thoughts": "I should track this task as a goal",
     "goal": "Document all agent pipelines"
   }
   ```
3. Action "addGoal" found in dictionary ✓
4. Required parameter "goal" present ✓
5. No extra parameters ✓

**Execution Phase:**
1. Lookup module: goals module
2. Get context: {agentId, state (JsonStore), ...}
3. Execute:
   ```typescript
   // src/module/definitions/goals.ts:43-51
   const goals = await state.get("goals") || [];
   await state.set("goals", [
     ...goals,
     { text: "Document all agent pipelines", complete: false }
   ]);
   sendMessage(messageBuilder.ok(agentId, "Goal added."));
   ```

**Output:**
- Message sent: `{type: "ok", content: "Goal added."}`
- Agent receives confirmation
- Next retrieval includes updated goals pinned message

### Data Transformations

1. **Action Text → JSON Object**
   - YAML-like format → JavaScript object
   - Multiline delimiters → Proper strings
   - Key:value pairs → Object properties

2. **JSON Object → Validated Action**
   - Lookup action definition
   - Validate parameters
   - Create Action object with actionDef reference

3. **Action → Execution Context**
   - Find owning module
   - Load module state
   - Prepare context object

### Error Messages

All errors include guidance to help the agent correct the issue:

- **Unknown action:** Lists all available actions
- **Missing parameter:** Shows required parameters and usage
- **Extra parameter:** Shows expected parameters and usage
- **Invalid format:** Explains correct action format

### Performance Characteristics

- **Parsing:** O(n) where n = number of lines in action text
- **Lookup:** O(1) hash map lookups for actions and modules
- **Validation:** O(p) where p = number of parameters
- **Execution:** Varies by action (I/O, computation, etc.)

---

## Message Bus Pipeline

**Purpose:** Enables publish/subscribe communication between agents and system components.

**Entry Points:**
- `src/in-memory-message-bus.ts:12` - `subscribe()` - Register listener
- `src/in-memory-message-bus.ts:20` - `send()` - Broadcast message

### Pipeline Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                     MESSAGE BUS PIPELINE                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────────────────────────────────────────────┐          │
│  │  INITIALIZATION (src/main.ts:25)                     │          │
│  ├──────────────────────────────────────────────────────┤          │
│  │                                                      │          │
│  │  const messageBus = new InMemoryMessageBus()        │          │
│  │  // Alternative: RedisMessageBus for distributed    │          │
│  │                                                      │          │
│  └──────────────────────────────────────────────────────┘          │
│                       │                                             │
│                       ▼                                             │
│  ┌──────────────────────────────────────────────────────┐          │
│  │  AGENT SUBSCRIPTION (src/agent.ts:30-34)             │          │
│  ├──────────────────────────────────────────────────────┤          │
│  │                                                      │          │
│  │  messageBus.subscribe(async (message) => {          │          │
│  │    // Filter by target                              │          │
│  │    if (!message.targetAgentIds.includes(this.id))   │          │
│  │      return; // Skip                                │          │
│  │                                                      │          │
│  │    // Append to memory                              │          │
│  │    await this.memory.append({                       │          │
│  │      type: "message",                               │          │
│  │      message                                        │          │
│  │    });                                              │          │
│  │  });                                                │          │
│  │                                                      │          │
│  └──────────────────────────────────────────────────────┘          │
│                       │                                             │
│                       │                                             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  MESSAGE FLOW                                               │   │
│  │                                                             │   │
│  │  [Source] ──────┐                                           │   │
│  │   - Action      │                                           │   │
│  │   - System      │                                           │   │
│  │   - User        │                                           │   │
│  │   - Agent       │                                           │   │
│  │                 │                                           │   │
│  │                 ▼                                           │   │
│  │          messageBuilder                                     │   │
│  │          (src/message.ts:36-52)                             │   │
│  │                 │                                           │   │
│  │                 ├─> spontaneous(agentId, content)           │   │
│  │                 ├─> ok(agentId, content)                    │   │
│  │                 ├─> error(agentId, content)                 │   │
│  │                 └─> agentToAgent(from, to[], content)       │   │
│  │                 │                                           │   │
│  │                 ▼                                           │   │
│  │          Message Object                                     │   │
│  │          {                                                  │   │
│  │            type: MessageType,                               │   │
│  │            source: MessageSource,                           │   │
│  │            targetAgentIds: string[],                        │   │
│  │            content: string                                  │   │
│  │          }                                                  │   │
│  │                 │                                           │   │
│  │                 ▼                                           │   │
│  │          messageBus.send(message)                           │   │
│  │          (src/in-memory-message-bus.ts:20)                  │   │
│  │                 │                                           │   │
│  │                 ▼                                           │   │
│  │          EventEmitter.emit("message", message)              │   │
│  │                 │                                           │   │
│  │                 ▼                                           │   │
│  │          Broadcast to All Subscribers                       │   │
│  │                 │                                           │   │
│  │       ┌─────────┼─────────┬─────────┐                       │   │
│  │       ▼         ▼         ▼         ▼                       │   │
│  │    Agent-1   Agent-2   Agent-3   System                     │   │
│  │       │         │         │         │                       │   │
│  │       ▼         ▼         ▼         ▼                       │   │
│  │    Filter    Filter    Filter    Process                    │   │
│  │    by ID     by ID     by ID                                │   │
│  │       │         │         │                                 │   │
│  │       ▼         ▼         ▼                                 │   │
│  │    Append    Skip       Append                              │   │
│  │    to        (not       to                                  │   │
│  │    Memory    targeted)  Memory                              │   │
│  │                                                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Message Types & Headers

When messages are converted to OpenAI format (`src/make-decision.ts:26-49`), headers are added:

1. **spontaneous** - No header
   ```
   This is your regularly scheduled heartbeat message. Is there anything you need to do?
   ```

2. **ok** - No header
   ```
   Goal added.
   ```

3. **error** - ERROR header
   ```
   --- ERROR ---

   Unknown action `foo`. Please refer to the list of available actions...
   ```

4. **agentToAgent** - FROM header
   ```
   --- MESSAGE FROM AGENT-2 ---

   I've completed the task you requested.
   ```

### Message Structure

```typescript
interface Message {
  type: "spontaneous" | "ok" | "error" | "agentToAgent";
  source:
    | { type: "user" }
    | { type: "agent", id: string };
  targetAgentIds: string[];  // Array of recipient IDs
  content: string;           // Message text
}
```

### Routing Logic

**1. Broadcasting:**
- All messages sent to all subscribers
- No selective routing at bus level
- Synchronous delivery (in-memory)

**2. Filtering:**
- Each subscriber filters by `targetAgentIds`
- Agent only processes messages where `targetAgentIds.includes(agentId)`
- Non-targeted messages are ignored

**3. Multi-Agent:**
- Single message can target multiple agents
- Each targeted agent receives and processes independently
- Useful for group communications

### Performance Characteristics

- **Delivery:** Synchronous (in-memory bus)
- **Scalability:** Single process only (use RedisMessageBus for distributed)
- **Latency:** Near-zero (EventEmitter)
- **Persistence:** None (messages not stored by bus)

---

## Web Content Pipeline

**Purpose:** Fetches and processes web pages, converting them to markdown and summarizing large content for agent consumption.

**Entry Points:**
- `src/module/definitions/web.ts:18` - `searchWeb` action
- `src/module/definitions/web.ts:49` - `readPage` action

### Pipeline Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                    WEB CONTENT PIPELINE                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────────────────────────────────────────────┐          │
│  │  WEB SEARCH FLOW (src/module/definitions/web.ts:18)  │          │
│  ├──────────────────────────────────────────────────────┤          │
│  │                                                      │          │
│  │  Action: searchWeb                                  │          │
│  │  Parameters: { query }                              │          │
│  │                  │                                   │          │
│  │                  ▼                                   │          │
│  │  Execute Google Custom Search API                   │          │
│  │  (src/module/definitions/web.ts:89-96)              │          │
│  │                  │                                   │          │
│  │                  ▼                                   │          │
│  │  Format Results                                     │          │
│  │  "Search results for '${query}':\n\n"               │          │
│  │  "1. [Title](URL)\n"                                │          │
│  │  "2. [Title](URL)\n"                                │          │
│  │  ...                                                │          │
│  │                  │                                   │          │
│  │                  ▼                                   │          │
│  │  Send to Agent via Message Bus                      │          │
│  │                                                      │          │
│  └──────────────────────────────────────────────────────┘          │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  PAGE READING FLOW (src/module/definitions/web.ts:49-84)     │  │
│  ├──────────────────────────────────────────────────────────────┤  │
│  │                                                              │  │
│  │  Action: readPage                                           │  │
│  │  Parameters: { url }                                        │  │
│  │                  │                                           │  │
│  │                  ▼                                           │  │
│  │  Calculate Token Limits                                     │  │
│  │  maxCompletionTokens = contextWindow / 4                    │  │
│  │                  │                                           │  │
│  │                  ▼                                           │  │
│  │  ┌────────────────────────────────────────────────────┐     │  │
│  │  │  PAGE SUMMARIZATION SUBPROCESS                     │     │  │
│  │  │  (src/module/definitions/web.ts:98-217)            │     │  │
│  │  ├────────────────────────────────────────────────────┤     │  │
│  │  │                                                    │     │  │
│  │  │  Step 1: Launch Browser                           │     │  │
│  │  │  ┌──────────────────────────────────┐             │     │  │
│  │  │  │ browser = await puppeteer.launch │             │     │  │
│  │  │  │ page = await browser.newPage()   │             │     │  │
│  │  │  └──────────────────────────────────┘             │     │  │
│  │  │                  │                                 │     │  │
│  │  │                  ▼                                 │     │  │
│  │  │  Step 2: Configure Markdown Converter             │     │  │
│  │  │  ┌──────────────────────────────────┐             │     │  │
│  │  │  │ turndownService = new Turndown() │             │     │  │
│  │  │  │ Remove: style, script, img tags  │             │     │  │
│  │  │  └──────────────────────────────────┘             │     │  │
│  │  │                  │                                 │     │  │
│  │  │                  ▼                                 │     │  │
│  │  │  Step 3: Fetch Page                               │     │  │
│  │  │  ┌──────────────────────────────────┐             │     │  │
│  │  │  │ await page.goto(url)             │             │     │  │
│  │  │  │ const html = await page.content()│             │     │  │
│  │  │  └──────────────────────────────────┘             │     │  │
│  │  │                  │                                 │     │  │
│  │  │                  ▼                                 │     │  │
│  │  │  Step 4: Convert to Markdown                      │     │  │
│  │  │  ┌──────────────────────────────────┐             │     │  │
│  │  │  │ markdown = turndown(html)        │             │     │  │
│  │  │  │ Remove escaped underscores       │             │     │  │
│  │  │  └──────────────────────────────────┘             │     │  │
│  │  │                  │                                 │     │  │
│  │  │                  ▼                                 │     │  │
│  │  │  Step 5: Chunk Content                            │     │  │
│  │  │  ┌──────────────────────────────────────────┐     │     │  │
│  │  │  │ Split by lines                           │     │     │  │
│  │  │  │ maxChunkTokens = maxTokens * 0.9         │     │     │  │
│  │  │  │                                          │     │     │  │
│  │  │  │ For each line:                           │     │     │  │
│  │  │  │   if (chunk + line > maxChunkTokens)     │     │     │  │
│  │  │  │     start new chunk                      │     │     │  │
│  │  │  │   else                                   │     │     │  │
│  │  │  │     append line to chunk                 │     │     │  │
│  │  │  │                                          │     │     │  │
│  │  │  │ Result: chunks[] (array of strings)      │     │     │  │
│  │  │  └──────────────────────────────────────────┘     │     │  │
│  │  │                  │                                 │     │  │
│  │  │                  ▼                                 │     │  │
│  │  │  Step 6: Calculate Chunk Summary Limits           │     │  │
│  │  │  ┌──────────────────────────────────────────┐     │     │  │
│  │  │  │ maxChunkSummaryTokens =                  │     │     │  │
│  │  │  │   maxSummaryTokens / chunks.length       │     │     │  │
│  │  │  │                                          │     │     │  │
│  │  │  │ Convert to character limit:              │     │     │  │
│  │  │  │ chunkSummaryLimitText =                  │     │     │  │
│  │  │  │   "${maxChunkSummaryTokens * 4} chars"   │     │     │  │
│  │  │  └──────────────────────────────────────────┘     │     │  │
│  │  │                  │                                 │     │  │
│  │  │                  ▼                                 │     │  │
│  │  │  Step 7: Summarize Each Chunk (Parallel)          │     │  │
│  │  │  ┌────────────────────────────────────────────┐   │     │  │
│  │  │  │ await Promise.all(chunks.map(chunk => {   │   │     │  │
│  │  │  │   createChatCompletion({                  │   │     │  │
│  │  │  │     model,                                │   │     │  │
│  │  │  │     messages: [{                          │   │     │  │
│  │  │  │       role: "user",                       │   │     │  │
│  │  │  │       content: [WEB SUMMARIZATION PROMPT] │   │     │  │
│  │  │  │     }]                                    │   │     │  │
│  │  │  │   })                                      │   │     │  │
│  │  │  │ }))                                       │   │     │  │
│  │  │  │                                           │   │     │  │
│  │  │  │ PROMPT USED:                              │   │     │  │
│  │  │  │ "Modify the following markdown excerpt   │   │     │  │
│  │  │  │  only as much as necessary to bring it   │   │     │  │
│  │  │  │  under a maximum of ${limit}, preserving │   │     │  │
│  │  │  │  the most essential information.          │   │     │  │
│  │  │  │  In particular, try to preserve links..." │   │     │  │
│  │  │  │                                           │   │     │  │
│  │  │  │ DATA SENT TO LLM:                         │   │     │  │
│  │  │  │ • Chunk markdown content                  │   │     │  │
│  │  │  │ • Character limit                         │   │     │  │
│  │  │  └────────────────────────────────────────────┘   │     │  │
│  │  │                  │                                 │     │  │
│  │  │                  ▼                                 │     │  │
│  │  │  Step 8: Combine Summaries                        │     │  │
│  │  │  ┌──────────────────────────────────┐             │     │  │
│  │  │  │ Join with headers:               │             │     │  │
│  │  │  │ "=== SUMMARIZED CHUNK ===\n"     │             │     │  │
│  │  │  │ summary1                         │             │     │  │
│  │  │  │ "\n=== SUMMARIZED CHUNK ===\n"   │             │     │  │
│  │  │  │ summary2                         │             │     │  │
│  │  │  │ ...                              │             │     │  │
│  │  │  └──────────────────────────────────┘             │     │  │
│  │  │                  │                                 │     │  │
│  │  │                  ▼                                 │     │  │
│  │  │  Step 9: Cleanup                                  │     │  │
│  │  │  ┌──────────────────────────────────┐             │     │  │
│  │  │  │ await browser.close()            │             │     │  │
│  │  │  └──────────────────────────────────┘             │     │  │
│  │  │                  │                                 │     │  │
│  │  │                  ▼                                 │     │  │
│  │  │  Return: Combined summary markdown                │     │  │
│  │  │                                                    │     │  │
│  │  └────────────────────────────────────────────────────┘     │  │
│  │                  │                                           │  │
│  │                  ▼                                           │  │
│  │  Send Summary to Agent via Message Bus                      │  │
│  │  "Here is the content from ${url}:\n\n${summary}"           │  │
│  │                                                              │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Prompts Used

**Web Page Summarization Prompt** (`src/module/definitions/web.ts:192-196`)
- Condenses markdown chunks to fit character limit
- Preserves essential information and links
- Maintains original voice (not meta-descriptive)
- See: [PROMPTS_DOCUMENTATION.md - Section 5](#)

### Data Transformations

1. **URL → HTML**
   - Puppeteer fetches page content
   - Full DOM with scripts and styles

2. **HTML → Markdown**
   - TurndownService converts HTML to markdown
   - Removes style, script, img tags
   - Preserves links and structure

3. **Markdown → Chunks**
   - Split by lines
   - Each chunk ≤ 90% of max completion tokens
   - Line-based to preserve formatting

4. **Chunks → Summaries**
   - Parallel LLM calls (one per chunk)
   - Each summary fits character limit
   - Preserves links and essential info

5. **Summaries → Combined Result**
   - Join with "=== SUMMARIZED CHUNK ===" headers
   - Include token counts
   - Send to agent

### Performance Characteristics

- **Browser Launch:** ~1-2 seconds (Puppeteer initialization)
- **Page Load:** Varies by page size and network
- **Conversion:** Fast (TurndownService)
- **Summarization:** Parallel LLM calls for chunks
- **Total Time:** Dominated by page load + LLM calls

### Token Budget Example

**Scenario:** 4000 token context window, reading a large article

1. Max completion tokens: 4000 / 4 = 1000 tokens
2. Max chunk tokens: 1000 * 0.9 = 900 tokens
3. Article has 3600 tokens → 4 chunks of 900 tokens each
4. Max summary tokens: 1000 tokens total
5. Per-chunk summary: 1000 / 4 = 250 tokens
6. Character limit: 250 * 4 = 1000 characters per chunk

Result: Agent receives 4 summarized chunks, total ≤ 1000 tokens

---

## Module Initialization Pipeline

**Purpose:** Loads and integrates modules with actions, state, and pinned messages at startup.

**Entry Points:**
- `src/main.ts:35` - ModuleManager creation
- `src/module/define-module.ts:9` - Module definition

### Pipeline Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                MODULE INITIALIZATION PIPELINE                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────────────────────────────────────────────┐          │
│  │  STEP 1: DEFINE MODULES                              │          │
│  │  (src/module/definitions/*.ts)                       │          │
│  ├──────────────────────────────────────────────────────┤          │
│  │                                                      │          │
│  │  export default defineModule({                      │          │
│  │    name: "moduleName",                              │          │
│  │    createState: ({ agentId }) => new Store()        │          │
│  │  }).with({                                          │          │
│  │    pinnedMessage: async ({ state }) => "...",       │          │
│  │    actions: {                                       │          │
│  │      actionName: {                                  │          │
│  │        description: "...",                          │          │
│  │        parameters: { ... },                         │          │
│  │        execute: async ({ parameters, context }) => {}│          │
│  │      }                                              │          │
│  │    }                                                │          │
│  │  })                                                 │          │
│  │                                                      │          │
│  └──────────────────────────────────────────────────────┘          │
│                       │                                             │
│                       ▼                                             │
│  ┌──────────────────────────────────────────────────────┐          │
│  │  STEP 2: LOAD BUILT-IN MODULES                       │          │
│  │  (src/main.ts:35-42)                                 │          │
│  ├──────────────────────────────────────────────────────┤          │
│  │                                                      │          │
│  │  const moduleDefinitions = [                        │          │
│  │    core,        // Basic actions (noop, help)       │          │
│  │    goals,       // Goal management                  │          │
│  │    notes,       // Note taking                      │          │
│  │    messaging,   // Inter-agent comms                │          │
│  │    system,      // System actions                   │          │
│  │    web          // Web search & reading             │          │
│  │  ]                                                  │          │
│  │                                                      │          │
│  └──────────────────────────────────────────────────────┘          │
│                       │                                             │
│                       ▼                                             │
│  ┌──────────────────────────────────────────────────────┐          │
│  │  STEP 3: CREATE MODULE MANAGER                       │          │
│  │  (src/module-manager.ts:10-32)                       │          │
│  ├──────────────────────────────────────────────────────┤          │
│  │                                                      │          │
│  │  new ModuleManager(agentId, allAgentIds, modules)   │          │
│  │                  │                                   │          │
│  │                  ▼                                   │          │
│  │  Substep 3a: Create Module Instances                │          │
│  │  ┌────────────────────────────────────────┐         │          │
│  │  │ this.modules = moduleDefinitions.map( │         │          │
│  │  │   def => new ModuleInstance(this, def)│         │          │
│  │  │ )                                      │         │          │
│  │  │                                        │         │          │
│  │  │ Each ModuleInstance wraps:            │         │          │
│  │  │ • Module definition                   │         │          │
│  │  │ • Lazy state initialization           │         │          │
│  │  │ • Context getter                      │         │          │
│  │  └────────────────────────────────────────┘         │          │
│  │                  │                                   │          │
│  │                  ▼                                   │          │
│  │  Substep 3b: Build Action Dictionary                │          │
│  │  ┌────────────────────────────────────────┐         │          │
│  │  │ this.actions = new Map()               │         │          │
│  │  │                                        │         │          │
│  │  │ For each module:                       │         │          │
│  │  │   For each action:                     │         │          │
│  │  │     actions.set(actionName, actionDef) │         │          │
│  │  │                                        │         │          │
│  │  │ Result: Flat map of all actions        │         │          │
│  │  └────────────────────────────────────────┘         │          │
│  │                  │                                   │          │
│  │                  ▼                                   │          │
│  │  Substep 3c: Build Action-to-Module Map             │          │
│  │  ┌────────────────────────────────────────┐         │          │
│  │  │ this.actionToModuleMap = new Map()     │         │          │
│  │  │                                        │         │          │
│  │  │ For each module:                       │         │          │
│  │  │   For each action:                     │         │          │
│  │  │     map.set(actionName, moduleInstance)│         │          │
│  │  │                                        │         │          │
│  │  │ Used for context lookup during exec    │         │          │
│  │  └────────────────────────────────────────┘         │          │
│  │                                                      │          │
│  └──────────────────────────────────────────────────────┘          │
│                       │                                             │
│                       ▼                                             │
│  ┌──────────────────────────────────────────────────────┐          │
│  │  STEP 4: MEMORY INITIALIZATION                       │          │
│  │  (src/memory/memory.ts:55-74)                        │          │
│  ├──────────────────────────────────────────────────────┤          │
│  │                                                      │          │
│  │  getIntroduction() {                                │          │
│  │    // Collect pinned messages from all modules      │          │
│  │    const pinnedMessages = await Promise.all(        │          │
│  │      modules.map(m => m.pinnedMessage(context))     │          │
│  │    )                                                │          │
│  │                                                      │          │
│  │    // Format with headers                           │          │
│  │    const formatted = pinnedMessages                 │          │
│  │      .filter(msg => msg)                            │          │
│  │      .map((msg, i) =>                               │          │
│  │        `--- ${moduleName[i].toUpperCase()} ---\n\n` │          │
│  │        + msg                                        │          │
│  │      )                                              │          │
│  │      .join("\n\n")                                  │          │
│  │                                                      │          │
│  │    return {                                         │          │
│  │      type: "message",                               │          │
│  │      message: {                                     │          │
│  │        type: "spontaneous",                         │          │
│  │        content: formatted                           │          │
│  │      }                                              │          │
│  │    }                                                │          │
│  │  }                                                  │          │
│  │                                                      │          │
│  └──────────────────────────────────────────────────────┘          │
│                       │                                             │
│                       ▼                                             │
│  ┌──────────────────────────────────────────────────────┐          │
│  │  RESULT: Initialized Agent System                    │          │
│  ├──────────────────────────────────────────────────────┤          │
│  │                                                      │          │
│  │  • All modules loaded                               │          │
│  │  • All actions registered                           │          │
│  │  • Module states initialized (lazy)                 │          │
│  │  • Pinned messages available                        │          │
│  │  • Action execution ready                           │          │
│  │                                                      │          │
│  │  Agent can now:                                     │          │
│  │  • Receive messages                                 │          │
│  │  • Make decisions                                   │          │
│  │  • Execute actions                                  │          │
│  │  • Manage state                                     │          │
│  │                                                      │          │
│  └──────────────────────────────────────────────────────┘          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Built-in Modules

**Load Order (src/main.ts:35-42):**

1. **core** - Foundational actions
   - Actions: `noop`, `help`, `currentObjective`
   - Pinned: Core system prompt with action list
   - State: None

2. **goals** - Goal management
   - Actions: `addGoal`, `completeGoal`
   - Pinned: Current goals list (numbered, with status)
   - State: JsonStore (array of Goal objects)

3. **notes** - Note taking
   - Actions: `writeNote`, `viewNote`, `deleteNote`
   - Pinned: Current note titles (bulleted list)
   - State: FileStore (one file per note)

4. **messaging** - Inter-agent communication
   - Actions: `sendMessage`, `queryAgentRegistry`
   - Pinned: None
   - State: None (uses message bus)

5. **system** - System operations
   - Actions: `executeCommand`, `readFile`, `writeFile`, `listDirectory`, etc.
   - Pinned: None
   - State: None

6. **web** - Web access
   - Actions: `searchWeb`, `readPage`
   - Pinned: None
   - State: None

### Module State Examples

**Goals Module State:**
```typescript
// File: .ai-legion/agents/{agentId}/goals.json
{
  "goals": [
    { "text": "Complete documentation", "complete": false },
    { "text": "Test all pipelines", "complete": true }
  ]
}
```

**Notes Module State:**
```
# Files in: .ai-legion/agents/{agentId}/notes/
important-info.txt
research-findings.txt
contact-list.txt
```

### Module Context

When an action executes, it receives a context object:

```typescript
interface ModuleContext {
  agentId: string;              // Current agent's ID
  allAgentIds: string[];        // All agent IDs in system
  actionDictionary: Map<...>;   // All available actions
  state: Store;                 // Module-specific state
}
```

This context allows actions to:
- Access and modify their module's state
- Query other agents
- Discover available actions
- Identify themselves

### Lazy State Initialization

Module state is created on first access:

```typescript
// src/module-instance.ts:12-20
get state() {
  if (!this._state && this.moduleDef.createState) {
    this._state = this.moduleDef.createState({
      agentId: this.moduleManager.agentId
    });
  }
  return this._state;
}
```

**Benefits:**
- Modules without state don't create unnecessary storage
- State creation can depend on runtime config
- Delayed initialization improves startup time

### Performance Characteristics

- **Module Loading:** O(n) where n = number of modules
- **Action Registration:** O(m) where m = total number of actions across all modules
- **Startup Time:** Fast (lazy state, no precomputation)
- **Memory:** Minimal until state is accessed

---

## Summary: Complete Agent Workflow

Here's how all pipelines work together in a typical agent operation:

```
USER INPUT
    │
    ▼
MESSAGE BUS ──────────────────┐
    │                         │
    ▼                         ▼
AGENT SUBSCRIPTION      OTHER AGENTS
    │                    (filtered out)
    ▼
MEMORY APPEND
    ├─> Error cleanup (if "ok" message)
    ├─> Append event
    ├─> Summarize (if needed) ────> MEMORY MANAGEMENT PIPELINE
    │                                ├─> Token counting
    │                                ├─> LLM summarization
    │                                └─> Summary creation
    └─> Persist to store
    │
    ▼
MAIN EXECUTION LOOP (1000ms cycle)
    ├─> Retrieve events ─────────> MEMORY RETRIEVE PIPELINE
    │                               ├─> Get introduction
    │                               │   └─> Collect pinned messages
    │                               └─> Load stored events
    ├─> Check if action needed
    ├─> Make decision (LLM call)
    │   ├─> PROMPTS: Core, Goals, Notes
    │   └─> Returns action text
    ├─> Append decision to memory
    ├─> Parse action ────────────> ACTION PARSING PIPELINE
    │   ├─> Validate format
    │   ├─> Transform to JSON
    │   ├─> Validate action exists
    │   ├─> Validate parameters
    │   └─> Return Action object or error
    │
    ├─> If error: Send error message
    │              └─────────────> MESSAGE BUS
    │
    └─> If success: Execute action ──> ACTION EXECUTION PIPELINE
        ├─> Lookup module context
        ├─> Execute action function
        │   ├─> Read/write module state
        │   ├─> Call external APIs
        │   │   └─> (e.g., WEB CONTENT PIPELINE)
        │   └─> Send result messages ──> MESSAGE BUS
        └─> Complete
            │
            └─────────────────────> LOOP BACK TO START
```

### Key Integration Points

1. **Memory ↔ Main Loop**
   - Memory provides events for decision-making
   - Main loop appends decisions back to memory
   - Summarization triggered automatically

2. **Message Bus ↔ Memory**
   - Message bus delivers messages to agents
   - Agent appends messages to memory
   - Memory may trigger error cleanup

3. **Actions ↔ Modules**
   - Actions execute within module context
   - Actions access module state
   - Module state persists across executions

4. **LLM ↔ Prompts**
   - Core system prompt (always)
   - Pinned messages (goals, notes)
   - Special prompts (summarization, web content)

5. **Web Pipeline ↔ Actions**
   - Invoked by `readPage` action
   - Returns summarized content
   - Content sent to agent via message bus

This architecture enables autonomous, stateful, multi-agent systems with managed memory and modular capabilities.

