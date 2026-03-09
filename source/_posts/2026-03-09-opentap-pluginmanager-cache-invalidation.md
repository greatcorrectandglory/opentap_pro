---
title: OpenTAP 插件扫描中的缓存失效机制
slug: opentap-pluginmanager-cache-invalidation
date: 2026-03-09 02:00:00
tags:
  - OpenTAP
  - PluginManager
  - 架构
categories:
  - OpenTAP源码分析
---

## 背景
在 OpenTAP 里，插件发现既要“快”，又要“准”。快，意味着 `GetPlugins<T>()` 不能每次都全盘扫描；准，意味着目录变化、包升级后必须及时看到新插件。源码里这两个目标并不是靠一个全局锁硬扛，而是通过“异步搜索 + 多层缓存 + 主动失效”配合完成，这也是 PluginManager 在大插件集合下仍能保持可用响应的关键。

## 框架分析
核心链路分三层：
1. **搜索状态层**：`PluginManager.SearchAsync()` 触发搜索，`searchTask`（`ManualResetEventSlim`）负责把“搜索进行中/已完成”暴露给读路径。
2. **查询缓存层**：`PluginFetcher` 内部用 `Memorizer<Type, ReadOnlyCollection<Type>>` 缓存 `GetPlugins(Type)` 结果，另有 `StaticPluginTypeCache<T>` 做泛型静态缓存。
3. **类型索引层**：`PluginSearcher` 持有 `AllTypes/PluginTypes`，真正存放扫描后的类型图。

当调用 `Search()` 时，会先 `searchTask.Reset()`，然后 `CacheState.OnUpdated()` 广播“缓存应失效”；搜索结束后 `searchTask.Set()` 放行查询。查询侧在 `PluginFetcher.GetPlugins` 里检测到“searcher 已替换”或“搜索未完成”，就会 `InvalidateAll()`，避免读到旧快照。

## 实现过程
源码里最值得复用的点是“先失效、后重建、最后放行”的顺序控制：

```bash
cd /home/ops/clawd/repos/opentap
# 快速定位关键实现
grep -n "SearchAsync\|searchTask\|InvalidateAll\|CacheState.OnUpdated" Engine/PluginManager.cs
```

对应实现可概括为：
- `SearchAsync()`：`searchTask.Reset()` → `CacheState.OnUpdated()` → 后台执行 `Search()`。
- `Search()`：刷新 resolver（`assemblyResolver.Invalidate`）后重扫程序集，再替换 `searcher`。
- `GetPlugins(Type)`：若发现搜索器变化或搜索进行中，先清查询缓存，再按需加载类型。

这套流程把“扫描耗时”隔离到后台，同时用事件和状态位保证一致性，避免查询线程拿到半更新数据。

## 注意事项
- **不要绕过失效通知**：如果二次开发里直接替换 `searcher` 却不触发 `CacheState.OnUpdated()`，`StaticPluginTypeCache<T>` 可能长期命中旧值。
- **并发读取要看门闩**：任何依赖搜索结果的入口都应经过 `GetSearcher()` 或等价等待逻辑，不能假设搜索已完成。
- **目录动态变更要配套 Invalidate**：`TapAssemblyResolver` 依赖 `Invalidate()` 同步搜索目录和解析缓存，否则会出现“新 DLL 在磁盘上、但解析器仍看不到”的错觉。

## 小结
PluginManager 的重点不在“找到插件”本身，而在“在并发环境下持续得到正确插件视图”。OpenTAP 通过 `searchTask + CacheState + Memorizer` 形成了一个轻量但完整的失效协议：搜索线程负责重建快照，查询线程负责感知版本变化并清缓存。对需要做热加载或包升级的平台来说，这个设计比单纯加锁更稳，也更容易扩展。

关键源码路径：
- `Engine/PluginManager.cs`
- `Engine/PluginSearcher.cs`
- `Engine/AssemblyFinder.cs`
