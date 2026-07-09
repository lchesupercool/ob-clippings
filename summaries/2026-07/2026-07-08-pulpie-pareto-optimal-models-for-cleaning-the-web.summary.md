---
title: "Pulpie：用于清洗 Web 的 Pareto 最优模型 — 中文摘要"
source: "https://usefeyn.com/blog/pulpie-pareto-optimal-models-for-cleaning-the-web/"
saved: "2026-07-08"
author: "Bhavnick Minhas, Shreyash Nigam"
summary_of: "Pulpie: Pareto-Optimal Models for Cleaning the Web"
tags: [web-extraction, data-cleaning, llm-data, encoder-models, inference-cost]
---

# Pulpie：用于清洗 Web 的 Pareto 最优模型 — 中文摘要

## 一句话结论

Feyn 发布了 Pulpie：一组专门用于从 HTML 页面中抽取正文、去除导航/广告/侧栏/页脚等 boilerplate 的 encoder 模型。它的核心价值是用接近 SOTA 的抽取质量，把 10 亿网页清洗成本从 Dripper 的约 15.9 万美元降到约 7900 美元，约便宜 20 倍。

## 文章主旨

文章认为 Web 内容抽取已经成为 LLM 数据管线中的关键瓶颈。语言模型在两个阶段消费 Web：

1. 预训练阶段：从网页中学习世界知识。
2. 推理阶段：RAG、搜索、上下文管理会把网页内容拉进模型上下文。

但原始 HTML 页面大部分是噪声。作者称典型 HTML 页面约 70% 的 block 是导航、广告、侧栏、页脚等 boilerplate，真正正文只占一小部分。抽取质量会直接影响训练语料质量和推理上下文质量。

Pulpie 的目标是：保留模型式抽取的质量，同时让它足够快、足够便宜，能真正用于大规模 Web 清洗。

## 为什么 Web 清洗重要

文章引用 AICC 的结果：同一个 Common Crawl 快照，用启发式抽取和模型式抽取分别构建语料，在其他训练条件不变的情况下，模型式抽取语料训练出的模型在 13 个 benchmark 平均准确率高 1.08 个百分点。

这说明数据清洗不只是工程洁癖，而是会直接变成模型能力差异。尤其是代码块、公式、表格这类结构化内容，传统启发式抽取很容易破坏结构。文章给出的对比中，Trafilatura 对代码块的相似度只有 0.13，而模型式抽取达到 0.91；公式相似度 Trafilatura 为 0.61，模型式抽取为 0.94。

推理阶段也一样。RAG 或搜索上下文中只要混入无关 passage，就可能干扰模型回答。干净上下文不仅提升准确率，也减少 token 浪费。

## 当前抽取方法的两类路线

文章把现有 Web 抽取器分为两类：

### 1. 结构式抽取器

代表包括 Trafilatura、Readability、magic-html、Boilerpipe。它们根据 HTML 标签、DOM 结构、文本密度等表面信号判断某个 block 是正文还是 boilerplate。

优点是便宜、易运行；缺点是容易把结构相似但语义不同的元素混淆。例如导航表格和数据表格在 DOM 上都像 table，但语义完全不同。

### 2. 阅读式抽取器

代表是 Dripper。它把页面交给 transformer，让模型读取内容后判断每个 block 是否属于正文。质量更高，但 Dripper 是 decoder，需要逐 token 生成标签。每一步都要从 GPU memory 读取整个模型，速度受内存带宽限制，因此成本高。

Pulpie 的创新点是保留“让模型读页面”的路线，但换成 encoder 架构：一次 forward pass 给所有 block 打标签，不逐 token 生成。这样把瓶颈从 memory bandwidth 转向 compute，大幅提升吞吐。

## Pulpie 的工作流程

完整管线分四步：

1. 简化 HTML：移除 scripts、styles 和格式噪声，为每个 block 加唯一 ID。
2. 分块：把 block tokenized 后打包进最多 8192 token 的 chunk；约 80% 页面可放进单个 chunk。
3. 分类：Pulpie 一次 forward pass 判断每个 block 是正文还是 boilerplate。
4. 返回：保留正文 block，输出 HTML 或 Markdown。

这个设计的一个重要好处是：页面长度不会直接导致失败。Dripper 有 32k 上下文限制，文章称它在 WebMainBench 上 135 个空结果中有 130 个是因为页面超过上下文窗口；Pulpie 通过 chunking 避免了这个问题。

## 训练数据与模型蒸馏

Pulpie 需要 block-level 标签数据，但公开数据不够，因此作者自己构建了一套数据集。

流程是：

1. 从 Common Crawl 采样 16670 个英文页面，每个 domain 最多一个。
2. 用 MinerU-HTML 把页面切成 block。
3. 用 DeepSeek V3.2 给每个 block 标注正文/boilerplate。
4. 过滤空页面、损坏页面等，剩下 15880 个页面。
5. 再用 Dripper 0.6B 作为第二标注器交叉验证。
6. DeepSeek 与 Dripper 的 block-level agreement 为 93.3%。
7. 最终只保留两者在至少 70% blocks 上一致的 14959 个页面。

然后训练 teacher：

- 基座：EuroBERT-2.1B
- 数据：14959 页
- loss：class-weighted cross-entropy
- 硬件：4x A100
- teacher 在 WebMainBench English 上 ROUGE-5 F1 = 0.873

由于 2.1B teacher 运行成本高，作者再蒸馏出两个学生模型：

| 模型 | 参数量 | ROUGE-5 F1 |
|---|---:|---:|
| Pulpie Orange Small | 210M | 0.862 |
| Pulpie Orange Base | 610M | 0.863 |
| Pulpie Orange Large / teacher | 2.1B | 0.873 |
| Dripper | 0.6B | 0.864 |

