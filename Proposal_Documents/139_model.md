Okay, let's generate the illustrative code for the final key pieces discussed in our design sessions: the **Meta-Agents** (Evaluator, PromptOptimizer), the **Suggestion Service** and related schemas, and the **Suggestion Review UI** component.

**Reminder:** This remains illustrative code, focusing on structure and interaction patterns. It omits robust error handling, complex LLM prompting details, UI polish, full state management integration, and comprehensive testing.

---

**Document 2 (Continued): Source Code for Key Workbench Modules**

**11. Meta-Agent Blueprints (Example Definitions - YAML)**

These would be stored as `.yaml` files (e.g., in `~/Documents/Cline/Blueprints/`).

*   **`EvaluatorAgent.yaml`:**

```yaml
id: evaluator-agent
version: 1
$schemaVersion: "1.0"
description: Analyzes evaluation results for specific blueprint versions to identify failure patterns and performance metrics.
baseSystemPrompt: |
  You are an AI evaluation analysis agent. Your goal is to process structured evaluation results and identify key performance indicators and failure patterns.
  Input: You will receive aggregated results and error samples for a specific Agent Blueprint version run on one or more benchmarks.
  Output: Produce a structured JSON PerformanceReport containing: overall success rate, average resource usage (tokens, cost, duration), a list of the top 3 most frequent tool errors (tool name, error message/code, count), and a brief natural language summary of findings.
  Constraints: Base your analysis ONLY on the provided input data. Do not hallucinate results. Output ONLY the structured JSON report.
allowedTools:
  # Primarily reads data, might need to write report to scratchpad
  - read_scratchpad
  - write_scratchpad
  # Might need to query DB directly in advanced versions
  # - query_evaluation_db
defaultModelConfig:
  apiProvider: anthropic # Or another powerful model suitable for analysis
  apiModelId: claude-3-7-sonnet-20250219
```

*   **`PromptOptimizerAgent_V1_ToolDesc.yaml`:** (Constrained version)

```yaml
id: prompt-optimizer-agent-v1-tooldesc
version: 1
$schemaVersion: "1.0"
description: Suggests a single-sentence addition to a specific tool description in a system prompt to address a common tool failure.
baseSystemPrompt: |
  You are an AI prompt engineering assistant specialized in refining tool descriptions.
  Input: You will receive:
  1. The target Agent Blueprint's full system prompt.
  2. A specific tool name (e.g., 'replace_in_file') that frequently failed.
  3. The most frequent error message associated with that tool's failure (e.g., 'SEARCH block does not match').
  4. Optionally, a specific parameter related to the error (e.g., 'diff').
  Output: Generate a *single*, concise sentence that could be *added* to the existing description of the specified tool within the provided system prompt to help prevent the specific error. Explain your reasoning clearly in 1-2 sentences. You MUST use the 'propose_prompt_change' tool to output your suggestion.
  Example Output Tool Call:
  <propose_prompt_change>
  <targetBlueprintId>target-id</targetBlueprintId>
  <targetVersion>1</targetVersion>
  <suggestedPromptDiff>--- a/prompt\n+++ b/prompt\n@@ -10,1 +10,2 @@\n Tool Description...\n+Remember to ensure SEARCH blocks exactly match file content including whitespace.</rationale>
  <rationale>Adding emphasis on exact matching for SEARCH blocks may help prevent 'SEARCH block does not match' errors.</rationale>
  </propose_prompt_change>
allowedTools:
  - propose_prompt_change # Specific tool for outputting suggestions
defaultModelConfig:
  apiProvider: anthropic
  apiModelId: claude-3-7-sonnet-20250219 # Needs strong reasoning/instruction following
```

---

**12. Suggestion Service (`src/services/workbench/suggestions/SuggestionService.ts`)**

