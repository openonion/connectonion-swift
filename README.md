# ConnectOnion-Swift

A Swift port of the ConnectOnion framework - a simple, powerful agent framework with behavior tracking for building AI-powered applications.

## 🚀 Features

- **🤖 Simple Agent Creation** - Build AI agents with just a few lines of code
- **🔧 Extensible Tool System** - Give agents capabilities through tools
- **📝 Automatic Behavior Tracking** - All interactions logged for analysis and replay
- **🎯 Type-Safe** - Leverage Swift's type system for safe tool creation
- **🔌 OpenAI Compatible** - Works with OpenAI API and compatible services
- **🎨 Multiple UI Options** - CLI, Terminal UI, and macOS desktop app examples

## 📦 Installation

### Swift Package Manager

Add ConnectOnion to your `Package.swift`:

```swift
dependencies: [
    .package(url: "https://github.com/openonion/connectonion-swift.git", from: "0.0.1")
]
```

### Build from Source

```bash
git clone https://github.com/openonion/connectonion-swift.git
cd connectonion-swift
swift build
```

## 🎯 Quick Start

### 1. Set up your environment

Create a `.env` file with your OpenAI API key:

```bash
OPENAI_API_KEY=your-api-key-here
OPENAI_MODEL=gpt-4o-mini  # Optional, defaults to gpt-4o-mini
OPENAI_BASE_URL=https://api.openai.com/v1  # Optional, for custom endpoints
```

### 2. Create a simple agent

```swift
import ConnectOnion

// Load environment variables
loadEnvironment()

// Create an agent with automatic LLM configuration
let agent = Agent(
    name: "my-assistant",
    systemPrompt: "You are a helpful assistant.",
    apiKey: ProcessInfo.processInfo.environment["OPENAI_API_KEY"]
)

// Get a response
let response = try await agent.input("Hello, how are you?")
print(response)
```

### 3. Add tools to give your agent capabilities

```swift
// Define a simple calculator tool
struct CalculatorTool: Tool {
    let name = "calculator"
    let summary = "Perform basic math operations"
    
    var parameters: JSONValue {
        .object([
            "type": .string("object"),
            "properties": .object([
                "operation": .object(["type": .string("string"), "enum": .array([.string("add"), .string("multiply")])]),
                "a": .object(["type": .string("number")]),
                "b": .object(["type": .string("number")])
            ]),
            "required": .array([.string("operation"), .string("a"), .string("b")])
        ])
    }
    
    func call(args: [String: JSONValue]) async throws -> JSONValue {
        guard case let .string(op)? = args["operation"],
              case let .number(a)? = args["a"],
              case let .number(b)? = args["b"] else {
            return .object(["error": .string("Invalid arguments")])
        }
        
        let result: Double
        switch op {
        case "add": result = a + b
        case "multiply": result = a * b
        default: return .object(["error": .string("Unknown operation")])
        }
        
        return .object(["result": .number(result)])
    }
}

// Register the tool with the agent
await agent.registerTool(CalculatorTool())

// The agent can now use the calculator
let answer = try await agent.input("What is 42 times 17?")
print(answer)  // "42 times 17 equals 714"
```

## 📚 Core Concepts

### Agents

Agents are the main interface for interacting with LLMs. They orchestrate:
- **Conversation context** - Maintains message history
- **Tool execution** - Calls tools when needed
- **Behavior tracking** - Logs all interactions
- **Iteration control** - Limits tool-calling rounds

```swift
let agent = Agent(
    name: "assistant",           // Unique identifier for history tracking
    systemPrompt: "...",         // System instructions for the LLM
    maxIterations: 10,           // Maximum tool-calling rounds (default: 10)
    apiKey: "...",              // API credentials (or use environment)
    historyDir: URL(...)        // Custom history location (optional)
)
```

### Tools

Tools give agents the ability to perform actions. There are two ways to create tools:

