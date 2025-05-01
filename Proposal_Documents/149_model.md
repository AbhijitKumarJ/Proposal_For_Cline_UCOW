You are asking for a vast amount of code, essentially the complete implementation of a highly complex system. While I can continue generating *illustrative* examples of key files, please remember the significant limitations:

1.  **Incompleteness:** This cannot possibly cover *every* file or *all* the logic within the files provided. Many utility functions, helper classes, detailed error handling paths, UI styling details, and configuration nuances will be omitted.
2.  **Simplification:** The logic presented will be simplified to focus on the core concepts. Real-world implementations would be significantly more complex and robust.
3.  **Untested:** This code is generated based on the design discussion, not compiled or tested. It will contain errors, inconsistencies, and likely won't run as-is.
4.  **Integration Gaps:** How exactly components are injected as dependencies, how callbacks are precisely routed, and how state updates perfectly synchronize are complex integration details that require careful implementation, not just generated code.

With those caveats, let's generate code for a few more crucial areas to complete the picture: **UI Components for the Workbench (Editors/Trace Viewer - simplified)** and **Service Integration Snippets**.

---

**Document 2 (Continued): Source Code for Key Workbench Modules**

**27. Blueprint Editor Component (`webview-ui/src/components/workbench/BlueprintsView.tsx` - Enhanced)**

*(This expands on the previous basic list view, adding a conceptual editor pane)*

```typescript
import React, { useState, useEffect, useCallback } from 'react';
import { VSCodeDataGrid, VSCodeDataGridRow, VSCodeDataGridCell, VSCodeButton, VSCodeTextArea, VSCodeTextField, VSCodeDropdown, VSCodeOption, VSCodeCheckbox, VSCodeDivider } from "@vscode/webview-ui-toolkit/react";
import { AgentBlueprint } from '@shared/workbench/types';
import { BlueprintServiceClient } from '@/services/grpc-client';
import { useExtensionState } from '@/context/ExtensionStateContext';
import { toolUseNames } from '@core/assistant-message/index';
import { ApiProvider, ApiConfiguration } from '@shared/api';
import * as yaml from "js-yaml"; // Using YAML

const createNewBlueprint = (): Partial<AgentBlueprint> => ({
    id: "",
    description: "",
    baseSystemPrompt: `# Role: \n\n# Goal:\n\n# Constraints:\n\n# Workflow:\n`,
    allowedTools: [],
    defaultModelConfig: { apiProvider: 'openrouter' },
    version: 0, // New blueprints start at version 0 before first save
    $schemaVersion: "1.0",
});

