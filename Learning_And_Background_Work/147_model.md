You're right to push for completeness! While generating *every single file* accurately is infeasible due to the sheer volume and iterative nature of real development, we can generate the content for several more **important supporting files and modified existing files** to round out the picture of the Cline Workbench implementation.

This batch focuses on:
1.  Protobuf definitions for the new Workbench services.
2.  The corresponding gRPC service handlers in the controller.
3.  Modifications to the main `extension.ts` to initialize Workbench components.
4.  Updates to `src/shared/ExtensionMessage.ts` for new state/message types.

**Reminder:** This remains illustrative, focusing on structure and integration points. Error handling, specific logic details, and UI interactions are simplified.

---

**Document 2 (Continued): Source Code for Key Workbench Modules**

**23. Protobuf Definitions (Examples - Additions to `proto/`)**

*   **`proto/blueprints.proto` (New):**

```protobuf
syntax = "proto3";

package cline;
option java_package = "bot.cline.proto";
option java_multiple_files = true;

import "common.proto";
import "api.proto"; // Assuming api.proto defines ApiConfiguration message

// Basic info for listing blueprints
message BlueprintInfo {
  string id = 1;
  string description = 2;
  int32 version = 3; // Latest version number
}

// Full definition for getting/saving
message BlueprintDefinition {
  string id = 1;
  int32 version = 2;
  string schema_version = 3;
  string description = 4;
  string base_system_prompt = 5;
  repeated string allowed_tools = 6;
  ApiConfiguration default_model_config = 7;
}

// Service definition
service BlueprintService {
  rpc ListBlueprints (EmptyRequest) returns (ListBlueprintsResponse);
  rpc GetBlueprint (GetBlueprintRequest) returns (GetBlueprintResponse);
  rpc SaveBlueprint (SaveBlueprintRequest) returns (Empty);
  // rpc ListBlueprintVersions (StringRequest) returns (ListBlueprintVersionsResponse); // Future
}

message ListBlueprintsResponse {
  repeated BlueprintInfo blueprints = 1;
}

message GetBlueprintRequest {
  string id = 1;
  optional int32 version = 2; // Optional: Get specific version
}

message GetBlueprintResponse {
  BlueprintDefinition blueprint = 1;
}

message SaveBlueprintRequest {
   BlueprintDefinition blueprint = 1;
}

// --- Add similar definitions for workflows.proto, suggestions.proto, logging.proto etc. ---
// Example for logging:
// service TraceLogService {
//   rpc GetTrace (GetTraceRequest) returns (GetTraceResponse);
// }
// message GetTraceRequest { string instance_id = 1; optional int32 limit = 2; }
// message LogEvent { int64 timestamp = 1; string eventType = 2; string payloadJson = 3; ... }
// message GetTraceResponse { repeated LogEvent logs = 1; }
```

*   **(Note:** Remember to run `npm run protos` after adding/modifying `.proto` files).

---

**24. gRPC Service Handlers (`src/core/controller/workbench/`)**

*   **`src/core/controller/workbench/blueprint_service.ts` (New):**

