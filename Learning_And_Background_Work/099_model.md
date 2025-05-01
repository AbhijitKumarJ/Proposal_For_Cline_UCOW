Okay, here is the detailed blog post for Part 2 of the "Evolving Cline" series, focusing on defining the Agent Blueprints and related schemas for the Workbench.

---

**Blog Series:** *Evolving Cline: Building the Universal Code Orchestrator Workbench*

**Part 2: Defining the Actors - Agent Blueprints & Schemas**

**Welcome back, Workbench Architects!**

In Part 1, we took the crucial first step in evolving Cline by refactoring the core agentic loop out of the monolithic `Task` class into a more focused, reusable `AgentInstance`. This new class is designed to execute a *single step* based on a given configuration and context, returning its result without managing long-term history.

But how do we *configure* these `AgentInstance` objects? How do we define different *types* of agents – planners, coders, testers – each with its own specific instructions, capabilities, and potentially preferred models? We need a way to formally describe these agent types.

This post focuses on designing and implementing the **Agent Blueprint** – the template or definition used to instantiate specialized agents within our Universal Code Orchestrator Workbench. We'll follow our design session format with **A (Ideator)**, **B (Critic)**, and **C/D (Implementers)** to arrive at a practical schema and storage mechanism.

---

**Session Start: Templating Agent Behavior**

**Coder A (Ideator):** Our `AgentInstance` needs configuration. We can't just pass a massive object with all possible settings every time. We need a reusable, persistent definition for *types* of agents. I propose the **`AgentBlueprint`**. Think of it like a class definition for our agents. It would capture the core characteristics needed to instantiate an agent for a specific role:
1.  **`id`:** A unique identifier (e.g., `python-pytest-writer`, `react-refactor-agent`).
2.  **`description`:** Human-readable explanation of the agent's purpose.
3.  **`baseSystemPrompt`:** The core instructions defining the agent's persona, goals, and constraints. This could potentially include placeholders for runtime context injection (e.g., `{{input_data}}`, `{{current_file_path}}`).
4.  **`allowedTools`:** An explicit list of tool names (built-in like `read_file` or MCP like `my-server/my-tool`) that *this specific agent type* is permitted to use. This enforces role specialization.
5.  **`defaultModelConfig`:** The preferred `ApiConfiguration` (provider, model ID, potentially specific API keys if needed *only* for this agent type, though generally keys should be global) to use when running this blueprint. Can be overridden at runtime by the workflow.
6.  **`contextStrategy` (Future):** A way to specify how context should be managed for this agent (e.g., 'standardTruncation', 'ragEnhanced', 'summarization'). Start with a default.
7.  **`outputSchema` (Future):** A JSON Schema defining the expected structure of the agent's final output, allowing for structured data passing between workflow steps.

**Coder B (Critic):** Okay, a blueprint concept makes sense for reusability and specialization. Questions:
1.  **Storage Format & Location:** Where do these live? Plain JSON/YAML files seem best for version control and human readability. Should they be workspace-local (`.cline/blueprints/`) for project-specific agents, global (`~/Documents/Cline/Blueprints/`) for reusable ones, or both? How do we handle naming conflicts?
2.  **Prompt Templating:** Basic string substitution for `{{placeholders}}` is simple, but what if we need more complex logic, like conditionally including prompt sections based on input? That requires a more sophisticated templating engine. Let's keep it simple initially.
3.  **Tool Specificity:** Just listing allowed tool *names* is a start. But what if we want to constrain *parameters* for a tool only for a specific agent? (e.g., `CodeReviewerAgent` can use `read_file` but *not* `write_to_file`). The current proposal only allows/disallows entire tools.
4.  **Schema Validation:** We absolutely need schema validation for these blueprint files. Users (or agents generating blueprints) *will* make mistakes in the structure. We need to catch these early.

**Coder A (Ideator):** Agreed.
1.  **Storage:** Both global and local. Workspace `.cline/blueprints/` overrides global definitions with the same `id`. This allows project-specific overrides of standard agents.
2.  **Templating:** Start with *no* templating in `baseSystemPrompt`. The `WorkflowOrchestrator` will be responsible for constructing the *final* prompt by combining the `baseSystemPrompt` with step-specific input/context *before* passing it to `AgentInstance`. This keeps the blueprint static and shifts dynamic context injection to the orchestrator.
3.  **Tool Constraints:** You're right, just allowing/disallowing tools is coarse. Phase 1: Only allow/disallow entire tools. Phase 2: Consider adding an `allowedToolParams` map within the blueprint (e.g., `replace_in_file: { allow_diff: true, allow_content: false }`) – but this adds complexity. Let's defer fine-grained parameter control.
4.  **Schema:** Absolutely. We'll use TypeScript types internally and potentially generate a JSON Schema for validation and maybe even editor support (e.g., for a future UI editor).

**Coder B (Critic):** Okay, focusing on static blueprints without prompt templating, initially allowing only full tool enable/disable, storing as versioned JSON/YAML in local/global locations (local overrides global), and requiring schema validation seems like a solid V1 plan for defining agents.

---