Small 只有 210M，却几乎追平 Dripper 0.6B，也只比 teacher 低约 1.1 F1 points，因此作者推荐 Small 作为生产默认模型。

## 质量结果

在 WebMainBench English 子集上，Pulpie 的结果是：

| 方法 | ROUGE-5 F1 | 空抽取页面数 |
|---|---:|---:|
| magic-html | 0.700 | 384 |
| Trafilatura | 0.619 | 16 |
| Pulpie Orange Small | 0.862 | 45 |
| Dripper | 0.864 | 135 |
| Pulpie Orange Base | 0.863 | 36 |
| Pulpie Orange Large | 0.873 | 21 |

结论是：

- Pulpie Large 单模型质量最高，超过 Dripper 0.9 F1 points。
- Pulpie Small 基本打平 Dripper，但只有三分之一参数量。
- 启发式方法在困难页面上掉分更严重；Pulpie / Dripper 这类模型式方法更稳。
- Pulpie 空结果明显少于 Dripper，主要因为它能 chunk 长页面。

## 速度与成本

速度是这篇文章最强的卖点。

在 NVIDIA L4 上：

| 方法 | 吞吐 |
|---|---:|
| Pulpie Orange Small | 13.7 pages/sec |
| Pulpie Orange Base | 3.9 pages/sec |
| Pulpie Orange Large | 1.3 pages/sec |
| Dripper | 0.68 pages/sec |

Pulpie Small 在 L4 上比 Dripper 快约 20 倍。

在 A100 上：

| 方法 | 吞吐 |
|---|---:|
| Pulpie Orange Small | 25.7 pages/sec |
| Pulpie Orange Base | 7.7 pages/sec |
| Pulpie Orange Large | 3.5 pages/sec |
| Dripper | 3.6 pages/sec |

Pulpie Small 在 A100 上比 Dripper 快约 7.1 倍。

按 10 亿页面计算成本，L4 每小时 0.39 美元：

| 方案 | GPU-hours / 1B pages | 成本 / 1B pages |
|---|---:|---:|
| Pulpie Small on L4 | 20300 | ~$7900 |
| Dripper on L4 | 408000 | ~$159000 |
| Pulpie Base on L4 | 71200 | ~$28000 |
| Pulpie Large on L4 | 214000 | ~$83000 |

在 A100 每小时 2.72 美元下，Pulpie Small 约 2.9 万美元 / 10 亿页，Dripper 约 21 万美元 / 10 亿页。

## 为什么便宜 GPU 更适合 Encoder

文章给了一个很好的硬件解释：decoder 和 encoder 的瓶颈不同。

Decoder 逐 token 生成标签，每一步都要读完整模型权重，因此受 GPU memory bandwidth 限制。Encoder 一次 forward pass 处理整个输入，主要是 dense matrix multiply，更偏 compute-bound。

A100 和 L4 的差距在显存带宽上更大：

| 指标 | A100 | L4 | A100/L4 |
|---|---:|---:|---:|
| Memory Bandwidth | 2039 GB/s | 300 GB/s | ~6.8x |
| Tensor Core TFLOPS | 312 | 120 | ~2.6x |

所以从 A100 换到 L4 时，Dripper 这种 bandwidth-bound decoder 掉速更严重；Pulpie 这种 compute-bound encoder 更能保住吞吐。这也是为什么 Pulpie 在便宜 L4 上优势更夸张。

## 如何使用

Pulpie 已开源并发布到 Hugging Face。安装：

```bash
pip install pulpie
```

基本用法：

```python
from pulpie import Extractor

extractor = Extractor()  # 默认 Pulpie Orange Small
result = extractor.extract(html)

print(result.markdown)
print(result.n_main, result.n_other)
```

也可指定更大模型：

```python
extractor = Extractor(model="large")
```

批处理可用 `Pipeline`，支持 CPU 预处理与 GPU 推理重叠，也支持多 GPU。

## 对 LLM 数据工程的意义

Pulpie 的重要性不只是“又一个网页抽取库”，而是它击中了 Web-scale 数据处理里的成本瓶颈。

如果模型式抽取只能用高成本 decoder 跑，那它更像高质量但不可规模化的工具；如果 210M encoder 能以近似质量、20 倍低成本跑在 L4 上，它就可能进入真正的大规模语料构建、RAG 预处理、搜索索引清洗、网页转 Markdown、上下文工程等生产管线。

对预训练来说，它可能提高语料信噪比，减少代码/公式/表格破坏；对 RAG 来说，它能减少无关上下文，降低 token 成本，提升回答稳定性。

## 我的理解

这篇文章最值得注意的是架构选择：Pulpie 不是单纯把模型做小，而是把任务重新表述成 encoder block classification。Web 正文抽取本质上是“给 DOM block 打标签”，并不天然需要 decoder 逐 token 生成。把任务形态和模型架构对齐后，质量、速度、成本同时改善。

这也给很多 LLM 工程任务一个启发：并不是所有“需要理解文本”的任务都该交给大 decoder。分类、过滤、路由、抽取、清洗、打分、action gating 这类任务，往往更适合专门训练的小 encoder 或小模型。它们的价值不是炫技，而是在高频、大规模、低延迟、低成本场景中把系统成本打下来。

对 Hermes / Obsidian / LLM Wiki 这类个人知识库工作流来说，Pulpie 也很契合：它可以作为比 Trafilatura / Readability 更高质量的网页正文抽取器，用来生成更干净的 Markdown、降低摘要和翻译时的噪声。后续如果本地或远端部署方便，值得考虑接进 clippings pipeline。
