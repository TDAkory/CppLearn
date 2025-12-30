# How to be Greater

#规划

> 分析补充个人技能栈，并记录需要学习的内容和学习进度
> <font color=red>[P0]</font>   <font color=orange>[P1]</font>   <font color=brown>[P2]</font> <font color=yellow>[Pending]</font> <font color=blue>[Doing]</font> <font color=green>[Done]</font>

## Overall Goals

* 技术能力：增强技术深度，有专精的技术领域，能够主导难题攻关；扩展技术广度，对相关的方面有较为全面的认识
* 领导力：可以清晰把握架构&演进方向，评估技术复杂度并合理拆解任务（组织团队高效开发）。追踪业界前沿进展、理解内部现状，能够识别差距并指定中长期技术规划
* 影响力：构建相关领域的影响力、内部外部的技术分享和输出、（相关领域的人才培养）
* 更进一步（行业专家）：深入研究、独到见解

## 🧭 核心原则：八年后的发展逻辑

工作8年后，规划不应再是“多学几门技术”，而是围绕**专业深度、行业影响力和资源杠杆**展开。同时，承认精力的有限性，将家庭作为战略目标而非约束条件。

核心思路是：**以当前工作时序数据库开发为“主战场”，将C++高性能计算作为攻克主战场的“核心兵器”，实现工作与学习的深度协同、相互成就。**

正确的策略是 **“在战略性深度中自然拓展有意义的广度”** 。你的深度是安身立命的基石，而广度应服务于加深你的深度、提升你的视野和影响力。

| 维度 | 对你而言的“深度” | 对你而言的“广度” |
| :--- | :--- | :--- |
| **定义** | 在**时序数据库存储与查询引擎**这一核心领域，达到能设计、优化、解决复杂线上问题的专家水平。 | 1. **垂直栈广度**：向下深入硬件（CPU缓存、SIMD、持久内存）、向上理解数据应用（分析场景、查询模式）。<br>2. **横向领域广度**：了解流处理、数据湖仓等相邻领域，知其优劣与融合趋势。 |
| **关系** | **深度是你的核心引擎和品牌**。它保证你在解决数据库内核问题时，能比90%的人想得更透、解决得更优雅。 | **广度是你的连接器和放大器**。它让你能将数据库与更广阔的云原生、数据架构结合，提出更整体的解决方案，从而创造更大价值。 |

1. **70/20/10时间分配模型**：
    * **70%时间**：聚焦当前工作直接相关的高性能优化任务（如用SIMD优化查询过滤）。
    * **20%时间**：探索与工作强相关的未来技术（如研究`Apache Arrow`的内存格式如何借鉴）。
    * **10%时间**：自由探索感兴趣的相邻领域（如看看游戏服务器或量化金融中的C++高性能应用，触类旁通）。

2. **创建“学习-应用”飞轮**：
    * **学习** -> **在实验项目中验证** -> **应用到工作** -> **复盘总结** -> **发现新学习点**。
    * 例如，学习完C++原子操作和无锁编程后，可以尝试优化一下数据库内部的并发计数器或缓存数据结构。

3. **打造个人技术品牌**：
    * 将你在工作中解决的高性能优化难题，以**案例分析、深度文章、内部分享**的形式输出。这既能巩固学习，又能建立你的专业影响力，直接助力职业发展。

### 📈 五年规划：成为领域中的关键节点

将未来五年视为从“资深开发者”向“领域专家/技术领导者”转型的关键期。

| 维度 | 五年愿景 (成为…) | 具体方向 |
| :--- | :--- | :--- |
| **技术能力** | 公司内外公认的**时序数据领域专家** | 1. **深度**：吃透1-2个顶级开源时序数据库内核。<br>2. **广度**：向数据栈上游（采集规范）下游（分析引擎）扩展。<br>3. **输出**：通过设计核心模块、专利、技术文章建立影响力。 |
| **职业发展** | **团队技术领袖**或**高价值个体贡献者** | 1. **路径选择**：明确适合你的技术管理（TL）或个人贡献者路线。<br>2. **决策影响**：从模块设计者晋升为产品技术方向的重要建议者。<br>3. **行业连接**：在技术社区建立连接，了解行业趋势与机会。 |
| **家庭生活** | **建立稳定、有弹性的家庭支持系统** | 1. **时间协议**：与家人商定可预测的工作时间与高质量的陪伴时间。<br>2. **健康储备**：制定家庭健康计划。<br>3. **财务规划**：为家庭阶段目标（如教育、置业）制定财务路线。 |

