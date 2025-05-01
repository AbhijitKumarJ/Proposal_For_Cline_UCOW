Okay, here is the detailed blog post for Part 3 of the "Evolving Cline" series, focusing on building the initial Workflow Orchestrator.

---

**Blog Series:** *Evolving Cline: Building the Universal Code Orchestrator Workbench*

**Part 3: Simple Sequences - The Basic Workflow Orchestrator**

**Welcome Back, Workbench Engineers!**

In Part 1, we refactored Cline's core execution logic into a reusable `AgentInstance` capable of performing a single agentic step. In Part 2, we designed the `AgentBlueprint` – the template defining the characteristics (prompt, tools, model) of specialized agents. Now, we need the conductor for our orchestra: the **Workflow Orchestrator**.

This crucial component is responsible for taking a defined sequence of agent steps and managing their execution, passing necessary information between them, and tracking the overall progress. This post details the design and initial implementation of a *basic* `WorkflowOrchestrator` capable of handling simple, linear sequences, laying the foundation for more complex orchestration later. We'll follow our design discussion format with **A (Ideator)**, **B (Critic)**, **C (Implementer)**, and **D (Refiner)**.

---

**Session Start: From Blueprint to Action**

**Coder A (Ideator):** We have `AgentInstance` (executes a single step) and `AgentBlueprint` (defines an agent type). Now we need the system to run a *sequence* of these. Let's define a simple `WorkflowDefinition` containing an ordered list of steps, where each step specifies which `AgentBlueprint` to use. The `WorkflowOrchestrator` service will take this definition and an initial input (like the user's task prompt), then instantiate and execute the `AgentInstance` for each step sequentially, feeding the output of one step as the primary input to the next.

**Coder B (Critic):** "Feeding the output" is the tricky part. What *is* the "output" of an `AgentInstance` step? Its final assistant text message? The result of its last tool call? A combination? This needs to be clearly defined. Also, where does the *conversation history* fit in? Each `AgentInstance` needs context. Does the Orchestrator maintain a growing history and pass the relevant segment to each step? What happens if a step fails? Does the whole workflow stop?

**Coder A (Ideator):** Good questions. Let's define the `AgentInstance.executeStep` return value more formally (refining from Part 1):
```typescript
interface AgentStepResult {
  success: boolean;
  outputContent: UserContent; // Formatted content suitable as input for the *next* user turn (e.g., final text, formatted tool result)
  finalAssistantMessage: AssistantMessageContent[]; // The full assistant message blocks generated
  error?: string;
  errorCode?: string; // For future conditional branching
  apiMetrics?: ApiMetrics; // Tokens, cost for this step
  // Maybe add tool call details later
}
```
The `outputContent` is key – it's what gets passed implicitly as the main user input to the *next* agent step in the sequence. The Orchestrator *will* maintain the overall conversation history (`Anthropic.MessageParam[]`) for the *entire workflow instance*. Before calling `executeStep` for step N, it passes the history up to that point *plus* the `outputContent` from step N-1 (formatted as a user message) as the input context. It then appends the `finalAssistantMessage` from step N's result to the history before proceeding to step N+1. If any step returns `success: false`, the workflow halts immediately, reporting the error.

**Coder B (Critic):** Okay, so the Orchestrator manages the persistent conversation history, and each `AgentInstance` operates on a segment of it plus the specific output from the previous step. Halting on first error is simple but maybe not always desirable. What about persistence? If VS Code crashes mid-workflow, how does it resume?

**Coder A (Ideator):** Persistence is vital. The `WorkflowOrchestrator` needs to manage the state of each *running* workflow instance. We need a `WorkflowInstance` state object that includes: `instanceId`, `definitionId`, `status` ('running', 'succeeded', 'failed', 'paused'), `currentStepIndex`, `conversationHistory`, `stepResults` (an array storing the `AgentStepResult` from each completed step), and the `scratchpad` (from Part 31). After *every* step completes successfully, the Orchestrator must persist this entire `WorkflowInstance` state to disk (using our task storage mechanism, keyed by `instanceId`). On VS Code restart, the Orchestrator can load incomplete instances and potentially offer to resume them (though full automatic resumption is complex; maybe just show failed/paused status initially).

