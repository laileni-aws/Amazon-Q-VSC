# Memory Bank Implementation - Amazon Q Developer Language Servers

## Overview

Memory Bank is a feature in Amazon Q Developer that automatically generates and maintains a set of project documentation files by analyzing the user's codebase. These generated documents are stored as **rules** inside the workspace and are automatically included in every chat conversation, giving the LLM persistent, structured context about the project's purpose, structure, technologies, and coding guidelines.

The feature lives entirely in the **language-servers** repository, specifically within the agentic chat subsystem of `aws-lsp-codewhisperer`.

---

## Key Concepts

| Concept                 | Description                                                       |
| ----------------------- | ----------------------------------------------------------------- |
| **Memory Bank**         | A set of 4 auto-generated `.md` files that describe a project     |
| **Storage Location**    | `.amazonq/rules/memory-bank/` inside the user's workspace         |
| **Generated Files**     | `product.md`, `structure.md`, `tech.md`, `guidelines.md`          |
| **Access Pattern**      | Loaded as **workspace rules** and injected into every chat prompt |
| **Trigger**             | User types a natural language phrase like "create a memory bank"  |
| **Re-trigger / Update** | User types "regenerate memory bank" (same trigger phrases)        |

---

## Architecture Components

### Source Files

All Memory Bank code resides in the language-servers repository:

```
language-servers/
└── server/aws-lsp-codewhisperer/src/language-server/agenticChat/
    ├── agenticChatController.ts          # Orchestrates the memory bank creation flow
    ├── constants/
    │   └── constants.ts                  # Memory bank constants (file limits, ranking count)
    └── context/
        ├── additionalContextProvider.ts  # Injects memory bank files into chat prompts
        └── memorybank/
            ├── memoryBankController.ts   # Core logic: detection, analysis, prompt building
            ├── memoryBankPrompts.ts      # LLM prompt templates for file ranking & generation
            └── memoryBankController.test.ts  # Unit tests
```

### Constants

Defined in `constants/constants.ts`:

| Constant                                      | Value         | Purpose                                                      |
| --------------------------------------------- | ------------- | ------------------------------------------------------------ |
| `FSREAD_MEMORY_BANK_MAX_PER_FILE`             | 20,000 chars  | Max characters per file read during memory bank generation   |
| `FSREAD_MEMORY_BANK_MAX_TOTAL`                | 100,000 chars | Max total characters read across all files during generation |
| `MAX_NUMBER_OF_FILES_FOR_MEMORY_BANK_RANKING` | 5             | Number of top files selected for deep analysis               |

### The 4 Generated Documents

Stored at `.amazonq/rules/memory-bank/` relative to the workspace root:

| File            | Purpose              | Content                                                                  |
| --------------- | -------------------- | ------------------------------------------------------------------------ |
| `product.md`    | Project overview     | Purpose, value proposition, key features, target users                   |
| `structure.md`  | Project organization | Directory structure, core components, architectural patterns             |
| `tech.md`       | Technology details   | Languages, versions, build systems, dependencies, dev commands           |
| `guidelines.md` | Development patterns | Code quality standards, naming conventions, design patterns, code idioms |

---

## Flow Diagrams

### High-Level: End-to-End Memory Bank Lifecycle

```mermaid
flowchart TD
    Start(["User types 'create a memory bank'"])
    Detect{"agenticChatController detects trigger phrase"}
    SetFlag["Set session.isMemoryBankGeneration = true"]
    CheckExist{"Memory bank already exists?"}
    Clean["Clean existing files from memory-bank dir"]
    CreateDir["Create .amazonq/rules/memory-bank/ directory"]
    Analysis["Run TF-IDF Analysis Pipeline"]
    LLMRank["LLM Call 1: Rank files by importance"]
    BuildPrompt["Build comprehensive generation prompt"]
    ReplaceUserPrompt["Replace user prompt with generation prompt"]
    AgenticLoop["Enter normal agentic chat loop - LLM uses fsWrite to create 4 files"]
    FilesCreated["4 md files created in memory-bank dir"]
    ResetFlag["Reset session.isMemoryBankGeneration = false"]
    Done(["Memory Bank available as workspace rules"])

    Start --> Detect
    Detect --> SetFlag
    SetFlag --> CheckExist
    CheckExist -->|Yes| Clean
    CheckExist -->|No| CreateDir
    Clean --> Analysis
    CreateDir --> Analysis
    Analysis --> LLMRank
    LLMRank --> BuildPrompt
    BuildPrompt --> ReplaceUserPrompt
    ReplaceUserPrompt --> AgenticLoop
    AgenticLoop --> FilesCreated
    FilesCreated --> ResetFlag
    ResetFlag --> Done

    style Start fill:#e1f5fe
    style Done fill:#e8f5e9
    style AgenticLoop fill:#fff3e0
    style LLMRank fill:#fce4ec
```

