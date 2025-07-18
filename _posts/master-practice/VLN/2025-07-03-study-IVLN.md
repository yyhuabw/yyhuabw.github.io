---
layout:     post
title:      "IVLN 论文研读"
subtitle:   "Iterative Vision-and-Language Navigation 论文研读"
date:       2025-07-03 19:00:00
author:     "hyy"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - VLN
---

本文提出**迭代视觉语言导航（IVLN）** 范式，旨在解决现有 VLN 基准中代理记忆在每轮结束后被清除的问题，使代理能在持久环境中利用记忆进行导航。作者构建了离散和连续的**IR2R 与 IR2R-CE 基准**，每个包含约 400 个 tours，覆盖 80 个室内场景。实验发现，简单扩展高性能 Transformer 代理的隐式记忆不足以应对 IVLN，而**构建地图的代理能从环境持久性中获益**，这促使 VLN 研究重新关注地图构建代理。

## 部分名词解释

- Room-to-Room（R2R）：视觉 - 语言导航（VLN）的经典基准任务，要求智能体根据**单个英文指令**在室内完成**短路径导航**
  - 每个 R2R episode 包含单个英语指令和对应的短路径，且每轮 episode 开始时清除记忆，场景基于 MatterPort3D 构建，评估代理在陌生环境中的冷启动导航能力
  - IVLN 通过将 R2R episodes 串联成 tours，扩展为持久化导航场景，使代理能利用历史经验
    - tours 是由一系列有序的 R2R episodes 组成的序列，每个 episode 包含独立的语言指令和目标路径
- ATSP（非对称旅行商问题）
  - 传统旅行商问题（TSP）：旅行商从起点出发，访问所有城市一次后返回起点，求最短路径
  - ATSP：城市间的往返距离不对称（即从 A 到 B 的距离≠从 B 到 A 的距离），需考虑方向对路径长度的影响
- LKH（Lin-Kernighan Heuristic）算法
  - **局部搜索与迭代优化**：从一个初始路径（如随机顺序或贪心生成的路径）开始，通过不断交换路径中的边（例如删除 k 条边并重新连接），寻找总距离更短的路径
  - **邻域搜索策略**：LKH 使用 “k-opt” 邻域结构（k 通常取 3），通过评估边交换后的距离变化，决定是否接受新路径，直至无法优化为止
  - 能在多项式时间内生成接近最优的 episode 顺序，平衡计算效率与解的质量
- nDTW：归一化动态时间规整
  - 动态时间规整（DTW）：用于衡量两个时间序列相似性的算法
    - 通过在时间维度上进行弯曲操作，允许时间序列在时间轴上进行局部的拉伸或压缩，从而找到两个序列之间的最佳匹配路径，使得两个序列之间的累积距离最小
    - 通过调整时间轴上的对应关系，让 A 和 B 中相似的特征点能够对应起来，进而计算它们的相似程度

  - 归一化动态时间规整（nDTW）：对 DTW 的一种改进，主要用于在一些需要更公平、更具可比性的场景中衡量序列之间的相似性
    - 在 DTW 的基础上，对计算得到的累积距离进行归一化处理
    - 通过归一化，使得 nDTW 的值在一个固定的范围内（通常是 0 到 1 之间），这样不同长度、不同尺度的序列之间的相似性度量结果更具可比性

- CLS（Classification Token，分类标记）：是 Transformer 模型中用于**聚合序列级信息**的特殊标记
  - 在处理序列数据时，模型通常会先提取每个元素的 “局部特征”，但最终决策往往需要**整个序列的全局信息**，CLS 作用：
    - 作为一个 “占位符” 插入序列开头（例如文本序列的第一个位置、历史动作序列的起始处）
    - 通过 Transformer 的自注意力机制，“吸收” 序列中所有元素的局部特征，最终生成一个**代表整个序列的全局特征向量**

- GRU：Gated Recurrent Unit，门控循环单元
  - 一种循环神经网络（RNN）的变体，专门设计用于处理序列数据中的长距离依赖问题
  - 相比传统 RNN，GRU 通过引入 “门控机制”，能更有效地捕捉序列中的长期依赖关系，同时缓解梯度消失问题
  - 两个门
    - **重置门（Reset Gate）**：
      - 决定 “忽略” 多少过去的信息
      - 当重置门接近 0 时，模型会 “忘记” 历史状态，只关注当前输入；接近 1 时，则保留大部分历史信息
    - **更新门（Update Gate）**：
      - 决定 “保留” 多少过去的信息，以及 “更新” 多少新信息

