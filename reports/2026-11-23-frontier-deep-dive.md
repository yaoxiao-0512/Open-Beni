# 2026-11-23 晨报四题深入调查 + 借鉴分析报告

- 日期：2026-11-23
- 阅读对象（四题）：
  1. WALL-OSS-1.0（X Square Robot，2026-11-23 发布）：原生 VLA 预训练、Qwen3.5 底座、312 个真机任务零样本 55.1%、Lite 4B / Nano 2B
  2. Google SayTap 专利 US 12801150 B2（2026-11-17 授权）：白话 → 每腿一行 0/1 的四腿触地模式 → 改步态
  3. Galileo C1-W（11-16 部署、11-23 更新）：四足仓库巡逻每 3 小时一班、稳定 400+ 小时、IP65、光学+热成像+声学三传感
  4. TongDou 多轨 timeline + touch gesture（Week 8 最后一道未取题）：多轨 timeline 表情编排、touch gesture 触摸输入
- 用途：BorderCollieRobot 项目（0.42× 边牧四足伴侣机器人）的对照与借鉴；同时存放在 open-beni 仓库作为研究档案
- 写作纪律：每个关键数字与结论都标注出处（官网/论文/专利/仓库文件路径与行号、新闻通稿段落）。凡是没读到的东西，本报告明确写“未读到”，不编数字。我们的项目事实只采用任务书给定的已确认事实（见 §0.3），不凭记忆补充项目数字。晨报原文只作线索，不作证据。

## §0 阅读方法与范围说明

### 0.1 四题各是怎么读的

| 题 | 一手来源（已读） | 阅读方式与完整度 |
|---|---|---|
| WALL-OSS-1.0 | X Square Robot 官博全文 + GlobeNewswire 通稿全文 | 官博 745 行全部读完；通稿 149 行全部读完。官博图表中的数值只取正文写明的部分；图表本身未逐像素核对 |
| SayTap / US 12801150 B2 | SayTap 论文 arXiv:2306.07580v1 全文（813 行）+ saytap.github.io 项目站 | 论文正文、附录 A（随机模式生成器）、附录 B（奖励设计）、附录 C（训练配置）全部读完。**专利原文未读到**：Google Patents 规范地址返回 HTTP 404（见附录 D 的 F-1），权利要求号、说明书细节均未读到；专利授权事实与标题以 Hot Off The Patent Press 通讯（2026-11-23，二手）+ Patexia 审查员页（申请号 19/079035，二手、抓取时状态仍为 Pending）交叉佐证，正文凡引专利特有细节处均标注“仅见于二手通讯” |
| Galileo C1-W | Galileo Robotics PR Newswire 通稿（2026-11-16，56 行）全文 | 通稿全文读完；另读 The Robotics Media 报道全文（2026-11-16 首发、2026-11-23 更新）与 RobotsUSA 经销商规格页全文（216 行，二手规格）作交叉核对；天津市政府英文站只读了搜索摘要，未作一手引用 |
| TongDou | gettongdou.com 官网全文（220 行）+ GitHub 发布页 + Hackaday 文章全文 + gitdiagram 架构图 | 官网、发布页、Hackaday 全文读完；gitdiagram 架构图（2026-10-03 据文件树/README/源码绘制，二手）只取组件与文件路径清单。**仓库本体未克隆**：克隆中途因执行风险提示被叫停（见附录 D 的 F-2），仓库 README 与源码树未直接读到；本报告关于 TongDou“有什么 / 没有什么”的结论以以上四处网页来源为据，并在 §4.5 逐项说明证据等级 |

### 0.2 本报告的结构

- §1：WALL-OSS-1.0 精读（模型、评估、部署、开源状态）
- §2：SayTap 专利 + 论文精读（接口、控制器、奖励、实验）
- §3：Galileo C1-W 精读（部署事实、规格、运维模式）
- §4：TongDou 精读（实际 repo 是什么、记忆说法错在哪里、timeline 与 touch gesture 的核查结论）
- §5：四题借鉴分析（每题：能借什么 / 不能借什么 / P0/P1/P2，映射到我们哪个模块）
- §6：一页纸结论
- 附录 A：出处清单；附录 B：数值交叉核对表；附录 C：未读项清单；附录 D：失败运行清单

### 0.3 本报告采用的我们项目事实（仅任务书给定，逐字为准）

- AGX 上跑 Qwen3-VL-2B（:8010）；另有 Qwen3-14B-FP8 vLLM（:8011）；AGX 余约 8 GiB 显存；统一内存带宽约 204.8 GB/s 是绑定约束。
- 步态指令路径：contract 表 1.0/2.5/4.0 Hz，B2 (a) 已定、落地排队中；`_freq_cmd` 停下待定。
- Session 制电池模型（玩一会儿回充）。
- 表情层 v0.3（torso 呼吸慢正弦 + 耳朵尾巴程序化），下一步是从“各自写死”走向编排。
- Touch gesture：触摸 sensor 到货后的输入（手势语言抚摸/轻拍/抱 vs 生理信号传感器；软件先行定手势→drives 契约）。
- 除以上五条外，本报告不引用任何其他项目数字；凡涉及项目现状而任务书未给的，一律写“任务书未给定”。

### 0.4 页脚口径说明

- 本报告按任务书口径带 LINES / SELF / FULL 页脚（SELF = md5 of `head -n -2`，FULL = md5 of `head -n -1`）。注：今早 RAIBO2 报告（662 行）实测文件末尾并无页脚行，本报告的页脚格式以任务书定义为准，不以 RAIBO2 文件末尾为准。
- P0/P1/P2 口径沿用 RAIBO2 报告 §5：P0 直接可用、P1 需适配、P2 明确不适用。

---

## §1 WALL-OSS-1.0（X Square Robot，2026-11-23）

### 1.1 出处与基本信息

- **发布**：2026-11-23，GlobeNewswire 通稿（SHENZHEN，2026-11-23T07:44:46Z）+ X Square Robot 官博《WALL-OSS-1.0: Understand More, Act Better》（官博引文格式自称 2026-11-23）。
- **定位**：vision-language-action（VLA）基础模型；自称“原生 VLA 预训练”（native VLA pretraining）——视觉、语言、物理动作在统一训练过程中共同塑造模型能力，而不是先训 VLM 再挂动作。
- **底座**：Qwen3.5（官博：“Starting from Qwen3.5 base”；通稿同）。动作表示：改进版 X-Tokenizer2，把连续轨迹转成紧凑离散动作 token 序列；另有 flow-matching Action Expert（MoT 架构）把视觉-语言上下文转成连续动作 chunk，配合实时分块（Real-Time Chunking, RTC）让 successive chunk 保持连续。
- **家族**：WALL-OSS-1.0（全量）+ Lite（4B）+ Nano（2B）+ Gen（生成式监督、学 ego 视频）+ AR（自回归动作生成、RTC 边生成边执行）。
- **配套**：XR Harness（agentic 框架：System 2 规划、System 1 执行、System 0 经 Robot Agent Protocol 接控制器与传感器；开发者预览已开放）与 WALL-API-1.0（API 接口）。
- **训练数据组织**：15 个任务类别、32 个场景类别、110 个物体类别（官博 §Generalizable Manipulation）；UMI（手持夹爪采集）+ 机器人演示 + 策略干预 + 开源数据集；UMI 数据过 20+ 项自动校验（逆运动学可行性、运动平滑、高频抖动、位姿不连续、视觉退化、6-DoF 位姿精度）。
- **监督层次**：同一条轨迹按四级标注——task（L3）、subtask（L2）、action（L1）、segment（L0），由 RL 优化的切分模型 + 多视角 caption + 视频验证器 + 全局一致性检查自动生成。

