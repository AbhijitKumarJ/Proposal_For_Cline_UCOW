Okay, let's create the detailed blog post for Part 6 of the "Evolving Cline" series, focusing on implementing the Workflow Scratchpad for basic inter-agent communication.

---

**Blog Series:** *Evolving Cline: Building the Universal Code Orchestrator Workbench*

**Part 6: Enabling Dialogue - The Workflow Scratchpad**

**Welcome Back, Workbench Connectors!**

We've made significant strides in designing our Universal Code Orchestrator Workbench. We have reusable `AgentInstance` objects configured by `AgentBlueprint` definitions (Parts 1-2), a `WorkflowOrchestrator` capable of executing them in sequence (Part 3), persistent storage for our designs (Part 4), and foundational monitoring and logging capabilities (Part 5).

Currently, communication between agent steps in a workflow is limited: the primary output of step N becomes the main input for step N+1. But what if agents need to share smaller pieces of information, status flags, or accumulated findings *without* cluttering the main output passed to the next agent's prompt? What if Agent A discovers a configuration value that Agent C needs later in the workflow?

This post focuses on implementing the first mechanism for richer inter-agent communication: the **Workflow Scratchpad**. We'll follow our design discussion with **A (Ideator)**, **B (Critic)**, **C (Implementer)**, and **D (Refiner)** to establish how this simple, shared state mechanism works and how agents interact with it.

---

**Session Start: The Need for Shared Notes**

**Coder A (Ideator):** Our current sequential workflow passes the main output (`outputContent` from `AgentStepResult`) between steps. This works for linear pipelines where the primary result of one step feeds the next. However, it's inefficient for sharing smaller, auxiliary pieces of information or state that multiple, potentially non-adjacent steps might need.

Consider these scenarios:
*   Agent `Setup` detects the project uses TypeScript. Agent `TestWriter` (several steps later) needs this info to generate `.ts` test files.
*   Agent `FileSearcher` identifies a list of files needing refactoring. Agent `RefactorRunner` (later) needs this exact list.
*   Agent `BuildRunner` sets a status flag indicating success/failure that a later `DeployAgent` needs to check.

Passing this information through the main `outputContent` of every intermediate agent is cumbersome and pollutes their primary input. We need a simple, shared blackboard. I propose the **Workflow Scratchpad**: a key-value store associated with each *workflow instance*, managed by the `WorkflowOrchestrator`, that agents can read from and write to using dedicated tools.

**Coder B (Critic):** A shared blackboard is a classic pattern, but also a potential source of coupling and hard-to-debug interactions.
1.  **Data Format:** Key-value sounds simple, but what are the values? Strings only? JSON? Allowing complex objects increases serialization/deserialization overhead and potential errors. Sticking to strings seems safer initially.
2.  **Key Naming Conflicts:** If Agent A writes to key `project_language` and Agent B later overwrites it unintentionally, how do we manage this? Requires clear conventions or perhaps namespacing (e.g., `agentA_output_language`).
3.  **Discoverability:** How does Agent C know that Agent A wrote the information it needs to the key `agentA_output_language`? This requires either implicit knowledge baked into agent prompts or a more structured way to declare inputs/outputs beyond the main sequence.
4.  **State Management:** The Orchestrator now has more state to manage and persist for each instance. How large can this Scratchpad get? Does it need cleanup?
5.  **Concurrency (Future):** While we deferred parallelism, if multiple agents *could* write to the Scratchpad concurrently, we'd need atomic operations or locking, which a simple `Map<string, string>` doesn't provide.

**Coder A (Ideator):** Valid concerns. Let's keep V1 simple and address these:
1.  **Data Format:** **Strings only** for V1. Agents needing complex data must serialize it to JSON strings before writing and parse it after reading. This keeps the core mechanism simple.
2.  **Key Naming:** Rely on **conventions and prompt engineering** initially. Prompts for agents within a specific workflow template should instruct them on which keys to read/write. E.g., "Read the target language from scratchpad key `project_language` before generating code." We can explore namespacing later if conflicts become common.
3.  **Discoverability:** Handled by the workflow design and agent prompts for now. An agent's prompt explicitly tells it where to find its necessary inputs (either from the main `outputContent` of the previous step or specific Scratchpad keys).
4.  **State Management:** The Orchestrator persists the `scratchpad: Record<string, string>` field within the `WorkflowInstance` state after each step (just like `conversationHistory`). Size limits aren't a primary concern for V1, assuming string values.
5.  **Concurrency:** Acknowledged limitation. For V1 (sequential/conditional workflows), concurrent writes aren't an issue. When we add parallelism (Phase 4+), we'll need to revisit this, potentially adding versioning or atomic updates to the Scratchpad mechanism or restricting which agents can write concurrently.

**Coder B (Critic):** Okay, starting with a string-only key-value store, relying on prompt conventions for key usage, and persisting it with the workflow instance state is a manageable V1. It provides the basic shared note capability without introducing heavy dependencies or complex concurrency controls immediately. The main risk is agents failing due to missing/incorrect keys if prompts aren't precise or conventions aren't followed.

**(A and B agree on the simple, string-based, convention-driven Scratchpad for V1.)**

---

**Implementation Details (Coder C & D)**

**(C outlines the initial implementation, D suggests refinements)**

