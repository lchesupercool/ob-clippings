---
---

![Image](https://pbs.twimg.com/media/HF-p1RUbEAIH-6t?format=jpg&name=large)

在我最近与 Claude Code 用户的交流中，一个主题反复出现：1M token 的上下文窗口是一把双刃剑。

它让 Claude Code 能够更长时间地自主运行、更可靠地处理任务，但如果你不刻意管理 session，它也会带来上下文污染的风险。

Session 管理比以往任何时候都更重要，相关疑问也很多。你是只在一个终端里开一个 session，还是两个？每次 prompt 都开新 session？什么时候该用 compact、rewind 或 subagent？什么会导致一次糟糕的 compact？

这里面有大量细节会真正影响你使用 Claude Code 的体验，而几乎所有这些细节都归结于对上下文窗口的管理。

## 关于 Context、Compaction 与 Context Rot 的快速入门

![Image](https://pbs.twimg.com/media/HF-nqWCbEAE3Oan?format=jpg&name=large)

上下文窗口是模型在生成下一次回复时能"看到"的全部内容。它包括你的 system prompt、目前为止的对话、每一次工具调用及其输出，以及读过的每个文件。Claude Code 的上下文窗口是 100 万 token。

不幸的是使用上下文有少许代价，通常被称为 context rot（上下文腐烂）。Context rot 是指：随着上下文增长，模型性能会下降——因为注意力被分散到更多 token 上，旧的、无关的内容开始干扰当前任务。对我们的 1M 上下文模型，大约在 30-40 万 token 处会出现某种程度的 context rot，但这高度依赖任务，不是硬性规则。

上下文窗口是一个硬截止点，所以接近窗口末尾时，你需要把当前正在做的任务压缩成一个较短的描述，然后在新的上下文窗口里继续工作——我们称之为 compaction（压缩）。你也可以自己手动触发 compaction。

![Image](https://pbs.twimg.com/media/HF-ntaxboAAZuCm?format=jpg&name=large)

# 每一次对话轮次都是一个分支点

假设你刚让 Claude 做完某件事，此时你的上下文里已经有一些信息（工具调用、工具输出、你的指令），而你接下来的选项其实多得惊人：

- **Continue（继续）**——在同一个 session 里再发一条消息
- **/rewind（连按两次 esc）**——跳回之前的某条消息，从那里重新尝试
- **/clear**——开一个新 session，通常带上你从刚才所学中提炼的简介
- **Compact**——总结目前为止的 session，在总结之上继续
- **Subagent**——把下一块工作委托给一个有自己干净上下文的 agent，只把结果带回来

最自然的做法是直接继续，但其他四个选项的存在就是为了帮你管理上下文。

![Image](https://pbs.twimg.com/media/HF-n6mMbEAEImhv?format=jpg&name=large)

## 何时开新 session

新的 1M 上下文窗口意味着你现在可以更可靠地做更长的任务，比如让它从零搭一个全栈应用。但模型还没用完上下文，并不意味着你就不该开新 session。

我们的一般经验法则是：开始一项新任务时，就应该同时开一个新 session。

灰色地带是：你想做一些相关任务，其中部分上下文仍然有用、但并非全部。

例如：为你刚实现的功能写文档。虽然你可以开新 session，但 Claude 就得重读那些你刚实现的文件，更慢更贵。由于写文档不是一个对智能高度敏感的任务，多带点上下文换来不用重读相关文件的效率，是划算的。

## 用 Rewind 代替更正

![Image](https://pbs.twimg.com/media/HF-oDqjbEAI94h5?format=jpg&name=large)

如果只能挑一个习惯作为良好上下文管理的标志，那就是 rewind。

在 Claude Code 里，双击 Esc（或运行 /rewind）可以跳回任何一条之前的消息，从那里重新 prompt。该点之后的消息会从上下文中被丢弃。

Rewind 通常是更好的更正方式。例如：Claude 读了五个文件、尝试一种方法、没成功。你的直觉可能是输入"那不行，试试 X"。但更好的做法是 rewind 到读完文件之后那一刻，带着你学到的东西重新 prompt："不要用方案 A，foo 模块没暴露那个——直接走 B。"

你也可以用"summarize from here"让 Claude 总结它学到的东西，生成一段交接消息——有点像未来版本的 Claude（试过某事失败了的那个）写给过去版本 Claude 的留言。

![Image](https://pbs.twimg.com/media/HF-oKwBbEAAdb6I?format=jpg&name=large)

## Compacting vs. 全新 session

一旦 session 变长，你有两种方式减负：/compact 或 /clear（开新的）。它们感觉相似，但行为差别很大。

**Compact** 让模型总结目前为止的对话，然后用这份总结替换历史。它是有损的，你在信任 Claude 判断什么重要——但好处是你自己不用写，而且 Claude 可能会更全面地保留重要的学习成果或文件。你也可以传指令来引导它（如 `/compact focus on the auth refactor, drop the test debugging`）。

![Image](https://pbs.twimg.com/media/HF-oPtxaAAAUKMr?format=jpg&name=large)

用 /clear，你自己把要紧的事写下来（"我们在重构 auth 中间件，约束是 X，相关文件是 A 和 B，我们已经排除了方案 Y"），然后清空开始。工作量更大，但得到的上下文是你亲自判定为相关的。

## 是什么导致了一次糟糕的 compact？

![Image](https://pbs.twimg.com/media/HF-oy22bEAE_Jd8?format=jpg&name=large)

如果你跑过很多长 session，可能注意到某些 compact 特别糟糕。我们发现，糟糕的 compact 往往发生在模型无法预测你工作走向的时候。

例如：autocompact 在一段长时间调试 session 后触发，总结了整个排查过程，而你下一条消息是"现在去修 [bar.ts](http://bar.ts/) 里我们看到的另一个 warning"。

但因为整段 session 都聚焦在调试上，那个"另一个 warning"可能已经从总结中被丢掉了。

这个问题特别棘手，因为受 context rot 影响，模型在 compact 时正处于最不智能的状态。有了 1M 上下文，你有更多时间可以主动 /compact，并附上你接下来想做什么的描述。

## Subagent 与全新上下文窗口

![Image](https://pbs.twimg.com/media/HF-o6v1bQAA7pS6?format=jpg&name=large)

Subagent 是一种上下文管理方式，适合于你事先就知道某块工作会产生大量你不会再用到的中间输出。

当 Claude 通过 Agent 工具启动一个 subagent，这个 subagent 会获得自己全新的上下文窗口。它可以做任何需要的工作，然后综合结果——只有最终报告回到父级。

我们使用的心智测试是：这个工具输出我之后还会用到吗，还是只需要结论？

虽然 Claude Code 会自动调用 subagent，但你可能会想明确告诉它这么做。例如你可能想告诉它：

- "起一个 subagent，根据下面这个 spec 文件验证这次工作的结果"
- "起一个 subagent 通读另一个代码库，总结它是怎么实现 auth 流程的，然后你自己照着同样的方式实现"
- "起一个 subagent，根据我的 git 改动写这个功能的文档"

# 总结

总结一下：当 Claude 结束一轮回复、你即将发送新消息时，你处在一个决策点。

长期来看我们期望 Claude 能自己帮你处理这件事，但目前为止，这是你引导 Claude 输出的方式之一。

![Image](https://pbs.twimg.com/media/HF-qwt9bEAEa1eq?format=jpg&name=large)