```typescript
import { Controller } from "@core/controller";
import { BlueprintService as BackendBlueprintService } from "@services/workbench/blueprints/BlueprintService"; // Assuming this path
import { EmptyRequest, Empty } from "@shared/proto/common";
import { ListBlueprintsResponse, GetBlueprintRequest, GetBlueprintResponse, SaveBlueprintRequest, BlueprintInfo, BlueprintDefinition } from "@shared/proto/blueprints"; // Generated types
import { Logger } from "@services/logging/Logger";

// Assuming BlueprintService instance is accessible via Controller or injected
// For simplicity, accessing via controller. Likely needs better dependency injection.
const getBlueprintService = (controller: Controller): BackendBlueprintService => {
    if (!controller.blueprintService) { // Assume controller holds service instances
        throw new Error("BlueprintService not initialized in Controller");
    }
    return controller.blueprintService;
};

export async function listBlueprints(controller: Controller, request: EmptyRequest): Promise<ListBlueprintsResponse> {
    try {
        const service = getBlueprintService(controller);
        const blueprints = await service.listBlueprints();
        // Convert internal AgentBlueprint to proto BlueprintInfo
        const infos: BlueprintInfo[] = blueprints.map(bp => ({
            id: bp.id,
            description: bp.description || "",
            version: bp.version || 0,
        }));
        return { blueprints: infos };
    } catch (error) {
        Logger.error("gRPC listBlueprints failed", error as Error);
        throw error; // Let grpc-handler format the error for the client
    }
}

export async function getBlueprint(controller: Controller, request: GetBlueprintRequest): Promise<GetBlueprintResponse> {
     try {
        const service = getBlueprintService(controller);
        const blueprint = await service.getBlueprint(request.id, request.version ?? undefined);
        if (!blueprint) {
            throw new Error(`Blueprint not found: ${request.id} ${request.version ? `v${request.version}`:''}`);
        }
        // Convert internal AgentBlueprint to proto BlueprintDefinition
        const definition: BlueprintDefinition = {
            id: blueprint.id,
            version: blueprint.version,
            schemaVersion: blueprint.$schemaVersion || "1.0",
            description: blueprint.description || "",
            baseSystemPrompt: blueprint.baseSystemPrompt || "",
            allowedTools: blueprint.allowedTools || [],
            defaultModelConfig: blueprint.defaultModelConfig || {}, // Needs conversion if proto differs
        };
        return { blueprint: definition };
    } catch (error) {
        Logger.error(`gRPC getBlueprint failed for ${request.id}`, error as Error);
        throw error;
    }
}

export async function saveBlueprint(controller: Controller, request: SaveBlueprintRequest): Promise<Empty> {
    try {
        if (!request.blueprint) {
            throw new Error("SaveBlueprintRequest missing blueprint definition");
        }
        const service = getBlueprintService(controller);
        // Convert proto BlueprintDefinition back to internal AgentBlueprint
        const blueprintToSave = { // Add required fields from proto
            id: request.blueprint.id,
            version: request.blueprint.version, // Service handles incrementing based on this
            $schemaVersion: request.blueprint.schemaVersion,
            description: request.blueprint.description,
            baseSystemPrompt: request.blueprint.baseSystemPrompt,
            allowedTools: request.blueprint.allowedTools,
            defaultModelConfig: request.blueprint.defaultModelConfig || {}, // Handle conversion
        };
        await service.saveBlueprint(blueprintToSave);
        return Empty.create();
    } catch (error) {
        Logger.error(`gRPC saveBlueprint failed for ${request.blueprint?.id}`, error as Error);
        throw error;
    }
}

// Add similar handlers for WorkflowService, SuggestionService, TraceLogService etc.
```

*   **`src/core/controller/workbench/methods.ts` (Auto-Generated by `npm run protos`):**
    ```typescript
    // AUTO-GENERATED FILE - DO NOT MODIFY DIRECTLY
    // Generated by proto/build-proto.js

    // Import all method implementations
    import { registerMethod } from "./index"; // Assumes index.ts exists in this dir
    import { listBlueprints, getBlueprint, saveBlueprint } from "./blueprint_service";
    // import { listWorkflows, getWorkflow, saveWorkflow, startWorkflow } from "./workflow_service";
    // import { listPendingSuggestions, applySuggestion, ignoreSuggestion } from "./suggestion_service";
    // import { getTrace } from "./logging_service";
    // import { getWorkbenchStatus } from "./workbench_service";


    // Register all workbench service methods (combining multiple services for simplicity here)
    export function registerAllMethods(): void {
        // Register each method with the registry
        registerMethod("ListBlueprints", listBlueprints);
        registerMethod("GetBlueprint", getBlueprint);
        registerMethod("SaveBlueprint", saveBlueprint);
        // registerMethod("ListWorkflows", listWorkflows);
        // ... register other methods ...
    }
    ```
*   **`src/core/controller/workbench/index.ts` (New):**
    ```typescript
    import { createServiceRegistry, ServiceMethodHandler } from "../grpc-service";
    import { registerAllMethods } from "./methods";

    // Create combined Workbench service registry (or could be multiple registries)
    const workbenchServiceRegistry = createServiceRegistry("workbench"); // Service name used in proto?

    // Export the method handler type and registration function
    export type WorkbenchMethodHandler = ServiceMethodHandler;
    export const registerMethod = workbenchServiceRegistry.registerMethod;

    // Export the request handler
    export const handleWorkbenchServiceRequest = workbenchServiceRegistry.handleRequest;

    // Register all workbench methods
    registerAllMethods();
    ```
