# Agentic Chat Compaction Flow

## Overview

The language-servers codebase implements **two distinct compaction mechanisms** for agentic chat:

1. **In-Session Compaction (Context Window Management)** — Summarizes conversation history when the LLM context window is nearing its limit, keeping the conversation coherent.
2. **Disk-Level History Trimming (Storage Management)** — Trims the oldest messages across all workspaces when total disk storage exceeds 200MB.

---

## Key Files

| File                           | Role                                                                                             |
| ------------------------------ | ------------------------------------------------------------------------------------------------ |
| `agenticChatController.ts`     | Orchestrates compaction triggers, agent loop, and LLM calls                                      |
| `chatDb.ts`                    | Database operations: `fixAndGetHistory`, `replaceWithSummary`, `calculateMessagesCharacterCount` |
| `chatHistoryMaintainer.ts`     | Disk-level trimming of old messages across all workspaces                                        |
| `util.ts`                      | Data types (`Tab`, `Conversation`, `Message`), priority queue, adapters                          |
| `tokenLimitsCalculator.ts`     | Calculates dynamic thresholds based on model's `maxInputTokens`                                  |
| `agenticChatTriggerContext.ts` | Builds the `COMPACTION_PROMPT` request to send to LLM                                            |
| `constants/constants.ts`       | `COMPACTION_PROMPT` (summarization system prompt), `COMPACTION_BODY`                             |

---

## Token Limits (Default: 200K input tokens)

| Parameter              | Formula                          | Default Value |
| ---------------------- | -------------------------------- | ------------- |
| `maxInputTokens`       | From API or 200,000              | 200,000       |
| `maxOverallCharacters` | `maxInputTokens × 3.5`           | 700,000       |
| `inputLimit`           | `maxOverallCharacters − 100,000` | 600,000       |
| `compactionThreshold`  | `0.7 × maxOverallCharacters`     | 490,000       |

---

## Flow 1: In-Session Compaction (Context Window Management)

### Trigger Paths

There are **two ways** compaction is triggered:

-   **Manual**: User types `/compact` command
-   **Nudge (Auto)**: After the agent loop ends, if `currentRequestCount > compactionThreshold`

```mermaid
flowchart TD
    A[User sends message / Agent loop starts] --> B{Is command /compact?}

    B -->|Yes: Manual Compaction| C[Build compaction request input<br/><i>getCompactionChatCommandInput</i>]
    C --> D[Run Compaction<br/><i>#runCompaction</i>]

    B -->|No: Normal message| E[Run Agent Loop<br/><i>#runAgentLoop</i>]
    E --> F[Agent loop iterations:<br/>LLM calls ↔ Tool executions]
    F --> G[Agent loop completes<br/>Calculate currentRequestCount]
    G --> H{currentRequestCount ><br/>compactionThreshold?<br/><i>70% of maxOverallCharacters</i>}

    H -->|No| I[Return final result to user]

    H -->|Yes: Nudge Compaction| J[Show compaction confirmation UI<br/><i>#processCompactConfirmation</i>]
    J --> K[Display: Context window is X% full<br/>Compact chat history?<br/>Button: Allow]
    K --> L{User accepts?}

    L -->|No / Ignored| M[Skip compaction<br/>Return final result]

    L -->|Yes| N[Build compaction request input<br/><i>getCompactionChatCommandInput</i>]
    N --> D

    D --> O[Load history from ChatDB<br/><i>fixAndGetHistory</i>]
    O --> P{History empty?}

    P -->|Yes| Q[Show: Nothing to compact<br/>Return early]

    P -->|No| R[Show: Compacting your chat history...<br/>Send COMPACTION_PROMPT + history to LLM]
    R --> S[LLM generates summary<br/>Stream response]
    S --> T{Summary received?}

    T -->|No| U[Log warning<br/>Skip history replacement]

    T -->|Yes| V[Replace history with summary<br/><i>chatDb.replaceWithSummary</i>]
    V --> W[Create new historyId<br/>Insert summary as prompt message<br/>+ dummy answer: Working...<br/>Delete old history entry]
    W --> X[Show: History compacted successfully!<br/>Return result]
```

### replaceWithSummary Detail

