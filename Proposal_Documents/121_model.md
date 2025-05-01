Okay, here is the first document for the Project Manager: the detailed target source code layout for the **Cline Universal Code Orchestrator Workbench**.

This document outlines the planned file and folder structure after the proposed evolution. It includes descriptions and details for each significant part, aiming to provide clarity for impact analysis.

---

**Document 1: Target Source Code Layout - Cline Universal Code Orchestrator Workbench**

**Purpose:** This document details the proposed final directory and file structure for the Cline project after its evolution into the Universal Code Orchestrator Workbench. It serves as a blueprint for development, aiding in understanding the scope, dependencies, and potential risks associated with the required changes.

**Overview:** The evolution involves significant refactoring of the core logic, introduction of new services for orchestration and knowledge management, expansion of shared types and communication protocols, and substantial additions to the webview UI to support workbench functionalities. The `evals/` framework will also be enhanced.

---

**Target Directory Structure & Descriptions:**

```
cline-cline/
│
├── .changeset/              # (Existing) Changeset files for versioning.
├── .clinerules/             # (Existing) Example global/local rules file (User-facing).
├── .github/                 # (Existing, Modified) CI/CD workflows, issue templates, CODEOWNERS.
│   ├── ISSUE_TEMPLATE/      # (Existing)
│   ├── scripts/             # (Existing, Modified) Helper scripts (e.g., coverage check, potentially build helpers).
│   └── workflows/           # (Existing, Modified) GitHub Actions (test, publish, changeset, PR checks). Needs updates for new test types/evals.
├── .husky/                  # (Existing) Git hooks (e.g., pre-commit for lint/format).
├── assets/                  # (Existing) Static assets like icons, documentation images.
├── docs/                    # (Existing, Expanded) User and developer documentation. Needs sections for Workbench features.
├── evals/                   # (Existing, Heavily Modified) Evaluation framework for benchmarking agent performance.
│   ├── README.md            # (Updated) Documentation for the evaluation framework.
│   ├── benchmarks/          # (New) Stores benchmark definitions (e.g., benchmark.yaml files) and potentially task data.
│   │   └── exercism/        # (Example) Specific benchmark data/configs.
│   │   └── swe-bench/       # (Example)
│   │   └── ...              # (Other benchmarks)
│   ├── cli/                 # (Existing, Heavily Modified) The command-line orchestrator for running evaluations.
│   │   ├── src/
│   │   │   ├── index.ts     # (Updated) CLI entry point.
│   │   │   ├── adapters/    # (Updated) Adapters for different benchmarks (Exercism, SWE-Bench, etc.).
│   │   │   │   ├── types.ts # (Updated) Defines adapter interface & richer MultiFacetVerificationResult.
│   │   │   │   └── ...      # (Implementations need updating for new metrics/benchmark specs).
│   │   │   ├── commands/    # (Updated) CLI command handlers (setup, run, report).
│   │   │   │   ├── run.ts   # (Needs update) To handle containerized environments, richer benchmark specs.
│   │   │   │   └── report.ts# (Needs update) To query/display new metrics from DB.
│   │   │   ├── db/          # (Updated) SQLite database setup and access for storing richer evaluation results.
│   │   │   │   ├── schema.ts# (Updated) DB schema including new metrics tables/columns.
│   │   │   │   └── index.ts # (Updated) Database access class.
│   │   │   └── utils/
│   │   │       ├── parsers.ts # (New) Utilities for parsing output from linters, SAST tools, etc.
│   │   │       ├── container.ts # (New) Utilities for managing Docker/Podman/devcontainer execution environments.
│   │   │       └── ...      # (Existing utils likely need updates).
│   │   ├── package.json     # (Updated) Dependencies for CLI (e.g., container management libs, parsers).
│   │   └── tsconfig.json
│   └── results/             # (Existing, Structure Maintained) Stores raw outputs and generated reports.
│       ├── runs/            # Raw output files from runs.
│       └── reports/         # Generated Markdown/JSON reports.
│       └── evals.db         # (Updated Schema) SQLite database storing all structured evaluation results.
│
├── locales/                 # (Existing) Localized documentation files.
│
├── proto/                   # (Existing, Expanded) Protocol Buffer definitions for Extension <-> Webview communication.
│   ├── common.proto         # (Existing) Basic shared types (Empty, String, etc.).
│   ├── agent.proto          # (New) Messages related to AgentInstance status or basic control.
│   ├── blueprints.proto     # (New) Messages/Service for listing, getting, saving Agent Blueprints.
│   ├── knowledge.proto      # (New) Messages/Service for querying the Shared Knowledge Base.
│   ├── logging.proto        # (New) Messages/Service for fetching trace logs.
│   ├── suggestions.proto    # (New) Messages/Service for listing/managing Meta-Agent suggestions.
│   ├── workflows.proto      # (New) Messages/Service for defining, managing, monitoring Workflows.
│   ├── workbench.proto      # (New) General Workbench state or aggregated data service.
│   ├── account.proto        # (Existing)
│   ├── browser.proto        # (Existing)
│   ├── checkpoints.proto    # (Existing)
│   ├── file.proto           # (Existing)
│   ├── mcp.proto            # (Existing)
│   ├── task.proto           # (Existing, May be Refactored/Renamed) Might evolve or be superseded by agent/workflow protos.
│   └── build-proto.js       # (Updated) Script to generate TS code from protos AND auto-generate gRPC method registries.
│
├── scripts/                 # (Existing) Utility scripts (e.g., test runner for CI).
│
├── src/                     # (Existing, Heavily Modified) Core Extension Host Code (TypeScript).
│   ├── extension.ts         # (Updated) Main activation point, initializes services, registers providers (including new WorkbenchViewProvider).
│   │
│   ├── api/                 # (Existing, Maintained) LLM API provider implementations and transformations.
│   │   ├── providers/       # (Add new providers here).
│   │   └── transform/       # (Add new format transformers here).
│   │
│   ├── core/                # (Existing, Refactored & Expanded) Core agent and workbench logic.
│   │   ├── agent/           # (New) Contains the refactored AgentInstance.
│   │   │   └── AgentInstance.ts # (New) Executes a single agent step based on a Blueprint. Stateless regarding history.
│   │   ├── assistant-message/ # (Existing) Logic for parsing LLM responses (tool calls, diffs).
│   │   ├── context/         # (Existing, Modified) Context management, prompt generation, mentions, rules, ignore.
│   │   │   ├── context-management/ # (Updated) ContextManager for truncation/optimization.
│   │   │   ├── instructions/  # (Existing) Prompt building components.
│   │   │   ├── mentions/      # (Existing) @mention parsing and content fetching.
│   │   │   └── ignore/        # (Existing) ClineIgnoreController.
│   │   ├── controller/      # (Existing, Refactored & Expanded) Handles communication, global state access, routes gRPC calls.
│   │   │   ├── index.ts     # (Updated) Main controller logic, state posting.
│   │   │   ├── grpc-handler.ts# (Updated) Routes incoming gRPC requests to appropriate service handlers.
│   │   │   ├── grpc-service.ts# (Existing) Base ServiceRegistry class.
│   │   │   ├── account/     # (Existing) gRPC handlers for AccountService.
│   │   │   ├── browser/     # (Existing) gRPC handlers for BrowserService.
│   │   │   ├── checkpoints/ # (Existing) gRPC handlers for CheckpointsService.
│   │   │   ├── file/        # (Existing) gRPC handlers for FileService.
│   │   │   ├── mcp/         # (Existing) gRPC handlers for McpService.
│   │   │   ├── task/        # (Existing, May be refactored/removed) Old handlers, potentially replaced by agent/workflow handlers.
│   │   │   ├── workbench/   # (New) gRPC handlers for general WorkbenchService.
│   │   │   ├── blueprints/  # (New) gRPC handlers for BlueprintService.
│   │   │   ├── workflows/   # (New) gRPC handlers for WorkflowService.
│   │   │   ├── suggestions/ # (New) gRPC handlers for SuggestionService.
│   │   │   ├── knowledge/   # (New) gRPC handlers for KnowledgeService.
│   │   │   └── logging/     # (New) gRPC handlers for fetching TraceLogs.
│   │   ├── storage/         # (Existing) State management utils, disk access utils.
│   │   ├── task/            # (Existing, Refactored/Removed) Original Task class. May be kept for basic chat mode or removed.
│   │   └── webview/         # (Existing) Webview management helpers (nonce, URI).
│   │
│   ├── integrations/        # (Existing, Maintained) Bridges to VS Code APIs & external libs (Checkpoints, Diff View, Terminal, Diagnostics, etc.).
│   │
│   ├── services/            # (Existing, Expanded) Self-contained services used by core logic.
│   │   ├── workbench/       # (New) Services specific to the Workbench functionality.
│   │   │   ├── orchestration/ # (New) WorkflowOrchestrator logic.
│   │   │   │   └── WorkflowOrchestrator.ts
│   │   │   ├── blueprints/  # (New) BlueprintService for managing definitions.
│   │   │   │   └── BlueprintService.ts
│   │   │   ├── workflows/   # (New) WorkflowService for managing definitions.
│   │   │   │   └── WorkflowService.ts
│   │   │   ├── suggestions/ # (New) SuggestionService for managing meta-agent suggestions.
│   │   │   │   └── SuggestionService.ts
│   │   │   ├── knowledge/   # (New) KnowledgeService interface for SKB.
│   │   │   │   └── KnowledgeService.ts
│   │   │   ├── indexing/    # (New) IndexingService for populating SKB (async).
│   │   │   │   └── IndexingService.ts
│   │   │   └── storage/     # (New) Specific storage adapters for SKB (SQLite, GraphDB interface, VectorDB interface).
│   │   ├── logging/         # (Existing, Expanded)
│   │   │   ├── Logger.ts    # (Existing) Basic Output Channel logger.
│   │   │   └── WorkflowLogger.ts # (New) Service for writing structured trace events to DB.
│   │   ├── mcp/             # (Existing) McpHub service.
│   │   ├── browser/         # (Existing) BrowserSession, UrlContentFetcher.
│   │   ├── search/          # (Existing) File search utilities.
│   │   ├── test/            # (Existing, Updated) Test mode server, Git helpers for evaluation file diffing.
│   │   ├── tree-sitter/     # (Existing) Code parsing service.
│   │   └── ...              # (Other existing services like account, telemetry)
│   │
│   ├── shared/              # (Existing, Expanded) Code shared between Extension Host & Webview.
│   │   ├── workbench/       # (New) Shared types/interfaces for Workbench concepts (AgentBlueprint, WorkflowDefinition, etc.).
│   │   │   └── types.ts
│   │   ├── proto/           # (Existing, Expanded) Generated Protobuf TypeScript code.
│   │   ├── proto-conversions/ # (Existing) Utils to convert between internal types and Protobuf types.
│   │   └── ...              # (Existing shared types like api.ts, ExtensionMessage.ts, etc.).
│   │
│   ├── utils/               # (Existing, Maintained) General utility functions.
│   └── workbench/           # (New) Specific providers or setup related to the Workbench UI View Container.
│       └── ui/
│           └── WorkbenchViewProvider.ts # (New) Manages the lifecycle and communication for the Workbench UI view.
│
├── test/                    # (Existing) Backend integration tests using @vscode/test-electron.
│
├── webview-ui/              # (Existing, Heavily Modified) React Frontend Code.
│   ├── src/
│   │   ├── App.tsx          # (Updated) Main application component, handles top-level routing between Chat and Workbench views.
│   │   ├── main.tsx         # (Existing) React entry point.
│   │   ├── index.css        # (Existing) Base styles.
│   │   ├── context/         # (Existing, Updated) React Context providers.
│   │   │   ├── ExtensionStateContext.tsx # (Updated) Manages state received from extension, expanded for Workbench data.
│   │   │   └── ...
│   │   ├── components/      # (Existing, Expanded) UI Components.
│   │   │   ├── chat/        # (Existing) Components for the standard chat view.
│   │   │   ├── settings/    # (Existing) Settings view components.
│   │   │   ├── history/     # (Existing) Task history view components.
│   │   │   ├── mcp/         # (Existing) MCP configuration/display components.
│   │   │   ├── account/     # (Existing) Account/Billing view components.
│   │   │   ├── common/      # (Existing) Reusable UI elements (CodeBlock, MarkdownBlock, etc.).
│   │   │   └── workbench/   # (New) Components specifically for the Workbench UI.
│   │   │       ├── WorkbenchRoot.tsx      # Root component for the Workbench view, likely with tabs/navigation.
│   │   │       ├── DashboardView.tsx      # Overview dashboard.
│   │   │       ├── BlueprintsView.tsx     # List/Editor for Agent Blueprints.
│   │   │       ├── WorkflowsView.tsx      # List/Editor for Workflow Definitions.
│   │   │       ├── InstancesView.tsx      # Monitor for Workflow Instances.
│   │   │       ├── TraceViewer.tsx        # Displays structured logs for an instance.
│   │   │       ├── SuggestionsView.tsx    # UI for reviewing Meta-Agent suggestions.
│   │   │       └── KnowledgeExplorer.tsx  # (Future) UI for browsing the SKB.
│   │   ├── services/        # (Existing, Expanded) Frontend services.
│   │   │   └── grpc-client.ts # (Updated) Generic gRPC client + specific service clients (BlueprintServiceClient, etc.).
│   │   └── utils/           # (Existing) Frontend utility functions.
│   ├── tests/               # (Existing) Frontend unit/component tests (Vitest).
│   ├── vite.config.ts       # (Existing) Vite build/dev server configuration.
│   └── package.json         # (Updated) Frontend dependencies (React, UI toolkit, potentially visualization libs).
│
├── .clinerules       # (New/User-Managed) Workspace-specific rules file.
├── .clineignore      # (New/User-Managed) Workspace-specific ignore file.
│
├── package.json             # (Existing, Updated) Root dependencies, scripts (including `npm run protos`).
├── tsconfig.json            # (Existing) Root TypeScript configuration.
└── ...                      # (Other existing config files: .eslintrc, .prettierrc, etc.)
```

