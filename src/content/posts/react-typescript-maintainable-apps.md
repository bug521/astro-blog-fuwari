---
title: "如何使用React和TypeScript构建可维护的前端应用"
published: 2025-11-09
tags: ["技术", "教程", "项目"]
description: "本文详细介绍了如何结合React和TypeScript的类型系统，构建具有良好可维护性和扩展性的前端应用程序，包含实际项目中的最佳实践和常见陷阱。"
author: "John Doe"
category: "编程技术"
cover: "../../assets/64ea7ae60b4ee9e2.jpg"
slug: "react-typescript-maintainable-apps"
---

# 如何使用React和TypeScript构建可维护的前端应用

Cloudflare 的「自动程序攻击模式」并不是由普通的 Firewall Rule 触发，而是位于 Security → Bots → Bot Attack Mode 这一独立模块。

因此你在 WAF 里写「UA = NotionAutomation → 跳过」对它无效，规则执行顺序里它晚于 Bot Management，所以请求仍会被质询（JS Challenge）→ Notion 拿不到 200，只能报 403 。

![](../../assets/620b8940c6d68b06.png)

下面给出免费版用户能用的 2 种「放行」办法，按顺序操作即可：

---

1. 先验证是 Bot Attack Mode 误杀
---

1. 控制台 → Security → Bots
把 Bot Attack Mode 开关关闭 → Save。
1. 立刻去 Notion 里点 Test webhook。
- 如果马上 200，就坐实了是它拦的；
- 仍旧 403 说明还有别的规则，再往下看第 2 步。
---

1. 长期放行（免费版思路）
---

免费版不能针对 UA/IP 单独关闭「Bot Attack Mode」，但可以用 IP Access Rules 把它整体放行：

1. 整理 Notion 目前常用的 AWS 出口段（实测够用）：
```plain text
3.248.180.0/22
3.251.104.0/22
52.48.0.0/16
54.78.0.0/16

```

1. 控制台 → Security → WAF → Tools → IP Access Rules
- Action 选 Allow
- 依次把上面 4 段填进去，Scope 选 This Website → Add。
1. 重新打开 Bot Attack Mode，再测试 webhook，Security Events 里会看到原来的 Challenge 变成 Allow 。
---

1. 如果还拦，再补一条「UA + IP」双层 Bypass
---

同页面继续加：

- Field: User-Agent
- Operator: contains
- Value: Notion
- Action: Allow
这样即使 Notion 以后扩容到新 IP，也能优先匹配 UA 被放行。

---

## 小结

- 你之前写的「UA 白名单」对 Bot Attack Mode 无效，所以没生效；
- 免费版唯一可行的是 IP Access Rules（Allow） + UA Allow，或者干脆关闭 Bot Attack Mode；
- 按上面 3 步做完，Notion webhook 基本都能稳定 200。仍有新事件把 Rule ID 贴出来再帮你查。
