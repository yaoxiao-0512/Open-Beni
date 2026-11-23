# RAIBO2 马拉松论文 + raisimGym_nature 代码 + G1 踝关节闭环文章精读与应用报告

- 日期：2026-11-23
- 阅读对象（三份）：
  1. 论文预印本：RAIBO2: Highly efficient quadruped robot completing full marathon with a single battery charge（Research Square rs-6040970 v1；已正式发表于 Nature，见 §1.1）
  2. 代码仓库：KAIST RAILAB `railabatkaist/raisimGym_nature`（上述论文的训练与评估代码）
  3. 技术文章：Kevin Zakka & Qiayuan Liao, “Closing the loop”（2026-11-16），Unitree G1 踝关节闭环传动在 MuJoCo 中的恢复
- 用途：BorderCollieRobot 项目（0.42× 边牧四足伴侣机器人）的对照与应用；同时存放在 open-beni 仓库（BENI 轮式腿机器人，mjlab + MuJoCo Warp，GRU 策略）作为研究档案
- 写作纪律：每个关键数字与结论都标注出处（PDF 章节/页码、仓库文件路径与行号、文章小节）。凡是没读到的东西，本报告明确写“未读到 / 无法确认”，不编数字。我们的项目事实只采用任务书给定的已确认事实。

## §0 阅读方法与范围说明

### 0.1 三份来源各是怎么读的

| 来源 | 阅读方式 | 完整度 |
|---|---|---|
| RAIBO2 预印本 PDF | 下载后 `pdftotext -layout` 全文提取，共 51 页、1716 行文本，逐节读完 | 正文、Methods、参考文献、Data/Code availability、Extended Data 图注全部读完；**Extended Data Table 1–4 是图片，pdftotext 无法提取其中的数值**（网络结构维度、奖励权重表、超参数表、地形参数表），本报告凡涉及这些表的内容只写正文与代码中能交叉验证的部分，并明确标注“表内数值未读到” |
| raisimGym_nature | `git clone --depth 1`（HEAD `14844f4`），读 README、cfg.yaml、Environment.hpp、RaiboController.hpp、module.py、ppo.py、runner.py、tester.py、URDF | 核心文件逐行读完；`rsc/policy/` 下 20 个预训练权重（4 变体 × 5 seeds）只核对了目录结构与文件命名，未加载权重验算 |
| Closing the loop | `browser.open` 全文 649 行（含全部代码块与 Table 1） | 全文读完；页面中的交互式 MuJoCo/WASM 图形无法运行，只读文字与代码 |

### 0.2 为什么这三份放在一起读

RAIBO2 论文与 raisimGym_nature 是同一工作的两面（论文讲“为什么高效”，仓库讲“训练到底怎么做的”），两者可以互相验证：论文 Methods 里的奖励描述可以在 `RaiboController.hpp` 里找到逐行实现，论文里的网络结构可以在 `module.py` 里找到逐行实现。第三份“Closing the loop”表面上是另一台机器人（Unitree G1 双足）的踝关节建模，但它直接命中我们项目当前最敏感的两个点：**踝关节的限位建模**（我们的踝是被动 B'-1 软弹簧铰链，刚度前 0.02 / 后 0.01 N·m/rad，范围 ±108°，2026-11-21 落地，覆盖 DOWN 所需的 102°）和 **mjlab 训练框架的限位处理**（我们的训练正是 MuJoCo + mjlab，obs 128-D / action 24-D）。所以第三份不是凑数的，它是为我们的踝与 mjlab 模型写的针对性阅读。

### 0.3 本报告的结构

- §1：RAIBO2 论文精读（方法、实验、结论、局限，带章节引用）
- §2：raisimGym_nature 仓库精读（结构、关键实现、与论文的对应）
- §3：“Closing the loop”精读（G1 踝关节闭环）
- §4：三份来源与我们项目的对比（RAIBO2 vs BorderCollieRobot vs open-beni/BENI）
- §5：对我们项目的应用建议（分 P0/P1/P2，哪些能直接借鉴、哪些要改造、哪些不适用，并说明理由）
- §6：一页纸结论
- 附录：出处清单与数值核对表

---

## §1 论文精读：RAIBO2

### 1.1 出处与基本信息

- **标题**：RAIBO2: Highly efficient quadruped robot completing full marathon with a single battery charge
- **作者**：Choongin Lee、Donghoon Youm、Jeongsoo Park（共同一作）等 17 人，通讯作者 Jemin Hwangbo，全部来自 KAIST Robotics and Artificial Intelligence Lab（RAILAB）（论文第 1 页作者栏）
- **版本**：Research Square 预印本，2025-03-26 发布，DOI `10.21203/rs.3.rs-6040970/v1`，CC-BY 4.0（PDF 第 1 页）；**已正式发表**：Nature，2026-09-23，DOI `10.1038/s41586-026-11102-5`（PDF 第 1 页 "Version of Record" 注记）。本报告读的是预印本 v1，不是 Nature 正式版——引用时注意版本差异的可能。
- **资助**：Samsung Research Funding and Incubation Center，项目号 SRFC-IT2002-02（Acknowledgments，p.39）
- **数据**：Zenodo `10.5281/zenodo.14825866`（Data availability，p.39）；**代码不公开**，只写 "available upon reasonable request"（Code availability，p.39）——这正是 raisimGym_nature 这个仓库的价值所在：它是论文训练代码的公开释放。
- **一句话概括**：RAIBO2 不是靠某一个 trick 省电，而是把“损耗分析 → 机械、电路、策略三域协同优化”做全了的系统工程，最后用一场真实马拉松验证。

### 1.2 核心结论与 headline 数字

| 指标 | 数值 | 出处 |
|---|---|---|
| 马拉松成绩 | 42.195 km，**4 h 19 m 52 s**，第 22 届 Sangju Gotgam 马拉松 | 摘要 p.1；§Efficiency validation p.15 |
| 平均速度 | 2.64 m/s | p.15 |
| 耗电 | **1280 Wh**，占 2016 Wh 电池的 63% | p.15；Fig.4g–h |
| 平均 TCOT（马拉松） | **0.248**（headline 取 0.25） | p.15；摘要 |
| 最低 COT | **0.237 @ 3 m/s**（跑步机实验） | Fig.3c |
| 人类基准 | 0.37；RAIBO2 是**首个低于人类 TCOT 的四足机器人** | 摘要 p.1；Fig.1d |
| 续航 | 理论可达 **67 km**（2016 Wh） | 摘要；p.4 |
| 相对优势 | 每 Wh 行驶距离约是现有四足机器人的 **3 倍以上** | 摘要 p.1–2；Fig.1e |
| 赛道难度 | 累计爬升 286 m，最大坡度 **18.4°**，含落叶湿滑沥青路段 | Fig.1c；p.15 |
| 电池电压 | 67.2 V → 57.6 V，截止电压 48 V | Fig.4h |

TCOT 的定义（Methods, Loss measurement, p.33）：总能耗（J）除以（质量 × 重力 × 距离）。马拉松 TCOT 用的是 45 kg（装 2016 Wh 电池的 RAIBO 整机质量）——注意这个 45 kg 与设计质量 40 kg（Methods Eq.1）不是同一个数，引用时要区分：**40 kg 是设计计算用质量，45 kg 是马拉松实测配置质量**。另：仓库 URDF 各 link 质量求和为 37.97 kg（§2.10 细述），三者各有出处，不要混用。

### 1.3 方法总览：三域协同优化（论文 Fig.1b、Fig.2）

论文的核心方法论是先做**全面的损耗分析**（§Quadruped robot loss analysis，p.9–14），再按损耗来源分配优化任务（Fig.1b）：

- **机械硬件**：力透明执行器（proprioceptive actuator）+ 轻腿（lightweight leg）
- **电路**：低损耗电机驱动电路（低导通损耗 + 低开关损耗）
- **策略**：RL 训练的节能步态（energy-optimized locomotion policy）

作者在 Introduction（p.3）明确批评了既往工作的局面：此前研究只做孤立改进（机械优化、姿态优化、RL 步态自适应各做各的），没有一篇工作把硬件设计与 locomotion policy 放在同一个平台上联合优化。RAIBO2 的 claim 就是“holistic optimization”。

控制层级（Fig.2c，p.8）：
- 状态估计：contact-aided invariant EKF（引 Hartley et al. 2018）
- 策略网络：GRU + MLP，**100 Hz**
- 电机控制：PD 控制 + 力矩控制，**15 kHz**
- 观测采样 1 kHz；控制 PC 与 12 个电机驱动器之间实时通信 **2 kHz**（EtherCAT）

### 1.4 损耗分析：299.7 W 的 Sankey 图（论文 §Quadruped robot loss analysis + Fig.3）

这是全篇最值得精读的部分。损耗在 3 m/s 最低 COT 工况下实测，总计 **299.7 W**，分解如下（Fig.3a；百分比为占总损耗比）：

| 损耗分量 | 功率 | 占比 | 性质 |
|---|---|---|---|
| 系统损耗（PC、IMU、通信 Hub、配电板焦耳热） | 28.7 W | 9.6% | 基线功耗 |
| 机械损耗小计 | 68.1 W | 22.7% | 随速度上升最陡（Fig.3b），是 COT 在 3 m/s 后回升的主因 |
| 其中：滑移损耗 | 8.8 W | 2.9% | 足-地滑动 |
| 其中：足碰撞损耗 | 19.3 W | **6.4%**（见 §1.10 注记：正文写 8.4%） | 触地冲击动能耗散 |
| 其中：关节摩擦损耗 | 40.0 W | 13.4% | 齿轮齿面摩擦（力透明度的反向指标） |
| 驱动器损耗小计 | 81.3 W | 27.1% | — |
| 其中：驱动器系统损耗 | 24 W | 8.0% | — |
| 其中：导通损耗 | 17.4 W | 5.8% | I²R |
| 其中：开关损耗 | 39.9 W | 13.3% | MOSFET 开关 |
| 电机损耗小计 | 121.6 W | 40.6% | — |
| 其中：铜损 | 120.6 W | 40.2% | **最大单项** |
| 其中：铁损 | 1.0 W | 0.3% | 可忽略 |
| **执行器电气损耗小计**（驱动器 + 电机） | **202.9 W** | **67.7%** | 全篇第一大头 |

校核：28.7 + 68.1 + 81.3 + 121.6 = 299.7 W ✓；81.3 + 121.6 = 202.9 W ✓。

几个关键的 trade-off 关系（论文 p.9–10，Fig.3g–h）：