```typescript
import * as vscode from "vscode";
import { MetaAgentSuggestion, SuggestionStatus, ValidationStatus } from "@shared/workbench/types";
import { WorkflowLogger } from "@services/logging/WorkflowLogger"; // Assume DB access is via logger or dedicated DB service
import { Logger } from "@services/logging/Logger";
// Assume DB access layer exists
// import { db } from "@services/db";

export class SuggestionService {

    constructor(
        private logger: WorkflowLogger,
        private context: vscode.ExtensionContext
        // Inject DB access layer if separate from logger
    ) {}

    async storeSuggestion(suggestion: Omit<MetaAgentSuggestion, 'suggestionId' | 'timestamp' | 'status' | 'validationStatus'>): Promise<string> {
        const suggestionId = `sug_${uuidv4()}`;
        const timestamp = Date.now();
        const fullSuggestion: MetaAgentSuggestion = {
            ...suggestion,
            suggestionId,
            timestamp,
            status: 'pending_review',
            validationStatus: 'unvalidated',
        };

        Logger.info(`Storing new suggestion: ${suggestionId} for blueprint ${suggestion.targetBlueprintId} v${suggestion.targetBlueprintVersion}`);

        try {
            // TODO: Implement database insertion logic
            // await db.prepare("INSERT INTO MetaAgentSuggestions (...) VALUES (...)").run(...);
            await this.logger.logTraceEvent({
                 // No workflow/step ID for suggestion storage itself? Maybe assign a meta-workflow ID?
                 workflowInstanceId: 'meta-agent-runs',
                 eventType: 'SUGGESTION_GENERATED',
                 payload: fullSuggestion
            });
            // TODO: Notify UI that a new suggestion is available
        } catch (error) {
             Logger.error(`Failed to store suggestion ${suggestionId}`, error as Error);
             throw error; // Propagate error
        }
        return suggestionId;
    }

    async listPendingSuggestions(): Promise<MetaAgentSuggestion[]> {
        Logger.info("Fetching pending suggestions");
        try {
             // TODO: Implement database query logic
             // const rows = await db.prepare("SELECT * FROM MetaAgentSuggestions WHERE status = 'pending_review' ORDER BY timestamp DESC").all();
             // return rows as MetaAgentSuggestion[];
             return []; // Placeholder
        } catch (error) {
             Logger.error("Failed to list pending suggestions", error as Error);
             return [];
        }
    }

     async updateSuggestionStatus(suggestionId: string, status: SuggestionStatus, validationStatus?: ValidationStatus): Promise<void> {
         Logger.info(`Updating suggestion ${suggestionId} status to ${status}, validation: ${validationStatus || 'unchanged'}`);
         try {
             // TODO: Implement database update logic
             // const params: any = { status, suggestionId };
             // let query = "UPDATE MetaAgentSuggestions SET status = @status";
             // if (validationStatus) {
             //     query += ", validationStatus = @validationStatus";
             //     params.validationStatus = validationStatus;
             // }
             // query += " WHERE suggestionId = @suggestionId";
             // await db.prepare(query).run(params);

             // TODO: Notify UI of status change
         } catch (error) {
             Logger.error(`Failed to update suggestion status ${suggestionId}`, error as Error);
         }
    }

     async getSuggestion(suggestionId: string): Promise<MetaAgentSuggestion | undefined> {
         Logger.info(`Fetching suggestion ${suggestionId}`);
         try {
            // TODO: Implement database query logic
            // const row = await db.prepare("SELECT * FROM MetaAgentSuggestions WHERE suggestionId = ?").get(suggestionId);
            // return row as MetaAgentSuggestion | undefined;
            return undefined; // Placeholder
         } catch (error) {
             Logger.error(`Failed to get suggestion ${suggestionId}`, error as Error);
             return undefined;
         }
     }
}
```

---

**13. AgentInstance Tool Handler for `propose_prompt_change` (`src/core/agent/AgentInstance.ts` - Snippet)**