### 1.2  headline 评估数字（全部来自官博与通稿，均为厂商自测，无独立复现）

| 评估 | 数值 | 出处 |
|---|---|---|
| 开放式真机零样本 | **55.1% 成功率 / 312 个评估员自由选取的任务**（377 次评估，47 个规范任务簇，8 个任务族，9 个能力维度；平均分 6.45/10）；多步序列 7.40/10、66.5%，pick-and-place 7.00/10、62.2%，双臂协调 6.67/10、57.7% | 官博 §Internal Real-Robot Evaluation；通稿 §Stronger Generalization |
| ManipArena（15 个桌面任务，各 10 次真机试验） | **1,101/1,500 分**，成功率 50.67%（WALL-OSS-0.5 为 722 分/19.33%，多任务微调的 π0.5 为 644.5 分/24.0%）；语义任务成功率 78%（0.5 为 20%，π0.5 为 44%）；14/15 任务领先或并列第一 | 官博同节；通稿同节 |
| X2Real 仿真（40+ 推理与操作任务） | **67.55/100**，比最强其他开源基线 Lingbot-VLA-2.0（40.94）高 65.0%；推理+操作 34.68/50，纯操作 32.87/50（GPT-6-Astra 纯操作 21.48）；移动操作 OOD 平均 44.17 vs π0.5 24.83（+77.9%），瓶到架任务 19 → 42.5 | 官博 §Internal Simulation Evaluation；通稿同节 |
| RoboDojo 公开仿真榜 | Lite **42.59 分**，截至 2026-11-05 在“无 agent 系统的策略”中排第一，领先第二名 VPP2（39.26）3.33 分 | 官博 §Public Simulation Leaderboard；通稿同节 |
| 通用能力保持 | Stage 1 模型在 VLA–VLM 基准套件 8 项上平均 **79.2% vs 底座 79.4%**（等权平均） | 官博 §Preserving General Intelligence |
| 包裹分拣生产部署 | 任务数据需求降低约两个数量级（微调到可用约 10 小时任务数据 vs 无预训练 1,000+ 小时）；跨站点保留 90% 以上原站点分拣速度与成功率（无预训练策略跨站成功率掉 90% 以上） | 官博 §WALL-OSS-1.0 in Production；通稿 §From AI Research to Real-World Logistics |
| 推理效率 | 内核优化 + 量化带来 **4 倍以上推理加速**（其部署配置下）；Nano 在生产负载评估中**单块 NVIDIA Blackwell GPU 支持 18 台机器人**（XInfer 服务框架：跨机器人调度 + 优化推理执行） | 官博 §WALL-OSS-1.0-Lite & Nano；通稿 §Expanding the Model Family |
| 部署硬件 | 端侧推理覆盖 NVIDIA Jetson Thor 与 Huixi 端侧 ASIC（硬件-软件协同设计、cycle-level GPU 仿真加速 kernel 开发、DSL kernel 优化） | 官博同节 |

### 1.3 开源状态（本节是本轮核查最容易误读的点，单独写清）

- **官方口径（2026-11-23 通稿原文）**：“the code and model weights will open-source soon”——即发布当日代码与权重**尚未开源**，只承诺“即将开源”，无日期、无许可证条款。
- **已公开的 Lite 预览权重（非最终版）**：Hugging Face 镜像站（HF//RADAR）列出 `x-square-robot/wall-oss-1.0-lite-robodojo-base`：5,323,759,872 参数（bf16）、Apache 2.0、仓库 21.3 GB，含 `normalizer_action.pth` / `normalizer_propri.pth` 与图像处理器配置；作者自述该检查点是 RoboDojo 训练的起点预览版，与最终版 Lite“主要差在训练迭代数”。**这是二手镜像站的条目，不是 HF 官方页也不是 X Square 的发布**；本报告引用它只为说明“已有预览权重在流通”，不把它当作 1.0 最终权重已开源的证据。
- **旧代权重已在 HF**：LeRobot 文档显示 `x-square-robot/wall-oss-flow` / `wall-oss-fast` / `wall-oss-0.5` 可下载（Apache 2.0），那是 WALL-OSS 0.5 代，不是 1.0。引用时不得把 0.5 代的开源状态说成 1.0 的。
- **全量模型参数量：未读到**。官博与通稿只给了 Lite 4B 与 Nano 2B 的档位，未写全量 WALL-OSS-1.0 的参数量；不得用 Lite 的 5.32B 反推全量。
- **评估所用真机型号：未读到**。官博称覆盖“单臂、双臂、移动底座”等多 embodiment，未列出具体机器人型号清单。

### 1.4 与我们项目的直接对照点

- Nano 2B 的对位物是我们 AGX 上的 Qwen3-VL-2B（:8010）：两者都是约 2B 级的端侧视觉-语言模型。但**任务不同**：Nano 是操作 VLA（输出机器人动作，wall-x 库，带动作/本体感受归一化器），:8010 是我们伴侣机器人的语言-视觉大脑；参数量相近不等于可互换。
- AGX 约束是硬的：余约 8 GiB 显存、统一内存带宽约 204.8 GB/s 是绑定约束。Nano 在 Blackwell 消费级 GPU 上 18 台共享服务的结果（厂商生产负载评估）不能平移到 AGX；XInfer 的跨机器人调度思想可以参考，但数字不能引用为我们的容量。
- 四级标注（task/subtask/action/segment）与 XR Harness 的 System 2/1/0 分层，是本节对我们最有用的两处“ pattern ”（见 §5.1）。

---

## §2 Google SayTap 专利 US 12801150 B2 + SayTap 论文

### 2.1 专利出处与核查状态（先写清证据等级）

- **专利号与标题**：US 12801150 B2，题为 'Foot Contact Pattern(s) As Interface For Language To Control Robot(s)'，权利人 Google LLC。
- **授权日期**：2026-11-17（Hot Off The Patent Press 通讯，2026-11-23；二手）。该通讯另称：Google 于 2024 年 3 月提交临时申请，即 SayTap 论文发表约九个月后。
- **申请号佐证**：Patexia 审查员页列出申请号 19/079035，标题相同，申请日 2025-03-12，抓取时状态仍为 Pending——该页抓取于授权之前（约 17 天前），状态滞后于授权事实，不构成矛盾，但也不能用它证明授权。
- **专利原文未读到**：Google Patents 规范地址（`patents.google.com/patent/US12801150B2/en`）于 2026-11-23 16:14 UTC 返回 HTTP 404（见附录 D 的 F-1）。因此**权利要求号、说明书附图、实施例细节一律未读到**，本报告不引任何权利要求号；凡下文写“专利称……”而无论文对应出处者，均只见于二手通讯，已逐处标注。
- **与论文的关系**：SayTap 论文（arXiv:2306.07580v1，CoRL 2023；作者 Yujin Tang、Wenhao Yu、Jie Tan、Heiga Zen、Aleksandra Faust、Tatsuya Harada；Google DeepMind + 东京大学）是该专利的技术来源，2023 年 9 月发表，早于 2024 年 3 月的临时申请。本节的技术细节以论文为准，专利只取“白话→触地模式→改步态”这一产品化主张。

### 2.2 核心接口：足触地模式（foot contact pattern）

