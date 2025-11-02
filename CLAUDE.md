# CLAUDE.md
m

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

ConnectOnion-Swift is a Swift port of the ConnectOnion AI agent framework. It maintains API compatibility and behavior parity with the Python implementation while leveraging Swift's type safety and async/await patterns.

**Core Philosophy**: "Keep simple things simple, make complicated things possible"

## Development Commands

### Build and Test
```bash
# Build the package
swift build

# Run all tests
swift test

# Run specific test
swift test --filter FunctionToolTests
swift test --filter AgentTests

# Verbose test output
swift test --verbose

# Build in release mode
swift build -c release
```

### Running Examples
```bash
# CLI Assistant (basic command-line interface)
cd examples/CLIAssistant
swift run

# Terminal UI (enhanced with colors)
cd examples/CLITerminalUI
swift run

# Desktop App (macOS SwiftUI)
cd examples/DesktopAssistantApp
swift run
```

### CLI Tool
```bash
# Run the main CLI
swift run connectonion-cli

# Build and install locally
swift build -c release
cp .build/release/connectonion-cli /usr/local/bin/
```

## High-Level Architecture

### Module Structure (Sources/)
```
ConnectOnion/
├── Agent.swift              # Main orchestrator (actor-based)
├── Core/
│   ├── Tool.swift          # Tool protocol + ToolRegistry
│   ├── FunctionTool.swift  # Type-safe Codable tool creation
│   ├── Message.swift       # Message(role, content, toolCalls)
│   └── JSONValue.swift     # Recursive enum for JSON handling
├── LLM/
│   ├── LLM.swift           # Protocol: chat(messages, tools) -> LLMResponse
│   └── LLMFactory.swift    # createLLM() with env/key resolution
├── OpenAI/
│   └── OpenAIClient.swift  # OpenAI implementation (function calling)
├── History/
│   └── History.swift       # HistoryStore for behavior.json persistence
└── ConnectOnion.swift      # Public API exports
```

### Core Interaction Loop (Agent.swift:97-150)

The agent's `input()` method implements the agentic loop:
1. Add system prompt if configured
2. Add user message to conversation
3. **Loop until no tool calls** (max iterations = 10 by default):
   - Call LLM with current messages + tool specs
   - If response contains tool calls:
     - Execute all tools in parallel
     - Add tool results to messages
     - Continue loop
   - If no tool calls: return assistant's message
4. Save interaction to history

**Key Implementation Details**:
- Agent is an `actor` for thread-safe state management
- Tool execution happens concurrently using `withThrowingTaskGroup`
- Tool errors are returned as JSONValue errors, not thrown (allows LLM to adapt)
- History saves to `.connectonion/agents/{name}/behavior.json` by default

### Tool System

**Two patterns for tool creation**:

1. **Protocol-based** (full control over JSON schema):
```swift
struct WeatherTool: Tool {
    let name = "get_weather"
    let summary = "Get weather for a city"
    var parameters: JSONValue { /* JSON schema */ }
    func call(args: [String: JSONValue]) async throws -> JSONValue
}
```

2. **Function-based** (type-safe with Codable):
```swift
struct WeatherRequest: Codable { let city: String }
let tool = createTool(
    name: "get_weather",
    summary: "Get weather",
    parameterType: WeatherRequest.self
) { (req: WeatherRequest) -> WeatherResponse in
    // Type-safe implementation
}
```

Tools are registered via `agent.registerTool()` and stored in `ToolRegistry` (Sources/ConnectOnion/Core/Tool.swift:58).

### LLM Abstraction

The `LLM` protocol (Sources/ConnectOnion/LLM/LLM.swift) allows swapping backends:
```swift
protocol LLM {
    func chat(messages: [Message], tools: [ToolSpec]) async throws -> LLMResponse
}
```

Current implementation: `OpenAIClient` (Sources/ConnectOnion/OpenAI/OpenAIClient.swift)
- Supports OpenAI-compatible APIs (including custom base URLs)
- Uses function calling for tool execution
- Converts between internal Message format and OpenAI JSON

