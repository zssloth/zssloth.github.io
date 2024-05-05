---
title: Masked Autoencoders Are Scalable Vision Learners
author: zssloth
date: 2024-05-05 15:00:00 +0800
categories: [Paper Reading] # only level-2 supported
tags: [论文笔记, computer-vision] # TAG names should always be lowercase
toc: true # table of contains, default to true
math: true
---

- Masked Autoencoders Are Scalable Vision Learners  

生成式自监督学习近期(2021.11)讨论较多的一个工作，主要贡献是讨论了masked autoencoding (like MLM in NLP)在CV和NLP中的不同，并提出了有效的利用masked autoencoder训练vision model的自监督学习方案。

CV和NLP中masked autoencoding任务的几个不同（导致NLP领域的masked autoencoder不能直接用于CV）：

1. 网络结构上的差异：经典CV模型主要基于regular grid的卷积操作，没有直观的实现类似于mask tocken的方式，但近期vision transformer上的一系列进展，将image切生patch tocken的方式解决了该问题；
2. 信息密度上的差异：NLP是人类生成的信号，具有很高的语义信息和信息密度，而图像这种自然信息具有很高的空间冗余性；
   - 因而对于图像，更高的mask比例可以引导模型学习图像中high-level信息

3. CV和NLP的decoder部分作用是不同的：NLP中预测masked的word具有比较丰富的语义信息，而图像重建的是pixel-level信息，相对low level，因此decoder本身的建模能力会比较大的影响前面encoder部分的学习到的latent representation的semantic level

具体实现细节：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20211120005849194.png" alt="image-20211120005849194" style="zoom:80%;" />

- encoder基于VIT，对输入也是切成patch，训练时按照一定比例drop掉一些patch（不是mask而是直接drop，输入tocken长度直接变短）；decoder的输入则会把masked token加回去，并且加上positional encoding；

  - decoder设计的比较小，计算量是encoder的<10%
  - mask比例与最终性能（在下游任务上的性能）

  <img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20211120010256426.png" alt="image-20211120010256426" style="zoom:80%;" />

  - autoencoder训练时采用MSE loss，而且值算masked patchs部分的MSE loss (similar to BERT)
  - random mask的性能最好

- 部分自变量的实验结果：

  <img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20211120010621648.png" alt="image-20211120010621648" style="zoom:80%;" />

  - 随着epoch的训练性能还是不断上升，不容易过拟合

  <img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20211120010841468.png" alt="image-20211120010841468" style="zoom:80%;" />

  Open Discussion：

  - image和video确实都带有不少冗余信息，基于输入数据的信息冗余情况，怎么做input-dependent的模型轻量化？除了adaptive computing是否还有其他实现层面更为友好的思路。