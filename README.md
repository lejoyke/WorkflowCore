# WorkflowCore - 轻量级工作流引擎

![Version](https://img.shields.io/badge/version-1.0.6-blue.svg)
![.NET](https://img.shields.io/badge/.NET-8.0-purple.svg)
![License](https://img.shields.io/badge/license-MIT-green.svg)

## 📖 项目简介

本项目是基于 [Workflow Core](https://workflow-core.readthedocs.io/en/latest/) 的简化版本，专为**工业控制桌面端应用**优化设计。

**⚠️ 重要提示：** 本文档仅说明与原项目的差异部分。基础使用方法请参考：
- **原项目 GitHub：** https://github.com/danielgerlag/workflow-core  
- **官方文档：** https://workflow-core.readthedocs.io/en/latest/

### 技术规格
- **目标框架：** .NET 8.0
- **当前版本：** 1.0.6
- **应用场景：** 工业控制、设备状态机、自动化流程

---

## 🔄 与原项目的主要区别

### ❌ 简化的功能

为了简化在单机桌面应用中的使用，本版本移除了以下功能：

| 功能 | 原项目 | 简化版 |
|------|--------|--------|
| 多节点集群 | ✅ 支持 | ❌ 已移除 |
| 分布式持久化 | ✅ Redis、MongoDB、SQL Server | ❌ 仅内存持久化 |
| 外部队列 | ✅ RabbitMQ、Azure Service Bus | ❌ 单节点队列 |

### ✨ 新增功能

以下是相对于原项目新增或增强的功能：

#### 1. ExecutionResult 增强方法 - 步骤内流程控制

**功能描述：**  
在步骤内部可以通过 `ExecutionResult` 的增强方法直接控制整个工作流的状态，无需在外部管理。

**新增的便捷方法：**

```csharp
// 1. 重试当前步骤
public static ExecutionResult Retry(IStepExecutionContext context)

// 2. 暂停工作流
public static ExecutionResult Suspend(SuspendMode suspendMode = SuspendMode.Immediate)

// 3. 终止工作流
public static ExecutionResult Terminate()

// 4. 标记工作流完成
public static ExecutionResult Complete()
```

**暂停模式：**
```csharp
public enum SuspendMode
{
    Immediate,                 // 立即暂停，重启后继续当前步骤
    WaitCurrentStepComplete    // 等待当前步骤完成后暂停，重启后执行下一步骤
}
```

**使用示例：**

```csharp
public class DeviceCheckStep : StepBody
{
    public override ExecutionResult Run(IStepExecutionContext context)
    {
        var data = context.GetData<DeviceData>();
        
        // 检查设备状态
        if (data.ErrorCount > 3)
        {
            // 错误过多，终止工作流
            return ExecutionResult.Terminate();
        }
        
        if (data.NeedRecalibrate)
        {
            // 需要重新校准，暂停等待人工干预
            return ExecutionResult.Suspend(SuspendMode.Immediate);
        }
        
        if (!data.IsDeviceReady)
        {
            // 设备未就绪，重试当前步骤
            return ExecutionResult.Retry(context);
        }
        
        if (data.TaskCompleted)
        {
            // 任务提前完成，标记工作流完成
            return ExecutionResult.Complete();
        }
        
        // 继续下一步
        return ExecutionResult.Next();
    }
}
```

**与原项目的区别：**
- 原项目需要在外部调用 `IWorkflowHost` 的方法来控制工作流状态
- 本版本支持在步骤内部直接返回 `ExecutionResult` 来控制工作流，更加灵活和便捷

---

#### 2. Scope 作用域控制机制

**功能描述：**  
`ExecutionPointer` 中新增的 `Scope` 属性用于跟踪步骤的执行作用域，记录步骤的父级执行链。

**属性定义：**
```csharp
public class ExecutionPointer
{
    /// <summary>
    /// 作用域，定义步骤的执行作用域（其主分支）
    /// </summary>
    public IReadOnlyCollection<string> Scope { get; set; }
}
```

**工作原理：**
- 当创建子执行指针时，父指针的 ID 会被添加到子指针的 Scope 中
- JumpTo 步骤利用 Scope 信息清理所有相关的子分支
- 支持嵌套的流程控制和作用域管理

---

#### 3. JumpTo 步骤 - 流程跳转控制

**功能描述：**  
`JumpTo` 允许在运行时跳转到指定的步骤，并自动清理当前作用域内的所有子执行指针。

**核心实现：**
```csharp
public sealed class JumpTo : StepBody
{
    public override ExecutionResult Run(IStepExecutionContext context)
    {
        // 遍历所有执行指针，清理当前作用域内的指针
        foreach (var item in context.Workflow.ExecutionPointers)
        {
            if (context.ExecutionPointer.Scope.Contains(item.Id))
            {
                item.Active = false;
                item.Status = PointerStatus.Complete;
                item.EndTime = DateTime.Now;
            }
        }
        
        context.ExecutionPointer.Scope = [];
        return ExecutionResult.Next();
    }
}
```

**使用方式：**
```csharp
builder
    .StartWith<Step1>()
    .Then<Step2>()
        .Name("LoopStart")  // 设置外部ID作为跳转目标
    .Then<Step3>()
    .If(data => data.ShouldContinue)
        .Do(then => then
            .JumpTo("LoopStart")  // 跳转到 Step2
        );
```

**应用场景：**
- 在复杂的分支流程中实现跨分支跳转
- 异常处理后返回到主流程
- 实现流程的循环和重试机制

---

#### 4. 单实例模式扩展方法

**功能描述：**  
为工业控制场景提供的便捷扩展方法，简化单实例工作流的管理。

**扩展方法列表：**

```csharp
// 启动或恢复单实例工作流（如果不存在则启动，如果已暂停则恢复）
await workflowHost.StartOrResumeWorkflowInSingleMode(workflowName, data);

// 暂停单实例工作流
await workflowHost.SuspendWorkflowInSingleMode(workflowName);

// 终止单实例工作流
await workflowHost.TerminateWorkflowInSingleMode(workflowName);

// 按定义ID查找工作流
var workflows = await workflowHost.FindWorkflowByDefinitionId(workflowName);

// 检查是否存在可运行的工作流
bool exists = await workflowHost.ExistsRunnableWorkflowByName(workflowName);
```

**使用场景：**
- 设备控制（一个设备只能有一个控制流程）
- 生产线管理（一条生产线只能有一个运行流程）
- 避免多实例冲突

---

## 🚀 快速开始

### 基础使用

关于工作流的基础使用（安装、配置、定义步骤等），请参考：
- **官方文档：** https://workflow-core.readthedocs.io/en/latest/
- **入门教程：** https://workflow-core.readthedocs.io/en/latest/getting-started/

### 配置服务

```csharp
using Microsoft.Extensions.DependencyInjection;

builder.Services.AddWorkflow(options =>
{
    options.UsePollInterval(TimeSpan.FromSeconds(3));
    options.UseMaxConcurrentWorkflows(5);
});
```

---

## 🧩 新增功能详解

### ExecutionResult 步骤内流程控制详解

**Scope 作用域机制：**

作用域用于跟踪执行指针的层级关系，支持 JumpTo 精确清理子分支：

```
主流程 (Pointer A) → Scope = []
  ├─ If 分支 (Pointer B) → Scope = [A]
  │   └─ 子步骤 (Pointer C) → Scope = [A, B]
  └─ 下一步 (Pointer D) → Scope = [A]
```

当 Pointer B 执行 JumpTo 时，所有 Scope 中包含 B 的指针都会被清理。

---

## 📚 新增 API 参考

### 单实例模式扩展方法

```csharp
// 启动或恢复单实例工作流
await workflowHost.StartOrResumeWorkflowInSingleMode(workflowName, data);

// 暂停单实例工作流
await workflowHost.SuspendWorkflowInSingleMode(workflowName);

// 终止单实例工作流
await workflowHost.TerminateWorkflowInSingleMode(workflowName);

// 查找工作流实例
var workflows = await workflowHost.FindWorkflowByDefinitionId(workflowName);

// 检查是否存在可运行的工作流
bool isRunning = await workflowHost.ExistsRunnableWorkflowByName(workflowName);
```

### JumpTo 步骤使用

```csharp
builder
    .StartWith<Step1>()
    .Then<Step2>()
        .Name("LoopStart")  // 设置跳转目标
    .Then<Step3>()
    .If(data => data.ShouldRetry)
        .Do(then => then
            .JumpTo("LoopStart")  // 跳转
        );
```

### ExecutionResult 流程控制

```csharp
// 在步骤内部控制工作流
public override ExecutionResult Run(IStepExecutionContext context)
{
    return ExecutionResult.Retry(context);    // 重试当前步骤
    return ExecutionResult.Suspend();          // 暂停工作流
    return ExecutionResult.Terminate();        // 终止工作流
    return ExecutionResult.Complete();         // 完成工作流
}
```

更多 API 使用说明，请参考[原项目文档](https://workflow-core.readthedocs.io/en/latest/)。

---

## 💡 使用场景示例

### 完整示例：设备控制流程（展示所有新增功能）

```csharp
public class DeviceControlData
{
    public string DeviceId { get; set; }
    public int ErrorCount { get; set; }
    public bool EmergencyStop { get; set; }
    public bool NeedMaintenance { get; set; }
}

// 设备运行步骤 - 演示 ExecutionResult 流程控制
public class DeviceOperationStep : StepBody
{
    public override ExecutionResult Run(IStepExecutionContext context)
    {
        var data = context.GetData<DeviceControlData>();
        
        // 紧急停止 - 立即终止整个工作流
        if (data.EmergencyStop)
            return ExecutionResult.Terminate();
        
        // 需要维护 - 暂停工作流等待人工干预
        if (data.NeedMaintenance)
            return ExecutionResult.Suspend(SuspendMode.Immediate);
        
        // 错误过多 - 重试当前步骤
        if (data.ErrorCount > 0 && data.ErrorCount < 3)
            return ExecutionResult.Retry(context);
        
        // 正常执行
        Console.WriteLine("设备正常运行中");
        return ExecutionResult.Next();
    }
}

// 工作流定义 - 演示 JumpTo 和 Scope
public class DeviceControlWorkflow : IWorkflow<DeviceControlData>
{
    public string Id => "DeviceControl";
    public int Version => 1;
    
    public void Build(IWorkflowBuilder<DeviceControlData> builder)
    {
        builder
            .StartWith(context => 
            {
                Console.WriteLine("设备初始化");
                return ExecutionResult.Next();
            })
            .Name("Initialize")
            
            // 主循环
            .Then<DeviceOperationStep>()
            .Name("MainLoop")
            
            // 错误处理 - 使用 JumpTo 跳转
            .If(data => data.ErrorCount >= 3)
                .Do(then => then
                    .StartWith(context =>
                    {
                        Console.WriteLine("错误过多，重新初始化");
                        var data = context.GetData<DeviceControlData>();
                        data.ErrorCount = 0;
                        return ExecutionResult.Next();
                    })
                    .JumpTo("Initialize")  // 跳转回初始化
                )
            
            // 继续循环
            .Delay(data => TimeSpan.FromMilliseconds(100))
            .JumpTo("MainLoop");  // 跳转回主循环
    }
}

// 使用单实例模式管理工作流
public class DeviceController
{
    private readonly IWorkflowHost _workflowHost;
    
    public async Task StartDevice(string deviceId)
    {
        var data = new DeviceControlData { DeviceId = deviceId };
        
        // 启动或恢复单实例工作流
        await _workflowHost.StartOrResumeWorkflowInSingleMode(
            "DeviceControl", data);
    }
    
    public async Task StopDevice()
    {
        // 暂停单实例工作流
        await _workflowHost.SuspendWorkflowInSingleMode("DeviceControl");
    }
    
    public async Task EmergencyStop()
    {
        // 终止单实例工作流
        await _workflowHost.TerminateWorkflowInSingleMode("DeviceControl");
    }
    
    public async Task<bool> IsDeviceRunning()
    {
        // 检查工作流是否运行中
        return await _workflowHost.ExistsRunnableWorkflowByName("DeviceControl");
    }
}
```

**此示例展示了所有新增功能：**
1. ✅ **ExecutionResult 流程控制** - 步骤内终止、暂停、重试
2. ✅ **JumpTo 跳转** - 循环和错误恢复
3. ✅ **Scope 作用域** - 自动清理子分支
4. ✅ **单实例扩展方法** - 简化工作流管理

更多使用场景和模式，请参考[原项目文档](https://workflow-core.readthedocs.io/en/latest/)。

---

## 🎯 适用场景

### ✅ 适合使用简化版的场景
- 工业控制桌面应用
- 单机设备控制系统
- 设备状态机管理
- 自动化测试流程

### ❌ 不适合的场景
- 需要多节点分布式协调
- 需要外部持久化的长期流程
- 需要高可用性的关键业务

**👉 对于分布式场景，请使用 [原版 WorkflowCore](https://github.com/danielgerlag/workflow-core)。**

---

## 📝 许可证

本项目基于原 [Workflow Core](https://github.com/danielgerlag/workflow-core) 项目修改而来。

**原项目：**
- 作者：Daniel Gerlag
- 许可证：MIT License
- 仓库：https://github.com/danielgerlag/workflow-core

**致谢：**  
感谢 Daniel Gerlag 和所有 Workflow Core 贡献者提供了优秀的工作流引擎基础框架。

---

## 🤝 贡献

欢迎提交 Issue 和 Pull Request 来改进本项目！

如果你在使用过程中遇到问题或有新的功能建议，请通过以下方式联系：

1. 提交 GitHub Issue
2. 发送 Pull Request
3. 通过邮件联系维护者

---

## 📞 支持

如果您需要帮助或有任何疑问，请：

1. 查看[原项目文档](https://workflow-core.readthedocs.io/)了解基础概念
2. 查看本 README 中的示例代码
3. 提交 GitHub Issue 获取支持

---

## 🔄 版本历史

**v1.0.6** (当前版本)
- ✨ ExecutionResult 增强方法（Retry, Suspend, Terminate, Complete）
- ✨ JumpTo 步骤支持流程跳转
- ✨ Scope 作用域控制机制
- ✨ 单实例模式扩展方法
- 🔧 移除多节点分布式支持

---

## 📚 参考资源

- [原项目官方文档](https://workflow-core.readthedocs.io/en/latest/)
- [原项目 GitHub](https://github.com/danielgerlag/workflow-core)
- [.NET 8.0 文档](https://docs.microsoft.com/dotnet/)

---

<div align="center">

**⭐ 如果这个项目对你有帮助，请给它一个星标！**

Made with ❤️ for Industrial Automation

</div>

