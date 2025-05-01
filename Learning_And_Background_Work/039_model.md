Okay, let's create the next advanced blog post, focusing on **Creating Custom MCP Servers** for Cline.

---

**Blog Series:** *Inside Cline: An Engineer's Guide to the Agentic Coding Assistant*

**Part 15: Extending Cline's Brain - Creating Custom MCP Servers**

*(Self-correction: This is Part 15, continuing the series)*

We've seen how Cline leverages the Model Context Protocol (MCP) to interact with external tools and resources (Part 8) and discussed the security implications (Part 14). For advanced users and contributors, the real power of MCP lies in the ability to create *custom* servers, tailoring Cline's capabilities precisely to your specific needs, APIs, or private codebases.

Imagine empowering Cline to interact with your company's internal build system, query a proprietary database, manage your specific cloud infrastructure via its CLI/SDK, or even interact with physical hardware in your lab. This is achievable by building your own MCP server.

This post provides a practical guide for developers looking to create custom MCP servers compatible with Cline. We'll cover the setup process, core concepts of the MCP SDK, implementing tools and resources, configuration within Cline, and best practices for building robust and secure servers.

**1. When to Build a Custom MCP Server?**

Before diving in, consider if a custom server is necessary:

*   **Need:** Does Cline need to interact with an API, database, system, or local script not covered by its built-in tools or existing community MCP servers?
*   **Complexity:** Is the required interaction complex enough that simply asking Cline to generate and run `curl` commands or scripts via `execute_command` is inefficient, unreliable, or insecure?
*   **Reusability:** Will this capability be needed frequently across multiple tasks or projects?
*   **Data Formatting:** Does the external system return data in a format that needs significant processing before it's useful to the LLM? An MCP server can pre-process this.
*   **Security/Abstraction:** Do you want to provide Cline access to a system *without* exposing raw API keys or complex command sequences directly in the chat history or prompts? An MCP server can encapsulate this logic and use environment variables for secrets.

If the answer to several of these is "yes," a custom MCP server is likely a good solution.

**2. Setting Up Your MCP Server Project**

The easiest way to start is using the official MCP creation tool:

1.  **Choose a Location:** Decide where to store your server code. While Cline defaults to suggesting `~/Documents/Cline/MCP/`, you can place it anywhere accessible on your system (e.g., within your main project, a dedicated `tools` directory).
2.  **Run the Scaffolding Tool:** Open your terminal, navigate *outside* the intended server directory, and run:
    ```bash
    npx @modelcontextprotocol/create-server your-server-name
    cd your-server-name
    ```
    Replace `your-server-name` with a descriptive name (e.g., `my-internal-api-mcp`, `jira-tools-mcp`).
3.  **Install Dependencies:**
    ```bash
    npm install
    # Add any specific SDKs or libraries your server needs
    npm install axios # Example for making HTTP requests
    ```
4.  **Project Structure:** You'll get a basic TypeScript project:
    ```
    your-server-name/
      ├── package.json
      ├── tsconfig.json
      └── src/
          └── index.ts  # <-- Your main server logic goes here
    ```
    Note the `"type": "module"` in `package.json` – use ES module syntax (`import`/`export`).

**3. Implementing Server Logic (`src/index.ts`)**

Open `src/index.ts`. The core involves using the `@modelcontextprotocol/sdk`:

```typescript
#!/usr/bin/env node // Makes the built script executable
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import {
  CallToolRequestSchema,
  ErrorCode,
  ListResourcesRequestSchema,
  ListResourceTemplatesRequestSchema,
  ListToolsRequestSchema,
  McpError,
  ReadResourceRequestSchema,
  // Import specific result schemas if needed
} from '@modelcontextprotocol/sdk/types.js';
// Import any other libraries you need (e.g., axios, aws-sdk)
// import axios from 'axios';

class YourServerNameServer {
  private server: Server;
  // Add any state your server needs (e.g., API clients)

  constructor() {
    this.server = new Server(
      { // ProtocolInfo: Identifies your server
        name: 'your-server-name', // Match the directory name/config key
        version: '0.1.0',
      },
      { // ServerInfo: Describes capabilities (can be empty initially)
        capabilities: {
          resources: {}, // We'll define these later if needed
          tools: {},     // We'll define these later
        },
      }
    );

    // --- Initialize API clients or other resources here ---
    // Example:
    // const MY_API_KEY = process.env.MY_CUSTOM_API_KEY;
    // if (!MY_API_KEY) throw new Error('MY_CUSTOM_API_KEY env var required');
    // this.apiClient = axios.create({ baseURL: '...', headers: {...}});

    // --- Register MCP Request Handlers ---
    this.registerToolHandlers();
    this.registerResourceHandlers(); // Optional

    // --- Standard Setup ---
    this.server.onerror = (error) => console.error('[MCP Error]', error);
    process.on('SIGINT', async () => { // Graceful shutdown
      await this.server.close();
      process.exit(0);
    });
  }

  private registerToolHandlers() {
    // 1. Handler for listing available tools
    this.server.setRequestHandler(ListToolsRequestSchema, async () => ({
      tools: [
        // --- Define your tools here ---
        {
          name: 'my_custom_action',
          description: 'Performs a specific custom action.',
          inputSchema: { // JSON Schema describing parameters
            type: 'object',
            properties: {
              param1: { type: 'string', description: 'The first parameter.' },
              requiredParam: { type: 'number', description: 'A mandatory number.' }
            },
            required: ['requiredParam'],
          },
        },
        // ... add more tool definitions
      ],
    }));

    // 2. Handler for executing a specific tool
    this.server.setRequestHandler(CallToolRequestSchema, async (request) => {
      const { name, arguments: args } = request.params;

      switch (name) {
        case 'my_custom_action':
          // --- Add logic for your tool here ---
          // a) Validate arguments against schema (important!)
          if (!args || typeof args.requiredParam !== 'number') {
            throw new McpError(ErrorCode.InvalidParams, 'Missing or invalid requiredParam.');
          }
          const param1 = args.param1 as string | undefined; // Cast or validate type

          try {
            // b) Perform the action (e.g., call an API)
            // const result = await this.apiClient.post('/action', { p1: param1, reqP: args.requiredParam });
            const result = `Action performed with requiredParam: ${args.requiredParam} and param1: ${param1 || 'not provided'}.`;

            // c) Return the result (must be in McpToolCallResponse format)
            return {
              content: [{ type: 'text', text: result }],
              // isError: false, // Optional, defaults to false
            };
          } catch (error) {
            // d) Handle errors gracefully
            console.error(`Error executing ${name}:`, error);
            return {
              content: [{ type: 'text', text: `Failed to execute ${name}: ${error instanceof Error ? error.message : String(error)}` }],
              isError: true,
            };
          }

        // ... add cases for other tools

        default:
          throw new McpError(ErrorCode.MethodNotFound, `Unknown tool: ${name}`);
      }
    });
  }

  private registerResourceHandlers() { // Optional: Implement if providing data resources
    // Handler for listing static resources (optional)
    this.server.setRequestHandler(ListResourcesRequestSchema, async () => ({
      resources: [
        // { uri: 'custom://data/item1', name: 'Item 1 Data', mimeType: 'application/json' },
      ],
    }));

    // Handler for listing resource templates (optional)
    this.server.setRequestHandler(ListResourceTemplatesRequestSchema, async () => ({
      resourceTemplates: [
        // { uriTemplate: 'custom://data/{itemId}', name: 'Get Item Data by ID', mimeType: 'application/json' },
      ],
    }));

    // Handler for reading resources (required if listing resources/templates)
    this.server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
      const uri = request.params.uri;
      // --- Add logic to match URI and fetch/generate resource content ---
      // Example:
      // if (uri === 'custom://data/item1') { ... }
      // const match = uri.match(/^custom:\/\/data\/([^/]+)$/);
      // if (match) { const itemId = match[1]; ... }

      throw new McpError(ErrorCode.ResourceUnavailable, `Resource not found: ${uri}`);
      // Return format:
      // return {
      //   contents: [{ uri: uri, mimeType: '...', text: '...' }],
      // };
    });
  }

  async run() {
    // Use Stdio transport for local servers
    const transport = new StdioServerTransport();
    await this.server.connect(transport);
    console.error(`${this.server.protocolInfo.name} v${this.server.protocolInfo.version} running on stdio`); // Use console.error for logs that shouldn't be sent to Cline
  }
}

// --- Instantiate and run the server ---
const server = new YourServerNameServer();
server.run().catch(console.error);
```

**Key Implementation Points:**

*   **SDK Imports:** Import necessary classes and schemas from `@modelcontextprotocol/sdk`.
*   **Server Instantiation:** Create a `new Server()` with unique `ProtocolInfo` (name, version) and optional `ServerInfo` (capabilities).
*   **Environment Variables:** Access secrets (API keys, tokens) passed from Cline's config using `process.env.YOUR_ENV_VAR_NAME`. **Never hardcode secrets.**
*   **Tool Definition (`ListToolsRequestSchema` handler):**
    *   Define each tool with a unique `name`, a clear `description` (this is what the LLM sees!), and an optional `inputSchema` (JSON Schema format) to specify parameters, types, and requirements. Good schemas help the LLM call tools correctly.
