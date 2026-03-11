---
title: "机制图解｜OpenTAP TestPlan 的 Open/Execute 状态复用"
slug: opentap-open-execute-state-reuse
date: 2026-03-11 02:00:00
tags:
  - OpenTAP
  - Engine
  - TestPlanExecution
  - Resource
---

## 背景
在长期跑产线或回归任务时，测试计划经常被重复执行。如果每次都完整打开/关闭资源，耗时会被仪器连接、监听器初始化放大。OpenTAP 在 `TestPlanExecution` 里提供了一个容易被忽略的机制：先 `Open()` 进入“已打开”状态，再多次 `Execute()` 复用上下文，最后统一 `Close()`。这不是简单缓存，而是带有执行阶段标记和资源生命周期约束的一套状态流转。

## 框架分析
核心状态由 `currentExecutionState` 持有。未预打开时，`DoExecute()` 会新建 `TestPlanRun`，并走 `OpenInternal(..., isOpen=false, ...)` 完整启动。已预打开时，执行分支会把 `continuedExecutionState=true`，基于已有运行态构造新的执行阶段对象，避免重复打开资源。

另一方面，`Open()` 只负责把资源推进到 `TestPlanExecutionStage.Open`，而 `Execute()` 再进入 `Execute` 阶段；`Close()` 对应 `EndStep(...Open)` 收尾。也就是说，Open/Execute/Close 并不是三个独立 API，而是共享同一资源管理状态机。

## 实现过程
实操建议是把“高成本连接”前移到 `Open()`，把“高频执行”放进循环，最后统一关闭：

```csharp
// 伪代码：强调调用时序
var plan = TestPlan.Load("demo.TapPlan");
plan.Open();                 // 资源只打开一次
for (int i = 0; i < 10; i++)
{
    var run = plan.Execute(); // 复用已打开资源
    Console.WriteLine(run.Verdict);
}
plan.Close();                // 统一释放
```

可在源码仓库直接复现关键分支定位：

```bash
cd /home/ops/clawd/repos/opentap
grep -n "currentExecutionState\|continuedExecutionState\|OpenInternal\|public void Open()\|public void Close()" Engine/TestPlanExecution.cs
```

## 注意事项
1. `IsRunning` 为真时调用 `Open()` 或 `Close()` 会抛异常，不能和运行线程抢状态。
2. `Open()` 失败会回滚 `currentExecutionState`，这是为“修完配置再重开”准备的保护逻辑。
3. 复用打开态执行时，元数据补充路径与冷启动不同，代码里有专门分支处理。
4. 若开启摘要监听，监听器会被并入结果链路，调试耗时时要把这部分算进去。

## 小结
OpenTAP 的状态复用设计本质上是在“资源生命周期”与“执行生命周期”之间做解耦：资源可长驻，执行可高频。理解 `currentExecutionState` 与 `continuedExecutionState` 这两个钩子后，很多“为什么第二次跑更快/为什么 close 后才能彻底释放”的问题都会变得清晰。

**关键源码路径：**
- `Engine/TestPlanExecution.cs`
- `Engine/TestPlanRun.cs`
- `Engine/TestStepRun.cs`