*   **`src/core/controller/grpc-handler.ts` (Modified):**
    ```typescript
    // ... other imports ...
    import { handleWorkbenchServiceRequest } from "./workbench"; // Import new handler

    export class GrpcHandler {
        // ... constructor ...

        async handleRequest(
            service: string,
            method: string,
            message: any,
            requestId: string,
        ): Promise<{ message?: any; error?: string; request_id: string; }> {
            try {
                // Add new case for workbench services
                if (service.startsWith("cline.BlueprintService") || // Match proto service names
                    service.startsWith("cline.WorkflowService") ||
                    service.startsWith("cline.InstanceService") ||
                    service.startsWith("cline.SuggestionService") ||
                    service.startsWith("cline.TraceLogService") ||
                    service.startsWith("cline.WorkbenchService"))
                {
                     return {
                         message: await handleWorkbenchServiceRequest(this.controller, method, message),
                         request_id: requestId,
                     };
                 }

                // Existing cases
                switch (service) {
                    case "cline.AccountService": /*...*/ ;
                    case "cline.BrowserService": /*...*/ ;
                    // ... other existing services ...
                    default:
                        throw new Error(`Unknown service: ${service}`);
                }
            } catch (error) {
                // ... error handling ...
            }
        }
    }

    // ... handleGrpcRequest function remains similar ...
    ```

---

**25. Extension Entry Point (`src/extension.ts` - Modifications)**

```typescript
// ... existing imports ...
import { Controller } from "@core/controller/index";
import { WorkbenchViewProvider } from "./workbench/ui/WorkbenchViewProvider"; // NEW Workbench UI Provider
import { BlueprintService } from "@services/workbench/blueprints/BlueprintService"; // NEW Service
import { WorkflowService } from "@services/workbench/workflows/WorkflowService"; // NEW Service
import { WorkflowOrchestrator } from "@services/workbench/orchestration/WorkflowOrchestrator"; // NEW Service
import { WorkflowLogger } from "@services/logging/WorkflowLogger"; // NEW Service
import { SuggestionService } from "@services/workbench/suggestions/SuggestionService"; // NEW Service
import { McpHub } from "@services/mcp/McpHub"; // Existing, but needed by AgentInstance/Orchestrator
import { WorkspaceTracker } from "@integrations/workspace/WorkspaceTracker"; // Existing

// ... existing code ...

export async function activate(context: vscode.ExtensionContext) { // Make activate async
    // ... existing setup (outputChannel, ErrorService, Logger) ...

    // --- Initialize Core Services ---
    // Note: Order matters for dependencies
    const logger = new WorkflowLogger(context.globalStorageUri.fsPath);
    await logger.initialize(); // Initialize DB asynchronously
    context.subscriptions.push({ dispose: () => logger.dispose() });

    const blueprintService = new BlueprintService(context, vscode.workspace.workspaceFolders?.[0]?.uri.fsPath);
    await blueprintService.initialize();
    // TODO: Add blueprintService disposal if needed

    const workflowService = new WorkflowService(context, vscode.workspace.workspaceFolders?.[0]?.uri.fsPath);
    await workflowService.initialize();
     // TODO: Add workflowService disposal if needed

    const suggestionService = new SuggestionService(logger, context.globalStorageUri.fsPath);
    await suggestionService.initialize();
     // TODO: Add suggestionService disposal if needed

    // Instantiate existing services needed by AgentInstance/Orchestrator
    const workspaceTracker = new WorkspaceTracker((msg) => sidebarWebview.controller.postMessageToWebview(msg)); // Assuming sidebar exists first
    const mcpHub = new McpHub(/*...*/); // Initialize McpHub as before
    // ... initialize other dependencies like TerminalManager, BrowserSession ...

    // Instantiate the Orchestrator, injecting dependencies
    const workflowOrchestrator = new WorkflowOrchestrator(
        blueprintService,
        logger,
        { // Pass dependencies needed by AgentInstance
            logger,
            controller: sidebarWebview.controller, // Pass the main controller for 'ask' routing
            mcpHub,
            workspaceTracker,
            // ... other services ...
        },
        context
    );
    await workflowOrchestrator.initialize(); // Load interrupted instances
     // TODO: Add orchestrator disposal

    // --- Register Webview Providers ---
    // Existing Sidebar Provider
    const sidebarWebview = new WebviewProvider(context, outputChannel);
    // Inject workbench services into the main controller so gRPC handlers can access them
    sidebarWebview.controller.blueprintService = blueprintService; // Add properties dynamically or refactor Controller constructor
    sidebarWebview.controller.workflowService = workflowService;
    sidebarWebview.controller.suggestionService = suggestionService;
    sidebarWebview.controller.workflowOrchestrator = workflowOrchestrator; // Allow triggering workflows
    sidebarWebview.controller.workflowLogger = logger; // For log fetching gRPC
    // ... inject other services if needed by gRPC handlers ...

    context.subscriptions.push(
        vscode.window.registerWebviewViewProvider(WebviewProvider.sideBarId, sidebarWebview, {
            webviewOptions: { retainContextWhenHidden: true },
        }),
    );

    // NEW Workbench Provider
    const workbenchProvider = new WorkbenchViewProvider(context, sidebarWebview.controller);
    context.subscriptions.push(
        vscode.window.registerWebviewViewProvider(WorkbenchViewProvider.viewId, workbenchProvider)
    );

    // --- Register Commands & Other Activation Logic ---
    // ... existing command registrations (buttons, etc.) ...

    // Example: Command to manually start a workflow (for testing)
    context.subscriptions.push(vscode.commands.registerCommand('cline.dev.runWorkflow', async () => {
        const workflowId = await vscode.window.showInputBox({ prompt: "Enter Workflow ID to run" });
        const initialInput = await vscode.window.showInputBox({ prompt: "Enter initial task input" });
        if (workflowId && initialInput) {
            try {
                const instanceId = await workflowOrchestrator.startWorkflow(workflowId, initialInput);
                vscode.window.showInformationMessage(`Started workflow instance: ${instanceId}`);
            } catch (error) {
                 vscode.window.showErrorMessage(`Failed to start workflow: ${error.message}`);
            }
        }
    }));


    // ... rest of existing activate function (URI handler, dev commands, API export) ...

    return createClineAPI(outputChannel, sidebarWebview.controller);
}

// ... deactivate function (ensure new services like logger are disposed) ...
export function deactivate() {
    // Call dispose methods on logger, orchestrator, etc.
    Logger.log("Deactivating Cline Workbench components...");
    // ... existing cleanup ...
}

```

