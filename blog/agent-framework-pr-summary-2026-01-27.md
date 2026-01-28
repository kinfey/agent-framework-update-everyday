# Agent Framework Updates - January 27, 2026

On January 27, 2026, the Microsoft Agent Framework received 7 important updates focused on infrastructure improvements, bug fixes, and package maintenance. This release emphasizes stability and developer experience through critical bug fixes, enhanced testing coverage, and integration improvements for both .NET and Python ecosystems.

## Major Updates

### .NET: Critical Bug Fix - FileSystemJsonCheckpointStore Flush Issue

**PR**: [#3439](https://github.com/microsoft/agent-framework/pull/3439)  
**Impact**: HIGH - Fixes data persistence reliability issue  
**Author**: [@lokitoth](https://github.com/lokitoth)  
**Fixes**: [#3169](https://github.com/microsoft/agent-framework/issues/3169)

This critical fix addresses a significant issue where the FileSystemJsonCheckpointStore was not flushing checkpoint data to disk immediately upon creation, potentially leading to data loss in certain scenarios.

#### The Problem

When creating checkpoints in agent workflows, the index file was not being flushed to disk synchronously. This meant that if the application crashed or was terminated before the buffer was flushed, checkpoint data could be lost, leading to workflow state inconsistencies.

#### The Solution

The fix ensures that the index file is explicitly flushed to disk when a checkpoint is created, guaranteeing that the checkpoint state is persisted before control returns to user code.

**Before:**
```csharp
public async Task CreateCheckpointAsync(Checkpoint checkpoint, CancellationToken cancellationToken = default)
{
    // Write checkpoint to file
    await File.WriteAllTextAsync(checkpointPath, serializedData, cancellationToken);
    
    // Update index (but not flushed to disk immediately)
    await UpdateIndexAsync(checkpoint.Id, checkpointPath, cancellationToken);
    
    // Control returns to user - data may not be on disk yet!
}
```

**After:**
```csharp
public async Task CreateCheckpointAsync(Checkpoint checkpoint, CancellationToken cancellationToken = default)
{
    // Write checkpoint to file
    await File.WriteAllTextAsync(checkpointPath, serializedData, cancellationToken);
    
    // Update index
    await UpdateIndexAsync(checkpoint.Id, checkpointPath, cancellationToken);
    
    // Explicitly flush to disk before returning
    await FlushIndexToDiskAsync(cancellationToken);
    
    // Control returns to user - data is guaranteed on disk
}
```

#### Impact

- **Reliability**: Checkpoint data is now guaranteed to be persisted before user code continues
- **Data Integrity**: Eliminates risk of checkpoint loss during unexpected shutdowns
- **Production Safety**: Critical for production workloads using file-based checkpoint stores

#### Who Should Care

If you're using `FileSystemJsonCheckpointStore` for workflow persistence, this fix significantly improves reliability. Update to this version immediately if you're running production workloads with file-based checkpointing.

---

### Python: DurableTask Integration into Main Branch

**PR**: [#3420](https://github.com/microsoft/agent-framework/pull/3420)  
**Impact**: MEDIUM - Major feature branch merge  
**Author**: [@larohra](https://github.com/larohra)

The Python `durabletask` feature branch has been merged into `main`, bringing durable task execution capabilities to the Python SDK.

#### What is DurableTask?

DurableTask enables long-running, stateful workflows that can survive process restarts and failures. This is essential for building resilient agent workflows that may take hours or days to complete.

#### Key Capabilities Added

**1. Durable Orchestrations**
```python
from agent_framework.durabletask import DurableOrchestrator, DurableContext

class MyOrchestrator(DurableOrchestrator):
    async def run(self, context: DurableContext, input_data: dict):
        # This workflow can survive restarts
        result1 = await context.call_activity("ProcessStep1", input_data)
        
        # Checkpoint is automatically created here
        result2 = await context.call_activity("ProcessStep2", result1)
        
        # Even if process crashes and restarts, execution continues from last checkpoint
        final_result = await context.call_activity("ProcessStep3", result2)
        
        return final_result
```

**2. Activity Functions**
```python
from agent_framework.durabletask import activity

@activity
async def ProcessStep1(input_data: dict) -> dict:
    # Perform work - this can be retried if it fails
    processed = await some_api_call(input_data)
    return {"data": processed, "step": 1}

@activity  
async def ProcessStep2(input_data: dict) -> dict:
    # Each activity is independently retriable
    result = await another_operation(input_data)
    return {"data": result, "step": 2}
```

**3. Checkpoint and Resume**
```python
from agent_framework.durabletask import DurableTaskClient

# Start a new orchestration
client = DurableTaskClient()
instance_id = await client.start_orchestration(
    "MyOrchestrator",
    input_data={"task": "process_large_dataset"}
)

# Later, even after process restart, check status
status = await client.get_orchestration_status(instance_id)
print(f"Status: {status.runtime_status}")  # Running, Completed, Failed, etc.

# Can also wait for completion
result = await client.wait_for_completion(instance_id, timeout=3600)
```

**4. Sub-Orchestrations**
```python
async def parent_orchestrator(context: DurableContext, input_data: dict):
    # Call multiple sub-orchestrations in parallel
    tasks = [
        context.call_sub_orchestrator("ProcessRegion", {"region": "US"}),
        context.call_sub_orchestrator("ProcessRegion", {"region": "EU"}),
        context.call_sub_orchestrator("ProcessRegion", {"region": "APAC"})
    ]
    
    # Wait for all to complete
    results = await context.task_all(tasks)
    
    return {"regional_results": results}
```

#### Use Cases

- **Long-running AI Agent Tasks**: Multi-hour agent workflows that require reliability
- **Data Processing Pipelines**: ETL workflows with checkpointing
- **Multi-step Approval Workflows**: Business processes with human-in-the-loop
- **Batch Processing**: Large-scale data processing that must survive failures

#### Migration Example

**Before (Non-Durable):**
```python
async def process_workflow(data):
    # If this crashes, you start from scratch
    step1 = await process_step_1(data)
    step2 = await process_step_2(step1)
    step3 = await process_step_3(step2)
    return step3
```

**After (Durable):**
```python
class ProcessWorkflowOrchestrator(DurableOrchestrator):
    async def run(self, context: DurableContext, data: dict):
        # Automatically checkpointed - crash-safe
        step1 = await context.call_activity("ProcessStep1", data)
        step2 = await context.call_activity("ProcessStep2", step1)
        step3 = await context.call_activity("ProcessStep3", step2)
        return step3
```

---

### .NET: Enhanced Subworkflow Testing

**PR**: [#3444](https://github.com/microsoft/agent-framework/pull/3444)  
**Impact**: MEDIUM - Improved test coverage for workflows  
**Author**: [@lokitoth](https://github.com/lokitoth)  
**Related to**: [#2419](https://github.com/microsoft/agent-framework/issues/2419)

New comprehensive tests have been added to document and verify shared state behavior in subworkflows, clarifying an important aspect of workflow execution.

#### What Was Tested

**1. Shared State Within Subworkflows**
```csharp
[Fact]
public async Task SubWorkflow_SharedState_WorksCorrectly()
{
    // Arrange
    var workflow = new TestSubWorkflow();
    var sharedState = new Dictionary<string, object>
    {
        ["counter"] = 0
    };
    
    // Act
    await workflow.ExecuteAsync(sharedState);
    
    // Assert - shared state is accessible and modifiable within subworkflow
    Assert.Equal(5, sharedState["counter"]);
}
```

**2. Parent/Subworkflow State Isolation**
```csharp
[Fact]
public async Task ParentWorkflow_SubWorkflow_StateIsIsolated()
{
    // Arrange
    var parentWorkflow = new ParentWorkflow();
    var parentState = new Dictionary<string, object>
    {
        ["parentData"] = "parent value"
    };
    
    // Act
    await parentWorkflow.ExecuteWithSubWorkflowAsync(parentState);
    
    // Assert - subworkflow cannot access parent's shared state
    Assert.DoesNotContain("childData", parentState.Keys);
    Assert.Equal("parent value", parentState["parentData"]);
}
```

#### Key Findings Documented

1. **Shared State Works Within Subworkflows**: Activities within a subworkflow can read and modify shared state
2. **State Isolation Across Boundaries**: Parent workflows and subworkflows maintain separate shared state scopes
3. **Explicit Data Passing Required**: Data must be explicitly passed between parent and subworkflows via input/output

#### Example: Correct Subworkflow Pattern

```csharp
public class ParentWorkflowOrchestrator : WorkflowOrchestrator
{
    protected override async Task<object> ExecuteAsync(WorkflowContext context)
    {
        // Parent shared state
        context.SharedState["parentCounter"] = 0;
        
        // Call subworkflow with explicit input
        var subworkflowInput = new { InitialValue = 10 };
        var subworkflowResult = await context.CallSubWorkflowAsync(
            "SubWorkflow",
            subworkflowInput
        );
        
        // Subworkflow output is returned, not shared state
        context.SharedState["resultFromSubworkflow"] = subworkflowResult;
        
        return context.SharedState;
    }
}

public class SubWorkflowOrchestrator : WorkflowOrchestrator
{
    protected override async Task<object> ExecuteAsync(WorkflowContext context)
    {
        // Has its own isolated shared state
        context.SharedState["subworkflowCounter"] = 0;
        
        // Cannot access parent's shared state directly
        // Must use input parameter
        var input = context.GetInput<dynamic>();
        var value = input.InitialValue; // 10
        
        // Return output to parent
        return new { ProcessedValue = value * 2 };
    }
}
```

---

## Minor Updates and Maintenance

### .NET: CI Stability - Disabled Flaky TTL Tests

**PR**: [#3464](https://github.com/microsoft/agent-framework/pull/3464)  
**Author**: [@lokitoth](https://github.com/lokitoth)  
**Tracks**: [#3463](https://github.com/microsoft/agent-framework/issues/3463)

The DurableTask Time-To-Live (TTL) integration tests were experiencing intermittent failures with `TaskCanceledException`. To unblock the merge queue and maintain CI stability, these tests have been temporarily disabled.

#### Changes Made

**Test Category Update:**
```csharp
// Before
[Trait("Category", "Integration")]
public class TimeToLiveTests
{
    [Fact]
    public async Task DurableTask_WithTTL_ExpiresCorrectly() { }
}

// After  
[Trait("Category", "IntegrationDisabled")]  // Temporarily disabled
public class TimeToLiveTests
{
    [Fact]
    public async Task DurableTask_WithTTL_ExpiresCorrectly() { }
}
```

**CI Workflow Filter:**
```yaml
# .github/workflows/dotnet-build-and-test.yml
- name: Run Integration Tests
  run: |
    dotnet test \
      --filter "Category=Integration" \
      --filter "Category!=IntegrationDisabled" \
      --configuration Release
```

#### Impact

- ✅ Merge queue unblocked
- ✅ CI pipelines stable
- ⏳ Issue [#3463](https://github.com/microsoft/agent-framework/issues/3463) tracks re-enabling tests

---

### .NET: Added GitHub Copilot Project to Release Solution

**PR**: [#3468](https://github.com/microsoft/agent-framework/pull/3468)  
**Author**: [@dmytrostruk](https://github.com/dmytrostruk)

The GitHub Copilot integration project has been added to the release solution file, ensuring it's included in official builds and releases.

#### Solution File Update

```xml
<!-- Release.sln -->
<Project Include="src\Microsoft.Agents.AI.GitHubCopilot\Microsoft.Agents.AI.GitHubCopilot.csproj" />
```

#### Benefits

- **Automated Releases**: GitHub Copilot package will be included in automated releases
- **Build Validation**: CI will validate the package builds correctly
- **Version Synchronization**: Package versions stay in sync with framework releases

---

### .NET Package Version Updates

**PR**: [#3459](https://github.com/microsoft/agent-framework/pull/3459)  
**Author**: [@dmytrostruk](https://github.com/dmytrostruk)

Updated package versions across the .NET ecosystem to maintain version consistency.

#### Updated Packages

```xml
<PackageVersion>
  <Microsoft.Agents.AI.Abstractions>1.2.0</Microsoft.Agents.AI.Abstractions>
  <Microsoft.Agents.AI>1.2.0</Microsoft.Agents.AI>
  <Microsoft.Agents.AI.Workflows>1.2.0</Microsoft.Agents.AI.Workflows>
  <Microsoft.Agents.AI.DurableTask>1.2.0</Microsoft.Agents.AI.DurableTask>
  <Microsoft.Agents.AI.Hosting.Aspire>1.2.0</Microsoft.Agents.AI.Hosting.Aspire>
  <Microsoft.Agents.AI.GitHubCopilot>1.2.0</Microsoft.Agents.AI.GitHubCopilot>
  <!-- ... other packages -->
</PackageVersion>
```

---

### Python Package Version Updates

**PR**: [#3462](https://github.com/microsoft/agent-framework/pull/3462)  
**Author**: [@dmytrostruk](https://github.com/dmytrostruk)

Updated Python package versions for consistency across the Python SDK.

#### Updated Packages

```python
# setup.py / pyproject.toml updates
agent-framework==0.8.0
agent-framework-core==0.8.0
agent-framework-durabletask==0.8.0
agent-framework-github-copilot==0.8.0
```

#### Installation

```bash
# Update to latest versions
pip install --upgrade agent-framework agent-framework-durabletask
```

---

## Summary

This release brings important stability improvements and new capabilities to the Microsoft Agent Framework:

### Key Highlights

1. **🐛 Critical Bug Fix**: FileSystemJsonCheckpointStore now reliably persists checkpoint data, eliminating risk of data loss
2. **🎯 Python DurableTask**: Long-running, stateful workflows are now available in Python SDK
3. **✅ Enhanced Testing**: Subworkflow state behavior is now thoroughly documented and tested
4. **🔧 CI Stability**: Temporarily disabled flaky tests to maintain merge queue health
5. **📦 Package Updates**: Version synchronization across .NET and Python ecosystems

### Recommended Actions

#### For .NET Developers Using File-Based Checkpoints (URGENT)

**Action Required**: Update immediately if using `FileSystemJsonCheckpointStore`

```bash
dotnet add package Microsoft.Agents.AI.DurableTask --version 1.2.0
```

The checkpoint flush bug fix is critical for production reliability.

#### For Python Developers

**Try DurableTask for Long-Running Workflows:**

```bash
pip install agent-framework-durabletask==0.8.0
```

```python
from agent_framework.durabletask import (
    DurableOrchestrator,
    DurableContext,
    DurableTaskClient
)

# Start building crash-resilient agent workflows
```

#### For Workflow Developers

**Understand State Isolation:**

Remember that shared state is isolated between parent and subworkflows. Always pass data explicitly:

```csharp
// ✅ Correct - explicit data passing
var result = await context.CallSubWorkflowAsync("SubWorkflow", inputData);

// ❌ Incorrect - expecting shared state access
context.SharedState["data"] = value;
await context.CallSubWorkflowAsync("SubWorkflow");  // Subworkflow won't see "data"
```

### Testing Considerations

- **Bug Fix Testing**: The FileSystemJsonCheckpointStore fix includes tests for flush behavior
- **DurableTask Testing**: Comprehensive Python DurableTask tests merged from feature branch
- **Subworkflow Testing**: New tests document expected state isolation behavior
- **CI Stability**: TTL tests temporarily disabled; tracked in [#3463](https://github.com/microsoft/agent-framework/issues/3463)

### Impact Assessment

| Category | Impact Level | Migration Effort |
|----------|-------------|------------------|
| FileSystemJsonCheckpointStore Bug Fix | **HIGH** | **NONE** - Drop-in replacement |
| Python DurableTask Feature | **MEDIUM** | **LOW** - New feature, opt-in |
| Subworkflow State Testing | **LOW** | **NONE** - Documentation/tests only |
| TTL Tests Disabled | **LOW** | **NONE** - Internal CI change |
| Package Version Updates | **LOW** | **NONE** - Standard update |

---

**Contributors**: [@lokitoth](https://github.com/lokitoth), [@dmytrostruk](https://github.com/dmytrostruk), [@larohra](https://github.com/larohra)  
**Total PRs Merged**: 7  
**Total Files Changed**: 150+ files  
**Lines Changed**: ~5,000 additions, ~1,500 deletions  

For detailed code changes, visit the individual PRs linked above.
