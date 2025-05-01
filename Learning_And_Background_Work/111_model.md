Okay, let's create the detailed blog post for Part 7 of the "Evolving Cline" series, focusing on implementing conditional logic within workflows.

---

**Blog Series:** *Evolving Cline: Building the Universal Code Orchestrator Workbench*

**Part 7: Adding Logic - Conditional Workflows**

**Welcome Back, Workbench Flow Designers!**

In our journey building the Cline Universal Code Orchestrator Workbench, we've established core components: reusable `AgentInstance` objects (Part 1) defined by `AgentBlueprint` templates (Part 2), a basic `WorkflowOrchestrator` to run them sequentially (Part 3), persistent storage for our designs (Part 4), initial monitoring/logging (Part 5), and a `WorkflowScratchpad` for simple inter-agent communication (Part 6).

Currently, our workflows are strictly linear pipelines. Agent A runs, its output feeds Agent B, which feeds Agent C, and so on. If any step fails, the entire workflow halts. However, real-world processes often require branching logic: "If the tests pass, deploy; otherwise, report the errors." "Try approach X; if it fails with error Y, try approach Z instead."

This post focuses on adding **conditional branching logic** to our workflows. We'll explore how to extend our `WorkflowDefinition` schema and enhance the `WorkflowOrchestrator` to support `onSuccess` and `onFailure` transitions, allowing workflows to react dynamically based on the outcome of individual agent steps. Join our design discussion with **A (Ideator)**, **B (Critic)**, **C (Implementer)**, and **D (Refiner)**.

---

**Session Start: Moving Beyond Linearity**

**Coder A (Ideator):** Linear sequences are too restrictive. We need our workflows to make decisions. The most fundamental decision point is the success or failure of an agent step. I propose extending our `WorkflowStep` definition to include optional `onSuccess` and `onFailure` properties.
*   **`onSuccess`:** Specifies the `stepId` to jump to if the current step completes successfully (`AgentStepResult.success === true`). If omitted, it defaults to the next step in the sequence.
*   **`onFailure`:** Specifies the `stepId` to jump to if the current step fails (`AgentStepResult.success === false`). If omitted, the workflow halts (current behavior). We could also potentially add a *global* failure handler step for the entire workflow.

This allows us to define basic branching, like try/catch blocks or if/else logic, directly within the workflow structure. For example:

```yaml
# Example Workflow Step
- stepId: run_tests
  blueprintId: jest-tester
  onSuccess: deploy_app # Jump to deploy if tests pass
  onFailure: report_errors # Jump to error reporter if tests fail
- stepId: report_errors
  blueprintId: error-summarizer
  # Implicitly ends workflow after this step
- stepId: deploy_app
  blueprintId: deployer
  # Implicitly ends workflow after this step
```