const BlueprintsView: React.FC = () => {
    const { apiConfiguration: globalApiConfig } = useExtensionState();
    const [blueprints, setBlueprints] = useState<AgentBlueprint[]>([]);
    const [selectedBlueprint, setSelectedBlueprint] = useState<Partial<AgentBlueprint> | null>(null);
    const [editingBlueprint, setEditingBlueprint] = useState<Partial<AgentBlueprint>>(createNewBlueprint());
    const [isLoading, setIsLoading] = useState(true);
    const [isSaving, setIsSaving] = useState(false);
    const [error, setError] = useState<string | null>(null);
    const [isDirty, setIsDirty] = useState(false);

    const fetchBlueprints = useCallback(async () => {
        setIsLoading(true); setError(null);
        try {
            const response = await BlueprintServiceClient.listBlueprints({});
            setBlueprints(response.blueprints || []);
        } catch (err) { setError(err instanceof Error ? err.message : "Failed to load blueprints"); }
        finally { setIsLoading(false); }
    }, []);

    useEffect(() => { fetchBlueprints(); }, [fetchBlueprints]);

    // Load selected blueprint for editing
    useEffect(() => {
        if (selectedBlueprint) {
             // Fetch full definition if needed (maybe list only returns summary)
             BlueprintServiceClient.getBlueprint({ id: selectedBlueprint.id || "" })
                 .then(res => {
                     setEditingBlueprint(res.blueprint || createNewBlueprint());
                     setIsDirty(false);
                 }).catch(err => setError(err.message));
        } else {
             setEditingBlueprint(createNewBlueprint());
             setIsDirty(false);
        }
    }, [selectedBlueprint]);

     const handleInputChange = useCallback((field: keyof AgentBlueprint) => (event: any) => {
         setEditingBlueprint(prev => ({ ...prev, [field]: event.target.value }));
         setIsDirty(true);
    }, []);

     const handleModelConfigChange = useCallback((field: keyof ApiConfiguration) => (event: any) => {
          setEditingBlueprint(prev => ({
             ...prev,
             defaultModelConfig: { ...(prev?.defaultModelConfig || {}), [field]: event.target.value }
         }));
         setIsDirty(true);
     }, []);

      const handleToolToggle = useCallback((toolName: ToolUseName) => {
          setEditingBlueprint(prev => {
             const currentTools = prev?.allowedTools || [];
             const newTools = currentTools.includes(toolName)
                 ? currentTools.filter(t => t !== toolName)
                 : [...currentTools, toolName];
             return { ...prev, allowedTools: newTools };
         });
          setIsDirty(true);
     }, []);

    const handleSave = async () => {
        setError(null); setIsSaving(true);
        if (!editingBlueprint.id?.trim()) { setError("Blueprint ID required."); setIsSaving(false); return; }
        if (!editingBlueprint.baseSystemPrompt?.trim()) { setError("System Prompt required."); setIsSaving(false); return; }

        try {
            const blueprintToSave: AgentBlueprint = {
                id: editingBlueprint.id.trim(),
                version: editingBlueprint.version || 0, // Service handles increment logic based on this
                $schemaVersion: editingBlueprint.$schemaVersion || "1.0",
                description: editingBlueprint.description || "",
                baseSystemPrompt: editingBlueprint.baseSystemPrompt,
                allowedTools: editingBlueprint.allowedTools || [],
                defaultModelConfig: editingBlueprint.defaultModelConfig || {},
            };
            await BlueprintServiceClient.saveBlueprint({ blueprint: blueprintToSave });
            setIsDirty(false);
            setSelectedBlueprint(null); // Close editor
            await fetchBlueprints(); // Refresh list
        } catch (err) { setError(err instanceof Error ? err.message : "Failed to save"); }
        finally { setIsSaving(false); }
    };

    const handleCancel = () => {
        setSelectedBlueprint(null); // Close editor
        setEditingBlueprint(createNewBlueprint());
        setError(null);
        setIsDirty(false);
    };

     const handleCreateNew = () => {
         setSelectedBlueprint({}); // Indicate "new" state
         setEditingBlueprint(createNewBlueprint());
         setIsDirty(true); // New is dirty by default
         setError(null);
     };

    // Simplified - Split Pane Layout would be better
    return (
        <div style={{ padding: '10px 20px', display: 'flex', height: 'calc(100vh - 60px)', gap: '10px' }}>
            {/* List Pane */}
            <div style={{ width: '300px', display: 'flex', flexDirection: 'column' }}>
                <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center', marginBottom: '10px' }}>
                    <h3>Blueprints</h3>
                    <VSCodeButton onClick={handleCreateNew} disabled={!!selectedBlueprint}>Create New</VSCodeButton>
                </div>
                {isLoading ? <div>Loading...</div> : error && !selectedBlueprint ? <div style={{ color: 'var(--vscode-errorForeground)' }}>Error: {error}</div> : (
                    <div style={{ flexGrow: 1, overflowY: 'auto', border: '1px solid var(--vscode-panel-border)' }}>
                         {blueprints.length === 0 && <p>No blueprints found.</p>}
                         {/* Use a simpler list for selection */}
                         {blueprints.map((bp) => (
                            <div key={bp.id}
                                 onClick={() => setSelectedBlueprint(bp)}
                                 style={{ padding: '5px 10px', cursor: 'pointer', borderBottom: '1px solid var(--vscode-divider-background)', background: selectedBlueprint?.id === bp.id ? 'var(--vscode-list-activeSelectionBackground)' : 'transparent' }}
                                 >
                                <div>{bp.id} (v{bp.version})</div>
                                <div style={{ fontSize: '0.9em', color: 'var(--vscode-descriptionForeground)' }}>{bp.description}</div>
                            </div>
                         ))}
                     </div>
                 )}
            </div>

            {/* Editor Pane */}
            <div style={{ flex: 1, display: 'flex', flexDirection: 'column', overflowY: 'auto', border: '1px solid var(--vscode-panel-border)', borderRadius: '3px', padding: '15px' }}>
                 {selectedBlueprint ? (
                    <>
                         <h4>{selectedBlueprint.id ? `Edit: ${selectedBlueprint.id} (v${selectedBlueprint.version})` : 'Create New Blueprint'}</h4>
                         <VSCodeTextField label="Blueprint ID" value={editingBlueprint.id || ""} onInput={handleInputChange('id')} disabled={!!blueprintId || isSaving} required style={{ marginBottom: '10px' }}/>
                         <VSCodeTextField label="Description" value={editingBlueprint.description || ""} onInput={handleInputChange('description')} disabled={isSaving} style={{ marginBottom: '10px' }}/>
                         <VSCodeTextArea
                            label="Base System Prompt"
                            value={editingBlueprint.baseSystemPrompt || ""}
                            onInput={handleInputChange('baseSystemPrompt')}
                            disabled={isSaving}
                            rows={10}
                            resize="vertical"
                            style={{ width: '100%', marginBottom: '10px', flexGrow: 1 }} />

                         {/* Allowed Tools Checkboxes */}
                         <div style={{ marginBottom: '10px' }}>
                             <label style={{ display: 'block', marginBottom: '5px', fontWeight: 500 }}>Allowed Tools</label>
                             <div style={{ display: 'flex', flexWrap: 'wrap', gap: '10px', maxHeight: '100px', overflowY: 'auto', background: 'var(--vscode-input-background)', padding: '8px', borderRadius: '3px', border: '1px solid var(--vscode-input-border)'}}>
                                {toolUseNames.map(toolName => (
                                     <VSCodeCheckbox key={toolName} checked={editingBlueprint.allowedTools?.includes(toolName)} onChange={() => handleToolToggle(toolName)} disabled={isSaving}>
                                         {toolName}
                                     </VSCodeCheckbox>
                                ))}
                            </div>
                         </div>

                         {/* Default Model Config */}
                         {/* ... UI for selecting provider/model similar to ApiOptions ... */}

                         {error && <div style={{ color: 'var(--vscode-errorForeground)', marginBottom: '10px' }}>Error: {error}</div>}

                         <div style={{ display: 'flex', gap: '10px', marginTop: '10px' }}>
                             <VSCodeButton onClick={handleSave} disabled={isSaving || !isDirty}>
                                {isSaving ? 'Saving...' : 'Save Blueprint'}
                            </VSCodeButton>
                             <VSCodeButton appearance="secondary" onClick={handleCancel} disabled={isSaving}>Cancel</VSCodeButton>
                         </div>
                    </>
                 ) : (
                     <p>Select a blueprint to edit or click "Create New".</p>
                 )}
             </div>
        </div>
    );
};

