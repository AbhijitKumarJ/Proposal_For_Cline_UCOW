Okay, let's continue generating illustrative source code for the **Cline Universal Code Orchestrator Workbench**. This batch will focus on the **Webview UI components** for the Workbench, specifically the basic list/editor views for Blueprints and Workflows, and the Instance Monitor.

**Reminder:** This is illustrative code. UI polish, complete state handling, error display, loading indicators, advanced features (like visual workflow editing or Git versioning UI), and full gRPC integration are simplified or omitted.

---

**Document 2 (Continued): Source Code for Key Workbench Modules**

**19. Blueprint Editor View (`webview-ui/src/components/workbench/BlueprintEditorView.tsx`)**

```typescript
import React, { useState, useEffect, useCallback } from 'react';
import { VSCodeButton, VSCodeTextArea, VSCodeTextField, VSCodeDropdown, VSCodeOption } from "@vscode/webview-ui-toolkit/react";
import { AgentBlueprint } from '@shared/workbench/types';
import { BlueprintServiceClient } from '@/services/grpc-client';
import { useExtensionState } from '@/context/ExtensionStateContext'; // To get tool names, API config defaults
import { toolUseNames } from '@core/assistant-message/index'; // Need access to tool names
import { ApiProvider } from '@shared/api';
import { vscode } from '@/utils/vscode';

interface BlueprintEditorViewProps {
    blueprintId?: string; // If editing existing blueprint
    onSave: (savedBlueprint: AgentBlueprint) => void; // Callback after saving
    onCancel: () => void;
}

// Helper to provide default values for a new blueprint
const createNewBlueprint = (): Omit<AgentBlueprint, 'id' | 'version' | '$schemaVersion'> => ({
    description: "",
    baseSystemPrompt: `# Role: [Agent Role Here]\n\n# Goal:\n[Describe the primary objective]\n\n# Constraints:\n- Use only the allowed tools.\n- [Add other constraints]\n\n# Workflow:\n1. Analyze input.\n2. [Step 2]\n3. ...\n4. Output result.`,
    allowedTools: [],
    defaultModelConfig: { apiProvider: 'openrouter' }, // Example default
});

