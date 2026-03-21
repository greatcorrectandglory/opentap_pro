---
title: "源码拆解｜tap run 外部参数是怎样灌进 TestPlan 的"
date: 2026-03-21 02:05:00
tags:
  - OpenTAP
  - CLI
  - ExternalParameter
categories:
  - OpenTAP
---

## 背景
在自动化产线里，同一份 TapPlan 往往要被不同批次、不同工位复用。最常见的做法不是改计划文件本身，而是通过 `tap run -e` 在运行时注入参数。表面看只是“命令行传个键值对”，但源码里这条链路其实分成“预装载参数、加载计划、按插件导入文件参数、严格校验”四段，顺序错了就会出现参数不生效或直接退出。

## 框架分析
OpenTAP CLI 的入口在 `RunCliAction.Execute()`。当命令包含 `-e/--external` 或 `-t/--try-external` 时，会进入 `HandleExternalParametersAndLoadPlan()`：
1. 先把 `name=value` 写入 `ExternalParameterSerializer.PreloadedValues`；
2. 再调用 `TestPlan.Load(...)` 反序列化计划；
3. 若 `-e` 里出现无等号项，按“外部参数文件”处理，交给 `IExternalTestPlanParameterImport` 插件；
4. 最后仅对 `-e` 做严格存在性检查，不存在就报错退出。

这个设计把“注入能力”和“容错策略”拆开了：`-e` 偏强约束，`-t` 偏兼容迁移。

## 实现过程
可复现实验（假设计划里有外部参数 `SerialNumber`）：

```bash
tap run ./Demo.TapPlan -e SerialNumber=SN-20260321 -e Station=A01
```

若参数名写错：

```bash
tap run ./Demo.TapPlan -e SerailNumber=SN-20260321
```

CLI 会在加载后触发“参数不存在”警告并返回参数错误码；如果改成 `-t SerailNumber=...`，则会忽略该错误继续跑计划。这正对应源码末尾那段“仅遍历 External（不遍历 TryExternal）并抛 `ArgumentException`”的分支。

## 注意事项
- `-e file.csv` 并不是魔法内建，依赖实现了 `IExternalTestPlanParameterImport` 的插件；缺插件会直接报不支持该扩展名。
- 读取外部参数文件前，代码会临时切到 `EngineSettings.StartupDir`，相对路径行为与当前 shell 目录可能不同。
- 命令行里用了 `--search` 时，源码会自动打开 `--ignore-load-errors`，这是兼容旧流程的“保守降级”，生产环境要谨慎依赖。

## 小结
`tap run` 的外部参数机制核心不是“字符串替换”，而是“先预装载、后反序列化、再插件导入、最后分级校验”的执行管线。理解这四步后，很多“本地能跑、产线偶发失败”的问题都能更快定位：到底是参数名、导入插件、还是容错开关选型不当。

**关键源码路径：**
- `Engine/Cli/RunCliAction.cs`
- `Engine/Cli/TestPlanRunner.cs`
