---
title: "Julia + Flux.jl（一）：从 MNIST 开始"
date: 2021-04-30T18:23:49+08:00
tags:
  - julia
  - machine learning
---

> 这篇博客可能是一系列博客的第一篇。

三月中旬时，我突发奇想想参加 [GSoC](https://summerofcode.withgoogle.com/)，并且选定了 [Julia](https://julialang.org/) 语言下属 [Flux.jl](https://github.com/FluxML/Flux.jl) 组织里的一个小项目。但问题在于，我此前对 Julia 语言只有耳闻，而从未实际学习过 —— 留给我的时间并不宽裕，GSoC 申请截止到四月十四日。我在这不到一个月的短暂时间内学习了 Julia 语言的基本用法，使用 Flux.jl 做了一些有趣的小实验（本篇博客所记录的就是其中之一），还给 Flux.jl 贡献了一个 [Pull request](https://github.com/FluxML/Flux.jl/pull/1545)。这样的结果让我颇为满意。

Julia 语言是一门专为科学计算设计的高性能的动态类型语言，Flux.jl 则是使用纯 Julia 语言编写出的机器学习框架。Julia 语言最吸引我的地方就在于，它切实地解决了很多痛点需求：

- 专为科学计算设计：从数字类型系统的设计，到诸如 $\LaTeX$ 变量名（`x\hat` => `x̂`）和复合函数（`f ∘ g`）这些语法糖，都可以感受到这门语言对科学家群体释放出的友好信号。
- 高性能：Julia 是一种编译型语言，这为它带来了极高的性能。也正是因此，Flux.jl 这种基于 Julia 的机器学习框架无需像 [PyTorch](https://pytorch.org/)、[TensorFlow](https://www.tensorflow.org/) 等一样，依靠 C++ 这种第三方静态编译语言来提高性能。
- 动态类型：尽管是一门编译型语言，Julia 同时也是一门动态类型语言 —— 我相信这对数据科学家来说是至关重要的（参见 [Exploratory Data Analysis](https://en.wikipedia.org/wiki/Exploratory_data_analysis)）。尽管前有 [goplus](https://github.com/goplus/gop) 这种项目和「有追求的科学家都用 Rust[^1]」这种暴论，但我私以为 Go 和 Rust 静态类型的本质仍阻止了其在数据科学领域的广泛应用（其他科学计算的子领域我几无了解，因此不予评论）。在分析数据时，我们最不想关心的事情就是与编译器的类型系统搏斗，而导致思路频频被打断。此外，Julia 语言生态圈还有 [IJulia](https://github.com/JuliaLang/IJulia.jl) 和 [Pluto.jl](https://github.com/fonsp/Pluto.jl) 这类项目为 EDA 提供了良好的支持。

本文将展示如何使用 Julia + Flux.jl 来解决机器学习界的 Hello World 问题 —— 识别 MNIST 手写数字！

<!--more-->

## 准备工作

本文假设你已经安装好了 Julia。如果你没有，请阅读 [Platform Specific Instructions for Official Binaries](https://julialang.org/downloads/platform/#platform_specific_instructions_for_official_binaries)。

本文同样假设读者对 Julia 语言和神经网络有基本的了解。

打开 Julia REPL，输入 `]` 进入包管理模式，然后安装 Flux.jl、MLDatasets 和 Pluto。

```
julia> ]
(v1.6) pkg> add Flux MLDatasets Pluto
```

安装完以后，运行 Pluto：

```julia
julia> import Pluto
julia> Pluto.run()
```

Pluto 会使用你的默认浏览器打开 <http://localhost:1234>，这是一个交互式的编程环境，我们将在这里完成剩下的工作。

## Dense NN

我们先尝试使用简单的全连接神经网络来识别手写数字。

使用 `MLDatasets` 载入 MNIST 数据集（分为训练集和测试集）：

```julia
using MLDatasets: MNIST

train_x, train_y = MNIST.traindata()
test_x, test_y = MNIST.testdata()
```

由于是使用全连接神经网络，我们需要将 `train_x` 和 `test_x` 中的 $28 \times 28$ 的二维图片拍扁成一维向量：

```julia
train_x = reshape(train_x, (28 * 28, :))
test_x = reshape(test_x, (28 * 28, :))
```

此外还需要对 `train_y` 和 `test_y` 进行 one hot 编码。Flux.jl 为此提供了方便的 `onehotbatch` 函数，用于进行批量 one hot 编码。

```julia
using Flux

train_y = Flux.onehotbatch(train_y, 0:9)
test_y = Flux.onehotbatch(test_y, 0:9)
```

最后，我们将训练数据集切分成多个 batch，方便之后进行 mini-batch 训练。

```julia
train_batched = Flux.Data.DataLoader((train_x, train_y), batchsize=32, shuffle=true)
```

到这一步数据已经基本处理完毕。接下来我们定义一个简单的全连接神经网络：网络输入层接受一个 $28 \times 28$ 的一维向量，即一张手写数字图片；中间有一个 32 维的隐藏层，使用 ReLU 激活函数；最后输出层有 10 维，分别代表输入图片为 0～9 这十个数字的概率。

```julia
model = Chain(
    Dense(28 * 28, 32, relu),
    Dense(32, 10),
    softmax,
)
```

想要训练这个神经网络，我们还需要一个损失函数，用来评判当前模型的优劣与求导更新参数。我们这里使用交叉熵。

```julia
loss(x, y) = Flux.crossentropy(model(x), y)
```

同时还可以定义一个 accuracy 函数，用于查看模型分类结果的准确率。由于模型的输出、`train_y` 和 `test_y` 都已被 one hot 编码，我们应使用 `onecold` 函数将其转换回 0～9 的数字。

```julia
using Statistics: mean

acc(x, y) = mean(Flux.onecold(model(x)) .== Flux.onecold(y))
```

最后定义一个优化器（optimizer），用于更新模型参数。这里我们使用最简单的梯度下降。

```julia
optimizer = Descent()
```

然后开始训练 10 个 epoch：

```julia
Flux.@epochs 10 begin
    Flux.train!(loss, params(model), train_batched, optimizer)
    @show acc(test_x, test_y)
    @show loss(test_x, test_y)
end
```

输出如下：

```
[ Info: Epoch 1
acc(test_x, test_y) = 0.9375
loss(test_x, test_y) = 0.20818216f0
[ Info: Epoch 2
acc(test_x, test_y) = 0.9532
loss(test_x, test_y) = 0.15715753f0
[ Info: Epoch 3
acc(test_x, test_y) = 0.9583
loss(test_x, test_y) = 0.14423494f0
[ Info: Epoch 4
acc(test_x, test_y) = 0.9611
loss(test_x, test_y) = 0.13178527f0
[ Info: Epoch 5
acc(test_x, test_y) = 0.9608
loss(test_x, test_y) = 0.12726997f0
[ Info: Epoch 6
acc(test_x, test_y) = 0.9637
loss(test_x, test_y) = 0.13048573f0
[ Info: Epoch 7
acc(test_x, test_y) = 0.9646
loss(test_x, test_y) = 0.124466665f0
[ Info: Epoch 8
acc(test_x, test_y) = 0.9672
loss(test_x, test_y) = 0.1138981f0
[ Info: Epoch 9
acc(test_x, test_y) = 0.9654
loss(test_x, test_y) = 0.1157888f0
[ Info: Epoch 10
acc(test_x, test_y) = 0.9671
loss(test_x, test_y) = 0.11115624f
```

可以看到第 8 轮训练以后模型已经基本收敛，最高准确率达到 96.72%，每轮训练平均耗时不到 10s。

## LeNet

简单的全连接神经网络已经展现出了良好的效；如果我们尝试更强大的卷积神经网络，又会得到怎样的结果呢？这里我们使用经典的 LeNet-5[^2] 卷积神经网络架构来实现 MNIST 手写数字分类任务。

与全连接网络不同的是，卷积神经网络可以接受二维向量作为输入。这也是卷积神经网络较全连接网络更擅长处理图像的原因：卷积神经网络能充分考虑图像在二维空间上的关系，而不是将图像简单粗暴地拍扁成一维向量处理。

```julia
train_x = reshape(train_x, (28, 28, 1, :)) .|> Float32
test_x = reshape(test_x, (28, 28, 1, :)) .|> Float32
```

`MLDatasets` 提供的数据形状为 $28 \times 28 \times 60000$，而 Flux 要求卷积层的输入数据形状应为 width $\times$ height $\times$ channel $\times$ batch，因此我们此处向数据插入一个大小为 1 的 channel 维度。由于 MNIST 数据集提供的是灰度处理过的图片，因此 channel 大小为 1；但通常来说图片的 channel 大小应为 3（RGB）。最后我们将 `train_x` 和 `test_x` 的数组元素类型转换为 Float32，有助于加快卷积层运算速度。

然后我们只需重新定义 `model` 为 LeNet-5 架构，就可以开始训练了：

```julia
model = Chain(
    Conv((5, 5), 1=>6, relu),
    MaxPool((2, 2)),
    Conv((5, 5), 6=>16, relu),
    MaxPool((2, 2)),
    flatten,
    Dense(256, 120, relu), 
    Dense(120, 84, relu), 
    Dense(84, 10),
    softmax,
)
```

训练结果如下：

```
[ Info: Epoch 1
acc(test_x, test_y) = 0.9777
loss(test_x, test_y) = 0.071477994f0
[ Info: Epoch 2
acc(test_x, test_y) = 0.984
loss(test_x, test_y) = 0.04953506f0
[ Info: Epoch 3
acc(test_x, test_y) = 0.9881
loss(test_x, test_y) = 0.036805704f0
[ Info: Epoch 4
acc(test_x, test_y) = 0.9858
loss(test_x, test_y) = 0.053402882f0
[ Info: Epoch 5
acc(test_x, test_y) = 0.9904
loss(test_x, test_y) = 0.0336451f0
[ Info: Epoch 6
acc(test_x, test_y) = 0.9892
loss(test_x, test_y) = 0.036136955f0
[ Info: Epoch 7
acc(test_x, test_y) = 0.9858
loss(test_x, test_y) = 0.046425987f0
[ Info: Epoch 8
acc(test_x, test_y) = 0.9906
loss(test_x, test_y) = 0.03592136f0
[ Info: Epoch 9
acc(test_x, test_y) = 0.9882
loss(test_x, test_y) = 0.040589117f0
[ Info: Epoch 10
acc(test_x, test_y) = 0.9904
loss(test_x, test_y) = 0.033295937f0
```

最高准确率达到 99.06%，每轮训练平均耗时约 20～30s。可以看到 LeNet 相较简单全连接网络确实有了较大的性能提升。

---

[^1]: 来源 https://twitter.com/laike9m/status/1334339942219014144，但原话并非 laike9m 所说。
[^2]: LeCun, Y., Bottou, L., Bengio, Y., & Haffner, P. (1998). Gradient-based learning applied to document recognition. *Proceedings of the IEEE*, *86*(11), 2278-2324.