const BlueprintEditorView: React.FC<BlueprintEditorViewProps> = ({ blueprintId, onSave, onCancel }) => {
    const { apiConfiguration: globalApiConfig } = useExtensionState(); // For provider list & defaults
    const [blueprint, setBlueprint] = useState<Partial<AgentBlueprint>>(createNewBlueprint());
    const [isLoading, setIsLoading] = useState(false);
    const [error, setError] = useState<string | null>(null);
    const [isDirty, setIsDirty] = useState(false);
    const [currentVersion, setCurrentVersion] = useState<number>(0); // For display if editing

    // Fetch existing blueprint if blueprintId is provided
    useEffect(() => {
        if (blueprintId) {
            setIsLoading(true);
            BlueprintServiceClient.getBlueprint({ id: blueprintId })
                .then(response => {
                    if (response.blueprint) {
                        setBlueprint(response.blueprint);
                        setCurrentVersion(response.blueprint.version || 0);
                        setIsDirty(false); // Reset dirty state after loading
                    } else {
                        setError(`Blueprint "${blueprintId}" not found.`);
                    }
                })
                .catch(err => setError(err.message))
                .finally(() => setIsLoading(false));
        } else {
             setIsDirty(true); // Mark as dirty for new blueprints
             setBlueprint(createNewBlueprint()); // Ensure state is clean for new
             setCurrentVersion(0);
        }
    }, [blueprintId]);

    const handleInputChange = useCallback((field: keyof AgentBlueprint) => (event: any) => {
        setBlueprint(prev => ({ ...prev, [field]: event.target.value }));
        setIsDirty(true);
    }, []);

    const handleModelConfigChange = useCallback((field: keyof ApiConfiguration) => (event: any) => {
         setBlueprint(prev => ({
            ...prev,
            defaultModelConfig: {
                 ...(prev?.defaultModelConfig || {}),
                 [field]: event.target.value
            }
        }));
        setIsDirty(true);
    }, []);

     const handleToolToggle = useCallback((toolName: ToolUseName) => {
         setBlueprint(prev => {
            const currentTools = prev?.allowedTools || [];
            const newTools = currentTools.includes(toolName)
                ? currentTools.filter(t => t !== toolName)
                : [...currentTools, toolName];
            return { ...prev, allowedTools: newTools };
        });
         setIsDirty(true);
    }, []);


    const handleSave = async () => {
        setError(null);
        setIsLoading(true);
        // Basic validation
        if (!blueprint.id?.trim()) {
            setError("Blueprint ID is required.");
            setIsLoading(false);
            return;
        }
         if (!blueprint.baseSystemPrompt?.trim()) {
            setError("Base System Prompt is required.");
            setIsLoading(false);
            return;
        }

        try {
             // Prepare blueprint for saving (ensure required fields)
             const blueprintToSave: AgentBlueprint = {
                 id: blueprint.id.trim(),
                 version: currentVersion, // Service will handle incrementing
                 $schemaVersion: "1.0",
                 description: blueprint.description || "",
                 baseSystemPrompt: blueprint.baseSystemPrompt,
                 allowedTools: blueprint.allowedTools || [],
                 defaultModelConfig: blueprint.defaultModelConfig || {},
             };

             // Call gRPC save method
            await BlueprintServiceClient.saveBlueprint({ blueprint: blueprintToSave });
            onSave(blueprintToSave); // Pass saved blueprint back
            setIsDirty(false);
        } catch (err) {
            setError(err instanceof Error ? err.message : "Failed to save blueprint");
        } finally {
            setIsLoading(false);
        }
    };

     if (isLoading && blueprintId) { // Only show loading if editing existing
         return <div>Loading blueprint {blueprintId}...</div>;
     }

     if (error && !isLoading) { // Only show error if not loading
         return <div style={{ color: 'var(--vscode-errorForeground)' }}>Error: {error}</div>;
     }

    // Determine available API providers (could be fetched or from a constant)
    const availableProviders: ApiProvider[] = ['openrouter', 'anthropic', 'openai', /* ... other providers */];


    return (
        <div style={{ padding: '15px', display: 'flex', flexDirection: 'column', gap: '15px', height: '100%', boxSizing: 'border-box' }}>
            <h3>{blueprintId ? `Edit Blueprint: ${blueprintId} (v${currentVersion})` : 'Create New Blueprint'}</h3>

            <VSCodeTextField
                value={blueprint.id || ""}
                onInput={handleInputChange('id')}
                disabled={!!blueprintId || isLoading} // Don't allow changing ID if editing
                required
                placeholder="e.g., python-test-writer">
                Blueprint ID
            </VSCodeTextField>

            <VSCodeTextField
                value={blueprint.description || ""}
                onInput={handleInputChange('description')}
                 disabled={isLoading}
                placeholder="e.g., Generates pytest tests for Python functions">
                Description
            </VSCodeTextField>

            <div>
                <label htmlFor="system-prompt-area" style={{ display: 'block', marginBottom: '5px', fontWeight: 500 }}>Base System Prompt</label>
                 <VSCodeTextArea
                    id="system-prompt-area"
                    value={blueprint.baseSystemPrompt || ""}
                    onInput={handleInputChange('baseSystemPrompt')}
                     disabled={isLoading}
                    rows={10}
                    resize="vertical"
                    style={{ width: '100%' }}
                 />
            </div>

            <div>
                <span style={{ display: 'block', marginBottom: '5px', fontWeight: 500 }}>Allowed Tools</span>
                <div style={{ display: 'flex', flexWrap: 'wrap', gap: '10px', maxHeight: '150px', overflowY: 'auto', background: 'var(--vscode-input-background)', padding: '8px', borderRadius: '3px', border: '1px solid var(--vscode-input-border)'}}>
                    {toolUseNames.map(toolName => (
                        <VSCodeCheckbox
                            key={toolName}
                            checked={blueprint.allowedTools?.includes(toolName)}
                            onChange={() => handleToolToggle(toolName)}
                             disabled={isLoading}
                        >
                            {toolName}
                        </VSCodeCheckbox>
                    ))}
                </div>
            </div>

             <div>
                <span style={{ display: 'block', marginBottom: '5px', fontWeight: 500 }}>Default Model Configuration (Overrides Global)</span>
                <div style={{ display: 'flex', flexDirection: 'column', gap: '10px', background: 'var(--vscode-input-background)', padding: '10px', borderRadius: '3px', border: '1px solid var(--vscode-input-border)' }}>
                     <VSCodeDropdown
                        value={blueprint.defaultModelConfig?.apiProvider || ""}
                        onChange={handleModelConfigChange('apiProvider')}
                        disabled={isLoading}
                     >
                         <VSCodeOption value="">Inherit Global ({globalApiConfig?.apiProvider || 'N/A'})</VSCodeOption>
                         {availableProviders.map(p => <VSCodeOption key={p} value={p}>{p}</VSCodeOption>)}
                    </VSCodeDropdown>
                     <VSCodeTextField
                        value={blueprint.defaultModelConfig?.apiModelId || ""}
                        onInput={handleModelConfigChange('apiModelId')}
                        disabled={isLoading}
                        placeholder={`Inherit Global (${globalApiConfig?.apiModelId || 'N/A'})`} >
                         Model ID (Optional)
                    </VSCodeTextField>
                     {/* Add other relevant ApiConfiguration fields as needed */}
                </div>
            </div>


            <div style={{ display: 'flex', gap: '10px', marginTop: 'auto' }}>
                <VSCodeButton onClick={handleSave} disabled={isLoading || !isDirty}>
                    {isLoading ? 'Saving...' : 'Save Blueprint'}
                </VSCodeButton>
                <VSCodeButton appearance="secondary" onClick={onCancel} disabled={isLoading}>Cancel</VSCodeButton>
            </div>
        </div>
    );
};