### Detailed: Memory Bank Creation Flow in agenticChatController

```mermaid
sequenceDiagram
    participant User
    participant Controller as agenticChatController
    participant MBCtrl as MemoryBankController
    participant Stream as ResultStream
    participant LLM as LLM Service
    participant FS as Filesystem

    User->>Controller: create a memory bank
    Controller->>MBCtrl: isMemoryBankCreationRequest
    MBCtrl-->>Controller: true

    Note over Controller: session.isMemoryBankGeneration = true
    Note over Controller: Store originalPrompt as backup

    Controller->>MBCtrl: memoryBankExists
    MBCtrl->>FS: Check memory-bank dir exists
    FS-->>MBCtrl: true or false
    MBCtrl-->>Controller: exists boolean

    Controller->>Stream: Preparing to analyze your project

    Controller->>MBCtrl: prepareComprehensiveMemoryBankPrompt

    Note over MBCtrl: Step 1 - Clean directory
    MBCtrl->>FS: Remove existing md files
    MBCtrl->>FS: mkdir memory-bank recursive

    Note over MBCtrl: Step 2 - TF-IDF Analysis Pipeline
    MBCtrl->>FS: Discover all source files
    MBCtrl->>FS: Read file contents in batches
    Note over MBCtrl: Calculate TF-IDF vectors and cosine similarity

    Note over MBCtrl: Step 3 - LLM File Ranking
    MBCtrl->>Controller: llmCallFn with ranking prompt
    Controller->>LLM: SendMessage with ranking prompt
    LLM-->>Controller: JSON array of top 5 file paths
    Controller-->>MBCtrl: ranked files response

    Note over MBCtrl: Step 4 - Parse ranked files with TF-IDF fallback

    Note over MBCtrl: Step 5 - Build final generation prompt
    MBCtrl-->>Controller: comprehensivePrompt

    Note over Controller: Replace prompt with comprehensivePrompt

    Note over Controller: Normal agentic loop begins
    Controller->>LLM: Send comprehensive prompt
    LLM->>Controller: Tool use readFile
    Note over Controller: Memory bank mode uses higher fsRead limits
    LLM->>Controller: Tool use fsWrite product.md
    Controller->>FS: Write product.md
    LLM->>Controller: Tool use fsWrite structure.md
    Controller->>FS: Write structure.md
    LLM->>Controller: Tool use fsWrite tech.md
    Controller->>FS: Write tech.md
    LLM->>Controller: Tool use fsWrite guidelines.md
    Controller->>FS: Write guidelines.md

    Note over Controller: session.isMemoryBankGeneration = false
    Controller-->>User: Memory Bank created successfully
```

### Detailed: TF-IDF Analysis Pipeline

```mermaid
flowchart TD
    Start(["Start Pipeline"])
    Discover["Discover source files with 17 extensions"]
    Filter1["Filter: Skip node_modules, .git, build, dist, etc."]
    Filter2["Filter: Remove files over 20000 lines"]
    LargeCheck{"More than 200 files?"}
    Sample["Random sample of 200 files"]
    AllFiles["Use all files"]
    TFIDF["Calculate TF-IDF Vectors"]
    Tokenize["Tokenize each file: lowercase, remove punctuation, filter short tokens"]
    BuildVocab["Build vocabulary across all files"]
    CalcTFIDF["For each file and term: compute TF-IDF score"]
    CosineSim["Calculate pairwise cosine similarity matrix"]
    Dissimilarity["Dissimilarity = 1 minus mean similarity per file"]
    SortSize["Sort files by size descending"]
    FormatString["Format: path + lines + dissimilarity score"]
    FallbackRank["Create fallback ranking: Sort by dissimilarity, take top 5"]
    Output(["Output: formattedFilesString + rankedFilesList"])

    Start --> Discover
    Discover --> Filter1
    Filter1 --> Filter2
    Filter2 --> LargeCheck
    LargeCheck -->|Yes| Sample
    LargeCheck -->|No| AllFiles
    Sample --> TFIDF
    AllFiles --> TFIDF
    TFIDF --> Tokenize
    Tokenize --> BuildVocab
    BuildVocab --> CalcTFIDF
    CalcTFIDF --> CosineSim
    CosineSim --> Dissimilarity
    Dissimilarity --> SortSize
    SortSize --> FormatString
    FormatString --> FallbackRank
    FormatString --> Output

    style Start fill:#e1f5fe
    style Output fill:#e8f5e9
    style TFIDF fill:#fff3e0
```

