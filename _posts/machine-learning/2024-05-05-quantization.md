---
title: 量化论文技术总结
author: zssloth
date: 2024-05-05 15:00:00 +0800
categories: [Machine Learning] # only level-2 supported
tags: [model-compression] # TAG names should always be lowercase
toc: true # table of contains, default to true
math: true
---
> 总结量化相关论文技术细节

#### 总纲

主要基于论文"A White Paper on Neural Network Quantization"

量化的几个主要参数：$(s,z,b)$，分别对应于scale (也叫step size)、zero point和量化位宽 bit-width

量化过程：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119220441848.png" alt="image-20220119220441848" style="zoom:80%;" />

其中，

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119220547745.png" alt="image-20220119220547745" style="zoom:80%;" />

反量化(de-quantization)过程：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119220534556.png" alt="image-20220119220534556" style="zoom:80%;" />

根据 $z$ 的取值可以分为对称(symmetric)和非对称(asymmetric)量化：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119220713694.png" alt="image-20220119220713694" style="zoom:80%;" />

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119220734999.png" alt="image-20220119220734999" style="zoom:80%;" />

一些需要特别考虑的量化层：element-wise, concatenation，一般需要先做反量化

推荐设置：weights使用对称量化，activation使用非对称量化，会有额外的zero point的计算但是可以fuse到bias项

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119221218278.png" alt="image-20220119221218278" style="zoom:80%;" /> 

##### Post training Quantization典型方案

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119221307540.png" alt="image-20220119221307540" style="zoom:80%;" />

**Quantization Range Setting**

对于高量化比特，用简单的min-max就可以，假设待量化的tensor是$V$：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119221548480.png" alt="image-20220119221548480" style="zoom:80%;" />

比较低的比特下用Mean square error (MSE)：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119221621222.png" alt="image-20220119221621222" style="zoom:80%;" />

求解上述公式可以用很多方法，比如grid search，框架的自动微分，closed-form analytical approximation等。

对于分类任务的logits层，我们主要关心softmax输出的相对大小，那些贡献不大的value比如负数对结果影响不大，因此range可以设置成减少large important logit values的quantization error：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119221909488.png" alt="image-20220119221909488" style="zoom:80%;" />

上面的方案都需要少量calibration set。在没有calibration set的情况下，可以用BN训练得到的normalization参数设置range，因为一般BN的参数是per-channel的，所以比较适用于per-channel量化：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119222122632.png" alt="image-20220119222122632" style="zoom:80%;" />

不同方案的量化误差对比：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119222146890.png" alt="image-20220119222146890" style="zoom:67%;" />

**Cross-Layer Equalization **(CLE)

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119221355470.png" alt="image-20220119221355470" style="zoom:80%;" />

对于depth-wise卷积，per-channel的range差别比较大，直接做per-layer量化会有较大的量化误差，其中一个减少这种误差的方式是做cross-layer equalization，思路是对于部分activation如relu，可以将做跨层的per-channel scale传递从而使得同一层内的不同channel的参数分布更加均匀：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119222520806.png" alt="image-20220119222520806" style="zoom:80%;" />

对于如何找到合适的scale值 $s_i$，有论文推导出一个optimal setting：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119222615037.png" alt="image-20220119222615037" style="zoom:80%;" />

最终收益：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119222711666.png" alt="image-20220119222711666" style="zoom:75%;" />

**Absorbing high biases **

做完CLE之后，会有些比较大的bias值，使得输出activation不同channel的数值差别比较大，这个情况下可以进一步将这部分bias数值absorb到下一层，需要选择合适的absorb的bias值大小保证这种absorb可以做cross-layer传播：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119223122294.png" alt="image-20220119223122294" style="zoom:80%;" />

bias值的确定需要calibration set，而且是个近似的过程，因此会有些误差，导致FP model精度降低，但是对per-layer的量化性能提升还是不错的：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119223417504.png" alt="image-20220119223417504" style="zoom:67%;" />

**AdaRound**

认为默认的round-to-nearest非最优并实验验证了，随机rounding可以获得比round-to-nearest更好的精度：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119223624584.png" alt="image-20220119223624584" style="zoom:75%;" />

提出可以通过优化找出怎么做rounding最合适，最初始的方程是优化量化rounding对最终loss的影响：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119223739733.png" alt="image-20220119223739733" style="zoom:80%;" />

对立面的公式做泰勒展开，假设模型收敛的情况下，可以忽略二阶信息；进一步假设没有cross-layer correlations，可以得到下面的quadretic unconstrained binary optimization (QUBO)：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119223942235.png" alt="image-20220119223942235" style="zoom:80%;" />

其中 $H$ 是block-diagonal的。求解上式是NP-hard，复杂度很高，需要进一步近似：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119224202140.png" alt="image-20220119224202140" style="zoom:80%;" />

最终只需要比较少的unlabeled数据就可以，在做极低比特量化时推荐使用，性能如下：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119224308913.png" alt="image-20220119224308913" style="zoom:75%;" />

目前典型PTQ精度：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119224353692.png" alt="image-20220119224353692" style="zoom:80%;" />



