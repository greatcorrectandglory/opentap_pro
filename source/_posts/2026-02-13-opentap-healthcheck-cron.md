---
title: 在 OpenTAP 上落地每日健康检查：从想法到可复现脚本
date: 2026-02-13 02:00:00
tags:
  - OpenTAP
  - 运维
  - Cron
categories:
  - 工程实践
summary: 记录一次把 OpenTAP 日常巡检从“想起来再看”改成“每天自动跑”的过程，包含脚本、定时配置与常见坑位。
---

最近把几个自动化任务迁到 OpenTAP 后，最明显的问题不是“功能不能用”，而是“状态看不见”。任务偶发失败时，如果当天没人手动点开面板，通常要等到第二天才发现。这个延迟在测试环境还能接受，放到生产就很危险了。

## 问题背景

最初我们依赖人工巡检：登录机器、看进程、看最近日志。流程不复杂，但极不稳定：忙的时候会漏看，夜里出问题也没人及时感知。目标很明确：把“是否健康”变成一个每天固定产出的结果，而不是靠记性。

## 框架分析

这件事可以拆成三层：

1. **采集层**：拿到 OpenTAP 当前状态（服务状态、最近错误日志）。
2. **判断层**：把原始信息变成可读结论（OK / WARN / FAIL）。
3. **触达层**：通过定时机制稳定触发，失败时留下明确线索。

OpenTAP 已经有 cron 能力，适合做触达层；采集和判断用本地脚本更灵活，也方便版本化。最终选择是“bash 脚本 + cron 任务”，尽量少引入额外依赖。

## 实现过程

先写健康检查脚本 `/home/ops/clawd/scripts/opentap-healthcheck.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail

STATUS=$(openclaw status 2>&1 || true)
NOW=$(date -u '+%Y-%m-%d %H:%M:%S UTC')

if echo "$STATUS" | grep -qi "running"; then
  echo "[$NOW] OK: OpenClaw/OpenTAP service is running"
else
  echo "[$NOW] FAIL: service not healthy"
  echo "$STATUS"
  exit 1
fi
```

然后在 OpenTAP 里加每日任务（UTC 02:10）：

```json
{
  "name": "opentap-daily-healthcheck",
  "schedule": { "kind": "cron", "expr": "10 2 * * *", "tz": "UTC" },
  "sessionTarget": "main",
  "payload": {
    "kind": "systemEvent",
    "text": "提醒：执行每日 OpenTAP 健康检查（脚本：/home/ops/clawd/scripts/opentap-healthcheck.sh）"
  }
}
```

触发后由会话执行脚本并记录输出；失败时会在同一上下文里留下错误文本，排查路径固定。

## 踩坑与注意事项

- **PATH 不一致**：cron 下的环境变量比交互式 shell 少，`openclaw` 可能找不到。稳妥做法是脚本里写绝对路径，或在开头手动 export PATH。
- **不要只看退出码**：有些异常会被包装成“命令成功但内容报错”，所以我额外 grep 了关键字。
- **时区要写死**：如果不显式写 `tz`，换机器后容易出现“昨天夜里跑到今天白天”的错觉。
- **日志要可定位**：脚本里统一打印 UTC 时间戳，回溯问题时能和系统日志对齐。

## 小结

这次改动本质上不是“加了个脚本”，而是把巡检从“人记得就做”改成“系统每天给结论”。对 OpenTAP 这种承载定时任务的平台来说，最有价值的不是炫技，而是把检查路径做短、做硬、做可重复。只要脚本和触发配置能在新机器一把落地，这件事就算真正完成了。