export default BlueprintEditorView;
```

---

**20. Workflow Editor View (`webview-ui/src/components/workbench/WorkflowsView.tsx` - Basic Text Editor)**

```typescript
import React, { useState, useEffect, useCallback } from 'react';
import { VSCodeButton, VSCodeTextArea, VSCodeTextField } from "@vscode/webview-ui-toolkit/react";
import { WorkflowDefinition } from '@shared/workbench/types';
import { WorkflowServiceClient } from '@/services/grpc-client';
import * as yaml from "js-yaml";
import MermaidBlock from '@/components/common/MermaidBlock'; // Assuming Mermaid component exists
import { Logger } from '@services/logging/Logger'; // Use your logger

interface WorkflowEditorViewProps {
    workflowId?: string;
    onSave: (savedWorkflow: WorkflowDefinition) => void;
    onCancel: () => void;
    onRun: (workflowId: string) => void; // Callback to trigger run
}

const createNewWorkflow = (): Partial<WorkflowDefinition> => ({
    description: "",
    steps: [],
    $schemaVersion: "1.0",
    version: 0, // Initial version
});

// Basic function to convert simple linear/conditional YAML/JSON to Mermaid flowchart
const generateMermaidDiagram = (definition?: Partial<WorkflowDefinition>): string => {
    if (!definition?.steps || definition.steps.length === 0) return "graph TD\n  Start --> End";

    let mermaid = "graph TD\n";
    mermaid += "  Start --> " + definition.steps[0].stepId + ";\n";

    definition.steps.forEach((step, index) => {
        mermaid += `  ${step.stepId}[${step.blueprintId}];\n`;

        const nextStepIndex = index + 1;
        const nextStep = definition.steps?.[nextStepIndex];

        // Success path
        const successTargetId = step.onSuccess?.targetStepId || nextStep?.stepId;
        if (successTargetId) {
            mermaid += `  ${step.stepId} -- Success --> ${successTargetId};\n`;
        } else {
            // Last step success path
             if (index === definition.steps!.length - 1) {
                 mermaid += `  ${step.stepId} -- Success --> End;\n`;
             }
        }

        // Failure paths
        if (step.onFailure && step.onFailure.length > 0) {
            step.onFailure.forEach(failTransition => {
                const label = failTransition.errorCode ? `Fail (${failTransition.errorCode})` : "Fail (Default)";
                 mermaid += `  ${step.stepId} -- "${label}" --> ${failTransition.targetStepId};\n`;
            });
             // If there's no default failure path explicitly to End
             if (!step.onFailure.some(f => !f.errorCode)) {
                  if (definition.error_handler_step_id) {
                      mermaid += `  ${step.stepId} -- Fail (Unhandled) --> ${definition.error_handler_step_id};\n`;
                  } else if (index === definition.steps!.length - 1) {
                       // Implicitly end on failure if last step has no failure path
                       mermaid += `  ${step.stepId} -- Fail --> StopGraceful((Stop));\n`;
                  }
             }
        } else if (definition.error_handler_step_id) {
            // Default to global error handler if no specific onFailure
            mermaid += `  ${step.stepId} -- Fail --> ${definition.error_handler_step_id};\n`;
        } else if (index === definition.steps!.length - 1) {
             // Implicitly end on failure if last step has no failure path
              mermaid += `  ${step.stepId} -- Fail --> StopGraceful((Stop));\n`;
        }
    });

    if (definition.error_handler_step_id && definition.steps.some(s => s.stepId === definition.error_handler_step_id)) {
         mermaid += `  ${definition.error_handler_step_id} --> End;\n`; // Assume error handler leads to end
    }
     mermaid += `  style StopGraceful fill:#f9f,stroke:#333,stroke-width:2px\n`;


    return mermaid;
};