##### QAT典型pipeline

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119224434541.png" alt="image-20220119224434541" style="zoom:80%;" />

几个需要注意的点：

- per-channel量化来说，一开始的CLE就没啥必要了；量化range默认安装PTQ的MSE或者Min-Max方案都差不多；
- 推荐per-channel下面不要fold BN, per-layer下要fold BN；
- 量化range的设置主要是scalar和offset，建议设置成trainable的；初始化方案虽然在一开始fintune的时候会有性能差别，但会很快手链趋于一致；

性能：相比PTQ要好不少；需要注意的是默认不做mix-precision quantization.

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119224627075.png" alt="image-20220119224627075" style="zoom:75%;" />



#### Quantization-Aware Training

##### Learned Step Size Quantization (LSQ, ICLR20, IBM)

[Contributions] per-layer的weight和activation quantization，令量化的scale trainable，每层只需要增加两个scalar值，就可以通过QAT实现比较高的低比特模型精度。量化中scale的采用方法：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119195451392.png" alt="image-20220119195451392" style="zoom:60%;" />

**trainable scale怎么做梯度回传：**

Loss关于step-size的梯度推导，rounding用STE做近似梯度回传：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119195645291.png" alt="image-20220119195645291" style="zoom:80%;" />

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119195658627.png" alt="image-20220119195658627" style="zoom:80%;" />

与其他工作的对比：在transition point上梯度值是比较大的，符合直观预期，因某个值如果在量化bin切换点上稍微变一下就会导致自身量化后的值发生变化，因此对scale应该有比较大的影响（较大的梯度值）。

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119195730424.png" alt="image-20220119195730424" style="zoom:80%;" />

**提高训练鲁棒性**

提出step size gradient scale，其出发点是当model中所有的weight parameter都以差不多一致的average magnitude更新时，可以提高整个model的收敛性。因而，希望scale的更新与weight的更新有相似的update magnitude：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119200419789.png" alt="image-20220119200419789" style="zoom:60%;" />

这里的 $R$ 应接近于1，但默认scale的update magnitude会比较大，因此再反向梯度回传的时候额外进行了scale：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119200752088.png" alt="image-20220119200752088" style="zoom:80%;" />

其中 $N_W$ 是该层weight parameters的数目，$Q_P$ 是前面提到的量化range，测试确实会让 $R$ 的值趋近于1，并且实际测试通过gradient scaling可以获得更高的finetune精度。

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119200552671.png" alt="image-20220119200552671" style="zoom:80%;" />

因为前向的时候并没有加这个额外的scaling factor，因此这里也需要做个stop-gradient的trick：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119201903478.png" alt="image-20220119201903478" style="zoom:80%;" />

**其他实现细节：**

- 基于pre-trained model, cosine lr (step lr 精度稍差些但差别<0.5%)，初始每层的scale值初始化为  $2|v|/\sqrt{Q_P}$
- <8-bit的设置下，需要finetune 90个epoch
- first&last layer 8-bit

- 越低的量化比特下需要更小的weight decay，精度差别在0.5%左右，量化本身可以视为一种正则化手段
- 可以进一步结合distillation，最高可以由1.1%精度提升（尤其在低比特下）
- LSQ的训练方式，最后并没有minimize quantization error，启发我们思考最小化量化误差这个思路是否是正确的
- 补充：对于optimizer是adam这种可以自适应lr的，gradient scaling应该可以去掉

最终性能：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119202342694.png" alt="image-20220119202342694" style="zoom:60%;" />

##### LSQ+: Improving low-bit quantization through learnable offsets and better initialization

扩展完善了LSQ的工作，主要技术点包括：

**增加learnable offset：**

LSQ只支持symmetric quantization，但是像Swish这类的激活函数比较适合asymmetric quantization。因此，本论文中提出了learnable offsets的训练方式：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119204214161.png" alt="image-20220119204214161" style="zoom: 67%;" />

更新后的公式梯度计算如下：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119204255273.png" alt="image-20220119204255273" style="zoom:67%;" />

asymmetric quantization学习到的offset后面可以直接跟bias融合，因此不会有额外的计算：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119204349873.png" alt="image-20220119204349873" style="zoom: 80%;" />

**better scalar initialization**

作者认为LSQ中提出的scale初始化方式得到的初始值普遍跟最终训练到的scale值差别很大，初始值设置并不是最优的，针对weight，提出了基于统计的初始值设置方案，假设每层的weight服从近似的高斯分布，由此设置scale初值：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119204644616.png" alt="image-20220119204644616" style="zoom:67%;" />

针对activation，通过最小化layer-wise apply scale-offset前后L2 error的方式设置scale和offset初值：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119204820006.png" alt="image-20220119204820006" style="zoom:80%;" />

上述公式没有闭式解，因此通过少量数据+pytorch的autograd function计算更新得到。

实验评估：

1. offset增加带来的收益：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119205104728.png" alt="image-20220119205104728" style="zoom: 60%;" />

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119205312551.png" alt="image-20220119205312551" style="zoom: 60%;" />

2. scale和offset初始化方式带来的收益：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog/image-20220119205146971.png" alt="image-20220119205146971" style="zoom: 60%;" />

