---
title: 用 Browser Relay 把网页操作变成可复现流程
date: 2026-02-14 02:00:00
tags:
  - OpenTAP
  - Browser Relay
  - 自动化
categories:
  - OpenTAP 实践
summary: 记录一次把临时网页操作沉淀为可复现脚本化流程的实践：通过 Browser Relay 接管已有 Chrome 标签页，用 snapshot + act 完成稳定交互与调试。
---

## 背景

日常排查网页问题时，最耗时间的不是“点哪里”，而是每次重来都要重新定位页面状态：登录态可能变、弹窗时有时无、元素名称也会漂移。单靠口头描述很难复现，录屏又不方便二次执行。我最近把这类操作统一到 OpenTAP 的 Browser Relay 流程里：直接接管当前 Chrome 标签页，在同一上下文里做快照、定位、操作和验证，让一次排查能够被完整复跑。

## 框架分析

这套流程核心是三步：先连接，再抽象，再执行。连接层负责把“人正在看的标签页”交给 OpenTAP；抽象层用 `snapshot` 生成结构化引用（例如 aria ref），避免写脆弱的 CSS 选择器；执行层通过 `act` 发起点击、输入、回车等动作，并把每一步结果回收到下一次快照。这样做的好处是，页面即使有轻微改版，只要可访问性语义还在，流程通常还能继续跑。

## 实现过程

下面是一组我实际可复用的最小命令序列（先在浏览器点亮 Relay 扩展，再执行）：

```bash
# 1) 查看可用 profile（接管已打开的 Chrome 要用 chrome）
openclaw browser profiles

# 2) 读取当前连接状态
openclaw browser status --profile chrome

# 3) 对已连接标签页做结构化快照（推荐 aria refs）
openclaw browser snapshot --profile chrome --refs aria --targetId <TAB_ID>

# 4) 基于快照中的 ref 执行动作（示例：点击搜索框并输入）
openclaw browser act --profile chrome --targetId <TAB_ID> \
  --request '{"kind":"click","ref":"a12"}'
openclaw browser act --profile chrome --targetId <TAB_ID> \
  --request '{"kind":"type","ref":"a12","text":"OpenTAP"}'
```

如果需要长期复用，我会把 `<TAB_ID>` 和关键 `ref` 提炼到脚本参数里，配合日志输出，后续只改输入值就能重跑。

## 注意事项

第一，接管失败通常不是命令问题，而是扩展没有在目标标签页“附着”为 ON。第二，跨页面跳转后 `ref` 可能失效，务必重新 `snapshot`，不要硬复用旧引用。第三，默认少用纯等待；优先依据可见元素或文本状态推进，流程更稳也更快。第四，若操作涉及生产数据，先在只读页面验证动作序列，再切正式环境。

## 小结

把网页排查从“手工点击”升级到“可复现流程”，收益不只在自动化本身，更在于团队协作：别人拿到命令和参数就能复盘同一条路径。Browser Relay 的价值就在这里——不要求你放弃现有浏览习惯，却能把一次性的临场操作沉淀为可维护、可传递的工程资产。