- 逆针孔相机模型：计算机视觉中用于将 2D 图像像素映射回 3D 空间点的数学方法

  - RGBD 传感器中，每个像素除了 RGB 颜色值外，还有一个 **深度值**，表示该像素对应的物体到相机的距离

  - 逆针孔模型，可以利用这个深度值，将每个像素 “还原” 到它在真实世界中的 3D 坐标位置，最终形成 **三维点云**

  - 针孔相机模型（正向投影）：真实世界中的 3D 点 $P = (X, Y, Z)$ 经过相机投影，会落在图像平面上的 2D 像素点 $p = (u, v)$

    这个过程可表示为：$\begin{bmatrix} u \\ v \\ 1 \end{bmatrix} = \frac{1}{Z} \cdot K \cdot \begin{bmatrix} X \\ Y \\ Z \end{bmatrix}$

  - 逆针孔模型（反向投影）：已知图像上的像素点 $p = (u, v)$ 和对应的深度值 $d$，求其在 3D 空间中的坐标 $P = (X, Y, Z)$

    公式为：$\begin{bmatrix} X \\ Y \\ Z \end{bmatrix} = d \cdot K^{-1} \cdot \begin{bmatrix} u \\ v \\ 1 \end{bmatrix}$

- EnvDrop：一种用于视觉语言导航（VLN）的**数据增强技术**，通过随机 “丢弃” 环境中的部分视觉信息，强制模型学习更鲁棒的语言 - 视觉对齐关系

  解决 VLN 任务中的 “过拟合” 问题

  - 原始环境可能包含与指令无关的视觉线索（如特定颜色、纹理），导致模型依赖这些虚假特征而非真正理解语义
  - EnvDrop 通过扰动环境，迫使模型关注指令中的关键语义而非表面视觉特征

- DAgger 算法（Dataset Aggregation）：是一种**模仿学习**算法，通过 “专家示范 + 模型预测” 的迭代过程，逐步优化模型策略

  - **初始阶段**：模型在少量专家示范数据（如人类标注的导航路径）上训练
  - **执行阶段**：模型在环境中执行任务，同时专家观察并标注模型 “未见过的状态”（即模型可能犯错的地方）
  - **数据聚合**：将专家新标注的数据加入训练集，重新训练模型，直到模型收敛

  解决传统模仿学习中的 “复合误差” 问题

  - 模型早期错误会导致后续决策偏离专家轨迹，而 DAgger 通过持续收集专家对新状态的反馈，逐步覆盖模型可能遇到的所有情况

- Progress Monitor 辅助损失：是一种**辅助训练目标**，用于监督模型在执行指令过程中的 “进度”，帮助模型理解 “当前已完成多少任务” 和 “剩余多少任务”

  - 在训练时，额外训练一个**进度预测器**（与主模型共享特征提取层），该预测器输入当前状态，输出 “指令完成百分比”
  - 通过最小化 “预测进度” 与 “真实进度” 的误差（如均方误差），为主模型提供额外的训练信号，引导其学习更合理的导航策略。
  - 防止模型在中途停滞或绕路，提升导航效率和成功率

## 引入

- 大多数关于执行导航或家务任务的语言引导具身智能体的研究本质上是 “按轮次的” 的 —— 在发出每个新指令之前，智能体的记忆会被清除
- 相比之下，物理机器人会从视觉观察中迭代构建地图，将其作为长期记忆的一种显式形式

智能体遵循有序的语言指令序列对室内空间进行巡视（tour）

- 每次巡视由带目标路径的独立语言指令片段（episode）组成，智能体可利用记忆更好地理解后续巡视指令

**离散场景**：

- 智能体在图的边线上移动，并观察清晰、构图良好的图像
- 扩展了一个最先进的 Transformer 智能体，该智能体在解释指令时会基于路径历史学习隐式记忆

**连续场景**：

- 智能体执行移动动作的同时，观察从离散全景图像重建的 3D 环境的含噪图像
- 提出了一个构建和解释显式语义地图的智能体

## 相关工作

### 指令引导导航

在 VLN 中，智能体必须遵循自然语言指令，在从未见过的环境中沿描述的路径导航