```typescript
// Inside AgentInstance.presentAssistantMessage, switch (block.name)

case 'propose_prompt_change': {
    // 1. Validate parameters
    const targetBlueprintId = block.params.targetBlueprintId;
    const targetVersionStr = block.params.targetVersion;
    const suggestedPromptDiff = block.params.suggestedPromptDiff;
    const rationale = block.params.rationale;

    if (!targetBlueprintId || !targetVersionStr || !suggestedPromptDiff || !rationale) {
        this.stepError = "Missing required parameters for propose_prompt_change";
        this.stepErrorCode = "MISSING_PARAMS_PROPOSE_CHANGE";
        break; // Failed step
    }
    const targetVersion = parseInt(targetVersionStr, 10);
    if (isNaN(targetVersion)) {
         this.stepError = "Invalid targetVersion for propose_prompt_change (must be a number)";
         this.stepErrorCode = "INVALID_PARAM_PROPOSE_CHANGE";
         break; // Failed step
    }

    // 2. Call Suggestion Service (Needs access via dependencies)
    const suggestionService = this.dependencies.suggestionService; // Assuming it's injected
    if (!suggestionService) {
         this.stepError = "SuggestionService not available";
         this.stepErrorCode = "SERVICE_UNAVAILABLE";
         break;
    }

    try {
        const suggestionId = await suggestionService.storeSuggestion({
            sourceMetaAgentId: this.config.id, // ID of the agent *making* the suggestion
            targetBlueprintId,
            targetBlueprintVersion: targetVersion,
            changeType: 'prompt_modification',
            proposedChange: suggestedPromptDiff,
            rationale,
            // Assuming confidence/auto-apply logic is handled elsewhere or later
        });
        toolSuccess = true;
        toolResult = `[Suggestion ${suggestionId} created successfully and submitted for review.]`;
    } catch (error) {
        toolSuccess = false;
        toolError = error instanceof Error ? error.message : String(error);
        toolErrorCode = "SUGGESTION_STORE_FAILED";
        this.stepError = toolError;
        this.stepErrorCode = toolErrorCode;
    }

    // Log and format result as done for other tools...
    // ...
    break;
}
```

---

**14. Suggestion Review UI (`webview-ui/src/components/workbench/SuggestionsView.tsx` - Basic Structure)**

