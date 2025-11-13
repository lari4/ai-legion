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