- **定义**（论文 §3）：触地模式模板是一个 4×T 的 0/1 矩阵，0 = 足离地、1 = 足着地；自上而下四行依次为前左（FL）、前右（FR）、后左（RL）、后右（RR）。期望触地模式是长为 Lw 的循环滑窗，从模板中取出 t+1 到 t+Lw 的四足触地标志，形状 4×Lw。
- **时间分辨率**：控制器 50 Hz，每列 0/1 持续 0.02 秒（论文 §3.2.1 与附录 A；二手通讯亦称“每列覆盖五十分之一秒”，与论文一致）。
- **窗口**：全部实验 Lw = 5（论文 §3.2.3）。
- **为什么是这个接口**（论文 §1–§2）：LLM 擅长文本 token，不擅长关节角/力矩等底层命令；足触地时序在很大程度上决定四足最终运动形态，因此 0/1 模式是连接自然语言与底层控制器的紧凑、灵活的中间表示。相比两类基线（离散步态参数；每足正弦函数 + 截断阈值），触地模式在 30 个任务上 pattern 预测准确率高约 50%，多解出 10 个任务（论文摘要与 §4.1）。

### 2.3 两段式管线

**上段：语言 → 触地模式（LLM prompt，四块，论文 §3.1、图 3）**

1. General instruction（任务说明：把任意命令翻成触地模式）；
2. Gait definition（四足步态基础知识 + 情绪挂钩：bounding 与“happy feeling”相连，帮助 LLM 处理未明说步态的含糊命令——该情绪挂钩是 prompt 设计选择，专利通讯亦引述此句）；
3. Output format definition（先输出速度、再输出模式；速度离散化为五档 v̂x ∈ {−1, −0.5, 0, 0.5, 1} m/s）；
4. Examples block（仅给三个常见步态示例，LLM 仍能泛化到未见过的命令，包括组合与悬空相等）。

**下段：触地模式 → 底层命令（DRL 控制器，论文 §3.2）**

- 策略为前馈网络，输出每关节期望位置；输入含基座角速度、基座系重力向量、本体速度命令、当前关节位置/速度、上一步动作、期望触地模式。Unitree A1（每腿 3 关节）上输入 65 维、输出 12 维；三层隐藏层 [512, 256, 128]，ELU(α=1.0)。
- **左右镜像双 pass 技巧**（§3.2.3）：a_t = 0.5[π(s_t) + f_act(π(f_obs(s_t)))]，即对左右镜像的状态与动作取平均；论文称该技巧显著改善动作自然度并缩小 sim-to-real 差距。
- 训练时期望模式由随机模式生成器产生，测试时由 LLM 翻译（§3.2.2 与附录 A）：五种步态 G ∈ {BOUND, TROT, PACE, STAND_STILL, STAND_3LEGS}；采样周期 T ∈ [24, 28]（50 Hz 下对应 0.48–0.56 秒）；触地占比 r_contact ∈ [0.5, 0.7]；BOUND 触地时间缩为 60%、后腿行移位 0.5·T·r_contact；PACE 周期减半；STAND_3LEGS 随机一行全 0。
- 训练：PPO + IsaacGym，1,000 迭代，单块 NVIDIA V100 约 15 分钟；PD 转换位置→力矩，kp = 20、kd = 0.5；身体高度低于 0.25 m 提前结束回合；每 150 步重采样一次步态，五种步态保持大致等比（附录 C）。
- 主任务：跟踪沿朝向的线速度，同时横向速度与 yaw 角速度尽量归零；同时按时落足，使实际触地模式尽量接近期望模式。

### 2.4 奖励设计（论文附录 B，全文已读，8 项加权和）

| # | 项 | 公式要点 | 权重 |
|---|---|---|---|
| r1 | 线速度跟踪 | exp(−4((vx−v̂x)² + vy²)) | +1 |
| r2 | 角速度跟踪 | exp(−4ωz²) | −0.5 |
| r3 | **触地模式违例惩罚** | (1/4)Σ\|ci − ĉi\|，ci/ĉi ∈ {0,1} 为实际/期望触地标志 | −1 |
| r4 | 动作变化率正则 | Σ(a_t − a_{t−1})²（12 关节） | −0.005 |
| r5 | roll/pitch 角速度惩罚（不用于 BOUND） | ωx² + ωy² | −0.05 |
| r6 | 垂向线速度惩罚（不用于 BOUND） | vz² | −2 |
| r7 | 身体碰撞惩罚 | Σ 1{Fi > 0.1}，K = 8（4 大腿 + 4 小腿） | −1 |
| r8 | 偏离默认姿态惩罚 | Σ\|a_t\|（仅髋关节） | −0.03 |

### 2.5 实验结果（论文 §4）

- **接口对比**（§4.1，25 条命令 × 每条 5 次采样、GPT-4 temperature 0.5）：触地模式在基础指令上几乎全对；唯一例外是命令 21（带悬空相的原地 trot），5 次仅对 1 次——LLM 难以生成全 0 列，有一次甚至在模板后追加一行 "S: 00…0" 试图“说服”评测者存在悬空相。基线 1（离散步态参数）在 21–25 号组合/泛化命令上全败；基线 2（正弦参数）整体最差——LLM 难以理解步态与数学参数之间的关系。
- **仿真 + 真机**（§4.2）：控制器学会跟踪速度（"slow"/"fast" 对应 0.5/1.0 m/s 绝对值），同时落足时序接近期望模式；无微调迁移到 Unitree A1 真机。
- **含糊/情绪命令**（§4.3，表 2，主观评估）： "Good news, we are going to a picnic!" → 上下跳跃；"Back off, don't hurt that squirrel!" → trot 慢速后退；"Act as if the ground is very hot" → 快速 pace、足几乎不沾地；"Act as if you have a limping rear left leg" → RL 腿几乎不动、行走挣扎；"Go catch that squirrel on the tree" → 快速 bound 追赶。

### 2.6 与我们项目的直接对照点

- 我们的步态指令路径现状（任务书给定）：contract 表 1.0/2.5/4.0 Hz，B2 (a) 已定、落地排队中；`_freq_cmd` 停下待定。SayTap 的触地模式是比标量频率更丰富的中间表示：一个 4×T 的 0/1 矩阵同时编码了步态类型、相位、占空比与悬空相，而频率只编码快慢。
- 映射时注意三处不对齐：① SayTap 的速度是五档离散线速度（−1 至 1 m/s），我们是三档频率（1.0/2.5/4.0 Hz）——速度档与频率档不是同一量纲，并表需要一次换算决策；② SayTap 的策略输入含 4×5 滑窗，我们的 obs 结构任务书未给定，不得假设能直接拼入；③ SayTap 的硬件是 Unitree A1（3 关节/腿、PD kp=20/kd=0.5、50 Hz），其增益与训练配置是 A1 专属，不可照搬。
- 专利状态提醒：US 12801150 B2 已授权，但权利要求原文未读到；在产品化之前，触地模式接口的自由实施（freedom-to-operate）需要一次独立的专利核查——论文（2023）早于临时申请（2024）这一时间线是相关事实，但本报告不作法律结论。

---

## §3 Galileo C1-W（Poznan 仓库巡逻部署）

### 3.1 出处与基本信息

- **一手来源**：Galileo Robotics PR Newswire 通稿，2026-11-16 08:10 UTC（Poznan 发稿），标题《Galileo C1-W Robot Dog Deployed in Polish Warehouses, Setting Overseas Benchmark for Domestic Embodied Intelligence》；通稿更新时间同日 08:12 UTC，无后续修订。
- **部署**：波兰 Poznan 一处大型物流园（与当地波兰企业合作），C1-W 四足机器人执行仓库巡检与安防巡逻；厂商称之为其具身智能技术在海外可复制的标杆部署。
- **“11-23 更新”的真实含义**：The Robotics Media 的报道首发 2026-11-16、**页面更新于 2026-11-23**；更新的是文章页面，不是部署事件。通稿本身没有 11-23 的新事件。本报告不把“11-23 更新”写成第二次部署或新数据发布。

