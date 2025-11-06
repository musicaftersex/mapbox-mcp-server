# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Essential Commands

### Development Workflow
```bash
npm install              # Install dependencies
npm run build           # Build: compile TypeScript (ESM + CommonJS), generate version info
npm test                # Run all tests with Vitest
npm run lint            # Check linting (ESLint)
npm run format:fix      # Auto-format code (Prettier)
```

### Testing
```bash
npm test                                    # Run all tests
npx vitest run path/to/test.ts             # Run single test file
npx vitest run -t "test name pattern"      # Run tests matching pattern
npx vitest --coverage                       # Generate coverage report
```

### Debugging with MCP Inspector
```bash
npm run inspect:dev    # Run server with Inspector using tsx (no build needed)
npm run inspect:build  # Run built server with Inspector
```

### OpenTelemetry Tracing
```bash
npm run tracing:jaeger:start   # Start Jaeger container (Docker required)
npm run tracing:jaeger:stop    # Stop Jaeger container
# View traces at http://localhost:16686 after starting server
```

### Scaffolding New Tools
```bash
npx plop create-tool   # Generate new tool with template (prompts for names)
```

## Architecture Overview

This is an **MCP (Model Context Protocol) server** that exposes Mapbox APIs as tools for AI agents. The architecture emphasizes dependency injection, explicit HTTP policy pipelines, and comprehensive tracing.

### Key Architectural Patterns

#### 1. Tool Implementation Hierarchy
```
BaseTool (abstract)
├── MapboxApiBasedTool (extends BaseTool, adds JWT validation & Mapbox patterns)
│   └── [MapboxTools] - SearchAndGeocodeTool, DirectionsTool, etc.
└── [Non-API Tools] - VersionTool
```

**Tool Structure:**
```
src/tools/[tool-name]-tool/
├── [Tool]Tool.ts              # Main class extending MapboxApiBasedTool
├── [Tool]Tool.input.schema.ts # Zod input validation
└── [Tool]Tool.output.schema.ts # Zod output schema
```

**Implementation Pattern:**
- Extend `MapboxApiBasedTool` for Mapbox API tools
- Constructor receives `{ httpRequest: HttpRequest }` dependency
- Override `execute()` method to implement tool logic
- Return `CallToolResult` with both text (human-readable) and structuredContent (machine-parseable)

#### 2. HTTP Policy Pipeline System

**Critical Pattern:** Do NOT patch `global.fetch`. Use the policy pipeline system.

```typescript
// src/utils/httpPipeline.ts exports a configured pipeline
const pipeline = new HttpPipeline();
pipeline.usePolicy(UserAgentPolicy);   // Adds User-Agent header
pipeline.usePolicy(RetryPolicy);        // Exponential backoff (3 retries)
pipeline.usePolicy(TracingPolicy);      // OpenTelemetry span capture
export const httpRequest = pipeline.execute.bind(pipeline);
```

**Pipeline Policies:**
- **UserAgentPolicy**: Adds version info to User-Agent header
- **RetryPolicy**: Retries failed requests (3 attempts, 200ms-2000ms delays)
- **TracingPolicy**: Captures HTTP response headers in OTEL spans

Tools receive the bound `httpRequest` function via dependency injection and use it instead of direct `fetch()`.

#### 3. Tool Registry Pattern

All tools are instantiated in `src/tools/toolRegistry.ts`:
```typescript
export const ALL_TOOLS = [
  new VersionTool(),
  new SearchAndGeocodeTool({ httpRequest }),
  new DirectionsTool({ httpRequest }),
  // ... 9 tools total
] as const;
```

Tool filtering happens via CLI args (`--enable-tools`, `--disable-tools`) parsed in `src/config/toolConfig.ts`.

#### 4. Input/Output Validation with Zod

- All tools define Zod schemas for input validation
- Schemas use `.describe()` for MCP protocol documentation
- Complex transforms handle multiple input formats (e.g., coordinates as string or object)
- Output validation is optional with graceful fallback

#### 5. OpenTelemetry Tracing

Tracing is built-in and initialized in `src/index.ts`:
- Enabled automatically unless `NODE_ENV=test`
- Configure via environment variables (`OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_SERVICE_NAME`, etc.)
- Creates spans for tool execution and HTTP requests
- Captures CloudFront correlation IDs and response metadata

### Project Structure

