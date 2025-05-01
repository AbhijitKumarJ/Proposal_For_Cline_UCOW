Okay, continuing with the generation of illustrative source code for the **Cline Universal Code Orchestrator Workbench**. Here are the next set of key files, focusing on the services managing Blueprints and Workflows, and the basic UI components for listing them.

**Reminder:** This remains illustrative code. It lacks full error handling, UI polish, performance optimizations, security hardening, and comprehensive testing required for a production system. Placeholders (`// ...`) indicate where more complex logic is needed.

---

**Document 2 (Continued): Source Code for Key Workbench Modules**

**4. Blueprint Service (`src/services/workbench/blueprints/BlueprintService.ts`)**

```typescript
import * as vscode from "vscode";
import * as path from "path";
import * as fs from "fs/promises";
import * as yaml from "js-yaml"; // Using YAML for readability
import { AgentBlueprint } from "@shared/workbench/types";
import { ensureBlueprintDirectoryExists, fileExistsAtPath, readDirectory } from "@core/storage/disk";
import { Logger } from "@services/logging/Logger";
// Assume a simple schema validator (e.g., using Zod or manual type guards)
// import { validateAgentBlueprint } from "./blueprintValidation";
// Assume Git service for versioning local blueprints
// import { DefinitionGit } from "./DefinitionGit";

export class BlueprintService {
    private globalBlueprintDir: string = "";
    private workspaceBlueprintDir: string = "";
    private blueprintCache: Map<string, AgentBlueprint> = new Map(); // In-memory cache of latest versions
    // private definitionGit?: DefinitionGit; // Git service for local .cline/ definitions

    constructor(private context: vscode.ExtensionContext, private cwd?: string) {}

    async initialize(): Promise<void> {
        this.globalBlueprintDir = await ensureBlueprintDirectoryExists(true); // Assuming true for global
        if (this.cwd) {
            this.workspaceBlueprintDir = await ensureBlueprintDirectoryExists(false, this.cwd);
            // Initialize Git repo for local definitions if CWD exists
            // this.definitionGit = await DefinitionGit.create(path.join(this.cwd, '.cline'));
        }
        await this.loadDefinitions();
        this.setupWatchers(); // Watch for external file changes
    }

    private async loadDefinitions(): Promise<void> {
        Logger.info("Loading agent blueprints...");
        const loadedBlueprints = new Map<string, AgentBlueprint>();

        const loadFromDir = async (dirPath: string, isLocal: boolean) => {
            if (!dirPath || !(await fileExistsAtPath(dirPath))) return;
            try {
                const files = await readDirectory(dirPath);
                for (const filename of files) {
                    if (filename.endsWith('.json') || filename.endsWith('.yaml') || filename.endsWith('.yml')) {
                        const blueprintId = path.basename(filename).split('.')[0]; // Simple ID from filename
                        // In Git versioning, we'd load the HEAD version here
                        // For file archiving, find latest vN or base file
                        // Simplified for V1: just load the base file name
                        if (filename.includes(".v")) continue; // Skip archived versions for now

                        const filePath = path.join(dirPath, filename);
                        try {
                            const content = await fs.readFile(filePath, 'utf8');
                            const blueprintData = yaml.load(content) as any; // Use YAML

                            // TODO: Add schema validation using 'validateAgentBlueprint'
                            // TODO: Add schema version migration logic
                            if (blueprintData && blueprintData.id === blueprintId) {
                                // Prioritize local over global
                                if (!loadedBlueprints.has(blueprintId) || isLocal) {
                                    loadedBlueprints.set(blueprintId, blueprintData as AgentBlueprint);
                                }
                            } else {
                                Logger.warn(`Skipping invalid blueprint file: ${filePath} (ID mismatch or parse error)`);
                            }
                        } catch (readError) {
                            Logger.error(`Error reading blueprint file ${filePath}`, readError as Error);
                        }
                    }
                }
            } catch (dirError) {
                 Logger.error(`Error scanning blueprint directory ${dirPath}`, dirError as Error);
            }
        };

        await loadFromDir(this.globalBlueprintDir, false);
        if (this.workspaceBlueprintDir) {
            await loadFromDir(this.workspaceBlueprintDir, true);
        }

        this.blueprintCache = loadedBlueprints;
        Logger.info(`Loaded ${this.blueprintCache.size} blueprints.`);
        // TODO: Notify UI via Controller/postMessage that blueprints are updated
    }

    private setupWatchers(): void {
        // Set up vscode.workspace.createFileSystemWatcher for global/local dirs
        // On change/create/delete, trigger this.loadDefinitions() after a debounce
        // ... implementation omitted for brevity ...
    }

    async listBlueprints(): Promise<AgentBlueprint[]> {
        // Return values from the cache
        return Array.from(this.blueprintCache.values());
    }

    async getBlueprint(id: string): Promise<AgentBlueprint | undefined> {
        // Return specific blueprint from cache
        // TODO: Implement version fetching logic (Git or file archive) if needed
        return this.blueprintCache.get(id);
    }

    async saveBlueprint(blueprint: AgentBlueprint): Promise<void> {
        // TODO: Add schema validation `validateAgentBlueprint(blueprint)`

        const saveDir = this.workspaceBlueprintDir || this.globalBlueprintDir; // Prefer workspace
        const isLocal = !!this.workspaceBlueprintDir;
        const filenameBase = `${blueprint.id}.yaml`; // Use YAML

        try {
            const currentVersion = this.blueprintCache.get(blueprint.id)?.version || 0;
            blueprint.version = currentVersion + 1; // Increment version
            blueprint.$schemaVersion = "1.0"; // Set current schema version

            const newFilePath = path.join(saveDir, filenameBase);
            const newFileContent = yaml.dump(blueprint);

            if (isLocal) {
                // --- Local: Use Git Versioning ---
                // 1. Write the new file content
                await fs.writeFile(newFilePath, newFileContent, 'utf8');
                // 2. Use this.definitionGit.commitFile(newFilePath, `Update blueprint ${blueprint.id} to v${blueprint.version}`);
                // ... implementation requires DefinitionGit class ...
                Logger.info(`Saved blueprint ${blueprint.id} v${blueprint.version} (Local - Git commit needed)`);
            } else {
                // --- Global: Use File Archiving ---
                const currentFilePath = path.join(saveDir, filenameBase); // Assumes base file is latest
                if (await fileExistsAtPath(currentFilePath)) {
                    const archiveFilename = `${blueprint.id}.v${currentVersion}.yaml`;
                    await fs.rename(currentFilePath, path.join(saveDir, archiveFilename));
                    Logger.info(`Archived previous global blueprint version: ${archiveFilename}`);
                }
                await fs.writeFile(newFilePath, newFileContent, 'utf8');
                Logger.info(`Saved blueprint ${blueprint.id} v${blueprint.version} (Global)`);
            }

            // Update cache immediately
            this.blueprintCache.set(blueprint.id, blueprint);
            // TODO: Notify UI

        } catch (error) {
            Logger.error(`Failed to save blueprint ${blueprint.id}`, error as Error);
            throw error; // Re-throw for caller
        }
    }

    // TODO: Implement listVersions, getSpecificVersion (interacting with Git or file archives)
}
```

