Okay, here is the fourth document for the Project Manager: the **Phased Implementation Plan** for evolving Cline into the Universal Code Orchestrator Workbench.

This document breaks down the overall vision into manageable stages, outlining goals, key features, and deliverables for each phase.

---

**Document 4: Phased Implementation Plan - Cline Universal Code Orchestrator Workbench**

**Purpose:** To provide a structured, phased roadmap for the development effort required to evolve the existing Cline extension into the proposed Universal Code Orchestrator Workbench. This plan outlines key milestones, features per phase, and dependencies, facilitating project management, resource allocation, and incremental delivery of value.

**Overall Principles:**

*   **Iterative Development:** Deliver functional increments in phases, allowing for feedback and course correction.
*   **Foundation First:** Prioritize core architectural refactoring and essential services before implementing advanced features.
*   **User Control:** Maintain human-in-the-loop oversight, especially for adaptation and modification features, introducing autonomy cautiously and optionally.
*   **Integrated Evaluation:** Build in performance measurement and validation mechanisms from early phases to guide development.
*   **Backward Compatibility (Initial):** Aim to keep the basic single-task chat functionality operational during initial refactoring phases where feasible.

---

**Phase 1: Foundation & Core Refactoring**

*   **Goal:** Decouple core agent execution logic from task/state management, establish basic blueprint definition and storage.
*   **Key Features/Components:**
    *   **`AgentInstance` Class (V1):** Refactor core execution loop (API call, parse, tool exec check, result format) from `Task` into `src/core/agent/AgentInstance.ts`. Accepts blueprint config & context segment as input, returns `AgentStepResult`. Stateless regarding long-term history. (Ref: Part 1)
    *   **`AgentBlueprint` Schema (V1):** Define initial TypeScript interface/JSON schema in `src/shared/workbench/types.ts` (ID, version, description, static `baseSystemPrompt`, `allowedTools` list, partial `defaultModelConfig`). (Ref: Part 2)
    *   **`BlueprintService` (V1):** Implement service in `src/services/workbench/blueprints/` to load/validate/save blueprint JSON files from workspace (`.cline/blueprints/`) and global (`~/Documents/Cline/Blueprints/`) locations (local overrides global). Implement basic file-based version archiving for global blueprints. (Ref: Part 4)
    *   **Protobuf/gRPC:** Define initial `BlueprintService.proto`, generate code (`npm run protos`), implement basic gRPC handlers (`src/core/controller/workbench/blueprints/`) and client calls (`webview-ui/`) for listing/getting blueprints.
    *   **UI (Placeholder):** Create basic "Workbench" view container in `webview-ui/` with a simple "Blueprints" tab showing a read-only list fetched via gRPC.
*   **Key Challenges Addressed:** Core architectural refactoring risk (mitigated by starting adaptation alongside original `Task`), definition storage.
*   **Deliverables:** Functional `AgentInstance` capable of single-step execution, `BlueprintService` for managing definitions on disk, basic UI list view. Existing Cline chat functionality remains operational (still using `Task`).
*   **Dependencies:** None beyond existing codebase. Enables Phase 2.

---

**Phase 2: Basic Orchestration & Observability**

*   **Goal:** Enable execution of simple, linear workflow sequences and provide foundational monitoring capabilities.
*   **Key Features/Components:**
    *   **`WorkflowDefinition` Schema (V1 - Linear):** Define schema for linear sequences (`steps: { stepId, blueprintId, ... }[]`) in `src/shared/workbench/types.ts`. (Ref: Part 3)
    *   **`WorkflowOrchestrator` (V1):** Implement service in `src/services/workbench/orchestration/` to execute linear workflows. Manages `WorkflowInstance` state (incl. `conversationHistory`), instantiates `AgentInstance` per step, passes `outputContent` between steps, handles basic error halting. Implements state persistence to disk after each step (`instance_state.json`). (Ref: Part 3)
    *   **`WorkflowService` (V1):** Manage loading/saving of workflow definition files (JSON/YAML) similar to `BlueprintService`.
    *   **Structured Logging Service:** Implement `WorkflowLogger.ts` (`src/services/logging/`) and define core `TraceLog` event schemas/protobufs. Integrate basic logging calls into Orchestrator and AgentInstance for key lifecycle events. Store logs in SQLite (`evals.db` or new `workbench.db`). (Ref: Part 5)
    *   **UI (Basic Monitor & Logs):**
        *   Enhance "Workflows" tab UI to allow defining simple linear workflows (text editor + Mermaid preview). Add "Run" button triggering the Orchestrator.
        *   Implement basic "Instances" view UI showing list of recent instances (ID, status, timestamps) fetched via gRPC.
        *   Implement basic "Trace Viewer" UI panel, displayed when an instance is selected, showing chronological, filterable logs fetched via gRPC. (Ref: Part 5)
    *   **Protobuf/gRPC:** Add services/methods for Workflows, Instances, and fetching Logs. Run `npm run protos`.