**Coder B (Critic):** Persisting state after every step ensures durability but could add I/O overhead. We'll need to make saving asynchronous and efficient. The `AgentStepResult` needs to be carefully designed to be JSON-serializable for storage. Okay, sequential execution with state persistence and clear input/output passing seems feasible for V1.

**(A and B agree on the core sequential orchestration logic, state management, and persistence.)**

---

**Implementation Details (Coder C & D)**

**(C outlines the initial implementation, D suggests refinements)**

**Coder C (Implementer):**
1.  **Define Schemas (`src/shared/workbench/types.ts`):**
    *   `WorkflowDefinition`: `{ id: string; description?: string; steps: WorkflowStep[]; }`
    *   `WorkflowStep`: `{ stepId: string; blueprintId: string; initialPromptModifier?: string; // Optional text to prepend to blueprint prompt for this step }` (Defer `onSuccess`/`onFailure` for now).
    *   `AgentStepResult`: As defined by Coder A (ensure `outputContent` using `UserContent` type alias, and `finalAssistantMessage` uses `AssistantMessageContent[]`). Add fields for `stepId`, `startTimestamp`, `endTimestamp`.
    *   `WorkflowInstance`: `{ instanceId: string; definitionId: string; status: 'running' | 'succeeded' | 'failed' | 'paused'; currentStepIndex: number; startTime: number; endTime?: number; conversationHistory: Anthropic.MessageParam[]; stepResults: AgentStepResult[]; scratchpad: Record<string, string>; error?: string; }`
2.  **`WorkflowOrchestrator` Service (`src/services/workbench/WorkflowOrchestrator.ts`):**
    *   **Dependencies:** Needs access to `BlueprintService`, `AgentInstance` factory (or pass dependencies like McpHub etc.), storage functions, `LoggingService`.
    *   **`activeInstances: Map<string, WorkflowInstance>`:** In-memory map for running workflows.
    *   **`async startWorkflow(definitionId: string, initialInput: UserContent): Promise<string>`:**
        *   Loads `WorkflowDefinition` using `BlueprintService`.
        *   Generates `instanceId`.
        *   Creates initial `WorkflowInstance` state (status 'running', step 0, initial history with user input).
        *   Persists initial state asynchronously (`saveWorkflowInstance`).
        *   Adds to `activeInstances`.
        *   Calls `_executeWorkflowStep(instanceId, 0)` (do not await, runs in background).
        *   Returns `instanceId`.
    *   **`async _executeWorkflowStep(instanceId: string, stepIndex: number): Promise<void>`:**
        *   **(Core Loop Logic):**
        *   Get `WorkflowInstance` state from `activeInstances` (or load if resuming). If not found or not 'running', return.
        *   Get current `WorkflowStep` definition from `definition.steps[stepIndex]`.
        *   Load the required `AgentBlueprint` using `BlueprintService`.
        *   Construct input for `AgentInstance`: Combine `instance.conversationHistory` with the `outputContent` from the *previous* step's result (`instance.stepResults[stepIndex - 1]`), formatted as a user message. Handle the very first step using `initialInput`. Apply `initialPromptModifier` if present.
        *   Log `AgentStepStartEvent`.
        *   Instantiate `AgentInstance` with blueprint config, dependencies, and potentially a callback for tool approval routing.
        *   `const stepResult = await agentInstance.executeStep(prompt, historySegment, inputContent)` (wrap in `try...catch`).
        *   Log `AgentStepEndEvent` (including `stepResult`).
        *   Update `WorkflowInstance` state:
            *   Append `stepResult.finalAssistantMessage` to `conversationHistory`.
            *   Add `stepResult` to `stepResults`.
            *   Increment `currentStepIndex`.
            *   Update `status` based on `stepResult.success` and whether it's the last step.
        *   Persist updated `WorkflowInstance` state (`saveWorkflowInstance`).
        *   Notify UI about instance update (`postStateToWebview` or dedicated message).
        *   If `stepResult.success` and not the last step, recursively call `_executeWorkflowStep(instanceId, stepIndex + 1)`.
        *   If `!stepResult.success`, set status to 'failed', persist, notify UI, remove from `activeInstances`.
        *   If last step and success, set status to 'succeeded', persist, notify UI, remove from `activeInstances`.
    *   **`async loadAndResumeWorkflows(): Promise<void>`:** On extension startup, scans storage for 'running' or 'paused' instances, loads them into `activeInstances`, and potentially offers users to resume (initially just mark as failed/paused).
    *   **`async saveWorkflowInstance(instance: WorkflowInstance): Promise<void>`:** Handles async saving of instance state to disk (e.g., `globalStorage/tasks/workflow_{instanceId}/instance_state.json`).
    *   **`async getWorkflowInstance(instanceId): Promise<WorkflowInstance | undefined>`:** Loads state from disk.
    *   **`pauseWorkflow`, `resumeWorkflow`, `cancelWorkflow` methods:** (Future) Interact with `activeInstances` and persist status.
