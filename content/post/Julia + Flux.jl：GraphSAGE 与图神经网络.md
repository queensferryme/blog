---
title: "Julia + Flux.jl：GraphSAGE 与图神经网络"
date: 2022-04-23T16:19:15+08:00
tags:
  - julia
  - machine learning
---

> 时隔多年，Julia 机器学习系列终于迎来~~并不值得期待的~~更新！

趁着毕业设计研究图神经网络的机会，让我们来聊聊如何用 Julia 实现一个简单的图神经网络！

<!--more-->

## 背景介绍

对神经网络有着基本了解的读者应该知道，神经网络比较适合处理一些结构化的数据 —— 这些数据通常可以以矩阵的形式表示，而神经网络只需要对这些矩阵不停的进行微分与代数运算就可以了。

对计算机科学有着基本了解的读者也应该知道，图并不是一种非常结构化的数据：一张图 $\mathcal{G}$ 由点集 $\mathcal{V}$ 与边集 $\mathcal{E}$ 组成，图中的每一个节点可以与任意多个其他节点相连，不存在严格规律的结构。一些算法试图使用图的邻接矩阵（adjacency matrix）来进行计算，但一般来说：

1. 邻接矩阵是稀疏的，如何在有限的内存中对较大的邻接矩阵进行存储和计算是个难题；
2. 邻接矩阵不易于扩展，新增的图节点或边往往会导致重新计算，耗费时间与资源。

本文将介绍一种名为 GraphSAGE[^1] 的图神经网络算法，它能利用常见的神经网络架构处理大量图结构数据。此外，GraphSAGE 还是一种归纳式（inductive）算法，而非一种直推式（transductive）算法 —— 归纳式学习算法能有效地泛化到未知的数据上；也就是说，当图中出现新增的节点或边时，GraphSAGE 无需重新训练就能得到比较好的结果。

## 问题设定

为了介绍图神经网络，我们有必要先引入一份图数据集 CORA[^2]。CORA 是一份论文引用网络数据集：在这份数据集中，每一篇论文构成图的一个节点，而论文之间的引用关系则形成图的无向边。此外，数据集还采用词袋（bag-of-words）模型为每一篇论文构建一个形如 $1433 \times 1$ 的特征向量：词袋考虑 1433 个彼此不同的单词，如果论文中包含该单词则特征值取 1，反之取 0。

而我们的任务，则是要利用论文节点的特征向量与节点之间的关系，来预测每个论文节点的类型。数据集中共包含七个类型的论文，每篇论文有且只有一个类型。如果不考虑图数据结构，这看起来就是个很简单的分类任务，对吧？

在 Julia 语言中，我们可以使用 `MLDatasets` 包来方便地下载 CORA 数据集：

```julia
using MLDatasets: Cora
data = Cora.dataset()
```

## 多层感知机

「如果不考虑图数据结构」其实是个不错的起步思路。至少我们可以先试试，在忽略图节点之间关联的情况下，我们能达到怎样的精度。

首先使用定义一个简单的多层感知机（Multi-Layer Perceptron，即全连接神经网络），输出层使用 `softmax` 激活。

```julia
using Flux

model = Chain(
    Dense(1433, 256),
    Dense(256, 7),
    softmax,
)
```

定义精确率（accuracy）函数，方便我们直观地感知模型的性能。损失（loss）函数使用交叉熵（cross entropy）。此外，我们还使用验证数据集来对模型训练进行早停（early stopping）以防止过拟合。

```julia
acc(x, y) = mean(Flux.onecold(model(x)) .== Flux.onecold(y))
loss(x, y) = Flux.crossentropy(model(x), y)

es = let f = () -> loss(val_x, val_y)
    Flux.early_stopping(f, 3; init_score=f())
end
```

训练 32 轮后停止，训练集精确度 100%，测试集精确度 57.3%。

```julia
Flux.@epochs 10000 begin
    Flux.train!(loss, Flux.params(model), dataset, opt)
    es() && break
end

@show acc(train_x, train_y)
@show loss(train_x, train_y)
@show acc(test_x, test_y)
@show loss(test_x, test_y)
```

不考虑图节点之间的关联，果然还是无法取得比较好的精确度啊 🤔

## GraphSAGE