- 该范式未考虑长期运行的持久性智能体如何利用先验经验来更好地遵循同一环境中的未来指令
- 在环境中积累先验经验是机器人部署的关键 —— 例如构建用于定位和推理的语义地图

### 1. 离散场景下的 VLN 基准

离散 VLN 任务的**基本特征**：

- 智能体在渲染的 2D 或 3D 场景中，根据语言指令推断动作
- 智能体的位置和方向只能以离散量变化（预设），或从预定义的动作选项中选择

相机技术的进步使语言引导导航能在逼真的室内和户外场景中实现

R2R 基准的**局限性**：

- R2R 要求智能体解析单个英文指令，沿室内短路径导航
- 智能体缺乏动力进行高效的环境记忆或建图

对**长期规划**的扩展研究：

- 现有扩展方法
  - 串联语言对齐的路径和指令，要求智能体不仅到达目标，还需严格遵循路径描述
    - 通过**串联多个 R2R 路径及对应指令**形成更长的导航任务
  - 收集多语言指令路径或协作对话式指令
    - 多语言数据可推动智能体理解不同语言的空间描述逻辑，避免仅适配单一语言
    - 不同语言对空间关系的表达存在差异，多语言数据可帮助研究如何将多样化的语言指令统一映射到空间动作，促进通用导航模型的发展
- IR2R 的创新
  - 提出最长的 “巡视”（tours）路径，覆盖区域随时间显著重叠，迫使智能体利用历史指令信息和场景经验

### 2. 连续场景下的 VLN 基准

连续场景导航的挑战与现状：

- 物理机器人的需求：移动物理机器人执行语言指令时，需应对真实连续世界（如连续的空间移动、传感器噪声），与离散场景有本质区别
  - 现有研究尝试将离散 VLN 的策略迁移到物理世界，但因无法适配连续环境的复杂性，效果有限

VLN-CE 基准的设计与不足：

- 技术实现：VLN-CE 通过对 MatterPort3D 室内场景进行连续 3D 重建，将 R2R 任务扩展到连续空间，允许智能体执行连续动作
- 评估缺陷：VLN-CE 仍采用 “独立同分布” 的评估方式 —— 仅测试智能体对单个指令的响应，未考虑环境持久性（如历史探索经验对后续导航的影响）

IR2R-CE 基准的创新与意义：

- 持久性环境：IR2R-CE 要求智能体在同一环境中处理多轮语言指令（形成 “tour”），强制利用历史记忆（如已探索区域的地图），符合真实机器人长期作业场景
- 消除抽象化：摒弃离散 VLN 的导航图抽象，直接在连续 3D 空间中评估，更贴近物理机器人的部署需求

### 3. VLN 中的预探索

预探索方法的定义与实现方式：

- 允许智能体在执行语言指令前**预先全面探索环境**，获取完整的环境信息，再基于此规划导航路径
- 具体实现：
  - **预训练显式探索**：通过预训练阶段让智能体遍历环境
  - **推理时波束搜索**：在推理阶段通过波束搜索策略探索可能的路径
- 预探索方法的导航效果显著优于标准 VLN 方法（如 R2R 基准），因其依赖完整的环境先验知识
- 预探索可视为 IVLN 的 “理论上限”—— 当智能体已完全探索环境时，其表现为 IVLN 任务的最优参考

IVLN 范式则在执行任务时收集获取信息，信息可能并不完整，研究如何利用增量信息提升长期导航效率。模拟真实场景中 “边执行边探索” 的需求，更贴近实际

- **动态环境中的增量学习**：智能体在执行指令的过程中逐步积累环境知识，并利用这些知识优化后续决策

### 4. 具身人工智能中的持久性环境

具身 AI 中视觉导航研究从简化场景转向关注**真实世界的复杂性**，例如动态障碍物、多任务耦合等

- 技术推动因素
  - **3D 场景数据集**：规模和质量提升，提供更真实的环境模拟
  - **高性能仿真平台**：支持复杂物理交互和大规模训练

持久性环境：智能体需在**长期持续存在的环境**中行动并与之交互，而非独立的单次任务场景

- **多目标导航**：寻找多个目标物体时，若独立处理每个目标，可能因重复路径导致效率低下
- **视觉房间重布置**：移动家具时需综合考虑空间布局和物体关系，独立处理子任务无法优化整体效果

