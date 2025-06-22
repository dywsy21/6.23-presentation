---
marp: true
size: 16:9
theme: am_orange
paginate: true
headingDivider: [2,3]
footer: \ *HTAP Query Optimization* *Rethink Query Optimization in HTAP Databases* *Haoze Song et al.*
---

# Rethink Query Optimization in HTAP Databases

<!-- _class: cover_e -->
<!-- _header: "" --> 
<!-- _footer: "" --> 
<!-- _paginate: "" --> 

Haoze Song, Wenchao Zhou, Feifei Li, Xiang Peng, Heming Cui
2023 SIGMOD
汇报人：王思宇 张霆轩

## 目录

<!-- _class: cols2_ol_ci fglass toc_a -->
<!-- _footer: "" -->
<!-- _header: "CONTENTS" -->
<!-- _paginate: "" -->

- [背景介绍](#3)
- [过往工作的不足](#7)
- [MetisDB系统](#15)
- [Demain成本模型](#18)
- [Visibility-aware计划选择](#22)
- [Proactive重优化](#27)
- [实验评估](#32)

## 背景介绍

<!-- _class: trans -->
<!-- _footer: "" -->
<!-- _paginate: "" -->

### HTAP Databases

<!-- _class: cols-2-73 -->

<div class=ldiv>

现代HTAP数据库通常维护两份数据副本来处理混合工作负载：

- 按行存储 --> 适合OLTP
- 按列存储 --> 适合OLAP

基于这种混合存储架构，查询计划可以有三种选择：

- 纯行存储计划
- 纯列存储计划
- **混合计划（hybrid plans）**

本文提出的工作即为对混合计划的全方位改进。

</div>

<div class=rimg>

![](pics/hybridlayout.png)

</div>

### 混合计划的性能优势

<!-- _class: cols-2-64 -->

<div class=ldiv>

混合计划允许单个查询同时从行存储和列存储中获取数据，这样可以发挥两种存储格式的各自优势。比如在星型模式中，维度表适合用主键索引访问，而事实表更适合列扫描。

在不同的查询场景下，混合计划能够：
- 为维度表选择行存储索引访问
- 为事实表选择列存储扫描
- 根据查询特点动态组合最优访问路径

实验结果显示混合计划在多种查询中都能获得显著的性能提升。

</div>

<div class=rimg>

![](pics/perfimpact.png)

</div>

### HTAP系统核心组件

系统的核心组件包括：

- **Row Store**: 存储完整的数据行，处理所有写操作
- **Column Store**: 按列组织数据，支持快速的列扫描
- **Delta Store**: 作为缓冲区吸收新的更新，定期合并到列存储
- **异步数据同步机制**: 将行存储的更新传播到列存储

## 过往工作的不足

<!-- _class: trans -->
<!-- _footer: "" -->
<!-- _paginate: "" -->

### 过往查询优化器的不足

传统的查询优化器在设计时并没有充分考虑HTAP数据库的**数据动态性**。

什么是数据动态性？

- OLTP引擎会持续执行写事务
- 不断地将新数据从行存储同步到列存储
- 这种持续的数据更新会影响查询计划的性能

本文提出的**MetisDB**将会解决这个问题：设计一个HTAP感知的查询优化器，能够捕获读操作和写操作之间的相互影响。

### 对HTAP的特性不敏感的Hybrid Plan的问题

传统的HTAP数据库在生成混合计划时存在一个核心问题：

基本上是把列扫描当作一个额外的数据访问路径加到现有的成本模型中，**进而忽略了HTAP环境的特殊性**。

这种做法会导致三个主要问题：

### 1. Data Synchronization的影响

**问题核心：** 在HTAP系统中，列存储需要通过delta store来吸收新的更新。

具体表现：
- 执行列扫描时，系统必须合并delta store和主列存储的数据
- 这个过程需要使用k-way merge算法，带来额外开销

实验发现：
- 随着OLTP并发量增加，列扫描的执行时间会**线性增长**
- 行扫描和列扫描性能相等的选择性阈值会随工作负载变化
- **没有固定的阈值**可以用来做访问路径选择

### 2. Data Freshness的影响

**问题核心：** 由于数据是异步从行存储同步到列存储的，列存储的数据新鲜度总是落后于行存储。

**可见性延迟（visibility delay）问题**：
- 新的更新从在行存储提交到能在列存储中读取之间有时间差
- 在高并发的OLTP工作负载下，延迟可能达到**10秒甚至更长**

性能矛盾：
- 即使列扫描在理论上比行扫描更快，但如果查询必须等待数据同步完成，总响应时间反而可能更长

### 3. Performance Isolation的影响

**问题核心：** 当使用混合计划时，OLAP查询会同时访问行存储和列存储，**打破了原本的性能隔离**。

具体影响：
- 在行存储负载较高时，混合计划会与OLTP事务竞争CPU、内存等资源
- 导致OLTP吞吐量下降多达**58%**
- 混合计划本身的性能也会因为资源竞争而下降

现有解决方案的局限性：
- 传统的配额限制方法（比如限制OLAP只能访问500MB以下的表）太过僵化
- 无法适应不同的工作负载变化


### MetisDB的改进

MetisDB针对这些问题提出了三个关键技术：

1. **Delta-aware的成本模型 Demain** - 精确建模数据访问路径的代价
2. **Visibility-aware的计划选择算法** - 通过预执行掩盖可见性延迟  
3. **Resource-aware的重优化算法** - 保证性能隔离的关键机制

这些技术共同构成了HTAP感知的查询优化器，能够在保持性能隔离的同时充分发挥混合计划的优势。

![#c h:200](pics/imp.png)

## MetisDB

<!-- _class: trans -->
<!-- _footer: "" -->
<!-- _paginate: "" -->

## MetisDB

![#c](./pics/屏幕截图%202025-06-22%20163431.png)

Metis算法基于MetisDB，遵循存算分离架构：

- 行存储：LSM-TREE 
- 列存储：基于append-only的B+树
- 计算单元：OLTP事务，OLAP事务，使用一个Consistent Meta进行内容统一控制（基于时间戳）

查询计划的优化使用***METIS***。

### Metis

Metis算法分为代价评估，计划选择和执行两部分:

![#c](./pics/屏幕截图%202025-06-22%20164119.png)

## Demain

<!-- _class: trans -->
<!-- _footer: "" -->
<!-- _paginate: "" -->

### Demain

<!-- _class: cols-2 -->

<div class=ldiv>

代价评估基于增量存储感知模型（Demain）和基数规模进行估计

代价评估部分：

以 ***SELECT col1 FROM table WHERE col1 between ${a} and ${b}*** 为例

**分布式系统的网络延迟**：

![alt](./pics/屏幕截图%202025-06-22%20164705.png)

**行扫描**：

![alt](./pics/屏幕截图%202025-06-22%20164726.png)

</div>

<div class=rimg>

![alt](./pics/屏幕截图%202025-06-22%20172849.png)

</div>

### Demain Cost Model

<!-- _class: cols-2 -->

<div class=ldiv>

**行索引遍历**：

![alt](./pics/屏幕截图%202025-06-22%20164743.png)

**列扫描**：

1. 增量树查找：

![alt](./pics/屏幕截图%202025-06-22%20164757.png)

![alt](./pics/屏幕截图%202025-06-22%20164816.png)

</div>

<div class=rimg>

![alt](./pics/屏幕截图%202025-06-22%20172849.png)

</div>

### Demain优化策略

<!-- _class: cols-2 -->

<div class=ldiv>

2. 稳定列查找：

![alt](./pics/屏幕截图%202025-06-22%20165504.png)

总代价：

![alt](./pics/屏幕截图%202025-06-22%20164846.png)

与传统方式一样，Metis参考sel进行选取，如果sel较小会考虑直接进行行读取，避免过多的IO

相较于传统方法，Metis在计算列存储时会考虑增量B+树的遍历代价，使得其会在sel适中时更偏向行存储

</div>

<div class=rimg>

![alt](./pics/屏幕截图%202025-06-22%20172849.png)

</div>

## Visibility-aware Plan Selection

<!-- _class: trans -->
<!-- _footer: "" -->
<!-- _paginate: "" -->

## Visibility-aware Plan Selection

进行代价估计之后，首先排除所有远非最优的计划，然后利用Visibility-aware算法生成一系列种子计划，用于后续的进一步优化和执行：

Visibility-aware Plan Selection通过提前调度可用数据上面的预执行，从而掩盖HTAP数据库中的可见性延迟。

![alt](./pics/屏幕截图%202025-06-22%20172733.png)

### Visibility-aware Plan Selection

可见性延迟的来源：

1. 新的写入被提交到数据传输管道中，然后从行存储运送到列存储

2. 传输的逻辑日志由delta-store解析

3. 新的更新被附加到delta-store中并由B+树索引

4. 更新被提交到delta-store中并最终对查询可见

**估计方法**：

Metis使用历史数据估计可见性延迟，并将OTLP的工作负载的并发映射到可见性延迟，同时全局元服务持续收集新的测量值，再运行时进行校准。计算每个并发的可见性延迟移动平均值（最后30个样本），用多项式回归曲线求解

### Visibility-aware Plan Selection

算法流程如图:

![#c h:500](./pics/屏幕截图%202025-06-22%20170402.png)

### Visibility-aware Plan Selection

进行计划枚举时，将每一个物理计划表现为**有向无环图**。

1. 顶点: 物理操作符。

2. 有两种边：**硬依赖**（后一个操作必须等待前一个操作的全部输出），**软依赖**（两者可以流水线执行）

执行算法时：
1. 首先通过删除图中unavailable pending tasks来计算每个计划的预执行（pre-execution）tasks。由于并行性，pre-execution的整体性能改进应该是最长任务的执行时间。

2. 随后将visibility delay的知识结合到第 18 行的计划成本中，该成本捕获了用户观察到的端到端查询延迟。

3. 最后使用修订后的成本来指导选择函数并生成一个具有最低查询延迟成本的visibility-aware物理计划。

## Proactive Query Re-optimizations

<!-- _class: trans -->
<!-- _footer: "" -->
<!-- _paginate: "" -->

## Proactive Query Re-optimizations

在执行查询优化时，Metis 会生成一组种子计划（即面向行的计划、面向列的计划和 HTAP 感知计划）以节省重新优化的成本。

在执行查询时，Metis 从最便宜的计划开始，并根据运行时统计数据不断重新优化计划。

有三个优化点：

1. 种子计划
2. Runtime Statistics考量
3. Plan stitch

### 种子计划

由之前的算法给出。子计划都具有相同的逻辑结构，但可能采用不同的物理运算符。

通常，当存在优于row-oriented和column-oriented计划的混合计划时，种子计划由三个物理计划组成；否则，种子计划由两个计划组成。不会单独优化面向行和面向列的计划的逻辑结构。

### Runtime Statistics

Metis 会根据运行时统计信息定量评估其决策的影响。为了高效执行，当估计基数与实际统计数据相差甚远（例如超过 20%）时，Metis 会重新优化物理运算符。

![#c](./pics/屏幕截图%202025-06-22%20175553.png)

### Plan Stitch

在重新优化计划时，Metis 会从种子计划中拼接子计划，而不会丢失之前的进度，重新优化本质在于对剩余未执行计划的物理运算符的选择进行重新优化。

![#c](./pics/屏幕截图%202025-06-22%20175459.png)



## 实验评估

<!-- _class: trans -->
<!-- _footer: "" -->
<!-- _paginate: "" -->

### 实验设置与结果

通过实验验证了Metis的有效性：

#### 实验环境
- **硬件**: 7台机器集群，每台机器24核CPU、64GB内存
- **工作负载**: CH-benCHmark、TPC-DS、YCSB三种基准测试

### 核心实验结果

**混合计划的有效性**：
- CH-benCHmark: **40.9%的查询**受益于混合计划，平均获得**1.68倍加速**
- TPC-DS: **77.8%的查询**受益，平均**3.06倍加速**
- 最高加速：Q72查询达到了**11倍加速**

**性能隔离的维护**：
- Metis能够很好地维护OLTP和OLAP之间的性能隔离
- 即使在高负载情况下，OLTP吞吐量下降也控制在**8%以内**
- 而传统方法可能导致**58%的性能下降**

### 关键发现

三个核心技术的不同贡献：

1. **Demain成本模型** - 性能提升的关键
   - 能够纠正传统方法的次优计划选择
   - 次优的计划选择可能导致高达**16.4倍的性能下降**

2. **可见性感知的计划选择** - 进一步性能提升
   - 通过预执行技术掩盖可见性延迟

3. **资源感知的重优化** - 保证性能隔离的关键
   - 即使成本模型完全准确，重优化机制仍然不可或缺
   - 能处理成本模型无法覆盖的资源冲突问题

### 实验结论

Metis成功实现了三个设计目标：
- 在不同HTAP场景下有效利用混合计划
- 保持OLTP和OLAP间强性能隔离  
- 对工作负载变化具有鲁棒性和适应性