---

**5. Workflow Service (`src/services/workbench/workflows/WorkflowService.ts`)**

```typescript
import * as vscode from "vscode";
import * as path from "path";
import * as fs from "fs/promises";
import * as yaml from "js-yaml";
import { WorkflowDefinition } from "@shared/workbench/types";
import { ensureWorkflowDirectoryExists, fileExistsAtPath, readDirectory } from "@core/storage/disk";
import { Logger } from "@services/logging/Logger";
// Assume simple schema validator
// import { validateWorkflowDefinition } from "./workflowValidation";
// Assume Git service for versioning local workflows
// import { DefinitionGit } from "../blueprints/DefinitionGit"; // Reuse or create separate

export class WorkflowService {
    private globalWorkflowDir: string = "";
    private workspaceWorkflowDir: string = "";
    private workflowCache: Map<string, WorkflowDefinition> = new Map();
    // private definitionGit?: DefinitionGit;

    constructor(private context: vscode.ExtensionContext, private cwd?: string) {}

    async initialize(): Promise<void> {
        this.globalWorkflowDir = await ensureWorkflowDirectoryExists(true);
        if (this.cwd) {
            this.workspaceWorkflowDir = await ensureWorkflowDirectoryExists(false, this.cwd);
            // Initialize Git repo if needed
            // this.definitionGit = await DefinitionGit.create(path.join(this.cwd, '.cline'));
        }
        await this.loadDefinitions();
        this.setupWatchers();
    }

    private async loadDefinitions(): Promise<void> {
        Logger.info("Loading workflow definitions...");
        const loadedWorkflows = new Map<string, WorkflowDefinition>();

         const loadFromDir = async (dirPath: string, isLocal: boolean) => {
             if (!dirPath || !(await fileExistsAtPath(dirPath))) return;
            // Similar loading logic as BlueprintService, prioritizing local
            // ... implementation omitted for brevity, assume it populates loadedWorkflows ...
             try {
                const files = await readDirectory(dirPath);
                 for (const filename of files) {
                     if (filename.endsWith('.json') || filename.endsWith('.yaml') || filename.endsWith('.yml')) {
                         const workflowId = path.basename(filename).split('.')[0];
                         if (filename.includes(".v")) continue; // Skip archived

                         const filePath = path.join(dirPath, filename);
                         try {
                             const content = await fs.readFile(filePath, 'utf8');
                             const workflowData = yaml.load(content) as any;

                             // TODO: Add schema validation `validateWorkflowDefinition`
                             // TODO: Add schema version migration
                             if (workflowData && workflowData.id === workflowId) {
                                 if (!loadedWorkflows.has(workflowId) || isLocal) {
                                     loadedWorkflows.set(workflowId, workflowData as WorkflowDefinition);
                                 }
                             } else {
                                 Logger.warn(`Skipping invalid workflow file: ${filePath}`);
                             }
                         } catch (readError) {
                             Logger.error(`Error reading workflow file ${filePath}`, readError as Error);
                         }
                     }
                 }
             } catch (dirError) {
                  Logger.error(`Error scanning workflow directory ${dirPath}`, dirError as Error);
             }
        };

        await loadFromDir(this.globalWorkflowDir, false);
        if (this.workspaceWorkflowDir) {
            await loadFromDir(this.workspaceWorkflowDir, true);
        }

        this.workflowCache = loadedWorkflows;
        Logger.info(`Loaded ${this.workflowCache.size} workflow definitions.`);
        // TODO: Notify UI
    }

     private setupWatchers(): void {
        // Watch global/local dirs for changes, call this.loadDefinitions() debounced
        // ... implementation omitted ...
    }

    async listWorkflows(): Promise<WorkflowDefinition[]> {
        return Array.from(this.workflowCache.values());
    }

    async getWorkflow(id: string): Promise<WorkflowDefinition | undefined> {
        // TODO: Add version support if using Git
        return this.workflowCache.get(id);
    }

     async saveWorkflow(definition: WorkflowDefinition): Promise<void> {
        // TODO: Add schema validation `validateWorkflowDefinition(definition)`

        const saveDir = this.workspaceWorkflowDir || this.globalWorkflowDir;
        const isLocal = !!this.workspaceWorkflowDir;
        const filenameBase = `${definition.id}.yaml`; // Use YAML

         try {
             const currentVersion = this.workflowCache.get(definition.id)?.version || 0;
             definition.version = currentVersion + 1;
             definition.$schemaVersion = "1.0";

             const newFilePath = path.join(saveDir, filenameBase);
             const newFileContent = yaml.dump(definition);

             if (isLocal) {
                 // --- Local: Use Git Versioning ---
                // await fs.writeFile(newFilePath, newFileContent, 'utf8');
                // await this.definitionGit.commitFile(newFilePath, `Update workflow ${definition.id} to v${definition.version}`);
                 Logger.info(`Saved workflow ${definition.id} v${definition.version} (Local - Git commit needed)`);
                // For now, just write file without Git:
                await fs.writeFile(newFilePath, newFileContent, 'utf8');

             } else {
                 // --- Global: Use File Archiving ---
                 const currentFilePath = path.join(saveDir, filenameBase);
                 if (await fileExistsAtPath(currentFilePath)) {
                     const archiveFilename = `${definition.id}.v${currentVersion}.yaml`;
                     await fs.rename(currentFilePath, path.join(saveDir, archiveFilename));
                     Logger.info(`Archived previous global workflow version: ${archiveFilename}`);
                 }
                 await fs.writeFile(newFilePath, newFileContent, 'utf8');
                 Logger.info(`Saved workflow ${definition.id} v${definition.version} (Global)`);
             }

             this.workflowCache.set(definition.id, definition);
             // TODO: Notify UI

         } catch (error) {
             Logger.error(`Failed to save workflow ${definition.id}`, error as Error);
             throw error;
         }
    }
}
```

