---
title: OpenTAP Verdict 传递机制：从 Step 到 TestPlan 的状态收敛
abbrlink: 2026-02-28-opentap-verdict-propagation
date: 2026-02-28 02:00:00
tags:
  - OpenTAP
  - Engine
  - Verdict
categories:
  - OpenTAP
---

## 背景

做自动化测试平台时，最怕“结果看起来对，但过程不可信”：某个子步骤已经 Fail，最终报告却是 Pass，或者异常被吞掉后只显示 Inconclusive。OpenTAP 在 Engine 层对 Verdict 的处理很实用：把 Verdict 当成“单向升级”的状态机，不允许在执行过程中被降级。这样即使并发执行和异步收尾混在一起，最终结论仍然稳定。

## 框架分析

OpenTAP 的 Verdict 枚举本身已经体现了严重度顺序：`NotSet < Pass < Inconclusive < Fail < Aborted < Error`。核心点不是“谁写 Verdict”，而是“谁有权把状态往回改”。在实现上，`UpgradeVerdict()` 只在 `当前值 < 新值` 时才更新，并通过锁保证并发安全。于是整个执行链路可以拆成三层：

1. **Step 层**：单个 TestStep 在运行中根据断言、异常或 Break 条件升级自己的 Verdict。
2. **StepRun 层**：每个 TestStepRun 完成后，把本次运行结果固定下来，供父级汇总。
3. **PlanRun 层**：`TestPlanExecution` 在等待各 StepRun 完成后，逐个 `planRun.UpgradeVerdict(run.Verdict)`，得到整次计划执行的最终结论。

## 实现过程

下面这段命令可以直接复现“Verdict 传递路径”的源码定位：

```bash
cd /home/ops/clawd/repos/opentap
grep -RIn "UpgradeVerdict" Engine --include='*.cs'
grep -RIn "planRun.UpgradeVerdict(run.Verdict)" Engine/TestPlanExecution.cs
```

实际执行时，关键流程是：

- `TestStep.DoRun()` 初始化 `Step.Verdict = NotSet`；
- 子步骤结束后通过 `step.UpgradeVerdict(run.Verdict)` 向父步骤传播；
- `ExecTestPlan()` 的 finally 块里等待所有 `run.WaitForCompletion()` 后统一汇总到 `planRun`；
- 若执行中断或异常，`DoExecute()` 会把 `execStage` 升级为 `Aborted` 或 `Error`，避免出现“异常但仍 Pass”的假阳性。

## 注意事项

- **不要手动降级 Verdict**：一旦某层出现 Fail/Error，再写 Pass 只会制造不可解释结果。
- **并发步骤要等完成再汇总**：OpenTAP 在 finally 中统一 `WaitForCompletion()`，这是避免竞态误判的关键。
- **区分 Aborted 与 Error**：前者偏控制流中止，后者偏异常故障，后处理策略通常不同。
- **插件里少做“二次判定”**：ResultListener 更适合记录和扩展，不建议重写核心 Verdict 逻辑。

## 小结

OpenTAP 的 Verdict 机制看起来简单，工程价值却很高：它把“最终结果”变成可推导、可追踪、可并发的收敛过程。对测试平台来说，这比“多打印几行日志”更重要——因为结论一旦不稳定，自动化就失去可信度。

**关键源码路径**：
- `Engine/Verdict.cs`
- `Engine/TestStepRun.cs`
- `Engine/TestStep.cs`
- `Engine/TestPlanExecution.cs`
- `Engine/TestPlanRun.cs`
