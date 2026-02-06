# Agent Framework Updates - February 5, 2026

The Microsoft Agent Framework witnessed significant activity on February 5, 2026, with 14 merged pull requests bringing substantial improvements to both .NET and Python implementations. This update introduces critical breaking changes alongside major architectural enhancements, bug fixes, and improved developer documentation.

## ⚠️ BREAKING CHANGES

### 1. .NET: Agent and Session Now Provided to AIContextProvider & ChatHistoryProvider

**Impact**: High - Requires updates to all custom AIContextProvider and ChatHistoryProvider implementations

**PR**: [#3695](https://github.com/microsoft/agent-framework/pull/3695)

The context provider interfaces have been enhanced to provide access to the agent and session instances during invocation. This enables providers to make context-aware decisions based on agent metadata and session state.

**Before:**
```csharp
var invokingContext = new ChatHistoryProvider.InvokingContext(messages);
var storeMessages = await chatHistoryProvider.InvokingAsync(invokingContext, cancellationToken);
```

**After:**
```csharp
var invokingContext = new ChatHistoryProvider.InvokingContext(this, session, messages);
var storeMessages = await typedSession.ChatHistoryProvider.InvokingAsync(invokingContext, cancellationToken);
```

**New InvokingContext Constructor:**
```csharp
public InvokingContext(
    AIAgent agent,
    AgentSession? session,
    IEnumerable<ChatMessage> requestMessages)
{
    this.Agent = Throw.IfNull(agent);
    this.Session = session;
    this.RequestMessages = Throw.IfNull(requestMessages);
}
```

**Migration Guide:**
- Update all `InvokingContext` instantiations to include `agent` and `session` parameters
- Update all `InvokedContext` instantiations similarly
- Access agent metadata via `context.Agent` property
- Access session state via `context.Session` property (nullable)

### 2. Python: Move Orchestrations to Dedicated Package

**Impact**: High - Requires package restructuring and import updates

**PR**: [#3685](https://github.com/microsoft/agent-framework/pull/3685)

Orchestration components (Magentic, GroupChat, Handoff, Sequential, Concurrent) have been extracted into a separate `agent-framework-orchestrations` package for better modularity and maintainability.

**Before:**
```python
from agent_framework import GroupChatBuilder, MagenticBuilder
from agent_framework import HandoffBuilder, SequentialBuilder
```

**After:**
```python
from agent_framework_orchestrations import GroupChatBuilder, MagenticBuilder
from agent_framework_orchestrations import HandoffBuilder, SequentialBuilder
```

**Migration Guide:**
- Install the new package: `pip install agent-framework-orchestrations`
- Update all orchestration imports from `agent_framework` to `agent_framework_orchestrations`
- The core `agent_framework` package now provides stub re-exports for backward compatibility during migration

**Affected Components:**
- `AgentBasedGroupChatOrchestrator`, `GroupChatBuilder`, `GroupChatOrchestrator`
- `MagenticBuilder`, `MagenticOrchestrator`, `StandardMagenticManager`
- `HandoffBuilder`, `HandoffConfiguration`, `HandoffAgentExecutor`
- `SequentialBuilder`, `ConcurrentBuilder`

### 3. .NET: Remove UserInputRequests Property

**Impact**: Medium - Affects code using the deprecated UserInputRequests property

**PR**: [#3682](https://github.com/microsoft/agent-framework/pull/3682)

The `UserInputRequests` property has been removed from agent session interfaces as part of API cleanup.

**Migration Guide:**
- Remove references to `UserInputRequests` property
- Use alternative session state management patterns

### 4. Python: Refactor SharedState to State with Sync Methods and Superstep Caching

**Impact**: High - Requires updates to workflow state management code

**PR**: [#3667](https://github.com/microsoft/agent-framework/pull/3667)

The `SharedState` class has been refactored to `State` with synchronous methods and built-in superstep caching for improved performance and simpler API.

**Before:**
```python
async def send_message(self, message: Message, shared_state: SharedState, ctx: RunnerContext) -> bool:
    # Work with async shared state
    await shared_state.set("key", value)
    data = await shared_state.get("key")
```

**After:**
```python
async def send_message(self, message: Message, state: State, ctx: RunnerContext) -> bool:
    # Work with synchronous state methods
    state.set("key", value)
    data = state.get("key")
```

**Key Changes:**
- Renamed `SharedState` → `State`
- State methods are now synchronous (no `await` needed)
- Automatic superstep caching for performance optimization
- Simplified API surface

**Migration Guide:**
- Replace `SharedState` with `State` in all method signatures
- Remove `await` from state access methods (`get`, `set`, etc.)
- Update executor implementations to use the new state API

### 5. Python: Fix Workflow as Agent Streaming Output

**Impact**: Medium - Changes to workflow streaming behavior

**PR**: [#3649](https://github.com/microsoft/agent-framework/pull/3649)

Workflow streaming output has been fixed to properly emit `AgentResponseUpdate` objects when used as an agent.

**Migration Guide:**
- Review workflow-as-agent integrations for streaming behavior changes
- Ensure consumers properly handle the updated streaming protocol

### 6. Python: Unified get_response and run API

**Impact**: High - Major API consolidation affecting all agent invocations

**PR**: [#3379](https://github.com/microsoft/agent-framework/pull/3379)

The Python agent API has been unified to a single `run()` method with a `stream` parameter, replacing the separate `run()` and `run_stream()` methods.

**Before:**
```python
# Non-streaming
response = await agent.run(messages)

# Streaming
async for update in agent.run_stream(messages):
    print(update)
```

**After:**
```python
# Non-streaming (returns Awaitable[AgentResponse])
response = await agent.run(messages, stream=False)

# Streaming (returns ResponseStream)
stream = agent.run(messages, stream=True)
async for update in stream:
    print(update)
```

**New Method Signatures:**
```python
@overload
def run(
    self,
    messages: str | ChatMessage | Sequence[str | ChatMessage] | None = None,
    *,
    stream: Literal[False] = ...,
    thread: AgentThread | None = None,
    **kwargs: Any,
) -> Awaitable[AgentResponse[Any]]: ...

@overload
def run(
    self,
    messages: str | ChatMessage | Sequence[str | ChatMessage] | None = None,
    *,
    stream: Literal[True],
    thread: AgentThread | None = None,
    **kwargs: Any,
) -> ResponseStream[AgentResponseUpdate, AgentResponse[Any]]: ...
```

**Migration Guide:**
- Replace `await agent.run(messages)` with `await agent.run(messages, stream=False)`
- Replace `agent.run_stream(messages)` with `agent.run(messages, stream=True)`
- Update type annotations to expect `Awaitable[AgentResponse]` or `ResponseStream` based on the `stream` parameter
- The `run_stream()` method is deprecated and should no longer be used

## Major Updates

### .NET: Add Decorator for Structured Output Support

**PR**: [#3694](https://github.com/microsoft/agent-framework/pull/3694)

Added decorator pattern support for structured output in agents that don't natively support it, improving flexibility for model integrations.

**Key Features:**
- Enables structured output for models without native support
- Decorator-based implementation for easy composition
- Comprehensive documentation and examples included

### .NET: Fix Error 404 Agent Hosted MCP

**PR**: [#3678](https://github.com/microsoft/agent-framework/pull/3678)

Fixed critical issue where hosted MCP agents failed with "ID cannot be null or empty" error when deployed to Azure AI Foundry.

**Changes:**
```csharp
// Added helper method to handle empty version IDs
private static AgentReference CreateAgentReference(...)
{
    // Default empty version to "latest"
    string version = string.IsNullOrEmpty(agentVersion.Version) 
        ? "latest" 
        : agentVersion.Version;
    
    // Generate fallback ID from name and version if needed
    string id = string.IsNullOrEmpty(agentVersion.Id)
        ? $"{agentVersion.Name}-{version}"
        : agentVersion.Id;
    
    return new AgentReference(id, version);
}
```

**Impact:**
- Resolves 404 errors for MCP agents in Azure AI Foundry
- Improves deployment reliability
- Adds unit tests for edge cases

### .NET: Adding AgentRunContext for External Components

**PR**: [#3476](https://github.com/microsoft/agent-framework/pull/3476)

Introduced `AgentRunContext` using `AsyncLocal` to provide agent run information to external downstream components like middleware and logging.

**Usage Example:**
```csharp
// Access current agent context from anywhere in the execution flow
var context = AgentRunContext.Current;
if (context != null)
{
    var agentName = context.Agent.Name;
    var sessionId = context.Session?.Id;
    // Use context in middleware, logging, etc.
}
```

**Key Features:**
- AsyncLocal-based context propagation
- Accessible from middleware and external components
- Nullable session support for stateless scenarios
- Comprehensive ADR documentation

### Python: Prevent WorkflowExecutor from Re-sending Answered Requests After Checkpoint Restore

**PR**: [#3689](https://github.com/microsoft/agent-framework/pull/3689)

Fixed bug where `WorkflowExecutor` would re-send already answered requests after restoring from a checkpoint, preventing duplicate processing.

**Impact:**
- Improved checkpoint/restore reliability
- Prevents duplicate request handling
- Better state consistency

## Minor Updates and Bug Fixes

### Python: Handle API Errors in Claude run_stream() Method

**PR**: [#3653](https://github.com/microsoft/agent-framework/pull/3653)

Enhanced error handling in the Claude agent's streaming implementation to properly catch and raise `ServiceException` for API errors.

**Changes:**
```python
# Check for errors in streaming response
if isinstance(message, AssistantMessage) and message.error:
    raise ServiceException(f"Claude API error: {message.error}")

if isinstance(message, ResultMessage) and message.is_error:
    error_text = message.content[0].text if message.content else "Unknown error"
    raise ServiceException(f"Claude execution error: {error_text}")
```

**Benefits:**
- Better error visibility
- Consistent exception handling
- Improved debugging experience

### .NET & Python: Add AGENTS.md Files and Update Coding Standards

**PR**: [#3644](https://github.com/microsoft/agent-framework/pull/3644)

Comprehensive documentation improvements including:

- Added `AGENTS.md` files describing package structure and main classes
- Updated `CODING_STANDARD.md` with API review guidance:
  - Future annotations convention
  - TypeVar naming convention (e.g., `GroupChatWorkflowContextOutT` instead of `GroupChatWorkflowContext_T_Out`)
  - Mapping vs MutableMapping usage
  - Avoiding shadowing built-ins
  - Explicit exports requirements
  - Exception documentation guidelines
- Enhanced Copilot instructions with language-specific guidelines
- Added ADR section explaining templates and purpose

### Python: Fix AG-UI Message Handling and MCP Tool Double-Call Bug

**PR**: [#3635](https://github.com/microsoft/agent-framework/pull/3635)

Fixed issues in the AG-UI package:
- Corrected message handling between AG-UI and agent framework
- Resolved MCP tool double-call bug that caused duplicate executions
- Improved message adapter reliability

### Python: Added Internal Kwargs Filtering for Anthropic Client

**PR**: [#3544](https://github.com/microsoft/agent-framework/pull/3544)

Implemented internal kwargs filtering for the Anthropic client to prevent invalid parameters from being passed to the API.

**Benefits:**
- Cleaner API calls
- Better parameter validation
- Reduced API errors

## Summary

February 5, 2026 represents a significant milestone for the Microsoft Agent Framework with 14 merged PRs introducing both breaking changes and substantial improvements:

**Breaking Changes Summary:**
- 5 breaking changes affecting both .NET and Python implementations
- Major API consolidation in Python with unified `run()` method
- Context providers in .NET now receive agent and session instances
- Orchestrations moved to dedicated package in Python
- State management refactored from `SharedState` to `State`

**Key Improvements:**
- Enhanced structured output support in .NET
- Better error handling across both platforms
- Improved documentation and coding standards
- Critical bug fixes for MCP agents and workflow execution
- New `AgentRunContext` for external component integration

**Recommended Actions:**

1. **For .NET Developers:**
   - Review and update all `AIContextProvider` and `ChatHistoryProvider` implementations
   - Remove references to deprecated `UserInputRequests` property
   - Test MCP agent deployments to Azure AI Foundry
   - Consider using `AgentRunContext` for middleware and logging

2. **For Python Developers:**
   - Install `agent-framework-orchestrations` package if using orchestration features
   - Update all agent invocations to use the new unified `run()` API
   - Migrate from `SharedState` to `State` in workflow implementations
   - Update orchestration imports to the new package
   - Review streaming behavior in workflow-as-agent scenarios

3. **For All Developers:**
   - Read the updated `AGENTS.md` and `CODING_STANDARD.md` documentation
   - Test checkpoint/restore functionality thoroughly
   - Update error handling to leverage improved exception messages

This release focuses on API consolidation, improved modularity, and enhanced developer experience. While the breaking changes require migration effort, they set the foundation for a more consistent and maintainable framework going forward.