### 3.2 部署事实（通稿原文，逐项带数）

| 事实 | 数值 | 出处 |
|---|---|---|
| 覆盖面积 / 货架长度 | 2,000+ 平方米库区；约 2,250 米货架 | 通稿正文 |
| 巡逻频次变化 | 人工每晚 2 班 → 机器人**每 3 小时一班**；夜间巡检由每天 2 次升至每天 8 次（天津市政府英文站引述，见下） | 通稿正文；tj.gov.cn 摘要 |
| 异常响应时间 | 从“小时级”缩短到“分钟级”（通稿未给具体分钟数） | 通稿正文 |
| 稳定运行时间 | **400+ 小时**（自上线起） | 通稿正文 |
| 防护与温度 | **IP65**；−20°C 至 55°C | 通稿正文 |
| 爬坡 | 支持 30° 坡 | 通稿正文 |
| 感知 | 光学 + 热成像 + 声学三模态；实时环境感知、风险识别与智能判断；可发现堵塞的消防通道、高货架区温度异常、设备异响 | 通稿正文 |
| 数据链路 | 全场数据实时传到统一平台，形成标准化、可检索、可分析的数据资产，用于 AI 模型迭代与仓库运营优化；开放标准接口，支持多级本地化定制以符合欧洲仓库管理规范 | 通稿正文 |
| 部署背景痛点 | 人工夜间巡逻存在本地人力成本高、夜班人手短缺、巡检频次低、盲区广、作业记录不可追溯 | 通稿正文 |

### 3.3 机身规格（经销商规格页，二手；与通稿有两处出入，见 §3.4）

RobotsUSA 经销商页（自称“规格以制造商为准、报价时对照 datasheet 确认”）给出：28.5 kg；站立尺寸 765×480×540 mm；16 自由度轮腿设计（足端带轮，可滚可走）；速度 0–5 m/s 可调；连续台阶 ≥20 cm；连续负载 12 kg；续航 3–5 小时；电池 20 Ah（960 Wh）、48 V 锂电；**支持自主充电**；低功耗模式 ≥365 天、支持远程唤醒；机载 ARM 计算机 8 核 8 线程 / 8 GB RAM / 64 GB SSD；双频 WiFi6；1920×1080 广角前置相机 + 补光灯；控制模式为手动遥控、自主巡检、智能跟随；导航为自主导航 + 避障；对外接口为以太网/USB/RS485 与 48 V/24 V 电源；经销商页 Open Graph 元数据标价 11,995 美元（按配置报价）。

### 3.4 两处出入与未读项（核对用，非结论性）

- **防护等级出入**：通稿（厂商原稿）写整机 IP65；经销商规格页写“关节 IP67”。两者可能同时成立（整机 IP65、关节 IP67），但没有任何一处来源同时写明这两个口径的关系。本报告引用时分开写，不合并为单一 IP 值。
- **夜间 8 次**：通稿只写“每 3 小时一班”；“夜间由 2 次升至 8 次”出自天津市政府英文站（tj.gov.cn）的报道摘要，本报告未打开该站全文，只在上表注明来源等级。
- **充电时间：未读到**。经销商页明确写制造商未公布充电时间（与最大载荷、轮径、噪声、软件/App 兼容性、完整相机规格、包装清单同列为“报价时确认”项）。“每 3 小时一班”与“3–5 小时续航”并不能推出充电时长或充电占空比，不得自行换算。
- **巡逻路线细节：未读到**。通稿未给单班里程、单班时长、充电坞数量与位置、故障/人工接管次数。400+ 小时是“稳定运行”总时长，不是无故障时间——通稿未提故障，**不得读作零故障**。

### 3.5 与我们项目的直接对照点

- 对位的是运维模式，不是硬件：C1-W 是 28.5 kg 工业轮腿四足（16 自由度、960 Wh、3–5 小时续航），我们是桌面伴侣机器人；形态、载荷、防护、感知三模态均不对位。
- 对得上的是 session 制：我们的电池模型是“玩一会儿回充”（任务书给定），C1-W 是“每 3 小时一班 + 自主回充 + 低功耗待机 ≥365 天 + 远程唤醒”。两者都是“工作段 + 回充段 + 待机段”的三段式调度，差别只在时间尺度与触发条件。
- 可借的度量口径：巡逻频次（班/天）、稳定运行小时数、异常响应时间（小时→分钟）——这三项是运维报告的写法，不是任务成功率；我们的 session 日志将来也可以按这三项记（见 §5.3）。

---

## §4 TongDou（核查结论：记忆说法不成立）

### 4.1 先写结论

本轮核查的头号发现是**否定性结论**：任务书转述的记忆说法——TongDou 是“96 repo 清单伴侣/HRI 类开源项目”，且其中有“多轨 timeline + touch gesture”可供我们表情层与触摸输入直接借鉴——**在四个一手/近一手来源中均未得到证实，且多处被直接证伪**。实际的 TongDou 是一个单人 maker 的桌面小摆件项目，与四足、与 HRI 框架、与多轨时间线编辑器均无关系。§4.2–§4.5 是证据；§5.4 写在这一结论之上能借什么、不能借什么。

### 4.2 实际的 TongDou 是什么（四处来源交叉一致）

- **身份**：自称 "tiny open-source desktop gremlin"（Hackaday 2026-07-31；官网 gettongdou.com 同款表述）。制作者 Wang，53 岁硬件工程师，单人项目；官网有《A Letter from the Maker》自述（为排解丧亲后的孤独而做）。
- **形态与硬件**（官网 V9 原型 + Hackaday）：黄铜管机身；ESP32-S3 主控；OLED 眼睛（脸）；两个轮子（N20 减速电机）；扬声器播放预录语音；24 GHz 存在雷达（LD2412）；IMU（QMI8658A 六轴，运动与 yaw）；PDM 麦克风（IM72D128）；1,200 mAh 电池；V9 新增**电容触摸输入，直接做到金色 TongDou logo 里**；无摄像头（官网明确 "No camera. No watching."）。
- **仓库**：`github.com/solitary-dev-50/tongdou`（发布页标题与 Hackaday 外链共同确认）。官网称开源分阶段进行：先公开已验证的原理图与板级测试固件，再逐步放出完整可复现的构建包。
- **固件状态**：完整应用固件**尚未公开**（官网 FAQ 与 V9 章节两次明说 "The full TongDou firmware is not public yet and will be released in a later open-source phase"）。目前公开的是：原理图 PDF、V9 硬件测试固件、浏览器硬件测试界面、原型照片/视频/测试笔记，以及一个开发者向的演示固件 v0.1.0（见 §4.3）。

### 4.3 演示固件 v0.1.0：行为单位是“场景”，不是“时间线”

GitHub 发布页（demo-v0.1.0，全文 15 行）原文：

- "Developer-oriented scene example firmware for TongDou V9."
- 包含两个完整的**音频驱动示例**：Scene 8 — First Summon / Nightmare；Scene 6 — Confused Accountant。
- Scene 1、2、3、5、7 在源码中仍为开发实验/场景模板，原始音频从未录制，不包含在本发布中；Scene 4 刻意静音。
- "This is not the complete TongDou application firmware."

gitdiagram 架构图（2026-10-03 据该仓库文件树、README 与源码绘制，二手；组件清单见附录 A.4）列出的 20 个组件包括：`App.cpp`（应用）、`ScenarioEngine.cpp`（场景选择）、`scenario/`（场景库与场景执行）、`demo/`（演示场景播放器）、`RealtimeVoiceService.cpp`（实时语音）、`HardwareManager.cpp`（硬件管理）、`GyroReturnWiggleController.cpp`（陀螺回摆控制）、`ReminderManager.h`（提醒）、`TimeService.cpp`（时间服务）、`WebConfigServer.cpp`（网页配置）。**其中没有 timeline 模块、没有 touch/gesture 模块、没有 arousal/混音模块**。行为的组织单位是“场景”（scenario/scene），由场景引擎选择与执行；演示场景是音频驱动的脚本化示例。

