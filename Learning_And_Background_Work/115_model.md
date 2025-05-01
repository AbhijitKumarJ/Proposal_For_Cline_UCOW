Okay, here is the detailed blog post for Part 9 of the "Evolving Cline" series, focusing on implementing the first human-supervised adaptation loop using Meta-Agents.

---

**Blog Series:** *Evolving Cline: Building the Universal Code Orchestrator Workbench*

**Part 9: Closing the First Loop - Supervised Meta-Agents & Review**

**Welcome Back, Workbench Cultivators!**

We've laid extensive groundwork for our Cline Universal Code Orchestrator Workbench. We have defined specialized agents (`AgentBlueprint` - Part 2), built an orchestrator for sequential and conditional workflows (`WorkflowOrchestrator` - Part 3, 7), designed persistent storage for definitions (Part 4), established foundational monitoring and logging (Part 5), enabled basic inter-agent communication (`WorkflowScratchpad` - Part 6), and planned for richer evaluation metrics (Part 8).

Now, we arrive at a pivotal stage: closing the first **feedback loop**. How can the Workbench leverage the performance data we're collecting to actually *improve* its agents? While full autonomy is a distant goal (Part 34), we can implement a *supervised* loop where **Meta-Agents** analyze performance and *suggest* improvements, which a human user then reviews and approves.

This post details the design and implementation of this initial adaptation cycle, focusing on the `EvaluatorAgent`, the `PromptOptimizerAgent` (in its suggestion-only role), the necessary database extensions, and the critical "Suggestion Review" UI. Join **A (Ideator)**, **B (Critic)**, **C (Implementer)**, and **D (Refiner)** as we build the first learning mechanism into the Workbench.

---

**Session Start: From Data to Suggestion**

**Coder A (Ideator):** We're collecting rich evaluation data (Part 8), including pass/fail, resource usage, and potentially quality/security metrics, all linked to specific `AgentBlueprint` versions and workflow runs stored in our SQLite DB (Part 5). The next step is to *use* this data. I propose implementing our first two Meta-Agents:
1.  **`EvaluatorAgent`:** Takes a set of `workflowInstanceId`s (selected by the user or triggered after a benchmark run) as input. Its task is to query the `TraceLogs` and `Metrics` tables, aggregate the data, and produce a structured **Performance Report** (e.g., JSON). This report should highlight overall success rates, resource consumption statistics, and, crucially, identify common failure modes (e.g., Tool X failed Y times with error Z, Step Q frequently exceeded timeout).
2.  **`PromptOptimizerAgent`:** Takes a Performance Report *and* the specific `AgentBlueprint` (prompt, tool list) associated with the analyzed runs as input. Its initial, constrained goal is to identify potential improvements to the *system prompt* (specifically tool descriptions or guidelines) based on the observed failure patterns in the report. It outputs a structured **Suggestion** object containing the proposed change (as a text diff against the original prompt), a clear rationale linking the change to the observed failures, and potentially a confidence score (though confidence is hard to gauge reliably initially).

These Meta-Agents run as regular `AgentInstance` steps within a dedicated "Analysis Workflow" initiated by the user from the Workbench UI.

**Coder B (Critic):** This separation of concerns (Evaluate -> Suggest) makes sense. Key challenges:
1.  **Input Data Volume for Evaluator:** Passing potentially thousands of raw log entries to the `EvaluatorAgent`'s prompt will likely exceed context limits. It needs to query the DB for *aggregated* statistics or *representative samples* of failure logs, not the raw dump.
2.  **PromptOptimizer Reliability:** Prompting an LLM to analyze another LLM's failures (via logs/reports) and suggest effective prompt changes is highly non-trivial meta-prompting. The suggestions might be irrelevant, incorrect, or even detrimental. How do we ensure the quality of suggestions before bothering the user?
3.  **Suggestion Format:** How exactly is the "structured Suggestion object" represented? Just a text diff isn't enough; we need the rationale, source analysis, target blueprint/version, etc., stored persistently.
4.  **User Review Burden:** If Meta-Agents generate many low-quality suggestions, the "Suggestion Review" UI becomes a chore, not a benefit. We need a way to filter or prioritize suggestions.

