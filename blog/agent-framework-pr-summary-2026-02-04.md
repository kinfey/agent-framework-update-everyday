# Agent Framework Updates - February 4, 2026

The Microsoft Agent Framework saw significant activity on February 4, 2026, with 13 merged pull requests introducing major architectural improvements, breaking changes, and important bug fixes. This update brings substantial enhancements to both .NET and Python implementations, focusing on API consistency, type safety, and performance optimization.

## ⚠️ BREAKING CHANGES

### 1. .NET: Obsoleting ReflectingExecutor in Favor of Source Generation

**Impact**: High - Requires migration from reflection-based to source generation approach

**PR**: [#3380](https://github.com/microsoft/agent-framework/pull/3380)

The reflection-based handler discovery approach (`ReflectingExecutor<T>`, `IMessageHandler<T>`, `IMessageHandler<T,R>`) has been marked as obsolete in favor of the new `[MessageHandler]` attribute with source generation. This change improves performance by eliminating runtime reflection overhead.

**Before (Reflection-based):**
```csharp
public class MyWorkflow : IMessageHandler<MyMessage>
{
    public async Task<object> HandleAsync(MyMessage message)
    {
        // Handler logic
    }
}

var executor = new ReflectingExecutor<MyWorkflow>();
```

**After (Source Generation):**
```csharp
public partial class MyWorkflow
{
    [MessageHandler]
    public async Task HandleMessage(MyMessage message)
    {
        // Handler logic
    }
}
```

**Migration Guide:**
- Add `[MessageHandler]` attribute to your handler methods
- Make your workflow class `partial`
- Remove `IMessageHandler<T>` interface implementations
- Update `Directory.Build.props` to include the generator package

### 2. Python: Types API Review - Major Type System Overhaul

**Impact**: High - Breaking changes to core type system

**PR**: [#3647](https://github.com/microsoft/agent-framework/pull/3647)

The Python API has undergone a comprehensive type system refactoring to improve type safety and developer experience.

**Key Breaking Changes:**

#### Role and FinishReason Now Use NewType + Literal

**Before:**
```python
from agent_framework import Role

role = Role.USER
if role.value == "user":  # .value access required
    pass
```

**After:**
```python
from agent_framework import Role

role = "user"  # Direct string usage
if role == "user":  # Direct comparison
    pass
```

#### ChatMessage Signature Changed

**Before:**
```python
message = ChatMessage(role="user", text="Hello")
```

**After:**
```python
message = ChatMessage("user", ["Hello"])
# or
message = ChatMessage("user", [Content.from_text("Hello")])
```

#### Removed try_parse_value Method

**Before:**
```python
result = response.try_parse_value(MyType)
if result:
    data = result
```

**After:**
```python
try:
    data = response.value
except Exception as e:
    # Handle parsing error
    pass
```

#### Method Renames

- `from_chat_response_updates` → `from_updates`
- `from_chat_response_generator` → `from_update_generator`
- `from_agent_run_response_updates` → `from_updates`

### 3. .NET: Structured Output Support

**Impact**: Medium - Changes to RunAsync API

**PR**: [#3658](https://github.com/microsoft/agent-framework/pull/3658)

Added structured output support with new `ChatResponse` property in `AgentRunOptions`. The `bool? useJsonSchemaResponseFormat` parameter has been removed from existing `RunAsync<T>` signatures.

**Before:**
```csharp
var result = await agent.RunAsync<MyType>(
    message,
    useJsonSchemaResponseFormat: true
);
```

**After:**
```csharp
var options = new AgentRunOptions
{
    ChatResponse = new ChatResponseOptions
    {
        ResponseFormat = ResponseFormat.JsonSchema<MyType>()
    }
};
var result = await agent.RunAsync<MyType>(message, options);
```

**New Features:**
- `RunAsync<T>` methods added to `AIAgent` base class (non-virtual)
- `Clone` method added to `AgentRunOptions`, `ChatClientAgentRunOptions`, `DurableAgentOptions`
- Support for structured output type represented by `System.Type`

### 4. .NET: Move AgentSession.Serialize to AIAgent

**Impact**: Medium - API location change

**PR**: [#3650](https://github.com/microsoft/agent-framework/pull/3650)

The `Serialize` method has been moved from `AgentSession` to `AIAgent` for better architectural consistency.

**Before:**
```csharp
string json = agentSession.Serialize();
```

**After:**
```csharp
string json = agent.Serialize();
```

### 5. .NET: Rename Session State JSON Parameter

**Impact**: Low - Parameter naming consistency

**PR**: [#3681](https://github.com/microsoft/agent-framework/pull/3681)

Session state JSON parameter names have been standardized across different classes in core libraries for consistency.

**Updated Classes:**
- Parameter renamed from various names to consistent `sessionStateJson` across all session-related methods

## 🚀 Major Updates

### .NET: Improved Unit Test Coverage for Microsoft.Agents.AI.Abstractions

**PR**: [#3381](https://github.com/microsoft/agent-framework/pull/3381) by Copilot

Comprehensive test coverage improvements bringing code coverage from 82.9% to above 85% target.

**Key Additions:**
- Tests for `AIAgentMetadata` (new test file)
- Tests for `AgentResponseExtensions.AsChatResponse` and `AsChatResponseUpdate`
- Tests for `AgentResponseExtensions.AsChatResponseUpdatesAsync`
- Tests for `AgentResponse.UserInputRequests` and `AgentResponseUpdate.UserInputRequests`
- Tests for `AgentResponse.ToAgentResponseUpdates`
- Tests for `DelegatingAIAgent.DeserializeThreadAsync`
- Tests for `InMemoryChatMessageStore` edge cases

**Results:** All 265 tests pass successfully on .NET 10.0

### .NET: Add Anthropic Claude Skills Sample

**PR**: [#3497](https://github.com/microsoft/agent-framework/pull/3497) by Copilot

New sample demonstrating Anthropic Claude Skills support in .NET, following Stephen Toub's recommended simplified approach using `AsAITool()`.

**Sample Features:**
```csharp
// List available Anthropic-managed skills
var skills = await anthropicClient.Beta.ListSkillsAsync();

// Create skill configuration
BetaSkillParams pptxSkill = new()
{
    Type = BetaSkillParamsType.Anthropic,
    SkillID = "pptx",
    Version = "latest"
};

// Create agent with skills
ChatClientAgent agent = anthropicClient.Beta.AsAIAgent(
    model: model,
    instructions: "You are a helpful agent for creating PowerPoint presentations.",
    tools: [pptxSkill.AsAITool()]
);

// Extended thinking configuration
var response = await agent.RunAsync(
    "Create a presentation about AI",
    new AnthropicRunOptions { ExtendedThinking = true }
);

// Download generated files via Files API
byte[] fileContent = await anthropicClient.Files.DownloadAsync(fileId);
```

**Key Advantage:** `AsAITool()` automatically handles beta headers and container configuration - no manual `RawRepresentationFactory` needed!

### .NET: Rename 3rdPartyThreadStorage to 3rdPartyChatHistoryStorage

**PR**: [#3643](https://github.com/microsoft/agent-framework/pull/3643)

Sample renamed for better clarity about its purpose - demonstrates chat history storage integration with third-party providers.

## 🐛 Bug Fixes and Improvements

### Python: Improve Logging in Declarative Code (Security Fix)

**PR**: [#3573](https://github.com/microsoft/agent-framework/pull/3573)

**Security Issue:** Fixed clear-text logging of sensitive information in PowerFx evaluation failures.

**Before:**
```python
if log_value:
    logger.debug(f"PowerFx evaluation failed for value '{value}': {exc}")
else:
    logger.debug(f"PowerFx evaluation failed for value (first five characters shown) '{value[:5]}': {exc}")
```

**After:**
```python
if log_value:
    logger.debug("PowerFx evaluation failed for a value: %s", exc)
else:
    logger.debug("PowerFx evaluation failed for a value (details redacted): %s", exc)
```

**Impact:** Prevents potential exposure of sensitive data (passwords, API keys, tokens) in log messages while preserving error debugging information.

### Python: Fix Workflow Cancellation Not Propagating to Active Executors

**PR**: [#3663](https://github.com/microsoft/agent-framework/pull/3663)

**Issue:** When a workflow was cancelled via `asyncio.create_task()` + `task.cancel()`, executors that were mid-execution continued running to completion instead of being cancelled.

**Root Cause:** The `CancelledError` was raised in the polling loop, but the `iteration_task` created via `asyncio.create_task()` was orphaned and kept running.

**Fix:** Proper cancellation propagation to ensure all active executors respect workflow cancellation.

**Test Coverage:** Added comprehensive unit tests for workflow cancellation scenarios.

### Python: Adjust TypeVars to Suffix Naming Convention

**PR**: [#3661](https://github.com/microsoft/agent-framework/pull/3661)

Updated TypeVar naming in `core/_workflows` to use suffix convention consistent with the rest of the codebase:

- `T_Out` → `OutT`
- `T_W_Out` → `W_OutT`
- `_T_EdgeGroup` → `EdgeGroupT` (moved to module level)
- `GroupChatWorkflowContext_T_Out` → `GroupChatWorkflowContextOutT`

**Additional Improvements:**
- Removed public export of `SharedState` (internal-only)
- Fixed MCP tool kwargs serialization bug where non-serializable framework kwargs (e.g., `response_format`) weren't filtered before telemetry JSON serialization

### Python: Fix Claude JSON Schema for Nested Pydantic Models

**PR**: [#3655](https://github.com/microsoft/agent-framework/pull/3655)

**Critical Bug:** `_function_tool_to_sdk_mcp_tool()` was dropping the `$defs` section from Pydantic JSON schemas, breaking tools with nested Pydantic models.

**Before:**
```python
input_schema: dict[str, Any] = {
    "type": "object",
    "properties": schema.get("properties", {}),
    "required": schema.get("required", []),
}
# $defs was not being preserved!
```

**After:**
```python
input_schema: dict[str, Any] = {
    "type": "object",
    "properties": schema.get("properties", {}),
    "required": schema.get("required", []),
}
if "$defs" in schema:
    input_schema["$defs"] = schema["$defs"]
```

**Impact:** Tools with nested Pydantic models now generate valid JSON schemas. Claude can properly resolve `$ref` references like `"$ref": "#/$defs/SomeNestedType"`.

### Python: Skip model_deployment_name Validation for Azure Foundry Application Endpoints

**PR**: [#3621](https://github.com/microsoft/agent-framework/pull/3621)

**Issue:** `AzureAIClient` incorrectly required `model_deployment_name` for application endpoints even though the model is pre-configured on the server.

**Before:**
```python
# Had to provide dummy placeholder
client = AzureAIClient(
    endpoint="https://app.foundry.azure.com/...",
    model_deployment_name="placeholder"  # Unnecessary!
)
# ValueError: model_deployment_name must be a non-empty string
```

**After:**
```python
# Application endpoints don't need model_deployment_name
client = AzureAIClient(
    endpoint="https://app.foundry.azure.com/..."
)
# Works correctly - model is pre-configured on server
```

**Implementation:**
```python
@override
def _check_model_presence(self, run_options: dict[str, Any]) -> None:
    # Skip model check for application endpoints - model is pre-configured on server
    if self._is_application_endpoint:
        return
    if not run_options.get("model"):
        if not self.model_id:
            raise ValueError("model_deployment_name must be a non-empty string")
        run_options["model"] = self.model_id
```

## 📊 Summary

February 4, 2026 brought transformative changes to the Microsoft Agent Framework:

**Breaking Changes (5):** Major improvements to type systems, API consistency, and performance through source generation adoption.

**New Features (2):** Anthropic Claude Skills support, enhanced structured output capabilities.

**Bug Fixes (6):** Critical security fixes, workflow cancellation improvements, and schema generation fixes.

**Test Coverage:** Significant improvements with 265+ passing tests in .NET and 1639+ passing tests in Python.

## 🎯 Recommended Actions

1. **Immediate:** Review breaking changes if using affected APIs
2. **High Priority:** Migrate from `ReflectingExecutor` to `[MessageHandler]` attribute (.NET)
3. **High Priority:** Update Python code to use new type system (Role, FinishReason, ChatMessage)
4. **Medium Priority:** Update structured output calls to use new `AgentRunOptions` pattern (.NET)
5. **Security:** Ensure sensitive data is not being logged (check PowerFx evaluation contexts)
6. **Testing:** Verify workflow cancellation behavior in your applications
7. **Azure Users:** Remove unnecessary `model_deployment_name` from Foundry application endpoint configurations

## 📚 Related Issues

- Closes #3334 (Unit test coverage)
- Closes #2690 (Anthropic Skills sample)
- Closes #3431 (Sample rename)
- Closes #3646 (AgentSession.Serialize location)
- Fixes #3608 (Workflow cancellation)
- Closes #3660 (TypeVar naming)
- Fixes #3654 (Claude nested Pydantic models)
- Fixes security scanning issue #49

---

*This update demonstrates the Agent Framework team's commitment to API consistency, type safety, and developer experience while maintaining backward compatibility where possible through migration guides and comprehensive testing.*