1. **执行器电气损耗 vs 足碰撞损耗**：降电气损耗的手段（增大减速比、抬高身体）都会增大足的表观惯量，从而增大碰撞损耗；反之亦然。这是全文反复出现的核心矛盾。
2. **身体高度**：抬高身体 → 同样的足端力只需更小的关节力矩 → 电气损耗下降；但腿过度伸展会降低可操作性、难以抑制滑移，机械损耗上升。RL 训练的策略在 **86% 腿伸展**高度下能稳定跑 3 m/s，再高就站不住了（Fig.3g）。这是一个有实测数据的最优高度点，不是拍脑袋的。
3. **支撑相的能量流像人走路**：支撑相存在 generating（发电/吸能）与 motoring（驱动）两个阶段（Fig.3d），受控的电气刚度模仿机械弹簧：压缩时吸能形成近非弹性碰撞以稳定触地，储存的能量随后用于推进。Trot 步态中前腿主要负责发电、后腿主要负责推进（Fig.3e）——这是值得我们记下的步态能量分工观察。
4. **磁饱和**：最低 TCOT 工况下 **84.1% 的运行时间处于额定力矩以内**（Fig.3h）。超过额定力矩后磁饱和使单位电流的力矩增益递减，电机常数下降——这直接导向了 §1.7 的饱和感知电流奖励。

测量方法（Methods, Loss measurement, p.33–37）值得注意其严谨性：总功率直接测电池 Vbat×Ibat（15 kHz 采样，EtherCAT 2 kHz 传输）；铜损用厂家数据 kT=0.281 Nm/A、kM=0.87 算出 Reff = kT²/kM² = 0.104 Ω；驱动器损耗用“定子不带转子 + 直流负载电流”的办法隔离测出（消除铁损与反电动势）；铁损用“定子 + 虚拟角度旋转正弦电流”测出；机械损耗 = Σ(kT·iq·ωm − 铁损)；滑移损耗是**残差**（总机械损耗减去碰撞与摩擦），即它还包含一切未建模的机械损耗（Eq.19）——引用滑移损耗数值时要知道这个口径。

### 1.5 机械硬件（Methods, Mechanical hardware design, p.24–27）

- **四条相同的腿，每腿 3 个相同的执行器**，模块化（Fig.2a）。
- **膝关节四连杆传动 + 单级行星减速 7.15:1**；腿机构（不含执行器）**1.1 kg**，腿机构占整机重量的 **10.2%**（p.6）。
- 执行器需求推导（Methods p.24–25，Eq.1–5）：
  - 额定力矩：膝关节在 trot 支撑期持续承重，τrated = mgLcalf·cos(θknee/2)/nleg,trot = **44.34 Nm**（m=40 kg，Lcalf=0.32 m，θknee=90°，nleg,trot=2）
  - 峰值力矩：身体减速按 2g 计，τpeak = **88.68 Nm**
  - 足速需求：目标 4 m/s，按 Raibert 落足启发式推导足相对身体速度 8 m/s，关节角速度需求 **17.7 rad/s**
  - 由此定执行器指标：额定 25 Nm、峰值 90 Nm、最高 20 rad/s（191 rpm）——注意额定 25 Nm 低于 Eq.1 算出的 44.34 Nm，论文称"initial design targets were subsequently validated"，但正文没有展开 25 与 44.34 之间的差距如何处理（可能是两腿分担与 duty factor 的细节未写进正文，见 §1.9 局限）。
- **电机**：TQ ILM115x25，电机常数 kM = kT/√R = **0.87 Nm/√W**，额定 3.9 Nm、峰值 12.7 Nm、1400 rpm @48 V。选型原则：单级减速 <10:1，要求电机约 2.5 Nm 额定 / 9 Nm 峰值 / 1910 rpm，**在满足指标的电机里优先选 kM 高的**（p.25）。
- **行星齿轮**：减速比由力矩下限（额定 6.41:1、峰值 7.09:1）与速度上限（7.34）夹出，选 7:1 设计值，最终齿数 160-67-26（ring-planet-sun）、3 个行星轮、模数 0.7，实际 **7.15:1**，齿厚 10 mm（AGMA 简化公式 Eq.9）。齿圈外径约束为与电机直径相同（115 mm）。
- **轻腿**：转子惯量 ∝ 减速比平方，所以减转子惯量是关键——太阳轮空心轴 + 铝转子毂；交叉滚子轴承实现单端支撑；滚针轴承代替球轴承减重；各部件按静力分析做重量优化（Extended Data Fig.1a）。

### 1.6 电路（Methods, Low loss motor driver circuit design, p.27–29）

- 驱动板：MCU + 门驱动 + 三相两电平逆变器（6 MOSFET），霍尔电流传感器。
- **导通损耗**：2 oz 铜厚（全部层），相对 1 oz 等效电阻降约 1.8 mΩ；霍尔传感器 ACS725LLCTR-50AB-T，导通电阻 1.5 mΩ，损耗与 1 mΩ 分流电阻方案相当但无地噪声问题；MOSFET 对比：BSC070N10NS5SC（RDS,on 7 mΩ，Qg 30 nC）vs ISC022N10NM6（2 mΩ，73 nC），BSC 方案总导通损耗高约 33 mΩ等效。
- **开关损耗**：Psw = Vin·Iout·fsw·(Qgs+Qgd)/Ig。80 V 下 ISC 虽然导通电阻低，但门电荷大，总损耗反而高于 BSC（Extended Data Fig.2e）——**只看 RDS,on 选 MOSFET 会选错**，这是本节最实用的教训。
- 两招降开关损耗：门电阻从评估板数据手册的 180 Ω 降到 **10 Ω**（靠布线优化降寄生电感）；开关频率从 20 kHz 降到 **15 kHz**（2 kHz 主从通信下，兼顾效率与控制性能）。

### 1.7 策略训练（Methods, Locomotion policy training, p.30–32）

- **算法**：PPO（Schulman et al. 2017），POMDP 建模（接触力、地形不可观测），**非对称 actor-critic**（critic 只在仿真里用特权信息），仿真器 **Raisim**（Hwangbo et al. 2018 的 per-contact iteration 接触求解器）。
- **网络**：Actor = **GRU + MLP**（GRU 处理感官观测生成隐状态；隐状态 + 感官观测 + 速度命令进 MLP 输出动作）；Critic = MLP。速度命令**绕过 GRU 直接进 MLP**，防止命令历史累积（p.31）——这个细节值得我们对照：我们的 128-D obs 里速度命令是否也应该绕过 GRU，见 §5。
- **Actor 观测**：身体 roll-pitch 向量、角速度、关节位置/速度、上一步动作（Methods p.31）。**动作：12 维残差关节位置偏移**，加到标称关节位置上得到最终关节位置参考。
- **Critic 特权信息**：身体线速度、足接触持续时间（摆动/支撑时间）、身体高度、当前与上一步地面反作用力、触地前足速度、接触状态、足接触法角、地形类型、局部地形高度图（p.31）。注意：**触地前足速度同时出现在 critic 观测与奖励里**——奖励与观测的配套设计，见 §2.6。
- **奖励三类**：命令跟踪、运动风格、能量最小化（p.6–7）。对效率起关键作用的三项（p.32 + Extended Data Fig.4）：
  1. **足触地前速度最小化**（foot pre-contact velocity）：直接降足碰撞损耗；配合 GRF 平滑项。
  2. **磁饱和感知电流最小化**：超过额定力矩后电流需求与力矩平方成正比，策略因此学会限制高力矩区的使用（Fig.4f–g 的力矩直方图对比）。
  3. **GRF 平滑**（ground reaction force smoothness）：降机械损耗。
  消融实验在 4 m/s 跑步机上做（Extended Data Fig.4a–b；Extended Data Table 3b）。**奖励权重表在 Extended Data Table 3a（图片），本报告未读到具体权重数值**——仓库 cfg.yaml 里有全部系数（§2.6），两者可以互补引用。
- **训练设置**：每次 reset 随机化身体姿态、关节角、广义速度、速度命令；初始线/角速度当作外部扰动；速度命令 vx,vy,ωz ~ **U(−1,4)、U(−1,1)、U(−1,1)**（Eq.10）；课程学习逐步提高初始状态分布、速度命令与地形难度；**三类地形按足-地最大接触法角划分**：<90°（坡/丘）、=90°（台阶/楼梯，练抬足反射）、>90°（带鼻沿的楼梯/管道，练先缩足再摆足）（p.32；Extended Data Fig.3b）。
- **Sim-to-real**：用动作正则化损失（Mysore et al. 2021），**刻意减少对大范围域随机的依赖**，只建模传感器噪声与关节摩擦（p.32）——与 Peng et al. 2018 的 dynamics randomization 路线形成对照。这个选择值得我们在 §5 里认真讨论。

### 1.8 马拉松验证与能量再生（§Efficiency validation, p.15–16；Fig.4）

- 赛道：第 22 届 Sangju Gotgam 马拉松，往返路线，中途折返点；官方计时系统记录（引 SPCT Co. 2024）。
- 坡度实验：平地 / +7° 上坡 / −7° 下坡的损耗对比（Fig.4k）：上坡额外机械损耗 **+100.5 W**，下坡回收 **107.5 W**——下坡回收基本抵消上坡多耗，这是长距离能效的关键机制之一；功率波形里 generating/motoring 相位分明（Fig.4l）。
- 电压从 67.2 V 掉到 57.6 V，截止 48 V；1280 Wh 是 2016 Wh 的 63%，据此外推 67 km。

### 1.9 局限（Discussion, p.17–18，作者自述）

1. **盲控（blind controller）**：策略只针对典型场景的最小损耗优化，没有感知；楼梯、山地等地形有不同的损耗模型与姿态约束，需要环境感知的节能策略。
2. **假设有人操作**：自主导航需要能基于全局地图找能量最优路径的规划器，本文没做。
3. 作者认为损耗模型本身可作为后续工作的基础，并推广到双足、轮腿等平台。

### 1.10 本报告发现的预印本内部不一致（核对用，非结论性）