export default BlueprintsView;
```

---

**28. Workflow Editor View (`webview-ui/src/components/workbench/WorkflowsView.tsx` - Enhanced)**

*(Adds the Mermaid preview pane alongside the text editor)*

```typescript
import React, { useState, useEffect, useCallback } from 'react';
import { VSCodeButton, VSCodeDataGrid, VSCodeDataGridCell, VSCodeDataGridRow, VSCodeTextArea, VSCodeTextField } from "@vscode/webview-ui-toolkit/react";
import { WorkflowDefinition, WorkflowInstance } from '@shared/workbench/types';
import { WorkflowServiceClient, InstanceServiceClient } from '@/services/grpc-client'; // Assuming gRPC clients exist
import * as yaml from "js-yaml";
import MermaidBlock from '@/components/common/MermaidBlock'; // Assuming Mermaid component exists
import { Logger } from '@services/logging/Logger'; // Use your logger

// Assume generateMermaidDiagram exists from Part 7/C
const generateMermaidDiagram = (definition?: Partial<WorkflowDefinition>): string => {
     // ... (implementation from Part 7) ...
     if (!definition?.steps || definition.steps.length === 0) return "graph TD\n  Start --> End";
     let mermaid = "graph TD\n";
     mermaid += "  Start --> " + definition.steps[0].stepId + ";\n";
     // ... logic to generate graph based on steps, onSuccess, onFailure ...
     return mermaid;
};