---

**6. Basic Blueprint List UI (`webview-ui/src/components/workbench/BlueprintsView.tsx`)**

```typescript
import React, { useState, useEffect } from 'react';
import { VSCodeDataGrid, VSCodeDataGridRow, VSCodeDataGridCell, VSCodeButton } from "@vscode/webview-ui-toolkit/react";
import { AgentBlueprint } from '@shared/workbench/types'; // Assuming types are shared
import { BlueprintServiceClient } from '@/services/grpc-client'; // Assuming gRPC client exists

const BlueprintsView: React.FC = () => {
    const [blueprints, setBlueprints] = useState<AgentBlueprint[]>([]);
    const [isLoading, setIsLoading] = useState(true);
    const [error, setError] = useState<string | null>(null);

    useEffect(() => {
        const fetchBlueprints = async () => {
            setIsLoading(true);
            setError(null);
            try {
                // Assume ListBlueprints returns { blueprints: AgentBlueprint[] }
                const response = await BlueprintServiceClient.listBlueprints({});
                setBlueprints(response.blueprints || []);
            } catch (err) {
                setError(err instanceof Error ? err.message : "Failed to load blueprints");
                console.error("Error fetching blueprints:", err);
            } finally {
                setIsLoading(false);
            }
        };
        fetchBlueprints();
    }, []);

    const handleCreateNew = () => {
        // TODO: Implement navigation or modal to create/edit blueprint
        console.log("Create new blueprint requested");
    };

    const handleEdit = (id: string) => {
        // TODO: Implement navigation or modal to create/edit blueprint
        console.log("Edit blueprint requested:", id);
    };

    if (isLoading) {
        return <div>Loading blueprints...</div>;
    }

    if (error) {
        return <div style={{ color: 'var(--vscode-errorForeground)' }}>Error: {error}</div>;
    }

    return (
        <div style={{ padding: '10px 20px' }}>
            <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center', marginBottom: '10px' }}>
                <h2>Agent Blueprints</h2>
                <VSCodeButton onClick={handleCreateNew}>Create New</VSCodeButton>
            </div>
            {blueprints.length === 0 ? (
                <p>No blueprints found. Create one to get started.</p>
            ) : (
                <VSCodeDataGrid aria-label="Agent Blueprints">
                    <VSCodeDataGridRow row-type="header">
                        <VSCodeDataGridCell cell-type="columnheader" grid-column="1">ID</VSCodeDataGridCell>
                        <VSCodeDataGridCell cell-type="columnheader" grid-column="2">Description</VSCodeDataGridCell>
                        <VSCodeDataGridCell cell-type="columnheader" grid-column="3">Version</VSCodeDataGridCell>
                        <VSCodeDataGridCell cell-type="columnheader" grid-column="4">Actions</VSCodeDataGridCell>
                    </VSCodeDataGridRow>
                    {blueprints.map((bp) => (
                        <VSCodeDataGridRow key={bp.id}>
                            <VSCodeDataGridCell grid-column="1">{bp.id}</VSCodeDataGridCell>
                            <VSCodeDataGridCell grid-column="2">{bp.description}</VSCodeDataGridCell>
                            <VSCodeDataGridCell grid-column="3">{bp.version}</VSCodeDataGridCell>
                             <VSCodeDataGridCell grid-column="4">
                                <VSCodeButton appearance="icon" title="Edit" onClick={() => handleEdit(bp.id)}>
                                    <span className="codicon codicon-edit"></span>
                                </VSCodeButton>
                                {/* Add View Versions button later */}
                            </VSCodeDataGridCell>
                        </VSCodeDataGridRow>
                    ))}
                </VSCodeDataGrid>
            )}
        </div>
    );
};

export default BlueprintsView;

```

