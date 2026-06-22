
# Hermes Agent 讲清楚（怎么用）

作者: The Startup Ideas Podcast (SIP) (@startupideaspod)

原文链接: https://x.com/startupideaspod/status/2046310040207016342

状态 ID: `2046310040207016342`

![](../../assets/x-startupideaspod-2046310040207016342/2026-04-21-startupideaspod-hermes-01.jpg)

Imran 拆解了他跑的是什么、怎么装的，以及他正在用它搭建的具体想法。

## 他为什么离开 OpenClaw
三个问题不断叠加：
- 没有内置 memory。他一遍又一遍地告诉它同样的东西。
- 网关需要重启。有一天甚至一小时一次。
- token 花销毫无可见度。他完全不知道钱烧在哪里。

他先试了 Nebula。作为 AI 同事还不错。但对那种会随时间学习的个性化工作流来说，不对味。然后是 Hermes。

![](../../assets/x-startupideaspod-2046310040207016342/2026-04-21-startupideaspod-hermes-02.jpg)

## Hermes 有什么不一样
内置 memory。它每成功完成一个任务，都会写入自己的 memory。用得越久越好用。

可搜索的历史。它用标准的 SQLite 数据库。如果它忘了保存某样东西（比如你传给它的一个 API key），它可以搜自己的日志把它找回来。

稳定性。Imran 一周多没重启过它了。

预装 40+ tools。浏览器、网页搜索、cron jobs、图像生成、home assistant。你不用到处找 tools。

预装 skills。在 MacBook 上，这意味着 Apple Notes、Apple Reminders、Find My、iMessage。装完就能用。

安装（Mac、Linux 或 Windows Subsystem for Linux）
从新的研究网站上的 Hermes Agent 文档里复制一行。
第一次在 Mac 上装 dev tool？你得先装 Xcode developer tools：xcode-select --install。

把安装命令粘过去。让它跑。你可以跳过引导。
唯一要记住的命令：hermes model。在这里你能看到开箱即用的每一个 provider 和每一个 model。

![](../../assets/x-startupideaspod-2046310040207016342/2026-04-21-startupideaspod-hermes-03.jpg)

## 他是怎么把 token 花销降低 90% 的
两步操作。

一：用 OpenRouter。它展示每个 model，带清晰的按 token 计价。它每周轮换免费 model。（录制这期播客时，NVIDIA 的 NemoTron 是免费的。）

二：把重复任务变成代码。如果你每天跑同一个任务，让 agent 把代码写一次。之后这个任务就是确定性的了。不用 agent 参与。不花 token。

他的数字：从大约每五天 130 美元降到大约每五天 10 美元。能力一样。

![](../../assets/x-startupideaspod-2046310040207016342/2026-04-21-startupideaspod-hermes-04.jpg)

## 安全配置
你可以让 Hermes 审计它自己的配置：「这安全吗？告诉我为什么安全或为什么不安全。」
它会检查明文的 key、薄弱的防火墙配置、暴露的 secrets。

三种运行模式：
- Bare metal（Imran 用的就是这个，每日更新）
- Docker container（和你的文件隔离开）
- Modal（serverless）

![](../../assets/x-startupideaspod-2046310040207016342/2026-04-21-startupideaspod-hermes-05.jpg)

## Android 安装（以及为什么这重要）
你用一个脚本装，思路和电脑端安装一样。需要额外装两个 app：
- Termux（Android 里的终端）
- Termux API（在 F-Droid 上可获取，让 agent 能访问手机传感器：电池、Wi-Fi、音量、摄像头、亮度、振动）

为什么要折腾？因为一台便宜的带 SIM 卡的 Android 就变成了一个全天候、低功耗的专用 agent 设备。不用抢那种卖光了的 Mac Mini。

它能解锁什么：
- 从真实设备发社交媒体（而不是通过会削弱触达的 scheduler API）
- 直接读 SMS
- 自动化短信收到的 2FA 验证码