**Coder A (Ideator):** We need to manage the inputs and outputs carefully:
1.  **Evaluator Input:** The `EvaluatorAgent` doesn't get raw logs. Instead, the Orchestrator (or a helper service) pre-queries the DB based on the selected `workflowInstanceId`s and provides *aggregated* data and *samples* of common errors as context within its prompt. For example: "Analyze these results for Blueprint 'X' v1 run on Benchmark 'Y': Success Rate: 65%. Most Common Failure: Tool 'replace_in_file' failed 15 times with error 'SEARCH block does not match'. Example error log: [...]".
2.  **Optimizer Reliability:** Start with *highly constrained* prompts for the `PromptOptimizerAgent`. Focus on one specific failure mode at a time. Example prompt: "Given that Tool 'T' frequently fails with error 'E' [log sample], suggest a *single sentence addition* to the description of Tool 'T' in this prompt [prompt snippet] to clarify usage regarding parameter 'P'. Output *only* the suggested sentence." This limits the scope and makes validation easier. Forget confidence scores for V1; all suggestions require human review.
3.  **Suggestion Format:** Define a `MetaAgentSuggestion` schema (TypeScript/Protobuf) with fields like `suggestionId`, `timestamp`, `sourceMetaAgentId`, `targetBlueprintId`, `targetBlueprintVersion`, `changeType` ('prompt_modification'), `proposedDiff` (standard text diff format), `rationale` (text from LLM), `status` ('pending_review'), `validationStatus` ('unvalidated'). Store these in a dedicated DB table.
4.  **User Review Burden:** Initially, users manually trigger the analysis workflow, controlling the frequency. The Suggestion Review UI should allow sorting/filtering (e.g., by target blueprint, change type). We only present suggestions with `status: 'pending_review'`.

**Coder B (Critic):** Okay, pre-aggregating data for the `EvaluatorAgent`, severely constraining the `PromptOptimizerAgent`'s task initially, using a structured DB table for suggestions, and relying on manual triggering addresses the key concerns for a V1 supervised loop. The focus is on *generating potentially useful hypotheses* for the human to evaluate, not on automated decision-making.

**(A and B agree on the supervised, human-reviewed suggestion loop with constrained Meta-Agents and structured data.)**

---

**Implementation Details (Coder C & D)**

**(C outlines the initial implementation, D suggests refinements)**

**Coder C (Implementer):**
1.  **Database Schema Extension (`evals/cli/db/schema.ts` or separate Workbench DB):**
    *   Add `MetaAgentSuggestions` table: `suggestionId TEXT PRIMARY KEY`, `timestamp INTEGER`, `sourceMetaAgentId TEXT`, `targetBlueprintId TEXT`, `targetBlueprintVersion INTEGER`, `changeType TEXT`, `proposedDiff TEXT`, `rationale TEXT`, `status TEXT`, `validationStatus TEXT`, `validationRunIds TEXT`. Index relevant fields like `status` and `targetBlueprintId`.
2.  **Meta-Agent Blueprints:**
    *   Create `AgentBlueprint` JSON/YAML files for `EvaluatorAgent` and `PromptOptimizerAgent`.
    *   **`EvaluatorAgent` Prompt:** Instruct it to parse structured input (aggregated stats, error samples) and output a structured JSON `PerformanceReport`. Define its allowed tools (likely just `read_scratchpad`, `write_scratchpad`, maybe a specialized `query_evaluation_db` tool).
    *   **`PromptOptimizerAgent` Prompt:** Highly constrained prompt (as discussed by A/B). Input includes `PerformanceReport` JSON and target `AgentBlueprint` content. Its primary tool is `propose_prompt_change(targetBlueprintId, targetVersion, suggestedPromptDiff, rationale)`.
3.  **New Tools:**
    *   Implement `propose_prompt_change` tool handler in `AgentInstance`. This handler simply writes the suggestion details to the `MetaAgentSuggestions` DB table via the `WorkflowOrchestrator` or a `SuggestionService`.
    *   (Optional) Implement `query_evaluation_db` tool if needed by `EvaluatorAgent`, which calls backend functions to get aggregated data.