#### 1. Protocol-based Tools (Full Control)

Implement the `Tool` protocol for complete control over parameters and execution:

```swift
struct WeatherTool: Tool {
    let name = "get_weather"
    let summary = "Get current weather for a location"
    
    var parameters: JSONValue {
        .object([
            "type": .string("object"),
            "properties": .object([
                "location": .object([
                    "type": .string("string"),
                    "description": .string("City name or coordinates")
                ]),
                "units": .object([
                    "type": .string("string"),
                    "enum": .array([.string("celsius"), .string("fahrenheit")]),
                    "default": .string("celsius")
                ])
            ]),
            "required": .array([.string("location")])
        ])
    }
    
    func call(args: [String: JSONValue]) async throws -> JSONValue {
        guard case let .string(location)? = args["location"] else {
            return .object(["error": .string("Location required")])
        }
        
        let units = args["units"].flatMap { 
            if case let .string(u) = $0 { return u } else { return nil }
        } ?? "celsius"
        
        // Fetch weather data...
        let temp = 22.5  // Example
        
        return .object([
            "location": .string(location),
            "temperature": .number(temp),
            "units": .string(units),
            "conditions": .string("Sunny")
        ])
    }
}
```

#### 2. Function-based Tools (Type-Safe)

Use `FunctionTool` for type-safe tool creation with Codable types:

```swift
// Define typed parameters and results
struct SearchParams: Codable {
    let query: String
    let limit: Int?
    let category: String?
}

struct SearchResult: Codable {
    let items: [String]
    let totalCount: Int
}

// Create a tool from a typed function
let searchTool = createTool(
    name: "search",
    summary: "Search for items in the database",
    parameterType: SearchParams.self
) { (params: SearchParams) async throws -> SearchResult in
    // Type-safe implementation
    let results = try await performSearch(
        params.query, 
        limit: params.limit ?? 10,
        category: params.category
    )
    return SearchResult(items: results, totalCount: results.count)
}

// Register with agent
await agent.registerTool(searchTool)
```

### Behavior Tracking

All agent interactions are automatically saved for analysis and debugging:

```swift
// History is saved to ~/.connectonion/agents/{name}/behavior.json by default
let agent = Agent(name: "tracker")

// Or specify custom location
let customDir = URL(fileURLWithPath: "/path/to/history", isDirectory: true)
let agent = Agent(name: "tracker", historyDir: customDir)

// Each interaction records:
// - Timestamp
// - Input/output messages
// - Tool calls with arguments and results
// - Execution timing
// - Model metadata
```

History format example:
```json
{
  "agent": "tracker",
  "task": "What's 2+2?",
  "timestamp": "2024-01-01T12:00:00Z",
  "messages": [...],
  "toolCalls": [
    {
      "name": "calculator",
      "args": {"operation": "add", "a": 2, "b": 2},
      "result": {"result": 4},
      "timingMS": 15
    }
  ],
  "metadata": {"model": "gpt-4o-mini"}
}
```

### Message Flow

Maintain conversation context across multiple interactions:

```swift
// Initialize conversation context
var context: [Message] = []

// First interaction
let response1 = try await agent.input(
    "My name is Alice and I like Swift programming", 
    messages: context
)
context.append(Message(role: .user, content: "My name is Alice and I like Swift programming"))
context.append(Message(role: .assistant, content: response1))

// Second interaction - agent remembers context
let response2 = try await agent.input(
    "What's my name and what do I like?", 
    messages: context
)
// Agent responds: "Your name is Alice and you like Swift programming"
```

## 🎨 Examples

### 1. CLI Assistant (`examples/CLIAssistant`)

A command-line assistant with safe shell command execution:

```bash
cd examples/CLIAssistant
swift run

# Interactive session:
You> List files in current directory
Assistant> I'll list the files for you... [executes ls command]
You> What's today's date?
Assistant> [executes date command and shows result]
```