**Coder B (Critic):** This is a good, structured start to branching. Key considerations:
1.  **Defining "Failure":** Currently, `AgentStepResult.success` is a simple boolean. What if we need more granular failure handling? Agent A might fail because a file wasn't found (maybe recoverable by creating it), while Agent B might fail due to an unrecoverable API authentication error. Should `onFailure` allow branching based on specific `errorCode`s returned in the `AgentStepResult`?
2.  **Loops:** This simple `onSuccess`/`onFailure` jump mechanism could easily create infinite loops if not carefully designed in the workflow definition (e.g., Step A fails -> jump to Step B -> Step B succeeds -> jump back to Step A). Does the orchestrator need loop detection?
3.  **Passing Error Context:** When jumping due to `onFailure`, the target step (e.g., `report_errors`) needs access to the *reason* for the failure (the `error` message and potentially `errorCode` from the failed step's `AgentStepResult`). How is this context passed? Via the Scratchpad? As part of the implicit input?
4.  **UI Representation:** How do we clearly visualize these conditional branches in the Workflow Editor and Instance Monitor UI? Simple linear diagrams won't suffice.

**Coder A (Ideator):** Excellent points. Let's refine:
1.  **Granular Failure:** Yes, `onFailure` should support routing based on error codes. Let's make it an array of objects: `onFailure: [{ errorCode?: 'FILE_NOT_FOUND', targetStepId: 'create_file_step' }, { targetStepId: 'generic_failure_handler' }]`. The orchestrator tries to match the `errorCode` returned by the failed step; if no specific match, it uses the entry without an `errorCode` (the default failure path). If no `onFailure` array exists, it halts.
2.  **Loops:** Loop detection can be complex. For V1, let's *not* implement automatic detection in the orchestrator. We'll rely on the *workflow designer* (human or future meta-agent) to avoid creating infinite loops. We can add a maximum step execution count per workflow instance as a basic safeguard later if needed.
3.  **Error Context Passing:** When transitioning via `onFailure`, the `WorkflowOrchestrator` should automatically write the failed step's `AgentStepResult.error` and `AgentStepResult.errorCode` to predefined keys in the `WorkflowScratchpad` (e.g., `lastError`, `lastErrorCode`). The error-handling agent's blueprint prompt would instruct it to read these keys using `read_scratchpad`.
4.  **UI:** Acknowledge the visualization challenge. The initial text-based editor (YAML/JSON) can represent this structure. The read-only visual preview (Part 3) could use Mermaid's flowchart syntax, which supports conditional diamond shapes, to represent the potential branches defined in the text.

**Coder B (Critic):** Making `onFailure` an array matching on `errorCode` provides the needed granularity. Passing error context via the Scratchpad is a clean solution using existing mechanisms. Deferring loop detection is pragmatic for V1, putting the onus on careful workflow design. Using Mermaid for visualization is a good start. This seems like a solid plan for introducing conditional logic.

**(A and B agree on the `onSuccess`/`onFailure` structure with `errorCode` matching and Scratchpad for error context.)**

---

**Implementation Details (Coder C & D)**

**(C outlines the initial implementation, D suggests refinements)**

**Coder C (Implementer):**
1.  **Update Schemas (`src/shared/workbench/types.ts`):**
    *   Modify `WorkflowStep` interface:
        ```typescript
        interface FailureTransition {
            errorCode?: string; // Optional error code to match
            targetStepId: string;
        }
        interface WorkflowStep {
            stepId: string;
            blueprintId: string;
            initialPromptModifier?: string;
            onSuccess?: { targetStepId: string }; // Optional target on success
            onFailure?: FailureTransition[];    // Optional array of targets on failure
        }
        ```
    *   Modify `AgentStepResult`: Ensure it includes `errorCode?: string;`.
2.  **`AgentInstance` Modifications:**
    *   Ensure the `executeStep` method and internal tool handlers can return specific `errorCode` strings within the `AgentStepResult` when known failure conditions occur (e.g., tool parameter validation failure, specific API errors, file not found). Define common error codes as constants.
3.  **`WorkflowOrchestrator._executeWorkflowStep` Logic:**
    *   After receiving `stepResult = await agentInstance.executeStep(...)`:
    *   **Check Success:**
        *   If `stepResult.success`:
            *   Find the *next* step index:
                *   Check `currentStepDef.onSuccess?.targetStepId`. If present, find the index of the step with that `stepId`.
                *   If `onSuccess` is not defined, the next step is simply `stepIndex + 1`.
            *   Handle case where `targetStepId` is invalid or index is out of bounds (workflow succeeds if it was the intended logical end).
            *   If a valid next step exists, update `instance.currentStepIndex` and recursively call `_executeWorkflowStep`.
            *   If no valid next step (end of workflow), set status to 'succeeded', persist, notify, remove from active.
        *   If `!stepResult.success`:
            *   Write `stepResult.error` and `stepResult.errorCode` to `instance.scratchpad` using keys like `lastError` and `lastErrorCode`.
            *   Check `currentStepDef.onFailure`. If present and an array:
                *   Try to find a transition matching `stepResult.errorCode`.
                *   If found, find the index of that transition's `targetStepId`.
                *   If no specific code match, look for a default failure transition (entry with no `errorCode`). If found, find its target index.
            *   If a valid target step index is found (from `onFailure`):
                *   Update `instance.currentStepIndex` to the target index.
                *   Persist state (including scratchpad updates).
                *   Recursively call `_executeWorkflowStep` with the *new* index.
            *   If no `onFailure` defined or no matching transition found: Set `status = 'failed'`, `error = stepResult.error`. Persist, notify UI, remove from `activeInstances`. **Stop.**
4.  **UI Updates (`webview-ui/`):**
    *   **Workflow Editor:** Update the text editor (YAML/JSON) to support the new `onSuccess`/`onFailure` schema fields. Add validation hints.
    *   **Visualization Pane:** Implement logic to parse the text definition and generate Mermaid flowchart syntax reflecting the conditional branches. Use diamond shapes for decision points based on step success/failure.
    *   **Trace Viewer:** When visualizing the workflow graph for a completed instance, highlight the path actually taken based on the logged `onSuccess`/`onFailure` transitions derived from `stepResult.success` and `stepResult.errorCode`.

**Coder D (Refiner):**
*   **Refine C:**
    *   **Error Code Standardization:** Define a set of standard `errorCode` constants (e.g., `TOOL_PARAM_ERROR`, `FILE_NOT_FOUND`, `API_AUTH_ERROR`, `NETWORK_TIMEOUT`) in `src/shared/` to be used consistently by `AgentInstance` and referenced in workflow definitions.
    *   **Workflow Validation:** Add validation logic to `WorkflowService.saveWorkflow` to detect potential issues like invalid `targetStepId` references or unreachable steps *before* attempting execution.
    *   **Scratchpad Keys:** Use clearly defined constants for the keys used to pass error context (e.g., `WORKFLOW_LAST_ERROR_MESSAGE`, `WORKFLOW_LAST_ERROR_CODE`) to avoid typos. Document these keys.
    *   **Mermaid Generation:** Ensure the Mermaid generation logic handles complex branching and potential cycles (even if execution logic doesn't prevent them yet) gracefully, perhaps by limiting graph depth or clearly marking potential loops in the visualization.
    *   **State Management:** Ensure the `conversationHistory` appended by the orchestrator accurately reflects the non-linear execution path when branches are taken. Each step's execution should still receive the linear history *up to that point*.

**Conclusion: Adding Intelligence to the Flow**

Introducing conditional logic via `onSuccess` and `onFailure` transitions elevates the Cline Workbench from a simple sequence runner to a true workflow orchestrator capable of basic decision-making. By allowing workflows to branch based on the success or specific error codes of agent steps, and by passing error context via the Scratchpad, we enable more robust and realistic automation scenarios like try/catch patterns or adaptive error handling. While careful workflow design is needed to avoid infinite loops, this structured approach provides essential control flow capabilities, laying the groundwork for eventually tackling more complex orchestration patterns like parallelism and dynamic replanning.

---

**Next Up (Part 38 - Confirmed):** Now that we have basic orchestration, we can revisit **Advanced Evaluation & Validation**, focusing on integrating richer metrics (code quality, security, performance) into the evaluation process managed by the `evals/` framework and benchmark adapters.