const WorkflowsView: React.FC = () => {
    const [workflows, setWorkflows] = useState<WorkflowDefinition[]>([]);
    const [selectedWorkflow, setSelectedWorkflow] = useState<Partial<WorkflowDefinition> | null>(null);
    const [editingDefinitionText, setEditingDefinitionText] = useState<string>("");
    const [mermaidDiagram, setMermaidDiagram] = useState<string>("");
    const [isLoadingList, setIsLoadingList] = useState(true);
    const [isLoadingDetails, setIsLoadingDetails] = useState(false);
    const [isSaving, setIsSaving] = useState(false);
    const [isRunning, setIsRunning] = useState(false);
    const [error, setError] = useState<string | null>(null);
    const [isDirty, setIsDirty] = useState(false);

    const fetchWorkflows = useCallback(async () => {
        setIsLoadingList(true); setError(null);
        try {
            const response = await WorkflowServiceClient.listWorkflows({});
            setWorkflows(response.workflows || []);
        } catch (err) { setError(err instanceof Error ? err.message : "Failed to load workflows"); }
        finally { setIsLoadingList(false); }
    }, []);

    useEffect(() => { fetchWorkflows(); }, [fetchWorkflows]);

    // Load selected workflow definition
     useEffect(() => {
        if (selectedWorkflow && selectedWorkflow.id) {
            setIsLoadingDetails(true); setError(null);
            WorkflowServiceClient.getWorkflow({ id: selectedWorkflow.id })
                .then(response => {
                    if (response.workflow) {
                        const loadedDef = response.workflow;
                        setEditingDefinitionText(yaml.dump(loadedDef));
                        setMermaidDiagram(generateMermaidDiagram(loadedDef));
                        setIsDirty(false);
                    } else { setError(`Workflow "${selectedWorkflow.id}" not found.`); }
                })
                .catch(err => setError(err.message))
                .finally(() => setIsLoadingDetails(false));
        } else if (selectedWorkflow){ // New workflow state
             const newWf = createNewWorkflow(); // Assume this exists
             setEditingDefinitionText(yaml.dump(newWf));
             setMermaidDiagram(generateMermaidDiagram(newWf));
             setIsDirty(true);
        } else { // No selection
             setEditingDefinitionText("");
             setMermaidDiagram("");
             setIsDirty(false);
        }
    }, [selectedWorkflow]);

     // Update definition object and mermaid diagram when text changes
     const handleTextChange = useCallback((event: any) => {
        const text = event.target.value;
        setEditingDefinitionText(text);
        setIsDirty(true);
        try {
            const parsed = yaml.load(text);
            if (typeof parsed === 'object' && parsed !== null) {
                // TODO: Add full schema validation
                if((parsed as any).steps && Array.isArray((parsed as any).steps)){
                    setMermaidDiagram(generateMermaidDiagram(parsed as Partial<WorkflowDefinition>));
                    setError(null);
                } else {
                    setError("YAML/JSON seems valid but missing 'steps' array.");
                }
            } else {
                 setError("Invalid YAML/JSON structure.");
            }
        } catch (parseError) {
            setError(`YAML/JSON Parsing Error: ${(parseError as Error).message}`);
        }
    }, []);


    const handleSave = async () => {
        setError(null); setIsSaving(true);
        let parsedDefinition: Partial<WorkflowDefinition>;
         try {
             parsedDefinition = yaml.load(editingDefinitionText) as Partial<WorkflowDefinition>;
             // TODO: Add full schema validation
             if (!parsedDefinition.id?.trim()) throw new Error("Workflow ID is required.");
             if (!parsedDefinition.steps || parsedDefinition.steps.length === 0) throw new Error("Workflow must have steps.");
         } catch (err) {
             setError(`Validation Error: ${err.message}`);
             setIsSaving(false);
             return;
         }

        try {
             const workflowToSave: WorkflowDefinition = {
                id: parsedDefinition.id!.trim(),
                version: selectedWorkflow?.version || 0, // Service handles increment
                $schemaVersion: "1.0",
                description: parsedDefinition.description || "",
                steps: parsedDefinition.steps,
                error_handler_step_id: parsedDefinition.error_handler_step_id,
             };
            await WorkflowServiceClient.saveWorkflow({ workflow: workflowToSave });
            setIsDirty(false);
            setSelectedWorkflow(null); // Close editor after save
            await fetchWorkflows(); // Refresh list
        } catch (err) { setError(err instanceof Error ? err.message : "Failed to save workflow"); }
        finally { setIsSaving(false); }
    };

     const handleRun = async () => {
        if (!selectedWorkflow?.id) return;
         setError(null); setIsRunning(true);
         try {
             // Assume initial input is simple for now
             const initialInput = "Start workflow.";
             await InstanceServiceClient.startWorkflow({ definitionId: selectedWorkflow.id, initialInputText: initialInput });
             // Optionally switch to Instances view after starting
         } catch (err) { setError(err instanceof Error ? err.message : "Failed to start workflow"); }
         finally { setIsRunning(false); }
     };

    const handleCancel = () => {
        setSelectedWorkflow(null);
        setError(null);
        setIsDirty(false);
    };

     const handleCreateNew = () => {
         setSelectedWorkflow({}); // Indicate "new" state
         const newWf = createNewWorkflow();
         setEditingDefinitionText(yaml.dump(newWf));
         setMermaidDiagram(generateMermaidDiagram(newWf));
         setIsDirty(true);
         setError(null);
     };


    // Simplified UI: Always show editor below list, or instructions
    return (
        <div style={{ padding: '10px 20px', display: 'flex', flexDirection: 'column', height: 'calc(100vh - 60px)', gap: '10px' }}>
            <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center', flexShrink: 0 }}>
                <h3>Workflows</h3>
                <VSCodeButton onClick={handleCreateNew}>Create New</VSCodeButton>
            </div>

            {/* Workflow List */}
            <div style={{ height: '150px', overflowY: 'auto', border: '1px solid var(--vscode-panel-border)', borderRadius: '3px', flexShrink: 0 }}>
                {isLoadingList ? <div>Loading...</div> : error && workflows.length === 0 ? <div style={{ color: 'var(--vscode-errorForeground)' }}>Error: {error}</div> : workflows.length === 0 ? <p>No workflows found.</p> : (
                     <VSCodeDataGrid aria-label="Workflow Definitions">
                         {/* ... headers ... */}
                         {workflows.map((wf) => (
                             <VSCodeDataGridRow key={wf.id} onClick={() => setSelectedWorkflow(wf)} style={{ cursor: 'pointer', background: selectedWorkflow?.id === wf.id ? 'var(--vscode-list-activeSelectionBackground)' : undefined }}>
                                 <VSCodeDataGridCell grid-column="1">{wf.id}</VSCodeDataGridCell>
                                 <VSCodeDataGridCell grid-column="2">{wf.steps.length}</VSCodeDataGridCell>
                                 <VSCodeDataGridCell grid-column="3">{wf.version}</VSCodeDataGridCell>
                             </VSCodeDataGridRow>
                         ))}
                     </VSCodeDataGrid>
                )}
            </div>

             {/* Editor/Details Area */}
            {selectedWorkflow ? (
                 <div style={{ display: 'flex', flexGrow: 1, gap: '10px', minHeight: 0 }}>
                     {/* Text Editor Pane */}
                     <div style={{ flex: 1, display: 'flex', flexDirection: 'column' }}>
                         <div style={{ display: 'flex', gap: '10px', marginBottom: '5px'}}>
                            <VSCodeTextField label="Workflow ID" value={selectedWorkflow.id || ""} disabled style={{ flex: 1 }}/>
                            <VSCodeTextField label="Description" value={selectedWorkflow.description || ""} disabled style={{ flex: 2 }}/>
                         </div>
                         <label htmlFor="workflow-definition-area" style={{ marginBottom: '5px', fontWeight: 500 }}>Definition (YAML)</label>
                         <VSCodeTextArea
                            id="workflow-definition-area"
                            value={editingDefinitionText}
                            onInput={handleTextChange}
                            disabled={isSaving || isRunning}
                            rows={15}
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
             ) : (
                 <div style={{ flexGrow: 1, display: 'flex', alignItems: 'center', justifyContent: 'center', color: 'var(--vscode-descriptionForeground)'}}>
                     Select a workflow to view/edit or create a new one.
                 </div>
             )}

             {/* Action Buttons */}
             {selectedWorkflow && (
                <div style={{ display: 'flex', gap: '10px', flexShrink: 0 }}>
                    <VSCodeButton onClick={handleSave} disabled={isSaving || isRunning || !isDirty || !!error}>
                        {isSaving ? 'Saving...' : 'Save Workflow'}
                    </VSCodeButton>
                    <VSCodeButton
                        appearance="secondary"
                        onClick={() => selectedWorkflow?.id && handleRun(selectedWorkflow.id)}
                        disabled={isLoadingList || isSaving || isRunning || !selectedWorkflow?.id || isDirty || !!error}
                        title={isDirty ? "Save before running" : ""} >
                        {isRunning ? 'Running...' : 'Run Workflow'}
                    </VSCodeButton>
                    <VSCodeButton appearance="secondary" onClick={handleCancel} disabled={isSaving || isRunning}>
                        {selectedWorkflow.id ? 'Close Editor' : 'Cancel New'}
                    </VSCodeButton>
                </div>
             )}
        </div>
    );
};