const WorkflowEditorView: React.FC<WorkflowEditorViewProps> = ({ workflowId, onSave, onCancel, onRun }) => {
    const [definition, setDefinition] = useState<Partial<WorkflowDefinition>>(createNewWorkflow());
    const [definitionText, setDefinitionText] = useState<string>(""); // YAML/JSON text
    const [mermaidDiagram, setMermaidDiagram] = useState<string>("");
    const [isLoading, setIsLoading] = useState(false);
    const [error, setError] = useState<string | null>(null);
    const [isDirty, setIsDirty] = useState(false);
    const [currentVersion, setCurrentVersion] = useState<number>(0);

     // Load definition
    useEffect(() => {
        if (workflowId) {
            setIsLoading(true);
            WorkflowServiceClient.getWorkflow({ id: workflowId })
                .then(response => {
                    if (response.workflow) {
                        setDefinition(response.workflow);
                         setCurrentVersion(response.workflow.version || 0);
                         setDefinitionText(yaml.dump(response.workflow)); // Convert loaded object to YAML
                         setMermaidDiagram(generateMermaidDiagram(response.workflow));
                         setIsDirty(false);
                    } else {
                        setError(`Workflow "${workflowId}" not found.`);
                    }
                })
                .catch(err => setError(err.message))
                .finally(() => setIsLoading(false));
        } else {
            const newWf = createNewWorkflow();
            setDefinition(newWf);
            setDefinitionText(yaml.dump(newWf)); // Initialize text area
            setMermaidDiagram(generateMermaidDiagram(newWf));
            setIsDirty(true);
            setCurrentVersion(0);
        }
    }, [workflowId]);

    // Update definition object when text changes (with validation)
    const handleTextChange = useCallback((event: any) => {
        const text = event.target.value;
        setDefinitionText(text);
        setIsDirty(true);
        try {
            const parsed = yaml.load(text);
            if (typeof parsed === 'object' && parsed !== null) {
                // TODO: Add proper schema validation using Zod or similar
                setDefinition(parsed as Partial<WorkflowDefinition>);
                setMermaidDiagram(generateMermaidDiagram(parsed as Partial<WorkflowDefinition>));
                setError(null); // Clear error on successful parse
            } else {
                 setError("Invalid YAML/JSON structure.");
            }
        } catch (parseError) {
            setError(`YAML/JSON Parsing Error: ${(parseError as Error).message}`);
            // Keep the mermaid diagram as is on parse error? Or clear it?
        }
    }, []);

    const handleSave = async () => {
         setError(null);
         setIsLoading(true);
         // Validation
         if (!definition.id?.trim()) { setError("Workflow ID is required."); setIsLoading(false); return; }
         if (!definition.steps || definition.steps.length === 0) { setError("Workflow must have at least one step."); setIsLoading(false); return; }
         // TODO: Add more schema validation here

        try {
             const workflowToSave: WorkflowDefinition = {
                 id: definition.id.trim(),
                 version: currentVersion,
                 $schemaVersion: "1.0",
                 description: definition.description || "",
                 steps: definition.steps,
                 error_handler_step_id: definition.error_handler_step_id,
             };
            await WorkflowServiceClient.saveWorkflow({ workflow: workflowToSave });
            onSave(workflowToSave); // Pass saved definition back
            setIsDirty(false);
        } catch (err) {
            setError(err instanceof Error ? err.message : "Failed to save workflow");
        } finally {
            setIsLoading(false);
        }
    };


    if (isLoading && workflowId) {
        return <div>Loading workflow {workflowId}...</div>;
    }

    // Don't show full error state if just loading new workflow
    if (error && !isLoading && workflowId) {
        return <div style={{ color: 'var(--vscode-errorForeground)' }}>Error: {error}</div>;
    }


    return (
        <div style={{ padding: '15px', display: 'flex', flexDirection: 'column', height: '100%', boxSizing: 'border-box', gap: '15px' }}>
            <h3>{workflowId ? `Edit Workflow: ${workflowId} (v${currentVersion})` : 'Create New Workflow'}</h3>

            <div style={{ display: 'flex', gap: '10px' }}>
                <VSCodeTextField
                    value={definition.id || ""}
                    onInput={(e: any) => {
                        setDefinition(prev => ({ ...prev, id: e.target.value }));
                        setDefinitionText(yaml.dump({ ...definition, id: e.target.value }));
                        setIsDirty(true);
                    }}
                    disabled={!!workflowId || isLoading}
                    required
                    placeholder="e.g., python-generate-and-test"
                    style={{ flex: 1 }}>
                    Workflow ID
                </VSCodeTextField>
                <VSCodeTextField
                    value={definition.description || ""}
                     onInput={(e: any) => {
                        setDefinition(prev => ({ ...prev, description: e.target.value }));
                        setDefinitionText(yaml.dump({ ...definition, description: e.target.value }));
                        setIsDirty(true);
                    }}
                    disabled={isLoading}
                    placeholder="e.g., Generates and tests Python code"
                    style={{ flex: 2 }}>
                    Description
                </VSCodeTextField>
            </div>

            <div style={{ display: 'flex', flexGrow: 1, gap: '10px', minHeight: 0 }}>
                 {/* Text Editor Pane */}
                 <div style={{ flex: 1, display: 'flex', flexDirection: 'column' }}>
                     <label htmlFor="workflow-definition-area" style={{ marginBottom: '5px', fontWeight: 500 }}>Definition (YAML/JSON)</label>
                     <VSCodeTextArea
                        id="workflow-definition-area"
                        value={definitionText}
                        onInput={handleTextChange}
                        disabled={isLoading}
                        rows={15} // Adjust as needed
                        resize="vertical"
                        style={{ width: '100%', flexGrow: 1, fontFamily: 'var(--vscode-editor-font-family)' }}
                     />
                     {error && <div style={{ color: 'var(--vscode-errorForeground)', fontSize: '12px', marginTop: '5px' }}>{error}</div>}
                 </div>

                 {/* Visualization Pane */}
                 <div style={{ flex: 1, display: 'flex', flexDirection: 'column', minWidth: 0 }}>
                    <span style={{ marginBottom: '5px', fontWeight: 500 }}>Visual Preview</span>
                    <div style={{ flexGrow: 1, overflow: 'auto', border: '1px solid var(--vscode-input-border)', borderRadius: '3px', background: 'var(--vscode-editor-background)', padding: '10px' }}>
                        <MermaidBlock code={mermaidDiagram} />
                     </div>
                 </div>
            </div>

            <div style={{ display: 'flex', gap: '10px', flexShrink: 0 }}>
                <VSCodeButton onClick={handleSave} disabled={isLoading || !isDirty || !!error}>
                    {isLoading ? 'Saving...' : 'Save Workflow'}
                </VSCodeButton>
                 <VSCodeButton
                    appearance="secondary"
                    onClick={() => workflowId && !error && onRun(workflowId)}
                    disabled={isLoading || !workflowId || !!error || isDirty} // Disable run if unsaved changes or error
                    title={isDirty ? "Save before running" : ""}
                 >
                    Run Workflow
                </VSCodeButton>
                <VSCodeButton appearance="secondary" onClick={onCancel} disabled={isLoading}>Cancel</VSCodeButton>
            </div>
        </div>
    );
};

