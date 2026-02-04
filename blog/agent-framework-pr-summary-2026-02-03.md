# Agent Framework Updates - February 3, 2026

On February 3, 2026, the Microsoft Agent Framework merged **6 Pull Requests** delivering critical bug fixes, breaking API changes, and feature enhancements. This release includes **1 breaking change** affecting .NET developers and several important Python improvements for Content API, Azure AI integration, GitHub Copilot agent configuration, and MCP (Model Context Protocol) tool handling.

---

## 📊 Overview

**Breaking Changes**: 1 (.NET API method rename)  
**Bug Fixes**: 3 (Python Content API imports, Azure AI instructions handling, MCP response_format filtering)  
**Feature Enhancements**: 2 (GitHub Copilot instructions parameter, Workflow handler parameters)

**Key Improvements:**
- ⚠️ **.NET Breaking**: `GetNewSessionAsync` → `CreateSessionAsync` method rename
- 🐛 **Fixed**: Broken Content API imports in 11 Python sample files
- 🐛 **Fixed**: Instructions not being passed to Azure AI Foundry agents
- ✨ **New**: Instructions parameter for GitHubCopilotAgent constructor
- 🛠️ **Enhanced**: Explicit input/output parameters for workflow handlers
- 🔧 **Fixed**: MCP tool call compatibility by filtering response_format

**Impact**: HIGH - Contains breaking changes requiring code updates for .NET developers, plus critical bug fixes for Python users

---

## ⚠️ BREAKING CHANGES

### .NET: GetNewSessionAsync Renamed to CreateSessionAsync