- **足碰撞损耗占比**：正文 p.10 写 "8.4 percent"，但 Fig.3a Sankey 图写 19.3 W（6.4%）；19.3/299.7 = 6.44%，与 Sankey 的 6.4% 一致，与正文的 8.4% 不一致（8.4% 对应 25.2 W）。其余分量（滑移 2.9%、摩擦 13.4%、系统 9.6%、驱动器 27.1%、铜损 40.2%、铁损 0.3%）正文与 Sankey 均一致。可能的原因是正文与图用了不同版本的数字（预印本 v1 的编辑残留），Nature 正式版可能已修正。引用足碰撞损耗时建议以 Sankey 的 19.3 W / 6.4% 为准，并注明正文另有一处 8.4%。
- **执行器额定力矩**：Eq.1 算出膝关节额定需求 44.34 Nm，但 p.25 定指标为额定 25 Nm、峰值 90 Nm；峰值 90 与 Eq.2 的 88.68 吻合，额定 25 与 44.34 的差距正文没有解释。可能与两腿分担、duty factor 或电机-齿轮联合选型有关，引用时不要把 44.34 当成最终执行器指标。
- **质量口径**：设计计算 40 kg、马拉松 TCOT 计算 45 kg、仓库 URDF 求和 37.97 kg（§2.10），三个数各有出处，引用时必须带定语。

---

## §2 仓库精读：raisimGym_nature

### 2.1 仓库定位

- 地址：`https://github.com/railabatkaist/raisimGym_nature`；本报告克隆 HEAD `14844f4`（"update README.md"）；许可证 MIT（Copyright (c) 2020 RaiSim Tech）。
- 定位（README 首行）："training and evaluation code used in our Nature submission"，即 RAIBO2 论文的训练与评估代码。
- **关键限制：仓库不含仿真器本身**。它必须与 `raisimLib`（RaiSim 官方库，另行克隆到同一 workspace）一起用；Raisim 是商业/闭源仿真器。这意味着**仓库本身不能独立复现训练**——没有 raisimLib 许可证就跑不起来。这是与我们的 MuJoCo + mjlab（开源）路线的本质区别，也是本仓库对我们的价值边界：奖励与网络结构可以照搬思想，训练 infra 不能照搬。
- 依赖：conda 环境 `rsg_raibo_nature`，Python 3.10.18，**PyTorch 2.5.1+cu121**，CUDA 12.1；`environment.yml` 钉死全部版本（含 `prefix: /home/user/anaconda3/envs/pj3` 这种作者本机残留路径，复现时须注意）。

### 2.2 仓库结构

```
raisimGym_nature/
├── README.md                      # 训练/测试说明
├── environment.yml                # conda 环境（torch 2.5.1+cu121）
├── setup.py                       # python setup.py develop 编译
├── CMakeLists.txt
├── raisimGymTorchNature/
│   ├── algo/ppo/                  # PPO：ppo.py / module.py / storage.py
│   ├── env/
│   │   ├── RaisimGymVecEnv.py     # 向量化环境 Python 侧
│   │   ├── VectorizedEnvironment.hpp / Yaml.* / raisim_gym.cpp
│   │   └── envs/rsg_raibo_nature/
│   │       ├── Environment.hpp    # 环境：reset/step/终止/随机化（593 行）
│   │       ├── RaiboController.hpp# 控制器：观测、奖励、PD、终止（695 行）
│   │       ├── RandomHeightMapGenerator.hpp  # 地形生成（17 KB）
│   │       ├── cfg.yaml           # 全部训练/奖励/课程超参
│   │       ├── runner.py          # 训练入口（244 行）
│   │       └── tester.py          # 评估入口（130 行）
│   └── helper/raisim_gym_helper.py
├── rsc/
│   ├── raibo2/urdf/raibo2.urdf   # 机器人 URDF（SolidWorks 导出）
│   ├── raibo2/meshes/            # 17 个 STL（TORSO + 每腿 HIP/THIGH/SHANK/FOOT）
│   └── policy/                    # 预训练权重：4 变体 × 5 seeds
│       ├── proposed/              # 正式方法
│       ├── remove_actuator_Joule_loss/
│       ├── remove_foot_collision_loss/
│       └── remove_grf_smoothness/
└── thirdParty/pybind11/           # 打包的 pybind11
```

每个 `policy/<变体>_<seed>/` 含 `full_50000.pt`（完整检查点）+ `mean/var/value_obs_mean/value_obs_var 50000.csv`（观测归一化统计）。

### 2.3 环境与训练配置（cfg.yaml 全文已读）

| 项 | 值 |
|---|---|
| 并行环境数 | 200（`num_envs: 200`），120 线程 |
| 仿真步长 / 控制步长 | 0.0025 s / 0.01 s（控制 100 Hz，与论文一致） |
| 每回合最长 | 4.0 s |
| 目标迭代数 | 50,000（`target_learning_episode_end: 50000`）；每 5000 保存/记录一次；每 50 评估一次 |
| 随机化 | 地形、观测噪声、关节摩擦三项开；`test_mode: False` |
| 网络 | GRU hidden 128；actor MLP [256, 128]；critic MLP [512, 256, 128] |
| 课程 | 每 5 次迭代更新一次；初始因子 0.2，衰减 0.97；地形初始因子 0.01，衰减 0.97 |

### 2.4 网络结构（algo/ppo/module.py 全文已读）

- **Actor = `GRU_MLP_Actor`**：单层 GRU（input = measurement_dim，hidden = 128）+ MLP（LeakyReLU，输入 = GRU 隐状态 + 全部输入，输出 = 动作）。关键结构细节与论文 Methods p.31 一致：**速度命令（3 维）不进 GRU，直接拼进 MLP 输入**——`forward` 里 `self.GRU(input[:, :, :measurement_dim])`，MLP 输入是 `torch.cat((latent, input), dim=2)`，其中 input 含命令的最后 3 维。`input_shape = [measurement_dim + 3]`。
- **Critic = MLP**（`mlp3_shape` [512, 256, 128]，LeakyReLU），输入为特权观测。
- **分布**：`MultivariateGaussianDiagonalCovariance`，对角高斯，初始标准差 3.5（runner.py），有 `enforce_minimum/maximum_std` 接口。
- **导出**：`Actor.save_deterministic_graph` 用 `torch.jit.trace` 导出——与我们的 ONNX 导出路线不同（我们走 ONNX [1,128]→[1,16]；RAIBO2 仓内是 TorchScript .pt，部署不在本仓库内）。

### 2.5 PPO 超参（ppo.py + runner.py 交叉核对）

| 超参 | runner.py 实际值 | ppo.py 默认值 | 说明 |
|---|---|---|---|
| clip | （未覆盖，用默认）0.2 | 0.2 | — |
| gamma | **0.99** | 0.998 | runner 覆盖了默认值，引用时以 runner 为准 |
| lam (GAE) | 0.95 | 0.95 | — |
| epochs / mini-batches | **32 / 1** | — | 每轮 32 个 epoch、单 mini-batch |
| policy lr / value lr | 5e-4 / **5e-4** | 5e-4 / 1e-3 | runner 把 value lr 从 1e-3 改为 5e-4 |
| lr scheduler | 每步 ×10^(−2/50000) | StepLR 0.99999 | runner 用指数衰减公式 |
| entropy coef | **0.005** | 0.0 | runner 覆盖 |
| value loss coef | 0.5 | — | — |
| smoothness loss coef | **0.025** | — | 动作平滑辅助损失（PPO 目标外的附加项，见 storage/ppo 的 smoothness 处理） |
| max grad norm | 0.5 | 0.5 | — |
| shuffle_batch | False | True（默认） | runner 显式关掉 shuffle |
| 每步采样量 | n_steps = 4.0/0.01 = 400 步 × 200 envs = 80,000 步/迭代 | — | 50,000 迭代 ⇒ 约 40 亿步采样量级（理论值；实际以 tensorboard 为准） |

注意：gamma、value lr、entropy coef 三处 runner 与 ppo.py 默认值不同，**只读 ppo.py 会读错**，必须以 runner.py 的传参为准。本报告已按 runner 为准核对。

### 2.6 奖励实现（RaiboController.hpp:384–499 + cfg.yaml，全文已读）

奖励共 **23 项**（`stepData_[0..22]`），聚合方式非常特殊（`getRewardSum`，RaiboController.hpp:500–551）：

```
positive = stepData[0..2] 之和   # 命令跟踪 + airtime + 身体高度
negative = stepData[3..20] 之和  # 其余全部惩罚项
total    = positive * exp(0.1 * negative)
```

即总奖励 = 正奖励 × exp(0.1 × 负奖励和)。负奖励越小（越接近 0），因子越接近 1；负奖励越负，因子指数衰减。这个聚合方式保证了**惩罚项只能打折正奖励，不能把总奖励打成绝对值很大的负数**——与常见的“正负直接相加”不同。这对我们的奖励设计有直接参考价值（§5 A4）。

全部 23 项与系数（cfg.yaml；系数符号：正项为正系数，惩罚项为负系数）：

| # | 奖励项 | 系数 | 实现要点（RaiboController.hpp 行号） |
|---|---|---|---|
| 0 | 命令跟踪（线速度 exp(−‖Δv‖²) + 角速度 exp(−1.5Δω²)，线速度项带 (1+exp(−0.5‖Δv‖)) 放大） | +1.5 | :388–392 |
| 1 | airtime（支撑/摆动时间，clip 到 ±0.25/0.2） | +0.9 | :432–441 |
| 2 | 身体高度（exp(−10‖Δh‖)，目标 0.5 m） | +0.5 | :394 |
| 3 | 足离地高度（foot clearance，高度扫描误差平方 × 足速 /(0.2+‖命令‖)） | −120.0 | :448–467；系数绝对值全表最大 |
| 4 | **触地前足速度**（foot-vel-before-contact） | −0.4 | :444–446；摆动相 airTime>0.05 s 时记录足速模长，触地瞬间计入惩罚 |
| 5 | 身体运动（垂向速度平方 + 0.02‖roll/pitch 角速度‖） | −20.0 | :396 |
| 6 | 力矩平方和 | −2.0e-4 | :398 |
| 7 | 关节速度平方和（×课程因子） | −1.6e-3 | :400 |
| 8 | 关节加速度平方和（×课程因子） | −1.0e-2 | :402 |
| 9 | 标称位置（站姿偏差， terrainLevel 越高权重越低，系数 1/(1+3·terrainLevel)；平地时 ×4） | −2.0 | :404–417 |
| 10 | 关节 roll 位置（HAA 关节角平方；高速时权重按 ‖vx‖/‖命令‖ 调节） | −2.5 | :433、:441 |
| 11 | 平滑 1（一阶动作差分平方和，课程渐入） | −5.0 | :420 |
| 12 | 平滑 2（二阶动作差分平方和，课程渐入） | −10.0 | :421 |
| 13 | **GRF 平滑**（GRF 二阶差分 0.5 倍 + 一阶差分平方和） | −3.0e-6 | :425 |
| 14 | 滑移（支撑相足水平速度平方和） | −8.0e-2 | :445 |
| 15 | 非期望接触（足/小腿以外的接触计数） | −1.0 | :473 |
| 16 | 关节限位（对数障碍式：log10 距限位距离平方和 /10，仅膝关节 i%3==2） | −1.0 | :479–497 |
| 17 | **关节功率（磁饱和感知电流）** | −2.0e-3 | :480–489，见下 |
| 18 | 非期望 GRF + 小腿接触力 | −1.0e-2 | :427 |
| 19 | 全腾空相（四足同时离地） | −2.0 | :476 |
| 20 | 姿态（acos(旋转矩阵 R33) 平方，平地 ×10） | −1.0 | :405 |
| 21/22 | （聚合用：positive / negative 小计） | +1.0 / +1.0 | :501–528 |

