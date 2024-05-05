---
title: 计算机体系结构——Instruction Set Architecture
author: zssloth
date: 2024-05-04 20:00:00 +0800
categories: [Computing, Computer Architecture] # only level-2 supported
tags: [computer-architecture] # TAG names should always be lowercase
toc: true # table of contains, default to true
math: true
# img_path: https://github.com/zssloth/zssloth.github.io/tree/main/_posts/computing/compute-arch
# typora-copy-images-to: ../../../_media/imgs/${filename}
---

### 定量分析基础

#### 性能度量的指标

- 延时 (Latency)
  - 响应时间 (response time) = CPU time + Waiting time (IO. scheduling, etc.)，也称Wall-Clock Time or Elapsed Time，指任务开始到任务结束所经历的时间
  - CPU执行时间：仅考虑CPU计算的时间

$$
\text{cpu time}= \frac{\text{seconds}}{\text{program}}=\frac{\text{instructions}}{\text{program}}\times \frac{\text{clock cycles}}{\text{instruction}}\times \frac{\text{seconds}}{\text{clock cycles}}
$$

其中，instructions/program 为instruction count (IC), clock cycles/instruction 为cycles per instruction (CPI), seconds/clock  cycles为clock cycle time (T)，因此CPU执行时间为：

```
CPU time = IC x CPI x T
```

IC取决于程序源码、编译技术和ISA；CPI取决于ISA和微架构设计，考虑大量指令执行时间的平均；T取决于微架构设计和芯片工艺。

执行时间是计算系统度量最实际、最可靠的方式。

- 吞吐率 (throughput) = 单位时间完成的任务数，类似于bandwidth
- 可靠性
  - Mean time to failure (MTTF)
  - Mean time to repair (MTTR)
  - Mean time between failures (MTBF) = MTTF + MTTR
  - Availability = MTTF / MTBF

#### 性能比较

#####  Amdahl's Law

$$
Speedup_{overal} = \frac{ExTime_{old}}{ExTime_{new}}=\frac{1}{(1-Fraction_{enhanced})+\frac{Fraction_{enhanced}}{Speedup_{enhanced}}}
$$

性能提升受限于任务重可加速部分所占的比例。

##### Benchmarks

Benchmarks指一组用于测试的程序用来系统比较计算机系统的性能，可以为: Kernels (e.g. matrix multiply)、Toy programs (e.g. sorting)、Synthetic benchmarks (e.g. Dhrystone)、Benchmark suites (e.g. SPEC06fp, TPC-C)。经典的desktop benchmarks为 [SPEC](https://www.spec.org/)，采用一组应用综合性能算几何平均作为性能的综合评价：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlogimage-20211025222321778-1714838107933-2-1714838217523-2.png" alt="image-20211025222321778" align=left style="zoom:40%;" />

基于几何平均的性质，性能评估结果与参考机器的选择无关：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlogimage-20211025222415523.png" alt="image-20211025222415523" align=left style="zoom:50%;" />

#### 功耗

随着摩尔定律的失效，功耗(能耗)越来越成为一个处理器设计的重要指标。几个有用的概念：

- 散热设计功耗（Thermal Design Power (TDP)），指在最糟糕、最坏情况下的功耗。散热解决方案的设计必须满足这种散热设计功耗；TDP是对散热系统提出要求，要求散热系统能够把CPU发出的热量散
  掉，TDP值一定比CPU满负荷运行时的发热量大一点；

- 功耗(Power) 指单位时间的能耗：1 Watt = 1 Joule / Second  
  - Energy (能耗) = Average Power x Execution Time

- 功耗由动态功耗和静态功耗组成
  - 动态功耗: Dynamic power $\propto$ capacitive load x Voltage$^2$ x Frequency Switched
  - 静态功耗: Static power $\propto$ Static current (漏电流) x Voltage
- 减少动态功耗的常用技术
  - 关闭不活动模块或处理器核的时钟 (Do nothing well)
  - Dynamic Voltage-Frequency Scaling (DVFS): 有些场景不需要CPU全力运行，降低电压和频率可降低功耗
  - 针对典型场景特殊设计 (Design for the typical case): 电池供电的设备常处于idle状态，DRAM和外部存储采用低功耗模式工作以降低能耗
    • Overclocking (Turbo Mode): 当在较高频率运行安全时，先以较高频率运行一段时间，直到温度开始上升至不安全区域; 一个core以较高频率运行，同时关闭其他核
- 减少静态功耗的技术
  - Power Gating: 通过切断供电减少漏电流
- EDP (Energy Delay Product): = Energy ∗ Delay = Power ∗ Delay$^2$  
- Energy Efficiency: FLOPS per watt

#### 并行计算架构的分类

分为SISD, SIMD, MISD, MIMO四种类型。其中MISD基本没有实现，有人宣称FPGA可以实现类MISD，不过也是个说辞而已。

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlog1-16422561074551.jpg" alt="1" align=left style="zoom:60%;" />



技术的进步与应用、软件兼容性之间的关系：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlogimage-20211028194301489.png" alt="image-20211028194301489" align=left style="zoom:50%;" />

### ISA

#### 基本概念

Instruction Set Architecture (ISA)是软件和硬件之间的抽象接口：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlogimage-20211028202600437.png" alt="image-20211028202600437" align=left style="zoom:50%;" />

ISA设计时会考虑特定的微体系结构(uArchitecture)实现方式，但理论上一个ISA可以由任何uArchitecture实现：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlogimage-20211028202741264.png" alt="image-20211028202741264" align=left style="zoom:40%;" />

##### 分类

早期分为几种，但现在基本为通用寄存器型 (General Purpose Register)，数据从memory load到register后再执行相关计算。

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlogimage-20211028210330379.png" alt="image-20211028210330379"  align=left style="zoom:60%;" />

register-register的优势和不足：

![image-20211028211531458](https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlogimage-20211028211531458.png)

比较明显的劣势是：1) 单个指令的信息密度低，完成相同操作所需指令的数目多；2) 所需寄存器资源较多

