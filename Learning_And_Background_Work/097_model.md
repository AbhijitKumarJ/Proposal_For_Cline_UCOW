Okay, here is the detailed blog post for Part 1 of the new series aimed at guiding research coders through evolving Cline into the Universal Code Orchestrator Workbench.

---

**Blog Series:** *Evolving Cline: Building the Universal Code Orchestrator Workbench*

**Part 1: The Vision & Refactoring the Core - From `Task` to `AgentInstance`**

**Welcome, Cline Developers!**

This series is for those already familiar with the Cline codebase – developers who have perhaps fixed a bug, added a feature, or simply explored its architecture (perhaps guided by our previous *Inside Cline* series). We're embarking on an ambitious journey: evolving Cline from a sophisticated *single* agentic assistant into a **Universal Code Orchestrator Workbench**. This workbench won't just execute tasks; it will be a platform to *define, manage, run, evaluate, and even evolve* multiple specialized coding agents working together within VS Code.

This first post sets the stage. We'll articulate the "Workbench" vision, explain why the current core `Task` class needs refactoring to support this vision, and guide you through the crucial first implementation step: extracting the core agentic execution logic into a new, reusable `AgentInstance` class.

**1. The Workbench Vision: Beyond a Single Agent**

Currently, when you give Cline a task, a `Task` object (`src/core/task/index.ts`) is instantiated. This object manages the *entire* lifecycle:
*   Holding the full conversation history (`apiConversationHistory`, `clineMessages`).
*   Managing state like `isStreaming`, `abort` flags.
*   Running the main agentic loop (`recursivelyMakeClineRequests`).
*   Interacting with the specific LLM API via its `ApiHandler`.
*   Executing *all* built-in tools (filesystem, terminal, browser, etc.).
*   Handling user interactions (`ask`/`say`).
*   Persisting state to disk.
*   Managing Checkpoints.

This monolithic approach works well for a single assistant handling one task at a time. However, our vision for the Workbench involves multiple, potentially specialized agents collaborating within defined workflows. Imagine:

*   A `PlanningAgent` outlines steps.
*   A `CodeGeneratorAgent` implements a function based on the plan.
*   A `TestWriterAgent` generates unit tests for that function.
*   A `RefactoringAgent` improves the generated code based on linting results.
*   A `DocumentationAgent` writes docstrings.

Running this requires orchestrating multiple, distinct agent *instances*, each potentially having a different system prompt, a limited set of allowed tools, and focused responsibilities. The current `Task` class, tightly coupled to managing a single, end-to-end conversation and possessing all tools, isn't designed for this modularity.

**2. The Need for Refactoring: Decoupling Execution from Orchestration**

To build the Workbench, we need to separate the *core agentic execution logic* (running one step of reasoning and action) from the *orchestration logic* (managing the overall workflow, passing state between steps, handling persistence).

*   **Current `Task` Responsibilities:**
    *   Agentic Execution Loop (Prompt -> API -> Parse -> Tool -> Result)
    *   Workflow Logic (Implicitly linear, continuous loop until `attempt_completion` or error)
    *   State Management (Conversation History, UI Messages, Checkpoints)
    *   UI Interaction (`say`, `ask`)
*   **Target Architecture:**
    *   **`AgentInstance` (`src/core/agent/AgentInstance.ts`):** Responsible for executing *one single turn* or step of an agent based on a given configuration (`AgentBlueprint`) and input context. Takes context in, performs LLM call and potential tool use, returns result/state changes. It is *stateless* regarding long-term history.
    *   **`WorkflowOrchestrator` (`src/services/workbench/WorkflowOrchestrator.ts`):** Manages the overall workflow lifecycle. Instantiates `AgentInstance` objects for each step, passes context/results between them, handles workflow logic (sequencing, conditionals), manages persistence of workflow state.
    *   **`Controller` (`src/core/controller/index.ts`):** Remains the top-level manager, handling UI communication, global state, and potentially initiating workflows via the `WorkflowOrchestrator`.