*   **Key Challenges Addressed:** Workflow execution, state persistence, basic observability.
*   **Deliverables:** Ability to define and execute simple linear workflows, persistence of instance state, basic monitoring and log viewing UI.
*   **Dependencies:** Phase 1 completed. Enables Phase 3.

---

**Phase 3: Communication, Conditionals & Initial Evaluation Integration**

*   **Goal:** Introduce basic inter-agent communication, conditional workflow logic, and integrate richer evaluation metrics.
*   **Key Features/Components:**
    *   **Workflow Scratchpad:** Add `scratchpad: Record<string, string>` to `WorkflowInstance`. Implement `read_scratchpad`/`write_scratchpad` tools in `AgentInstance` using callbacks to Orchestrator. Update Orchestrator persistence. (Ref: Part 6)
    *   **Conditional Workflows:** Extend `WorkflowDefinition` schema and Orchestrator logic to support `onSuccess`/`onFailure` transitions, including `errorCode` matching. Pass error context via Scratchpad. (Ref: Part 7)
    *   **Richer Evaluation Metrics:**
        *   Define `benchmark.yaml` schema V1 (specifying optional lint/complexity/SAST tools).
        *   Enhance `evals/` framework adapters (`verifyResult`) to optionally run specified tools (initially via `execa`, planning for containerization) and parse common outputs (ESLint JSON, basic complexity).
        *   Extend evaluation DB schema and `storeTaskResult` to store these new metrics. (Ref: Part 8)
    *   **UI Updates:**
        *   Enhance Workflow Editor UI (text view) to support conditional schema. Update Mermaid visualization for branches.
        *   Enhance Trace Viewer to clearly show branching decisions and Scratchpad operations.
        *   Enhance Evaluation Reporting UI (or `report` command) to display new metrics.
*   **Key Challenges Addressed:** Limited workflow logic, basic evaluation metrics.
*   **Deliverables:** Ability to define/execute conditional workflows, agents can share state via Scratchpad, evaluation framework captures basic code quality metrics.
*   **Dependencies:** Phase 2 completed. Enables Phase 4.

---

**Phase 4: Supervised Adaptation Loop & Blueprint Versioning**

*   **Goal:** Implement the first human-in-the-loop adaptation cycle using Meta-Agents.
*   **Key Features/Components:**
    *   **Blueprint Versioning:** Implement robust versioning for local `AgentBlueprint` files (using dedicated Git repo in `.cline/definitions.git`). Update `BlueprintService` to manage versions (save, list, get specific). (Ref: Part 4)
    *   **Meta-Agents (V1 - Suggestion Only):**
        *   Implement `EvaluatorAgent` blueprint/logic (takes instance IDs, queries DB aggregates/samples, outputs structured report).
        *   Implement `PromptOptimizerAgent` blueprint/logic (takes report + blueprint, outputs structured `MetaAgentSuggestion` with prompt diff and rationale, stored in DB via `SuggestionService`). Focus on constrained prompts initially. (Ref: Part 9)
    *   **Suggestion Service & DB:** Implement `SuggestionService` and `MetaAgentSuggestions` DB table.
    *   **Suggestion Review UI:** Build the dedicated view in the Workbench UI. Display pending suggestions, diffs, rationale. Implement "Apply" (creates new blueprint version via `BlueprintService`) and "Ignore" buttons (updates suggestion status). (Ref: Part 9)
    *   **Validation Trigger:** Implement "Apply & Validate" button which, after applying the change, triggers a call to a new `EvaluationService` (or similar) that invokes the `evals/` framework to run specific benchmarks against the *new* blueprint version. (Requires `evals/` CLI to be callable programmatically or via command line).