### How Memory Bank Files Are Accessed in Chat Prompts

```mermaid
flowchart TD
    Start(["User sends any chat message"])
    CollectRules["additionalContextProvider collectWorkspaceRules"]
    ScanFS["Scan .amazonq/rules/ directory including memory-bank/ subdirectory"]
    Found{"Memory bank files found?"}
    NoMB["No memory bank context added"]
    AutoEnable["Auto-enable memory-bank folder and files in rulesState"]
    CheckRequest{"Is this a memory bank creation request?"}
    Exclude["Exclude memory bank files from context to avoid stale data"]
    Include["Include memory bank files in pinnedContextCommands"]
    Inject["Inject as workspace rules in conversationState.history first message"]
    LLM(["LLM receives project context from memory bank in every message"])
    GenFlow(["Proceed with regeneration flow using fresh analysis"])

    Start --> CollectRules
    CollectRules --> ScanFS
    ScanFS --> Found
    Found -->|No| NoMB
    Found -->|Yes| AutoEnable
    AutoEnable --> CheckRequest
    CheckRequest -->|"Yes - Regenerating"| Exclude
    CheckRequest -->|"No - Normal chat"| Include
    Include --> Inject
    Inject --> LLM
    Exclude --> GenFlow

    style Start fill:#e1f5fe
    style LLM fill:#e8f5e9
    style GenFlow fill:#fff3e0
```

### Rules Panel: Memory Bank File Display

```mermaid
flowchart LR
    MB[".amazonq/rules/memory-bank/"]
    P["product.md - active"]
    S["structure.md - active"]
    T["tech.md - active"]
    G["guidelines.md - active"]
    RulesPanel["Rules Panel in Chat UI"]
    Toggle["User can toggle individual files on/off"]
    FolderToggle["User can toggle entire memory-bank folder"]

    MB --> P
    MB --> S
    MB --> T
    MB --> G
    P --> RulesPanel
    S --> RulesPanel
    T --> RulesPanel
    G --> RulesPanel
    RulesPanel --> Toggle
    RulesPanel --> FolderToggle

    style MB fill:#e3f2fd
    style RulesPanel fill:#f3e5f5
```

---

## Prompt Architecture

Memory Bank uses **two LLM prompts**, defined in `memoryBankPrompts.ts`:

### Prompt 1: File Ranking (`getFileRankingPrompt`)

**Purpose:** Ask the LLM to select the top N most important/representative files from the TF-IDF analysis results.

**Input:** A formatted string of all discovered files with their line counts and lexical dissimilarity scores.

**Output:** A JSON array of file paths (e.g., `["src/main.ts", "src/core/engine.ts", ...]`)

**Fallback:** If LLM ranking fails to parse, the system falls back to the deterministic TF-IDF dissimilarity ranking.

```
Location: memoryBankPrompts.ts → getFileRankingPrompt()
Called from: memoryBankController.ts → prepareComprehensiveMemoryBankPrompt()
```

### Prompt 2: Complete Memory Bank Generation (`getCompleteMemoryBankPrompt`)

**Purpose:** Instruct the LLM to explore the codebase and create the 4 memory bank files using `fsWrite` tool.

**Input:** The ranked file list and the normalized workspace root path.

**Key Instructions in the Prompt:**

-   Always regenerate fresh (never skip if files exist)
-   Fresh exploration policy: ignore previous chat history
-   Create files in order: `product.md` → `structure.md` → `tech.md` → `guidelines.md`
-   Use `fsWrite` tool with `command: "create"` for file creation
-   For `guidelines.md`: iteratively read ranked files in chunks of 2, analyze patterns
-   Keep completion summary brief (max 8 lines)

```
Location: memoryBankPrompts.ts → getCompleteMemoryBankPrompt()
Called from: memoryBankController.ts → prepareComprehensiveMemoryBankPrompt()
Injected into: agenticChatController.ts → replaces params.prompt.prompt
```

---

## Trigger Detection

The `MemoryBankController.isMemoryBankCreationRequest()` method checks the user's prompt against a list of trigger phrases:

| Trigger Phrase           | Creates New | Updates Existing |
| ------------------------ | :---------: | :--------------: |
| `create a memory bank`   |     ✅      |        ✅        |
| `create memory bank`     |     ✅      |        ✅        |
| `generate a memory bank` |     ✅      |        ✅        |
| `generate memory bank`   |     ✅      |        ✅        |
| `regenerate memory bank` |     ✅      |        ✅        |
| `build memory bank`      |     ✅      |        ✅        |
| `make memory bank`       |     ✅      |        ✅        |
| `setup memory bank`      |     ✅      |        ✅        |

The detection is case-insensitive and uses `.includes()` matching, so phrases like "Please create a memory bank for my project" will also trigger it.

---

## How to Re-trigger / Update Memory Bank

When a user makes code changes and wants to update their memory bank documentation:

```mermaid
flowchart TD
    CodeChange(["User modifies code"])
    Trigger["User types: regenerate memory bank"]
    Detect["Controller detects trigger phrase"]
    CheckExist{"Existing memory bank files found?"}
    LogRegen["Log: Regenerating Memory Bank"]
    LogGen["Log: Generating Memory Bank"]
    Clean["Delete existing 4 files: product, structure, tech, guidelines"]
    CreateDir["Create directory"]
    ExcludeOld["Exclude old memory bank files from chat context"]
    FreshAnalysis["Run fresh TF-IDF analysis on current codebase"]
    NewRanking["New LLM file ranking"]
    NewPrompt["Build new generation prompt with MANDATORY REGENERATION POLICY"]
    Generate["LLM generates fresh 4 files reflecting current code state"]
    Done(["Updated Memory Bank immediately available in chat"])

    CodeChange --> Trigger
    Trigger --> Detect
    Detect --> CheckExist
    CheckExist -->|Yes| LogRegen
    CheckExist -->|No| LogGen
    LogRegen --> Clean
    LogGen --> CreateDir
    Clean --> ExcludeOld
    CreateDir --> ExcludeOld
    ExcludeOld --> FreshAnalysis
    FreshAnalysis --> NewRanking
    NewRanking --> NewPrompt
    NewPrompt --> Generate
    Generate --> Done

    style CodeChange fill:#fff3e0
    style Done fill:#e8f5e9
```

**Key behavior during regeneration:**

1. Old memory bank files are **deleted** before generation starts
2. Old memory bank files are **excluded from chat context** (via `additionalContextProvider.ts`) so the LLM doesn't see stale documentation
3. The generation prompt explicitly instructs the LLM to **NEVER reference existing files** and to **always create fresh**
4. After regeneration, the new files are automatically detected as workspace rules and included in subsequent chats

---

## Session Flag: `isMemoryBankGeneration`

A boolean flag on the chat session object that tracks whether the current conversation turn is a memory bank generation:

| When Set                 | Value   | Effect                                                |
| ------------------------ | ------- | ----------------------------------------------------- |
| Trigger phrase detected  | `true`  | Marks session as memory bank generation               |
| `fsRead` tool invoked    | checked | Uses higher file size limits (20KB/file, 100KB total) |
| Generation completes     | `false` | Returns to normal chat behavior                       |
| Preparation fails        | `false` | Restores original prompt, proceeds normally           |
| Session ends / chat ends | `false` | Cleanup on session termination                        |

This flag ensures that during memory bank generation, the LLM can read larger files than normal to perform thorough codebase analysis.

---

## How Generated Documents Are Accessed in Prompts

Once the 4 memory bank files exist on disk, they are automatically included in every subsequent chat:

```mermaid
sequenceDiagram
    participant User
    participant ACP as AdditionalContextProvider
    participant FS as Filesystem
    participant Rules as Rules State chatDb
    participant API as GenerateAssistantResponse API

    User->>ACP: Send chat message
    ACP->>FS: collectWorkspaceRulesInternal
    Note over FS: Scan .amazonq/rules/ recursively including memory-bank/

    FS-->>ACP: All rule files including memory bank md files

    ACP->>Rules: getRules for tabId
    Note over ACP: Check if memory-bank folder and files are in rulesState

    alt First time seeing memory bank files
        ACP->>Rules: Auto-enable memory-bank folder = true
        ACP->>Rules: Auto-enable each file = true
    end

    Note over ACP: Filter rules based on user toggle preferences

    ACP->>ACP: Add memory bank files to pinnedContextCommands

    ACP->>API: Send as workspace rules in conversationState history

    Note over API: LLM now has project context from memory bank
```

### Access Path Summary

