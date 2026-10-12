# 伺服执行器扩展摩擦模型论文精读

> 来源覆盖：完整论文，7 页，含全部主图与表格。
> 提取可信度：正文高；公式和图表已结合页面图像核对。
> 定位模式：page-grounded；PDF p. N 指本地 PDF 从 1 开始计数的页码。
> 主要分析类型：methods，方法与工程系统。
> 次要分析类型：None。
> 外部核验：targeted external check，仅核实发表信息与代码来源。
> 完整性：Complete relative to supplied source；未开展全面查新，也未运行代码复现实验。

## 01 基本信息

[Paper] 标题：Extended Friction Models for the Physics Simulation of Servo Actuators，可译为《用于伺服执行器物理仿真的扩展摩擦模型》。作者：Marc Duclusaud、Grégoire Passault、Vincent Padois、Olivier Ly。机构：波尔多大学、CNRS、LaBRI 与 Inria Auctus。前两位作者共同贡献。[Paper: PDF p. 1, Title and affiliations]

[External] 正式发表于 **2025 IEEE International Conference on Robotics and Automation（ICRA 2025）**，页码 12091–12097，DOI：10.1109/ICRA55743.2025.11128640。依据：[IEEE 出版记录](https://ieeexplore.ieee.org/abstract/document/11128640)、[作者代码仓库引用信息](https://github.com/Rhoban/bam)。因此它是会议论文，不能称为“顶刊论文”。

[External] 所给文件是 arXiv:2410.08650v4，更新于 2025 年 11 月 4 日；最早上传于 2024 年 10 月 11 日。arXiv 版本时间与正式发表年份是不同信息。[arXiv 记录](https://arxiv.org/abs/2410.08650)

[External] [ICRA 2025 官网](https://2025.ieee-icra.org/)将其定义为 IEEE Robotics and Automation Society 的旗舰会议。

[Analysis] 按机器人学界的常用评价，它属于顶会论文。具体成果认定仍需对应所采用的会议目录；会议层级不能代替对这篇论文贡献的判断。本文的优势是物理可解释性、实验设计和可实施性，贡献主要集中在工程建模与验证。

领域：机器人动力学、摩擦建模、系统辨识、物理仿真。代码：[Rhoban/bam](https://github.com/Rhoban/bam)。论文说明提供开源模型与辨识方法；本文分析没有验证数据下载完整性或代码可复现性。[Paper: PDF p. 7, Conclusion]

阅读日期：2026 年 10 月 12 日。用户尚未提供具体研究方向，不能据此判断与个人课题的直接契合度。

术语表：

| 统一术语 | 首次解释 | 原文写法与处理 |
|---|---|---|
| 伺服执行器 | 包含电机、减速器及控制环节的执行单元 | servo actuator；小型产品常称舵机 |
| 库仑—黏性摩擦 | 恒定摩擦项与随速度增长的黏性项 | Coulomb-Viscous；图中 C.-V. |
| Stribeck 效应 | 低速附近摩擦随速度变化而衰减的现象 | Stribeck effect；保留原名 |
| 反驱 | 外部载荷推动执行器运动 | backdrive；区别于电机主动驱动 |
| 2R 机械臂 | 两个旋转关节构成的机械臂 | 2R arm；图中 RR arm，统一为 2R |
| MAE | 平均绝对误差，Mean Absolute Error | 关节角误差单位 rad；图 1 另给末端误差 m |
| CMA-ES | 协方差矩阵自适应进化策略 | 用于黑箱参数优化，不是神经网络训练 |

## 02 一句话概括

[Paper] 本文将速度、载荷、驱动与反驱不对称及二次载荷项纳入伺服执行器摩擦模型，用单摆实验辨识参数，再在四款执行器和两套 2R 机械臂上验证，相比库仑—黏性基线降低了仿真轨迹误差。[Paper: PDF p. 6, Section VI]

## 03 研究问题

[Paper] 简单摩擦模型难以同时再现启动、静止保持、缓慢运动和反驱现象，低增益控制下尤其明显。本文提出的问题是：能否通过少量有物理含义的扩展项，加上可操作的参数辨识流程，提高真实执行器的仿真准确度？[Paper: PDF p. 2, Introduction; PDF p. 3, Figure 3]

[Analysis] 这直接影响仿真模型能否预测真实运动，但“轨迹仿真更准”与“强化学习策略实机迁移更好”是两个不同的验证目标。

## 04 研究背景和发展路径

[Paper-framed; external verification not performed] 论文把已有路线分为简化物理模型、扩展摩擦物理模型与学习式执行器模型。它指出神经网络模型可以拟合执行器行为，但物理解释较弱，且部分既有工作缺少与认真辨识过的物理模型比较。这里仅复述论文的背景组织，不将其作为独立核实过的领域结论。[Paper: PDF p. 1, Introduction; PDF p. 2, Introduction]

[Analysis] 本文主要价值是把已有摩擦效应整理成层次化模型，贯通辨识和仿真验证；不宜将每个摩擦效应都描述为作者首次发现。文中公式 (6)–(10) 明确关联既有文献。[Paper: PDF p. 3, Section III-A to III-C; PDF p. 4, Section III-D to III-E]

## 05 论文识别的核心痛点

| 痛点 | 表现 | 作者解释 | 证据 |
|---|---|---|---|
| 简单速度函数难以描述静止保持 | 零速度时直接使用符号函数会给出零摩擦 | 静摩擦需要作为可抵抗运动的力矩额度处理 | [Paper: PDF p. 2, Section II-B] |
| 摩擦随载荷变化 | 主动驱动与反驱边界随力矩改变 | 减速器负载增加时接触摩擦增大 | [Paper: PDF p. 3, Figure 3] |
| 不同减速器需要不同模型 | 谐波减速器与普通齿轮减速器的最优模型不同 | 方向性和二次载荷效应的贡献不同 | [Paper: PDF p. 6, Figure 5] |
| 单执行器拟合不等于系统泛化 | M5 在 Dynamixel 辨识阶段较好，在 2R 验证中未超越 M4 | 作者认为可能过拟合 | [Paper: PDF p. 6, Section VI-B] |

## 06 核心思想

[Paper] 表层方法是构造六种摩擦模型并优化参数。关键思想是先计算摩擦“最多能提供多少抵抗力矩”，再根据实际停止运动所需的力矩进行截断，而不是在所有状态下强行施加同一个带符号阻力。[Paper: PDF p. 2, Equation 3 and Equation 4; PDF p. 3, Equation 5]

[Analysis] 可推广的启发是：当低速、静止、反驱成为主要误差来源时，应先检查模型结构是否表达了这些状态，再继续调整参数。本文实验支持针对这四款执行器开展这种改进，不能由此断言所有任务都需要最复杂模型。

## 07 方法总览

[Paper] 输入包括执行器位置与速度、目标位置、控制律、负载质量和摆长，以及真实运动日志。输出包括摩擦参数、电机等效参数、仿真轨迹和 MAE。流程如下：[Paper: PDF p. 4, Section IV; PDF p. 5, Algorithm 1 and Section V]

实机单摆试验 → 记录不同载荷和控制增益下的四类轨迹 → 将 75% 日志用于辨识 → CMA-ES 优化仿真与实测轨迹的 MAE → 用 25% 日志验证 → 在 MuJoCo 中验证 2R 机械臂。

[Paper] 每种模型的辨识重复三次，作者报告约 4000 次迭代、不到 5 分钟收敛。该时间是文中实验设置下的报告值，不能推为所有机器上的耗时保证。[Paper: PDF p. 5, Section V-B]

[Paper] 在物理引擎中，外力矩与摩擦同时求解；论文提出用上一时间步的外力矩更新当前摩擦额度，依赖于外力矩不会骤变的近似。[Paper: PDF p. 5, Section IV-E]

## 08 核心模块拆解

| 模块 | 功能和必要性 | 输入与输出 | 支持证据 | 移除或变更的影响 |
|---|---|---|---|---|
| 分层摩擦模型 | 表达速度、载荷和方向效应 | 状态和力矩 → 摩擦额度 | [Paper: PDF p. 3, Section III; PDF p. 4, Figure 4] | 六模型比较显示贡献随执行器而变，不能只按参数数量排序 |
| 力矩截断 | 表达静止和停止运动 | 停止所需力矩与额度 → 实际摩擦 | [Paper: PDF p. 3, Equation 5] | 未单独做该计算机制的消融，不能量化其独立收益 |
| 控制律与电机模型 | 将目标位置转为驱动力矩 | 目标、状态、控制参数 → 电压或电流与力矩 | [Paper: PDF p. 4, Section IV; PDF p. 5, Equation 14 and Equation 15] | 忽略控制器可能把控制误差误归因于摩擦；这是分析判断 |
| 激励轨迹与辨识 | 让不同物理效应得到激励 | 实机日志 → 参数 | [Paper: PDF p. 5, Section V] | 未报告每一类轨迹的独立消融 |
| 多关节验证 | 检验单摆辨识参数的迁移 | 参数、机械臂与路径 → 关节 MAE | [Paper: PDF p. 6, Section VI-B] | 若省去该步，会遗漏 Dynamixel M5 的泛化不足 |

六模型及摩擦参数数量：M1 库仑—黏性，2 个；M2 加 Stribeck，5 个；M3 加载荷依赖，3 个；M4 合并 Stribeck 与载荷依赖，7 个；M5 再加入驱动与反驱不对称，9 个；M6 再加入二次载荷项，11 个。这里不包括另外辨识的电机参数。[Paper: PDF p. 2, Equation 3; PDF p. 3, Section III-A to III-C; PDF p. 4, Section III-D to III-E; PDF p. 6, Section VI-A]

## 09 关键公式和符号

[Paper] 单摆动力学为：

\[
J\ddot\theta=\tau_m+\tau_e+\tau_f,\qquad \tau_e=-mgl\sin\theta.
\]

其中 J 为等效转动惯量，θ 为关节角，τm、τe、τf 分别为电机、外部和摩擦力矩。它给出了模型预测运动的物理基础。[Paper: PDF p. 2, Equation 1]

[Paper] 摩擦计算为：

\[
\tau_f^{stop}=-\left(\frac{J}{\Delta t}\dot\theta+\tau_m+\tau_e\right),\qquad
\tau_f=\operatorname{clip}(\tau_f^{stop},[-\tau_f^m,\tau_f^m]).
\]

τf^m 是可用摩擦额度，Δt 是仿真步长；额度足够时可以抵消运动趋势，不够时按上限施加阻力。[Paper: PDF p. 2, Equation 4; PDF p. 3, Equation 5]

[Paper] 对 Dynamixel 机械臂表现最好的 M4 为：

\[
\tau_f^m=K_v|\dot\theta|+K_c+K_l|\tau_m-\tau_e|
+\exp\left[-\left|\frac{\dot\theta}{v_s}\right|^\alpha\right]
\left(K_c^s+K_l^s|\tau_m-\tau_e|\right).
\]

Kv 控制黏性项，Kc 控制基础摩擦，Kl 控制载荷依赖；上标 s 对应低速附近增加的项，vs 和 α 控制其速度范围与曲线形状。[Paper: PDF p. 3, Equation 8]

其余公式索引，便于定位完整模型：

| 公式 | 作用与位置 |
|---|---|
| Equation 2；Equation 3 | 传统带符号摩擦写法与 M1 额度写法。[Paper: PDF p. 2, Section II-B] |
| Equation 6；Equation 7 | M2 Stribeck 模型与 M3 载荷依赖模型。[Paper: PDF p. 3, Section III-A and III-B] |
| Equation 9；Equation 10 | M5 方向性及 M6 二次载荷模型。[Paper: PDF p. 4, Section III-D and III-E] |
| Equation 11 | 执行器状态与目标位置到电机力矩的映射。[Paper: PDF p. 4, Section IV-A] |
| Equation 12；Equation 13 | 电压控制与含反电动势的电机力矩模型。[Paper: PDF p. 4, Section IV-B] |
| Equation 14；Equation 15 | 电流限制与电流到力矩的映射。[Paper: PDF p. 5, Section IV-C] |

## 10 实验设计与证据链

[Paper] 四款执行器：Dynamixel MX-64、MX-106，以及减速比不同的 eRob80:50、eRob80:100。每款约 100 段日志，每段 6 秒；改变质量、摆长和比例增益。所有模型都辨识相应的摩擦、电机参数与等效惯量。四类激励为变频振荡、叠加振荡、慢升慢降、抬升后释放。[Paper: PDF p. 5, Section V-A; PDF p. 6, Table I]

| 实验 | 检验的主张 | 对照与条件 | 结果 | 支持和边界 | 来源 |
|---|---|---|---|---|---|
| 单执行器验证 | 扩展模型减少仿真误差 | M1–M6，25% 留出日志 | 最优模型使基线误差除以 2.93、2.02、2.34、1.51 | 支持这四款执行器的留出轨迹结果；没有给出对所有伺服执行器的保证 | [Paper: PDF p. 6, Figure 5] |
| Dynamixel 2R 机械臂 | 参数可迁移到多关节系统 | 圆、方形、方波、三角波，高低增益 | M4 最佳；作者报告各路径和增益下 MAE 均不到 M1 的一半 | 支持两关节轨迹仿真；M5 辨识优势未保持 | [Paper: PDF p. 6, Figure 7 and Section VI-B] |
| eRob80 2R 机械臂 | 谐波减速器需要更复杂模型 | 同样四种路径及高低增益 | M6 最准确，作者报告误差相对 M1 减半以上 | 支持所测机械臂工况；不等于所有谐波执行器都应选 M6 | [Paper: PDF p. 6, Figure 7 and Section VI-B] |

单执行器误差降幅换算如下。百分比为根据原文比值计算的结果，不是原文另报的新实验：降幅 = 1 − 1 / 比值。

| 执行器 | 原文基线误差与最优模型误差之比 | 换算后的误差降低 |
|---|---:|---:|
| MX-64 | 2.93 | 约 65.9% |
| MX-106 | 2.02 | 约 50.5% |
| eRob80:100 | 2.34 | 约 57.3% |
| eRob80:50 | 1.51 | 约 33.8% |

依据：[Paper: PDF p. 6, Section VI-A]。指标是仿真与实测关节角的 MAE，不能解释为任务成功率或策略性能同比提高。

全部主图与表格的证据清单如下。自动提取器未识别这些图表标题，已人工补齐并核对核心页面：

| 图表 | 论证作用 | 位置 |
|---|---|---|
| Figure 1 | 用 Dynamixel 三角波路径直观展示 M1 与 M4 的末端轨迹差异 | [Paper: PDF p. 1, Figure 1] |
| Figure 2 | 定义驱动、静止与反驱区域 | [Paper: PDF p. 3, Figure 2] |
| Figure 3 | 展示静态边界测量方法与真实载荷依赖 | [Paper: PDF p. 3, Figure 3] |
| Figure 4 | 展示六模型各自能表达的驱动与反驱曲线形状 | [Paper: PDF p. 4, Figure 4] |
| Figure 5 | 四款执行器的留出日志 MAE 比较 | [Paper: PDF p. 6, Figure 5] |
| Figure 6 | 定义 2R 机械臂的四种测试路径 | [Paper: PDF p. 6, Figure 6] |
| Figure 7 | 比较两套机械臂的关节 MAE，以及高低增益下的模型表现 | [Paper: PDF p. 6, Figure 7] |
| Table I | 给出辨识实验质量、摆长和比例增益范围 | [Paper: PDF p. 6, Table I] |

## 11 结论的正确解读

[Analysis] 最有力的结论是：在本文所测伺服执行器与两关节系统中，考虑载荷、低速及部分方向性效应，可以明显降低运动仿真误差。M4 对 Dynamixel 2R 机械臂的表现说明，模型选择应以迁移验证为依据，不能仅追求单摆验证集最低误差。[Paper: PDF p. 6, Section VI]

[Analysis] 本文没有训练强化学习策略，也没有报告实机强化学习回报、步行成功率或跌倒率；因此不能声称已证明强化学习迁移性能提高。作者把复杂机器人及策略迁移列为未来工作。[Paper: PDF p. 7, Section VII-B]

[Analysis] 图 5 和图 7 没有误差条或置信区间；三次辨识的收敛描述不能替代跨硬件样本的重复性分析。正文使用“significantly”描述幅度，不应转述为已完成统计显著性检验。[Paper: PDF p. 5, Section V-B; PDF p. 6, Figure 5 and Figure 7]

## 12 作者明确承认的局限

| 局限 | 具体表现 | 作者提出的方向 | 来源 |
|---|---|---|---|
| 未考虑温度 | 模型没有描述热状态变化 | 加入热模型，并在采集时测量和控制温度 | [Paper: PDF p. 7, Section VII-A] |
| 忽略径向力 | 为简化实验装置而不建模 | 未在该段给出具体解决方案 | [Paper: PDF p. 7, Section VII-A] |
| 未考虑停留时间 | 静止持续时间对重新启动摩擦的影响未纳入 | 可引入摩擦响应延迟 | [Paper: PDF p. 7, Section VII-A] |

作者另在结果部分指出：M5 在 Dynamixel 2R 上不再改善，可能源于过拟合。这是作者提出的解释，尚无专门实验确证其原因。[Paper: PDF p. 6, Section VI-B]

## 13 批判性分析

以下均为基于本文证据的分析判断，不作为作者自己的表述。

| [Analysis] 观察 | 潜在问题 | 为什么重要 | 如何检验 | 依据 |
|---|---|---|---|---|
| 只有四种执行器和两套 2R 机械臂 | 尚未覆盖足式碰撞、全身耦合与更多硬件样本 | 限制对复杂机器人的推广 | 多台同型号执行器、完整机器人及接触任务验证 | [Paper: PDF p. 6, Section VI] |
| 用上一时间步外力矩 | 接触冲击时可能滞后 | 足式机器人落地时外力矩可能快速变化 | 不同步长与冲击条件下比较力矩和轨迹误差 | [Paper: PDF p. 5, Section IV-E] |
| 辨识集和验证集按日志 75%/25% 划分 | 未说明是否严格留出整个载荷或增益工况 | 区分同类条件插值与新工况泛化 | 留一载荷、留一增益、留一设备评估 | [Paper: PDF p. 5, Section V-B; PDF p. 6, Table I] |
| 只比较本文物理模型族 | 没有与学习式模型及其他动态摩擦模型同预算对比 | 无法得出优于所有建模方法的结论 | 相同数据、辨识时间、评价轨迹下比较 | [Paper: PDF p. 6, Figure 5 and Figure 7] |
| 同时辨识摩擦与电机参数 | 多组参数可能解释相近轨迹 | 预测拟合好不等于参数唯一或物理量准确 | 参数敏感性、置信区间与独立电气测量 | [Paper: PDF p. 5, Section VI-A; PDF p. 6, Section VI-A] |

## 14 可迁移的知识

Agent-derived knowledge candidates：

- [Analysis] 系统辨识要设计能区分物理效应的激励；本文“抬升后释放”通过控制电路状态减少电磁制动干扰，是值得学习的实验设计。[Paper: PDF p. 5, Section V-A]
- [Analysis] 高增益控制可能掩盖部分模型差异；高低增益并列测试能让模型评估更充分。[Paper: PDF p. 6, Figure 7]
- [Analysis] 拟合、留出日志验证和不同系统上的迁移验证应分开看待，后者能发现复杂模型的泛化问题。[Paper: PDF p. 6, Section VI-B]

## 15 与已有知识的联系

[Analysis] Agent-derived candidate：对机器人仿真工程，可以把执行器辨识与质量、惯量、接触参数校准并列处理。本文具体解决的是执行器内部动力学的一部分，不能独自解决整个仿真到现实的差距。[Paper: PDF p. 5, Section IV-E; PDF p. 7, Section VII-A and VII-B]

[External] 作者维护 BAM 开源项目，为扩展摩擦模型及辨识提供实现入口。[BAM 仓库](https://github.com/Rhoban/bam)

[Analysis] 若应用于其他型号舵机，应复用实验与辨识流程，再测量该型号的参数，而不是直接照搬 MX-64 或 MX-106 的参数。本文没有证明跨型号参数通用。[Paper: PDF p. 6, Section VI-A]

## 16 候选研究方向

Agent-derived research candidates；以下是待验证假设，不声称已有查新结论。

**候选方向：带温度状态的轻量摩擦辨识。**

- 起点：[Paper] 本文明确忽略温度。[Paper: PDF p. 7, Section VII-A]
- [Hypothesis] 如果长时间运行后的误差主要受温升驱动，那么让少量摩擦系数随温度变化，可以比固定参数 M4 更好地预测热机状态轨迹。
- 相对本文的变化：保持原有控制器和摩擦结构，仅让 Kc、Kl 等少量系数成为温度的简单函数；避免直接增加大量自由参数。
- 初步方法：在受控冷机、升温和热机状态采集相同激励，联合辨识温度系数，并按完整温度区间划分训练与测试。
- 如何验证：在相同日志数量和优化预算下，对比固定 M4、温度相关 M4 及具有相当参数量的固定模型。使用至少三台同型号执行器，每个工况重复三次；报告关节 MAE、低速启动误差、设备间差异及运行开销。如果未见温区外测试改善，或者收益仅来自更多参数，则假设不获支持。
- 可能失败：外壳温度不能准确代表齿轮接触温度；电阻变化和控制器温漂被误归因于摩擦；数据不足使温度项过拟合。
- 创新状态：unverified，需先检索温度相关摩擦建模的既有工作；原文参考文献 [13] 已涉及温度，不能把“考虑温度”本身声称为新颖贡献。
