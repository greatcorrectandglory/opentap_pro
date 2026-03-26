---
title: "性能视角｜GitVersion 在 RC 分支为何改用 First-Parent 计数"
date: 2026-03-26 02:00:00
tags:
  - OpenTAP
  - Package
  - GitVersion
  - SemVer
categories:
  - OpenTAP
---

## 背景

在持续集成里，版本号最怕两件事：一是同一提交在不同 runner 上算出不同结果，二是分支合并方式一变，预发布号就“跳号”。OpenTAP 的 `tap sdk gitversion` 近期在 RC（release candidate）场景做了一个很实用的收敛：RC 分支按 first-parent 统计提交数，避免把复杂 merge 历史全部折算进去。

## 框架分析

这条链路在 Package 模块里很清晰：`GitVersionAction` 负责 CLI 入参和输出，真正计算在 `GitVersionCalulator.GetVersion(Commit)`。核心步骤是：
1) 读取 `.gitversion`；
2) 判定分支类型（beta/release/tag）；
3) 找默认分支共同祖先；
4) 计算“自版本变更点以来”和“自分叉点以来”的提交数；
5) 拼装 SemVer 的 prerelease 与 metadata。

关键点在第 4 步：当 `preRelease` 为 `rc` 时，`countCommitsBetween(..., firstParentOnly: true)`，而 alpha/beta 仍走完整提交计数。这样 RC 版本更接近“主线推进节奏”，不会被侧分支 merge 噪音放大。

## 实现过程

可以直接在仓库里复现：

```bash
cd /home/ops/clawd/repos/opentap
# 看最近 20 个提交对应的语义版本
./tap sdk gitversion --dir . --gitlog 20

# 对某个提交单独计算（示例）
./tap sdk gitversion --dir . 66564c7 --fields 5
```

如果你在 CI 使用浅克隆，命令可能报“history is incomplete”。`GitVersionAction` 已明确提示需要 `git fetch --unshallow`。这不是“多余防御”，而是版本计算依赖共同祖先与历史遍历，浅历史会让计数失真。

## 注意事项

- `.gitversion` 可以放在子目录，代码会沿仓库相对路径向上回溯匹配，不一定只认根目录。
- 默认分支优先取远端 `refs/remotes/*/HEAD`，本地分支落后时会尽量跟踪上游，减少“本地看起来对、CI 不一致”。
- 若本地分支包含未推送提交，`findFirstCommonAncestor` 会抛错提醒先对齐 upstream，这个限制非常必要。
- 分支名写入 metadata 前会做字符清洗和长度裁剪，避免生成不合法 SemVer。

## 小结

这次 RC first-parent 计数策略的价值，不在“算法更复杂”，而在“版本号更稳定且可解释”。对发布线来说，能稳定反映主线推进，比精确统计所有图上的提交更实用。若你在 OpenTAP 做多分支并行发布，建议优先检查 `.gitversion` 规则和 CI 克隆深度，再看版本跳动问题。

**关键源码路径：**
- `Package/CreatePackage/GitVersionCalculator.cs`
- `Package/CreatePackage/GitVersionAction.cs`
- `Package/XmlEvaulation/IVariableProvider.cs`
