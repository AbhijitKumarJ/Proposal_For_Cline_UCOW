Okay, generating more illustrative source code for the **Cline Universal Code Orchestrator Workbench**. This time, we'll focus on the core `WorkflowOrchestrator` and the enhanced `evals/` components needed for richer metrics.

**Reminder:** This is still illustrative, focusing on structure and key logic flows. Many details (error handling, edge cases, full UI integration, performance optimization) are omitted or simplified.

---

**Document 2 (Continued): Source Code for Key Workbench Modules**

**8. Workflow Orchestrator (`src/services/workbench/orchestration/WorkflowOrchestrator.ts`)**

```typescript
import * as vscode from "vscode";
import { Anthropic } from "@anthropic-ai/sdk";
import { v4 as uuidv4 } from 'uuid';
import { AgentBlueprint, AgentStepResult, UserContent, WorkflowDefinition, WorkflowInstance, WorkflowStatus, WorkflowStep } from "@shared/workbench/types";
import { AgentInstance } from "@core/agent/AgentInstance";
import { BlueprintService } from "../blueprints/BlueprintService";
import { WorkflowLogger } from "@services/logging/WorkflowLogger";
import { Controller } from "@core/controller"; // Needed for 'ask' callback
import { McpHub } from "@services/mcp/McpHub";
import { ensureTaskDirectoryExists, fileExistsAtPath, getSavedWorkflowInstance, saveWorkflowInstance } from "@core/storage/disk"; // Assuming these helpers exist/are adapted
import { Logger } from "@services/logging/Logger";
import { formatResponse } from "@core/prompts/responses";
import { ClineAskResponse } from "@shared/WebviewMessage";

// Dependencies needed by AgentInstance (example)
interface AgentDependencies {
    logger: WorkflowLogger;
    controller: Controller; // For routing 'ask' approval requests
    mcpHub?: McpHub;
    // ... other services like TerminalManager, BrowserSession, FileContextTracker...
}

export class WorkflowOrchestrator {
    private activeInstances: Map<string, WorkflowInstance> = new Map();
    private isInitializing: boolean = true;

    constructor(
        private blueprintService: BlueprintService,
        private logger: WorkflowLogger,
        private agentDependencies: AgentDependencies, // Inject dependencies needed by AgentInstance
        private context: vscode.ExtensionContext // For storage path
    ) {}

    async initialize(): Promise<void> {
        await this.loadAndMarkFailedInstances();
        this.isInitializing = false;
        Logger.info("WorkflowOrchestrator initialized.");
    }

    async startWorkflow(definitionId: string, initialInputText: string): Promise<string> {
        if (this.isInitializing) {
            throw new Error("Orchestrator is still initializing.");
        }
        Logger.info(`Starting workflow from definition: ${definitionId}`);

        const definition = await this.blueprintService.getWorkflowDefinition(definitionId); // Assume BlueprintService handles workflows too for now
        if (!definition) {
            throw new Error(`Workflow definition not found: ${definitionId}`);
        }

        const instanceId = `wf_${uuidv4()}`;
        const initialUserContent: UserContent = [{ type: 'text', text: initialInputText }];
        const instance: WorkflowInstance = {
            instanceId,
            definitionId,
            definitionVersion: definition.version,
            status: 'running',
            currentStepIndex: 0,
            startTime: Date.now(),
            conversationHistory: [{ role: 'user', content: initialUserContent }],
            stepResults: [],
            scratchpad: {},
        };

        await this.logger.logTraceEvent({ workflowInstanceId: instanceId, eventType: 'WORKFLOW_START', payload: { definitionId, initialInputSummary: initialInputText.slice(0, 100) } });

        this.activeInstances.set(instanceId, instance);
        await this._saveWorkflowInstance(instance); // Persist initial state

        // Start execution asynchronously
        this._executeWorkflowStep(instanceId, 0).catch(err => {
            Logger.error(`Unhandled error during workflow execution for ${instanceId}`, err);
            // Attempt to mark as failed
            const failedInstance = this.activeInstances.get(instanceId);
            if (failedInstance) {
                failedInstance.status = 'failed';
                failedInstance.error = `Orchestrator error: ${err.message}`;
                failedInstance.endTime = Date.now();
                this._saveWorkflowInstance(failedInstance).finally(() => {
                     this.activeInstances.delete(instanceId);
                     // TODO: Notify UI of failure
                });
            }
        });

        // TODO: Notify UI that workflow started

        return instanceId;
    }

    private async _executeWorkflowStep(instanceId: string, stepIndex: number): Promise<void> {
        const instance = this.activeInstances.get(instanceId);
        if (!instance || instance.status !== 'running') {
            Logger.warn(`Workflow ${instanceId} execution halted or already completed. Status: ${instance?.status}`);
            if(instance) this.activeInstances.delete(instanceId); // Clean up ended instance
            return;
        }

        const definition = await this.blueprintService.getWorkflowDefinition(instance.definitionId);
        if (!definition || stepIndex >= definition.steps.length) {
            // Reached end of defined steps successfully
            instance.status = 'succeeded';
            instance.endTime = Date.now();
            this.logger.logTraceEvent({ workflowInstanceId: instanceId, eventType: 'WORKFLOW_END', payload: { status: 'succeeded', durationMs: instance.endTime - instance.startTime } });
            await this._saveWorkflowInstance(instance);
            this.activeInstances.delete(instanceId);
             // TODO: Notify UI of success
            return;
        }

        const stepDefinition = definition.steps[stepIndex];
        const agentInstanceId = `agent_${instanceId}_${stepIndex}`; // Unique ID for this agent run

        try {
            const blueprint = await this.blueprintService.getBlueprint(stepDefinition.blueprintId);
            if (!blueprint) {
                throw new Error(`Blueprint not found: ${stepDefinition.blueprintId}`);
            }

            // --- Prepare Input for AgentInstance ---
            // Get output from previous step (if any)
            const previousStepResult = instance.stepResults[stepIndex - 1];
            const stepInputContent: UserContent = stepIndex === 0
                ? (instance.conversationHistory[0].content as UserContent) // Use initial input
                : previousStepResult?.outputContent || []; // Use previous output

            // Combine blueprint prompt with step modifier
            const systemPrompt = `${blueprint.baseSystemPrompt}\n${stepDefinition.initialPromptModifier || ''}`.trim();

            // Provide conversation history segment
            const historySegment = instance.conversationHistory; // Pass full history for now

             // Instantiate AgentInstance
            const agentInstance = new AgentInstance(blueprint, {
                ...this.agentDependencies,
                // Need a way for AgentInstance to request approval via Controller's 'ask'
                // This requires passing the controller or a specific callback
                // approveToolCallback: async (askType, askPayload) => {
                //     return await this.agentDependencies.controller.askForToolApproval(askType, askPayload);
                // }
            });
             this.logger.logTraceEvent({ workflowInstanceId: instanceId, stepId: stepDefinition.stepId, agentInstanceId, eventType: 'STEP_START', payload: { blueprintId: blueprint.id } });

            // --- Execute Agent Step ---
            const stepResult: AgentStepResult = await agentInstance.executeStep(
                instanceId,
                stepDefinition.stepId,
                agentInstanceId,
                historySegment,
                stepInputContent
            );

            // --- Process Step Result ---
            // Update conversation history (MUST happen before next step input is determined)
            // 1. Add the User message that triggered this step (containing previous output)
            // Note: This was already added before calling executeStep in the previous iteration,
            // EXCEPT for the very first step where history starts with the initial input.
             if (stepIndex > 0 && previousStepResult) {
                 instance.conversationHistory.push({ role: 'user', content: previousStepResult.outputContent });
             }
             // 2. Add the Assistant message generated by this step
             instance.conversationHistory.push({ role: 'assistant', content: stepResult.finalAssistantMessageBlocks });

            // Update instance state
            instance.stepResults.push(stepResult);
            // TODO: Update scratchpad based on write_scratchpad calls logged within stepResult or via callbacks

            let nextStepIndex = -1; // Default to stop/fail

            if (stepResult.success) {
                const targetStepId = stepDefinition.onSuccess?.targetStepId;
                if (targetStepId) {
                    nextStepIndex = definition.steps.findIndex(s => s.stepId === targetStepId);
                } else {
                    nextStepIndex = stepIndex + 1; // Default sequential
                }
                 if (nextStepIndex >= definition.steps.length) { // Reached logical end
                    instance.status = 'succeeded';
                 }
            } else { // Step failed
                 instance.error = stepResult.error; // Store last error
                 const failureTransitions = stepDefinition.onFailure || [];
                 // Find matching error code transition
                 const specificHandler = failureTransitions.find(t => t.errorCode === stepResult.errorCode);
                 // Find default handler (no error code specified)
                 const defaultHandler = failureTransitions.find(t => !t.errorCode);

                 const targetStepId = specificHandler?.targetStepId || defaultHandler?.targetStepId;

                 if (targetStepId) {
                     nextStepIndex = definition.steps.findIndex(s => s.stepId === targetStepId);
                 } else if (definition.error_handler_step_id) { // Global handler
                     nextStepIndex = definition.steps.findIndex(s => s.stepId === definition.error_handler_step_id);
                 }
                 // If no handler found, nextStepIndex remains -1, leading to failure state below
            }

             // --- State Update & Next Action ---
             if (nextStepIndex === -1 || nextStepIndex >= definition.steps.length) {
                // Workflow ends (either success on last step or failure with no handler)
                instance.status = instance.status === 'running' ? (stepResult.success ? 'succeeded' : 'failed') : instance.status;
                instance.endTime = Date.now();
                this.logger.logTraceEvent({ workflowInstanceId: instanceId, eventType: 'WORKFLOW_END', payload: { status: instance.status, durationMs: instance.endTime - instance.startTime, error: instance.error } });
                await this._saveWorkflowInstance(instance);
                this.activeInstances.delete(instanceId);
                // TODO: Notify UI
            } else {
                 // Proceed to the next step
                 instance.currentStepIndex = nextStepIndex;
                 await this._saveWorkflowInstance(instance);
                 // TODO: Notify UI of step completion/progress
                 // Use setImmediate to avoid deep recursion / potential stack overflow
                 setImmediate(() => this._executeWorkflowStep(instanceId, nextStepIndex));
             }

        } catch (error) {
            // Catch errors *within* the orchestration step itself
            const instance = this.activeInstances.get(instanceId);
            if (instance) {
                instance.status = 'failed';
                instance.error = `Orchestrator error during step ${stepIndex}: ${error instanceof Error ? error.message : String(error)}`;
                instance.endTime = Date.now();
                 this.logger.logTraceEvent({ workflowInstanceId: instanceId, stepId: definition?.steps[stepIndex]?.stepId, agentInstanceId, eventType: 'STEP_ERROR', payload: { error: instance.error } });
                 this.logger.logTraceEvent({ workflowInstanceId: instanceId, eventType: 'WORKFLOW_END', payload: { status: 'failed', durationMs: instance.endTime - instance.startTime, error: instance.error } });
                await this._saveWorkflowInstance(instance);
                this.activeInstances.delete(instanceId);
                 // TODO: Notify UI
            }
            Logger.error(`Critical orchestrator error for workflow ${instanceId}`, error as Error);
        }
    }

     private async _saveWorkflowInstance(instance: WorkflowInstance): Promise<void> {
        try {
             const instanceDir = await ensureTaskDirectoryExists(this.context, `workflow_${instance.instanceId}`);
             await saveWorkflowInstance(instanceDir, instance); // Assumes this function handles async write
         } catch (error) {
             Logger.error(`Failed to save workflow instance state for ${instance.instanceId}`, error as Error);
             // Decide if this should halt the workflow or just log
         }
    }

     private async loadAndMarkFailedInstances(): Promise<void> {
         const tasksDir = path.join(this.context.globalStorageUri.fsPath, "tasks");
         if (!(await fileExistsAtPath(tasksDir))) return;

         const entries = await fs.readdir(tasksDir);
         for (const entry of entries) {
             if (entry.startsWith("workflow_")) {
                 const instanceDir = path.join(tasksDir, entry);
                 try {
                     const instance = await getSavedWorkflowInstance(instanceDir); // Assumes this function exists
                     if (instance && instance.status === 'running') {
                         instance.status = 'failed';
                         instance.error = 'Workflow interrupted by VS Code shutdown/restart.';
                         instance.endTime = Date.now(); // Mark end time as now
                         await this._saveWorkflowInstance(instance);
                         Logger.warn(`Marked interrupted workflow ${instance.instanceId} as failed.`);
                     }
                 } catch (error) {
                     Logger.error(`Error loading potentially interrupted workflow from ${instanceDir}`, error as Error);
                 }
             }
         }
     }

     // TODO: Add methods for pauseWorkflow, resumeWorkflow, cancelWorkflow
     // cancelWorkflow would set status to 'cancelled', potentially call agentInstance.cancelStep(),
     // save state, and remove from activeInstances.

}
```

