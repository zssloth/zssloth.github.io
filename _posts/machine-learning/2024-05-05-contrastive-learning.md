---
title: 对比学习概述
author: zssloth
date: 2024-05-05 15:00:00 +0800
categories: [Machine Learning] # only level-2 supported
tags: [unsupervised-learning] # TAG names should always be lowercase
toc: true # table of contains, default to true
math: true
---
对比学习是无监督学习的一种范式，希望利用无标注的数据本身作为监督信息学习样本数据的特征表达，并用于下游任务。当前无监督学习大致可以分为两类：

- Generative Methods
- Contrastive Methods 

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/v2-4b8e593f5911ae30e6e217628924ae3c_720w.jpg" alt="img" style="zoom: 60%;" />

Generative Methods（生成式方法）以VAE、GAN为代表，关注像素级的重构，重构的效果比较好则说明模型学到了比较好的特征表达，而重构的效果通过pixel label的loss来衡量。

Contrastive Methods（对比式方法）通过自动构造正例和负例样本，要求训练一个表示学习模型，通过这个模型，使得正例样本在投影空间中比较接近，而与负例在投影空间中距离比较远；相比起Generative Methods需要对像素细节进行重构来学习到样本特征，Contrastive Methods只需要在特征空间上学习到区分性。因此Contrastive Methods不会过分关注像素细节，而能够关注抽象的语义信息，并且相比于像素级别的重构，优化也变得更加简单。其一般学习范式为：

对任意数据 $x$，对比学习的目标是学习一个编码器$f$使得：
$$
score(f(x),f(x^+) >> score(f(x), f(x^-)))
$$
其中$x^+$是和$x$相似的正样本，$x^-$是和$x$不相似的负样本，$score$是一个度量函数来衡量样本间的相似度。

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/v2-7f0cb4f5a90df300585de6e24a516bad_720w.jpg" alt="img" style="zoom:50%;" />

**几个技术点：**

1. 模型训练策略(loss的设计)，牵引正例样本在投影空间中比较接近，而与负例在投影空间中距离比较远；隐含的一个技术点是如何在特征空间度量不同样本间的相似度

比较经典的contrastive loss：

- Triplet Loss

三元组损失，最小化锚点(anchor)和具有相同身份的正样本之间的距离，最小化锚点和具有不同身份的负样本之间的距离。

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/v2-f3f7681539639a56e368f565cb5167b6_720w.jpg" alt="img" style="zoom:50%;" />

对于一个三元组$(x,x^+,x^-)$，定义其优化的损失为：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20211114220455946.png" alt="image-20211114220455946" style="zoom:80%;" />

$d(\cdot)$可以使用L2范数及欧式距离，是线性空间中常用的相似度表示。该损失使相同标签的特征在空间位置上尽量靠近，同时不同标签的特征在空间位置上尽量远离。同时为了不让样本的特征聚合到一个非常小的空间中（防止模型坍塌(Model Collapse)），要求对于同一类的两个正实例和一个负实例，负例应该比正例的距离至少为margin值$\alpha$。

- InfoNCE Loss

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20211114220800103.png" alt="image-20211114220800103" style="zoom:80%;" />

用向量内积来计算两个样本的相似度。对应样本$x$有1个样本和N-1个负样本。可以发现，这个形式类似于交叉熵损失函数，学习的目标就是让$x$的特征和正样本的特征更相似，同时和N-1个负样本的特征更不相似。上述公式中相似度计算可以进一步引入温度$\tau$，通过控制$\tau$控制loss对正负样本差距的敏感度：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/v2-8468c5a6b359716e13d8da063a09e5c0_720w.jpg" alt="img" style="zoom:80%;" />

除了向量点积，也可以使用向量间的cosine相似性衡量两个样本的相似度：
$$
S(x_i,x_j) = z^T_i\cdot z_j/(||z_i||_2|\cdot |z_j||_2)
$$
其背后逻辑是先将向量映射到单位超平面，再进行距离度量，可以增加模型训练稳定性。

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/v2-1e089e04890db88c8239ae2bbdf78616_720w-163689976453716.jpg" alt="img" style="zoom:80%;" />

2. 如何构造正例和负例

CV和NLP领域各自由构造正例和负例的思路，这里以CV领域为例，常见做法是对某张图片经由不同数据增强操作得到的增强后图片相互之间视为正例，而在模型训练阶段，同一个Batch data中其余的图片数据视为负例，相关工作为SimCLR系列文章。也有其他做法，大致总结如下：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/v2-379c0aed15b25149c47f27b66aa1f5e4_720w.jpg" alt="img" style="zoom:80%;" />

以SimCLR为例，其训练过程如下图所示：

![img](https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/v2-9bb6848d5b07642dae063acaa9225e71_720w.jpg)

注意有encoder和projector两个部分，也就是说encoding得到的特征空间向量是经过projector后再计算相似度。非线性变换结构Projector由[FC->BN->ReLU->FC]两层MLP构成。SimCLR论文中claim了这个projector的必要性。对于Projector和Encoder的编码差异进行了对比实验，结论是：Encoder后的特征表示，会有更多包含图像增强信息在内的细节特征，而这些细节信息经过Projector后，很多被过滤掉了。太多的细节特征可能会影响关键特征间相似度的比较。另外，负样本的数量也比较直接影响对比学习的性能（越多越好？）。

相关的工作包括MoCo、SimCLR、BYOL、SwAV、DeepCluster-v2、SimCSE等，此处不再展开。对比学习的思想也不仅用在自监督学习领域，可以用于提升其他很多任务的性能。

局限性：CV领域的constrastive learning算法实现方案比较依赖data augmentation

[参考资料]

- [对比学习综述](https://helicqin.github.io/2020/12/26/%E5%AF%B9%E6%AF%94%E5%AD%A6%E4%B9%A0%E7%BB%BC%E8%BF%B0/)
- 知乎专栏：[ref1](https://zhuanlan.zhihu.com/p/346686467)、[ref2](https://zhuanlan.zhihu.com/p/367290573)、[ref3](https://zhuanlan.zhihu.com/p/141141365)