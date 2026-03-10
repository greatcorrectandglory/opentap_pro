---
title: OpenTAP CLI 的中断处理与退出码链路
slug: opentap-cli-run-cancel-and-exitcode
date: 2026-03-10 02:00:00
tags:
  - OpenTAP
  - CLI
  - TestPlan
categories:
  - OpenTAP源码分析
---

## 背景
在 CI 或产线脚本里，`tap run` 是否“可控”比“能跑起来”更重要。可控体现在两件事：一是按下 Ctrl+C（或收到 SIGTERM）时要优雅停机；二是退出码必须稳定，让上层调度系统能准确判定是失败、参数错误，还是用户取消。OpenTAP 在 CLI 层把这条链路打通了：入口负责信号接入，执行器负责统一异常映射，`run` 命令再把测试结论映射到进程退出码。

## 框架分析
这一主题涉及四个文件，职责分层很清晰：
1. `Cli/TapEntry.cs`：CLI 主入口，初始化日志与插件搜索，然后进入命令执行。
2. `Engine/Cli/CliActionExecutor.cs`：统一处理命令解析、信号中断、异常捕获和返回码。
3. `Engine/Cli/RunCliAction.cs`：`tap run` 的业务执行，加载计划、处理外部参数、返回 Verdict 对应状态码。
4. `Engine/Cli/ExitCodes.cs`：保留退出码区间（192-255），定义通用错误语义。

设计重点是“业务码”和“框架码”分离：`ExitCodes` 提供公共语义（如参数错误、用户取消），`RunCliAction` 额外用 `ExitStatus` 表达测试结论（如 Fail/Error），避免所有失败都挤成同一个数字。

## 实现过程
可复现地看这条链路，先定位关键代码：

```bash
cd /home/ops/clawd/repos/opentap
grep -n "CancelKeyPress\|SIGTERM\|OperationCanceledException\|ExitCodes" \
  Engine/Cli/CliActionExecutor.cs Engine/Cli/RunCliAction.cs Engine/Cli/ExitCodes.cs Cli/TapEntry.cs
```

执行流程可概括为：
- `TapEntry.Go()` 先做 `PluginManager.Search()`，保证命令类型可被发现。
- `CliActionExecutor.Execute()` 注册 `Console.CancelKeyPress`，在非 Windows 还挂 `SIGINT/SIGTERM`；触发时调用 `TapThread.Current.AbortNoThrow()`。
- 执行具体 `ICliAction` 时，若抛出 `OperationCanceledException`，统一映射为 `UserCancelled(192)`。
- `RunCliAction.Execute()` 在计划运行后根据 `Verdict` 返回 20/30/50（Inconclusive/Fail/Error），参数问题返回 197，加载失败返回 70。

这让 shell 脚本可以直接按退出码分流，而不必解析日志文本。

## 注意事项
- `RunCliAction` 里 `ExitStatus`（20/30/50/70）与 `ExitCodes`（192+）并存，写自动化脚本时要同时覆盖两组区间。
- `--search` 已标注弃用，代码会自动把 `IgnoreLoadErrors` 打开并给出 warning；新脚本尽量别依赖这条旧路径。
- 若你自己实现 `ICliAction`，建议抛 `ExitCodeException` 或明确返回码，不要把所有异常都交给最外层 `GeneralException(193)`，否则 CI 可观测性会变差。

## 小结
OpenTAP 的 CLI 退出策略不是“最后 catch 一把”，而是从入口到动作层的分层协议：信号统一转取消、取消统一转固定码、测试结论保留独立状态码。这个设计对自动化环境非常实用：既能优雅中断，也能让流水线据码决策，不会把“用户停止”“参数错误”“测试失败”混为一谈。

关键源码路径：
- `Cli/TapEntry.cs`
- `Engine/Cli/CliActionExecutor.cs`
- `Engine/Cli/RunCliAction.cs`
- `Engine/Cli/ExitCodes.cs`