Imran 在一台 Solana Seeker Android 手机上跑了一个。他给它起名叫 Cookie Monster。（他所有 agent 都以布偶（Muppets）角色命名。）

![](../../assets/x-startupideaspod-2046310040207016342/2026-04-21-startupideaspod-hermes-06.jpg)

## 怎么真正用起来（大多数人跳过的那部分）
Imran 的模式：先解决个人问题。这样你才学得会这种范式。

他第一个真正的收获是晚餐。他录了一段八分钟的 Telegram 语音备忘，把储藏柜里每一种食材都过了一遍。现在 agent 每天根据那里有什么以及他的健身目标给他发三个食谱。

小问题。大幅减少心智负担。

他已经在跑的其他东西：
- 早晨 Gmail 整理（删垃圾、退订没用的列表、返回一份关于什么重要的摘要）
- 报销单
- 一个 agent 自己整理的 Obsidian vault

关于 Obsidian：他以前不是用户。现在这是他的主控面板。agent 每天早上操作的 markdown 文件。今天的任务、本周的优先事项、即将到来的出行、工作、个人。全部由 agent 组织。

他没设计过这些。agent 在用了大约 20 天之后自己搭起来的。Imran 认为持续用 7 天就能让你走到大部分位置。

![](../../assets/x-startupideaspod-2046310040207016342/2026-04-21-startupideaspod-hermes-07.jpg)

## 他对自己跑的 prompt
他每天结束时对 agent 做 meta-prompt。问它：
- 我在拖延什么？
- 今天最重要该做的事是什么？
- 我每天在做哪些应该自动化的任务？
- 今晚你能给我造一个什么 tool，让我明天的生活更轻松？
- 今天有什么重要的事我漏掉了吗？

读完之后这些看起来很显然。但大多数人从来没问过自己。

## 一个 agent 还是多个
Imran 跑了四个。他是个折腾派。他觉得真正的答案是一个或两个：一个个人的，一个工作的。

原因：如果你在 Fortune 500 工作，他们不会让你在工作机上跑一个塞满私人数据的个人 agent。分开就干净，就像 to-do app 把个人列表和工作列表分开一样。

Cron jobs 还是 sub-agents：他把重复性任务作为 cron jobs 来跑，而不是 sub-agents。Sub-agents 让你把更便宜的 model 分配给更便宜的任务（一个 Gmail 整理 sub-agent 可以跑在小 model 上）。两种都行。这个领域还在摸索。

## 值得装的 skills
- Obsidian skill（即使你不用 Obsidian CLI）
- Gary Tan 的 G-Stack。最初是给 Cloud Code 做的。把 YC 的创业流程（每周迭代、问对问题、代码级决策）嫁接到你的 agent 上。免费。
- Honcho dev memory skill（有帮助，因为 Hermes 有 memory 限制，更小的 context 会好一些）
- 你自己的。银行对账单、个人财务、健身。为你已经在付费的东西做。

## 两条不可妥协的
- 每晚更新。它还在 beta。Imran 九天没更新，落后了 535 个 commits。
- 锁死它。配置 Telegram 或 WhatsApp 访问。装 Tailscale，这样你的手机和电脑都在同一个虚拟网络里，然后你从任何地方 SSH 进去。

## 更大的想法
学会用个人 agent 不是一项技能。它正在成为一项要求。

Imran 在一家基金工作。因为他的 agent 处理了背景工作，他多和 20%–30% 的创始人聊天。更好的信号。更好的 deal flow。更好的回报。

重点不是这个工具。而是这个工具从你的盘子里拿走了什么。

定制你的 agent 不是一项技能。用它把事情做完才是。

## 结语
Hermes Agent 就像 90 年代的改装车文化。找到零件。装上去。让它属于你。

但记住你在为什么而优化。不是这辆车。而是你要开去的那个地方。

关注 Imran：@imranye。
