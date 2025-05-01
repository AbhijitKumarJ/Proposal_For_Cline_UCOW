Okay, here is the second document for the Project Manager: **Detailed Description of Key Modules and Files** for the Cline Universal Code Orchestrator Workbench.

This document elaborates on the purpose and key responsibilities of the major new or significantly modified directories and files outlined in the target source code layout.

---

**Document 2: Detailed Module & File Descriptions - Cline Universal Code Orchestrator Workbench**

**Purpose:** To provide a detailed functional description of the core modules and key files involved in the proposed Cline Workbench architecture. This clarifies the role and responsibilities of each component, aiding in impact analysis and task allocation.

---

**I. Core Extension (`src/`)**

1.  **`src/core/agent/AgentInstance.ts` (New/Refactored)**
    *   **Purpose:** Encapsulates the logic for executing a *single turn* of an AI agent based on a provided configuration (derived from an `AgentBlueprint`). It is designed to be largely stateless regarding long-term conversation history.
    *   **Key Responsibilities:**
        *   Accepting configuration (system prompt, allowed tools, model settings) and input context (current relevant history segment, previous step's output).
        *   Constructing the final prompt for the LLM API call.
        *   Making the API call via the appropriate `ApiHandler`.
        *   Streaming and parsing the LLM response (`parseAssistantMessage`).
        *   Handling `<thinking>` tags.
        *   Identifying requested tool calls.
        *   *Validating* if the requested tool is in the `allowedTools` list provided during configuration.
        *   Requesting user approval for tools via a callback to the orchestrator/controller.
        *   Executing *allowed* and *approved* tools by calling relevant integration/service functions (passed as dependencies).
        *   Formatting tool results or errors.
        *   Returning a structured `AgentStepResult` containing success status, output content for the next step, the full assistant message generated, any tool result/error, and step-specific metrics.
    *   **Dependencies:** API Handlers, Tool parsing logic, Integration functions (for tool execution), Logging Service, Approval callback mechanism.
    *   **Excludes:** Does *not* manage overall workflow state, conversation history persistence, or Checkpoint creation (these belong to the Orchestrator or Task wrapper).

2.  **`src/core/controller/workbench/` (New Folder Structure)**
    *   **Purpose:** Houses the gRPC service handlers specifically for Workbench-related functionality requested by the UI. Acts as the bridge between UI requests and backend Workbench services.
    *   **Key Files:**
        *   `blueprint_service.ts`: Handles gRPC requests related to listing, getting, saving, and versioning `AgentBlueprints` by calling the `BlueprintService`.
        *   `workflow_service.ts`: Handles gRPC requests for `WorkflowDefinitions`.
        *   `instance_service.ts`: Handles gRPC requests for listing/getting `WorkflowInstance` status and potentially triggering actions like start/cancel.
        *   `suggestion_service.ts`: Handles gRPC requests for listing/managing `MetaAgentSuggestions`.
        *   `logging_service.ts` / `trace_service.ts`: Handles gRPC requests for fetching structured logs for specific workflow instances.
        *   `knowledge_service.ts` (Future): Handles gRPC requests for querying the SKB.
        *   `methods.ts` (Auto-Generated): Registers all handlers within this directory.
        *   `index.ts`: Exports the main request handler for this service group.
    *   **Dependencies:** Relevant Workbench Services (Blueprint, Workflow, Orchestrator, Logging), Protobuf definitions/conversions.

3.  **`src/services/workbench/` (New Folder Structure)**
    *   **Purpose:** Contains the core backend services driving the Workbench functionality, distinct from the UI interaction handled by the controller.
    *   **Key Files:**
        *   `orchestration/WorkflowOrchestrator.ts`: The heart of the Workbench. Manages workflow instance lifecycles, executes steps using `AgentInstance`, handles state persistence, manages the Scratchpad, implements conditional logic, and coordinates resource locking (future).
        *   `blueprints/BlueprintService.ts`: Manages loading, validation, saving, and versioning of `AgentBlueprint` definitions from global and local storage.
        *   `workflows/WorkflowService.ts`: Manages loading, validation, and saving of `WorkflowDefinition` files.
        *   `suggestions/SuggestionService.ts`: Manages storage and retrieval of `MetaAgentSuggestions`.
        *   `knowledge/KnowledgeService.ts` (Future): Provides the API for querying the hybrid Shared Knowledge Base.
        *   `indexing/IndexingService.ts` (Future): Runs asynchronously to populate the SKB by parsing code, logs, etc.
        *   `storage/` (Future): Adapters for interacting with specific SKB backends (GraphDB, VectorDB interfaces).
        *   `validation/BenchmarkRunner.ts` (New): Service invoked by the Orchestrator or UI to trigger specific `evals/` benchmark runs for validating blueprint changes. Interacts with the `evals/cli` or its refactored components.
    *   **Dependencies:** `AgentInstance`, Storage utils, DB access, potentially `evals/` components.

4.  **`src/services/logging/WorkflowLogger.ts` (New)**
    *   **Purpose:** Provides a dedicated, asynchronous service for writing structured `TraceLog` events to the persistent store (initially SQLite).
    *   **Key Responsibilities:**
        *   Receiving structured log event objects.
        *   Serializing the event payload (likely to JSON).
        *   Writing the record to the `TraceLogs` database table efficiently (potentially batching writes).
        *   Handling database write errors gracefully without crashing the caller.
    *   **Dependencies:** DB access layer.

5.  **`src/shared/workbench/types.ts` (New)**
    *   **Purpose:** Central location for defining shared TypeScript interfaces and types used across the Workbench backend and frontend (e.g., `AgentBlueprint`, `WorkflowDefinition`, `WorkflowInstance`, `AgentStepResult`, `MetaAgentSuggestion`, `LogEventPayloads`).
    *   **Key Responsibilities:** Ensure type consistency between Extension Host and Webview for Workbench-related data structures.

6.  **`proto/*.proto` & `src/shared/proto/*.ts` (Expanded)**
    *   **Purpose:** Define the contracts for gRPC communication between the Webview UI and the Extension Host Controller for all Workbench functionalities.
    *   **Key Responsibilities:** Specify services (e.g., `BlueprintService`, `WorkflowService`), methods (`ListBlueprints`, `StartWorkflow`), and message types (`BlueprintInfo`, `WorkflowInstanceStatus`). Ensure generated TypeScript code (`src/shared/proto/`) provides type safety for client and server implementations.

**II. Webview UI (`webview-ui/`)**

1.  **`webview-ui/src/components/workbench/` (New Folder)**
    *   **Purpose:** Houses all React components specific to the new Workbench UI views.
    *   **Key Files:**
        *   `WorkbenchRoot.tsx`: Top-level component for the Workbench view container, likely managing tabs or navigation between the different sections.
        *   `DashboardView.tsx`: Displays summaries and quick actions.
        *   `BlueprintListView.tsx`, `BlueprintEditorView.tsx`: Components for managing Agent Blueprints.
        *   `WorkflowListView.tsx`, `WorkflowEditorView.tsx`: Components for managing Workflow Definitions (including text editor and Mermaid visualization).
        *   `InstanceMonitorView.tsx`: Displays running/completed workflow instances.
        *   `TraceViewer.tsx`: Component for displaying structured logs (likely using Virtuoso and CodeBlock).
        *   `SuggestionReviewView.tsx`: Component for displaying and acting on Meta-Agent suggestions (includes diff viewer).
    *   **Dependencies:** React, VS Code Toolkit components, gRPC client services, `ExtensionStateContext`.

2.  **`webview-ui/src/context/ExtensionStateContext.tsx` (Modified)**
    *   **Purpose:** Extended to manage the state received from the backend related to Workbench entities (lists of blueprints, workflows, instances, suggestions).
    *   **Key Responsibilities:** Listen for `state` updates containing Workbench data, provide this data to UI components via the `useExtensionState` hook, potentially manage local UI state related to the Workbench (e.g., selected instance ID).

3.  **`webview-ui/src/services/grpc-client.ts` (Expanded)**
    *   **Purpose:** The generic gRPC client factory is used to create *new* typed client instances for the Workbench-specific Protobuf services (BlueprintService, WorkflowService, TraceService, etc.).
    *   **Key Responsibilities:** Provide type-safe functions for the UI components to call backend Workbench gRPC methods via the `postMessage` bridge.

**III. Evaluation Framework (`evals/`)**

1.  **`evals/benchmarks/` (New Structure)**
    *   **Purpose:** Store benchmark definitions and associated data/scripts.
    *   **Key Files:** `benchmark.yaml` (or similar) per task/benchmark, defining not just the task but also required analysis tools, configurations, and performance scripts.
2.  **`evals/cli/src/adapters/types.ts` (Modified)**
    *   **Purpose:** Update `VerificationResult` to `MultiFacetVerificationResult` to accommodate structured metrics for quality, security, performance.
3.  **`evals/cli/src/adapters/*.ts` (Modified)**
    *   **Purpose:** Update adapter implementations' `verifyResult` methods to:
        *   Read the extended `benchmark.yaml`.
        *   Execute specified analysis tools (potentially within a container).
        *   Call new parser utilities.
        *   Return the structured `MultiFacetVerificationResult`.
4.  **`evals/cli/src/utils/parsers.ts` (New)**
    *   **Purpose:** Centralize reusable functions for parsing the output of common static analysis, SAST, and complexity tools into structured metric objects.
5.  **`evals/cli/src/utils/container.ts` (New)**
    *   **Purpose:** Encapsulate logic for starting, running commands within, and stopping containerized environments (Docker/Podman/Devcontainer) used for evaluation runs.
6.  **`evals/cli/src/db/schema.ts` & `index.ts` (Modified)**
    *   **Purpose:** Extend the SQLite schema and database access layer to store the new, richer evaluation metrics.
7.  **`evals/cli/src/commands/report.ts` (Modified)**
    *   **Purpose:** Update report generation logic to query and display the newly added metrics in a configurable way.

**IV. User Configuration Files (`.cline/`, `~/Documents/Cline/`)**

1.  **`.cline/blueprints/`, `.cline/workflows/` (New - Workspace Local):**
    *   **Purpose:** Store project-specific Agent Blueprints and Workflow Definitions as JSON/YAML files. These override global definitions with the same ID. Managed via Git for versioning within the project (potentially a dedicated shadow repo).
2.  **`~/Documents/Cline/Blueprints/`, `~/Documents/Cline/Workflows/` (New - Global):**
    *   **Purpose:** Store user-level, reusable Blueprints and Workflows accessible across all projects. Versioning is simpler file archiving initially.
3.  **`~/.config/Code/User/globalStorage/saoudrizwan.claude-dev/settings/cline_mcp_settings.json` (Existing):**
    *   Continues to store MCP server configurations.
4.  **`.clinerules` / `~/Documents/Cline/Rules/` (Existing):**
    *   Continue to store user-defined prompting rules.

---

This detailed breakdown clarifies the roles and interactions of the key components in the proposed Workbench architecture. It provides a clearer map for distributing development work and analyzing the impact of these significant changes on the existing Cline codebase.