Factory function `createLLM()` handles environment variable resolution:
- `OPENAI_API_KEY` (required)
- `OPENAI_MODEL` (default: gpt-4o-mini)
- `OPENAI_BASE_URL` (default: https://api.openai.com/v1)

### Behavior Tracking

Every agent interaction auto-saves to JSON (Sources/ConnectOnion/History/History.swift):
```json
{
  "agent": "name",
  "task": "user input",
  "timestamp": "2024-01-01T12:00:00Z",
  "messages": [...],
  "toolCalls": [{"name": "calc", "args": {...}, "result": {...}, "timingMS": 15}],
  "metadata": {"model": "gpt-4o-mini"}
}
```

History schema matches Python implementation for cross-platform replay/analysis.

## Key Design Patterns

### Actor-Based Agent
`Agent` is declared as `actor` (not class) to prevent data races when managing tools, history, and conversation state.

### Sendable Constraints
All shared types conform to `Sendable`:
- `Tool` protocol
- `Message`, `ToolCall`, `JSONValue` structs
- Ensures safe concurrency across actor boundaries

### Error Handling Philosophy
- Tool errors are captured as `JSONValue` and returned to LLM (not thrown)
- Allows agent to react to failures and retry
- Only throw for unrecoverable errors (network, auth, etc.)

### Environment Configuration
Use `loadEnvironment()` (Sources/ConnectOnion/ConnectOnion.swift) to load `.env` files:
```swift
loadEnvironment()  // Loads from ./.env
loadEnvironment(from: "/path/to/file")
```

## Testing Strategy

Location: `Tests/ConnectOnionTests/`

### Test Files
- `AgentTests.swift` - Agent loop, tool calling, iterations
- `FunctionToolTests.swift` - Type-safe tool creation
- `OpenAITests.swift` - Request/response formatting
- `HistoryTests.swift` - Persistence and JSON roundtrip
- `LLMFactoryTests.swift` - Environment variable resolution
- `CLITests.swift` - CLI argument parsing

### Mock Infrastructure
- `MockLLM` - Returns deterministic responses for testing
- `MockTransport` - Fakes HTTP for OpenAI client tests
- `TempDataDir` - Isolates filesystem writes per test

### Running Tests
```bash
swift test                           # All tests
swift test --filter AgentTests       # Specific test target
swift test --verbose                 # Detailed output
```

## Development Patterns

### Creating a New Tool
1. Define the tool struct conforming to `Tool` protocol
2. Implement `name`, `summary`, `parameters`, and `call()`
3. Register with agent: `await agent.registerTool(MyTool())`

For type-safe tools with Codable:
```swift
let tool = createTool(
    name: "my_tool",
    summary: "Description",
    parameterType: MyRequest.self
) { (req: MyRequest) async throws -> MyResponse in
    // Implementation
}
```

### Adding a New LLM Backend
1. Create new file in `Sources/ConnectOnion/LLM/`
2. Implement `LLM` protocol
3. Update `LLMFactory.swift` to instantiate your backend

### Conversation Context
Maintain context across interactions by passing message array:
```swift
var context: [Message] = []
let response = try await agent.input("Hello", messages: context)
context.append(Message(role: .user, content: "Hello"))
context.append(Message(role: .assistant, content: response))
```

## Platform and Compatibility

### Requirements
- **Platform**: macOS 13.0+ (defined in Package.swift:7)
- **Swift**: 5.9+ (uses async/await, actors, Sendable)
- **Dependencies**: None (stdlib only)

### Cross-Platform Considerations
While currently macOS-only, the core is portable. To add Linux/Windows:
1. Update `Package.swift` platforms
2. Test `FileManager` and `ProcessInfo` usage
3. Verify URL path handling on target platform

### Python Compatibility
History JSON schema mirrors Python ConnectOnion for replay compatibility.
Tool naming and behavior should match Python equivalents when porting.

## Common Gotchas

1. **Agent is an actor** - Must use `await` for all agent method calls
2. **Tools must be Sendable** - Avoid captured mutable state
3. **Tool errors don't throw** - Return errors as JSONValue for LLM visibility
4. **History dir must exist** - Create `.connectonion/` or pass custom dir
5. **Environment vars are optional** - Can pass `apiKey` directly to Agent/createLLM

## Project Philosophy Reminders

- **Simplicity first**: Default everything, make complex cases possible
- **Type safety without ceremony**: Use Swift types but avoid generic complexity
- **Functions as primitives**: Tools are just functions with schemas
- **Observable by default**: Automatic history tracking
- **No over-engineering**: No try/catch unless explicitly needed (see ~/.claude/CLAUDE.md)