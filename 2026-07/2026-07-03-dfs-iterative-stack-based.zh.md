---
title: 基于栈的迭代式 DFS 图遍历
source: https://dwf.dev/blog/2024/09/23/2024/dfs-iterative-stack-based/
source_markdown: https://raw.githubusercontent.com/farlowdw/software-development-handbook/master/blog/2024/2024-09-23-dfs-iterative-stack-based/index.md
author: Daniel Farlow
published: '2024-09-23'
saved: '2026-07-03'
translation_of: 2026-07-03-dfs-iterative-stack-based.md
tags:
- clippings
- dfs
- graph-traversal
- stack
- algorithms
- python
---

# 基于栈的迭代式 DFS 图遍历

图上的深度优先搜索（DFS，无论是二叉树还是其他图）最常见的实现方式是*递归*，但有些场景下我们可能希望考虑一种*迭代式*方法。比如，当我们担心调用栈溢出时，就有理由用我们自己的栈来实现 DFS，而不是依赖程序隐式的调用栈。但如果不小心，这样做也会引出一些问题。

具体来说，正如[另一篇博客文章](https://11011110.github.io/blog/2013/12/17/stack-based-graph-traversal.html)所指出的，我们很容易落入一个陷阱：随意使用一个栈，结果执行的图搜索并不是真正的 DFS。这个问题在某些题目中可能不会暴露，但依赖*真正* DFS 的算法可能会失败，例如用于寻找强连通分量的 [Kosaraju 算法](https://en.wikipedia.org/wiki/Kosaraju%27s_algorithm)和 [Tarjan 算法](https://en.wikipedia.org/wiki/Tarjan%27s_strongly_connected_components_algorithm)。上面链接的博客非常有效地指出了潜在问题，但讲解相当简洁。因此，为了便于参考，该博客已在[下文](#blog-post)原样转载。

*本文*主要是为了进一步探索下面转载的那篇博客，并配上可运行的 Python 代码。

## 展示问题

**TLDR**

看下面这张图。如果我们从顶点 `S` 开始做 DFS，那么下一个访问的顶点可以是 `A` 或 `C`。如果我们从 `S` 访问 `A`，那么 DFS 接下来自然应该访问 `B` 或 `D`（我们不会回到 `S`，因为它已经被访问过）。如果我们访问的是 `C`，那么 DFS 接下来自然应该访问 `D` 或 `F`（同样不会重新访问 `S`）。

但在有缺陷的基于栈的遍历中，发生的不是这件事。相反，从顶点 `S` 出发后，我们会先访问 `A`，然后访问 `C`，这不是一个合法的 DFS。

考虑下面这张图：

![graph traversal diagram](../../assets/dfs-iterative-stack-based/image-01.png)

我们可以在代码中用索引数组形式的邻接表来表示这张图，其中顶点 `S` 映射到索引 `0`，顶点 `A`、`B`、`C` 等分别映射到索引 `1`、`2`、`3` 等。尽管图中顶点用小写表示，本文接下来仍然使用大写记号表示顶点：

```python
lookup = {
    0: 'S',
    1: 'A',
    2: 'B',
    3: 'C',
    4: 'D',
    5: 'E',
    6: 'F',
    7: 'G',
    8: 'H',
}

graph = [         # Edges:
    [1, 3],       # S: (S, A), (S, C)
    [0, 2, 4],    # A: (A, S), (A, B), (A, D)
    [1, 5],       # B: (B, A), (B, E)
    [0, 4, 6],    # C: (C, S), (C, D), (C, F)
    [1, 3, 5, 7], # D: (D, A), (D, C), (D, E), (D, G)
    [2, 4, 8],    # E: (E, B), (E, D), (E, H)
    [3, 7],       # F: (F, C), (F, G)
    [4, 6, 8],    # G: (G, D), (G, F), (G, H)
    [5, 7],       # H: (H, E), (H, G)
]
```

假设我们想从顶点 `S` 作为源点开始探索上面的图。如果执行标准 BFS 和标准 DFS，其他顶点会以什么顺序被访问？我们会得到类似下面的结果（注意，上面的 `graph` 定义让相邻顶点按字母顺序被访问）：

```python title="Standard BFS" showLineNumbers
def bfs(graph, source):
    n = len(graph)
    queue = deque([source])
    visited = [False] * n
    visited[source] = True

    while queue:
        node = queue.popleft()
        for nbr in graph[node]:
            if not visited[nbr]:
                visited[nbr] = True
                queue.append(nbr)

bfs(graph, 0) # (S) A C B D F E G H
```

```python title="Standard DFS" showLineNumbers
def dfs(graph, source):
    n = len(graph)
    visited = [False] * n
    
    def visit(node):
        visited[node] = True
        for nbr in graph[node]:
            if not visited[nbr]:
                visit(nbr)
                
    visit(source)


dfs(graph, 0) # S A B E D C F G H
```

现在我们尝试通过把标准 BFS 中的队列替换成栈，来实现一个迭代式 DFS：

```python title="Standard BFS" showLineNumbers
def bfs(graph, source):
    n = len(graph)
    queue = deque([source])
    visited = [False] * n
    visited[source] = True

    while queue:
        node = queue.popleft()
        for nbr in graph[node]:
            if not visited[nbr]:
                visited[nbr] = True
                queue.append(nbr)
    
bfs(graph, 0) # (S) A C B D F E G H
```

```python title="Attempted DFS with stack (replace queue in BFS)" showLineNumbers
def dfs_stack(graph, source):
    n = len(graph)
    #highlight-next-line
    stack = [source]
    visited = [False] * n
    visited[source] = True

    #highlight-start
    while stack:
        node = stack.pop()
    #highlight-end
        for nbr in graph[node]:
            if not visited[nbr]:
                visited[nbr] = True
                #highlight-next-line
                stack.append(nbr)

dfs_stack(graph, 0) # (S) A C D F G H E B
```

**关于上面代码块的说明：**

- 访问顺序：如果我们在标准 BFS 的第 `11` 行、标准 DFS 的第 `6` 行、以及尝试版迭代式 DFS 的第 `11` 行插入 `print(lookup[nbr])`，那么调用已定义函数时，会得到每个代码块第 `14` 行注释中标出的输出。
- 输出记号：`(S)` 表示 `S` 已经被访问过，因此不会被上面提到的 `print` 语句打印出来。只有标准 DFS 不打印 `S`，因为标准 DFS 是在进入 `visit` 函数时把顶点标记为已访问。

如果我们在每次发现后续顶点时加入下面的 `print` 语句，就能从上面的代码块中获得更多洞察：

```python
print(f'Vertex {lookup[nbr]} discovered by {lookup[node]}')
```

这一行让我们能够观察每个后续顶点是从哪个顶点被发现的。我们可以把它插入标准 BFS 的第 `11` 行、标准 DFS 的第 `9` 行、尝试版迭代式 DFS 的第 `11` 行。三种情况下的输出如下：

```python title="Standard BFS"
Vertex A discovered by S
Vertex C discovered by S
Vertex B discovered by A
Vertex D discovered by A
Vertex F discovered by C
Vertex E discovered by B
Vertex G discovered by D
Vertex H discovered by E
```

```python title="Standard DFS"
Vertex A discovered by S
Vertex B discovered by A
Vertex E discovered by B
Vertex D discovered by E
Vertex C discovered by D
Vertex F discovered by C
Vertex G discovered by F
Vertex H discovered by G
```

```python title="Attempted iterative DFS"
Vertex A discovered by S
Vertex C discovered by S
Vertex D discovered by C
Vertex F discovered by C
Vertex G discovered by F
Vertex H discovered by G
Vertex E discovered by H
Vertex B discovered by E
```

如果我们画出不同的搜索树，问题会更明显。图中绿色边连接每个顶点与先前发现它的顶点；红色边表示非发现边或“非树边”，也就是原图中存在、但在发现后续顶点时没有使用的边：

![graph traversal diagram](../../assets/dfs-iterative-stack-based/image-02.png)
![graph traversal diagram](../../assets/dfs-iterative-stack-based/image-03.png)
![graph traversal diagram](../../assets/dfs-iterative-stack-based/image-04.png)

[链接文章](#blog-post)指出了上面栈遍历的问题：

> 问题在于，树中较高层的节点过早地把它的一些邻居作为子节点压入；而在一个正确的深度优先搜索中，这些子节点本应在树中更深的位置作为后代被发现。

回忆一下我们一开始的输入图：

![graph traversal diagram](../../assets/dfs-iterative-stack-based/image-05.png)

当我们从顶点 `S` 开始执行 DFS 时，对于接下来访问哪个顶点，有哪些选项？

- **顶点 `A`：** 假设我们先访问顶点 `A`。那么搜索中接下来可能访问的顶点应该是 `B` 或 `D`（我们不想重新访问 `S`）。
- **顶点 `C`：** 假设我们先访问顶点 `C`。那么搜索中接下来可能访问的顶点应该是 `D` 或 `F`（我们不想重新访问 `S`）。

无论哪种情况，在搜索的下两个顶点中同时访问 `A` *和* `C` 都说不通。

但这正是上面栈遍历中发生的事情。具体来说，用[链接文章](#blog-post)的说法，对于树中最高的顶点（即 `S`，搜索树的根顶点），我们过早地把它的一些邻居作为子节点压入了栈（在这个例子中，就是那个没有被首先选中的顶点）。在正确的 DFS 中，例如上面中间那张图所展示的那样，子节点应该在树中更深处作为后代被发现——当选择 `A` 时，我们看到 `C` 是在树中更深处作为后代被发现的。

最后，再看链接文章中的另一段摘录：

> 在深度优先搜索树中，所有边都连接祖先和后代。在广度优先搜索树中，所有边都连接同一层或相邻层的顶点。但在栈遍历树中，所有非树边都连接一对互相不是祖先也不是后代的顶点，这与深度优先树的性质正好相反。换一种说法，如果 $w$ 是 $v$ 的后代，并且与 $v$ 相邻，那么 $w$ 必须是 $v$ 的直接子节点。

这是什么意思？我们可以先回忆一下：如果我们从某个顶点 `X` 发起 DFS，那么最终会创建一棵以 `X` 为根的 DFS 树。对于我们正在讨论的例子，这意味着顶点 `S` 是 DFS 树的根。上面中间那张正确 DFS 的图展示了*每个*顶点都是 `S` 的后代；具体来说，红色高亮的非树边 `(S, C)`、`(A, D)`、`(E, H)`、`(D, G)` 表明：`S` 是 `C` 的祖先，`A` 是 `D` 的祖先，`E` 是 `H` 的祖先，`D` 是 `G` 的祖先。这些都是一棵树被认为是合法 DFS 树的必要条件：

> 在深度优先搜索树中，所有边都连接祖先和后代。

但栈遍历树不是这样。`A` 唯一的祖先是 `S`。`D` 的祖先只有 `C` 和 `S`。但非树边 `(A, B)`、`(A, D)`、`(D, G)`、`(D, E)` 会造成问题，因为每条边的两个端点既不是祖先也不是后代。每个端点位于不同的分支上。这棵栈遍历树*不是*深度优先树。

## 修复问题

[那篇博客文章](#blog-post)提到了两种修复方式。第一种是使用迭代器栈，而不是顶点栈：

```python
def dfs_stack_iterators(graph, source):
    n = len(graph)
    visited = [False] * n
    visited[source] =  True
    stack = [iter(graph[source])]
    while stack:
        try:
            node = next(stack[-1])
            if not visited[node]:
                visited[node] = True
                stack.append(iter(graph[node]))
        except StopIteration:
            stack.pop()
```

这种方法在使用迭代器方面是空间高效的，本质上允许迭代式方法暂停并恢复对某个顶点邻居的探索，也就是类似标准递归方法中的函数调用。效率来自每个迭代器都能记录自己在邻接表中停在了哪里，从而避免重新处理邻居。就顶点访问顺序和空间需求而言，这种方法都与标准递归 DFS 匹配。不过，使用迭代器栈多少有些不常见。

第二种修复方式仍然使用顶点栈，但改变未来顶点的处理方式。不要在把邻居加入栈*之前*检查某个顶点的邻居是否已访问（也就是有问题的栈遍历方式）；而是把所有邻居都压入栈，并把“相邻顶点是否已经访问过”的检查延迟到它从栈中弹出*之后*：

```python title="Attempted iterative DFS (bad)" showLineNumbers
def dfs_stack_bad(graph, source):
    n = len(graph)
    stack = [source]
    visited = [False] * n
    visited[source] = True

    while stack:
        node = stack.pop()
        for nbr in graph[node]:
            if not visited[nbr]:
                visited[nbr] = True
                stack.append(nbr)

dfs_stack(graph, 0) # (S) A C D F G H E B
```

```python title="Attempted iterative DFS (good)" showLineNumbers
def dfs_stack_good(graph, source):
    n = len(graph)
    visited = [False] * n
    stack = [source]
    

    while stack:
        node = stack.pop()
        if not visited[node]:
            visited[node] = True
            for nbr in graph[node]:
                stack.append(nbr)

dfs_stack_good(graph, 0) # S C F G H E D A B
```

注意，判断顶点是否已访问的检查应该在顶点从栈中弹出后立即发生（good attempt 中的第 `9` 行）。第二种使用顶点栈的方法的取舍是：由于栈中会有重复条目，它会使用更多空间。此外，第二种方法会*反转*每个顶点的子节点顺序（如果我们把 `1, ... , N` 依次加入栈，那么它们会以相反顺序被弹出和处理：`N, ... , 1`）。如果我们*真的*想执行一种尽可能模拟标准递归 DFS 的迭代式栈 DFS，那么就需要按反向顺序处理邻居：

```python
def dfs_iterative(graph, source):
    n = len(graph)
    visited = [False] * n
    stack = [source]
    
    while stack:
        node = stack.pop()
        if not visited[node]:
            visited[node] = True
            for i in range(len(graph[node]) - 1, -1, -1):
                nbr = graph[node][i]
                stack.append(nbr)
```

正如[博客文章](#blog-post)也提到的，上面两种方法都有一个性质：如果把栈替换成队列，就会得到广度优先搜索，尽管这种 BFS 的实现方式有些非标准。

## 尝试不同遍历方式

:::caution 代码编辑器限制

下面的代码编辑器使用 [Piston API](https://piston.readthedocs.io/en/latest/configuration/#compilerun-timeouts)，它对*输出*（即 `stdout`）有 1024 字符的[硬上限](https://github.com/engineer-man/piston/issues/301#issuecomment-883423481)。还有其他限制性的[安全特性](https://github.com/engineer-man/piston?tab=readme-ov-file#security)，但最主要要记住的是：如果输出超过 1024 字符上限，编辑器下面的输出框会显示 `No output`。因此，对于下面第一个小节，不要一次运行所有函数（会超过输出字符限制）；相反，应该通过取消注释来分别尝试运行每个函数。

:::

### 博客文章中的遍历方式

上文提到的所有遍历及其变体，一次性吸收可能比较困难。如果能在同一个环境中看到每种遍历的实现，并明确展示被发现的节点以及它们是由哪个节点发现的，会更容易理解：

```text
Interactive code editor omitted in clipping; see source page.
```

### 你自己的遍历方式

可以通过调整上面编辑器中的不同方法，或者设计一种全新的方法，来实验你自己的遍历方式。为了便于使用和参考，`graph` 定义以及对应的 `lookup` 已经预先填好。

```text
Interactive code editor omitted in clipping; see source page.
```

## 基于栈的图遍历 ≠ 深度优先搜索 {#blog-post}

:::info 归属说明

下面的博客文章转载自 David Eppstein 的文章：[stack-based graph traversal ≠ depth first search](https://11011110.github.io/blog/2013/12/17/stack-based-graph-traversal.html)。为便于参考，下文原样转载。

:::

我刚刚结束了必修本科算法课的教学，在批改期末考试时发现，有几个学生（不是从我这里）形成了一个错误认知：把标准广度优先搜索算法中的队列替换为栈，就会变成深度优先搜索。令人尴尬的是，维基百科的[深度优先搜索](https://en.wikipedia.org/wiki/Depth-first_search)条目也犯了同样的错误（直到今天才改正），一些教材也是如此，例如 Skiena 的 *Algorithm Design Manual* 第 169 页、Jeff Edmonds 的 *How to Think about Algorithms* 第 175–178 页、Gilberg 和 Forouzan 的 *Data Structures: A Pseudocode Approach Using C* 第二版第 497 页。

如果你把广度优先搜索中的队列换成栈，会得到下面这个东西：

```python
def stack_traversal(G,s):
    visited = {s}
    stack = [s]
    while stack:
        v = stack.pop()
        for w in G[v]:
            if w not in visited:
                visited.add(w)
                stack.append(w)
```

在树中，或者在 AI 搜索语境中，如果不使用 visited 集合来消除重复顶点，这个想法确实会产生深度优先搜索。但在带有 visited 集合的任意图中，由这个例程得到的遍历并不是深度优先。问题在于，树中较高层的节点过早地把它的一些邻居作为子节点压入；而在正确的深度优先搜索中，这些子节点本应在树中更深的位置作为后代被发现。下面是一个例子，照例用搜索树来说明，其中每条树边连接一个顶点与较早发现它的顶点：

![graph traversal diagram](../../assets/dfs-iterative-stack-based/image-06.png)

在深度优先搜索树中，所有边都连接祖先和后代。在广度优先搜索树中，所有边都连接同一层或相邻层的顶点。但在栈遍历树中，所有非树边都连接一对互相不是祖先也不是后代的顶点，这与深度优先树的性质正好相反。换句话说，如果 $w$ 是 $v$ 的后代，并且与 $v$ 相邻，那么 $w$ 必须是 $v$ 的直接子节点。

Goodrich 和 Tamassia（我在课上使用他们的书）以及 CLRS 通过只讨论递归式深度优先搜索避免了这个错误；CLRS 有一道练习要求写出迭代式、基于栈的深度优先搜索，但没有给出答案。有一本讨论如何用栈实现迭代式深度优先搜索并且讲对了的教材是 Sedgewick 的 *Algorithms in Java*。至少有两种不同做法，Sedgewick 都讨论过。你可以使用迭代器栈，而不是顶点栈：

```python
def dfs(G,s):
    visited = {s}
    stack = [iter(G[s])]
    while stack:
        try:
            w = stack[-1].next()
            if w not in visited:
                visited.add(w)
                stack.append(iter(G[w]))
        except StopIteration:
            stack.pop()
```

另一种方法是压入所有邻居，并把“某个顶点是否已经访问过”的检查推迟到它被弹出时再做。这是 Kleinberg 和 Tardos 的 *Algorithm Design* 采用的方法：

```python
def dfs2(G,s):
    visited = set()
    stack = [s]
    while stack:
        v = stack.pop()
        if v not in visited:
            visited.add(v)
            for w in G[v]:
                stack.append(w)
```

这两种方法都有一个性质：如果把栈替换成队列，就会得到广度优先搜索，只是实现方式有点非标准。它们彼此不会给出相同的遍历（第一种匹配通常的递归 dfs，而第二种会反转每个顶点的子节点顺序），并且第二种因为栈中存在重复条目会使用更多空间，但至少二者都给出了合法的深度优先搜索。

深度优先搜索、广度优先搜索，以及[词典序广度优先搜索](https://en.wikipedia.org/wiki/Lexicographic_breadth-first_search)，在算法设计中都很有用，因为它们对图中其余部分如何连接到搜索树施加了受限结构。非 DFS 的栈遍历是一种不同类型的图遍历，因此可以想象它也可能以这种方式有用。但我不知道有任何算法会刻意使用它来替代 BFS 或 DFS。
