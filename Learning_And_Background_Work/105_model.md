Okay, here is the detailed blog post for Part 4 of the "Evolving Cline" series, focusing on the persistent storage and management of Agent Blueprints and Workflow Definitions.

---

**Blog Series:** *Evolving Cline: Building the Universal Code Orchestrator Workbench*

**Part 4: Remembering Designs - Storing Blueprints & Workflows**

**Welcome Back, Workbench Crafters!**

In the previous parts, we laid the groundwork for our Universal Code Orchestrator Workbench. We refactored the core agent logic into a reusable `AgentInstance` (Part 1), defined the `AgentBlueprint` schema for specifying agent types (Part 2), and built a basic `WorkflowOrchestrator` to execute simple linear sequences of these agents (Part 3).

But where do these crucial `AgentBlueprints` and the `WorkflowDefinitions` they participate in actually *live*? How does the Workbench find them? How can we update them, share them, or revert to previous versions? A robust system needs a reliable way to store, retrieve, and manage these definitions.

This post focuses on designing and implementing the **persistent storage layer** for blueprints and workflows. We'll discuss storage formats, locations (global vs. workspace), versioning strategies, and the services needed to manage these definitions, following our design discussion format with **A (Ideator)**, **B (Critic)**, **C (Implementer)**, and **D (Refiner)**.

---

**Session Start: Where Do the Plans Live?**

**Coder A (Ideator):** Our Orchestrator needs to load Blueprints and Workflow Definitions by ID. These definitions aren't just temporary; they represent reusable patterns and configurations that users will want to save, modify, and potentially share. We need a dedicated storage solution. I propose:
1.  **Storage Format:** Use **JSON** or **YAML** files. They are human-readable, easily editable in VS Code, and work well with version control systems like Git. YAML might be slightly better for the multi-line `baseSystemPrompt` field in blueprints.
2.  **Storage Locations:** Support *both* global and workspace-local storage:
    *   **Global:** A dedicated directory within the user's system (e.g., `~/Documents/Cline/Blueprints/` and `~/Documents/Cline/Workflows/`). This stores reusable blueprints/workflows accessible across all projects. Use `ensureBlueprintDirectoryExists()` helpers (similar to Part 37's `ensureRulesDirectoryExists`).
    *   **Workspace-Local:** A `.cline/` directory within the current VS Code workspace (`.cline/blueprints/`, `.cline/workflows/`). This allows for project-specific agents or overrides of global definitions.
3.  **Loading Priority:** When loading a definition by ID, the system should check the workspace location *first*. If found, use that version. If not found locally, check the global location. This allows projects to customize standard agents.
4.  **Versioning:** Definitions need versioning, especially Blueprints, as Meta-Agents will be suggesting changes. A simple approach: embed a `version: number` field in the definition file. When saving an update, increment the version, save the *new* version as the primary file (e.g., `my-agent.json` or `my-agent.v2.json`), and optionally archive the *previous* version (e.g., rename it to `my-agent.v1.json`).

**Coder B (Critic):** This hybrid local/global storage with local override makes sense. YAML is probably better for prompts. The versioning approach is simple, but raises questions:
1.  **Versioning Granularity:** Just incrementing a number is basic. What if we want semantic versioning? What about tracking *what* changed between versions? Simply archiving the old file loses the diff history unless the user manually commits these definition files to their *own* Git repo.
2.  **Discovery & Naming:** How does the UI (or an agent trying to invoke a workflow) know which blueprints/workflows *exist* across both locations? Does the system scan both directories on startup? What prevents naming collisions beyond the local override?
3.  **Schema Evolution:** What happens if we change the `AgentBlueprint` or `WorkflowDefinition` schema itself in a future Cline update? How do we handle migrating existing user definition files?
4.  **Archiving Old Versions:** Renaming old versions (`id.v(N-1).json`) prevents data loss but could lead to directory clutter over time. Do we need a cleanup mechanism?