```typescript
import React, { useState, useEffect } from 'react';
import { VSCodeDataGrid, VSCodeDataGridRow, VSCodeDataGridCell, VSCodeButton, VSCodeBadge } from "@vscode/webview-ui-toolkit/react";
import { MetaAgentSuggestion } from '@shared/workbench/types';
import { SuggestionServiceClient } from '@/services/grpc-client'; // Assuming gRPC client exists
// Assume a DiffViewer component exists or import one like 'react-diff-viewer'
// import ReactDiffViewer from 'react-diff-viewer';

const SuggestionsView: React.FC = () => {
    const [suggestions, setSuggestions] = useState<MetaAgentSuggestion[]>([]);
    const [isLoading, setIsLoading] = useState(true);
    const [error, setError] = useState<string | null>(null);
    const [selectedSuggestion, setSelectedSuggestion] = useState<MetaAgentSuggestion | null>(null);

    const fetchSuggestions = async () => {
        setIsLoading(true);
        setError(null);
        try {
            const response = await SuggestionServiceClient.listPendingSuggestions({});
            setSuggestions(response.suggestions || []); // Adjust based on actual protobuf response
        } catch (err) {
            setError(err instanceof Error ? err.message : "Failed to load suggestions");
            console.error("Error fetching suggestions:", err);
        } finally {
            setIsLoading(false);
        }
    };

    useEffect(() => {
        fetchSuggestions();
        // TODO: Add mechanism to refresh list (e.g., polling, websocket)
    }, []);

    const handleApply = (suggestionId: string, validate: boolean) => {
        console.log(`Action: ${validate ? 'Apply & Validate' : 'Apply'} for ${suggestionId}`);
        // TODO: Call gRPC SuggestionService.applySuggestion({ suggestionId, validate })
        // This backend call would then trigger BlueprintService.saveBlueprint and potentially EvaluationService/BenchmarkRunner
        // After success, call fetchSuggestions() to refresh the list.
    };

    const handleIgnore = (suggestionId: string) => {
         console.log(`Action: Ignore for ${suggestionId}`);
        // TODO: Call gRPC SuggestionService.updateSuggestionStatus({ suggestionId, status: 'ignored' })
        // After success, call fetchSuggestions() to refresh the list.
    };

    const handleEdit = (suggestionId: string) => {
         console.log(`Action: Edit & Apply for ${suggestionId}`);
        // TODO: Fetch suggestion details, open Blueprint editor with proposed changes applied
    };


    if (isLoading) return <div>Loading suggestions...</div>;
    if (error) return <div style={{ color: 'var(--vscode-errorForeground)' }}>Error: {error}</div>;

    return (
        <div style={{ padding: '10px 20px', display: 'flex', height: '100%', boxSizing: 'border-box' }}>
            {/* Suggestion List Pane */}
            <div style={{ width: '300px', marginRight: '10px', overflowY: 'auto', borderRight: '1px solid var(--vscode-panel-border)'}}>
                 <h3>Pending Suggestions</h3>
                 {suggestions.length === 0 ? (
                     <p>No pending suggestions.</p>
                 ) : (
                    <VSCodeDataGrid aria-label="Pending Suggestions">
                        <VSCodeDataGridRow row-type="header">
                            <VSCodeDataGridCell cell-type="columnheader" grid-column="1">Target</VSCodeDataGridCell>
                            <VSCodeDataGridCell cell-type="columnheader" grid-column="2">Source</VSCodeDataGridCell>
                            <VSCodeDataGridCell cell-type="columnheader" grid-column="3">Date</VSCodeDataGridCell>
                        </VSCodeDataGridRow>
                        {suggestions.map((sug) => (
                             <VSCodeDataGridRow key={sug.suggestionId} onClick={() => setSelectedSuggestion(sug)} style={{cursor: 'pointer'}}>
                                <VSCodeDataGridCell grid-column="1">{sug.targetBlueprintId} (v{sug.targetBlueprintVersion})</VSCodeDataGridCell>
                                <VSCodeDataGridCell grid-column="2">{sug.sourceMetaAgentId}</VSCodeDataGridCell>
                                <VSCodeDataGridCell grid-column="3">{new Date(sug.timestamp).toLocaleDateString()}</VSCodeDataGridCell>
                            </VSCodeDataGridRow>
                         ))}
                     </VSCodeDataGrid>
                 )}
            </div>

            {/* Detail Pane */}
            <div style={{ flex: 1, overflowY: 'auto', paddingLeft: '10px' }}>
                 {selectedSuggestion ? (
                    <div>
                        <h3>Suggestion Details ({selectedSuggestion.suggestionId.substring(4, 10)})</h3>
                        <p><strong>Target:</strong> {selectedSuggestion.targetBlueprintId} (Version {selectedSuggestion.targetBlueprintVersion})</p>
                        <p><strong>Rationale:</strong> {selectedSuggestion.rationale}</p>
                        <div>
                            <strong>Proposed Change (Diff):</strong>
                             {/* TODO: Integrate a proper Diff Viewer component */}
                             <pre style={{ background: 'var(--vscode-editor-background)', padding: '10px', borderRadius: '3px', border: '1px solid var(--vscode-editorGroup-border)', maxHeight: '300px', overflow: 'auto' }}>
                                 <code>{selectedSuggestion.proposedChange}</code>
                             </pre>
                         </div>
                         {/* TODO: Display validation status and results if available */}
                         <div style={{ marginTop: '15px', display: 'flex', gap: '10px' }}>
                             <VSCodeButton onClick={() => handleApply(selectedSuggestion.suggestionId, false)}>Apply</VSCodeButton>
                             <VSCodeButton appearance="secondary" onClick={() => handleApply(selectedSuggestion.suggestionId, true)}>Apply & Validate</VSCodeButton>
                             {/* <VSCodeButton appearance="secondary" onClick={() => handleEdit(selectedSuggestion.suggestionId)}>Edit & Apply</VSCodeButton> */}
                             <VSCodeButton appearance="secondary" onClick={() => handleIgnore(selectedSuggestion.suggestionId)}>Ignore</VSCodeButton>
                         </div>
                     </div>
                 ) : (
                     <p>Select a suggestion to view details.</p>
                 )}
            </div>
        </div>
    );
};

export default SuggestionsView;
```

---

**Conclusion:**

This set of files illustrates the practical implementation of the supervised adaptation loop. Key elements include defining Meta-Agent roles via `AgentBlueprints`, storing their output structurally using a `SuggestionService` and database table, and providing a dedicated UI (`SuggestionsView`) for users to review, validate, and apply these AI-generated improvement suggestions. While the implementation requires careful handling of state, asynchronous operations, and robust validation, it establishes the core mechanism for Cline to start learning from its own performance data under human guidance.

This represents a significant step towards the self-improving capabilities envisioned for the Universal Code Orchestrator Workbench.