1. **Storage:** `.amazonq/rules/memory-bank/{product,structure,tech,guidelines}.md`
2. **Discovery:** `additionalContextProvider.ts` → `collectWorkspaceRulesInternal()` scans `.amazonq/rules/` recursively
3. **Auto-activation:** Memory bank files are automatically enabled when first discovered (unlike other rules which may default based on folder state)
4. **Special folder name:** Memory bank files appear under the `memory-bank` folder in the Rules Panel UI
5. **Injection:** Added to `pinnedContextCommands` → sent as the first message in `conversationState.history` in the GenerateAssistantResponse API call
6. **Toggle control:** Users can toggle individual memory bank files or the entire `memory-bank` folder on/off in the Rules Panel

---

## Error Handling & Fallbacks

| Scenario                          | Behavior                                                          |
| --------------------------------- | ----------------------------------------------------------------- |
| No workspace folder found         | Error thrown, original prompt restored                            |
| No source files discovered        | Pipeline throws error, original prompt restored                   |
| TF-IDF calculation fails          | Returns fallback dissimilarity value of 0.85 for all files        |
| LLM file ranking fails to parse   | Falls back to deterministic TF-IDF dissimilarity ranking          |
| LLM ranking returns empty         | Falls back to TF-IDF ranking                                      |
| Comprehensive prompt is empty     | Original user prompt is used instead                              |
| Overall preparation fails         | Original prompt restored, `isMemoryBankGeneration` reset to false |
| File read errors during TF-IDF    | Empty content used, continues with other files                    |
| Files over 20,000 lines           | Filtered out to prevent conversation overflow                     |
| Projects with more than 200 files | Random sample of 200 taken for analysis                           |

---

## Complete Data Flow Summary

```mermaid
flowchart TB
    subgraph Trigger["1. Trigger Detection"]
        UserPrompt["User: create a memory bank"]
        TriggerCheck["isMemoryBankCreationRequest"]
        UserPrompt --> TriggerCheck
    end

    subgraph Prep["2. Pre-processing - Deterministic"]
        CleanDir["Clean or create memory-bank dir"]
        DiscoverFiles["Discover source files - 17 extensions, skip 100+ dirs"]
        FilterFiles["Filter: under 20K lines, max 200 files"]
        TFIDF["TF-IDF Vectorization"]
        CosineSim["Cosine Similarity Matrix"]
        Dissimilarity["Lexical Dissimilarity Scores"]
        CleanDir --> DiscoverFiles
        DiscoverFiles --> FilterFiles
        FilterFiles --> TFIDF
        TFIDF --> CosineSim
        CosineSim --> Dissimilarity
    end

    subgraph Rank["3. LLM File Ranking"]
        FormatFiles["Format files with stats"]
        RankPrompt["getFileRankingPrompt"]
        LLMCall1["LLM Call 1: Select top 5 files"]
        ParseJSON["Parse JSON response"]
        RankedFiles["Ranked file list"]
        FallbackRank["TF-IDF fallback ranking"]
        FormatFiles --> RankPrompt
        RankPrompt --> LLMCall1
        LLMCall1 --> ParseJSON
        ParseJSON -->|Success| RankedFiles
        ParseJSON -->|Fail| FallbackRank
    end

    subgraph Generate["4. Memory Bank Generation"]
        GenPrompt["getCompleteMemoryBankPrompt with ranked files and workspace path"]
        ReplacePrompt["Replace user prompt"]
        AgenticLoop["Agentic Chat Loop"]
        ReadFiles["LLM reads ranked files via readFile tool"]
        WriteFiles["LLM writes 4 files via fsWrite tool"]
        GenPrompt --> ReplacePrompt
        ReplacePrompt --> AgenticLoop
        AgenticLoop --> ReadFiles
        ReadFiles --> WriteFiles
    end

    subgraph Access["5. Ongoing Access"]
        RulesDiscovery["collectWorkspaceRulesInternal scans .amazonq/rules/"]
        AutoEnable["Auto-enable in rulesState"]
        InjectPrompt["Inject into every chat as pinned context"]
        RulesPanel["Display in Rules Panel under memory-bank folder"]
        RulesDiscovery --> AutoEnable
        AutoEnable --> InjectPrompt
        InjectPrompt --> RulesPanel
    end

    Trigger --> Prep
    Prep --> Rank
    Rank --> Generate
    Generate --> Access

    style Trigger fill:#e1f5fe
    style Prep fill:#fff3e0
    style Rank fill:#fce4ec
    style Generate fill:#e8f5e9
    style Access fill:#f3e5f5
```

---

## Testing

Tests are located in `memoryBankController.test.ts` and cover:

-   Trigger phrase detection (positive and negative cases)
-   Memory bank existence checking
-   Directory cleaning and creation
-   TF-IDF analysis pipeline
-   File ranking prompt generation
-   Error handling and fallback scenarios