export default WorkflowEditorView;
```

---

**19. Instance Monitor View (`webview-ui/src/components/workbench/InstancesView.tsx` - Basic List)**

```typescript
import React, { useState, useEffect } from 'react';
import { VSCodeDataGrid, VSCodeDataGridRow, VSCodeDataGridCell, VSCodeButton, VSCodeBadge } from "@vscode/webview-ui-toolkit/react";
import { WorkflowInstance, WorkflowStatus } from '@shared/workbench/types';
import { InstanceServiceClient } from '@/services/grpc-client'; // Assuming gRPC client exists

// Assume TraceViewer component exists and takes instanceId
// import TraceViewer from './TraceViewer';

const InstancesView: React.FC = () => {
    const [instances, setInstances] = useState<WorkflowInstance[]>([]);
    const [isLoading, setIsLoading] = useState(true);
    const [error, setError] = useState<string | null>(null);
    const [selectedInstanceId, setSelectedInstanceId] = useState<string | null>(null);

    const fetchInstances = async () => {
        setIsLoading(true);
        setError(null);
        try {
            // Assume ListWorkflowInstances returns { instances: WorkflowInstance[] }
            const response = await InstanceServiceClient.listWorkflowInstances({ limit: 50 }); // Example limit
            // Sort by start time descending
            const sortedInstances = (response.instances || []).sort((a, b) => (b.startTime || 0) - (a.startTime || 0));
            setInstances(sortedInstances);
        } catch (err) {
            setError(err instanceof Error ? err.message : "Failed to load workflow instances");
            console.error("Error fetching instances:", err);
        } finally {
            setIsLoading(false);
        }
    };

    useEffect(() => {
        fetchInstances();
        // TODO: Implement polling or websocket for real-time updates
        const interval = setInterval(fetchInstances, 5000); // Refresh every 5 seconds
        return () => clearInterval(interval);
    }, []);

    const handleCancel = async (instanceId: string) => {
        console.log("Cancel instance requested:", instanceId);
        // TODO: Call gRPC InstanceService.cancelWorkflowInstance({ instanceId })
        // After success, call fetchInstances()
    };

    const getStatusBadge = (status: WorkflowStatus) => {
        switch (status) {
            case 'running': return <VSCodeBadge style={{ background: 'var(--vscode-charts-yellow)' }}>Running</VSCodeBadge>;
            case 'succeeded': return <VSCodeBadge style={{ background: 'var(--vscode-charts-green)' }}>Succeeded</VSCodeBadge>;
            case 'failed': return <VSCodeBadge style={{ background: 'var(--vscode-errorForeground)' }}>Failed</VSCodeBadge>;
            case 'paused': return <VSCodeBadge>Paused</VSCodeBadge>;
            case 'cancelled': return <VSCodeBadge>Cancelled</VSCodeBadge>;
            default: return <VSCodeBadge>Idle</VSCodeBadge>;
        }
    };

    const formatDuration = (start?: number, end?: number) => {
        if (!start) return '-';
        const endTime = end || Date.now();
        const durationMs = endTime - start;
        if (durationMs < 1000) return `${durationMs}ms`;
        return `${(durationMs / 1000).toFixed(1)}s`;
    };

    return (
        <div style={{ padding: '10px 20px', display: 'flex', flexDirection: 'column', height: '100%', boxSizing: 'border-box' }}>
            <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center', marginBottom: '10px' }}>
                <h2>Workflow Instances</h2>
                 <VSCodeButton appearance="icon" title="Refresh" onClick={fetchInstances} disabled={isLoading}>
                    <span className="codicon codicon-refresh"></span>
                </VSCodeButton>
            </div>

            {isLoading && instances.length === 0 ? ( // Show loading only on initial load
                <div>Loading instances...</div>
            ) : error ? (
                <div style={{ color: 'var(--vscode-errorForeground)' }}>Error: {error}</div>
            ) : instances.length === 0 ? (
                <p>No workflow instances found.</p>
            ) : (
                <div style={{ flexGrow: 1, overflowY: 'auto', border: '1px solid var(--vscode-panel-border)', borderRadius: '3px' }}>
                    <VSCodeDataGrid aria-label="Workflow Instances">
                        <VSCodeDataGridRow row-type="header">
                            <VSCodeDataGridCell cell-type="columnheader" grid-column="1">Instance ID</VSCodeDataGridCell>
                            <VSCodeDataGridCell cell-type="columnheader" grid-column="2">Workflow</VSCodeDataGridCell>
                            <VSCodeDataGridCell cell-type="columnheader" grid-column="3">Status</VSCodeDataGridCell>
                            <VSCodeDataGridCell cell-type="columnheader" grid-column="4">Started</VSCodeDataGridCell>
                            <VSCodeDataGridCell cell-type="columnheader" grid-column="5">Duration</VSCodeDataGridCell>
                            <VSCodeDataGridCell cell-type="columnheader" grid-column="6">Actions</VSCodeDataGridCell>
                        </VSCodeDataGridRow>
                        {instances.map((inst) => (
                            <VSCodeDataGridRow key={inst.instanceId} onClick={() => setSelectedInstanceId(inst.instanceId)} style={{cursor: 'pointer'}}>
                                <VSCodeDataGridCell grid-column="1">{inst.instanceId.substring(3, 9)}</VSCodeDataGridCell>
                                <VSCodeDataGridCell grid-column="2">{inst.definitionId} (v{inst.definitionVersion})</VSCodeDataGridCell>
                                <VSCodeDataGridCell grid-column="3">{getStatusBadge(inst.status)}</VSCodeDataGridCell>
                                <VSCodeDataGridCell grid-column="4">{new Date(inst.startTime).toLocaleString()}</VSCodeDataGridCell>
                                <VSCodeDataGridCell grid-column="5">{formatDuration(inst.startTime, inst.endTime)}</VSCodeDataGridCell>
                                <VSCodeDataGridCell grid-column="6">
                                     <VSCodeButton
                                        appearance="icon"
                                        title="View Logs/Trace"
                                        onClick={(e) => { e.stopPropagation(); setSelectedInstanceId(inst.instanceId); }}>
                                        <span className="codicon codicon-output"></span>
                                    </VSCodeButton>
                                     {inst.status === 'running' && (
                                         <VSCodeButton
                                             appearance="icon"
                                             title="Cancel Workflow"
                                             onClick={(e) => { e.stopPropagation(); handleCancel(inst.instanceId); }}>
                                             <span className="codicon codicon-stop-circle"></span>
                                        </VSCodeButton>
                                    )}
                                    {/* Add Pause/Resume later */}
                                </VSCodeDataGridCell>
                            </VSCodeDataGridRow>
                        ))}
                    </VSCodeDataGrid>
                </div>
            )}
            {/* Conditionally render TraceViewer panel below or in a separate area */}
            {/* {selectedInstanceId && <TraceViewer instanceId={selectedInstanceId} />} */}
        </div>
    );
};

export default InstancesView;

```

---

**Conclusion:**

This concludes the illustrative code generation for the core Workbench components discussed in the design sessions. These files provide concrete examples of the new services (`BlueprintService`, `WorkflowService`, `SuggestionService`), the refactored core (`AgentInstance`), the extended schemas (`workbench/types.ts`, `db/schema.ts`), and the basic UI views (`BlueprintsView`, `WorkflowsView`, `InstancesView`, `SuggestionsView`, `WorkbenchRoot`, `WorkbenchViewProvider`) needed to realize the initial phases of the Cline Universal Code Orchestrator Workbench vision.

Further development would involve implementing the detailed logic within these structures, building out the remaining UI components (Editors, Trace Viewer, etc.), integrating the enhanced evaluation framework, and rigorously testing the entire system.