**Coder C (Implementer):**
1.  **Update Schemas (`src/shared/workbench/types.ts`):**
    *   Add `scratchpad: Record<string, string>;` to the `WorkflowInstance` interface.
2.  **Define New Tools (`src/core/assistant-message/index.ts`):**
    *   Add `"read_scratchpad"` and `"write_scratchpad"` to `toolUseNames` and `ToolUseName`.
    *   Add `"key"` and `"value"` to `toolParamNames` and `ToolParamName`.
3.  **Update System Prompt (`src/core/prompts/system.ts`):**
    *   Add descriptions for the new tools under the `# Tools` section:
        ```xml
        ## read_scratchpad
        Description: Reads a value associated with a key from the current workflow's shared scratchpad. Use this to retrieve information saved by previous steps in the workflow.
        Parameters:
        - key: (required) The key of the value to read.
        Usage:
        <read_scratchpad>
        <key>key_name_here</key>
        </read_scratchpad>

        ## write_scratchpad
        Description: Writes a key-value pair to the current workflow's shared scratchpad. Use this to save information or status flags for subsequent steps in the workflow. Existing keys will be overwritten. Values must be strings (serialize complex data to JSON if necessary).
        Parameters:
        - key: (required) The key to associate with the value.
        - value: (required) The string value to save.
        Usage:
        <write_scratchpad>
        <key>key_name_here</key>
        <value>value_to_save</value>
        </write_scratchpad>
        ```
4.  **Implement Tool Handlers (`AgentInstance.presentAssistantMessage`):**
    *   Add `case "read_scratchpad":` and `case "write_scratchpad":` to the tool execution `switch` statement.
    *   These handlers will need access to the *current* workflow instance's scratchpad. This implies the `WorkflowOrchestrator` needs to pass either the scratchpad itself or callback functions (`readScratchpad(key)`, `writeScratchpad(key, value)`) to the `AgentInstance` when calling `executeStep`. Passing callbacks might be cleaner to encapsulate state management within the Orchestrator.
    *   **`read_scratchpad` handler:**
        *   Validate required `key` parameter.
        *   Call `orchestratorReadCallback(key)`.
        *   Format the retrieved value (or a "key not found" message) using `formatResponse.toolResult`.
        *   Use `pushToolResult` to return the value to the LLM.
    *   **`write_scratchpad` handler:**
        *   Validate required `key` and `value` parameters.
        *   Call `orchestratorWriteCallback(key, value)`.
        *   Format a simple success message (e.g., "Value saved to scratchpad for key: [key].") using `formatResponse.toolResult`.
        *   Use `pushToolResult`.
    *   Remember to handle potential errors from the callbacks/orchestrator.
5.  **`WorkflowOrchestrator` Modifications:**
    *   Modify `_executeWorkflowStep` to pass the appropriate read/write callbacks bound to the *current* `WorkflowInstance.scratchpad`.
    *   Ensure the `WorkflowInstance.scratchpad` is correctly loaded from storage when resuming/starting and saved back to storage via `_saveWorkflowInstance` after *every* step (as writes can happen anytime).
6.  **UI (Trace Viewer):**
    *   Enhance the `TraceLog` schema/events for `TOOL_ATTEMPT` and `TOOL_RESULT` to include scratchpad keys/values involved.
    *   Update the Trace Viewer UI to display Scratchpad read/write operations clearly within the step details.

**Coder D (Refiner):**
*   **Refine C:**
    *   **Callback vs. Passing State:** Passing callbacks (`readScratchpad`, `writeScratchpad`) from the Orchestrator to `AgentInstance` is indeed cleaner than passing the entire mutable scratchpad map. It keeps state management centralized.
    *   **Serialization for Storage:** Ensure the `Record<string, string>` for the scratchpad serializes correctly to JSON when persisting the `WorkflowInstance` state.
    *   **Error Handling:** `read_scratchpad` should return a distinct, predictable message (e.g., `"[SCRATCHPAD_KEY_NOT_FOUND]"`) if the key doesn't exist, rather than throwing an error, allowing the LLM to handle the missing information gracefully. The tool handler should check for this specific return value.
    *   **Tool Result Content:** The success message for `write_scratchpad` is mostly for debugging/tracing; consider making it very minimal in the content returned to the LLM to save tokens, maybe just `"[SCRATCHPAD_WRITE_OK]"`.
    *   **Prompt Injection:** While values are strings, be mindful that an agent could write a value intended to be interpreted as instructions or even a tool call by a *later* agent reading it. The prompts for agents reading from the scratchpad should be clear that the values are data, not instructions.

**Conclusion: Enabling Basic Collaboration**

The Workflow Scratchpad provides a simple yet effective mechanism for basic state sharing and communication between agents in a sequential or conditional workflow. By implementing it as a key-value store managed by the Orchestrator and accessed by agents via dedicated tools (`read_scratchpad`, `write_scratchpad`), we enable more sophisticated workflows where agents can pass auxiliary data, configuration flags, or accumulated results without overloading the primary input channel. While limited initially (strings only, convention-based keys, no concurrency control), it lays the groundwork for more complex inter-agent communication and shared knowledge management in future iterations of the Cline Workbench.

---

**Next Up (Part 38 - Revisiting):** We previously discussed **Advanced Evaluation & Validation**. With the core orchestration and communication pieces in place, we can now more concretely revisit how to integrate static analysis, performance, and security metrics into the `verifyResult` phase of the evaluation framework.