**Key Changes Summarized:**

1.  **Core Refactoring:** `src/core/task/Task.ts` logic split into `src/core/agent/AgentInstance.ts` (single-step execution) and `src/services/workbench/orchestration/WorkflowOrchestrator.ts` (multi-step management).
2.  **New Workbench Services:** Dedicated services under `src/services/workbench/` for managing Blueprints, Workflows, Instances, Suggestions, Knowledge, and Indexing.
3.  **Expanded Shared Types:** New interfaces for Workbench concepts in `src/shared/workbench/types.ts`.
4.  **Expanded Protobuf/gRPC:** New `.proto` files and corresponding generated code/handlers/clients for Workbench features.
5.  **New Workbench UI:** A dedicated `WorkbenchViewProvider` and associated React components under `webview-ui/src/components/workbench/`.
6.  **Enhanced Evaluation:** Modifications within `evals/` to support richer metrics, benchmark specifications, and potentially containerized execution.
7.  **New User Configuration:** Introduction of `.cline/` directory in user workspaces for local blueprints/workflows, complementing global storage (`~/Documents/Cline/`).

This structure aims for modularity, separating the core agent execution logic from orchestration, definition management, knowledge storage, and UI presentation, facilitating targeted development and experimentation.

---

This document provides the structural blueprint. Subsequent documents can detail the specific interfaces, data flows, and logic within these new or modified components.