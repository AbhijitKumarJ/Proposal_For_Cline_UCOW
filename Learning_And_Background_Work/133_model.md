Okay, here is the second document for the Project Manager: **Source Code for Key Workbench Modules**.

As stated before, this is an *illustrative* implementation based on our design discussions. It provides the structure and core logic patterns but is **not** a complete, runnable, or fully tested codebase. It omits many details like comprehensive error handling, dependency injection specifics, complete UI implementations, detailed logging payloads, performance optimizations, and complex edge cases.

---

**Document 2: Source Code for Key Workbench Modules - Cline Universal Code Orchestrator Workbench**

**Purpose:** To illustrate the implementation patterns and core logic for the key new and refactored components required for the Cline Workbench vision. This provides a concrete code-level view to supplement the architectural design.

---

**1. Shared Workbench Types (`src/shared/workbench/types.ts`)**

```typescript
import { Anthropic } from "@anthropic-ai/sdk";
import { ToolUseName, AssistantMessageContent, ToolResponse } from "@core/assistant-message/index";
import { ApiConfiguration, ModelInfo } from "@shared/api";
import { ClineApiReqInfo, ClineMessage } from "@shared/ExtensionMessage"; // Assuming ClineApiReqInfo exists

// --- Agent Blueprint ---

export interface AgentBlueprint {
    id: string; // Unique identifier (e.g., "python-pytest-writer", "react-refactor-agent")
    version: number; // Simple integer version
    $schemaVersion: string; // For handling future schema changes (e.g., "1.0")
    description: string;
    baseSystemPrompt: string; // The core prompt defining the agent's role and general instructions
    allowedTools: ToolUseName[]; // Explicit list of tools this agent can use
    defaultModelConfig: Partial<ApiConfiguration>; // Default model/provider, can be overridden by workflow
    // outputSchema?: any; // Future: JSON Schema for structured output
    // contextStrategy?: string; // Future: e.g., 'standardTruncation', 'ragEnhanced'
}

// --- Workflow Definition ---

export interface WorkflowStep {
    stepId: string; // Unique identifier within the workflow
    blueprintId: string; // ID of the AgentBlueprint to use for this step
    initialPromptModifier?: string; // Text prepended to the blueprint's prompt for this specific step
    onSuccess?: { targetStepId: string }; // Step ID to jump to on success (defaults to next step)
    onFailure?: FailureTransition[]; // Array of failure transitions
    // parallel?: WorkflowStep[]; // Future: For fork-join parallelism
    // inputMapping?: Record<string, string>; // Future: Map scratchpad keys to prompt placeholders
    // outputMapping?: Record<string, string>; // Future: Map agent output to scratchpad keys
}

export interface FailureTransition {
    errorCode?: string; // Optional specific error code to match
    targetStepId: string; // Step ID to jump to
}

export interface WorkflowDefinition {
    id: string; // Unique identifier (e.g., "generate-and-test-python")
    version: number; // Simple integer version
    $schemaVersion: string;
    description: string;
    steps: WorkflowStep[];
    error_handler_step_id?: string; // Optional global step ID to jump to on unhandled failure
}

// --- Agent Execution Result ---

// Alias for clarity, content expected by the *next* user turn
export type UserContent = Array<Anthropic.ContentBlockParam>;

export interface AgentStepResult {
    success: boolean;
    outputContent: UserContent; // Primary output formatted as UserContent for the next step's input
    finalAssistantMessageBlocks: AssistantMessageContent[]; // Full assistant response blocks from this step
    finalToolResult?: ToolResponse; // Raw result from the tool, if one was executed
    error?: string; // Error message if success is false
    errorCode?: string; // Specific error code if success is false
    stepId: string; // ID of the step that produced this result
    startTimestamp: number;
    endTimestamp: number;
    apiMetrics?: ClineApiReqInfo; // Token/cost metrics for this step
}

// --- Workflow Instance State ---

export type WorkflowStatus = 'idle' | 'running' | 'succeeded' | 'failed' | 'paused' | 'cancelled';

export interface WorkflowInstance {
    instanceId: string;
    definitionId: string;
    definitionVersion: number; // Version of the definition used for this instance
    status: WorkflowStatus;
    currentStepIndex: number;
    startTime: number;
    endTime?: number;
    // Complete conversation history for this instance
    conversationHistory: Anthropic.MessageParam[];
    // Results from each completed step
    stepResults: AgentStepResult[];
    // Shared key-value store for this instance
    scratchpad: Record<string, string>;
    error?: string; // Final error if status is 'failed'
}

// --- Meta-Agent Suggestion ---

export type SuggestionChangeType = 'prompt_modification' | 'tool_allowlist_change' | 'blueprint_creation';
export type SuggestionStatus = 'pending_review' | 'applied' | 'ignored' | 'validating' | 'validation_passed' | 'validation_failed';
export type ValidationStatus = 'unvalidated' | 'running' | 'passed' | 'failed' | 'regressed';

export interface MetaAgentSuggestion {
    suggestionId: string;
    timestamp: number;
    sourceMetaAgentId: string; // ID of the agent that generated the suggestion
    targetBlueprintId: string;
    targetBlueprintVersion: number; // Version the suggestion applies TO
    changeType: SuggestionChangeType;
    // For prompt changes: standard diff patch format
    // For tool changes: { added: ToolUseName[], removed: ToolUseName[] }
    // For blueprint creation: The full AgentBlueprint JSON/YAML content
    proposedChange: string;
    rationale: string; // LLM explanation for the suggestion
    status: SuggestionStatus;
    validationStatus: ValidationStatus;
    validationRunIds?: string[]; // Link to evaluation runs used for validation
    confidenceScore?: number; // Optional score from Meta-Agent
    isAutoApplicable?: boolean; // If generated by a trusted template/rule
}

// --- Structured Logging --- (Simplified Example Payloads)
// In practice, these would be more detailed and potentially have dedicated interfaces

export type LogEventType =
    | 'WORKFLOW_START' | 'WORKFLOW_END'
    | 'STEP_START' | 'STEP_END' | 'STEP_ERROR'
    | 'TOOL_ATTEMPT' | 'TOOL_RESULT'
    | 'API_CALL_START' | 'API_CALL_END' // Replacing SUMMARY for more granularity
    | 'CONTEXT_UPDATE' | 'SCRATCHPAD_WRITE' | 'SCRATCHPAD_READ'
    | 'SUGGESTION_GENERATED' | 'SUGGESTION_APPLIED' | 'VALIDATION_START' | 'VALIDATION_END';

export interface BaseLogEvent {
    timestamp: number;
    workflowInstanceId: string;
    stepId?: string;
    agentInstanceId?: string; // Unique ID for the AgentInstance run within the step/workflow
    eventType: LogEventType;
    payload: Record<string, any>; // Consider defining specific payload interfaces per eventType
}

// --- END OF TYPES ---
```

