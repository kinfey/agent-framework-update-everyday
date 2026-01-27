# Agent Framework Updates - January 26, 2026

On January 26, 2026, the Microsoft Agent Framework received 5 important updates, including one major breaking change that fundamentally changes how agent sessions are handled. This release focuses on improving developer experience through better terminology, enhanced testing coverage, and new integrations with the GitHub Copilot SDK for Python.

## ⚠️ BREAKING CHANGES

### .NET: AgentThread Renamed to AgentSession

**PR**: [#3430](https://github.com/microsoft/agent-framework/pull/3430)  
**Impact**: HIGH - Requires code changes across all .NET projects using AgentThread  
**Author**: [@westey-m](https://github.com/westey-m)

This significant rename addresses issue [#775](https://github.com/microsoft/agent-framework/issues/775) by changing `AgentThread` to `AgentSession` throughout the entire .NET codebase to better describe what the concept actually represents. The term "session" more accurately reflects a single execution context for an agent interaction, whereas "thread" was causing confusion with execution threads and conversation threads.

#### What Changed

The rename affects all core APIs, including:
- **Core Classes**: `AgentThread` → `AgentSession`
- **Store Abstractions**: `AgentThreadStore` → `AgentSessionStore`
- **Implementation Classes**: `InMemoryAgentThreadStore` → `InMemoryAgentSessionStore`
- **Provider-Specific Types**: 
  - `A2AAgentThread` → `A2AAgentSession`
  - `CopilotStudioAgentThread` → `CopilotStudioAgentSession`
  - `DurableAgentThread` → `DurableAgentSession`

#### Code Migration Examples

**Before:**
```csharp
// Creating and using an agent thread
var thread = await agent.GetNewThreadAsync();

// Running an agent with a thread
var response = await agent.RunAsync(
    messages, 
    thread: thread,  // Old parameter name
    options: runOptions
);

// Storing threads
builder.WithInMemoryThreadStore();

// In custom agent implementations
protected override Task<AgentResponse> RunCoreAsync(
    IEnumerable<ChatMessage> messages, 
    AgentThread? thread = null,  // Old type
    AgentRunOptions? options = null, 
    CancellationToken cancellationToken = default)
{
    // Implementation
}
```

**After:**
```csharp
// Creating and using an agent session
var session = await agent.GetNewSessionAsync();

// Running an agent with a session
var response = await agent.RunAsync(
    messages, 
    session: session,  // New parameter name
    options: runOptions
);

// Storing sessions
builder.WithInMemorySessionStore();

// In custom agent implementations
protected override Task<AgentResponse> RunCoreAsync(
    IEnumerable<ChatMessage> messages, 
    AgentSession? session = null,  // New type
    AgentRunOptions? options = null, 
    CancellationToken cancellationToken = default)
{
    // Implementation
}
```

#### Streaming API Changes

**Before:**
```csharp
protected override async IAsyncEnumerable<AgentResponseUpdate> RunCoreStreamingAsync(
    IEnumerable<ChatMessage> messages, 
    AgentThread? thread = null,  // Old parameter
    AgentRunOptions? options = null, 
    [EnumeratorCancellation] CancellationToken cancellationToken = default)
{
    await foreach (var update in this.InnerAgent.RunStreamingAsync(
        messages, thread, options, cancellationToken))
    {
        yield return update;
    }
}
```

**After:**
```csharp
protected override async IAsyncEnumerable<AgentResponseUpdate> RunCoreStreamingAsync(
    IEnumerable<ChatMessage> messages, 
    AgentSession? session = null,  // New parameter
    AgentRunOptions? options = null, 
    [EnumeratorCancellation] CancellationToken cancellationToken = default)
{
    await foreach (var update in this.InnerAgent.RunStreamingAsync(
        messages, session, options, cancellationToken))
    {
        yield return update;
    }
}
```

#### Memory Provider Scope Changes

**Before:**
```csharp
public class ChatHistoryMemoryProviderScope
{
    public string ThreadId { get; set; }  // Old property name
}
```

**After:**
```csharp
public class ChatHistoryMemoryProviderScope
{
    public string SessionId { get; set; }  // New property name
}
```

#### Workflow and Durable Task Integration

**DurableTask Context Before:**
```csharp
public class DurableAgentContext
{
    public DurableAgentThread CurrentThread { get; set; }
}
```

**DurableTask Context After:**
```csharp
public class DurableAgentContext
{
    public DurableAgentSession CurrentSession { get; set; }
}
```

#### Hosting and Dependency Injection

**Before:**
```csharp
// Configure agent hosting
builder.Services.AddSingleton<AgentThreadStore>(
    new InMemoryAgentThreadStore());

// Or use extension method
builder.WithInMemoryThreadStore();

// In Azure Functions
services.AddSingleton<IAgentThreadStore, CosmosAgentThreadStore>();
```

**After:**
```csharp
// Configure agent hosting
builder.Services.AddSingleton<AgentSessionStore>(
    new InMemoryAgentSessionStore());

// Or use extension method
builder.WithInMemorySessionStore();

// In Azure Functions
services.AddSingleton<IAgentSessionStore, CosmosAgentSessionStore>();
```

#### Migration Checklist

To migrate your codebase:

1. **Find and Replace** all occurrences (case-sensitive):
   - `AgentThread` → `AgentSession`
   - `AgentThreadStore` → `AgentSessionStore`
   - `InMemoryAgentThreadStore` → `InMemoryAgentSessionStore`
   - `NoopAgentThreadStore` → `NoopAgentSessionStore`
   - `WithInMemoryThreadStore` → `WithInMemorySessionStore`
   - `WithThreadStore` → `WithSessionStore`
   - `ThreadId` → `SessionId` (in appropriate contexts)
   - `thread:` → `session:` (parameter names)
   - `CurrentThread` → `CurrentSession`

2. **Update Custom Agent Implementations**: Change method signatures to use `AgentSession` instead of `AgentThread`

3. **Update DI Configuration**: Replace thread store registrations with session store equivalents

4. **Review Serialization**: Update JSON serialization contexts if using `AgentJsonUtilities` or similar

5. **Test Thoroughly**: Run all tests to ensure no broken references

#### Files Changed

This breaking change touched **150+ files** across:
- Core abstractions (`Microsoft.Agents.AI.Abstractions`)
- Agent implementations (`Microsoft.Agents.AI`)
- Workflow integration (`Microsoft.Agents.AI.Workflows`)
- Durable Task integration (`Microsoft.Agents.AI.DurableTask`)
- Hosting libraries (`Microsoft.Agents.AI.Hosting.*`)
- Provider integrations (A2A, Azure Functions, Copilot Studio, Purview, Mem0)
- All samples and tests

---

## Major Updates

### Python: GitHub Copilot SDK Integration

**PR**: [#3404](https://github.com/microsoft/agent-framework/pull/3404)  
**Author**: [@dmytrostruk](https://github.com/dmytrostruk)

A new package `agent-framework-github-copilot` has been added, providing seamless integration between the Agent Framework and GitHub Copilot SDK. This enables developers to leverage Copilot's agentic capabilities within the Agent Framework ecosystem.

#### New Package Structure

```python
from agent_framework_github_copilot import (
    GithubCopilotAgent,
    GithubCopilotOptions,
    GithubCopilotSettings
)
```

#### Key Features

**1. BaseAgent Implementation**

The new `GithubCopilotAgent` class implements the Agent Framework's `BaseAgent` interface, providing:
- Seamless integration with existing agent workflows
- Standard agent lifecycle management
- Consistent API surface with other Agent Framework agents

**2. Configuration Options**

```python
from agent_framework_github_copilot import GithubCopilotOptions

options = GithubCopilotOptions(
    model="gpt-4",
    temperature=0.7,
    max_tokens=2000,
    # Additional Copilot-specific settings
)

agent = GithubCopilotAgent(options=options)
```

**3. Settings Management**

```python
from agent_framework_github_copilot import GithubCopilotSettings

settings = GithubCopilotSettings(
    api_key="your-api-key",
    endpoint="https://api.github.com/copilot"
)
```

#### Usage Example

```python
from agent_framework_github_copilot import GithubCopilotAgent, GithubCopilotOptions
from agent_framework import AgentRuntime

# Create Copilot agent
copilot_agent = GithubCopilotAgent(
    options=GithubCopilotOptions(
        model="gpt-4",
        temperature=0.7
    )
)

# Use within Agent Framework
runtime = AgentRuntime()
runtime.register_agent("copilot", copilot_agent)

# Run the agent
response = await copilot_agent.run("Help me write a Python function")
print(response.content)
```

#### Benefits

- **Unified Interface**: Use GitHub Copilot with the same API as other Agent Framework agents
- **Composability**: Combine Copilot agents with other agents in workflows
- **Type Safety**: Full type hints and IntelliSense support
- **Async Support**: Built-in async/await patterns for efficient execution

---

## Minor Updates and Bug Fixes

### .NET: Improved Unit Test Coverage for Microsoft.Agents.AI.Anthropic

**PR**: [#3382](https://github.com/microsoft/agent-framework/pull/3382)  
**Author**: GitHub Copilot (automated)

Unit test coverage for the Anthropic integration package was increased from **68.7% to 85%+** to meet the required coverage threshold.

#### Tests Added

**AnthropicClientExtensions Tests:**
```csharp
[Fact]
public async Task AsAIAgent_WithChatOptions_ReturnsValidAgent()
{
    // Arrange
    var chatClient = new AnthropicChatClient(apiKey, modelId);
    var chatOptions = new ChatOptions { Temperature = 0.7f };
    
    // Act
    var agent = chatClient.AsAIAgent(chatOptions);
    
    // Assert
    Assert.NotNull(agent);
    Assert.IsType<ChatClientAgent>(agent);
}

[Fact]
public async Task AsAIAgent_WithNullClient_ThrowsArgumentNullException()
{
    // Arrange
    AnthropicChatClient? client = null;
    
    // Act & Assert
    Assert.Throws<ArgumentNullException>(() => client.AsAIAgent());
}
```

**AnthropicBetaServiceExtensions Tests:**
```csharp
[Fact]
public void AddAnthropicChatClient_RegistersServices()
{
    // Arrange
    var services = new ServiceCollection();
    
    // Act
    services.AddAnthropicChatClient(config => 
    {
        config.ApiKey = "test-key";
        config.ModelId = "claude-3-opus-20240229";
    });
    
    // Assert
    var provider = services.BuildServiceProvider();
    var chatClient = provider.GetService<IChatClient>();
    Assert.NotNull(chatClient);
}
```

#### Impact

- Increased confidence in Anthropic integration reliability
- Better error handling validation
- Comprehensive edge case coverage
- All 25 existing tests continue to pass

---

### .NET: Improved Unit Test Coverage for Microsoft.Agents.AI.AzureAI.Persistent

**PR**: [#3384](https://github.com/microsoft/agent-framework/pull/3384)  
**Author**: GitHub Copilot (automated)

Enhanced test coverage for the Azure AI Persistent Agents integration, focusing on edge cases and error handling.

#### Key Test Improvements

**1. AsAIAgent Overload Tests**
```csharp
[Fact]
public async Task AsAIAgent_WithChatOptions_ConfiguresCorrectly()
{
    // Test that ChatOptions are properly passed through
    var client = CreateTestClient();
    var options = new ChatOptions 
    { 
        MaxTokens = 1000,
        Temperature = 0.5f 
    };
    
    var agent = client.AsAIAgent(options);
    
    // Verify options are applied
    Assert.NotNull(agent);
}
```

**2. Null Safety Tests**
```csharp
[Fact]
public void AsAIAgent_WithNullClient_ThrowsException()
{
    AzureAIClient? client = null;
    Assert.Throws<ArgumentNullException>(() => client.AsAIAgent());
}
```

**3. Session Management Tests**
```csharp
[Fact]
public async Task CreateSession_WithValidParameters_ReturnsSession()
{
    var client = CreateTestClient();
    var agent = client.AsAIAgent();
    
    var session = await agent.GetNewSessionAsync();
    
    Assert.NotNull(session);
    Assert.NotNull(session.Id);
}
```

#### Results

- Coverage increased to meet 85% threshold
- All 25 initial tests continue passing
- New tests added for previously uncovered code paths
- Better validation of null and edge cases

---

### DevContainer: Workaround for Expired Key Issue

**PR**: [#3432](https://github.com/microsoft/agent-framework/pull/3432)  
**Author**: [@westey-m](https://github.com/westey-m)

Fixed a critical issue where .NET devcontainers wouldn't start due to an expired GPG key in the Yarn APT repository.

#### The Problem

Microsoft's official .NET devcontainers were failing to start with errors during `apt-get update` due to an expired signing key for the Yarn repository.

#### The Solution

**New Dockerfile:**
```dockerfile
# .devcontainer/dotnet/dotnet.Dockerfile
FROM mcr.microsoft.com/devcontainers/dotnet:2-10.0

# Remove the Yarn APT repository with expired GPG key
RUN rm -f /etc/apt/sources.list.d/yarn.list

# Update package lists
RUN apt-get update && apt-get upgrade -y
```

**Updated devcontainer.json:**
```json
{
  "build": {
    "dockerfile": "dotnet.Dockerfile"
  },
  // Commented out direct image reference
  // "image": "mcr.microsoft.com/devcontainers/dotnet:1-9.0-bookworm",
  
  "features": {
    "ghcr.io/devcontainers/features/dotnet:2": {
      "version": "10.0"
    },
    "ghcr.io/devcontainers/features/azure-cli:1": {
      "version": "latest"
    },
    "ghcr.io/devcontainers/features/docker-in-docker:2": {
      "version": "latest"
    }
  }
}
```

#### Benefits

- Devcontainers now start successfully
- Tracks latest .NET dev container image
- Modernized feature versions:
  - .NET 10.0 support
  - Latest Azure CLI
  - Latest Docker-in-Docker

#### Impact

Developers can now successfully:
- Open the repository in GitHub Codespaces
- Use VS Code Dev Containers locally
- Contribute without environment setup issues

---

## Summary

This release brings significant improvements to the Microsoft Agent Framework:

### Key Highlights

1. **⚠️ Breaking Change - Better Terminology**: The rename from `AgentThread` to `AgentSession` provides clearer semantics and reduces confusion with threading concepts. While this requires migration effort, it improves long-term code clarity.

2. **🎯 GitHub Copilot Integration**: Python developers can now use GitHub Copilot SDK within the Agent Framework ecosystem, enabling powerful agentic capabilities with a unified interface.

3. **✅ Improved Quality**: Automated test coverage improvements for Anthropic and Azure AI integrations increase reliability and reduce potential bugs.

4. **🔧 Better Developer Experience**: DevContainer fixes ensure smooth onboarding and development environment setup.

### Recommended Actions

#### For .NET Developers (URGENT)

**Action Required**: Migrate from `AgentThread` to `AgentSession`

```bash
# Use find-and-replace in your codebase:
# AgentThread → AgentSession
# AgentThreadStore → AgentSessionStore
# InMemoryAgentThreadStore → InMemoryAgentSessionStore
# thread: → session: (in parameter names)
# ThreadId → SessionId (in appropriate contexts)
```

**Testing Steps:**
1. Update all references to use new naming
2. Run your test suite to identify any missed updates
3. Verify serialization/deserialization if using persistent sessions
4. Check dependency injection configurations

#### For Python Developers

**Try the New GitHub Copilot Integration:**

```bash
pip install agent-framework-github-copilot
```

```python
from agent_framework_github_copilot import GithubCopilotAgent, GithubCopilotOptions

agent = GithubCopilotAgent(
    options=GithubCopilotOptions(model="gpt-4")
)

response = await agent.run("Your prompt here")
```

#### For All Developers

**Update Your DevContainer:**
- Pull latest changes to get the devcontainer fix
- Rebuild your dev container if you experienced startup issues
- Enjoy improved development environment stability

### Testing Considerations

- **Breaking Change Testing**: All 150+ affected .NET files have been updated and tested
- **Python Integration**: New GitHub Copilot package includes comprehensive tests
- **Coverage Improvements**: Both Anthropic and Azure AI packages now exceed 85% coverage
- **DevContainer**: Verified across multiple environments (Codespaces, local VS Code)

### Impact Assessment

| Category | Impact Level | Migration Effort |
|----------|-------------|------------------|
| .NET AgentThread → AgentSession | **HIGH** | **MEDIUM** - Find and replace across codebase |
| Python GitHub Copilot SDK | **MEDIUM** | **LOW** - New feature, opt-in |
| Test Coverage Improvements | **LOW** | **NONE** - No API changes |
| DevContainer Fix | **MEDIUM** | **NONE** - Automatic on rebuild |

---

**Contributors**: [@westey-m](https://github.com/westey-m), [@dmytrostruk](https://github.com/dmytrostruk), GitHub Copilot  
**Total PRs Merged**: 5  
**Total Files Changed**: 200+ files  
**Lines Changed**: ~8,000 additions, ~4,000 deletions  

For detailed code changes, visit the individual PRs linked above.