```
src/
├── index.ts                    # Entry point: loads .env, init tracing, registers tools
├── config/toolConfig.ts        # CLI argument parser for tool filtering
├── tools/
│   ├── BaseTool.ts            # Abstract base for all tools
│   ├── MapboxApiBasedTool.ts   # Base for Mapbox API tools
│   ├── toolRegistry.ts         # Central registry (ALL_TOOLS array)
│   └── [tool-name]-tool/       # Individual tool directories
└── utils/
    ├── httpPipeline.ts         # HTTP policy pipeline implementation
    ├── tracing.ts              # OpenTelemetry setup
    └── versionUtils.ts         # Build version metadata

test/                           # Mirrors src/ structure
└── utils/httpPipelineUtils.ts  # Test helpers for mocking HTTP
```

## Coding Standards

### TypeScript Requirements
- Strict TypeScript only (no `any` without justification)
- All code in `src/` and `test/` must be `.ts` files
- Use Zod for runtime validation, not just type annotations

### HTTP Requests
- **NEVER** patch `global.fetch` or other globals
- Use dependency injection: pass `httpRequest` through constructors
- Add cross-cutting concerns via policies, not inline code

### Testing with Vitest
- Place tests in `test/` directory, mirroring `src/` structure
- Use `setupHttpRequest()` from `test/utils/httpPipelineUtils.ts` to mock HTTP
- Test through real pipeline (with UserAgentPolicy), not global mocks
- Cover: URL construction, error handling, input validation, output formatting

**Test Pattern:**
```typescript
import { setupHttpRequest } from '../../utils/httpPipelineUtils.js';

const { httpRequest, mockHttpRequest } = setupHttpRequest();
const tool = new MyTool({ httpRequest });
const result = await tool.run({ param: 'value' });

// Assert URL construction
expect(mockHttpRequest.mock.calls[0][0]).toContain('expected-path');
```

### Tool Development Checklist
1. Use `npx plop create-tool` for scaffolding
2. Define input/output schemas with `.describe()` for documentation
3. Extend `MapboxApiBasedTool` for Mapbox API tools
4. Accept `{ httpRequest }` in constructor
5. Implement `execute()` method with error handling
6. Register in `toolRegistry.ts` (Plop does this automatically)
7. Write tests covering URL construction, errors, validation
8. Update README.md with tool description (Plop adds stub)

## Environment Variables

### Required
- `MAPBOX_ACCESS_TOKEN` - Mapbox API token (JWT format validated)

### Optional
- `MAPBOX_API_ENDPOINT` - Override API endpoint (default: `https://api.mapbox.com/`)

### OpenTelemetry Configuration
- `OTEL_EXPORTER_OTLP_ENDPOINT` - OTLP endpoint (e.g., `http://localhost:4318`)
- `OTEL_SERVICE_NAME` - Override service name (default: `mapbox-mcp-server`)
- `OTEL_EXPORTER_OTLP_HEADERS` - JSON string of additional headers
- `OTEL_LOG_LEVEL` - Diagnostic log level: `NONE` (default), `ERROR`, `WARN`, `INFO`, `DEBUG`, `VERBOSE`

## Important Constraints

### From Copilot Instructions (`.github/copilot-instructions.md`)
- Review all AI-generated code for correctness, security, and style
- Never accept code with hardcoded secrets or that patches globals
- All AI-generated code requires unit tests and documentation
- Follow project standards (TypeScript, linting, typing, testing)

### Build System
- Uses `tshy` to generate ESM and CommonJS outputs
- Build process: `npm run prepare` → `tshy` → `generate-version` → `add-shebang`
- Pre-commit hooks enforce linting and formatting via Husky
- Requires Node.js >= 22 (see `engines` in package.json)

### MCP Protocol
- Tools expose both `text` (human-readable) and `structuredContent` (machine-parseable) outputs
- Use MCP annotations (`readOnlyHint`, `idempotentHint`, etc.) on tool classes
- Server runs on stdio transport (not HTTP)

## Additional Resources

- **README.md** - Comprehensive user documentation, integration guides, example prompts
- **docs/tracing.md** - Detailed OpenTelemetry setup for various platforms
- **docs/claude-desktop-setup.md** - Claude Desktop integration guide
- **.github/copilot-instructions.md** - GitHub Copilot usage guidelines
- **CHANGELOG.md** - User-facing changes (semantic versioning)