---

**9. Richer Evaluation Adapter (`evals/cli/src/adapters/ExercismAdapter.ts` - Modified `verifyResult`)**

```typescript
import * as path from "path";
import * as fs from "fs/promises";
import { execa } from "execa";
import { BenchmarkAdapter, Task, MultiFacetVerificationResult } from "./types"; // Import updated type
import { parsePylintJson } from "../utils/parsers"; // Assume parser exists
// ... other imports

const EVALS_DIR = path.resolve(__dirname, "../../../");

export class ExercismAdapter implements BenchmarkAdapter {
    name = "exercism";
    // ... setup(), listTasks(), prepareTask() remain similar ...

    async verifyResult(task: Task, result: any): Promise<MultiFacetVerificationResult> {
        const baseResult: MultiFacetVerificationResult = {
            success: false, // Default to false
            metrics: {},
            qualityMetrics: undefined, // Initialize optional metrics
            securityMetrics: undefined,
            performanceMetrics: undefined,
        };
        const benchmarkSpecPath = path.join(task.workspacePath, 'benchmark.yaml');
        let benchmarkSpec: any = {}; // Load and parse benchmark.yaml here
        // ... try { benchmarkSpec = yaml.load(...) } catch { ... } ...

        // --- 1. Functional Correctness (Unit Tests) ---
        let testsPassed = 0;
        let testsFailed = 0;
        let testsTotal = 0;
        let testOutput = "";
        let functionalSuccess = false;

        if (task.verificationCommands && task.verificationCommands.length > 0) {
            for (const command of task.verificationCommands) {
                try {
                    // TODO: Run command within container if specified/needed
                    const [cmd, ...args] = command.split(" ");
                    const { stdout, stderr } = await execa(cmd, args, { cwd: task.workspacePath, reject: false });
                    testOutput += `COMMAND: ${command}\nSTDOUT:\n${stdout}\nSTDERR:\n${stderr}\n---\n`;
                    // Basic parsing - specific adapters need better parsing
                    testsPassed += (stdout.match(/PASS/gi) || []).length + (stderr.match(/PASS/gi) || []).length;
                    testsFailed += (stdout.match(/FAIL/gi) || []).length + (stderr.match(/FAIL/gi) || []).length;
                    // Assume success if no explicit 'FAIL' keywords found and command exits 0 (simple check)
                    if (stderr.toLowerCase().includes('fail')) functionalSuccess = false;
                    else functionalSuccess = true; // Simplified

                } catch (error: any) {
                    functionalSuccess = false;
                    testOutput += `COMMAND FAILED: ${command}\nERROR: ${error.message}\n---\n`;
                    testsFailed += 1; // Count command failure as a failed test
                }
            }
            testsTotal = testsPassed + testsFailed;
            baseResult.metrics.testsPassed = testsPassed;
            baseResult.metrics.testsFailed = testsFailed;
            baseResult.metrics.testsTotal = testsTotal;
            baseResult.metrics.functionalCorrectness = testsTotal > 0 ? testsPassed / testsTotal : 0;
        } else {
            functionalSuccess = true; // No tests defined means functionally correct by default? Or should it fail? Debatable.
             baseResult.metrics.functionalCorrectness = 1.0;
        }
        baseResult.success = functionalSuccess; // Overall success initially tied to tests passing

        // --- 2. Static Analysis (Optional - based on benchmark.yaml) ---
        if (functionalSuccess && benchmarkSpec.lint) {
            try {
                // TODO: Run benchmarkSpec.lint.command in container
                const { stdout } = await execa(benchmarkSpec.lint.command.split(' ')[0], benchmarkSpec.lint.command.split(' ').slice(1), { cwd: task.workspacePath, reject: false });
                // TODO: Call appropriate parser based on benchmarkSpec.lint.parser
                // Example:
                if (benchmarkSpec.lint.parser === 'pylint-json') {
                     const { errors, warnings } = parsePylintJson(JSON.parse(stdout));
                     baseResult.qualityMetrics = { ...(baseResult.qualityMetrics || {}), lintErrors: errors, lintWarnings: warnings };
                 }
                // ... handle other parsers ...
            } catch (lintError) {
                 Logger.warn(`Linting step failed for task ${task.id}: ${lintError.message}`);
                 if (!baseResult.qualityMetrics) baseResult.qualityMetrics = {};
                 baseResult.qualityMetrics.lintErrors = (baseResult.qualityMetrics.lintErrors || 0) + 1; // Indicate failure
            }
        }
         // ... Add similar blocks for complexity, SAST, performance based on benchmarkSpec ...

        return baseResult;
    }
}
```