```mermaid
flowchart TD
    A[replaceWithSummary called] --> B[Get or create historyId for tabId]
    B --> C[Create new historyId<br/><i>crypto.randomUUID</i>]
    C --> D[Insert new tab entry with:<br/>- New historyId<br/>- Summary as prompt message<br/>- Dummy answer: Working...]
    D --> E[Delete old tab entry<br/><i>findAndRemove old historyId</i>]
    E --> F[History ID mapping updated<br/>Old conversation erased]
```

### COMPACTION_PROMPT Summary

The `COMPACTION_PROMPT` instructs the LLM to generate a summary with these sections:

1. **Conversation Summary** — Key topics discussed
2. **Files and Code Summary** — File paths, function signatures, key changes
3. **Key Insights** — User preferences, technical details, decisions
4. **Most Recent Topic** — Detailed summary of latest topic + all tools executed

---

## Flow 2: Disk-Level History Trimming (Storage Management)

This runs **asynchronously on ChatDatabase initialization** to prevent unbounded disk growth.

### Thresholds

| Parameter                   | Value               |
| --------------------------- | ------------------- |
| `maxHistorySizeInBytes`     | 200 MB              |
| `targetHistorySizeInBytes`  | 150 MB (75% of max) |
| `batchDeleteIterations`     | 200                 |
| `messagePairPerBatchDelete` | 5                   |
| `maxTrimIterations`         | 100                 |

```mermaid
flowchart TD
    A[ChatDatabase constructor] --> B[Create ChatHistoryMaintainer]
    B --> C[Async: trimHistoryToMaxSize]

    C --> D[Calculate total size of all<br/>chat-history-*.json files]
    D --> E{totalSize > 200MB?}

    E -->|No| F[No trimming needed<br/>Return]

    E -->|Yes| G[trimHistoryForAllWorkspaces]
    G --> H[List all chat-history-*.json files<br/>in ~/.aws/amazonq/history/]
    H --> I[Load all LokiJS databases]
    I --> J[Build Priority Queue of tabs<br/>sorted by oldest message timestamp]

    J --> K[runHistoryTrimmingLoop]
    K --> L[Calculate total size of all DBs]
    L --> M{totalSize ≤ 150MB?}

    M -->|Yes| N[Trimming complete<br/>Close all non-current DBs]

    M -->|No| O{iterationCount ><br/>maxTrimIterations?}
    O -->|Yes| P[Safety exit<br/>Log warning, close DBs]

    O -->|No| Q[batchDeleteMessagePairs]
    Q --> R[Dequeue tab with oldest messages<br/>from priority queue]
    R --> S[Remove oldest message pairs<br/><i>up to messagePairPerBatchDelete</i>]
    S --> T{Tab has messages left?}

    T -->|Yes| U[Update tab in collection<br/>Re-enqueue with new oldest date]
    T -->|No| V[Remove entire tab from collection]

    U --> W{More iterations in batch?}
    V --> W
    W -->|Yes| R
    W -->|No| X[Save updated DBs to disk]
    X --> L

    N --> Y[Done]
```

### Message Pair Removal Detail

```mermaid
flowchart TD
    A[removeOldestMessagePairFromTab] --> B{Tab has conversations?}
    B -->|No| C[Return false - nothing to remove]
    B -->|Yes| D[Get first conversation<br/><i>oldest chronologically</i>]
    D --> E{Conversation has > 2 messages?}
    E -->|Yes| F[Remove first 2 messages<br/><i>splice 0, 2 - prompt+answer pair</i>]
    E -->|No| G[Remove entire conversation<br/><i>splice 0, 1</i>]
    F --> H[Return true]
    G --> H
```

---

## Flow 3: Request Truncation (Pre-Send Budget Management)

Before each LLM request, the controller truncates the request to fit within the `inputLimit`.

```mermaid
flowchart TD
    A[truncateRequest called<br/>before each LLM request] --> B[Calculate remainingCharacterBudget<br/>= inputLimit]
    B --> C[1. Fit user input message<br/>Truncate if > inputLimit]
    C --> D[2. Fit @context docs + images<br/>Add items until budget exhausted]
    D --> E[3. Fit current file context<br/>Include if budget allows]
    E --> F[4. Fit pinned context<br/>Include items until budget exhausted]
    F --> G[Return remaining budget<br/>for chat history]
```

---

## Flow 4: History Validation (fixAndGetHistory)

Before each LLM request, history is loaded and validated.

