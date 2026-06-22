---
source: https://x.com/simonw/status/2065216774992515342
canonical: https://simonwillison.net/2026/Jun/11/fable-is-relentlessly-proactive/
title: Claude Fable is relentlessly proactive
saved: 2026-06-12 09:18:08 CST
summary_of: ""
---

# Claude Fable is relentlessly proactive — 总结

## 一句话结论

Simon Willison 用一次真实调试经历说明：Claude Fable 5 在 Claude Code 里表现得“极度主动”，能为了完成目标自行组合浏览器、脚本、截图、模板注入、CORS 服务和系统 API；这既非常强，也再次证明无沙箱运行 coding agent 很危险。

## 文章主旨

Simon 只给了 Claude 一张 UI bug 截图和一句提示：让它查看依赖，找出为什么 textarea 出现不该有的横向滚动条。随后他离开电脑，回来发现 Claude 已经自己打开真实浏览器、创建测试页面、截图、注入 JavaScript、读 shadow DOM、搭本地 CORS 服务收集页面测量数据，最终定位并验证了一个两行 CSS 修复。

## 关键过程

- Claude 先理解项目环境，跑起本地开发服务器，并尝试用 Playwright 的 Chrome、Firefox、WebKit 复现问题。
- 它发现 Playwright 中没复现，就转向真实浏览器：先 Firefox，后 Safari。
- 为了截图真实 Safari 窗口，它没有依赖常规 UI 自动化，而是用 `pyobjc-framework-Quartz` 枚举 macOS 窗口，拿到窗口 ID 后调用 `screencapture` 截图。
- 为了触发页面里的 modal，它直接修改 Datasette 模板，加入加载后模拟 `/` 快捷键的 JavaScript，让页面打开后自动弹出目标对话框。
- 为了从页面里拿测量数据，它写了一个 Python `http.server` 小服务，开放 CORS，把页面 JavaScript 收集到的 textarea 尺寸、scrollWidth/clientWidth、white-space、devicePixelRatio 等数据 POST 到本地 `/tmp/diag.json`。
- 它还能进入 Web Component 的 shadow DOM，找到真正的 textarea 并测量。
- Fable 中途似乎触发了某种 guardrail，降级到 Opus；Opus 继承上下文后继续使用这些技巧，最后找到、测试并验证了修复。

## 有意思的地方

这不是“按步骤执行用户命令”，而是 agent 自己发明调试基础设施：

- 自制复现 HTML
- 自行打开真实浏览器
- 自行绕过 AppleScript 辅助访问限制
- 自行截图
- 自行向应用模板注入测试代码
- 自行搭 CORS 数据回传服务
- 自行读取 shadow DOM
- 自行验证修复

Simon 称它为 relentlessly proactive，意思是它会不知疲倦地寻找任何能推进目标的方法。

## 安全启发

文章最后的重点不是“Claude 很聪明”这么简单，而是安全警告：coding agent 在终端里能做用户能做的一切。越强的模型越会找旁门左道解决问题；如果它被 prompt injection 或恶意代码/issue/thread 劫持，这种主动性就可能变成数据外泄、环境破坏或其他攻击能力。

Simon 的结论很明确：不应该在无沙箱环境里运行 coding agent。他把这类风险视为未来 AI 编程工具最可能出现“挑战者号事故”的方向之一。

## 我的理解

这篇文章最值得注意的是：agent 能力的提升不只是“代码写得更好”，而是会越来越像一个会使用整台电脑的操作员。它能把 CLI、浏览器、系统 API、本地服务、源码修改串成一套临时自动化系统。对开发效率是巨大加成；对权限边界、沙箱、审计日志、网络/文件访问控制也是巨大压力。