既然无视图结构无法取得较好的成果，那就让我们尝试引入 GraphSAGE 图神经网络算法吧！

### 原理

GraphSAGE 的核心思想非常优雅简洁。个人总结起来就是：图中每一个节点的特征都受到周边节点的影响，距离越近影响越大。因此，图节点可以通过不断聚合（aggregate）邻居节点的特征来增强自身的特征 —— 通过多轮聚合，节点特征不仅包含自身的特征信息，还将包含节点在图拓扑结构中的位置信息、以及与周边节点的关系信息。

用更为形式化的语言表述如下：给定图 $\mathcal{G}(\mathcal{V},\mathcal{E})$，假设我们要进行 $K$ 轮聚合。对于 $k\in1\dots K$，$v\in\mathcal{V}$，令 $h_v^0$ 为节点 $v$ 的原始特征，$h_v^k$ 为节点 $v$ 在第 $k$ 轮中聚合得到的特征，$\mathcal{N}(v)$ 为 $v$ 邻居节点的集合，$\mathbf{W}$ 为权重矩阵，$\sigma$ 为激活函数，则

<tex>
$$
\begin{aligned}
&\mathbf{h}_{\mathcal{N}(v)}^k=\operatorname{\small{AGGREGATE}}_k(\big\{\mathbf{h}_u^{k-1},\ u\in\mathcal{N(v)}\big\})
\newline
&\mathbf{h}_v^k=\sigma(\mathbf{W}\cdot\operatorname{\small{CONCAT}}(\mathbf{h}_v^{k-1},\ \mathbf{h}_{\mathcal{N}(v)}^{k}))
\end{aligned}
$$
</tex>

GraphSAGE 算法在进行聚合时会依次遍历点集 $v\in\mathcal{V}$，这样做可行但性能不佳。神经网络通常会利用 mini-batch 技术来加速计算，即一次性批量处理多个样本。对上述 GraphSAGE 算法稍作修改，也能利用 mini-batch 加速计算：对于 $k\in1\dots K$，假设我们要批量处理一个节点集合 $\mathcal{B}^0\subset\mathcal{V}$，每个节点特征形如 $L\times1$，整个批的节点特征形如 $L\times|\mathcal{B}^0|$。令 $\mathcal{N}(v)$ 为邻居节点采样函数，采样函数会从节点 $v$ 的邻居节点中抽取 $n$ 个样本，如果

- 邻居节点数量大于等于 $n$ 个，则不带放回地抽取 $n$ 个样本；
- 邻居节点数量少于 $n$ 个，则有放回地采样，直到获得 $n$ 个样本。

> 之所以选择采样，而非直接使用所有邻居节点的集合，是因为确保采样数恒为 $n$ 至关重要 —— 下文会说到，如果采样数不一致，我们就无法利用矩阵运算加速技术，批量处理也就失去了提升性能的意义。

我们对 $\mathcal{B}^0$ 集合中所有节点的邻居节点进行采样，采样结果放入 $\mathcal{B}^1$ 集合，对 $\mathcal{B}^k$ 采样放入 $\mathcal{B}^{k+1}$ 集合，依次类推，直到取得 $\mathcal{B}^K$ 集合。由于采样数恒为 $n$，我们有 $|\mathcal{B}^k|\times n=|\mathcal{B}^{k+1}|$。

<tex>
$$
\begin{aligned}
&\mathbf{h}_{\mathcal{N}(\mathcal{B}^k)}^k=\operatorname{\small{AGGREGATE}}_k(\mathbf{h}_{\mathcal{B}^{k+1}}^{k-1})
\newline
&\mathbf{h}_{\mathcal{B}^k}^k=\sigma(\mathbf{W}\cdot\operatorname{\small{CONCAT}}(\mathbf{h}_{\mathcal{B}^k}^{k-1},\ \mathbf{h}_{\mathcal{N}(\mathcal{B}^k)}^k))
\end{aligned}
$$
由于 $\mathbf{h}_{\mathcal{B}^{k+1}}^{k-1}$ 形如 $L\times|\mathcal{B}^{k+1}|$，可以通过变形转换为 $L\times n\times|\mathcal{B}^k|$，将聚合函数 $\operatorname{\small{AGGREGATE}}_k$ 对张量每 $n$ 个一组批量聚合，即可得到邻居节点形如 $L\times|\mathcal{B}^k|$ 的聚合特征表示。将该聚合特征 $h^k_{\mathcal{N}(\mathcal{B}^k)}$ 与当前批自身特征 $\mathbf{h}_{\mathcal{B}^k}^k$ 结合，再与特征矩阵 $\mathbf{W}$ 相乘即可。

