---
title: "实战避坑｜TestPlanExecution 的 PrePlanRun 隐藏开销与优化"
date: 2026-03-31 02:00:00
tags:
  - OpenTAP
---

OpenTAP 在执行测试计划前，会递归调用所有启用步骤的 `PrePlanRun()` 方法。官方文档只提到“做预准备”，却鲜少提及：当计划含上千步骤且嵌套层级深时，这一阶段可能成为启动瓶颈。今天从源码角度拆解其执行链路，并给出可落地的提速方案。

<!-- more -->

## 背景：启动慢，别只怪仪器连接

不少开发者把“测试启动卡几秒”归咎于仪器 `Open()`，却忽视 PrePlanRun 的串行递归。引擎主函数 `ExecTestPlan` 在真正跑步骤前，会：
1. 调用 `RunPrePlanRunMethods` 递归遍历所有启用节点；
2. 对每一步包裹 `ResourceManager.BeginStep/EndStep`；
3. 默认串行，且单线程内完成。

若单步 PrePlanRun 平均 5 ms，1000 步就是 5 s，且无法通过并行仪器连接隐藏。

## 框架分析：递归与资源令牌

核心代码在 `Engine/TestPlanExecution.cs`：

```csharp
static bool RunPrePlanRunMethods(IList<ITestStep> steps, TestPlanRun planRun)
{
    foreach (ITestStep step in steps)
    {
        if (step.Enabled == false) continue;
        bool runPre = true;
        if(step is TestStep s)
            runPre = s.PrePostPlanRunUsed;      // 关键开关
        if (runPre)
        {
            planRun.ResourceManager.BeginStep(..., TestPlanExecutionStage.PrePlanRun, ...);
            step.PrePlanRun();                  // 用户代码在此
            planRun.ResourceManager.EndStep(...);
        }
        if (!RunPrePlanRunMethods(step.ChildTestSteps, planRun))
            return false;                       // 深度优先
    }
}
```

- `PrePostPlanRunUsed` 是 `TestStep` 的 internal bool，默认 true；
- `BeginStep/EndStep` 会拿资源锁，即使步骤本身并不访问仪器；
- 递归返回 false 即短路，任何异常都会中止整个计划。

## 实现过程：把“无实际预操作”的步骤标记为跳过

若自定义步骤无需 PrePlanRun，可在构造函数关闭标记：

```csharp
[DisplayName("MyFastStep")]
public class MyFastStep : TestStep
{
    public MyFastStep()
    {
        // 关闭 Pre/Post 钩子，引擎将直接跳过
        PrePostPlanRunUsed = false;
    }

    public override void Run()
    {
        // 真正逻辑
    }
}
```

对于无状态、不占用硬件的步骤（计算、判读、延迟等），此举可把耗时从 O(n) 降到 0。

## 注意事项

1. 仅当步骤确定不依赖 `PrePlanRun()` 内初始化资源、不抛出异常时再关闭；
2. 关闭后 `PlanRun` 属性在 `PrePlanRun` 阶段不再注入，若代码后期重构依赖该属性会 NPE；
3. 官方内置的 `DelayStep`、`RepeatStep` 已默认关闭，可作为参考；
4. 若步骤需要动态启用/禁用 PrePlanRun，可在 `OnEnabledChanged` 里同步修改 `PrePostPlanRunUsed`。

## 小结

PrePlanRun 的设计让“准备动作”有了统一钩子，却也带来串行开销。通过合理使用 `PrePostPlanRunUsed = false`，可把启动时间从秒级降到毫秒级，尤其对大批量无状态步骤场景效果显著。下次遇到“计划启动慢”，先 profile 一下 PrePlanRun，再决定要不要并行开仪器，别一味加线程。

## 可复现验证

```bash
# 创建 500 个空步骤并测量启动耗时
tap run --verbose --profile PrePlanRun.json preplan-demo.TapPlan
# 观察日志 "PrePlanRun Methods completed" 时间戳
# 修改步骤源码关闭 PrePostPlanRunUsed 后重新运行，对比耗时
```

## 关键源码路径

- `Engine/TestPlanExecution.cs`：RunPrePlanRunMethods、ExecTestPlan
- `Engine/TestStep.cs`：PrePostPlanRunUsed 属性定义
- `Engine/ResourceManager.cs`：BeginStep/EndStep 资源锁实现