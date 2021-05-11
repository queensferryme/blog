---
title: "Julia + Flux.jl：向神经网络发起进攻！"
date: 2021-05-10T17:08:10+08:00
tags:
  - julia
  - machine learning
---

上回说到如何使用 Julia + Flux.jl 完成一个简单的 MNIST 手写数字识别任务。通过使用 LeNet 架构，我们的神经网络在测试集上已经能达到约 99% 的正确率 —— 看起来很棒不是吗？但实际上我们的神经网络对训练数据集有着极强的依赖性。只需对测试数据做加入一点肉眼甚至难以察觉的噪音，就足以使神经网络产生「失之毫厘，谬以千里」的结果。本文就将讨论如何使用 Fast Gradient Sign Method（FGSM）[^1] 扰动测试数据、从而对神经网络发起攻击。

神经网络很多时候在训练数据上保持着微妙而脆弱的平衡，这也在一定程度上佐证了「数据和特征决定了机器学习的上限，而模型和算法只是逼近这个上限而已」的观点。

<!--more-->

## 准备工作

本文假设你对 Julia 和神经网络有了基本的了解。但是你并不需要对 FGSM 有所了解 —— 与其强大程度相比，其基本思想显得出奇的简单。如果你不满足于仅仅只是学会使用 FGSM，还想要对其原理与动机有更深入的了解，我推荐你阅读原论文。

在开始之前，让我们先安装一些必须的 Julia 软件包：

```shell
julia> ]
(v1.6) pkg> add Flux MLDatasets # for model training
(v1.6) pkg> add ImageCore Plots # for visualization
(v1.6) pkg> add IJulia
```

在本次教程中，我决定使用 IJulia 来代替 Pluto 作为实验的交互式编程环境。IJulia 是基于 [Jupyter](https://jupyter.org/) 的，有过 Python 编程经验的同学对它应该会比较熟悉。就我个人而言，IJulia 相对 Pluto 的一大优势就在于我能直接在浏览器中看对标准输入输出，方便使用 print 调试大法 :)

```julia
import IJulia
IJulia.notebook()
```



## FGSM

要使用 FGSM 对神经网络发起攻击，我们首先需要有一个神经网络。你可以重复上一篇文章中的步骤，在现代 CPU 上应该只需几分钟就能训练好一个约 99% 精确度的 LeNet 模型。

```julia
# LeNet
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

FGSM 的基本思想在其名字中已经得到的完整的体现：我们首先需要求导获得模型 `model` 对某个输入 x 的梯度（gradient），然后取得梯度的符号（-1，0，+1），再然后将梯度的符号乘以扰动值 $\epsilon$ 得到一个噪音，最后将噪音与原始输入 x 相加即可得到被扰动的数据。代码如下：

```julia
function fgsm(image, ϵ, grad)
    grad_sign = sign.(grad)
    perturbed_image = image + ϵ * grad_sign
    clamp!(perturbed_image, 0, 1)
    return perturbed_image
end
```

由于这里我们的输入是一张手写数字图片，所以将变量命名为 `image`（而不是更一般的 `x`）。此外还要注意我们使用了 `clamp!` 函数，确保扰动过后图片每一个像素点的值仍在 [0, 1] 区间内。

函数 `fgsm` 的输入 `image` 可以是任意一至多张图片，形如 28 $\times$ 28 $\times$ 1 $\times$ `batch_size`；$\epsilon$ 则是用户指定的超参数，代表噪音的大小；而要求梯度 `grad` 也很简单。假设我们此前已经定义好了一个损失函数：

```julia
loss(x, y) = Flux.crossentropy(model(x), y)
```

此时整个模型可以被视作一个函数：此函数接受 x、y 作为输入，并输出一个 loss 损失值。要求模型对某个输入 x 的梯度，就是在求 $\dfrac{\partial\operatorname{loss}(x,y)}{\partial x}$。于是有：

```julia
grad_x, grad_y = gradient(loss, x, y)
```

>   如果你不知道 `gradient` 函数，请认真阅读 Flux [文档](https://fluxml.ai/Flux.jl/stable/models/basics/#Taking-Gradients)。

万事俱备，让我们发动攻击吧！我们将整个测试数据集 `test_x` 作为输入，生成一个被扰动的测试数据集 `perturbed_test_x`。

```julia
epsilons = 0:0.05:0.3

for ϵ in epsilons
    grad, _ = gradient(loss, test_x, test_y)
    perturbed_test_x = fgsm(test_x, ϵ, grad)
    println("ϵ=$ϵ, acc=$(acc(perturbed_test_x, test_y))")
end
```

输出为：

```
ϵ=0.0, acc=0.9898
ϵ=0.05, acc=0.9509
ϵ=0.1, acc=0.8576
ϵ=0.15, acc=0.7135
ϵ=0.2, acc=0.5722
ϵ=0.25, acc=0.4777
ϵ=0.3, acc=0.4013
```

可以看到，随着 $\epsilon$ 的增长，我们的模型在被扰动的测试数据上的准确率也逐渐下降，从 99% 左右下降到了 40% —— FGSM 真是太可怕啦！

我们也可以再添加亿点代码，做一下可视化：

```julia
using ImageCore
using Plots
using Random

epsilons = 0:0.05:0.3
plots = []

for ϵ in epsilons
    grad, _ = gradient(loss, test_x, test_y)
    perturbed_test_x = fgsm(test_x, ϵ, grad)
    println("ϵ=$ϵ, acc=$(acc(perturbed_test_x, test_y))")

    indices = collect(1:10000)
    # show successful attacks only
    if ϵ != 0
        indices = indices[Flux.onecold(model(perturbed_test_x)) .!= Flux.onecold(model(test_x))]
        shuffle!(indices)
    end

    for (i, index) in enumerate(indices[1:5])
        # original prediction
        image, target = test_x[:, :, :, index:index], test_y[:, index]
        prediction = Flux.onecold(model(image), 0:9)[1]
        # perturbed prediction
        grad, _ = gradient(loss, image, target)
        perturbed_image = fgsm(image, ϵ, grad)
        perturbed_prediction = Flux.onecold(model(perturbed_image), 0:9)[1]
        # visualization
        push!(plots, plot(
            MNIST.convert2image(perturbed_image[:, :, 1, 1]);
            ticks=false,
            xguide="$prediction -> $perturbed_prediction",
            yguide=i == 1 ? "ϵ=$ϵ" : ""
        ))
    end
end

plot(plots..., layout=(7, 5), size=(800, 1050), fmt=:png)
```

[![fgsm-mnist](https://i.loli.net/2021/05/11/d2ogMEDGhqpCmNa.png)](https://i.loli.net/2021/05/11/d2ogMEDGhqpCmNa.png)

午饭时间到了，今天的博客就水到这里，拜拜👋

---

[^1]: Goodfellow, I. J., Shlens, J., & Szegedy, C. (2014). Explaining and harnessing adversarial examples. *arXiv preprint arXiv:1412.6572*.