</tex>

### 实现

> 整这么多数学公式，咱也挺晕的，不如直接上代码吧！

首先实现 `GraphSampler` 采样层，对输入的批 $\mathcal{B}^0$ 进行采样，产生 $\mathcal{B}^1\cdots\mathcal{B}^K$。输入 `x` 即为 $\mathcal{B}^0$，包含当前批所有节点的唯一标识。我们会循环 $K$ 次，依次生成 $\mathcal{B}^1\cdots\mathcal{B}^K$（即 `ids`） 与 $\mathbf{h}^0_{\mathcal{B}^0}\cdots\mathbf{h}^0_{\mathcal{B}^K}$（即 `layers`）。

```julia
import Zygote
using StatsBase: sample

struct GraphSampler
    K::Integer
    n::Integer
    
    GraphSampler(; K::Integer, n::Integer) = new(K, n)
end

function (g::GraphSampler)(x::AbstractArray)
    Zygote.ignore() do
        ids = x
        layers = [data.node_features[:, ids]]
        for k in 1:g.K
            ids = vcat([sample(
                data.adjacency_list[u], g.n,
                replace=length(data.adjacency_list[u]) < g.n
            ) for u in ids]...)
            push!(layers, data.node_features[:, ids])
        end
        
        layers
    end
end
```

由于采样层没有进行任何参数运算，因此也不需要微分，我们将整个层用 [`Zygote.ignore`](https://fluxml.ai/Zygote.jl/latest/utils/#Zygote.ignore) 包裹起来，表示不要对这一段代码进行自动微分（auto differentiation）。

其次实现 `MeanAggregator` 聚合层，我们这里采用平均聚合。由于每个聚合层只进行一次聚合，最终图神经网络中应该有 $K$ 个聚合层。第 $k\in1\cdots K$ 个聚合层的输入为 $\mathbf{h}^{k-1}\_{\mathcal{B}^0}\cdots\mathbf{h}^{k-1}\_{\mathcal{B}^{K-k+1}}$，输出为 $\mathbf{h}^k_{\mathcal{B}^0}\cdots\mathbf{h}^k_{\mathcal{B}^{K-k}}$。因此，当前批 $\mathcal{B}^0$ 通过一个采样层与 $K$ 个聚合层后会输出 $h^K_{\mathcal{B}^0}$。

```julia
struct MeanAggregator
    w::Dense
    
    MeanAggregator((in, out)::Pair{<:Integer, <:Integer}, σ::F) where {F} = new(Dense(in * 2 => out, σ))
end

function (m::MeanAggregator)(layers::Vector{Matrix{Float32}})
    [
        cat(
            layers[i],
            layers[i + 1] |>
                x -> reshape(x, (size(layers[i], 1), :, size(layers[i], 2))) |>
                x -> mean(x, dims=2)[:, 1, :],
            dims=1
        ) |> m.w
        for i in 1:length(layers) - 1
    ]
end

Flux.@functor MeanAggregator
```

最终图神经网络结构定义如下：

```julia
model = Chain(
    GraphSampler(K=2, n=8),
    MeanAggregator(1433 => 512, relu),
    MeanAggregator(512 => 256, relu),
    x -> x[1],
    Dense(256, 7),
    softmax
)
```

在其他设置均不改变的情况下，训练 33 轮后停止，训练集精确度 100%，测试集精确度 78.2%，较原来精确度提升超过 20%。这个结果还是比较令人满意的。

完整代码放在这个 Jupyter Notebook [上面](https://gist.github.com/queensferryme/1526acfd04a29b86fbeeaf5c09db17e1)，感兴趣的朋友可以看看。

---

[^1]: Hamilton, W., Ying, Z., & Leskovec, J. (2017). Inductive representation learning on large graphs. *Advances in neural information processing systems*, *30*.
[^2]: https://relational.fit.cvut.cz/dataset/CORA