**三项关键奖励的代码级实现**（论文 §1.7 的对应物）：

1. **磁饱和感知电流（jointPowerReward，:480–489）**：
   ```
   motorTorque = |jointTorque| / 7.15
   if motorTorque < 4 Nm:  current = motorTorque / 0.281        # 线性区，kT = 0.281 Nm/A
   else:                   current = 1.53 + 2.65·motorTorque + 0.13·motorTorque²  # 饱和区，二次增长
   jointPowerReward += coeff · (current² · (0.104 + 0.0148) + current · 0.21)
   ```
   其中 0.104 Ω 就是论文 Eq.13 的 Reff（kT²/kM²），0.0148 与 0.21 是驱动器侧的拟合系数（论文 Methods p.34 的驱动器损耗多项式 Eq.14 的对应物）。**4 Nm 是额定力矩对应的电机侧阈值**（电机额定 3.9 Nm）。这段代码是全仓库最值得抄的部分：它把“磁饱和”从论文文字变成了三行可执行的分段函数。
2. **GRF 平滑（:425）**：`0.5·(GRF[t−2]+GRF[t]−2·GRF[t−1])² + (GRF[t−1]−GRF[t])²`，即二阶差分与一阶差分并罚，系数 −3.0e-6。注意 GRF 在这里是 critic 特权信息的一部分，但奖励计算在控制器侧直接取用。
3. **触地前足速度（:444–446）**：摆动相且 airTime > 0.05 s 时持续记录 `FootVelBeforeContact_[i] = ‖footVel‖`；足一触地，该值乘系数 −0.4 计入奖励。实现极简，但需要“摆动相计时 + 触地事件”两个信号——在 mjlab 里对应的是接触状态的边沿检测，见 §5。

另注意：**关节限位奖励只对膝关节（i%3==2）生效**（:491），且用 log10 距离障碍——这是个很克制的选择，不是全关节限位惩罚。

### 2.7 观测与特权信息（RaiboController.hpp:282–357；Environment.hpp:515）

- **Actor 观测 45 维**（`obDim_ = 45`，:622）：身体姿态行向量（3）+ 身体角速度（3）+ 关节位置（12）+ 关节速度（12）+ 上一步动作（12）+ 速度命令（3）。与论文 Methods p.31 的描述逐项一致。
- **观测噪声**（observation randomization，:296–306）：姿态 σ=0.05、角速度 σ=0.22、关节位置 σ=0.055、关节速度 σ=0.55（高斯加性）。注意关节速度噪声 0.55 rad/s 是个不小的值——域随机的量级本身就是个超参。
- **Critic 特权观测**（`updateValueObservation`，:318–357）：actor 观测（42 维，不含命令）+ 身体线速度（3）+ air/stance 时间（8）+ 身体高度（1）+ GRF（4）+ 上一步 GRF（4）+ 触地前足速（4）+ 接触状态组（16：足接触/非期望足接触/非期望 GRF/小腿接触 ×4 腿）+ 接触法角（4）+ 地形类型（3）+ 高度扫描（4×scanConfig）+ 命令（3）。与论文 p.31 的特权信息清单逐项一致。
- **标称关节姿态**：`[0, 0.580099, −1.195] × 4`（HAA 0、HFE 0.58、KFE −1.195 rad）（:48）；目标身体高度 0.5 m（:385）。
- **力矩限幅**：71.5 Nm（Environment.hpp:567 注释："current limit: 40A -> torque limit: 10Nm, consider gear ratio: 7.15 --> torque limit: 71.5Nm"）。
- **终止条件**（`isTerminalState`，:555 起）：非足/小腿部位接触即终止；关节超限位即终止。

### 2.8 课程与随机化（Environment.hpp + cfg.yaml）

- **地形**：`RandomHeightMapGenerator.hpp`（17 KB），三类地形（SLOPE/HILLS/台阶/带鼻沿楼梯与管道），与论文 Extended Data Fig.3b 的三类划分一致；`terrainLevel` 进入标称位置奖励的权重（1/(1+3·terrainLevel)），即地形越难，越不强求标称姿态——这是个细腻的设计。
- **地面摩擦**：支撑相时足-地摩擦系数每步重采样 **U(0.1, 0.4)**（Environment.hpp:337），即每次触地都可能换摩擦——比固定摩擦的 DR 更狠。
- **平滑奖励课程**：smoothReward1/2 的系数按 `min(iter, target_smoothness_end)/target_smoothness_end` 渐入（:420–421），`target_smoothness_end` 由 runner.py 按 `50000/50000 · ln(3.25/0.05) · 10000` 算出——即平滑约束是**后半程才加满**的，不是开局就罚。这个时序值得我们借鉴（§5）。
- **初始状态**：reset 时随机化身体姿态、关节角、广义速度与速度命令；初始线/角速度作为外部扰动（与论文 p.32 一致）。

### 2.9 消融实验与预训练策略（tester.py + rsc/policy/）

- Tester 的 `--weight_type`（0–3）对应四种权重：`proposed`（正式方法）、`remove_actuator_Joule_loss`（去掉执行器焦耳损耗项）、`remove_foot_collision_loss`（去掉足碰撞损耗项）、`remove_grf_smoothness`（去掉 GRF 平滑项）；`--seed` 0–4；`--vis_target` 可视化关节目标。
- 每个变体 5 个 seeds，检查点统一为 `full_50000.pt`（50,000 迭代）。**消融范式 = 一次只去掉一项 + 5 seeds + 固定速度跑步机对照**（论文 Extended Data Fig.4 在 4 m/s 跑步机上做）。这是我们做奖励实验时应该直接抄的范式（§5 B1）。
- Tester 支持 pygame 手柄（Axis 1 前后、Axis 0 横向、Axis 3 转向、Button 0 换地形、Button 1 重置），可视化用 raisimUnity。

### 2.10 与论文的对应关系核对表

| 论文说法 | 仓库实现 | 核对结果 |
|---|---|---|
| GRU + MLP actor，命令绕过 GRU | module.py `GRU_MLP_Actor` | ✓ 逐行一致 |
| 动作 12 维残差关节位置 | runner.py `act_dim` = 12（环境定义）；module 输出 12 | ✓ |
| Actor 观测 45 维 | `obDim_ = 45` | ✓（论文未给总数，仓库补全） |
| 铜损 Reff = 0.104 Ω | RaiboController.hpp:489 `(0.104 + 0.0148)` | ✓ |
| 磁饱和分段电流 | :483–487（4 Nm 阈值 + 二次式） | ✓（论文只讲原理，仓库有公式） |
| GRF 平滑 / 触地前足速 | :425 / :444–446 | ✓ |
| 三类地形（接触法角） | RandomHeightMapGenerator + cfg | ✓ |
| 奖励聚合 positive·exp(0.1·negative) | :525–530 | 论文未提聚合公式，**仓库独有信息** |
| PPO gamma 0.99 / epochs 32 | runner.py | 论文 Extended Data Table 2d（图片）未读到，仓库补全 |
| 机器人总质量 40 kg | URDF 各 link 求和 **37.97 kg**（躯干 19.86 kg；另有注释 `<!--6.78667481-->` 疑似旧值） | ⚠ URDF 与论文设计质量差 2 kg，引用时注明口径；躯干注释值未证实，不采用 |
| 减速比 7.15 | URDF 注释 `<!--<gear_ratio value="7.154"/>-->`（已注释掉）；控制器用 7.15 | ✓ |
| 训练 200 envs / 120 线程 | cfg.yaml | ✓ |

### 2.11 可复现性评估（诚实结论）

- **能复现的部分**：奖励函数、网络结构、PPO 超参、地形课程、消融范式——代码全在，逐行可读。
- **不能独立复现的部分**：① Raisim 仿真器本身不在仓库内（商业许可）；② 论文 Extended Data Table 2（网络维度/超参表）与 Table 3（奖励权重表）是图片，本报告未读到表内数值——但 cfg.yaml 与代码补全了大部分，交叉核对后可信；③ 预训练权重在仓内，但没有 raisimLib 就加载不了；④ 论文的马拉松实测数据在 Zenodo（10.5281/zenodo.14825866），本报告未下载核对。
- 对我们的实际意义：**抄思想与奖励实现，不抄 infra**（§5 详述）。

---

## §3 文章精读：Closing the loop（Unitree G1 踝关节闭环）

### 3.1 出处与问题

- 作者：Kevin Zakka、Qiayuan Liao；2026-11-16；https://kzakka.com/closing-the-loop/（全文 649 行，含代码块与 Table 1；配图为浏览器内实时运行的 MuJoCo/WASM，未运行，只读文字与代码）。
- 背景引用：MuJoCo（Todorov et al. IROS 2012）；**mjlab**（Zakka et al., arXiv:2601.22074, 2026）；LAFAN1 动作重定向（Harvey et al. TOG 2020）。
- 问题：Unitree G1 的常见仿真模型把每只踝写成两个独立驱动的串联关节（pitch + roll），这做了两个近似：① 把踝的活动范围当作 pitch–roll 空间的矩形；② 忽略电机到足的传动。真实 G1 是小腿里的两个电机经**曲柄 + 推杆**驱动足部，两根推杆形成两个闭环运动链，传动比与反射惯量随踝姿态变化，机械限位进一步约束可达姿态。文章要做的就是把这两样东西在 MuJoCo 里恢复出来。

### 3.2 机械限位实测与角面编码（"Measuring the ankle's range of motion" / "Encoding the mechanical stops"）