---

**10. Evaluation DB Schema (`evals/cli/src/db/schema.ts` - Additions)**

```sql
-- Add new columns to the Tasks table (or create a new Metrics table)
ALTER TABLE tasks ADD COLUMN functionalCorrectness REAL;
ALTER TABLE tasks ADD COLUMN lintErrors INTEGER;
ALTER TABLE tasks ADD COLUMN lintWarnings INTEGER;
ALTER TABLE tasks ADD COLUMN complexityScore REAL;
ALTER TABLE tasks ADD COLUMN sastHigh INTEGER;
ALTER TABLE tasks ADD COLUMN sastMedium INTEGER;
ALTER TABLE tasks ADD COLUMN sastLow INTEGER;
ALTER TABLE tasks ADD COLUMN perfBeforeMs REAL;
ALTER TABLE tasks ADD COLUMN perfAfterMs REAL;

-- Potentially add a table for detailed file metrics if needed
-- CREATE TABLE IF NOT EXISTS fileMetrics ( ... );
```

---

**Conclusion:**

This set of illustrative code outlines the core structures needed for the basic `WorkflowOrchestrator`, persistent definition management via `BlueprintService`/`WorkflowService`, and the enhancements to the `evals/` framework required to capture richer metrics. The `AgentInstance` provides the reusable execution core, the Orchestrator manages the sequence and state, the services handle definition persistence, and the evaluation framework provides the data for analysis and future adaptation loops. While many implementation details and robust error handling are simplified here, this provides a concrete starting point for building the Cline Universal Code Orchestrator Workbench.