---

**26. Shared Extension State (`src/shared/ExtensionMessage.ts` - Additions)**

```typescript
import { AgentBlueprint, WorkflowDefinition, WorkflowInstance, MetaAgentSuggestion } from "./workbench/types";
// ... other imports ...

export interface ExtensionState {
    // ... existing fields ...
    apiConfiguration?: ApiConfiguration;
    autoApprovalSettings: AutoApprovalSettings;
    browserSettings: BrowserSettings;
    chatSettings: ChatSettings;
    // ... etc ...

    // --- NEW Workbench State ---
    availableBlueprints: AgentBlueprint[]; // List of latest versions for UI display
    availableWorkflows: WorkflowDefinition[]; // List of latest versions for UI display
    activeWorkflowInstances: WorkflowInstance[]; // Summary of running instances
    pendingSuggestions: MetaAgentSuggestion[]; // Suggestions needing review
}

// Add new message types for Workbench UI updates
export interface ExtensionMessage {
    type:
        | "action" | "state" | "theme" | "workspaceUpdated" | "invoke" | "partialMessage"
        // ... existing types ...
        | "workbenchStateUpdate" // Send full Workbench-related state
        | "workflowInstanceUpdate" // Send update for a single instance status/progress
        | "newSuggestionAvailable" // Notify UI about a new suggestion
        | "grpc_response"
        // ... other types ...
    ;

    state?: ExtensionState; // Existing state
    // Add specific payloads for new message types if needed
    workbenchState?: { // Example payload for workbench update
        availableBlueprints: AgentBlueprint[];
        availableWorkflows: WorkflowDefinition[];
        activeWorkflowInstances: Pick<WorkflowInstance, 'instanceId' | 'definitionId' | 'status' | 'currentStepIndex' | 'startTime'>[]; // Summary only
        pendingSuggestions: Pick<MetaAgentSuggestion, 'suggestionId' | 'targetBlueprintId' | 'targetBlueprintVersion' | 'sourceMetaAgentId'>[]; // Summary only
    };
    instanceUpdate?: Pick<WorkflowInstance, 'instanceId' | 'status' | 'currentStepIndex' | 'endTime' | 'error'>;
    newSuggestionId?: string;
    // ... other fields ...
    grpc_response?: { message?: any; request_id: string; error?: string; };
}

```

---

**Conclusion:**

This batch provides illustrative code for essential supporting components: Protobuf definitions structuring the new communication needs, gRPC handlers routing UI requests to backend services, updates to the extension entry point (`activate`) to initialize these new services, and additions to the shared state/message types.

With these pieces, along with the previously defined `AgentInstance`, `WorkflowOrchestrator`, `BlueprintService`, `WorkflowService`, `SuggestionService`, `WorkflowLogger`, and basic UI views, the core functional skeleton of the Cline Universal Code Orchestrator Workbench is established. The next crucial steps involve fleshing out the detailed logic within each component, implementing robust error handling and state management, building out the richer UI interactions (editors, visualizers, trace viewers), and integrating the enhanced evaluation framework for validation and adaptation loops.