- **实测方法**：把 G1 仰放，徒手移动每只足遍历活动范围，同时记录 pitch 与 roll（电机释放态与阻尼态的测量结果在 1° 内一致）。
- **实测结果**：中立 pitch 附近，roll 可达 **16–18°**（略超串联模型的 ±15°）；pitch 走到极限时，可用 roll 急剧收窄，矩形的四个角被切掉。串联模型在足充分上勾（toe-up）时仍允许 ±15° roll，而真机几乎不允许 roll。
- **量化影响**：用原矩形限位做 LAFAN1 重定向时，**1.5% 的着地足姿态超出机械限位 1° 以上**（在 pitch–roll 空间里沿垂直方向量）。
- **编码方法（角面 / corner face）**：保留串联模型的 ±15° roll 限位（略小于中立附近实测的 16–18°，取保守值），再用四条直线各切掉矩形的一个角。每条直线是一个线性不等式：
  ```
  c_r · θ_roll + c_p · θ_pitch ≤ b
  ```
  MuJoCo 的 fixed tendon 计算关节坐标的线性组合，所以每条不等式实现为一条带上限的 tendon。代码（文章原样，左踝）：
  ```xml
  <joint name="left_ankle_roll_joint" range="-0.2618 0.2618"/>
  <tendon>
    <fixed name="left_ankle_face0" limited="true" range="-100 0.13708">
      <joint joint="left_ankle_roll_joint" coef="-0.2618"/>
      <joint joint="left_ankle_pitch_joint" coef="0.2618"/>
    </fixed>
    <fixed name="left_ankle_face1" limited="true" range="-100 0.13708">
      <joint joint="left_ankle_roll_joint" coef="0.2618"/>
      <joint joint="left_ankle_pitch_joint" coef="0.2618"/>
    </fixed>
    <fixed name="left_ankle_face2" limited="true" range="-100 0.22846">
      <joint joint="left_ankle_roll_joint" coef="-0.14967"/>
      <joint joint="left_ankle_pitch_joint" coef="-0.2618"/>
    </fixed>
    <fixed name="left_ankle_face3" limited="true" range="-100 0.22846">
      <joint joint="left_ankle_roll_joint" coef="0.14967"/>
      <joint joint="left_ankle_pitch_joint" coef="-0.2618"/>
    </fixed>
  </tendon>
  ```
  直线近似与实测角的误差约 2°（手描轨迹在角部越过角面约 2°）。下限 −100 是 inactive 的，只有上限起限位作用。右踝同理（自己的四条 tendon）。
- **mjlab 警告（原文明确，见 §3.8）**：mjlab 这类训练框架从**关节 range**推导软限位、reset 截断与限位惩罚，而不是从 tendon 约束推导。用 tendon 编码限位后，这些功能必须同步考虑 tendon 限位，例如把四个边界向内收并把 reset 姿态投影进所得区域。

### 3.3 闭环传动重建（"Modeling the transmission"）

- **闭合方程**：每根推杆约束曲柄尖 p(u) 与足附着点 a(x) 之间的距离等于杆长 L：‖p(u) − a(x)‖ = L。u 为两个电机角，x 为踝的 pitch/roll。两根杆各贡献一个约束，踝从 4 个自由度（2 电机 + 2 踝关节）降为 2 个独立自由度。
- **闭式 IK**：曲柄尖在圆上，p = c + ρ(e₁cos u + e₂sin u)；代入杆长约束得 αcos u + βsin u = γ，其中 α = 2ρ(c−a)·e₁、β = 2ρ(c−a)·e₂、γ = L² − ‖c−a‖² − ρ²。解为：
  ```
  u = atan2(β, α) ± arccos(γ / √(α²+β²))
  ```
  两个符号对应两种装配（assembly）；|γ| = √(α²+β²) 时两解重合（杆长极限），|γ| > √(α²+β²) 时无解。求导即得传动雅可比 **G = ∂h/∂x**（u = h(x)），满足 u̇ = Gẋ。
- **力矩与惯量映射**：虚功原理，τ_x = Gᵀτ_u；动能等价，反射惯量 I_x = GᵀI_uG。两者都是局部的（在物理装配分支上、远离奇异点成立），且都随踝姿态变化。MuJoCo 通过闭环约束自动捕捉这两项（只要电机侧惯量与驱动器建模正确），串联模型做不到。

### 3.4 两种实现：tendon vs rod-body（"Enforcing closure in MuJoCo" + Table 1）

MuJoCo 的 body 层级必须是运动树，不能直接把杆连在曲柄与足之间。两种实现：

**（a）Spatial tendon + equality（推荐用于训练）**：
```xml
<tendon>
  <spatial name="rod">
    <site site="crank_tip"/>
    <site site="foot_anchor"/>
  </spatial>
</tendon>
<equality>
  <tendon tendon1="rod" solref="0.01 1" solimp="0.95 0.99 0.0001"/>
</equality>
```
MuJoCo 把 tendon 长度固定为 `qpos0` 时的值，因此**参考位形必须满足连杆几何**。每根杆只贡献一个标量约束，力沿两端连线作用，复现理想无质量杆的几何与力传递，不引入额外 body 或关节。

**（b）显式杆 body + connect 约束**：杆作为刚体（实测/估计的惯性），两端用两个铰链（不用球铰——球铰会让细杆绕自身轴自由旋转，成为几乎无阻力的自由度）从曲柄垂下，另一端用 `connect` equality 连到足上。相比 tendon 模型，每根杆多 2 个自由度与 2 行约束。

**Table 1（文章原表，逐项转录）**：

|  | Serial（串联） | Tendon | Rod body |
|---|---|---|---|
| 电机角度与速度作为状态 | – | ✓ | ✓ |
| 连杆可达集（不含机械限位） | – | ✓ | ✓ |
| 随姿态变化的力矩/速度/惯量 | – | ✓ | ✓ |
| 各电机自己的力矩限幅（驱动器在电机上） | – | ✓ | ✓ |
| 杆的质量、惯量与几何 | – | – | ✓ |
| 杆端关节（含摩擦或限位） | – | – | ✓ |
| 增加的自由度（双踝） | 0 | 4 | 12 |
| 闭环约束行数（双踝） | 0 | 4 | 12 |

约束求解参数：两种实现都用 10 ms 时间常数、单位阻尼比、阻抗从 0.95 到 0.99（前 0.1 mm 违反量）。Rod-body 模型里 MuJoCo 默认的 body-based 约束空间惯量对角近似（冻结在 `qpos0`）会把轻杆的 `connect` 行缩放得很差，须开 **`diagexact`**（在当前位形精确计算对角）。

### 3.5 力矩与增益映射的实测影响（"Motor torque and reflected inertia"）

- 每电机限幅 ±25 N·m 时，踝可用力矩是随姿态变形的平行四边形；串联模型则允许每轴固定 ±50 N·m。
- **力矩饱和很少成为瓶颈**：LAFAN1 重定向的承重足样本中，<0.1% 需要任一电机超过 25 N·m（几乎全是足载荷 >1.2 倍体重的样本）。结论：**串联模型 + 机械限位对动作重定向与初期控制器训练可能已经足够，部署前再用闭环模型验证**。
- **增益映射的影响更大**：连杆把电机侧刚度/阻尼变换到踝侧，K_u = G⁻ᵀK_xG⁻¹，D_u = G⁻ᵀD_xG⁻¹。即使踝侧增益是对角阵，电机侧也会出现耦合项；独立电机 PD 不能精确复现这个耦合（可用前馈力矩补）。数值例子：串联训练模型在每踝关节用 28.5 N·m/rad（mjlab 给两个踝电机的合计增益）；若改在每电机上用单电机增益 14.25 N·m/rad，标称站姿（踝 pitch −0.363 rad）下的等效踝 pitch 刚度约 **62 N·m/rad**（训练值的两倍多），roll 刚度约 **19 N·m/rad**（约三分之二）。256 次配对 rollout（6 s 舞蹈片段）：独立电机增益的失败率从 **16% 升到 20%**（对照组为训练好的踝空间控制器经 G⁻ᵀ映射到电机）；失败定义：躯干/足/手任一高度偏离参考 >0.25 m，或躯干系重力向量垂直分量偏离参考 >0.8；电机均限幅 ±25 N·m。
- 如果机器人板载控制器已经实现了期望的踝空间阻抗，这个映射不需要在外部做。

### 3.6 仿真实践（"Simulating the closed loop"）

**（1）一致的状态初始化**：reset 到动捕帧时，帧只给定踝 pitch/roll，不给电机角；只改踝关节不改电机，杆长约束就被违反。MuJoCo 对 equality 约束是软执行，靠恢复力纠正；小偏差无害，不一致的 reset 会产生大的纠正力、扰动目标足姿。解决办法：恢复仿真前先满足杆长约束——用 MuJoCo 的约束残差与雅可比，对电机角做几步 Gauss–Newton 迭代（固定踝姿态），再线性求解满足微分约束的电机速度。文章给出完整代码（`equality_jacobian` + `assemble`，free 指标是 qvel 指标而非 qpos 指标；无阻尼 Gauss–Newton 只适用于已接近有效装配的状态，如动捕 reset）。
**G1 闭式 reset**：杆长方程有闭式解，可直接算电机角（`close_crank` 函数：atan2 ± arccos，选离当前曲柄角最近的装配分支；再由 (p−a)·(ṗ−ȧ)=0 与 site 雅可比算电机速度，杆与曲柄对齐时判奇异）。随机浮动基座姿态/速度测试中，两踝闭合到机器精度，与迭代求解器在容差内一致。RL 需要数千次 reset 时，闭式解比迭代更有价值。

**（2）积分器稳定性**：G1 的控制接口接受踝 pitch/roll 目标，因此在恢复连杆后仍保留踝关节上的位置驱动器、电机关节被动——保留了踝空间控制接口。但 `implicitfast` + 4 ms 步长下这个配置不稳定：杆长误差快速增长，一秒内连杆张开。根因：`implicit`/`implicitfast` 把驱动器阻尼计入有效惯量 M̂=M+hD，但约束求解仍用原质量矩阵 M；当位置驱动器直接驱动 pitch/roll（自身惯量小，阻力主要来自连杆另一侧的电机）时，不匹配在约 2 ms 以上步长就导致闭环误差增长。`Euler` 与 `RK4`（显式处理驱动器阻尼）在较大步长下也不稳定（取决于 loop 表示）。**`discrete` 积分器对约束求解与动力学更新使用同一有效惯量 M̂=M+hD+h²K（含驱动器刚度项），在测试的全部步长下保持稳定，文章推荐该配置**。驱动器改放到电机上时，全部测试积分器都稳定。实验用 MuJoCo 4.0.1；模型要求 MuJoCo ≥3.16（`discrete` 积分器本身 3.13 引入）。