This refactoring allows us to reuse the core LLM interaction and tool execution logic (`AgentInstance`) within different workflow structures managed by the `WorkflowOrchestrator`.

**3. The Refactor Plan: Creating `AgentInstance`**

Our first concrete step is to create the `AgentInstance` class by extracting the relevant logic from the existing `Task` class.

*   **Coder A (Ideator):** Let's create `src/core/agent/AgentInstance.ts`. We'll copy methods like `attemptApiRequest`, `parseAssistantMessage`, `presentAssistantMessage` (and its tool-handling switch statement), and the core logic *inside* `recursivelyMakeClineRequests` (but not the recursive call itself). The `AgentInstance` should take its configuration (system prompt, allowed tools, model config derived from an `AgentBlueprint`) and the current conversation history segment as input to an `executeStep` method, and return the result (assistant message content, tool results, errors).
*   **Coder B (Critic):** Copying large chunks of code is risky for maintenance. Can we make `Task` *use* `AgentInstance` internally first, ensuring backward compatibility? Also, how much state does `AgentInstance` truly need? It still needs things like `isStreaming`, `abort` flag for *within* a single step, and access to services like `TerminalManager`, `McpHub`, `DiffViewProvider`. We can't make it purely functional easily.
*   **Refinement (A):** Good point. Let's start by identifying methods purely related to the *single-turn execution cycle*. We'll duplicate and adapt these into `AgentInstance`. `AgentInstance` will still need references to shared services (passed via constructor or method calls), and minimal internal state for managing the stream parsing and tool execution *within that one step*. It will *not* manage `apiConversationHistory` or `clineMessages` persistence. We'll keep the original `Task` class functional for now and slowly migrate the main Cline chat view to use a simple workflow involving a single `AgentInstance` step later.
*   **Implementation Details (C):**
    1.  **Create Files:**
        *   `src/core/agent/AgentInstance.ts`
        *   `src/shared/workbench/types.ts` (for `AgentBlueprint` and related types)
    2.  **Define `AgentBlueprint` (Basic):** In `types.ts`, define an initial interface:
        ```typescript
        interface AgentBlueprint {
            id: string;
            description?: string;
            baseSystemPrompt: string;
            allowedTools: ToolUseName[]; // From src/core/assistant-message/index.ts
            modelConfig: ApiConfiguration; // Reusing existing config type
            // contextStrategy?: string; // Add later
        }
        ```
    3.  **`AgentInstance` Class:**
        *   **Constructor:** Takes dependencies needed for tool execution (e.g., `mcpHub`, `workspaceTracker`, `diffViewProvider`, `terminalManager`, `browserSession`, `fileContextTracker`, `clineIgnoreController`, `context`). Store these as private members.
        *   **`executeStep` Method:**
            *   Accepts `blueprint: AgentBlueprint`, `currentHistorySegment: Anthropic.MessageParam[]`, `inputContent: UserContent` as parameters.
            *   Constructs the *full system prompt* using `blueprint.baseSystemPrompt` (and potentially injected context - simplified for now).
            *   Builds the *temporary* `apiConversationHistory` for *this specific API call* using `currentHistorySegment` plus the new `inputContent`.
            *   Calls `this.api = buildApiHandler(blueprint.modelConfig)` to set the correct API handler for this step.
            *   Calls `this.attemptApiRequest(...)` using the constructed prompt and history segment.
            *   Processes the stream using logic adapted from `recursivelyMakeClineRequests` (parsing, calling `presentAssistantMessage`).
            *   **Crucially:** Modifies `presentAssistantMessage`:
                *   It should *check* if a tool requested by the LLM is in the `blueprint.allowedTools` list before executing. If not, return a specific error result.
                *   Instead of calling `this.say`/`this.ask` directly, it should *buffer* the intended UI messages and tool results.
                *   It should *return* a result object like `{ success: boolean; assistantOutput: AssistantMessageContent[]; toolResult?: ToolResponse; error?: string; finalApiMetrics: ApiMetrics }` when the single turn completes (either a tool is executed/rejected, or the LLM responds with only text).
            *   Does *not* modify the main `apiConversationHistory` or `clineMessages`.
            *   Does *not* call itself recursively.
    4.  **Adapt Core Methods:** Copy `attemptApiRequest`, `parseAssistantMessage`, `presentAssistantMessage`, `handleError`, `loadContext` (simplified initially), etc., from `Task.ts` into `AgentInstance.ts`. Remove recursive calls and direct state mutations related to conversation history. Modify tool handlers to check `allowedTools` and buffer/return results instead of calling `say`/`ask`.