4.  **`WorkflowOrchestrator` Modifications:**
    *   Add logic to trigger "Analysis Workflows" (which use Meta-Agents). This could be initiated via a new gRPC call from the UI.
    *   Needs to handle the input preparation for `EvaluatorAgent` (querying DB, aggregating data).
    *   Needs to handle the output of `PromptOptimizerAgent` (calling the service to store the suggestion).
5.  **Blueprint Versioning (`BlueprintService`):**
    *   Ensure `saveBlueprint` creates a *new version* when applying a change originating from an approved suggestion. It should likely take an optional `basedOnVersion` parameter.
    *   Implement `listVersions`, `getSpecificVersion` using the chosen versioning mechanism (Git for local, file archives for global - Part 4).
6.  **Suggestion Review UI (`webview-ui/src/components/workbench/SuggestionReview.tsx`):**
    *   **Data Fetching:** gRPC call `SuggestionService.listPendingSuggestions()`.
    *   **Display:** List suggestions. For each: show target blueprint/version, rationale. Use a diff viewer component (e.g., `react-diff-viewer`) to display `proposedDiff`.
    *   **Actions:**
        *   "Ignore": Updates suggestion `status` to 'ignored' via gRPC.
        *   "Apply":
            1.  Fetches the `targetBlueprintVersion` using `BlueprintService.getSpecificVersion`.
            2.  Applies the `proposedDiff` (needs a robust text patching utility).
            3.  Calls `BlueprintService.saveBlueprint` with the *new* content (this creates the next version).
            4.  Updates the suggestion `status` to 'applied'.
        *   "Apply & Validate": Does "Apply", then triggers an `evals/` validation run via the Orchestrator (calls `WorkflowOrchestrator.runValidation(newBlueprintVersion, benchmarkSuite)`). Updates suggestion `validationStatus` to 'running'. (Validation results update status later).
        *   "Edit & Apply": Opens the target blueprint in the Blueprint Editor, pre-filling with the *suggested* content, allowing user tweaks before saving (which triggers versioning). Updates suggestion status.
7.  **gRPC Services:** Define/implement `SuggestionService` (`listPendingSuggestions`, `updateSuggestionStatus`) and potentially extend `WorkbenchService` or `EvaluationService` for triggering analysis/validation workflows. Run `npm run protos`.

**Coder D (Refiner):**
*   **Refine C:**
    *   **Diff Format:** Standardize on the `diff` library's patch format for `proposedDiff` storage and application. Ensure the patching utility handles edge cases correctly.
    *   **Meta-Agent Input/Output:** Use strict JSON schemas for the `PerformanceReport` output by `EvaluatorAgent` and the arguments for `propose_prompt_change`. Validate these rigorously.
    *   **Suggestion Context:** The Suggestion Review UI should provide easy links back to the *evaluation runs* that led to the suggestion for user context.
    *   **Validation Workflow:** The "Apply & Validate" flow needs careful design. The `evals/` run should be triggered asynchronously. The UI needs to reflect the "validating" status, and receive updates (via polling or WebSocket/SSE) when validation completes (pass/fail/regressed).
    *   **UI Performance:** Loading and displaying many suggestions with potentially large diffs needs efficient rendering (virtualization if necessary).

**Conclusion: Enabling Supervised Learning**

Implementing this first adaptation loop, centered around Meta-Agents generating suggestions for human review, is a major step towards a self-improving Workbench. By structuring the analysis (`EvaluatorAgent`), constraining the initial suggestion generation (`PromptOptimizerAgent`), storing suggestions persistently, and providing a dedicated UI for review, application, and validation triggering, we create a controlled environment for learning. This human-in-the-loop approach allows us to leverage AI for analysis and hypothesis generation while retaining essential oversight. It establishes the core feedback mechanism upon which more sophisticated (and eventually, perhaps more autonomous) adaptation strategies can be built and rigorously tested.

---

**Next Up (Part 42 - Potential):** A final synthesis of the entire Workbench vision, reflecting on the journey, the key architectural pillars, the remaining research challenges, and the potential impact of such a tool on the future of software development.