### 3.7 选择模型的建议（"Choosing a model"，文章原意转述）

- 动作重定向与部分训练问题：**串联模型 + 机械限位可能已足够**（结构简单，适合 RL）。
- 电机侧动力学重要时：tendon 模型（不增加显式杆 body 就捕捉随位形变化的传动）。
- 杆的质量/惯量/碰撞也重要时：rod-body 模型。
- 实用路线：**用最简单的、能捕捉相关行为的模型训练，部署前在闭环模型上验证控制器**——验证时还须保证一致的状态初始化与合适的积分器。

### 3.8 mjlab caveat（原文，"Encoding the mechanical stops" 末段）

> "Training frameworks such as mjlab derive soft limits, reset clipping, and limit penalties from joint ranges, not tendon constraints. With the stops encoded this way, these features must also account for the tendon limits, for example by shrinking the four boundaries inward and projecting reset poses into the resulting region."

即：**mjlab 从关节 range 推导软限位、reset 截断、限位惩罚，而不是从 tendon 约束推导**。用 tendon 编码限位后，这三处功能必须同步考虑 tendon 限位（例如把四个边界向内收，并把 reset 姿态投影进所得区域）。这对我们的 mjlab 训练有直接意义，见 §5 A9。

---

## §4 与我们项目的对比

### 4.1 三方对照总表

| 维度 | RAIBO2（论文+仓库） | BorderCollieRobot（我们） | open-beni / BENI |
|---|---|---|---|
| 机器人 | 四足，12 腿关节，40 kg（设计）/45 kg（马拉松配置） | 四足，14-DoF（12 腿关节 + 2 脊柱），仿真约 10.43 kg，0.42× 边牧 | 轮腿机器人：2 腿（每腿 L1/L2 + 轮 LW/RW）+ 头 yaw/roll，MJCF 共 9 关节 |
| 尺度/质量 | 全尺寸工作犬级 | 桌面伴侣级（0.42×，约 1/4 质量） | 小型桌面机器人 |
| 驱动 | 自研执行器：ILM115x25 + 单级行星 7.15:1 + 膝四连杆；额定 25 Nm / 峰值 90 Nm | RobStride RS00 一体舵机 ×14（12 腿 + 2 脊柱），48 V、CAN 1 Mbps | 未在本报告核对驱动器型号（beni_robot.py 未细读） |
| 仿真器 | **Raisim**（商业闭源） | **MuJoCo + mjlab**（开源，GPU 向量化） | **MuJoCo + mjlab**（MuJoCo Warp） |
| 训练框架 | raisimGym（C++ 环境 + PyTorch PPO） | mjlab（rsl_rl 系） | mjlab（rsl_rl 系） |
| 策略网络 | **GRU + MLP**（GRU hidden 128，MLP [256,128]）；critic MLP [512,256,128] | GRU（open-beni 同款路线；部署 ONNX [1,128]→[1,16]） | **单层 GRU-512**（部署要求，headless_export.py:72），MLP (512,256,128)（rl_cfg.py:643–647） |
| Obs / Action | Actor 45-D / 动作 12-D 残差关节位置 | **128-D / 24-D**（16-D 关节位置：14-D 前缀 + 脊柱 yaw2 + 脖子 pan，再加 8-D 每腿阻抗占位） | **40-D / 7-D**（mdp.py:40 确认 POLICY_OBSERVATION_DIM = 40） |
| 训练规模 | 200 envs × 120 线程（CPU），50,000 迭代，sim dt 0.0025，控制 100 Hz | GPU 向量化（mjlab；BENI 用 4096 envs，见 open-beni README） | 4096 envs，15,000 PPO 迭代（README） |
| 速度目标 | 高速高效：3–4 m/s，vx 命令 U(−1,4) | **慢速 locomotion（walk/trot）是重点** | 速度跟踪 −2~2 m/s + 跳跃 0.18 m 参考 |
| 核心难点 | 能效（TCOT 0.25）、长航时 | **髋伸展、步幅、平滑性、sim-to-real** | 速度跟踪、恢复、跳跃 |
| 踝关节 | 未在论文中单独讨论（腿末端为足；膝四连杆是传动） | **被动 B'-1 软弹簧铰链**：刚度前 0.02 / 后 0.01 N·m/rad，范围 ±108°（2026-11-21 落地，覆盖 DOWN 所需 102°） | 轮式足（LW/RW 为轮关节，无踝铰链） |
| 奖励重点 | 能量最小化三项：触地前足速、磁饱和电流、GRF 平滑 | （任务书未给定当前奖励清单，本报告不编） | 速度跟踪、恢复、跳跃奖励（mdp.py/jump.py/recovery.py） |
| 部署 | 未公开（代码 on request） | ONNX [1,128]→[1,16]，AGX Orin | ONNX + MNN 双导出（headless_export.py） |
| 硬件阶段 | 成熟：马拉松实测 | **CAD 优先，MJCF 目前停着等 CAD 闭合** | 已开源可跑 |

### 4.2 相似处（值得我们信心的）

1. **GRU + MLP 是共识结构**：RAIBO2（GRU-128）、open-beni/BENI（GRU-512）都在用 GRU 处理本体感知历史 + MLP 输出动作；我们的 GRU 路线与学界最强四足工作同构，不存在结构路线的方向性风险。
2. **残差关节位置动作**：RAIBO2 的动作是 12 维残差关节位置（加到标称姿态上）；我们的 16-D 关节位置输出同样是位置参考 + PD 的结构，属于同一范式。
3. **非对称 critic + 特权信息**：RAIBO2 的 critic 拿身体线速度、GRF、触地前足速、地形高度图；我们的 128-D obs 含 52-D 追加块（任务背景），特权信息的思路一致。
4. **平滑是共同硬门**：RAIBO2 用 smooth1/smooth2 + GRF 平滑 + 动作正则化损失；平滑性也是我们明确的难点之一（任务背景）。两边的手段可以互通（§5）。
5. **轻腿 + 力透明是共同设计哲学**：RAIBO2 腿 1.1 kg 占 10.2%、单级减速 <10:1；我们的 RS00 直驱 + 减重轮（结构优化减重）是同一方向的不同实现。

### 4.3 不同处（决定什么能抄、什么不能抄）

1. **目标速度段完全不同**：RAIBO2 优化 3–4 m/s 的高速能效；我们优化慢速 walk/trot 的髋伸展与步幅。RAIBO2 的高速结论（例如 86% 腿伸展最优高度、4 m/s 消融）不能直接平移到慢速段——慢速的损耗结构不同（碰撞与铜损占比会变），须在我们的速度段重测。
2. **质量差 4 倍**：40–45 kg vs 10.43 kg。RAIBO2 的力矩/电流数值（25/90 Nm、40 A、71.5 Nm 限幅）全部不能照搬；只能搬**形式**（分段饱和函数、I²R 结构），常数必须用 RS00 的参数重标定。
3. **仿真器不同**：Raisim（CPU、per-contact 迭代）vs MuJoCo + mjlab（GPU、Warp）。RAIBO2 的训练 infra（200 envs、120 线程、50k 迭代）对我们没有参考价值；我们的 mjlab 4096 envs 路线在吞吐上更现代。
4. **踝的形态不同**：RAIBO2 的足是刚性末端（论文未讨论踝自由度）；G1 的踝是双电机曲柄-连杆主动踝；我们的踝是**单自由度被动弹簧铰链**。三者各不相同，§5 对 kzakka 的应用必须按“被动单自由度”逐项过筛，不能把 G1 的结论平移过来。
5. **开发阶段不同**：RAIBO2 是成熟系统做马拉松；我们是 CAD 优先、MJCF 暂停。RAIBO2 的硬件级结论（电机选型、驱动板设计）对我们当前阶段基本不适用——我们的执行器已经定型为 RS00，驱动板不在设计范围内。

---

## §5 对我们项目的应用建议

分级：P0 = 可直接借鉴/低成本高价值；P1 = 需改造或实验验证；P2 = 长期储备或明确不适用。每条都说明理由与落地形式。

### P0——直接借鉴（建议下一轮训练/奖励实验优先做）

**A1. 加一项“触地前足速度”奖励（RAIBO2 奖励 #4，系数 −0.4）**
- 依据：论文 p.32 + Extended Data Fig.4c–e；代码 RaiboController.hpp:444–446。触地前足速直接决定足碰撞损耗（论文 Eq.18：碰撞损耗 ∝ 触地前足速的二次型），而我们的慢速难点之一正是“足抬不起来/拖地”与平滑性——触地前足速是比“足抬多高”更直接的物理量。
- 落地形式：在 mjlab critic 特权观测里加“触地前足速”（RAIBO2 的 critic 本来就有这一项，p.31），奖励项在足触地瞬间按摆动相末段足速模长计罚。实现需要接触状态的边沿检测（摆动→支撑的切换），mjlab 的接触信号可以直接取。系数不要照搬 −0.4（RAIBO2 的奖励聚合是 positive·exp(0.1·negative)，系数含义不同），先在我们自己的奖励尺度上做小范围扫描。
- 注意：不要与已有的“足抬高度”奖励重复计罚——先核对我们当前奖励清单（本报告未读到，不编），再决定是新增还是替换。

**A2. 加一项 GRF 平滑奖励（RAIBO2 奖励 #13，系数 −3.0e-6）**
- 依据：论文 p.32；代码 :425：`0.5·(GRF[t−2]+GRF[t]−2·GRF[t−1])² + (GRF[t−1]−GRF[t])²`。GRF 的二阶差分直接度量触地冲击的陡峭程度，与我们的平滑硬门（0.5 rad/s）是同一物理直觉的不同量度：我们罚关节速度抖动，他们罚地面反作用力抖动。
- 落地形式：mjlab 里 GRF 可从接触力取（critic 特权信息已有 GRF 类信号的先例，见 RAIBO2 critic 清单）。先做离线测量：在现有策略的 rollout 里算 GRF 二阶差分，看它与我们的平滑门（dv）是否相关——如果相关，再进奖励；如果不相关，说明慢速段的 GRF 冲击不是主要矛盾，不必加。

