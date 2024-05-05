---
title: Diffusion Model for Video Generation——随想
author: zssloth
date: 2024-05-05 18:00:00 +0800
categories: [Machine Learning] # only level-2 supported
tags: [diffusion-model] # TAG names should always be lowercase
toc: true # table of contains, default to true
math: true
---

> 关于 diffusion-based Video Generation的一些随想



视频生成相对图像生成，主要的挑战：

- 在空间维度的基础上，增加了时间维度（temporal dimension）上不同时间帧（frames）需要保证连贯性和一致性的需求。因此，模型需要具备有处理时间和空间维度信息以及时间-空间跨域的信息融合能力
  - 当前图像生成的diffusion model，仅有spatial dimension的信息融合能力
- 相对于图像，高质量的视频数据相对而言更少
- （高分辨率、高帧率的）长视频生成，生成的视频序列越长，对一致性的要求就约严格
  - 计算量（算力）需求也要要高很多



从目前的技术路线上看，基于diffusion model的视频生成，有如下的技术讨论：



## 训练策略

已有方案大体基于一个已经训练好的image diffusion model，通过增加额外的temporal attention/3D conv layers来捕捉frame之间的一致性。已经训练好的模型的权重可以frozen 来减少模型fine-tuning时间。**Video LDM** ([Blattmann et al. 2023](https://arxiv.org/abs/2304.08818)) 中设计的模型结构如下：

![video-LDM](https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/video-LDM.png)

新增的temporal layer交织地添加在原始的spatial layer之后，注意右图tensor shape的变换是如何处理时间维度 $$t$$​ 的。此外，原始的VAE仅在图像上训练，当对video UNet模型输出的时间上多帧的latent做decode时会出现闪烁伪影(flickering artifacts)，因此为decoder添加了额外的temporal layers在视频数据上做微调：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20240505165304016.png" alt="image-20240505165304016" style="zoom: 80%;" />

如何获得高分辨率+高帧率的视频：

- 在latent space先生成少量关键帧，再对这些关键帧进行插帧得到高帧率的latent
  - 插帧用的是masking-conditioning，在模型训练阶段有个auxiliary task让模型学习做这个事情
- 解码后在pixel space做超分得到高分辨率的结果

![image-20240505170356626](https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20240505170356626.png)

早期的做法类似上述流程，似乎都是将temporal和spatial 维度的upsampling过程以cascade的形式挨个搞，其他诸如下图**Imagen Video** ([Ho, et al. 2022](https://arxiv.org/abs/2210.02303)) 中的实现思路。

![imagen-video](https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/imagen-video.png)

近期发表的论文，已经慢慢趋于用一个统一的E2E模型完成。比如 **Lumiere** ([Bar-Tal et al. 2024](https://arxiv.org/abs/2401.12945)) 中提出的 **space-time U-Net (STUNet)** ，还有最近大家讨论较多的SoRA，均是如此。

![image-20240505163143057](https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20240505163143057.png)

![image-20240505163825593](https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20240505163825593.png)

Overall framework of [Sora](https://arxiv.org/pdf/2402.17177):

![image-20240505171201444](https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20240505171201444.png)

### **数据集**

高质量、多样的text-video标注数据对效果提升的正向影响

## **模型结构的选取：U-Net vs. DiT**

U-Net是imgae diffusion model目前最为主流的模型结构，但最近的Sora和Stable Diffusion 3.0都选用Transformet-Based DiT作为模型基本的骨架，看起来似乎是个趋势，背后的逻辑是什么？

看一下主流stable diffusion model中UNet的模型结构：

![base](https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/base.png)

几个问题：

- 模型包括多个down-sampling stage和up-sampling stage，靠近中间 (low-resolution) 的stage在不少做model compression的论文中claim对效果影响是较小的（参考[MibileDiffusion](https://arxiv.org/abs/2311.16567) 和 [BK-SDM](https://github.com/segmind/distill-sd)），金字塔形式的结构在直觉上容易实现不同抽象粒度上特征的融合，对于diffusion model来说本身已经在latent space，哪个分辨率上对结果影响最大
  - 从已有工作的分析，UNet中主干网络和残差分支在diffusion过程中起到的作用是不一样的：根据[《FreeU: Free Lunch in Diffusion U-Net》](https://papers.cool/arxiv/2309.11497)的讨论，U-Net中残差分支主要负责添加高频细节，主干部分则主要负责去噪。
  - **是否保留UNet骨架里这种长跳连的信息流形式就行了？**
  - ~~如此对于主要的去噪这个任务来说，是否U-Net中的残差连接可以认为并不起“关键”作用？~~
- 模型里其实每个resolution stage的block都是resnet+attention的组合，那么这里面resent重要么？
  - 由于resnet block中conv天生respective field是受限的，因此只能做局部的图像信息融合，对全局语义理解的帮助可能有限？
- 每个resent的输出送如attention block时，都需要先做一次转换将NCHW转换为N(HW)C，也就是说对于attention block而言，sequence length大小为HW，是将整个spation dimension上的图像按pixel完全展开，这种spatial conv+attention的信息融合方式相比于DiT中切patch的方式相比，哪种高效？

使用DiT作为基本的骨架的building block的潜在好处：

- 更强的scaling能力：类似于LLM，如果可以通过block的堆叠持续提高模型能力，UNet那种金字塔型的结构似乎没那么重要
- 更强的多模态能力？

附：DiT block结构与U-VIT

![image-20240505175743649](https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20240505175743649.png)

附：Stable Diffusion 3.0中采用的基于DiT的backbone：

![image-20240505175449464](https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20240505175449464.png)

## 参考材料

- https://lilianweng.github.io/posts/2024-04-12-diffusion-video/
- [Sora: A Review on Background, Technology, Limitations, and Opportunities of Large Vision Models](https://arxiv.org/abs/2402.17177)
- https://stability.ai/news/stable-diffusion-3-research-paper
- [Align your Latents: High-Resolution Video Synthesis with Latent Diffusion Models](https://arxiv.org/abs/2304.08818)