> 需基于**持久性的语义和空间信息**推理（记住已探索区域的物体位置、语义标签），而非每次任务重置记忆

IVLN 范式的价值：

- 将自然语言引入场景感知问题，实现持久性视觉语义与语言信息的关联
  - 用语言指令 “客厅的蓝色沙发旁放茶几” 关联已记忆的沙发位置（视觉语义）和空间动作
- 通过语言引导增强智能体的理解和规划能力

IVLN 是具身 AI 中持久性环境研究的自然延伸，通过语言将碎片化的视觉信息整合成可推理的知识体系

## 迭代视觉-语言导航

从 R2R 数据集到迭代巡视的扩展

- 将 R2R 数据集中的独立情节（单个指令 + 短路径）串联为 “巡视（tours）”，形成覆盖场景大片区域且包含**回溯路径**的长序列
- 每个 tour 包含多个 episode，使路径长度和指令上下文远超传统离散（IR2R）或连续（IR2R-CE）基准

### 1. 迭代范式设计

**序列指令 + 双阶段迭代**

1. 智能体导航阶段
   - 智能体接收语言指令并自主推断导航动作，等价于一次标准 VLN episode
   - 结束条件：
     - 智能体主动发出 “STOP” 信号（表示到达目标）
     - 执行最大允许次数的动作后仍未完成任务
2. 神谕导航阶段
   - **第一部分：目标修正**
     若智能体未到达目标 0.5 米范围内，神谕强制其动作至目标点，模拟 “人类教导正确终点” 的过程，帮助智能体学习正确路径
   - **第二部分：路径过渡**
     神谕引导智能体至下一个 episode 的起点，期间智能体被动观察环境，类比 “跟随人类等待下一条指令”，为后续任务积累空间记忆

**环境持久性模拟 + 双向学习机制**

### 2. 从 VLN 数据生成巡视序列

如何将独立导航情节串联成连贯长序列

#### 1. 生成 tours 的核心目标

1. **最小化连续 episode 的起止距离**：让前一 episode 的终点与下一 episode 的起点尽可能接近，减少神谕导航阶段的移动成本
2. **最大化包含的 episode 数量**：因 IR2R-CE 等连续场景中路径查找可能因障碍物失败，需通过优化提高 tours 的完整性和可行性
   - 在连续 3D 场景中，障碍物会将空间分割成多个不连通的区域
   - 最大化 tours 中包含的 episode 数量 算法会优先选择那些在空间上更密集、连通性更好的 episode 组合
   - 生成 tours 时需计算路径间的可导航测地距离，最大化数量的过程等价于在连通区域内寻找最多的可串联 episode，自然绕过了障碍物阻断的路径

#### 2. 路径子集划分

场景中封闭的门或障碍物可能形成不连通区域（如两个房间被锁门隔开），导致部分 episode 无法串联

- 定义集合 $X$ 为场景中所有唯一路径，将 $X$ 划分为多个子集 $p$，每个子集内的路径可相互导航
- 连通性判断：计算每对路径间的 “可导航测地距离”，若距离有限则视为连通
  - **离散场景**：基于导航图计算节点间距离
  - **连续场景**：基于 3D 网格计算，考虑智能体尺寸和动作限制（如移动步长）

#### 3. 路径排序与优化算法

将子集路径排序为 tours

- **优化目标**：确定子集 $p$ 中路径的顺序，使神谕导航阶段的总移动距离最小

- **问题转化**：等价于求解 “非对称旅行商问题（ATSP）”

  - episode 的起点 / 终点位置：城市
  - 两个 episode 间的可导航测地距离：城市间距离
  - 访问顺序：episode 在 tours 中的排列顺序
  - 神谕导航阶段总移动距离最小的 tours：最短回路

  > 非对称性：由于场景中障碍物分布不均，从 episode i 的终点到 episode i+1 的起点的距离，可能与反向路径不同
  >
  > - 例如，从房间 A 到房间 B 需绕开障碍物，而反向路径可能有更短路线，导致距离不对称

- **求解方法**：使用 Lin-Kernighan 启发式算法（LKH）近似求解，该算法常用于解决大规模 TSP 问题，通过迭代交换路径边来优化路线

#### 4. 多指令场景的处理策略