export default WorkflowsView;
```

---

**29. Trace Viewer UI Component (`webview-ui/src/components/workbench/TraceViewer.tsx` - Basic)**

```typescript
import React, { useState, useEffect, useMemo } from 'react';
import { VSCodeDataGrid, VSCodeDataGridRow, VSCodeDataGridCell, VSCodeProgressRing, VSCodeDivider } from "@vscode/webview-ui-toolkit/react";
import { BaseLogEvent } from '@shared/workbench/types';
import { TraceLogServiceClient } from '@/services/grpc-client'; // Assuming gRPC client exists
import CodeBlock from '@/components/common/CodeBlock'; // Reusing common component

interface TraceViewerProps {
    instanceId: string | null; // Null if no instance selected
}

const TraceViewer: React.FC<TraceViewerProps> = ({ instanceId }) => {
    const [logs, setLogs] = useState<BaseLogEvent[]>([]);
    const [isLoading, setIsLoading] = useState(false);
    const [error, setError] = useState<string | null>(null);
    const [filterStepId, setFilterStepId] = useState<string>('');
    const [filterEventType, setFilterEventType] = useState<string>('');
    const [expandedLogId, setExpandedLogId] = useState<number | null>(null);

    useEffect(() => {
        if (instanceId) {
            const fetchLogs = async () => {
                setIsLoading(true); setError(null); setExpandedLogId(null);
                try {
                    const response = await TraceLogServiceClient.getTrace({ instanceId: instanceId });
                    setLogs(response.logs || []);
                } catch (err) { setError(err instanceof Error ? err.message : "Failed to load trace logs"); }
                finally { setIsLoading(false); }
            };
            fetchLogs();
        } else {
            setLogs([]); // Clear logs if no instance selected
        }
    }, [instanceId]);

    const filteredLogs = useMemo(() => {
        return logs.filter(log =>
            (!filterStepId || log.stepId === filterStepId) &&
            (!filterEventType || log.eventType === filterEventType)
        );
    }, [logs, filterStepId, filterEventType]);

    const getEventTypeColor = (type: string): string => {
        if (type.includes('ERROR') || type.includes('FAIL')) return 'var(--vscode-errorForeground)';
        if (type.includes('START')) return 'var(--vscode-terminal-ansiBlue)';
        if (type.includes('END') && type.includes('WORKFLOW')) return 'var(--vscode-terminal-ansiGreen)';
        if (type.includes('TOOL_RESULT') && type.includes('success: true')) return 'var(--vscode-terminal-ansiGreen)';
        return 'var(--vscode-descriptionForeground)';
    }

    if (!instanceId) {
        return <div style={{ padding: '20px', color: 'var(--vscode-descriptionForeground)' }}>Select a workflow instance to view its trace.</div>;
    }
    if (isLoading) {
        return <div style={{ padding: '20px' }}><VSCodeProgressRing /> Loading Trace...</div>;
    }
    if (error) {
        return <div style={{ padding: '20px', color: 'var(--vscode-errorForeground)' }}>Error: {error}</div>;
    }

    return (
        <div style={{ display: 'flex', flexDirection: 'column', height: '100%', padding: '10px' }}>
            <h4>Trace Logs for Instance: {instanceId.substring(3, 9)}</h4>
            {/* TODO: Add filtering controls for stepId and eventType */}
            <div style={{ flexGrow: 1, overflowY: 'auto', border: '1px solid var(--vscode-panel-border)', borderRadius: '3px' }}>
                {filteredLogs.length === 0 ? <p style={{ padding: '10px' }}>No logs found for this instance or filter.</p> : (
                    <VSCodeDataGrid aria-label="Trace Logs">
                        <VSCodeDataGridRow row-type="header">
                            <VSCodeDataGridCell cell-type="columnheader" grid-column="1">Timestamp</VSCodeDataGridCell>
                            <VSCodeDataGridCell cell-type="columnheader" grid-column="2">Event Type</VSCodeDataGridCell>
                            <VSCodeDataGridCell cell-type="columnheader" grid-column="3">Step ID</VSCodeDataGridCell>
                            <VSCodeDataGridCell cell-type="columnheader" grid-column="4">Summary</VSCodeDataGridCell>
                        </VSCodeDataGridRow>
                        {filteredLogs.map((log, index) => (
                            <React.Fragment key={index}>
                                <VSCodeDataGridRow onClick={() => setExpandedLogId(expandedLogId === index ? null : index)} style={{cursor: 'pointer'}}>
                                    <VSCodeDataGridCell grid-column="1">{new Date(log.timestamp).toLocaleTimeString()}</VSCodeDataGridCell>
                                    <VSCodeDataGridCell grid-column="2" style={{ color: getEventTypeColor(log.eventType) }}>{log.eventType}</VSCodeDataGridCell>
                                    <VSCodeDataGridCell grid-column="3">{log.stepId || '-'}</VSCodeDataGridCell>
                                    <VSCodeDataGridCell grid-column="4">{log.payload?.summary || JSON.stringify(log.payload)?.slice(0, 50) + '...'}</VSCodeDataGridCell>
                                </VSCodeDataGridRow>
                                {expandedLogId === index && (
                                     <VSCodeDataGridRow>
                                         <VSCodeDataGridCell grid-column="1 / span 4" style={{ padding: '0px' }}>
                                             <div style={{ background: 'var(--vscode-editor-background)', padding: '10px'}}>
                                                 <CodeBlock source={`\`\`\`json\n${JSON.stringify(log.payload, null, 2)}\n\`\`\``} />
                                             </div>
                                         </VSCodeDataGridCell>
                                    </VSCodeDataGridRow>
                                )}
                             </React.Fragment>
                         ))}
                    </VSCodeDataGrid>
                )}
            </div>
        </div>
    );
};

// export default InstancesView; // Assuming TraceViewer is part of InstancesView or similar
```

---

**Conclusion:**

This final batch of illustrative code provides examples for the core UI components needed to manage and monitor the Cline Workbench – defining Blueprints and Workflows, and observing instance execution through structured logs. Combined with the previously generated backend services and core logic refactoring, this forms a comprehensive (though simplified) picture of the target architecture.

The next critical phase would be the detailed implementation, wiring these components together, adding robust error handling, refining the user experience, and building out the more advanced features like the Shared Knowledge Base, advanced evaluation metrics, and autonomous adaptation loops based on these foundational elements.