*   **Key Challenges Addressed:** Lack of automated improvement mechanisms, managing blueprint evolution.
*   **Deliverables:** Ability for users to trigger analysis of workflow runs, review AI-generated suggestions for prompt improvements, apply valid suggestions to create new blueprint versions, and manually trigger validation runs.
*   **Dependencies:** Phase 3 completed. Enables Phase 5+.

---

**Phase 5+: Advanced Features & Research (Parallel, Iterative)**

*   **Goal:** Implement more advanced Workbench capabilities and create a platform for ongoing research. This phase is less linear and involves parallel exploration.
*   **Potential Features/Components:**
    *   **Parallel Workflow Execution:** Implement fork-join parallelism in Orchestrator, including resource management (implicit file-write locks, dedicated terminals). (Ref: Part 32)
    *   **Shared Knowledge Base (SKB):** Phased implementation - start with Graph DB for code semantics populated by `IndexingService` using Tree-sitter, add Vector DB for semantic search later. Implement `KnowledgeService` and `query_knowledge_base` tool. (Ref: Part 37)
    *   **External Knowledge Integration:** Build core mediated MCP servers (PackageInfo, DocsFetcher, Status). Implement query sanitization. Integrate freshness/provenance tracking in SKB and UI. (Ref: Part 40)
    *   **Advanced Meta-Agents:** Implement `DebuggerAgent` (log analysis), supervised `ArchitectAgent` (blueprint generation). Explore more sophisticated `PromptOptimizerAgent` techniques.
    *   **Enhanced UI/UX:** Visual workflow editor, advanced observability dashboards, Knowledge Explorer UI.
    *   **Constrained Autonomous Optimization:** Implement opt-in auto-apply loop for *validated* suggestions with robust safeguards and monitoring. (Ref: Part 34)
    *   **Advanced Evaluation:** Containerized evaluation environments, more analysis tool parsers, richer metrics (functional equivalence research).
*   **Key Challenges Addressed:** Scalability, true agent collaboration, knowledge consistency, reliable autonomous adaptation, advanced evaluation, usability at scale.
*   **Deliverables:** Incremental delivery of advanced features based on research priorities and feasibility. Enhanced platform capabilities for complex automation and agent experimentation.
*   **Dependencies:** Phase 4 provides the core adaptation loop infrastructure. Subsequent features build upon this and the evolving SKB.

---

**Cross-Cutting Concerns (Addressed Throughout)**

*   **Testing:** Unit and integration tests must be added/updated for all new services, components, and features in each phase. `evals/` framework used for end-to-end validation.
*   **Documentation:** Update `docs/` with information on new Workbench features, schemas, and usage as they are developed. Add inline code comments.
*   **Security:** Continuously assess security implications of new features (inter-agent communication, external access, meta-agent capabilities). Apply security best practices.
*   **UI/UX Refinement:** Continuously gather feedback and refine the Workbench UI for clarity, usability, and performance.
*   **Performance:** Monitor and optimize performance of orchestration, logging, indexing, and UI rendering throughout development.

**Conclusion**

This phased plan provides a structured path to transform Cline into the Universal Code Orchestrator Workbench. It prioritizes building a solid foundation (core refactoring, basic orchestration, definition storage, logging) before layering on more complex capabilities like conditional logic, inter-agent communication, richer evaluation, and supervised adaptation loops. Advanced features like parallelism, the full Shared Knowledge Base, external knowledge integration, and constrained autonomy are treated as later-stage goals or ongoing research areas built upon the established platform. This iterative approach allows for incremental value delivery, risk management, and continuous integration of new discoveries and research findings.