**Features:**
- Safe shell execution (only allowed commands: echo, ls, pwd, date, whoami)
- Sandboxed file access
- Interactive command-line interface
- Context preservation

### 2. Terminal UI (`examples/CLITerminalUI`)

Enhanced terminal interface with visual feedback:

```bash
cd examples/CLITerminalUI
swift run
```

**Features:**
- Full-screen terminal UI with colored output
- Scrollable conversation history
- Real-time tool execution feedback
- Keyboard shortcuts:
  - `/help` - Show available commands
  - `/clear` - Clear conversation
  - `/tools` - List available tools
  - `/quit` or `exit` - Exit application

### 3. Desktop App (`examples/DesktopAssistantApp`)

Native macOS SwiftUI application:

```bash
cd examples/DesktopAssistantApp
swift run
```

**Features:**
- Native macOS interface with SwiftUI
- Markdown rendering for responses
- Copy-to-clipboard functionality
- Quick action buttons for common tasks
- Real-time streaming responses
- Tool execution visualization

## 🔧 Advanced Usage

### Custom LLM Configuration

```swift
// Use a custom OpenAI-compatible endpoint
let llm = createLLM(
    model: "claude-3-opus",
    apiKey: "your-key",
    baseURL: URL(string: "https://custom-api.example.com/v1")
)

let agent = Agent(name: "custom", llm: llm)
```

### Batch Tool Registration

```swift
// Register multiple tools at once
let tools: [any Tool] = [
    CalculatorTool(),
    WeatherTool(),
    SearchTool(),
    FileTool()
]

await agent.registerTools(tools)
```

### Dynamic Iteration Control

```swift
// Set default max iterations for the agent
let agent = Agent(name: "limited", maxIterations: 5)

// Override for specific complex tasks
let result = try await agent.input(
    "Perform a complex multi-step analysis",
    maxIterations: 20  // Allow more iterations for this task
)
```

### Environment Configuration

```swift
// Load from default .env file
loadEnvironment()

// Load from custom path
loadEnvironment(from: "/path/to/config.env")

// Environment variables used:
// - OPENAI_API_KEY: Your OpenAI API key
// - OPENAI_MODEL: Model to use (default: gpt-4o-mini)
// - OPENAI_BASE_URL: API endpoint (default: https://api.openai.com/v1)
// - CONNECTONION_WORKDIR: Working directory for file operations
```

## 🏗 Architecture

```
ConnectOnion/
├── Agent.swift              # Main agent orchestrator
├── Core/
│   ├── Tool.swift          # Tool protocol and registry
│   ├── Message.swift       # Message types (user/assistant/system/tool)
│   ├── JSONValue.swift     # Type-safe JSON handling
│   └── FunctionTool.swift  # Function-based tool creation
├── LLM/
│   ├── LLM.swift           # LLM protocol definition
│   └── LLMFactory.swift    # Factory for LLM creation
├── OpenAI/
│   └── OpenAIClient.swift  # OpenAI API implementation
├── History/
│   └── History.swift       # Behavior tracking and persistence
└── ConnectOnion.swift      # Public API exports
```

## 🧪 Testing

Run the comprehensive test suite:

```bash
# Run all tests
swift test

# Run specific test
swift test --filter FunctionToolTests

# Run with verbose output
swift test --verbose
```

Test coverage includes:
- **Unit tests** for all components
- **Integration tests** for agent workflows
- **Mock LLM** for testing without API calls
- **Tool execution** tests
- **History tracking** verification

## 🤝 Contributing

We welcome contributions! Please:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Write tests for new functionality
4. Ensure all tests pass
5. Submit a Pull Request

## 📄 License

MIT License - See LICENSE file for details

## 🙏 Acknowledgments

This is a Swift port of the original [ConnectOnion](https://github.com/openonion/connectonion) Python framework, maintaining API compatibility and behavior parity for cross-platform agent development.