---

**2. Agent Instance (`src/core/agent/AgentInstance.ts`)**

```typescript
import { Anthropic } from "@anthropic-ai/sdk";
import * as vscode from "vscode";
import { AgentBlueprint, AgentStepResult, UserContent } from "@shared/workbench/types";
import { ApiConfiguration, ApiHandler, buildApiHandler } from "@shared/api";
import { ToolUseName, parseAssistantMessage, AssistantMessageContent, ToolUse } from "@core/assistant-message/index";
import { formatResponse } from "@core/prompts/responses";
import { WorkflowLogger } from "@services/logging/WorkflowLogger"; // Assumed import
import { Controller } from "@core/controller"; // Needed for 'ask' callback via orchestrator
import { McpHub } from "@services/mcp/McpHub"; // Example dependency for tools
// ... other necessary imports for tool execution (fs, path, execa, services...)

export class AgentInstance {
    private config: AgentBlueprint;
    private apiHandler: ApiHandler;
    private logger: WorkflowLogger;
    private controllerRef: WeakRef<Controller>; // To access 'ask' via callback
    private mcpHub?: McpHub; // Example service dependency
    // ... other service dependencies (TerminalManager, BrowserSession, etc.)

    // Internal state for a single step execution
    private abortSignal: AbortController;
    private isStreaming: boolean = false;
    private assistantMessageContent: AssistantMessageContent[] = [];
    private currentStreamingContentIndex: number = 0;
    private userMessageContentForNextStep: UserContent = [];
    private toolResultForNextStep?: ToolResponse;
    private finalApiMetrics?: ClineApiReqInfo;
    private stepError?: string;
    private stepErrorCode?: string;

    constructor(
        blueprint: AgentBlueprint,
        dependencies: { // Pass needed services via DI
            logger: WorkflowLogger;
            controller: Controller;
            mcpHub?: McpHub;
            // ... other services
        }
    ) {
        this.config = blueprint;
        // Combine global config with blueprint defaults
        const mergedApiConfig: ApiConfiguration = {
            ...dependencies.controller.globalApiConfiguration, // Assuming Controller exposes this
            ...blueprint.defaultModelConfig,
            taskId: "agent-" + Date.now() // Provide a temporary ID for API handlers if needed
        };
        this.apiHandler = buildApiHandler(mergedApiConfig);
        this.logger = dependencies.logger;
        this.controllerRef = new WeakRef(dependencies.controller);
        this.mcpHub = dependencies.mcpHub;
        // ... store other dependencies
        this.abortSignal = new AbortController();
    }

    public cancelStep(): void {
        this.abortSignal.abort();
        // Add cleanup logic if needed (e.g., stopping ongoing tool execution)
    }

    /**
     * Executes a single step of the agent's logic.
     */
    async executeStep(
        workflowInstanceId: string,
        stepId: string,
        agentInstanceId: string, // Unique ID for this specific run/step
        conversationHistorySegment: Anthropic.MessageParam[], // History up to this point
        inputContent: UserContent // Primary input from previous step or initial task
    ): Promise<AgentStepResult> {
        const startTime = Date.now();
        this.logger.logTraceEvent({ workflowInstanceId, stepId, agentInstanceId, eventType: 'STEP_START', payload: { blueprintId: this.config.id } });

        // Reset step state
        this.isStreaming = false;
        this.assistantMessageContent = [];
        this.currentStreamingContentIndex = 0;
        this.userMessageContentForNextStep = [];
        this.toolResultForNextStep = undefined;
        this.finalApiMetrics = undefined;
        this.stepError = undefined;
        this.stepErrorCode = undefined;
        this.abortSignal = new AbortController(); // Fresh signal for each step

        try {
            // 1. Construct Prompt & History for this step
            const systemPrompt = this.config.baseSystemPrompt; // Apply templating later if needed
            const currentTurnHistory: Anthropic.MessageParam[] = [
                ...conversationHistorySegment,
                { role: 'user', content: inputContent }
            ];

            // 2. Make API Call (using logic adapted from Task.attemptApiRequest)
            const stream = await this.streamApiRequest(systemPrompt, currentTurnHistory, workflowInstanceId, stepId, agentInstanceId);

            // 3. Process Stream & Handle Tool Calls
            await this.processApiResponseStream(stream, workflowInstanceId, stepId, agentInstanceId);

            // 4. Determine Step Outcome
            const success = !this.stepError;
            const endTime = Date.now();
            const durationMs = endTime - startTime;

            const result: AgentStepResult = {
                success,
                outputContent: this.userMessageContentForNextStep, // This is the crucial output for the *next* step
                finalAssistantMessageBlocks: this.assistantMessageContent, // The full assistant response for history
                finalToolResult: this.toolResultForNextStep,
                error: this.stepError,
                errorCode: this.stepErrorCode,
                stepId,
                startTimestamp: startTime,
                endTimestamp: endTime,
                apiMetrics: this.finalApiMetrics,
            };

            this.logger.logTraceEvent({
                workflowInstanceId, stepId, agentInstanceId, eventType: 'STEP_END',
                payload: { durationMs, success, outputSummary: this.summarizeOutput(result.outputContent), error: this.stepError, errorCode: this.stepErrorCode }
            });

            return result;

        } catch (error) {
            // Catch errors during API call setup or top-level stream processing
            const endTime = Date.now();
            const durationMs = endTime - startTime;
            this.stepError = error instanceof Error ? error.message : String(error);
            this.logger.logTraceEvent({
                workflowInstanceId, stepId, agentInstanceId, eventType: 'STEP_ERROR',
                payload: { durationMs, error: this.stepError }
            });
            // Return failure result
            return {
                success: false,
                outputContent: [],
                finalAssistantMessageBlocks: [],
                error: this.stepError,
                stepId,
                startTimestamp: startTime,
                endTimestamp: endTime,
            };
        }
    }

    private async streamApiRequest(
        systemPrompt: string,
        history: Anthropic.MessageParam[],
        workflowInstanceId: string, stepId: string, agentInstanceId: string // For logging
    ): Promise<ApiStream> {
        this.logger.logTraceEvent({ workflowInstanceId, stepId, agentInstanceId, eventType: 'API_CALL_START', payload: { model: this.apiHandler.getModel().id } });
        const startTime = Date.now();
        try {
            // Simplified: Assumes attemptApiRequest logic is now within the handler or called here
            // In a real implementation, add retry logic, context checks etc. adapted from Task.ts
            const stream = this.apiHandler.createMessage(systemPrompt, history);
            // Note: We don't await the full stream here, just return the generator
            // We'll track metrics *after* processing the stream.
            return stream;
        } catch (error) {
            const endTime = Date.now();
            this.logger.logTraceEvent({
                workflowInstanceId, stepId, agentInstanceId, eventType: 'API_CALL_END',
                payload: { durationMs: endTime - startTime, success: false, error: error instanceof Error ? error.message : String(error) }
            });
            throw error; // Re-throw for executeStep to catch
        }
    }

    private async processApiResponseStream(
        stream: ApiStream,
        workflowInstanceId: string, stepId: string, agentInstanceId: string
    ): Promise<void> {
        let assistantMessage = "";
        let inputTokens = 0, outputTokens = 0, cacheWrites = 0, cacheReads = 0, cost = 0;
        this.isStreaming = true;
        const streamStartTime = Date.now();

        try {
            for await (const chunk of stream) {
                if (this.abortSignal.aborted) {
                    throw new Error("Agent step execution cancelled.");
                }

                switch (chunk.type) {
                    case "usage":
                        inputTokens += chunk.inputTokens;
                        outputTokens += chunk.outputTokens;
                        cacheWrites += chunk.cacheWriteTokens || 0;
                        cacheReads += chunk.cacheReadTokens || 0;
                        cost += chunk.totalCost || 0; // Needs careful aggregation if yielded incrementally
                        break;
                    case "reasoning":
                        // Could log this or store it if needed
                        break;
                    case "text":
                        assistantMessage += chunk.text;
                        this.assistantMessageContent = parseAssistantMessage(assistantMessage);
                        // Trigger potential tool execution based on updated content
                        await this.presentAssistantMessage(workflowInstanceId, stepId, agentInstanceId);
                        break;
                }

                // If a tool was executed and potentially rejected, presentAssistantMessage sets stepError
                if (this.stepError) {
                    break; // Stop processing stream if a tool failed or was rejected
                }
            }
        } catch (error) {
            // Catch errors during streaming itself
            this.stepError = `Stream processing error: ${error instanceof Error ? error.message : String(error)}`;
            this.logger.logTraceEvent({
                workflowInstanceId, stepId, agentInstanceId, eventType: 'STEP_ERROR',
                payload: { error: this.stepError, duringStream: true }
            });
        } finally {
            this.isStreaming = false;
            // Final processing of any remaining buffered content
            this.assistantMessageContent = parseAssistantMessage(assistantMessage);
            // Ensure the last state is presented/processed if no error occurred mid-stream
            if (!this.stepError) {
                 await this.presentAssistantMessage(workflowInstanceId, stepId, agentInstanceId, true); // Force final presentation
            }

            // Log final API metrics
            const streamEndTime = Date.now();
            this.finalApiMetrics = { tokensIn: inputTokens, tokensOut: outputTokens, cacheWrites, cacheReads, cost };
            this.logger.logTraceEvent({
                workflowInstanceId, stepId, agentInstanceId, eventType: 'API_CALL_END',
                payload: {
                    durationMs: streamEndTime - streamStartTime,
                    success: !this.stepError,
                    error: this.stepError, // Include error if stream aborted due to tool failure
                    ...this.finalApiMetrics
                }
            });
        }
    }

    /**
     * Processes parsed AssistantMessageContent blocks, handling text and tool calls.
     * Adapted from Task.presentAssistantMessage.
     */
    private async presentAssistantMessage(
        workflowInstanceId: string, stepId: string, agentInstanceId: string, isFinalPass: boolean = false
    ): Promise<void> {
         // Simplified version - focuses on logic flow
        let executedTool: ToolUseName | null = null;

        for (let i = this.currentStreamingContentIndex; i < this.assistantMessageContent.length; i++) {
             if (this.abortSignal.aborted) return;
             if (this.stepError) break; // Stop if a previous tool in this turn failed/rejected

             const block = this.assistantMessageContent[i];
             const isLastBlockInCurrentParse = i === this.assistantMessageContent.length - 1;
             const isPartial = block.partial && !isFinalPass; // Only truly partial if not the final forced pass

             if (block.type === "text") {
                 // Buffer text for final output, or handle UI updates if needed immediately
                 // In this simplified version, we mostly care about tool calls
                 if (!isPartial) {
                     this.currentStreamingContentIndex = i + 1; // Mark as processed
                 }
             } else if (block.type === "tool_use") {
                if (executedTool) {
                    // Constraint: Only one tool per turn
                    this.stepError = formatResponse.toolAlreadyUsed(block.name);
                    this.stepErrorCode = "MULTIPLE_TOOLS_USED";
                    this.logger.logTraceEvent({ /* ... Log multiple tools error ... */ });
                    break;
                }

                // Check if tool is allowed by the blueprint
                if (!this.config.allowedTools.includes(block.name)) {
                    this.stepError = `Tool "${block.name}" is not allowed for agent "${this.config.id}".`;
                    this.stepErrorCode = "TOOL_NOT_ALLOWED";
                    this.logger.logTraceEvent({ /* ... Log tool not allowed error ... */ });
                    break;
                }

                // If the block is partial, wait for it to complete unless it's the final pass
                if (isPartial) continue;

                executedTool = block.name;
                this.currentStreamingContentIndex = i + 1; // Mark as processed

                const toolStartTime = Date.now();
                this.logger.logTraceEvent({ workflowInstanceId, stepId, agentInstanceId, eventType: 'TOOL_ATTEMPT', payload: { toolName: block.name, parameters: block.params } });

                let toolSuccess = false;
                let toolResult: ToolResponse | undefined = undefined;
                let toolError: string | undefined = undefined;
                let toolErrorCode: string | undefined = undefined;

                try {
                    // 1. Parameter Validation (Example for execute_command) - NEEDS ZOD or similar
                    if (block.name === 'execute_command') {
                        if (!block.params.command) {
                             toolError = formatResponse.missingToolParameterError("command");
                             toolErrorCode = "MISSING_PARAM_COMMAND";
                             throw new Error(toolError);
                         }
                         if (block.params.requires_approval === undefined || typeof block.params.requires_approval !== 'string') {
                             toolError = formatResponse.missingToolParameterError("requires_approval");
                             toolErrorCode = "MISSING_PARAM_REQ_APPROVAL";
                             throw new Error(toolError);
                         }
                         // ... more validation ...
                    }
                    // ... add validation for other tools ...

                    // 2. Approval Request (Simplified - needs callback)
                    const controller = this.controllerRef.deref();
                    if (!controller) throw new Error("Controller context lost");

                    const needsApproval = this.determineIfApprovalNeeded(block); // Needs implementation based on tool + params + settings
                    let approved = !needsApproval;
                    if (needsApproval) {
                        const { response: askResponse } = await controller.askForToolApproval(block); // Needs implementation in Controller
                        if (askResponse === 'yesButtonClicked') {
                            approved = true;
                        } else {
                            toolError = formatResponse.toolDenied();
                            toolErrorCode = "USER_REJECTED";
                            this.stepError = toolError; // Set step error if user rejects
                            this.stepErrorCode = toolErrorCode;
                        }
                    }

                    // 3. Execute Tool if Approved
                    if (approved) {
                        const [execSuccess, resultData] = await this.executeTool(block); // Implement executeTool switch
                        toolSuccess = execSuccess;
                        toolResult = resultData;
                        if (!toolSuccess && typeof toolResult === 'string') {
                            toolError = toolResult; // Assume string result on failure is error message
                            toolErrorCode = "TOOL_EXECUTION_FAILED"; // Generic code
                            this.stepError = toolError; // Propagate error to step level
                            this.stepErrorCode = toolErrorCode;
                        }
                    }

                } catch (error) {
                    // Catch validation or execution errors
                    toolSuccess = false;
                    toolError = error instanceof Error ? error.message : String(error);
                    toolErrorCode = toolErrorCode || "TOOL_INTERNAL_ERROR"; // Use specific code if set, else generic
                    this.stepError = toolError;
                    this.stepErrorCode = toolErrorCode;
                } finally {
                    // 4. Log Tool Result
                    const toolEndTime = Date.now();
                    this.logger.logTraceEvent({
                        workflowInstanceId, stepId, agentInstanceId, eventType: 'TOOL_RESULT',
                        payload: {
                            toolName: block.name, durationMs: toolEndTime - toolStartTime,
                            success: toolSuccess && !this.stepError, // Successful only if executed AND step didn't fail
                            resultSummary: this.summarizeOutput(toolResult), error: toolError, errorCode: toolErrorCode
                        }
                    });
                    // 5. Store result for next step's input
                    this.userMessageContentForNextStep = typeof toolResult === 'string' ? [{ type: 'text', text: toolResult }] : (toolResult || []);
                    this.toolResultForNextStep = toolResult;

                     // If tool failed or was rejected, stop processing further blocks in this turn
                    if (!toolSuccess || this.stepError) {
                         break;
                    }
                }
             }
        }
    }

    private determineIfApprovalNeeded(block: ToolUse): boolean {
        // Simplified: In a real system, this would check AutoApprovalSettings
        // against the block.name and potentially block.params (like requires_approval).
        // For now, assume most tools require approval unless explicitly allowed.
        const SAFE_READ_TOOLS: ToolUseName[] = ['read_scratchpad', 'list_files']; // Example
        return !SAFE_READ_TOOLS.includes(block.name);
    }

    private async executeTool(block: ToolUse): Promise<[boolean, ToolResponse | undefined]> {
         // --- Simplified Tool Execution Logic ---
         // This needs to map block.name to actual service/integration calls
         // like in the original Task.presentAssistantMessage, but returning results.
         console.warn(`Executing tool: ${block.name} with params:`, block.params);
         await new Promise(resolve => setTimeout(resolve, 50)); // Simulate async work

         switch (block.name) {
             case 'read_file':
                 // Call filesystem service...
                 // return [true, `Content of ${block.params.path}`];
                 return [true, `[Simulated content for ${block.params.path}]`];
            case 'write_scratchpad':
                 // Interact with orchestrator's scratchpad...
                 return [true, `[Wrote to scratchpad key: ${block.params.key}]`];
             case 'read_scratchpad':
                 // Interact with orchestrator's scratchpad...
                 return [true, `[Read from scratchpad key: ${block.params.key} - Value: simulated_value]`];
             // ... other tool cases ...
             default:
                 return [false, `Tool "${block.name}" execution not implemented in this example.`];
         }
         // --- End Simplified Logic ---
    }

     private summarizeOutput(output: any): string {
         // Helper to create short summaries for logging
         if (typeof output === 'string') {
             return output.slice(0, 100) + (output.length > 100 ? '...' : '');
         }
         if (Array.isArray(output)) {
             const text = output.find(b => b.type === 'text')?.text || '';
             return text.slice(0,100) + (text.length > 100 ? '...' : '');
         }
         return JSON.stringify(output).slice(0, 100) + '...';
     }
}
```

---

**Conclusion:**

This detailed implementation sketch provides the structure and core logic for the essential new components (`AgentInstance`, `AgentBlueprint`, `WorkflowOrchestrator`, `WorkflowLogger`, etc.) and the necessary modifications to existing systems (`shared types`, `protobufs`, UI components). While significant implementation effort, testing, and refinement remain, this blueprint aligns with the design decisions made in our simulated research sessions, paving the way for building the Cline Universal Code Orchestrator Workbench.