```mermaid
flowchart TD
    A[fixAndGetHistory called] --> B[Get all messages for tabId]
    B --> C[ensureValidMessageSequence]
    C --> D[First message must be user prompt<br/>Drop leading assistant messages]
    D --> E[First user message must NOT<br/>have tool results - drop pairs if so]
    E --> F[Last message must be assistant answer<br/>Add dummy answer if needed]
    F --> G[validateAndFixNewMessageToolResults]
    G --> H[Ensure tool results match<br/>previous assistant's tool uses]
    H --> I[Add cancelled status for<br/>missing tool results]
    I --> J[Prepend pinned context<br/>as fake message pair]
    J --> K[Return history + character counts]
```

---

## Complete End-to-End Flow

```mermaid
flowchart TD
    subgraph Initialization
        INIT[ChatDatabase Constructor] --> TRIM[Async: Disk Trimming<br/><i>trimHistoryToMaxSize</i>]
        TRIM --> TRIM_CHECK{Total DB files > 200MB?}
        TRIM_CHECK -->|Yes| TRIM_EXEC[Trim oldest messages<br/>across all workspaces<br/>Target: 150MB]
        TRIM_CHECK -->|No| TRIM_DONE[No action needed]
    end

    subgraph "User Message Flow"
        USER[User sends message] --> CMD{/compact command?}

        CMD -->|Yes: Manual| COMPACT_DIRECT[Run Compaction directly]

        CMD -->|No| AGENT[Run Agent Loop]
        AGENT --> TRUNC[Truncate request to inputLimit<br/><i>truncateRequest</i>]
        TRUNC --> FIX[Fix & get history<br/><i>fixAndGetHistory</i>]
        FIX --> CALC[Calculate currentRequestCount<br/>= historyCharCount + currentInputCount]
        CALC --> LLM[Send to LLM<br/>Process response + tool uses]
        LLM --> LOOP{More tool uses?}
        LOOP -->|Yes| TRUNC
        LOOP -->|No| THRESHOLD{currentRequestCount ><br/>compactionThreshold?<br/><i>70% of maxOverallChars</i>}

        THRESHOLD -->|No| DONE[Return final result]

        THRESHOLD -->|Yes: Nudge| NUDGE[Show compaction UI to user]
        NUDGE --> APPROVE{User approves?}
        APPROVE -->|No| DONE
        APPROVE -->|Yes| COMPACT_DIRECT
    end

    subgraph "Compaction Execution"
        COMPACT_DIRECT --> LOAD[Load history from DB<br/><i>fixAndGetHistory</i>]
        LOAD --> EMPTY{History empty?}
        EMPTY -->|Yes| SKIP[Nothing to compact]
        EMPTY -->|No| SEND[Send COMPACTION_PROMPT<br/>+ full history to LLM]
        SEND --> SUMMARY[LLM returns summary]
        SUMMARY --> REPLACE[replaceWithSummary:<br/>1. Create new historyId<br/>2. Insert summary + dummy answer<br/>3. Delete old history]
        REPLACE --> COMPACT_DONE[Compaction complete!]
    end
```

---

## Summary

| Mechanism              | Trigger                                                          | What it does                                                          | Where                                                            |
| ---------------------- | ---------------------------------------------------------------- | --------------------------------------------------------------------- | ---------------------------------------------------------------- |
| **Manual Compaction**  | User types `/compact`                                            | Sends history to LLM for summarization, replaces history with summary | `agenticChatController.ts` → `#runCompaction`                    |
| **Nudge Compaction**   | `currentRequestCount > 70% maxOverallChars` at end of agent loop | Shows confirmation UI, then same as manual                            | `agenticChatController.ts` → `#shouldCompact` → `#runCompaction` |
| **Disk Trimming**      | On `ChatDatabase` init, total DB files > 200MB                   | Deletes oldest message pairs across all workspaces until < 150MB      | `chatHistoryMaintainer.ts` → `trimHistoryToMaxSize`              |
| **Request Truncation** | Before every LLM request                                         | Truncates user input, context, and pinned context to fit `inputLimit` | `agenticChatController.ts` → `truncateRequest`                   |
| **History Validation** | Before every LLM request                                         | Ensures valid message sequence, fixes tool result mismatches          | `chatDb.ts` → `fixAndGetHistory`                                 |