*   **Refinements (D):**
    *   The `AgentInstance` needs a mechanism to receive user responses for tool approvals. The `executeStep` method could accept an optional callback function `requestApproval(askType, askPayload): Promise<{response: ClineAskResponse; ...}>`. The `WorkflowOrchestrator` would provide this callback, linking it back to the main `Controller`'s `ask` mechanism and the UI.
    *   Ensure all dependencies required by the copied methods (like `formatResponse`, utility functions) are correctly imported.
    *   Add basic unit tests for `AgentInstance.executeStep` using mocked dependencies and API responses to verify tool filtering and result returning.

**4. Using Cline to Assist the Refactor (Meta-Development Example)**

Let's try using *current* Cline to help with this refactoring:

1.  **Start Cline:** Open the Cline project, start a new task.
2.  **Provide Context:**
    ```
    I am refactoring the `Task` class in `@/src/core/task/index.ts` to create a new `AgentInstance` class in `@/src/core/agent/AgentInstance.ts`. The goal is to separate the single-turn agent execution logic from the long-term state and workflow management.

    Load these files for context:
    @/src/core/task/index.ts
    @/src/core/assistant-message/index.ts
    @/src/shared/workbench/types.ts (Assume AgentBlueprint is defined here)
    ```
3.  **Specific Instruction:** "Copy the `attemptApiRequest` method from `Task` into `AgentInstance`. Identify any direct dependencies within this method on `Task`'s instance variables (like `this.apiConversationHistory`, `this.clineMessages`, `this.contextManager`, `this.abort`) and modify the method signature and internal logic so that these dependencies are passed in as parameters or accessed via service dependencies injected in the `AgentInstance` constructor."
4.  **Review & Iterate:** Cline will likely propose a `replace_in_file` for `AgentInstance.ts`. Review the diff carefully. Does it correctly identify dependencies? Does the new signature make sense? Does it remove the history-modifying lines?
    *   *Feedback Example:* "Rejected. You correctly identified `this.apiConversationHistory` as a dependency, but you removed the context management call (`this.contextManager.getNewContextMessagesAndMetadata`). This logic should be moved *outside* `attemptApiRequest` to the caller (the future `WorkflowOrchestrator`), but the API call *still needs* to receive the processed history segment as input. Modify the signature to accept `processedHistory: Anthropic.MessageParam[]` instead of the raw history."
5.  **Repeat:** Continue this process for other key methods (`parseAssistantMessage`, `presentAssistantMessage`), guiding Cline to adapt the logic for a single-step execution context and return values instead of mutating class state or calling UI functions directly.

**Conclusion: Laying the Foundation**

Refactoring the monolithic `Task` class into a more focused `AgentInstance` is the essential first step towards the Universal Code Orchestrator Workbench. It decouples core execution logic from state and workflow management, creating a reusable building block for our multi-agent system. While seemingly just code restructuring, this requires careful thought about dependencies, state management, and the precise inputs/outputs of a single agentic "turn." The next steps will involve building the `WorkflowOrchestrator` to manage these `AgentInstance` objects and define how they connect in sequence.

---

**Next Up (Part 2):** We'll dive into defining `AgentBlueprints` and the `WorkflowDefinition` schema, and implement the services needed to store and retrieve these configurations.