### 4.4 “多轨 timeline”核查：未找到

- 在官网全文（220 行）、发布页全文、Hackaday 全文、gitdiagram 组件清单中，**没有任何一处出现 timeline / 多轨 / track / arousal / 混音**等表述；也没有任何关于“呼吸/耳朵/尾巴/脊柱各走一条轨”的设计。
- 晨报线索中“多轨 timeline 是表情编排（呼吸/耳朵/尾巴/脊柱摆动各走一条轨、按 arousal 混音）”的描述，**与 TongDou 的公开材料无一对应**：TongDou 没有耳朵、没有尾巴、没有脊柱、没有呼吸（它是两轮 + OLED 眼睛的桌面摆件）；其表情载体是 OLED 眼睛的眨眼/坏笑/困倦/恼怒表情与轮子动作。
- 结论：**“多轨 timeline”不是 TongDou 的东西**。把它当作“Week 8 最后一道未取题”的题目本身没有错，但把它归因于 TongDou 是错的；我们表情层 v0.3 从“各自写死”走向编排的下一步（见 §5.4）是我们自己的设计工作，不是“借鉴 TongDou”。

### 4.5 “touch gesture”核查：只有一个电容触摸点，没有手势语言

- **确实存在**：V9 在金色 logo 里加了一个电容触摸输入（官网 V9 章节原文）。这是 TongDou 唯一的触摸输入。
- **不存在**：手势语言（抚摸/轻拍/抱的区分）、多点触摸、手势→情绪/驱动的契约、触摸相关的固件模块（gitdiagram 组件清单中无触摸/手势组件；演示固件是音频驱动与场景驱动，未提触摸触发）。
- 结论：TongDou 证明的是“一个电容触摸点可以作为桌面伙伴的最小可行触摸输入”，不是“手势语言”。我们 touch sensor 到货后的手势→drives 契约（任务书给定：软件先行定手势→drives 契约）是我们自己的设计工作；TongDou 能借的只有“先上单比特触摸事件”这一最小起点（见 §5.4）。
- **证据等级说明**：本节“没有”的结论以四处网页来源为据；仓库 README 与源码树因克隆被叫停而未直接读到（附录 D 的 F-2），理论上存在“网页未提但源码里有”的残余可能。但 gitdiagram 架构图正是据文件树与源码绘制的，其 20 组件清单中同样没有 timeline/touch 模块，两路来源一致，残余可能已很小；若后续有人直接读到源码发现反例，应以勘误附录更正本节。

### 4.6 “96 repo 清单伴侣/HRI 类开源项目”核查：未找到

- 在全部四个来源中，没有任何“96 个仓库清单”“伴侣机器人清单”“HRI（人机交互）框架/类别”的表述。TongDou 是单个仓库的单人项目，不是清单、不是合集、不是框架。
- 可能的混淆来源（推测，须验证员复核）：任务书称“记忆称其为 96 repo 清单伴侣/HRI 类开源项目”——该记忆条目本身未给出清单出处；本轮在 GitHub 搜索、官网、Hackaday 三路均未找到与该说法对应的清单。本报告不采纳该记忆说法；若协调员能提供该记忆的原始出处，应另开核查。

---

## §5 借鉴分析（每题：能借什么 / 不能借什么 / P0/P1/P2）

口径（沿用 RAIBO2 报告 §5）：P0 直接可用（不改接口、不改架构，照做即可）；P1 需适配（思想或 pattern 可用，须按我们的硬件/接口/现状改写）；P2 明确不适用（带理由，不做）。“我们哪个模块/哪行口径”一栏只写任务书给定的锚点。

### 5.1 WALL-OSS-1.0

**能借什么**

| # | 借鉴点 | 映射到我们哪里 | 优先级 |
|---|---|---|---|
| W1 | 四级轨迹标注（task L3 / subtask L2 / action L1 / segment L0）作为行为数据的标注约定 | 表情层 v0.3 的行为片段与步态指令：同一段行为同时记四级描述（目标/子目标/动作/片段），为后续编排与检索留接口；纯约定，零代码 | P0 |
| W2 | XR Harness 的 System 2/1/0 分层（规划 / 执行 / 硬件接口经 RAP）作为伴侣软件栈的分层 pattern | :8010 Qwen3-VL-2B（规划）/ 运动策略（执行）/ CAN+固件（硬件接口）的分层对照；只借分层思想，不引其代码（未开源） | P1 |
| W3 | 小模型 + 内核优化 + 量化的推理提速路线（厂商称 4 倍以上，其部署配置下） | :8010 Qwen3-VL-2B 在 AGX 上的推理：量化的收益须我们自己实测；厂商的 4 倍数字不得引用为我们的预期 | P1 |
| W4 | Agent 驱动的接管循环（策略执行、人在失败处接管、纠错数据回流训练）作为数据收集方法 | 未来训练数据收集的方法论；当前无训练在跑，先记为方法储备 | P1 |
| W5 | 原生联合预训练的思想（理解与动作同一训练过程塑造表示，而非先 VLM 后挂动作） | :8010 作为大脑、RL 策略作为身体的两段式现状的对照原则：下次动大脑接口时，优先统一指令→行为映射，少做事后外挂 | P1 |

**不能借什么**

| # | 不能借 | 理由 |
|---|---|---|
| W-N1 | 直接部署 Nano 2B（或 Lite 4B）到 AGX | Nano 是操作 VLA（wall-x 库、动作/本体感受归一化器、输出机械臂动作），不是四足运动控制器；2B 参数量相近不等于任务可换；1.0 最终权重尚未开源（只有 Lite RoboDojo 预览版）；AGX 余约 8 GiB 显存、204.8 GB/s 带宽是绑定约束，Blackwell 上 18 台共享的数字与 AGX 无关 |
| W-N2 | 55.1% / 67.55 / 42.59 等评估数字作为我们的目标或基线 | 全部是厂商自测的操作任务（桌面抓取、分拣、移动操作），无四足 locomotion、无伴侣交互；无独立复现；与我们的任务无交集 |
| W-N3 | MoT flow-matching Action Expert + RTC 的动作生成实现 | 我们的运动控制是 RL 策略（任务书未给定其 obs/action 定义，本报告不假设），不是自回归 VLA 动作解码；chunk 连续性问题在我们的栈里不存在对应物 |
| W-N4 | Gen 的 ego 视频生成式监督 | 需要 dexterous 操作数据与 WAM 训练管线；与伴侣四足无关 |

### 5.2 SayTap / US 12801150 B2

**能借什么**