**Coder A (Ideator):** Good points. Let's refine:
1.  **Versioning:** Let's stick with the simple integer `version` field *within* the JSON/YAML file for V1. *However*, we should strongly consider leveraging the **Checkpoint system's shadow Git repository** (or a dedicated one just for `.cline/`) to manage the *history* of the definition files themselves within the workspace. Saving a blueprint/workflow update would then involve:
    *   Incrementing the `version` field in the file content.
    *   Writing the new file content.
    *   Creating a Git commit in the *definition* shadow repo, tagging it perhaps with `blueprint/my-agent/v2`.
    This gives us robust history, diffing, and rollback capabilities via Git, without cluttering the filesystem with `vN` files. Global definitions would still use the simpler file-based archiving initially.
2.  **Discovery:** Yes, a `BlueprintService` and `WorkflowService` will scan both locations on startup (and potentially watch for file changes) to build an in-memory catalog of available definitions (prioritizing local overrides). This catalog is what the UI queries.
3.  **Schema Evolution:** This is a classic software problem. We need to include a `$schemaVersion` field (separate from the blueprint's content `version`) in our definitions. When loading, the service checks this version. If it's older than the current supported version, it either attempts an automatic migration (if feasible and backward-compatible) or flags the definition as incompatible/requiring manual update in the UI.
4.  **Archiving:** If we use Git for versioning local definitions, archiving becomes less critical as history is preserved. For global definitions using file renaming, we can implement a simple cleanup policy later (e.g., keep only the last 5 versions).

**Coder B (Critic):** Using Git to version control the definitions within `.cline/` is a much more robust approach than simple file renaming. It provides diffs, history, and easy reverts. The `$schemaVersion` field is essential for future maintainability. The discovery mechanism sounds right.

**(A and B agree on JSON/YAML format, local/global storage with local override, Git-based versioning for local definitions, simple file archiving for global, schema versioning, and service-based discovery.)**

---

**Implementation Details (Coder C & D)**

**(C outlines the initial implementation, D suggests refinements)**

**Coder C (Implementer):**
1.  **Storage Locations & Setup:**
    *   Implement `ensureBlueprintDirectoryExists()` and `ensureWorkflowDirectoryExists()` in `src/core/storage/disk.ts` for both global (`~/Documents/Cline/...`) and workspace (`.cline/`) locations. Ensure `.cline/` is added to the project's `.gitignore` if possible (or advise users to).
2.  **Schema Definitions (`src/shared/workbench/types.ts`):**
    *   Finalize `AgentBlueprint` and `WorkflowDefinition` interfaces, ensuring they include `id: string`, `version: number`, and `$schemaVersion: string` (e.g., `"1.0"`).
3.  **`BlueprintService` & `WorkflowService` (`src/services/workbench/`):**
    *   **Constructor:** Takes storage paths. Initializes an in-memory `Map<id, LatestBlueprintInfo>` (and similar for workflows) populated by `loadDefinitions`. Sets up file watchers (`vscode.workspace.createFileSystemWatcher`) on the `.cline/blueprints` and `.cline/workflows` directories (and potentially global ones) to trigger reloads via `loadDefinitions`.
    *   **`async loadDefinitions()`:**
        *   Scans global and local directories for `.json` / `.yaml` files.
        *   Reads files, validates using Zod/type guards against the *current* `$schemaVersion`. Handles migration/warnings for older schema versions.
        *   For local definitions managed by Git: Uses `simple-git` on the dedicated `.cline/definitions.git` repo (needs initialization logic similar to Checkpoints) to find the *latest* version of each `id.json`/`id.yaml` file in the `main` branch (or equivalent). Parses the `version` field from the content.
        *   For global definitions: Finds the file with the highest `vN` suffix or the base file. Parses `version`.
        *   Populates the in-memory map, respecting local-over-global priority.
    *   **`getBlueprint(id)` / `getWorkflow(id)`:** Returns the latest definition from the in-memory map.
    *   **`saveBlueprint(blueprint)` / `saveWorkflow(definition)`:**
        *   Validates input.
        *   Determines save location (workspace `.cline/` if available, else global).
        *   **If Local (Git):**
            *   Reads current content using `git show HEAD:path/to/id.json`. Parses `version`.
            *   Sets `blueprint.version = currentVersion + 1`.
            *   Writes the *new* content to `path/to/id.json`.
            *   Uses `simple-git` to `add` the file and `commit` with a message like "Update blueprint [id] to vN". Consider adding a Git tag.
        *   **If Global (File Archive):**
            *   Reads current file `id.json` or highest `id.vN.json`. Parses `version`.
            *   Renames current latest file to `id.v(currentVersion).json`.
            *   Sets `blueprint.version = currentVersion + 1`.
            *   Writes the new content to `id.json`.
        *   Calls `loadDefinitions` asynchronously to update the in-memory map (or use watcher).
    *   **Versioning Methods:** Add `listVersions(id)`, `getSpecificVersion(id, version)` which interact with the Git history (local) or archived files (global).
4.  **Git for Definitions:**
    *   Create utility functions (perhaps in `src/services/workbench/DefinitionGit.ts`) to initialize (`git init`) and manage the dedicated Git repository within `.cline/definitions.git` (or similar). This repo *only* tracks the blueprint/workflow definition files within `.cline/`. It needs its own minimal Git configuration.
5.  **UI Integration:**
    *   Update Blueprint/Workflow Editors in `webview-ui/` to use new gRPC methods to list, load, and save definitions. Add UI elements to display and potentially select different versions.

**Coder D (Refiner):**
*   **Refine C:**
    *   **YAML Support:** Definitely use YAML for readability, especially for prompts. Use a library like `js-yaml` for parsing/stringifying.
    *   **Git Versioning Robustness:** Committing definition files to Git is good. Ensure commit messages are informative. Tagging versions (`blueprint/id/vN`) makes specific versions easy to check out. The `BlueprintService` needs robust error handling for Git operations. Consider if this definition repo should be part of the main Checkpoint shadow repo or entirely separate (separate seems cleaner). Ensure this definition repo itself is ignored by the main Checkpoint repo and the user's project Git.
    *   **Schema Validation:** Generate JSON Schema from TypeScript types (`typescript-json-schema`) during the build. Load this schema at runtime in the services for validation. Provide schema links in generated JSON/YAML files for editor IntelliSense.
    *   **Discovery/Loading Performance:** Scanning directories on startup might be slow if users have many global definitions. Consider caching the discovered list in VS Code global state and relying primarily on file watchers for updates after the initial scan.
    *   **UI for Conflicts:** If a local blueprint overrides a global one, the UI should clearly indicate this.
    *   **Migration:** For schema evolution, write explicit migration functions (`migrateV1toV2(data)`) within the services to handle transitions between known schema versions automatically where possible.

**Conclusion: Building a Persistent Foundation**

Storing Agent Blueprints and Workflow Definitions as human-readable, version-controlled files (JSON/YAML) provides the necessary persistence and manageability for the Cline Workbench. The hybrid approach of workspace-local (Git-versioned) and global (file-archived) storage offers flexibility, while prioritizing local definitions allows project-specific customization. Dedicated services (`BlueprintService`, `WorkflowService`) abstract the storage details and handle discovery, loading, saving, versioning, and schema validation. This persistent foundation is crucial for enabling users and agents to build upon past designs, track changes, and reliably instantiate agents for complex orchestration tasks.

---

**Next Up (Part 42 - Potential):** We could now tackle the implementation of the initial Meta-Agents (`EvaluatorAgent`, `PromptOptimizerAgent`) and the Suggestion Review UI, closing the first supervised adaptation loop.