##### 寻址方式

有很多种，总结如下：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlogimage-20211028210532770.png" alt="image-20211028210532770" align=left style="zoom:60%;" />

因为存储器一般按照字节编址，而一个数据可能由多个字节表示，涉及到数据再存储时候的存储顺序和字节对齐，两个问题：

1. 尾端问题：大端 vs. 小端，小端将数据低字节放在低地址，大端反之

   <img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlogimage-20211028210810628.png" alt="image-20211028210810628" align=left  style="zoom:60%;" />

2. 对齐问题：要求数据访问按照一定字节对齐

**基于程序实测跑的数据，可以统计不同寻址方式使用频次：**

- 重要的寻址方式: 偏移寻址方式, 立即数寻址方式, 寄存器间址方式
  - SPEC测试表明，使用频度达到 75%--99%
- 偏移字段的大小应该在 12 - 16 bits, 可满足75%-99%的需求
- 立即数字段的大小应该在 8 -16 bits, 可满足50%-80%的需求

##### 操作和操作数类型

一般需要支持的操作类型：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlogimage-20211028211851792.png" alt="image-20211028211851792" align=left style="zoom:80%;" />

一般计算机都支持前三类所有的操作；不同计算机系统 对系统支持程度不同，但都支持基本的系统功能；对最后四类操作的支持程度差别也很大，有些机器不支持，有些机器还在此基础上做一些扩展，这些指令有时作为可选的指令。

基于实际应用的统计，需要的操作数类型和大小为：

- 对单字、双字的数据访问具有较高的频率
- 支持64位双字操作，更具有一般性

##### 指令编码

可以是定长(fixed)、变长(variable)和混合编码(hybrid)三种类型，一般的选择依据为：

- 代码规模更重要，则选择变长指令格式；性能重要则使用固定长度指令

##### 功能设计

ISA功能设计是基于堆计算负载统计确定硬件要支持那些操作的过程，粗劣分为两种类型：

- CISC（Complex Instruction Set Computer）
  - 目标：强化指令功能，通过减少运行的指令条数提高系统性能
  - 方法：面向目标程序的优化，面向高级语言和编译器的优化，一般先统计分析目标程序执行情况，找出使用频度高，执行时间长的指令或指令串，以此优化ISA设计

- RISC（Reduced Instruction Set Computer）
  - 目标：通过简化指令系统，用高效的方法实现最常用的指令
  - 方法：充分发挥流水线的效率，降低（优化）CPI

RISC计算特点的一般描述(可以看出基本为了优化CPI而设计)：

- 大多数指令在单周期内完成
- 采用Load/Store结构
- 硬布线控制逻辑
- 减少指令和寻址方式的种类
- 固定的指令格式
- 注重代码的优化
- 面向寄存器结构
- 十分重视流水线的执行效率－尽量减少断流
- 重视优化编译技术

一些典型的ISA：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlogimage-20211028213247831.png" alt="image-20211028213247831" align=left style="zoom:50%;" />

##### 编译器的重要性

编译器将高级语言翻译成目标机器代码，本身包含多种代码优化手段，因此设计有利于编译器的ISA也很重要，通用设计原则：

- 有利于编译器的ISA
  - 规整性和正交性:  没有特殊的寄存器，例外情况尽可能少; 所有操作数模式可用于任何数据类型和指令类型,要求操作、数据类型和寻址方式必须正交，以简化代码生成过程。
  - 完整性:  支持基本的操作和满足目标的应用系统需求
  - 帮助编译器设计者了解各种代码序列的执行效率和代价，有助于编译器的优化
  - 对于在编译时就已经可确定的量，提供能够将其变为常数的指令
- 寄存器分配是关键问题
  - 寄存器数目多有利于编译器的设计与实现
  - 提供至少16个通用寄存器和独立的浮点寄存器 保证所有的寻址方式可用于各种数据传送指令
- 最小指令集

目前已有的一些研究结论：

- 大量优化可以减少存储器访问频次和部分ALU操作，而优化分支语句较困难，使得控制类指令再统计上占较大比例，难以加速

#### 实例：RISC-V ISA

设计理念：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlogimage-20211028214024223.png" alt="image-20211028214024223" align=left style="zoom:40%;" />

技术目标：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlogimage-20211028214105974.png" alt="image-20211028214105974" align=left style="zoom:35%;" />

参考：https://riscv.org/

##### 微程序设计(micro programming)实现

micro programming是80年代以前较为流行的一种处理器实现方案，处理器设计可分为datapath和control两个部分，早期设计中控制逻辑的实现较为复杂，后面提出了micro programming的概念设计处理器的控制逻辑：

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlogimage-20211028215402319.png" alt="image-20211028215402319" align=left style="zoom:40%;" />

早期ROM比RAM速度快很多，普遍将微指令保存在ROM中，复杂指令/指令修改不需要修改数据通路，成为主流实现方案。

<img src="https://cdn.jsdelivr.net/gh/zssloth/image-resource@main/githubBlogimage-20211028215735354.png" alt="image-20211028215735354" align=left style="zoom:40%;" />

随着技术的进步，SRAM在速度和性价比上超过了ROM，且RAM支持修改，该实现方案也被逐渐替代，不过在现代处理器中也有相关的功能用于，e.g.，芯片出厂后的bug fixing。