| # | 借鉴点 | 映射到我们哪里 | 优先级 |
|---|---|---|---|
| S1 | 触地模式（4×T 的 0/1 矩阵 + 4×Lw 滑窗）作为步态指令的中间表示 | 步态指令路径（contract 表 1.0/2.5/4.0 Hz，B2 (a) 已定、落地排队中）：在频率档之外增加一层“模式”表示；表示本身是数据格式定义，可直接写进设计文档 | P0（表示定义）；P1（策略侧接线，见 S2） |
| S2 | r3 触地模式违例惩罚（(1/4)Σ\|ci − ĉi\|，权重 −1）作为奖励项设计 | 下次动奖励时可加的接触时序项；权重 −1 是 A1 上的值，须按我们的策略重调，不得照搬 | P1 |
| S3 | LLM prompt 四块结构（任务说明 / 步态知识 + 情绪挂钩 / 输出格式与五档速度 / 示例）作为 :8010 的 prompt 模板 | :8010（Qwen3-VL-2B）把白话翻成触地模式的 prompt 设计；五档速度 {−1, −0.5, 0, 0.5, 1} m/s 与我们的三档频率（1.0/2.5/4.0 Hz）并表须一次换算决策 | P0（模板结构）；P1（速度档并表） |
| S4 | 随机模式生成器（T ∈ [24, 28]、r_contact ∈ [0.5, 0.7]、分步态缩放规则）作为训练时模式分布的设计 | 策略训练时期望模式的采样分布；A1 的可行性规则须按我们的腿（任务书未给定腿构型）重做，不得照搬数值 | P1 |
| S5 | 左右镜像双 pass 技巧（a_t = 0.5[π(s_t) + f_act(π(f_obs(s_t)))]） | 策略训练的对称性约束方法；须改训练代码，当前无训练在跑，先记为方法储备 | P1 |
| S6 | 含糊/情绪命令的用例集（"地面很烫"→快 pace、"去野餐"→跳跃、"跛行"→单腿少动）作为伴侣交互的用例库 | 表情层 v0.3 + 步态指令的联合用例：情绪既改表情也改步态；用例可直接抄进用例清单，映射须我们自己定 | P0（用例） |

**不能借什么**

| # | 不能借 | 理由 |
|---|---|---|
| S-N1 | SayTap 的策略网络与训练配置（65-D 输入 / 12-D 输出、MLP [512,256,128]、PPO+IsaacGym、1,000 迭代/V100 15 分钟） | Unitree A1 专属（3 关节/腿）；我们的 obs/action 定义任务书未给定，无法对位；15 分钟是 A1 + IsaacGym 的数 |
| S-N2 | PD 增益 kp=20、kd=0.5 | A1 的执行器与减速比专属；我们的舵机与传动不同 |
| S-N3 | 专利权利要求的任何内容 | 原文未读到（附录 D 的 F-1）；产品化前须独立做 freedom-to-operate 核查。本报告不作法律结论，只记录时间线事实：论文（2023-09）早于临时申请（2024-03） |
| S-N4 | 把“bounding = happy”当作情绪真理 | 那是 prompt 里写给 LLM 的一句话，是设计选择不是动物行为学结论；我们的情绪→步态映射须自己定 |

### 5.3 Galileo C1-W

**能借什么**

| # | 借鉴点 | 映射到我们哪里 | 优先级 |
|---|---|---|---|
| G1 | 三段式调度（工作段 + 自主回充 + 低功耗待机 + 远程唤醒）作为 session 制的操作模板 | Session 制电池模型（玩一会儿回充）：把“回充”与“待机/唤醒”写成一等行为（回充坞行为 + 低功耗待机态 + 远程唤醒），而不只是电池百分比阈值 | P1 |
| G2 | 运维度量口径（班/天、稳定运行小时数、异常响应时间）作为 session 日志的记法 | Session 日志将来按“每日班数、累计稳定小时、发现→响应延迟”三项记，而不只记任务成功率 | P1 |
| G3 | 覆盖度量口径（面积 m² + 路线长度）作为报告写法 | 室内活动报告将来按覆盖面积/路线长度记，而不只记时长 | P1 |
| G4 | “把重复、 awkward、高频的人工巡查换成机器”的选址逻辑 | 伴侣机器人的日常巡视/ presence 路线选址思路：优先替代高频重复的人工查看 | P1 |

**不能借什么**

| # | 不能借 | 理由 |
|---|---|---|
| G-N1 | 轮腿 16 自由度形态、960 Wh / 3–5 小时续航、28.5 kg 机身 | 工业巡检硬件，与桌面伴侣的形态、重量、能量预算全不对位 |
| G-N2 | 光学+热成像+声学三模态感知 | 我们没有热成像与声学阵列硬件；不得以此为由加硬件 |
| G-N3 | IP65/IP67 与 −20~55°C 环境 hardening | 室内伴侣机器人不需要；且两处来源的 IP 口径本身有出入（见 §3.4） |
| G-N4 | “400+ 小时稳定运行”作为可靠性目标 | 那是单点部署的厂商自报总时长，未提故障与接管次数；不得读作零故障，也不得直接立为我们的目标值 |

### 5.4 TongDou

**能借什么**

| # | 借鉴点 | 映射到我们哪里 | 优先级 |
|---|---|---|---|
| T1 | “场景”（scenario/scene）作为行为组织单位（场景引擎选择与执行、音频驱动的演示场景） | 表情层 v0.3 从“各自写死”走向编排的下一步：一个命名场景 = 一段多轨协调的行为片段（呼吸/耳朵/尾巴/脊柱各轨在场景内对齐）；场景是组织单位，时间线编辑器是我们自己的设计 | P1 |
| T2 | 单比特电容触摸作为最小可行触摸输入（V9 金色 logo 触摸点） | Touch sensor 到货后的输入：先上单比特触摸事件 → drives 的映射，再建手势词汇；不要等手势语言完备才接线 | P1 |
| T3 | 分阶段开源的顺序（先板级测试固件与原理图，后完整固件）作为发布方法 | 我们对外发布硬件/固件时的方法参考：先测试固件，后完整包 | P1 |
| T4 | 无摄像头 presence 感知（24 GHz 雷达只知有人在附近）的隐私姿态 | 伴侣机器人隐私设计的方法参考：存在感应用雷达/触摸而不用摄像头，是可抄的姿态，不是模块 | P1 |

**不能借什么**

| # | 不能借 | 理由 |
|---|---|---|
| T-N1 | “多轨 timeline”本身 | TongDou 里没有这个东西（§4.4）；我们的时间线编排是原创设计，不得写成“借鉴 TongDou” |
| T-N2 | “touch gesture”手势语言（抚摸/轻拍/抱的区分） | TongDou 里没有手势词汇（§4.5）；手势→drives 契约是我们软件先行的设计，不得写成“TongDou 已有” |
| T-N3 | TongDou 的固件/场景代码 | 完整应用固件尚未公开；演示固件是 ESP32-S3 + OLED 眼睛 + 两轮的场景播放器，与我们的栈无交集 |
| T-N4 | “96 repo 清单伴侣/HRI 类开源项目”这一归类 | 未找到出处（§4.6）；不得再引用 |

### 5.5 四题优先级总表

| 题 | P0（直接可用） | P1（需适配） | P2（明确不适用） |
|---|---|---|---|
| WALL-OSS-1.0 | 四级标注约定（W1） | 分层 pattern、量化路线、接管循环、原生联合思想（W2–W5） | Nano/Lite 直部署、评估数字作目标、MoT+RTC 实现、Gen（W-N1–W-N4） |
| SayTap | 触地模式表示定义、prompt 模板结构、情绪用例（S1/S3/S6 部分） | 策略侧接线、r3 奖励项、速度档并表、模式生成器、双 pass（S1/S2/S3/S4/S5 部分） | A1 策略与训练配置、A1 PD 增益、专利权利要求、情绪等式（S-N1–S-N4） |
| Galileo C1-W | —（无直接可用项） | 三段式调度、运维度量、覆盖度量、选址逻辑（G1–G4） | 轮腿形态、三模态感知、环境 hardening、400 小时作目标（G-N1–G-N4） |
| TongDou | —（无直接可用项；记忆说法证伪后无 P0） | 场景为组织单位、单比特触摸起点、分阶段发布、无摄像头姿态（T1–T4） | 多轨 timeline、手势语言、固件/场景代码、96 清单归类（T-N1–T-N4） |