若每个路径对应 $n>1$ 条不同语言指令（如同一目标路径有多种描述方式），需增强数据多样性

- 将每个 tours 复制 $n$ 次，每次复制时对每条路径**不放回采样**一条指令，确保同一 tours 内的指令不重复
- 目的：模拟真实场景中用户可能用不同指令描述同一任务，提升智能体对语言多样性的适应能力

### 3. 数据集特征

基于离散 R2R 数据集生成 IR2R 基准，基于连续 R2R（VLN-CE）生成 IR2R-CE 基准

- **训练集**：用于智能体模型训练
- 已见验证集：包含训练阶段见过的场景情节，用于验证模型对已知环境的泛化能力
- 未见验证集：包含训练阶段未见过的场景情节，用于测试模型对新环境的适应能力

---

<img src="/img/VLN/IVLN/dataset-features.png" alt="dataset-features" style="zoom:33%;" />

- **离散场景**：tours 数量较少（375 个），但平均长度更长
  - 场景以导航图表示，每个节点（位置）到其他节点均有可行路径（无障碍物阻断），因此可串联更多 episode 形成长 tours
- **连续场景**：tours 数量更多（414 个），但平均长度更短
  - 真实 3D 场景中存在障碍物，导致 episode 端点间可能无法导航，场景被分割为多个不连通区域，每个区域仅能生成短 tours

每个 tours 包含的 episode 数量差异很大，采样随机性和场景规模多样性

> 由于情节在 IVLN 中是相互串联的，当前情节目标路径上的位置可能在巡视序列（tour）的前期就已被观测到

### 4. 迭代评估的指标

评估目标：智能体从巡视序列（tour）一开始就表现出 “成功” 和 “高效”，还得随着巡视推进持续进步

- 为了给 “基于巡视序列的迭代评估” 设计指标，改用 **nDTW（归一化动态时间规整）**
  - “最短路径” 限制等，能适配更多数据集，更加灵活

- 传统情节式：每个情节单独算 nDTW ，再把所有情节的结果平均
- 迭代式：一个 tour 的多个情节需要整体评估。否则智能体可能 “钻空子”：用第一个情节探索整个环境（哪怕这情节表现差），靠 “后续情节” 拉高平均

> 即：
>
> - **问题 1**：按单 episode 计算 nDTW 后取平均，智能体可能 “牺牲某个 episode”，以换取后续 episode 的高表现，导致整体评估失真
> - **问题 2**：未限制跨 episode 的路径对齐，智能体可能利用其他 episode 的路径完成当前任务（如用前一个 episode 的轨迹 “凑数” 当前任务），违背 “按指令独立完成每个 episode” 的要求

**具体做法**：

- 评估对象从 “单 episode 路径” 改为 “整个 tour 路径”（解决第一个问题）

  - **候选路径\($Q^T$\)**：包含智能体在 tour 的 “智能体导航阶段”（排除神谕导航阶段）所访问的所有 3D 点（即智能体自主导航的完整轨迹）
  - **目标路径\($R^T$\)**：将 tour 中所有 episode 的目标路径串联起来，同样排除神谕导航阶段的路径（即理想情况下智能体应遵循的完整轨迹）

  计算逻辑：基于 $Q^T$ 和 $R^T$ 计算 nDTW，而非单 episode 的路径 $Q$ 和 $R$

- 尊重 episode 边界，限制跨 episode 的路径对齐：在 DTW 算法对齐候选路径（$Q^T$）和目标路径（$R^T$）的 3D 点时（解决第二个问题）
  - 若两个点属于 tour 中的**不同 episode**，则赋予二者 “无限距离”（即无法对齐）
  - 仅允许**同一 episode 内**的点进行对齐匹配

为计算数据集分块的性能，通过情节数量对 nDTW 进行加权聚合，以避免仅在短巡视序列上表现良好而导致分数虚高

## 方法

### 1. 视觉语言导航（VLN）基准模型

如何通过改进历史信息处理机制，让模型支持巡视序列级的迭代任务推理