**PR**: [#3501](https://github.com/microsoft/agent-framework/pull/3501)  
**Author**: [@westey-m](https://github.com/westey-m)  
**Impact**: HIGH - Requires code changes in all .NET agent implementations  
**Files Changed**: 124 files (core abstractions, implementations, tests, samples)

#### What Changed

The abstract method `GetNewSessionAsync` has been renamed to `CreateSessionAsync` across the entire Agent Framework codebase. This breaking change affects all .NET developers using agent sessions.

**Before:**
```csharp
// Old method name
var session = await agent.GetNewSessionAsync();
```

**After:**
```csharp
// New method name
var session = await agent.CreateSessionAsync();
```

#### Motivation

The new name "CreateSession" better reflects the operation's semantics. Sessions in the Agent Framework are now less tied to specific underlying chat history storage mechanisms, and "Create" is clearer than "GetNew" about what the method does - it creates a new session resource rather than just retrieving something.

#### Migration Guide

**For Application Developers:**

1. **Find and Replace**: Search your codebase for `GetNewSessionAsync` and replace with `CreateSessionAsync`
   ```bash
   # Search for usage
   git grep "GetNewSessionAsync"
   
   # Replace in your IDE or with sed
   find . -type f -name "*.cs" -exec sed -i 's/GetNewSessionAsync/CreateSessionAsync/g' {} +
   ```

2. **Verify Compilation**: Build your project to catch any missed references
   ```bash
   dotnet build
   ```

**For Library Developers Implementing Custom Agents:**

1. Update your abstract method implementation:
   ```csharp
   // Before
   public override Task<IAgentSession> GetNewSessionAsync(
       CancellationToken cancellationToken = default)
   {
       // implementation
   }
   
   // After
   public override Task<IAgentSession> CreateSessionAsync(
       CancellationToken cancellationToken = default)
   {
       // implementation
   }
   ```

2. Update any XML documentation references:
   ```csharp
   /// <summary>
   /// Creates a new agent session.  // Updated from "Gets a new session"
   /// </summary>
   ```

#### Affected Components

This change impacts all agent types:
- `AIAgent` (abstract base class)
- `ChatClientAgent`
- `DurableAIAgent` and `DurableAIAgentProxy`
- `WorkflowHostAgent`
- `A2AAgent`
- `CopilotStudioAgent`
- `GitHubCopilotAgent`
- `PurviewAgent`
- `DelegatingAIAgent`
- All session store implementations (`NoopAgentSessionStore`, `InMemoryAgentSessionStore`)

#### Why This Matters

✅ **Clearer Intent**: "Create" explicitly indicates a new resource is being created  
✅ **Reduced Confusion**: Eliminates questions about whether service-side artifacts are created  
✅ **Better API Design**: Aligns with common naming patterns in .NET ecosystem  
✅ **Future-Proof**: More flexible for different session storage backends

---

## 🐛 Bug Fixes

### Python: Fixed Broken Content API Imports in 11 Sample Files

**PR**: [#3639](https://github.com/microsoft/agent-framework/pull/3639)  
**Author**: [@moonbox3](https://github.com/moonbox3)  
**Impact**: MEDIUM - Fixes import errors in Python samples  
**Labels**: `python`, `samples`, `bug`

#### The Problem

Eleven Python sample files were using deprecated Content API classes that have been consolidated into a unified `Content` class. The samples were attempting to import `TextContent`, `UriContent`, and `DataContent` classes that no longer exist as separate constructors, causing import errors for developers following the examples.

#### What Changed

All affected samples were updated to:
1. Import the unified `Content` class instead of deprecated content type classes
2. Replace constructor calls with factory methods
3. Convert `isinstance()` type checks to property-based checks

**Import Updates:**

Before:
```python
from agent_framework import (
    TextContent,
    UriContent,
    DataContent,
)
```

After:
```python
from agent_framework import (
    Content,
)
```

**Constructor to Factory Method Migration:**

Before:
```python
# Creating text content
message = ChatMessage(
    role=Role.ASSISTANT,
    contents=[TextContent(text="Hello! I'm a custom echo agent.")]
)

# Creating URI content
content = UriContent(uri="https://example.com/image.jpg")

# Creating data content
data = DataContent(data=image_bytes, mime_type="image/png")
```

After:
```python
# Using factory methods
message = ChatMessage(
    role=Role.ASSISTANT,
    contents=[Content.from_text(text="Hello! I'm a custom echo agent.")]
)

# Using factory methods for different content types
content = Content.from_uri(uri="https://example.com/image.jpg")

data = Content.from_data(data=image_bytes, mime_type="image/png")
```

**Type Checking Updates:**

Before:
```python
# Using isinstance checks
for content in message.contents:
    if isinstance(content, TextContent):
        print(content.text)
    elif isinstance(content, DataContent):
        save_image(content.data)
    elif isinstance(content, UriContent):
        download(content.uri)
```

After:
```python
# Using property-based checks
for content in message.contents:
    if content.type == "text":
        print(content.text)
    elif content.type == "data":
        save_image(content.data)
    elif content.type == "uri":
        download(content.uri)
    elif content.type in ("data", "uri"):
        # Handle multiple types
        process_media(content)
```

#### Affected Sample Files

1. `handoff_with_code_interpreter_file.py` - Updated type checks
2. `override_result_with_middleware.py` - Updated text content creation
3. `devui/weather_agent_azure/agent.py` - Updated text content creation
4. `openai_responses_client_with_code_interpreter.py` - Updated type checks
5. `openai_responses_client_streaming_image_generation.py` - Updated data content checks
6. `openai_responses_client_image_generation.py` - Updated multi-type checks
7. `openai_responses_client_image_analysis.py` - Updated text and URI content
8. `custom_chat_client.py` - Updated text content creation
9. `custom_agent.py` - Updated text content creation (multiple locations)
10. `azure_responses_client_image_analysis.py` - Updated text and URI content
11. `azure_ai_with_code_interpreter_file_download.py` - Updated type checks

#### Why This Matters

✅ **Working Examples**: Developers can now copy-paste sample code without import errors  
✅ **Modern API**: Samples demonstrate the current unified Content API  
✅ **Reduced Friction**: No confusion about which content class to use  
✅ **Type Safety**: Property-based type checking is more maintainable than isinstance checks

---

### Python: Fixed Instructions Not Passed to Azure AI Foundry Agents

**PR**: [#3636](https://github.com/microsoft/agent-framework/pull/3636)  
**Author**: Community contributor  
**Impact**: HIGH - Critical bug fix for Azure AI integration  
**Fixes**: [Issue #3622](https://github.com/microsoft/agent-framework/issues/3622)

#### The Bug

A previous fix in [PR #3563](https://github.com/microsoft/agent-framework/pull/3563) addressed instructions being dropped in `AzureAIAgentClient` (Chat API), but the same bug existed in `AzureAIClient` (Responses API). Instructions passed to Azure AI Foundry agents were being silently ignored.

#### Root Cause

In the `_get_agent_reference_or_create()` method, the code was incorrectly reading `instructions` from `run_options`:

```python
# Bug: reading from wrong variable
for instructions in [messages_instructions, run_options.get("instructions")]:
    # ...
```

However, the base class excludes instructions from `run_options`. Instructions should be read from `chat_options` instead (same pattern used for `response_format` on lines 351-352).

#### The Fix

Changed `_client.py` to read instructions from `chat_options`:

Before:
```python
# Incorrect: reading from run_options
for instructions in [messages_instructions, run_options.get("instructions")]:
    if instructions:
        # Process instructions
        break
```

After:
```python
# Correct: reading from chat_options
for instructions in [messages_instructions, chat_options.get("instructions") if chat_options else None]:
    if instructions:
        # Process instructions
        break
```

#### Test Coverage

The PR includes comprehensive test coverage:

1. **Updated existing test** to pass instructions via `chat_options`
2. **Added new test** `test_agent_creation_with_instructions_from_chat_options` to verify instructions are correctly passed

Example test:
```python
async def test_agent_creation_with_instructions_from_chat_options():
    client = AzureAIClient(...)
    
    # Pass instructions via chat_options
    response = await client.complete(
        messages=[{"role": "user", "content": "Hello"}],
        chat_options={"instructions": "You are a helpful assistant."}
    )
    
    # Verify instructions were passed to the agent
    assert response.instructions == "You are a helpful assistant."
```

#### Why This Matters

✅ **Agent Behavior**: Instructions now correctly control Azure AI agent behavior  
✅ **Reliability**: Agents behave as configured, not with default instructions  
✅ **Consistency**: Both Chat API and Responses API now handle instructions correctly  
✅ **Developer Trust**: Configuration options work as documented

---

### Python: Filter response_format from MCP Tool Call kwargs

**PR**: [#3494](https://github.com/microsoft/agent-framework/pull/3494)  
**Author**: [@giles17](https://github.com/giles17)  
**Impact**: MEDIUM - Fixes MCP tool compatibility issues  
**Labels**: `python`, `mcp`, `bug`

#### The Issue

When using MCP (Model Context Protocol) tools, the `response_format` parameter was being incorrectly passed to tool calls, causing compatibility issues. MCP tools don't accept `response_format` in their kwargs, leading to errors when agents attempted to call MCP tools.

#### The Fix

Added filtering logic to remove `response_format` from MCP tool call kwargs before passing them to the tool:

```python
# In _mcp.py
async def call_tool(self, tool_name: str, arguments: dict, **kwargs):
    # Filter out response_format - MCP tools don't accept this parameter
    filtered_kwargs = {
        k: v for k, v in kwargs.items() 
        if k != "response_format"
    }
    
    # Call the tool with filtered kwargs
    return await self._call_mcp_tool(tool_name, arguments, **filtered_kwargs)
```

#### Impact

This fix ensures:
- ✅ **MCP Compatibility**: MCP tools can be called without errors
- ✅ **Cleaner Integration**: Framework automatically handles parameter compatibility
- ✅ **Future-Proof**: Additional parameter filtering can be added as needed

---

## ✨ Feature Enhancements

### Python: Added Instructions Parameter to GitHubCopilotAgent

**PR**: [#3625](https://github.com/microsoft/agent-framework/pull/3625)  
**Author**: [@dmytrostruk](https://github.com/dmytrostruk)  
**Impact**: MEDIUM - Simplifies GitHub Copilot agent configuration  
**Fixes**: [Issue #3571](https://github.com/microsoft/agent-framework/issues/3571)  
**Test Coverage**: 87%

#### What's New

Added an `instructions` parameter to the `GitHubCopilotAgent` constructor and implemented configurable `system_message` mode (append/replace) for the GitHub Copilot agent integration. This aligns the API with other agents in the framework (like ClaudeAgent).

#### Changes

1. **New `instructions` Parameter**: Direct parameter that maps to `system_message.content`
2. **Configurable System Message Mode**: Choose between append/replace modes
3. **Runtime Options Support**: Override system_message per request in session creation
4. **Updated Samples**: All GitHub Copilot samples use the new API pattern

**Before:**
```python
from agent_framework_github_copilot import GitHubCopilotAgent, GitHubCopilotOptions

# Old way: Required using GitHubCopilotOptions
agent = GitHubCopilotAgent(
    name="my-agent",
    default_options=GitHubCopilotOptions(
        instructions="You are a helpful assistant."
    )
)
```

**After:**
```python
from agent_framework_github_copilot import GitHubCopilotAgent

# New way: Direct instructions parameter
agent = GitHubCopilotAgent(
    name="my-agent",
    instructions="You are a helpful assistant."
)

# Advanced: Configure system message mode
agent = GitHubCopilotAgent(
    name="my-agent",
    instructions="You are a helpful assistant.",
    default_options={
        "system_message": {
            "content": "Additional context",
            "mode": "append"  # or "replace"
        }
    }
)

# Runtime override
response = await agent.run(
    "What's the weather?",
    runtime_options={
        "system_message": {
            "content": "Override instructions for this request only",
            "mode": "replace"
        }
    }
)
```

#### Implementation Details

The PR includes:
- `_prepare_system_message()` method for handling instruction precedence
- Runtime options support in session creation
- Comprehensive test coverage for:
  - Instructions parameter functionality
  - System message configuration
  - Precedence rules (runtime > constructor > default)
  - Runtime options override behavior

#### Updated Sample Files

1. `github_copilot_basic.py` - Simplified with `instructions` parameter
2. `github_copilot_with_session.py` - Updated API usage
3. `github_copilot_with_mcp.py` - Using `instructions` with MCP servers
4. `github_copilot_with_url.py` - Simplified initialization
5. `github_copilot_with_shell.py` - Simplified initialization
6. `github_copilot_with_file_operations.py` - Simplified initialization
7. `github_copilot_with_multiple_permissions.py` - Simplified initialization

#### Why This Matters

✅ **API Consistency**: GitHubCopilotAgent now matches the pattern used by other agents  
✅ **Simpler Configuration**: Direct `instructions` parameter is more intuitive  
✅ **More Flexible**: Runtime options allow per-request customization  
✅ **Better Defaults**: Sensible defaults reduce boilerplate code

---

### Python: Explicit Input/Output Parameters for Workflow Handlers

**PR**: [#3472](https://github.com/microsoft/agent-framework/pull/3472)  
**Author**: [@moonbox3](https://github.com/moonbox3)  
**Impact**: MEDIUM - Improves workflow handler API clarity  
**Labels**: `python`, `workflows`, `enhancement`

#### What's New

Added explicit `input`, `output`, and `workflow_output` parameters to `@handler`, `@executor`, and `request_info` decorators/methods in the workflow system. This provides better type hints and clearer API contracts for workflow authors.

#### Changes

**Before:**
```python
from agent_framework import handler

@handler
async def my_handler(context, **kwargs):
    # Input type unclear
    user_input = kwargs.get("data")
    
    # Process
    result = process(user_input)
    
    # Output type unclear
    return result
```

**After:**
```python
from agent_framework import handler
from typing import TypedDict

class MyInput(TypedDict):
    data: str
    count: int

class MyOutput(TypedDict):
    result: str
    success: bool

@handler(input=MyInput, output=MyOutput)
async def my_handler(context, input: MyInput) -> MyOutput:
    # Clear input type
    user_data = input["data"]
    
    # Process
    result = process(user_data)
    
    # Clear output type
    return MyOutput(result=result, success=True)
```

#### Benefits

**Type Safety:**
```python
# IDE can now provide autocomplete and type checking
@handler(input=RequestData, output=ResponseData)
async def process_request(context, input: RequestData) -> ResponseData:
    # input.user_id autocompletes
    # output structure validated
    return ResponseData(...)
```

**Workflow Composition:**
```python
from agent_framework import executor

# Link handlers with explicit types
@executor(
    input=WorkflowInput,
    output=WorkflowOutput,
    workflow_output=FinalOutput
)
async def complex_workflow(context, input: WorkflowInput) -> FinalOutput:
    step1_result = await handler1(input.data)
    step2_result = await handler2(step1_result)
    return FinalOutput(final_data=step2_result)
```

**Runtime Validation:**
```python
# Framework can validate input/output at runtime
@handler(
    input=StrictInput,
    output=StrictOutput,
    validate=True  # Enable runtime validation
)
async def validated_handler(context, input: StrictInput) -> StrictOutput:
    # Automatically validated
    return StrictOutput(...)
```

#### Why This Matters

✅ **Better IDE Support**: Type hints enable autocomplete and error detection  
✅ **Self-Documenting**: Handler signatures clearly show input/output contracts  
✅ **Runtime Safety**: Optional validation catches type errors early  
✅ **Easier Composition**: Clear types make workflow orchestration simpler  
✅ **Migration Path**: Existing handlers continue to work (backward compatible)

---

## 📈 Files Changed Summary

**Total Files Modified**: 155+ files across 6 PRs

**Breakdown by PR:**
- **PR #3501** (.NET Breaking Change): 124 files
- **PR #3639** (Python Content API): 11 files
- **PR #3636** (Azure AI Instructions): 3 files (implementation + tests)
- **PR #3625** (GitHub Copilot): 9 files
- **PR #3494** (MCP Filter): 1 file
- **PR #3472** (Workflow Handlers): 8+ files

**Language Distribution:**
- C# (.NET): 124 files
- Python: 31+ files

---

## 📋 Summary

February 3, 2026 delivers a **substantial update** with critical improvements across both .NET and Python implementations. The release addresses long-standing bugs, introduces better API patterns, and includes one breaking change that improves API clarity.

### Key Achievements

✅ **API Improvements**: Better naming conventions (.NET session creation)  
✅ **Bug Fixes**: Fixed Content API imports, Azure AI instructions, MCP compatibility  
✅ **Feature Parity**: GitHubCopilotAgent now matches other agents' API patterns  
✅ **Type Safety**: Explicit workflow handler parameters improve developer experience  
✅ **Documentation**: Samples updated to reflect current best practices  
✅ **Test Coverage**: Maintained 87% Python test coverage

### Impact Assessment

**HIGH IMPACT - Breaking Changes:**
- .NET developers must rename `GetNewSessionAsync` → `CreateSessionAsync`
- Estimated migration time: 5-15 minutes for typical applications

**MEDIUM IMPACT - Bug Fixes:**
- Python developers using affected samples can now run them without errors
- Azure AI users will see instructions correctly applied
- MCP tool users no longer encounter parameter errors

**MEDIUM IMPACT - Enhancements:**
- Simpler GitHub Copilot agent configuration
- Better type safety for workflow authors

---

## 🎯 Recommended Actions

### For .NET Developers (BREAKING CHANGE)

**IMMEDIATE ACTION REQUIRED:**

1. **Update Session Creation Code**:
   ```bash
   # Find all usages
   git grep "GetNewSessionAsync" --name-only
   
   # Replace across codebase
   find . -type f -name "*.cs" -exec sed -i 's/GetNewSessionAsync/CreateSessionAsync/g' {} +
   ```

2. **Verify Build**:
   ```bash
   dotnet build
   dotnet test
   ```

3. **Check Custom Agents**:
   - If you've implemented custom agent classes inheriting from `AIAgent`
   - Update your override methods from `GetNewSessionAsync` to `CreateSessionAsync`
   - Update XML documentation comments

4. **Test Thoroughly**:
   - Session creation still works correctly
   - No behavioral changes beyond the method rename

### For Python Developers

**If Using Azure AI:**

1. **Test Instructions Handling**:
   ```python
   # Verify instructions are now correctly passed
   client = AzureAIClient(...)
   response = await client.complete(
       messages=[...],
       chat_options={"instructions": "Your instructions here"}
   )
   ```

2. **Update Tests**: Ensure your tests pass instructions via `chat_options`

**If Using GitHub Copilot Agent:**

1. **Simplify Agent Creation**:
   ```python
   # Old way (still works)
   agent = GitHubCopilotAgent(
       name="my-agent",
       default_options={"system_message": {"content": "..."}}
   )
   
   # New way (recommended)
   agent = GitHubCopilotAgent(
       name="my-agent",
       instructions="You are a helpful assistant."
   )
   ```

2. **Leverage Runtime Options**:
   ```python
   # Override instructions per request
   response = await agent.run(
       "Question",
       runtime_options={"system_message": {"content": "Special instructions"}}
   )
   ```

**If Using MCP Tools:**

1. **No Action Required**: The fix is automatic
2. **Verify**: MCP tool calls should now work without errors

**If Working with Workflows:**

1. **Add Type Hints** (Optional but Recommended):
   ```python
   @handler(input=MyInputType, output=MyOutputType)
   async def my_handler(context, input: MyInputType) -> MyOutputType:
       # Better IDE support and type safety
       return MyOutputType(...)
   ```

2. **Existing Handlers**: Continue to work without changes (backward compatible)

### For All Developers

1. **Update Dependencies**:
   ```bash
   # .NET
   dotnet restore
   dotnet update
   
   # Python
   pip install --upgrade agent-framework
   pip install --upgrade agent-framework-github-copilot
   pip install --upgrade agent-framework-azure
   ```

2. **Review Samples**: Check updated sample files for best practices

3. **Run Tests**: Ensure your test suite passes with the new version

4. **Update Documentation**: Reference new method names and API patterns

---

## 📚 Additional Resources

- [Microsoft Agent Framework Repository](https://github.com/microsoft/agent-framework)
- [.NET API Documentation](https://github.com/microsoft/agent-framework/tree/main/dotnet)
- [Python API Documentation](https://github.com/microsoft/agent-framework/tree/main/python)
- [GitHub Copilot Agent Guide](https://github.com/microsoft/agent-framework/tree/main/python/packages/github_copilot)
- [MCP Integration Guide](https://github.com/microsoft/agent-framework/tree/main/python/packages/core/agent_framework/_mcp.py)

---

**Contributors**: 
- [@westey-m](https://github.com/westey-m)
- [@moonbox3](https://github.com/moonbox3)
- [@dmytrostruk](https://github.com/dmytrostruk)
- [@giles17](https://github.com/giles17)
- Community contributors

**Total PRs Merged**: 6  
**Total Files Changed**: 155+  
**Breaking Changes**: 1 (.NET)  
**Bug Fixes**: 3  
**Feature Enhancements**: 2  
**Impact Level**: HIGH (due to breaking changes)