### **未来1-2年具体行动路径：**

| 时间线 | 工作结合点（主战场） | C++高性能计算学习重点（兵器库） | 目标产出与价值 |
| :--- | :--- | :--- | :--- |
| **第1年 (扎根)** | **存储引擎优化**：数据压缩、磁盘I/O、内存分配器、索引结构。 | 1. **现代C++特性**：熟练使用移动语义、智能指针、模板元编程编写高效、安全的内核代码。<br>2. **内存与缓存**：深入研究CPU缓存层次结构，优化数据布局，减少缓存失效。<br>3. **数据并行基础**：掌握AVX2等SIMD指令集，用于过滤、聚合等核心算子的加速。 **压缩与文件结构**：研究业界主流存储引擎的压缩方式和文件结构，分析其性能和成本的优势来源 | 主导1-2个存储层性能优化项目，性能指标显著提升，并沉淀为团队最佳实践或技术文章。 |
| **第2年 (拓展)** | **查询执行引擎**：向量化执行、并行聚合、连接算法、查询编译。 | 1. **并发与并行**：深入理解C++内存模型、无锁数据结构，优化多核并行执行。<br>2. **性能剖析**：精通Perf、VTune等工具，能进行系统级的瓶颈分析与调优。<br>3. **特定领域库**：学习并使用`std::execution`、`mimalloc`等高性能库的思想。 | 优化关键查询路径，提升复杂查询性能；能对团队代码进行高性能编码规范的指导和评审。 |
| **长期视野** | **系统架构**：参与或主导下一代架构设计，考虑存算分离、异构计算、云原生部署。 | 1. **异构计算**：了解GPU、FPGA等在数据库加速中的潜在应用。<br>2. **领域特定语言**：探索查询编译技术。<br>3. **社区影响力**：参与或借鉴开源项目。 | 成为团队在性能和架构方面的核心决策者之一，具备跨团队的影响力。 |

### 🗓️ 一年计划：从“执行者”到“所有者”的转变

接下来的一年，重心是**主动创造价值**和**建立个人标签**。建议使用OKR来管理。

| 维度 | 年度目标 (O) | 关键结果 (KR) |
| :--- | :--- | :--- |
| **技术能力** | **在时序数据库存储/查询性能优化上取得一项可量化的突破** | 1. 主导完成一个核心性能优化项目(数据压缩)，吞吐提升**20%** 或延迟降低**15%**。<br>2. 深入分析**1个**业界顶级时序库的存储引擎，产出内部报告。<br>3. 在团队/公司内进行**2次**技术分享，或在外部技术社区发表**1篇**深度文章。 |
| **职业发展** | **显著提升在关键项目中的技术主导权和能见度** | 1. 主动承担或发起一个跨模块/跨团队的项目，并成功交付。<br>2. 建立自己的“技术招牌”，例如成为团队内查询优化、压缩算法等领域的**第一联系人**。<br>3. 系统性建立**导师关系**（Mentorship），至少找到一位可深度请教的前辈。 |
| **家庭生活** | **与家人共同完成一件有仪式感的事，并建立稳定的家庭作息** | 1. 共同计划并完成一次**7天以上**的家庭旅行，全程投入。<br>2. 确立每周**至少两个晚上**为高质量家庭时间。<br>3. 共同制定一份家庭年度财务预算与储蓄计划。 |

### 💡 给资深工程师的特别建议

1. **杠杆你的经验**：你的价值不在于多写代码，而在于**解决复杂问题、避免重大技术债务、培养新人**。把时间投资在这些高杠杆率的事情上。
2. **打造“作品集”** 而非“简历”：专注于做出几个能体现你技术深度和行业理解的“代表作”（如关键系统设计、性能优化案例、高质量的技术分析）。
3. **平衡不是平均主义**：接受某些季度重点攻坚技术项目，某些季度则侧重家庭。提前与家人沟通好节奏，获得理解和支持。
4. **健康是核心资产**：将规律的锻炼、睡眠和饮食纳入计划，这是你能够同时应对高强度工作和家庭责任的基础。