- 历史感知多模态 Transformer（HAMT）模型

  - 是一种基于 Transformer 的经典 VLN 模型，先在代理任务（辅助训练任务）上预训练，再在 VLN 任务上微调
  - 三个编码器
    - 指令编码器：处理语言指令（如 “走到红色沙发旁”）
    - 视觉编码器：处理当前环境的视觉观测（如摄像头画面）
    - 历史编码器：处理过去的 “状态 - 动作对”（即智能体之前的位置和执行的动作，如 “在 A 点向左转”）
  - 历史嵌入：在时间步t，历史嵌入为 $\{h_{CLS}, h_1, ..., h_{t-1}\}$
    - $h_t$：第 $t$ 步的状态 - 动作对特征
    - $h_{CLS}$：CLS 标记的特征，用于汇总整个历史序列的全局信息
  - 跨模态融合：通过 Transformer 将上述三种信息融合，预测智能体下一步的动作

  > **局限**：仅包含当前 episode 内的历史信息，无法处理跨 episode 的巡视序列任务

- TourHAMT：过修改历史嵌入机制，实现对巡视序列（多 episode 串联）的支持

  - 历史嵌入的扩展：纳入先前 episode 的信息
    - 在处理第 $i$ 个 episode 的第 $t$ 步时，历史嵌入包含：$\{h_{PREV}, h_1^1, ..., h_{l_1}^1, ..., h_1^{i-1}, ..., h_{l_{i-1}}^{i-1}, h_{CLS}, h_1^i, ..., h_{t-1}^i\}$
      - $h_t^i$：第 $i$ 个 episode 第 $t$ 步的状态 - 动作嵌入
      - $h_{PREV}$：特殊标记，用于分隔不同 episode 的边界（避免历史信息混淆）
      - 包含前 $i-1$ 个 episode 的所有状态 - 动作对（$h_1^1$ 到 $h_{l_{i-1}}^{i-1}$），以及当前 episode 截至 $t-1$ 的信息
    - 步数限制：为避免计算量过大，仅保留最近 50 步的历史信息
    - 解冻历史编码器（原始 HAMT 的历史编码器是固定的），使其能学习新的历史嵌入规则
    - 采用 “带拐点加权的教师强制” 训练（强化关键动作的学习），并在每个 episode 结束后更新模型梯度（适配 tour 的迭代特性）

### 2. 视觉-语言导航-连续环境 VLN-CE

基准智能体 —— 跨模态注意力（CMA）智能体

- 输入：RGBD 数据（彩色图像 + 深度信息）、导航指令、上一步动作。
- 输出：从 “左转（TURN-LEFT）、右转（TURN-RIGHT）、前进（MOVE-FORWARD）、停止（STOP）” 中预测下一步动作
- 双 GRU 结构
  - 第一个 GRU：专门追踪视觉信息（处理 RGBD 观测到的环境特征）
  - 第二个 GRU：追踪全局状态（整合各类信息），并基于此预测下一步动作

#### 1. 带非结构化潜在记忆的智能体

如何通过调整隐藏状态和记忆结构，让智能体具备跨情节（episode）或巡视序列（tour）的推理能力

<img src="/img/VLN/IVLN/models.png" alt="models" style="zoom:50%;" />

| 模型           | 核心记忆逻辑（隐藏状态重置规则）                             | 目标效果                                                     |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **CMA**        | 每个情节（episode）结束后，**所有隐藏状态清零**              | 作为 “无跨情节推理能力” 的基准，对比其他模型                 |
| **TourCMA**    | 仅在**每个巡视序列（tour）开始时**，重置视觉 GRU 的隐藏状态；**状态 GRU 按情节（episode）重置** | 让视觉 GRU 记住 “整个巡视序列的所有步骤”（扩展时间感受野），同时状态 GRU 保留 “单情节内的推理” |
| **PoolCMA**    | 每个情节（episode）重置隐藏状态，但用**最大池化（max-pool）** 把隐藏状态压缩成 “巡视序列持久向量”，并输入视觉 GRU | 让视觉 GRU 同时保留 “单情节记忆”（隐藏状态）和 “巡视序列级记忆”（池化后的持久向量） |
| **PoolEndCMA** | 在 PoolCMA 基础上，**把巡视序列记忆和动作预测绑定**：拼接 “巡视序列记忆向量” 和 “最终状态向量”，共同预测动作 | 强制智能体利用 “跨情节观测”（如之前走过的走廊）辅助决策，避免智能体忽略长程信息 |

- 在 IR2R-CE 环境中，用标注数据引导模型学习来训练模型
- 对 “巡视序列持久记忆结构”，**禁用 “当前情节之前步骤” 的梯度** → 等价于 **自适应时间截断反向传播（TBPTT）**，且 “时间步长 = 情节长度”