**Implementation Details (Coder C & D)**

**(C outlines the initial implementation, D suggests refinements)**

**Coder C (Implementer):**
1.  **Define `AgentBlueprint` Interface (`src/shared/workbench/types.ts`):**
    ```typescript
    import { ToolUseName } from "@core/assistant-message/index";
    import { ApiConfiguration } from "@shared/api"; // Assuming this exists

    export interface AgentBlueprint {
        id: string; // e.g., "python-pytest-writer"
        version: number; // Simple integer version
        description: string;
        baseSystemPrompt: string; // Static prompt for this agent type
        allowedTools: ToolUseName[];
        defaultModelConfig: Partial<ApiConfiguration>; // Allow specifying only relevant parts, rest inherit from global
        // outputSchema?: any; // JSON Schema object - Future
        // contextStrategy?: string; // Future
    }
    ```
2.  **Storage Locations:**
    *   Define constants for global path (`await ensureBlueprintDirectoryExists()`) and local path (`path.join(cwd, '.cline/blueprints/')`).
3.  **`BlueprintService` (`src/services/workbench/BlueprintService.ts`):**
    *   `async loadBlueprints(): Promise<Map<string, AgentBlueprint>>`: Scans both global and local directories for `.json` files. Reads, validates against `AgentBlueprint` schema (using Zod or type guards). Handles versioning (loads latest `*.vMax.json` or `*.json`). Merges results, prioritizing local over global for conflicting IDs. Returns a map of `blueprintId` to the latest `AgentBlueprint` object.
    *   `async getBlueprint(id: string, version?: number): Promise<AgentBlueprint | undefined>`: Loads a specific blueprint, optionally a specific version.
    *   `async saveBlueprint(blueprint: AgentBlueprint): Promise<void>`: Validates input blueprint. Increments version number. Saves the *previous* version as `id.v(N-1).json` (if it exists). Saves the *new* version as `id.vN.json` (or `id.json` if v1). Saves to the *workspace* directory (`.cline/blueprints/`) if inside a workspace, otherwise to the global directory. Needs error handling for file writes.
    *   `async listBlueprintVersions(id: string): Promise<{ version: number; timestamp: Date }[]>`: Lists available versions for a blueprint ID.
4.  **Schema Validation:** Use Zod or manual type guards within `BlueprintService` methods to validate loaded JSON against the `AgentBlueprint` interface. Throw informative errors on validation failure.
5.  **gRPC Integration:**
    *   Define `BlueprintService.proto` with messages (`BlueprintInfo`, `BlueprintDefinition`) and methods (`ListBlueprints`, `GetBlueprint`, `SaveBlueprint`, `ListBlueprintVersions`).
    *   Run `npm run protos`.
    *   Implement gRPC handlers in `src/core/controller/workbench/blueprint_service.ts` that call the underlying `BlueprintService` methods.
    *   Implement client calls in `webview-ui/src/services/grpc-client.ts`.
6.  **`AgentInstance` Adaptation:** Modify its constructor or `executeStep` method to accept an `AgentBlueprint` object or its relevant parts (`baseSystemPrompt`, `allowedTools`, `modelConfig`) for configuration.

**Coder D (Refiner):**
*   **Refine C:**
    *   **Storage:** Using JSON is fine, but consider YAML for `baseSystemPrompt` for better readability/multiline handling. `BlueprintService` should support both.
    *   **Versioning:** Saving previous versions as `id.vN.json` is simple but could clutter directories. Consider a more robust approach long-term, maybe using the Checkpoint Git system itself to version control the `.cline/blueprints` directory, storing only the latest JSON in the directory but allowing checkout of previous versions via Git hash. For V1, `id.vN.json` is acceptable.
    *   **Validation:** Use a dedicated JSON Schema file generated from the TypeScript types. This schema can be used for runtime validation in `BlueprintService` *and* potentially provide IntelliSense if users edit blueprint files directly in VS Code (via `.vscode/settings.json`).
    *   **Error Handling:** `BlueprintService.loadBlueprints` should gracefully handle errors reading/parsing individual files (log error, skip file) rather than failing entirely. `saveBlueprint` needs robust file write error handling.
    *   **Tool Naming:** Ensure consistency between `ToolUseName` enum/type and the strings used in the `allowedTools` array within blueprints. Maybe generate the `ToolUseName` type *from* the system prompt's tool list during build?

**Conclusion: Defining the Building Blocks**

The `AgentBlueprint` provides the essential abstraction for defining specialized agents within the Cline Workbench. By storing these definitions persistently as structured, versioned files (JSON/YAML) and providing services to manage them, we create the reusable building blocks needed for orchestration. Starting with static prompts and whole-tool permissions keeps the initial implementation manageable, while the schema provides clear extension points for future enhancements like prompt templating or fine-grained tool parameter control. The focus on schema validation and clear storage locations (local overriding global) ensures a degree of robustness from the outset.

---

**Next Up (Part 3):** With agent definitions in place, we'll build the initial `WorkflowOrchestrator` service capable of running simple, linear sequences of these agents, focusing on state management and passing context between steps.