3.  **`AgentInstance` Modifications:**
    *   Ensure `executeStep` accepts context/history as input and returns the structured `AgentStepResult`.
    *   Remove internal history persistence.
    *   Implement logic to use the passed-in `allowedTools` list.
    *   Integrate with `LoggingService` for structured tracing.
    *   Accept and use an `approveToolCallback` function passed from the Orchestrator.
4.  **UI (`Instance Monitor`):**
    *   Fetch running/recent instances via gRPC (`WorkbenchService.getWorkflowInstances()`).
    *   Display list with basic status.
    *   On select, fetch detailed state (`getWorkflowInstance(id)`) and display logs/step results in the detail panel.

**Coder D (Refiner):**
*   **Refine C:**
    *   **State Passing:** Passing the *entire* `outputContent` (which could be large, e.g., a full generated file) to the next step's prompt might quickly bloat context. Consider alternatives:
        *   Agent steps explicitly write key results to the `Scratchpad` using `write_to_scratchpad`. The next step's prompt template (or the Orchestrator) explicitly references needed keys `{{scratchpad.key_name}}`.
        *   Return only a *summary* or *reference* (like a file path) in `outputContent`, and the next agent uses tools (`read_file`, `read_scratchpad`) if it needs the full details. This requires more intelligent agents but saves context. Let's start with passing full `outputContent` but plan for refinement.
    *   **Error Handling:** The Orchestrator needs robust error handling *around* the `agentInstance.executeStep` call to catch exceptions thrown *by* the agent instance itself (not just failed steps). Mark workflow as 'failed' in these cases.
    *   **Persistence Format:** JSON is simple for `WorkflowInstance` state. Ensure complex objects like `conversationHistory` (which might contain non-standard elements like image data if not careful) serialize correctly.
    *   **Concurrency Control:** While full parallelism is deferred, ensure `activeInstances` map access is safe if future features introduce concurrent operations (e.g., use a Map or ensure atomic updates if needed, though less critical in single-threaded Node.js extension host).
    *   **Tool Approval Routing:** The mechanism for `AgentInstance` to request approval via a callback provided by the Orchestrator, which then routes it to the central `Controller.ask`, needs careful implementation to handle the async flow correctly.

**Conclusion: Orchestrating the Orchestra**

The `WorkflowOrchestrator` is the brain of the Cline Workbench, translating static definitions into dynamic execution. By starting with simple linear sequences, clearly defining the input/output contract for each agent step (`AgentStepResult`), managing conversation history centrally, and ensuring persistent state for workflow instances, we establish a functional core. This foundation allows us to layer on more complex features like conditional branching, richer inter-agent communication via the Scratchpad, and eventually parallelism, while the structured logging implemented here paves the way for advanced observability and meta-agent analysis in future stages.

---

**Next Up (Part 37):** We'll revisit the Shared Knowledge Base (SKB), discussing the practical implementation details of the hybrid storage approach (SQLite, Graph, Vector) and the crucial `IndexingService` needed to populate and maintain it.