#### 2. MAP-CMA：具备语义地图的智能体

**地图类型与基本定义**

1. **占据地图（Occupancy Maps）**：网格单元取值为`0`（空）或`1`（被占据），用于表示环境中哪些区域可通行
2. **语义地图（Semantic Maps）**：每个网格包含一个**13 维独热向量**，对应 R2R 环境中的 13 种常见语义标签（如 “地板”“墙壁”“沙发” 等）
   - 例如：某个网格的独热向量为`[0,1,0,...,0]`，表示该位置是 “墙壁”。

**三维点云生成**

1. **深度图反投影**：使用**逆针孔相机模型**，将 RGBD 传感器的**深度测量值**转换为**三维点云**
2. **语义信息反投影**
   - 将以智能体为中心的语义标签（**真实标签**，**预训练模型预测**）投影到三维点云
   - 最终形成**带语义的三维点云**，每个点包含坐标和语义标签

**二维地图生成**

1. **高度维度压缩**：将三维点云**沿高度方向压缩**为二维网格（俯视图）
2. **语义冲突处理**：当同一垂直列存在多个语义标签时（如 “地板” 上方有 “桌子”），选择**最高的语义标签**进行投影，优先保留 “物体顶部” 的语义，忽略被遮挡的底层
3. **楼层约束**：考虑到智能体可能在多层建筑中穿梭，仅投影**位于当前楼层地板和天花板之间的特征**（忽略其他楼层的信息，避免混乱）

通过上述步骤，MAP-CMA 模型将复杂的三维环境信息，简化为**以智能体为中心、包含语义和占据信息的二维地图**，降低计算复杂度，并增强决策能力

---

原始 CMA 模型的输入包含 RGB 视觉信息，而 MAP-CMA 通过**用 “语义地图 + 占据地图” 替换 RGB 输入**，增强模型对环境的结构化理解能力

- **输入维度**：14×64×64
  - 空间尺寸：64×64 的网格（覆盖智能体周围环境的局部区域）
  - 通道组成：共 14 个通道，包括 **13 个语义通道**和 **1 个占据通道**（0 为空、1 为占据）
- 通过**4 个卷积块（CoBRA 模块）** 对拼接后的地图进行编码，最终输出**128×4×4 的特征向量**
- 编码后的 “语义空间特征” 被**直接替代原始 CMA 中的 RGB 视觉特征**，在 CMA 的后续模块中传播

<img src="/img/VLN/IVLN/method.png" alt="method" style="zoom:50%;" />

- **输入**：智能体接收的原始信息，包括：
  - RGB 图像（环境视觉画面）
  - Depth 深度图（环境距离信息）
  - Language Instruction 语言指令（如 “左转进入走廊，看到木地板停下” ）
- **地图构建**：从实时 RGB / 深度图，构建 **全局语义地图**，用于长期环境理解
  - **场景分割**：用 “Pretrained RedNet” 对 RGB 图像做语义分割（识别墙面、地板、家具等类别）
  - **逆投影与坐标转换**：结合 Depth 深度图和智能体位姿，把 2D 分割结果转换为 3D 空间坐标，对齐真实环境
  - **时间聚合**：把当前帧的地图信息，与上一步地图融合，逐步构建 **全局语义地图**
- **决策策略**：融合多模态信息（地图、指令、历史状态），预测下一步动作
  - **以自我为中心的地图裁剪**：从全局语义地图中，裁剪出智能体 **当前视角的局部地图**—— 只关注智能体周围环境，降低计算量
  - 多模态特征提取：
    - **CoBRA 模块 ×4**：处理局部语义地图，提取空间特征
    - **ResNet50 (PointNav)**：处理原始 RGB 图像，提取视觉特征
    - **Bi-LSTM**：处理语言指令，提取文本特征，并建模指令的时序依赖（如 “左转 → 走到底 → 停下” 的顺序）
  - **动作预测**：融合 “地图特征、视觉特征、语言特征、上一步动作和循环状态”，预测下一步动作

---

**核心训练流程**

1. **初始训练**：在移植到 IR2R-CE 的 “增强版 EnvDrop” 数据集上，用**教师强制法**进行初始训练
2. **微调**
   - 从初始训练的模型中，选出在 “EnvDrop Val-Unseen（未见过的验证集）” 上表现最好的**检查点**
   - 在 IR2R-CE 的训练集上，用 **DAgger 算法**对该检查点进行微调，进一步提升性能