**本轮最优先的三件事**（给协调员的建议，不替用户决策）：① 把 SayTap 的触地模式表示定义写进设计文档（S1，P0），与 B2 (a) 的频率表并行存在、待一次换算决策；② 把四级标注约定（W1，P0）用在表情层 v0.3 的行为片段上；③ 把 session 日志按 Galileo 口径（G2，P1）记班数/稳定小时/响应延迟。③ 的前置是 session 功能本身，任务书未给定其现状，本报告不假设它已存在。

---

## §6 一页纸结论

1. **WALL-OSS-1.0 是操作 VLA 的里程碑，不是四足的里程碑**：55.1% 零样本 / 312 任务、ManipArena 1,101/1,500、X2Real 67.55，都是桌面操作与移动操作的厂商自测；对我们最有用的不是模型本身（Nano 2B 不能直部署到 AGX：任务不同、权重未开源、8 GiB/204.8 GB/s 是硬约束），而是四级标注约定（P0）与 System 2/1/0 分层（P1）。
2. **SayTap 是本轮含金量最高的一题**：4×T 的 0/1 触地模式 + 4×5 滑窗 + r3 违例惩罚 + prompt 四块结构，是一套完整、可抄的“白话改步态”接口设计；它与我们 contract 表 1.0/2.5/4.0 Hz 的频率档互补而非替代，并表需要一次速度档↔频率档的换算决策。专利原文未读到，产品化前须独立核查 freedom-to-operate。
3. **Galileo C1-W 借的是运维，不是硬件**：每 3 小时一班、400+ 小时、自主回充 + ≥365 天低功耗待机，是 session 制电池模型在工业尺度上的同款三段式调度；三模态感知与 IP 等级与我们无关。注意“11-23 更新”是报道页面更新，不是新部署；充电时间未公布，不得换算。
4. **TongDou 一题的结论是否定性的**：记忆说法不成立——实际 repo 是单人 ESP32-S3 桌面摆件，没有多轨 timeline、没有手势语言、完整固件尚未公开；能借的只有“场景为组织单位”与“单比特触摸先行”两个 P1 级思想。表情编排的时间线与手势→drives 契约是我们自己的原创设计，不得再写成借鉴 TongDou。
5. **三件 P0 可以直接做**：触地模式表示定义进设计文档、四级标注约定用到表情层行为片段、SayTap 情绪用例进用例清单。其余皆为 P1（需适配）或 P2（不做），见 §5.5 总表。

---

## 附录 A 出处清单

### A.1 WALL-OSS-1.0

- 官博：X Square Robot Team, "WALL-OSS-1.0: Understand More, Act Better", X Square Robot Blog, 2026-11-23, https://x2robot.com/blogs/wall-oss-1-en/（全文 745 行，已读完；关键小节：Native VLA foundation、MoT + flow-matching Action Expert、Lite & Nano、Gen、AR、Internal Real-Robot Evaluation、Internal Simulation Evaluation、Public Simulation Leaderboard、Parcel Sorting、XR Harness）
- 通稿：GlobeNewswire, "X Square Robot Introduces WALL-OSS-1.0: Understand More, Act Better — A Native Foundation Model for the Physical World", 2026-11-23T07:44:46Z, https://www.globenewswire.com/news-release/2026/11/23/3408046/0/en/x-square-robot-introduces-wall-oss-1-0-understand-more-act-better-a-native-foundation-model-for-the-physical-world.html（全文 149 行，已读完）
- Lite 预览权重（二级镜像，非官方发布）：HF//RADAR, "wall-oss-1.0-lite-robodojo-base", https://radar-huggingface.686f6c61.dev/en/models/x-square-robot-wall-oss-1-0-lite-robodojo-base（5,323,759,872 参数 bf16、Apache 2.0、21.3 GB；作者自述为预览版）
- 旧代权重（0.5 代，非 1.0）：Hugging Face LeRobot 文档，https://huggingface.co/docs/lerobot/walloss（`x-square-robot/wall-oss-flow` / `wall-oss-fast` / `wall-oss-0.5`）

### A.2 SayTap / US 12801150 B2

- 论文：Yujin Tang, Wenhao Yu, Jie Tan, Heiga Zen, Aleksandra Faust, Tatsuya Harada, "SayTap: Language to Quadrupedal Locomotion", arXiv:2306.07580v1, CoRL 2023, https://arxiv.org/html/2306.07580v1（全文 813 行，已读完；关键：§3 方法、§3.1 prompt、§3.2 控制器、§4 实验、附录 A 随机模式生成器、附录 B 奖励设计、附录 C 训练配置）
- 项目站：https://saytap.github.io/（摘要与方法概述；搜索摘要中已读，未全文爬取）
- 专利授权事实与标题（二级）：Hot Off The Patent Press, "Sit! Fetch!... Prompt?", 2026-11-23, https://newsletter.hotoffthepatentpress.com/p/sit-fetch-prompt（全文 241 行，已读完；专利特有细节如 1/50 秒列、五档速度、bounding "happy feeling" 仅见于此）
- 申请号佐证（二级、状态滞后）：Patexia 审查员页，https://services.patexia.com/examiner/peter-y-ning（申请 19/079035，申请日 2025-03-12，抓取时状态 Pending）
- 专利原文：Google Patents 规范地址返回 HTTP 404，未读到（附录 D 的 F-1）

### A.3 Galileo C1-W

- 通稿（一手）：Galileo Robotics / PR Newswire, "Galileo C1-W Robot Dog Deployed in Polish Warehouses, Setting Overseas Benchmark for Domestic Embodied Intelligence", 2026-11-16 08:10 UTC, https://www.prnewswire.com/news-releases/galileo-c1-w-robot-dog-deployed-in-polish-warehouses-setting-overseas-benchmark-for-domestic-embodied-intelligence-302940191.html（全文 56 行，已读完）
- 报道（二级）：The Robotics Media, "Galileo's C1-W Robot Dog Patrols Polish Warehouse Every 3 Hours", 2026-11-16 首发、2026-11-23 更新，https://theroboticsmedia.com/article/galileo-c1-w-robot-dog-polish-warehouse-deployment（全文 86 行，已读完）
- 经销商规格页（二级）：RobotsUSA, "Galileo C1-W Wheeled Quadruped Robot Dog", https://www.robotsusa.com/GALILEO-C1-W.htm（全文 216 行，已读完）
- 夜间 8 次（仅搜索摘要，未作一手引用）：天津市政府英文站，https://en.tj.gov.cn/Updates/News/202611/t20261120_7660404.html

### A.4 TongDou

- 官网（一手）：https://gettongdou.com（全文 220 行，已读完；关键：V9 章节、FAQ、"Available now / Planned phased releases"）
- 发布页（一手）：GitHub, "TongDou Demo Firmware v0.1.0", https://github.com/solitary-dev-50/tongdou/releases/tag/demo-v0.1.0（全文 15 行，已读完）
- 文章（二级）：Hackaday, Zoe Skyforest, "Tiny Desktop Robot Has Radar", 2026-07-31, https://hackaday.com/2026/07/31/tiny-desktop-robot-has-radar/（全文 22 行，已读完）
- 架构图（二级，据文件树/README/源码绘制，2026-10-03）：GitDiagram, http://gitdiagram.com/solitary-dev-50/tongdou（20 组件清单；文件路径如 `demo_firmware/firmware/src/app/App.cpp`、`demo_firmware/firmware/src/scenario/ScenarioEngine.cpp`、`demo_firmware/firmware/src/motion/GyroReturnWiggleController.cpp`）
- 仓库本体：未克隆（附录 D 的 F-2）；README 与源码树未直接读到

### A.5 我们项目事实的出处（任务书给定）