- [How to be Greater](#how-to-be-greater)
  - [Overall Goals](#overall-goals)
  - [🧭 核心原则：八年后的发展逻辑](#-核心原则八年后的发展逻辑)
    - [📈 五年规划：成为领域中的关键节点](#-五年规划成为领域中的关键节点)
    - [**未来1-2年具体行动路径：**](#未来1-2年具体行动路径)
    - [🗓️ 一年计划：从“执行者”到“所有者”的转变](#️-一年计划从执行者到所有者的转变)
    - [💡 给资深工程师的特别建议](#-给资深工程师的特别建议)
  - [Fundations](#fundations)
    - [OS](#os)
    - [Linux](#linux)
    - [Algorithm \& Data Structure](#algorithm--data-structure)
    - [Learn From The Best](#learn-from-the-best)
    - [Learn From Framwork](#learn-from-framwork)
    - [Domain Knowledge](#domain-knowledge)
      - [Storage](#storage)
      - [Observebility](#observebility)
  - [Languages](#languages)
    - [C++](#c)
      - [高性能编程技巧](#高性能编程技巧)
      - [C++ Topics](#c-topics)
    - [Rust](#rust)
    - [Golang](#golang)
  - [OutPuts](#outputs)
    - [long-term project](#long-term-project)
    - [blog topics](#blog-topics)
  - [交流语言](#交流语言)
  - [体质](#体质)

## Fundations

### OS

- `深入理解计算机系统` CSAPP
- `程序员的自我修养-链接、装载与库`
- `TCP\IP`
- `编译原理`

### Linux

- `Unix环境高级编程`
- `Unix网络编程`
- `Linux高性能服务器编程`
- Linux源码

### Algorithm & Data Structure

- 复习 Algorithm & Data Structure
- 学习 System Design : System Desing 101

### Learn From The Best

* [Textbooks](https://github.com/kaitoukito/Computer-Science-Textbooks)
* CppCons 2023 2024 学习&记录
* C++ Weekly from Jason
* 概率统计 & 线性代数
* MIT6.824
* MIT6.712

### Learn From Framwork

* <font color=red>[P0]</font> leveldb 完成源码解析的笔记
* <font color=orange>[P1]</font> boost   学习异步io的源码
* <font color=orange>[P1]</font> folly   学习优化的数据结构
* <font color=orange>[P1]</font> abseil 学习优化的数据结构
* <font color=red>[P0]</font> influxdb 学习数据类型、文件结构、存储排布

### Domain Knowledge

* 分布式系统
* 云原生（加深理解）
* **压缩算法**
* Topics: K8s OLAP-Engine
* 广度：AI-GPT 业界TSDB AI-Agent

#### Storage

* <font color=green>[Done]</font> `数据库系统内幕`
* <font color=red>[P0]</font> `数据密集型应用系统设计` (《Designing Data-Intensive Applications》 by Martin Kleppmann)
* `PingCap awesome database learning from github`
* `《Database Internals: A Deep Dive into How Distributed Data Systems Work》 by Alex Petrov`

#### Observebility

## Languages

### C++

* <font color=red>[P0]</font> ProcFrameWork
* CodeSnippets
* C++ 20 23 26 blogs refs cppconf
* 编译器 clang(thinLTO PGO FDO) llvm(basic source code，理解框架)
  
#### 高性能编程技巧

* [C++ 高性能编程（全）(译)](https://www.cnblogs.com/apachecn/p/18172912)
* [Abseil C++ Tips of Week](https://abseil.io/tips/)
* [Abseil C++ Performance Guide](https://abseil.io/fast/)
* hotpatch: 方法、demo
* 模板元
* coroutine
* executor
* lock-free
* kernel code unix

#### C++ Topics

* [C++20 Coroutines and io_uring - Part 1/3](https://pabloariasal.github.io/2022/11/12/couring-1/)

### Rust

> 一种面向未来的可能性

* <font color=red>[P0]</font> rust by practice （目前进度在lifetime）
* leveldb-rs
* 语言特性深入理解
* 写三个练手小项目（minigrep、）

### Golang

> 按照团队要求，作为第二开发语言

* 复习 Basic concept
* go in deep

## OutPuts

### long-term project

* 长期增强 StellarKit（线程池、连接池、缓存、限流器、熔断）
* C++ ProcFrameWork
* C++ 反射框架
* C++ 超时工具
* a simple coroutine framework
* a simple c++ runtime lib
* Rust leveldb-rs

### [blog topics](BlogSrcREADME.md)

## 交流语言

* 英语词汇增强、口语练习，maybe 挑战下中级口译
* 日语基础，日常口语，年底日语N3考试

## 体质

* 体脂 < 15%
* 学会自由泳