3. **最终模型选择**：以 “Val-Unseen（未见过的验证集）” 上表现最佳的检查点作为最终模型，确保模型对未知环境的泛化能力

- **辅助损失函数**：两个训练阶段均使用 **Progress Monitor 辅助损失**，用于监督模型 “是否按指令进度推进”（例如避免绕路或停滞），提升导航效率

- **地图的时间构建方式**：训练时测试了三种不同的地图重置策略，以验证 “地图记忆周期” 的影响：

  - **情节地图**：每个情节（episode）结束后重置地图（仅保留当前情节的环境信息）

  - **迭代地图**：每个巡视序列（tour）结束后重置地图（保留整个巡视序列的环境信息）
  - **已知地图**：每个场景的地图预先计算好（无需动态构建，作为上限参考）

每种地图构建方式下，分别训练两种模型：

- 基于**真实语义标签**（理想情况）
- 基于**预测语义标签**（模型自动预测，接近实际应用场景）

## 实验与结果

**评估指标**

- **TL**（轨迹长度）：导航路径的总长度（越低越高效）
- **NE**（导航误差）：终点与目标的距离（越低越好）
- **OS**（oracle success）：在最优路径提示下的成功率（越高越好）
- **nDTW**：轨迹与理想路径的相似度（越高越好）
- **SR**（成功率）：成功到达目标的比例（越高越好）
- **SPL**：成功率加权路径长度（越高说明 “既成功又高效”）
- **t-nDTW**（主要指标）：巡视序列级的轨迹相似度（越高说明跨情节导航一致性越好）

### 1. IR2R 和 IR2R-CE 中的非结构化记忆

对非结构化记忆的简单扩展无法充分利用巡视序列信息 → 不仅未能提升巡视序列性能，反而常常会产生负面影响

- 即便是没有任何巡视序列记忆，情节式 HAMT 模型仍是一个性能强劲的基准模型
- 仅仅是扩展记忆，就使 t-nDTW 值较 HAMT 模型降低了一半

> 与预训练任务相比，历史编码器难以应对其输入中的分布偏移，而直接微调不足以纠正这一问题

相比单情节记忆，巡视序列记忆更容易导致过拟合

### 2. IR2R-CE 中的语义地图记忆

采用一种结构化记忆，具体形式为包含语义和占据信息的度量地图

1. **迭代构建地图显著优于单情节地图**，验证了 “累积巡视序列记忆” 对结构化导航的重要性
2. **真实语义性能 > 预测语义，但差距可通过优化构建方式缩小**。即使语义有误差，合理的地图构建仍能发挥作用
3. **“训练 - 评估” 构建方式匹配时性能更优**：当训练和评估采用相同的地图构建方式，性能普遍高于不匹配的情况。这提示模型对地图构建策略的 “一致性” 敏感

**核心发现**：

1. **迭代更新的地图能有效利用先前经验**：无论语义来源，评估时使用 “迭代更新的地图（It.）” 的智能体，性能均优于 “每个情节重置的地图（Ep.）”
   - 迭代地图能累积巡视序列中 “先前情节的环境信息”（如已走过的走廊、见过的物体），辅助后续新指令的执行
   - “利用跨情节记忆” 需要专门训练。
2. **基于地图的智能体显著优于非结构化记忆的 CMA 模型**，证明**结构化地图记忆比非结构化记忆更适合长巡视序列任务**
3. **推理语义的鲁棒性超出预期**，MAP-CMA 对 “语义分割误差” 有较强的鲁棒性，适合实际应用（真实场景中难获取完美语义标签）
4. **已知地图（预计算）不如迭代式地图**：预先知道完整地图的智能体，性能反而不如 “边探索边迭代构建地图” 的智能体
   - 原因可能是：迭代式地图训练让智能体学习了 “已探索区域” 与 “未探索区域” 的差异（如 “此处未去过，可能有新路径”），而已知地图缺乏这种探索经验，限制了推理能力
   - 对后续研究的启示：已知地图性能不佳的现象，引出了 “如何更好地编码场景信息以支持推理” 的开放性问题

## 局限性

- 多种语言
- 多种环境
- 性能