# Functional Agent Framework (FAF)

[![CI](https://github.com/xynehq/faf/workflows/CI/badge.svg)](https://github.com/xynehq/faf/actions)
[![Documentation](https://img.shields.io/badge/docs-mkdocs-blue)](https://xynehq.github.io/faf/)
[![npm version](https://img.shields.io/npm/v/@xynehq/faf.svg)](https://www.npmjs.com/package/@xynehq/faf)
[![npm downloads](https://img.shields.io/npm/dm/@xynehq/faf.svg)](https://www.npmjs.com/package/@xynehq/faf)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

![Functional Agent Framework](/docs/cover.png?raw=true "Functional Agent Framework")

A purely functional agent framework built on immutable state, type safety, and composable policies. FAF enables building production-ready AI agent systems with built-in security, observability, and error handling.

📚 **[Read the Documentation](https://xynehq.github.io/faf/)**

## 🎯 Core Philosophy

- **Immutability**: All core data structures are deeply `readonly`
- **Pure Functions**: Core logic expressed as pure, predictable functions
- **Effects at the Edge**: Side effects isolated in Provider modules
- **Composition over Configuration**: Build complex behavior by composing simple functions
- **Type-Safe by Design**: Leverages TypeScript's advanced features for compile-time safety

## 🚀 Quick Start

### Installation

```bash
# Install from npm
npm install @xynehq/faf

# Or using yarn
yarn add @xynehq/faf

# Or using pnpm
pnpm add @xynehq/faf
```

### Development Setup

```bash
# Clone the repository
git clone https://github.com/xynehq/faf.git
cd faf

# Install dependencies
npm install

# Build the project
npm run build

# Run tests
npm test
```

## 📁 Project Structure

```
src/
├── core/           # Core framework types and engine
│   ├── engine.ts   # Main execution engine
│   ├── errors.ts   # Error handling and types
│   ├── tool-results.ts # Tool execution results
│   ├── tracing.ts  # Event tracing system
│   └── types.ts    # Core type definitions
├── memory/         # Memory providers for conversation persistence
│   ├── factory.ts  # Memory provider factory
│   ├── types.ts    # Memory system types
│   └── providers/
│       ├── in-memory.ts  # In-memory provider
│       ├── postgres.ts   # PostgreSQL provider
│       └── redis.ts      # Redis provider
├── providers/      # External integrations
│   ├── mcp.ts      # Model Context Protocol integration
│   └── model.ts    # LLM provider integrations
├── policies/       # Validation and security policies
│   ├── handoff.ts  # Agent handoff policies
│   └── validation.ts # Input/output validation
├── server/         # HTTP server implementation
│   ├── index.ts    # Server entry point
│   ├── server.ts   # Express server setup
│   └── types.ts    # Server-specific types
├── __tests__/      # Test suite
│   ├── engine.test.ts     # Engine tests
│   └── validation.test.ts # Validation tests
└── index.ts        # Main framework exports
examples/
├── rag-demo/       # Vertex AI RAG integration demo
│   ├── index.ts    # Demo entry point
│   ├── rag-agent.ts # RAG agent implementation
│   └── rag-tool.ts  # RAG tool implementation
└── server-demo/    # Development server demo
    └── index.ts    # Server demo entry point
docs/               # Documentation
├── getting-started.md
├── core-concepts.md
├── api-reference.md
├── tools.md
├── memory-system.md
├── model-providers.md
├── server-api.md
├── examples.md
├── deployment.md
└── troubleshooting.md
```

## 🏗️ Key Components

### Core Types

```typescript
import { z } from 'zod';
import { Agent, Tool, RunState, run } from '@xynehq/faf';

// Define your context type
type MyContext = {
  userId: string;
  permissions: string[];
};

// Create a tool
const calculatorTool: Tool<{ expression: string }, MyContext> = {
  schema: {
    name: "calculate",
    description: "Perform mathematical calculations",
    parameters: z.object({
      expression: z.string().describe("Math expression to evaluate")
    }),
  },
  execute: async (args) => {
    const result = eval(args.expression); // Don't do this in production!
    return `${args.expression} = ${result}`;
  },
};

// Define an agent
const mathAgent: Agent<MyContext, string> = {
  name: 'MathTutor',
  instructions: () => 'You are a helpful math tutor',
  tools: [calculatorTool],
};
```

### Running the Framework

```typescript
import { run, makeLiteLLMProvider } from '@xynehq/faf';

const modelProvider = makeLiteLLMProvider('http://localhost:4000');
const agentRegistry = new Map([['MathTutor', mathAgent]]);

const config = {
  agentRegistry,
  modelProvider,
  maxTurns: 10,
  onEvent: (event) => console.log(event), // Real-time tracing
};

const initialState = {
  runId: generateRunId(),
  traceId: generateTraceId(),
  messages: [{ role: 'user', content: 'What is 2 + 2?' }],
  currentAgentName: 'MathTutor',
  context: { userId: 'user123', permissions: ['user'] },
  turnCount: 0,
};

const result = await run(initialState, config);
```

## 🛡️ Security & Validation

### Composable Validation Policies

```typescript
import { createPathValidator, createPermissionValidator, composeValidations } from '@xynehq/faf';

// Create individual validators
const pathValidator = createPathValidator(['/shared', '/public']);
const permissionValidator = createPermissionValidator('admin', ctx => ctx);

// Compose them
const combinedValidator = composeValidations(pathValidator, permissionValidator);

// Apply to tools
const secureFileTool = withValidation(baseFileTool, combinedValidator);
```

### Guardrails

```typescript
import { createContentFilter, createRateLimiter } from '@xynehq/faf';

const config = {
  // ... other config
  initialInputGuardrails: [
    createContentFilter(),
    createRateLimiter(10, 60000, input => 'global')
  ],
  finalOutputGuardrails: [
    createContentFilter()
  ],
};
```

## 🔗 Agent Handoffs

```typescript
import { handoffTool } from '@xynehq/faf';

const triageAgent: Agent<Context, { agentName: string }> = {
  name: 'TriageAgent',
  instructions: () => 'Route requests to specialized agents',
  tools: [handoffTool],
  handoffs: ['MathTutor', 'FileManager'], // Allowed handoff targets
  outputCodec: z.object({
    agentName: z.enum(['MathTutor', 'FileManager'])
  }),
};
```

## 📊 Observability

### Real-time Tracing

```typescript
import { ConsoleTraceCollector, FileTraceCollector } from '@xynehq/faf';

// Console logging
const consoleTracer = new ConsoleTraceCollector();

// File logging
const fileTracer = new FileTraceCollector('./traces.log');

// Composite tracing
const tracer = createCompositeTraceCollector(consoleTracer, fileTracer);

const config = {
  // ... other config
  onEvent: tracer.collect.bind(tracer),
};
```

### Error Handling

```typescript
import { FAFErrorHandler } from '@xynehq/faf';

if (result.outcome.status === 'error') {
  const formattedError = FAFErrorHandler.format(result.outcome.error);
  const isRetryable = FAFErrorHandler.isRetryable(result.outcome.error);
  const severity = FAFErrorHandler.getSeverity(result.outcome.error);
  
  console.error(`[${severity}] ${formattedError} (retryable: ${isRetryable})`);
}
```

## 🔌 Provider Integrations

### LiteLLM Provider

```typescript
import { makeLiteLLMProvider } from '@xynehq/faf';

// Connect to LiteLLM proxy for 100+ model support
const modelProvider = makeLiteLLMProvider(
  'http://localhost:4000', // LiteLLM proxy URL
  'your-api-key'           // Optional API key
);
```

### MCP (Model Context Protocol) Tools

```typescript
import { makeMCPClient, mcpToolToFAFTool } from '@xynehq/faf';

// Connect to MCP server
const mcpClient = await makeMCPClient('python', ['-m', 'mcp_server']);

// Get available tools
const mcpTools = await mcpClient.listTools();

// Convert to FAF tools with validation
const fafTools = mcpTools.map(tool => 
  mcpToolToFAFTool(mcpClient, tool, myValidationPolicy)
);
```

## 🚀 Development Server

FAF includes a built-in development server for testing agents locally via HTTP endpoints:

```typescript
import { runServer, makeLiteLLMProvider, createInMemoryProvider } from '@xynehq/faf';

const myAgent = {
  name: 'MyAgent',
  instructions: 'You are a helpful assistant',
  tools: [calculatorTool, greetingTool]
};

const modelProvider = makeLiteLLMProvider('http://localhost:4000');
const memoryProvider = createInMemoryProvider();

// Start server on port 3000
const server = await runServer(
  [myAgent], 
  { modelProvider },
  { port: 3000, defaultMemoryProvider: memoryProvider }
);
```

Server provides RESTful endpoints:
- `GET /health` - Health check
- `GET /agents` - List available agents  
- `POST /chat` - General chat endpoint
- `POST /agents/{name}/chat` - Agent-specific endpoint

## 📚 Documentation

Comprehensive documentation is available in the [`/docs`](./docs) folder:

- **[Getting Started](./docs/getting-started.md)** - Installation, basic concepts, and first agent
- **[Core Concepts](./docs/core-concepts.md)** - FAF's functional architecture and principles  
- **[API Reference](./docs/api-reference.md)** - Complete TypeScript API documentation
- **[ADK Layer](./docs/adk-layer.md)** - Agent Development Kit for simplified agent creation
- **[A2A Protocol](./docs/a2a-protocol.md)** - Agent-to-Agent communication and task management
- **[Tools](./docs/tools.md)** - Building robust tools with validation and error handling
- **[Memory System](./docs/memory-system.md)** - Conversation persistence (in-memory, Redis, PostgreSQL)
- **[Model Providers](./docs/model-providers.md)** - LLM integration and configuration
- **[Server & API](./docs/server-api.md)** - HTTP server setup and REST API
- **[Visualization](./docs/visualization.md)** - Generate Graphviz diagrams of agents and tools
- **[Examples](./docs/examples.md)** - Tutorials and integration patterns
- **[Deployment](./docs/deployment.md)** - Production deployment guide
- **[Troubleshooting](./docs/troubleshooting.md)** - Common issues and debugging

### 📖 Documentation Website

Browse the full documentation online at **[https://xynehq.github.io/faf/](https://xynehq.github.io/faf/)**

The documentation site features:
- 🔍 Full-text search
- 🌓 Dark/light mode toggle  
- 📱 Mobile-friendly responsive design
- 🔗 Deep linking to sections
- 📋 Code block copy buttons

#### Running Documentation Locally

```bash
# Install documentation dependencies
pip install -r requirements.txt

# Run local documentation server
mkdocs serve
# Visit http://127.0.0.1:8000

# Or use the convenience script
./docs/serve.sh
```

## 🎮 Example Applications

Explore the example applications to see the framework in action:

### Development Server Demo

```bash
cd examples/server-demo
npm install
npm run dev
```

The server demo showcases:
- ✅ Multiple agent types with different capabilities
- ✅ RESTful API with type-safe validation
- ✅ Tool integration (calculator, greeting)
- ✅ Real-time tracing and error handling
- ✅ CORS support and graceful shutdown

### Vertex AI RAG Demo

```bash
cd examples/rag-demo
npm install
npm run dev
```

The RAG demo showcases:
- ✅ Real Vertex AI RAG integration with Google GenAI SDK
- ✅ Permission-based access control
- ✅ Real-time streaming responses with source attribution
- ✅ Performance metrics and comprehensive error handling
- ✅ FAF framework orchestration with type-safe tools
- ✅ Multi-turn conversations with observability

## 🧪 Testing

```bash
npm test        # Run tests
npm run lint    # Lint code
npm run typecheck # Type checking
```

## 🏛️ Architecture Principles

### Immutable State Machine
- All state transformations create new state objects
- No mutation of existing data structures
- Predictable, testable state transitions

### Type Safety
- Runtime validation with Zod schemas
- Compile-time safety with TypeScript
- Branded types prevent ID mixing

### Pure Functions
- Core logic is side-effect free
- Easy to test and reason about
- Deterministic behavior

### Effect Isolation
- Side effects only in Provider modules
- Clear boundaries between pure and impure code
- Easier mocking and testing

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Add tests for new functionality
4. Run the test suite
5. Submit a pull request

---

**FAF** - Building the future of functional AI agent systems 🚀