**A3. 奖励聚合改用 positive·exp(0.1·negative) 形式（小规模 A/B）**
- 依据：RaiboController.hpp:500–551。RAIBO2 的总奖励 = 正奖励 × exp(0.1 × 负奖励和），惩罚项只能打折正奖励，不能把总奖励打成绝对值很大的负数。这对我们有直接意义：我们的奖励项很多（任务背景未给清单，但 128-D obs / 24-D action 的训练必然有多项惩罚），直接相加时某一项惩罚过大可能淹没命令跟踪信号。
- 落地形式：不要全量切换。先在现有奖励上做一次离线 A/B：同一批 rollout，分别用“直接相加”与“positive·exp(0.1·negative)”算总奖励，看后者是否改善“命令跟踪项被惩罚项淹没”的情况（可通过各奖励项的量级占比判断）。有效再进训练。

**A4. 平滑奖励后半程渐入（RAIBO2 的课程时序）**
- 依据：RaiboController.hpp:420–421 + runner.py：smooth1/smooth2 系数按 `min(iter, target_end)/target_end` 渐入，`target_end` 由 `ln(3.25/0.05)·10000` 量级算出——即**平滑约束是后半程才加满的**。直觉：开局先学会走，再学会走得平滑；开局就重罚平滑会压制探索。
- 落地形式：如果我们的平滑惩罚目前是全程满系数，建议改成课程渐入（前 1/3 迭代小系数，后 2/3 加满），并与 A1/A2 的引入节奏配合。这是一个零硬件成本、只改课程表的实验。

**A5. 速度命令绕过 GRU，直接进 MLP（结构微调）**
- 依据：论文 p.31 + module.py `GRU_MLP_Actor.forward`。RAIBO2 与 open-beni 都把速度命令绕过 GRU（防止命令历史累积污染隐状态）。我们的 128-D obs 里如果速度命令也进了 GRU，建议检查：命令是否应该走旁路。这个改动只动网络输入拼接方式，不动 obs 定义与部署接口（ONNX [1,128]→[1,16] 不变），成本低。
- 落地形式：先在代码里确认我们 actor 的 GRU 输入是否包含命令维；如果包含，做一次对照实验（旁路 vs 全进 GRU），以速度跟踪误差为指标。

**A6. 消融范式：一次只去掉一项 + 5 seeds + 固定速度对照**
- 依据：tester.py `--weight_type` 0–3 × `--seed` 0–4；论文 Extended Data Fig.4 在 4 m/s 跑步机上做消融。RAIBO2 的四个变体（proposed / 去焦耳损耗 / 去足碰撞 / 去 GRF 平滑）各 5 个 seeds，是奖励实验的教科书范式。
- 落地形式：我们下一轮奖励实验（例如引入 A1/A2 时）照此办理：基线 vs 去 A1 vs 去 A2，每个 3–5 seeds，固定速度段对照（我们的慢速段，例如 0.3–0.5 m/s），不要一次改多项后无法归因。

**A7. 评估里加 TCOT / 单位距离能耗度量**
- 依据：论文 Methods p.33（Ptotal = Vbat·Ibat；TCOT = P/(mgv)）。我们有 session-based 电池模型（任务背景：玩一会儿回充），但评估里是否以 TCOT 度量能效，本报告未读到。
- 落地形式：在 rollout 评估脚本里加两列：平均功率（仿真里用 Σ|τ·ω| 或电流模型估算）与 TCOT = P/(mgv)。不进奖励，只做评估指标——先看我们的慢速策略 TCOT 在什么量级，再决定要不要把能效推进奖励。这一步零训练成本。

### P1——需改造或实验验证

**B1. 磁饱和感知电流奖励：形式可借，常数必须重标定**
- 依据：论文 p.32 + RaiboController.hpp:480–489。分段函数的形式（线性区 I=τ_m/kT；饱和区 I=1.53+2.65τ_m+0.13τ_m²；功率项 I²·R + I·线性）可以直接抄结构，但 **0.281、4 Nm、0.104 Ω、0.0148、0.21 全部是 RAIBO2 的电机/驱动器专属常数**，对 RS00 完全不适用。
- 落地形式：先查 RS00 的力矩-电流曲线（手册/实测）：RS00 是 48 V 一体舵机，内部 FOC 与电流环由驱动器处理，我们的训练侧只能拿到关节力矩——要做这一项，需先标定“关节力矩 → 电机电流”的映射（额定点 + 饱和拐点）。在标定完成前，不要把 RAIBO2 的常数写进我们的奖励。另：我们的关节力矩量级（10 kg 级机器人）远小于 RAIBO2（40 kg），饱和区可能根本踩不到——先测现有策略的力矩分布，看有多少比例超过 RS00 额定，再决定这一项有没有意义。

**B2. 动作正则化损失 + 轻域随机（sim-to-real 路线对照）**
- 依据：论文 p.32（Mysore et al. 2021 的 action-regularizing loss；只建模传感器噪声与关节摩擦，不过度依赖大范围域随机）。我们的 sim-to-real 难点正在于此（任务背景）。
- 落地形式：与我们现有的 DR 清单逐项对照：RAIBO2 只随机化地形、观测噪声（σ 见 §2.7）、关节摩擦、地面摩擦 U(0.1,0.4)。如果我们的 DR 范围比这个宽得多，值得做一次“收窄 DR + 加动作正则化”的对照实验——过度 DR 会拖慢收敛、学出保守步态，这正是慢速段“步幅小、拖步”的可能成因之一。但这只是假说，必须以实验为准，不要直接收窄。

**B3. 身体高度作为效率-稳定 trade-off 做扫描**
- 依据：论文 Fig.3g（86% 腿伸展最优；再高站不住）+ §1.4 第 2 点。我们的站姿高度是否处于最优，值得在慢速段系统扫描一次。
- 落地形式：在仿真里固定速度、扫描 3–5 个站姿高度，记录命令跟踪误差、足碰撞指标与功率代理量（Σ|τ·ω|），画一条我们自己的“高度-损耗”曲线。注意 RAIBO2 的 86% 是高速段的结论，慢速段的最优点可能不同——不要直接取 86%。

**B4. 地形课程按“最大接触法角”分级**
- 依据：论文 p.32 + Extended Data Fig.3b（三类地形：<90° / =90° / >90°）。我们的地形课程如果目前只按高度/粗糙度分级，可以参考这个按接触法角的分类——它直接对应“足会不会被台阶鼻沿卡住”这个物理机制，比纯高度更贴合 sim-to-real 的失效模式。
- 落地形式：先核对我们现有的地形课程定义（本报告未读到），再决定是否引入法角分级。地形生成在 mjlab 里已有工具，不必重写生成器。

**B5. kzakka 角面限位：方法论记下来，但我们的踝目前用不上（见 §5.3 C1 的诚实分析）**
- 依据：§3.2。角面技巧的前提是**两个自由度耦合**（pitch 与 roll 的可达集不是矩形）。我们的踝是**单自由度**被动铰链（pitch），限位就是一维的 ±108°，不存在二维耦合——角面编码无的放矢。
- 落地形式：不做。但把方法记为设计储备：如果将来踝加横滚自由度（例如 B'-2 被动球窝方案复活），或发现 pitch 与某个自由度存在耦合限位，再启用 tendon 不等式。届时 §3.2 的代码可以直接抄（改系数即可）。

### P2——长期储备或明确不适用（诚实写不转移的）

**C1. G1 的闭环传动建模（曲柄-连杆、G、闭式 IK）对我们的踝不适用**
- 理由：G1 的踝是**双电机主动曲柄-连杆**，需要恢复的是电机→足的传动；我们的踝是**被动弹簧铰链，无电机、无连杆**，没有需要恢复的传动。kzakka 的全部传动内容（§3.3–3.5：闭合方程、G、τ_x=Gᵀτ_u、增益映射、62 vs 19 N·m/rad 例子）对我们的踝没有对应物。
- 唯一相关的类比：我们的**膝/髋是 RS00 直驱**，同样没有连杆传动——整机不存在需要闭环建模的传动环节。**如果将来踝改成主动曲柄传动，才需要这套方法**；在此之前，§3.3–3.5 只作为方法储备，不进入任何训练或模型改动。
- 例外（诚实补充）：kzakka 的“串联模型 + 机械限位可能已足够，部署前在闭环模型上验证”这一工作流（§3.7）对我们是普适的——我们现在用的正是串联树模型；将来如果模型里加入任何闭环结构（例如四连杆膝），就按这条工作流走。

**C2. kzakka 的积分器/diagexact/初始化问题目前不适用，但要记为设计约束**
- 理由：我们的 MJCF 模型是串联运动树，没有 equality 约束，没有闭环——`implicitfast` 不稳定、Gauss–Newton 初始化、`diagexact` 这些问题**现在不存在**。
- 落地形式：不做任何改动。但记一条设计约束进项目文档：**将来任何在 MJCF 里引入 equality 约束（tendon/closed-loop）的改动，必须同时满足三条**：① reset 时用闭式或 Gauss–Newton 保证约束一致（kzakka §3.6(1)）；② 积分器改用 `discrete` 或把驱动器放到电机侧（§3.6(2)）；③ rod-body 表示时开 `diagexact`。这三条是 kzakka 用实验换来的，不要等踩坑再发现。

**C3. mjlab 的 tendon-limit caveat：现在不踩坑，但必须记下来（预防项）**
- 依据：§3.8 原文。mjlab 从关节 range 推导软限位、reset 截断、限位惩罚；tendon 编码的限位不会被这三处自动考虑到。
- 对我们的现状：我们的模型目前**没有 tendon 限位**（全部限位都是关节 range），所以这个 caveat 现在不触发。但它是一个**沉默的接口假设**：将来任何人（包括未来的我们）如果为了编码耦合限位而引入 fixed tendon，必须同步处理这三处——否则训练用的软限位与 reset 截断会与真实限位不一致，策略会学到 tendon 限位外的姿态，部署时被机械限位挡住（kzakka 的 1.5% 超限位姿态就是前车之鉴）。
- 落地形式：在 MJCF/mjlab 模型文档里加一条注记（不改代码）：“若引入 tendon 限位，须同步处理 mjlab 的软限位/reset 截断/限位惩罚（参考 Zakka & Liao 2026 §Encoding the mechanical stops）”。等真有 tendon 的那一天再执行。

**C4. RAIBO2 的硬件级结论不适用（执行器已定型）**
- 电机选型（kM 优先、ILM115x25）、行星齿轮设计（160-67-26、模数 0.7）、驱动板（2 oz 铜、MOSFET 选型、门电阻 10 Ω、15 kHz）——全部针对自研执行器与驱动板。我们的执行器是现成的 RS00 一体舵机（48 V、CAN），驱动板不在设计范围内。**这些内容不进入任何设计改动**，只作为“为什么 RS00 路线（低减速比直驱、力透明）与 RAIBO2 的设计哲学一致”的佐证：RAIBO2 选单级减速 <10:1 的理由（力透明、降摩擦）同样适用于我们选直驱 RS00。
- 膝四连杆传动：我们的膝是直驱，无连杆，不适用。

**C5. 马拉松式长航时验证不适用，但度量方法可借**
- 42 km / 67 km 续航不是伴侣机器人的目标（室内使用、session-based 电池模型、玩一会儿回充）。不做长航时实验。但 §5 A7 的 TCOT 度量可以借——度量本身与目标无关。

**C6. RAIBO2 的高速段结论不要平移到慢速段**
- 86% 腿伸展最优高度、4 m/s 消融、3 m/s 最低 COT——全部是高速段的数。我们的慢速段（walk/trot，<1 m/s 量级）损耗结构不同：碰撞损耗占比下降，铜损与静止支撑损耗占比上升。引用 RAIBO2 的结论时必须带速度定语，慢速段的结论要我们自己测（B3 的高度扫描就是第一步）。

---

## §6 一页纸结论

1. **RAIBO2 是系统工程的范本，不是某个 trick**：299.7 W 的损耗账本里铜损占 40.2%（最大单项），执行器电气合计 67.7%；机械 22.7%、系统 9.6%。马拉松 42.195 km / 4h19m52s / TCOT 0.25（人类 0.37），67 km 理论续航。方法论是“先算清损耗账，再按账本分配三域优化”。
2. **仓库把论文变成了可执行的代码**：23 项奖励（含磁饱和分段电流、GRF 平滑、触地前足速）、GRU+MLP（命令旁路）、PPO（gamma 0.99、epochs 32、50k 迭代）、消融范式（一次去一项 × 5 seeds）。最值得抄的是三行分段电流函数与 `positive·exp(0.1·negative)` 的奖励聚合。
3. **G1 踝文章的精华是方法论**：角面限位（tendon 不等式）、闭环传动（G、闭式 IK）、仿真实践（一致初始化、`discrete` 积分器、`diagexact`）、mjlab tendon-limit caveat。对我们的踝（被动单自由度），传动部分不适用，限位方法与 mjlab caveat 是必须记下来的储备。
4. **对我们最优先的三件事**：① 加触地前足速奖励（A1）；② 加 GRF 平滑奖励并先离线验证相关性（A2）；③ 平滑奖励后半程渐入 + 消融范式（A4/A6）。这三件都不动硬件、不动 obs/action 定义、不动部署接口，可以在 MJCF 重开后直接进奖励实验。
5. **不要抄的东西**：RAIBO2 的电机/驱动板常数（全部是 40 kg 级专属）、高速段结论平移到慢速、Raisim infra、G1 的闭环传动建模（我们的踝是被动的）。

---

## 附录 A 出处清单

### A.1 论文（RAIBO2 预印本）

- 全文：`https://assets-eu.researchsquare.com/files/rs-6040970/v1_covered_f0e801d4-5457-4dd1-95ef-06b933d6df6b.pdf`（本地 `/tmp/raisim_nature/paper.pdf`，3,194,707 字节，51 页；`paper.txt` 1716 行）
- 预印本 DOI：10.21203/rs.3.rs-6040970/v1（2025-03-26）；正式发表 Nature DOI：10.1038/s41586-026-11102-5（2026-09-23）
- 关键页码（PDF 页码）：摘要 p.1–2；Fig.1 p.5；Fig.2 p.8；损耗分析 p.9–14（Fig.3 p.12）；马拉松 p.15–16（Fig.4 p.16）；Discussion p.17–18；Methods 机械 p.24–27；电路 p.27–29；策略训练 p.30–32；损耗测量 p.33–37；数据/代码 p.39

### A.2 仓库（raisimGym_nature）

- 地址：`https://github.com/railabatkaist/raisimGym_nature`（本地 `/tmp/raisimGym_nature`，HEAD `14844f4`）
- 关键文件：`README.md`；`raisimGymTorchNature/env/envs/rsg_raibo_nature/cfg.yaml`；同目录 `Environment.hpp`（593 行）、`RaiboController.hpp`（695 行）、`runner.py`（244 行）、`tester.py`（130 行）、`RandomHeightMapGenerator.hpp`；`raisimGymTorchNature/algo/ppo/module.py`、`ppo.py`、`storage.py`；`rsc/raibo2/urdf/raibo2.urdf`；`rsc/policy/`（4 变体 × 5 seeds）

### A.3 文章（Closing the loop）

- 地址：`https://kzakka.com/closing-the-loop/`（Kevin Zakka & Qiayuan Liao，2026-11-16；全文 649 行）
- 关键小节：Measuring the ankle's range of motion；Encoding the mechanical stops（含 mjlab caveat）；Kinematic closure（含闭式解）；Enforcing closure in MuJoCo；Table 1；Motor torque and reflected inertia；Consistent state initialization；Choosing a stable integrator；Choosing a model

### A.4 我们项目事实的出处（任务书给定）

- BorderCollieRobot：0.42× 边牧、14-DoF（12 腿 + 2 脊柱）、仿真约 10.43 kg、MuJoCo + mjlab、obs 128-D、action 24-D（16-D 关节位置 + 8-D 每腿阻抗占位）、部署 ONNX [1,128]→[1,16]、慢速 locomotion 重点、髋伸展/步幅/平滑性/sim-to-real 难点、CAD 优先、MJCF 暂停、RTX 3090 + AutoDL
- 踝：被动 B'-1 软弹簧铰链，刚度前 0.02 / 后 0.01 N·m/rad，范围 ±108°（2026-11-21 落地，覆盖 DOWN 所需 102°）
- open-beni：`Mondo-Robotics/Open-Beni`，commit `4545b86`（2026-09-24），MIT；BENI 轮腿机器人，mjlab + MuJoCo Warp，单层 GRU-512（`python/beni_rl/rl_cfg.py:643–647`、`headless_export.py:72`），POLICY_OBSERVATION_DIM = 40（`python/beni_rl/mdp.py:40`），动作 7-D（项目既有记录），MJCF `assets/beni_base/mjcf/beni.xml`（9 关节：root + head_yaw/head_roll + L1/L2/LW + R1/R2/RW）

## 附录 B 数值核对表（本报告引用的全部关键数字，一处出处）

| 数字 | 值 | 出处 |
|---|---|---|
| 马拉松距离/时间 | 42.195 km / 4h19m52s | 论文摘要；p.15 |
| 耗电/电池 | 1280 Wh / 2016 Wh（63%） | p.15；Fig.4g |
| TCOT 平均/最低/人类 | 0.248 / 0.237@3m/s / 0.37 | p.15；Fig.3c；摘要 |
| 理论续航 | 67 km | 摘要；p.4 |
| 总损耗 @3m/s | 299.7 W | Fig.3a |
| 铜损/铁损 | 120.6 W（40.2%）/ 1.0 W（0.3%） | Fig.3a |
| 执行器电气/机械/系统 | 202.9 W（67.7%）/ 68.1 W（22.7%）/ 28.7 W（9.6%） | Fig.3a |
| 足碰撞/滑移/摩擦 | 19.3 W（6.4%）/ 8.8 W（2.9%）/ 40.0 W（13.4%） | Fig.3a（正文另写碰撞 8.4%，见 §1.10） |
| 额定内运行占比 | 84.1% @3m/s 最低 TCOT | p.13；Fig.3h |
| 腿重/占比 | 1.1 kg / 10.2% | p.6 |
| 减速比/齿数 | 7.15:1 / 160-67-26，模数 0.7 | Methods p.26 |
| 电机 kM/额定/峰值 | 0.87 Nm/√W / 3.9 Nm / 12.7 Nm，1400 rpm@48V | Methods p.25 |
| τrated/τpeak/ωreq | 44.34 Nm / 88.68 Nm / 17.7 rad/s | Methods Eq.1/2/5 |
| 门电阻/开关频率 | 180 Ω → 10 Ω / 20 kHz → 15 kHz | Methods p.29 |
| 策略频率 | 100 Hz；PD+力矩 15 kHz；通信 2 kHz | Fig.2c |
| 速度命令分布 | vx U(−1,4)，vy U(−1,1)，ωz U(−1,1) | Methods Eq.10 |
| 奖励聚合 | positive·exp(0.1·negative) | RaiboController.hpp:525–530 |
| 奖励项数 | 23（stepData_[0..22]） | RaiboController.hpp:500–528 |
| 触地前足速/GRF平滑/关节功率系数 | −0.4 / −3.0e-6 / −2.0e-3 | cfg.yaml |
| 足离地高度系数 | −120.0 | cfg.yaml |
| Reff | 0.104 Ω（kT 0.281 Nm/A） | 论文 Eq.13；RaiboController.hpp:489 |
| 饱和阈值 | 电机侧 4 Nm；I = 1.53+2.65τ+0.13τ²（饱和区） | RaiboController.hpp:483–487 |
| Actor obs / 动作 | 45-D / 12-D | RaiboController.hpp:622；Methods p.31 |
| GRU hidden / MLP | 128 / [256,128]；critic [512,256,128] | cfg.yaml；module.py |
| PPO gamma/epochs/lr | 0.99 / 32 / 5e-4（runner 为准） | runner.py:82–100 |
| 训练规模 | 200 envs，120 线程，50,000 迭代 | cfg.yaml |
| 力矩限幅 | 71.5 Nm | Environment.hpp:567 |
| 标称关节/目标高度 | [0, 0.580099, −1.195]×4 / 0.5 m | RaiboController.hpp:48、:385 |
| URDF 总质量 | 37.97 kg（躯干 19.86 kg） | rsc/raibo2/urdf/raibo2.urdf（本报告实测求和） |
| G1 roll 实测 | 中立附近 16–18°；串联模型 ±15° | kzakka §Measuring |
| G1 超限位姿态占比 | 1.5%（>1°） | kzakka §Measuring |
| G1 电机限幅/串联限幅 | ±25 N·m / ±50 N·m 每轴 | kzakka §Motor torque |
| G1 饱和样本占比 | <0.1% 需 >25 N·m | kzakka §Motor torque |
| G1 增益例子 | 62 vs 19 N·m/rad @pitch −0.363 rad；失败率 16%→20%（256 rollout） | kzakka §Mapping controller gains |
| Tendon/rod-body 增加自由度 | +4 / +12（双踝）；约束行 4 / 12 | kzakka Table 1 |
| 积分器推荐 | `discrete`；implicitfast 4 ms 不稳定 | kzakka §Choosing a stable integrator |
| MuJoCo 版本要求 | ≥3.16（discrete 自 3.13）；实验用 4.0.1 | kzakka §Try the models |