- 任务书：`~/workspace/goals/bordercollierobot/hidden_files/frontier_deepdive_20261123/TASK.md`（四题锚点；见本报告 §0.3，全文转载口径）

## 附录 B 数值核对表（本报告引用的全部关键数字，一处出处）

| 数字 | 值 | 出处 |
|---|---|---|
| WALL 零样本成功率 / 任务数 | 55.1% / 312（377 次评估；47 簇；8 族；9 维度；均分 6.45/10） | 官博 §Internal Real-Robot Evaluation |
| WALL ManipArena | 1,101/1,500；成功率 50.67%（0.5：722/19.33%；π0.5：644.5/24.0%）；语义 78%（20%/44%）；14/15 领先或并列 | 官博同节 |
| WALL X2Real | 67.55（Lingbot-VLA-2.0 40.94，+65.0%）；推理+操作 34.68/50；纯操作 32.87/50（GPT-6-Astra 21.48）；移动操作 OOD 44.17 vs π0.5 24.83（+77.9%）；瓶到架 19→42.5 | 官博 §Internal Simulation Evaluation |
| WALL RoboDojo Lite | 42.59，第一（截至 2026-11-05）；VPP2 39.26，领先 3.33 | 官博 §Public Simulation Leaderboard |
| WALL 通用能力保持 | Stage 1 79.2% vs 底座 79.4%（8 项基准等权平均） | 官博 §Preserving General Intelligence |
| WALL 包裹分拣 | 约 10 h vs 1,000+ h 任务数据；跨站保留 >90%（无预训练掉 >90%） | 官博 §Parcel Sorting |
| WALL 推理效率 | 4 倍以上加速（其部署配置）；Nano 单块 Blackwell GPU 支持 18 台机器人（生产负载评估，XInfer） | 官博 §Lite & Nano；通稿 |
| WALL 数据覆盖 | 15 任务 / 32 场景 / 110 物体类别；UMI 20+ 校验 | 官博 §Generalizable Manipulation |
| WALL Lite 预览权重 | 5,323,759,872 参数 bf16；Apache 2.0；21.3 GB | HF//RADAR 镜像（二级） |
| SayTap 模板 / 滑窗 / 频率 | 4×T；4×Lw，Lw=5；50 Hz（每列 0.02 s） | 论文 §3、§3.2.3 |
| SayTap 策略 | 输入 65 维 / 输出 12 维；MLP [512,256,128] ELU；A1 每腿 3 关节 | 论文 §3.2.3 |
| SayTap 模式生成器 | T ∈ [24,28]（0.48–0.56 s）；r_contact ∈ [0.5,0.7]；5 步态 | 论文 §3.2.2、附录 A |
| SayTap 奖励权重 | r1 +1；r2 −0.5；r3 −1；r4 −0.005；r5 −0.05；r6 −2；r7 −1；r8 −0.03 | 论文附录 B |
| SayTap 训练 | PPO + IsaacGym；1,000 迭代；单 V100 约 15 分钟；PD kp=20、kd=0.5；提前结束高度 <0.25 m | 论文附录 C |
| SayTap 实验 | 30 任务；pattern 准确率高约 50%；多解 10 个任务；GPT-4 temperature 0.5，每命令 5 次；速度 slow/fast = 0.5/1.0 m/s | 论文摘要、§4.1、§4.2 |
| SayTap prompt 速度档 | v̂x ∈ {−1, −0.5, 0, 0.5, 1} m/s（五档） | 论文 §3.1 |
| 专利 | US 12801150 B2；授权 2026-11-17；申请 19/079035，申请日 2025-03-12；临时申请 2024-03 | 通讯（二级）；Patexia（二级） |
| Galileo 部署 | 2,000+ m²；约 2,250 m 货架；2 班/天 → 每 3 h 一班；响应小时→分钟；400+ h；IP65；−20~55°C；30° 坡 | PR Newswire 通稿 |
| Galileo 规格（经销商） | 28.5 kg；765×480×540 mm；16 自由度；0–5 m/s；台阶 ≥20 cm；负载 12 kg；3–5 h；20 Ah（960 Wh）48 V；自主充电；低功耗 ≥365 天 + 远程唤醒；ARM 8C/8T 8 GB/64 GB；11,995 美元（OG 元数据） | RobotsUSA 规格页 |
| TongDou 硬件 | ESP32-S3；OLED 眼睛；N20 轮子 ×2；24 GHz LD2412 雷达；QMI8658A IMU；IM72D128 麦克；1,200 mAh；V9 电容触摸 logo；无摄像头 | gettongdou.com；Hackaday |
| TongDou 演示固件 | v0.1.0；Scene 8 + Scene 6 完整（音频驱动）；Scene 1/2/3/5/7 模板；Scene 4 静音；非完整应用固件 | GitHub 发布页 |
| 我们 AGX | Qwen3-VL-2B :8010；Qwen3-14B-FP8 :8011；余约 8 GiB；带宽约 204.8 GB/s | 任务书给定 |
| 我们步态 | contract 表 1.0/2.5/4.0 Hz；B2 (a) 已定；`_freq_cmd` 停下待定 | 任务书给定 |

## 附录 C 未读项清单（本报告明确未读到的东西）

- C-1 WALL-OSS-1.0 全量模型参数量：官博与通稿均未写（只给 Lite 4B / Nano 2B 档位）。
- C-2 WALL 312 任务评估所用的真机型号清单：官博只写多 embodiment，未列型号。
- C-3 WALL-OSS-1.0 最终版代码与权重的开源日期与许可证：官方口径只有 "will open-source soon"；Lite 预览版不等于最终版。
- C-4 US 12801150 B2 专利原文：Google Patents 404（附录 D 的 F-1）；权利要求号、说明书、附图、实施例全部未读到；Patexia 页状态滞后（Pending）。
- C-5 Galileo 充电时间、单班里程/时长、充电坞数量与位置、故障/接管次数：通稿与经销商页均未给；经销商页明言充电时间未公布。
- C-6 Galileo IP65（整机）与 IP67（关节）两个口径的关系：无任何来源同时说明，不得合并。
- C-7 TongDou 仓库 README 与源码树：克隆被叫停，未直接读到（附录 D 的 F-2）；"timeline/touch 模块不存在"的结论以官网、发布页、Hackaday、gitdiagram 四路为准，残余可能见 §4.5。
- C-8 “96 repo 清单伴侣/HRI 类开源项目”这一说法的原始出处：四路来源均未找到；记忆条目本身未给出处。

## 附录 D 失败运行清单（日志原文已归档，非“见 exec 记录”）

- D-1（F-1）Google Patents 抓取失败：`browser.open https://patents.google.com/patent/US12801150B2/en` → HTTP 404（2026-11-23 ~16:14 UTC）。未重复尝试同一地址。完整记录见工作目录 `work/failed_runs.log` 的 F-1 节。
- D-2（F-2）TongDou 仓库克隆中途叫停：`git clone --depth 1 https://github.com/solitary-dev-50/tongdou.git tongdou` 运行 266+ 秒零输出、部分下载约 37 MB；其进程清单的 exec 结果触发执行风险提示，克隆已被 SIGTERM 终止（exitSignal 15），部分目录已删除。完整记录见工作目录 `work/failed_runs.log` 的 F-2 节。
- 本轮无其他失败运行。工作目录：`~/workspace/goals/bordercollierobot/hidden_files/frontier_deepdive_20261123/`（`research/`、`work/research_notes.md`、`work/failed_runs.log`）。
LINES: 428
SELF: 140900cfb7881af2bab3a86444c419d0
FULL: 6c5b8731c0ea5fe135cece0d9452a726