---

**7. Basic Workflow List UI (`webview-ui/src/components/workbench/WorkflowsView.tsx`)**

```typescript
import React, { useState, useEffect } from 'react';
import { VSCodeDataGrid, VSCodeDataGridRow, VSCodeDataGridCell, VSCodeButton } from "@vscode/webview-ui-toolkit/react";
import { WorkflowDefinition } from '@shared/workbench/types';
import { WorkflowServiceClient } from '@/services/grpc-client'; // Assuming gRPC client exists

const WorkflowsView: React.FC = () => {
    const [workflows, setWorkflows] = useState<WorkflowDefinition[]>([]);
    const [isLoading, setIsLoading] = useState(true);
    const [error, setError] = useState<string | null>(null);

    useEffect(() => {
        const fetchWorkflows = async () => {
            setIsLoading(true);
            setError(null);
            try {
                // Assume ListWorkflows returns { workflows: WorkflowDefinition[] }
                const response = await WorkflowServiceClient.listWorkflows({});
                setWorkflows(response.workflows || []);
            } catch (err) {
                setError(err instanceof Error ? err.message : "Failed to load workflows");
                console.error("Error fetching workflows:", err);
            } finally {
                setIsLoading(false);
            }
        };
        fetchWorkflows();
    }, []);

     const handleCreateNew = () => {
        // TODO: Implement navigation or modal to create/edit workflow
        console.log("Create new workflow requested");
    };

     const handleEdit = (id: string) => {
        // TODO: Implement navigation or modal to create/edit workflow
        console.log("Edit workflow requested:", id);
    };

     const handleRun = (id: string) => {
        // TODO: Trigger workflow execution via gRPC
        console.log("Run workflow requested:", id);
         // Example: await WorkflowServiceClient.startWorkflow({ definitionId: id, initialInput: "Start Task" });
    };

    if (isLoading) {
        return <div>Loading workflows...</div>;
    }

    if (error) {
        return <div style={{ color: 'var(--vscode-errorForeground)' }}>Error: {error}</div>;
    }

    return (
        <div style={{ padding: '10px 20px' }}>
             <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center', marginBottom: '10px' }}>
                <h2>Workflows</h2>
                <VSCodeButton onClick={handleCreateNew}>Create New</VSCodeButton>
            </div>
            {workflows.length === 0 ? (
                 <p>No workflows found. Create one to automate agent sequences.</p>
            ) : (
                <VSCodeDataGrid aria-label="Workflow Definitions">
                    <VSCodeDataGridRow row-type="header">
                        <VSCodeDataGridCell cell-type="columnheader" grid-column="1">ID</VSCodeDataGridCell>
                        <VSCodeDataGridCell cell-type="columnheader" grid-column="2">Description</VSCodeDataGridCell>
                        <VSCodeDataGridCell cell-type="columnheader" grid-column="3">Steps</VSCodeDataGridCell>
                         <VSCodeDataGridCell cell-type="columnheader" grid-column="4">Actions</VSCodeDataGridCell>
                    </VSCodeDataGridRow>
                    {workflows.map((wf) => (
                        <VSCodeDataGridRow key={wf.id}>
                            <VSCodeDataGridCell grid-column="1">{wf.id}</VSCodeDataGridCell>
                            <VSCodeDataGridCell grid-column="2">{wf.description}</VSCodeDataGridCell>
                            <VSCodeDataGridCell grid-column="3">{wf.steps.length}</VSCodeDataGridCell>
                             <VSCodeDataGridCell grid-column="4">
                                 <VSCodeButton appearance="icon" title="Run Workflow" onClick={() => handleRun(wf.id)}>
                                    <span className="codicon codicon-play"></span>
                                </VSCodeButton>
                                <VSCodeButton appearance="icon" title="Edit Workflow" onClick={() => handleEdit(wf.id)}>
                                    <span className="codicon codicon-edit"></span>
                                </VSCodeButton>
                                {/* Add delete button later */}
                            </VSCodeDataGridCell>
                        </VSCodeDataGridRow>
                    ))}
                </VSCodeDataGrid>
            )}
        </div>
    );
};

export default WorkflowsView;
```

---

**Next Steps:**

This selection provides the core interfaces, services for managing definitions, and basic UI views for listing them. The subsequent steps would involve:

1.  Implementing the actual file reading/writing/validation logic within the services.
2.  Implementing the Git-based versioning for local definitions.
3.  Building out the editor UIs for Blueprints and Workflows.
4.  Integrating these services fully with the `WorkflowOrchestrator` and `AgentInstance`.
5.  Developing the `InstanceMonitor` and `TraceViewer` components based on the logging service.