*   **Tool Execution (`CallToolRequestSchema` handler):**
    *   Use a `switch` statement on `request.params.name`.
    *   **Validate arguments:** Check if `request.params.arguments` exist and match your `inputSchema`. Throw `McpError(ErrorCode.InvalidParams, ...)` if invalid.
    *   Perform the tool's core logic (API calls, script execution, etc.).
    *   **Return results:** Package the output into the `McpToolCallResponse` format (`{ content: [{ type: 'text', text: '...' }] }`). You can also return images (`{ type: 'image', data: 'base64...', mimeType: '...' }`) or resource references. Set `isError: true` if the tool logic failed.
*   **Resource Handlers (Optional):** Implement `ListResources`, `ListResourceTemplates`, and `ReadResource` handlers if your server provides data via URIs.
*   **Error Handling:** Use `try...catch` within tool handlers. Return errors using the `{ content: [...], isError: true }` format or throw `McpError` for protocol-level issues. Log internal errors using `console.error`.
*   **Build Step:** The `package.json` includes a build script (`npm run build`) that uses `tsc` to compile your TypeScript to JavaScript (usually in `build/`) and makes the entry point executable (`chmod +x`).

**4. Configuring the Server in Cline**

Once your server is built (`npm run build`), you need to tell Cline how to run it:

1.  **Open Settings:** Use the `Configure MCP Servers` button in Cline's Settings > MCP Servers > Installed tab, or manually open `cline_mcp_settings.json` (location revealed by the button).
2.  **Add Configuration:** Add an entry to the `mcpServers` object. The key should be your unique server name.
    ```json
    {
      "mcpServers": {
        // ... other servers
        "your-server-name": {
          "command": "node", // Or python, deno, or direct executable path
          "args": ["/full/path/to/your-server-name/build/index.js"], // Absolute path to the built script
          "env": { // Optional: Environment variables for secrets
            "MY_CUSTOM_API_KEY": "value-from-user-or-docs",
            "ANOTHER_VAR": "some_value"
          },
          "timeout": 60, // Optional: Request timeout in seconds (default 60)
          "autoApprove": ["my_tool_to_always_allow"], // Optional: List of tool names to auto-approve
          "disabled": false // Optional: Set to true to temporarily disable
        }
      }
    }
    ```
    *   **`command`:** The executable to run (e.g., `node`, `python`, `/usr/local/bin/my-custom-server`).
    *   **`args`:** An array of arguments to pass to the command. Typically, this includes the absolute path to your server's main script.
    *   **`env`:** An object mapping environment variable names to their values (often API keys).
3.  **Save:** Saving the file automatically triggers `McpHub` to restart and attempt connection. Check the "Installed" tab in Cline's MCP settings for the connection status.

**5. Best Practices & Security**

*   **Input Validation:** Rigorously validate all arguments received in `CallToolRequestSchema` against your defined schema. Never trust that the LLM will provide perfectly formatted or safe input.
*   **Error Reporting:** Provide clear, informative error messages back to Cline when a tool fails. This helps the LLM (and the user) understand what went wrong.
*   **Secrets Management:** Use environment variables (`process.env`) passed via the `env` field in `cline_mcp_settings.json` for all sensitive data. Avoid hardcoding.
*   **Least Privilege:** If your server runs commands or accesses files, ensure it runs with the minimum necessary permissions. Be cautious with tools that execute arbitrary code or shell commands.
*   **Idempotency:** Design tools to be idempotent where possible (running them multiple times with the same input produces the same result) to handle retries gracefully.
*   **Descriptions:** Write clear, detailed descriptions for your tools and their parameters. This is the primary way the LLM understands how to use your server's capabilities.
*   **Keep it Focused:** Design servers around specific domains or APIs rather than creating monolithic servers with dozens of unrelated tools.

**Conclusion: Unleashing Cline's Full Potential**

Creating custom MCP servers transforms Cline from a powerful tool into an extensible platform uniquely suited to *your* development environment and workflows. While it requires some development effort, the `@modelcontextprotocol/sdk` and `create-server` tool provide a solid foundation. By carefully defining tools, handling arguments securely, managing state appropriately, and configuring the server correctly in Cline, you can significantly enhance your AI assistant's capabilities, automating complex, domain-specific tasks directly within VS Code. Remember to prioritize security and provide clear descriptions to enable effective collaboration between the LLM and your custom logic.

---

This concludes the planned 12-part series deep dive into Cline!