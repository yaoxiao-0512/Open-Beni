# LocoMuJoCo 论文 + 仓库精读与应用报告

- 日期：2026-11-23
- 阅读对象（两份 + 一份部署库）：
  1. 论文预印本：LocoMuJoCo: A Comprehensive Imitation Learning Benchmark for Whole-Body Control（arXiv:2611.26293，2026-11-19，Firas Al-Hafez 等 7 人，26 页）
  2. 代码仓库：`robfiras/loco-mujoco`（MIT 开源；本轮克隆 HEAD `df9869e8b1d94e7a4470f6fc42f67d3f7defe6a2`，commit "docs(readme): add the PyPI install command and badge"）
  3. 部署库：`robfiras/loco-mujoco-real`（README 读完；HEAD `dc9c4d0`，2026-11-18 建库，6 个 commit；未克隆，只读 GitHub 页面文本）
- 用途：BorderCollieRobot 项目（0.42× 边牧四足伴侣机器人，mjlab + MuJoCo Warp + RSL-RL PPO + torch）的对照与应用；同时存放在 open-beni 仓库作为研究档案
- 写作纪律：每个关键数字与结论都标注出处（论文章节、仓库文件路径与行号、我们代码的文件与行号）。凡是没读到的东西，本报告明确写"未读到 / 无法确认"，不编数字。我们的项目事实只采用任务书给定的已确认事实 + 本轮亲读的代码。

## §0 阅读方法与范围说明

### 0.1 三份来源各是怎么读的

| 来源 | 阅读方式 | 完整度 |
|---|---|---|
| 论文 arXiv:2611.26293 | `browser.open` 读 arxiv.org HTML 全文，共 2230 行文本，逐节读完 | 正文 §1–§3.5、附录 A–H 全部读完；参考文献（L1846 起）只扫过标题行；**图（Fig.1–11）是图片，HTML 文本版只读到图注，图内数值未读到**；凡涉及图内曲线读数的结论，本报告不引用具体数值 |
| loco-mujoco 仓库 | `git clone --depth 1`（HEAD `df9869e`），读 README、核心插件、奖励/观测/DR/地形/控制/终止/初始、四足 env、工厂、算法、评估度量、轨迹格式、`smpl/robot_confs/defaults.yaml` | 核心文件逐行或按类读完（见 §2 各节行号）；`smpl/` 重定向代码只读了结构，`defaults.yaml` 的关键值已 grep 核实（见 §2.14）；`datasets/` 下的数据文件未下载；**仓库代码未运行、未安装、未 import**——所有结论来自静态阅读 |
| loco-mujoco-real | `browser.open` 读 GitHub 仓库页全文（461 行，含完整 README） | README 全文读完；`training/`、`deployment/`、`lmj_real/` 下的代码文件未逐行读到，凡涉及其内部实现的说法只以 README 文字为准，并标注 |

### 0.2 本报告的结构

- §1：论文精读（设计、实验、结论、局限，带章节引用）
- §2：仓库精读（结构、关键实现、与论文的对应，带文件行号）
- §3：三方对照（LocoMuJoCo vs 我们的现状，逐项：对方证据 + 我们对应的文件/行 + 适用/不适用 + 理由 + 采纳它会改变我们什么）
- §4：对我们项目的应用建议（分 P0/P1/P2，哪些能直接借鉴、哪些要改造、哪些不适用，并说明理由）
- §5：一页纸结论
- 附录：出处清单、数值核对表、未确认清单、勘误（空）

### 0.3 两处先行订正（读代码时发现的，与任务书措辞不同）

1. **任务书说"contract.py（128-D obs / 24-D action 合同）"， live 口径是：训练 24-D、部署 16-D。** `contract.py:2` 头注释与 `rl/AGENTS.md` Golden rule 1–2 一致：obs 128-D（76-D 冻结前缀 + 52-D 追加块，pert_active 插在索引 75）；训练 action 24-D（16-D 关节位置：14-D 神圣前缀 + SPINE_yaw2 + NECK_pan，再加 8-D 每腿阻抗冻结占位）；部署 ONNX 是 `[1,128]→[1,16]`（v0.6，2026-11-20 由 `[1,128]→[1,14]` 升版）。本报告凡说 action 维数都分"训练/部署"写，不笼统写 24-D。
2. **PD 增益不是 6.0/0.2。** `collie_params.py:62–63` 现行值是 `PD_KP_NOMINAL = 19.0`、`PD_KD_NOMINAL = 0.2`（2026-11-21 kp→19 落地轮：站立二分阈值 19.0，8/12/16/17/18 塌、19/20/24 站；kd 0.2 不动）。6.0 是 2026-10-07 的旧值，8.0 是 2026-10-21 的值。本报告按 19.0/0.2 引用。

---

## §1 论文精读：LocoMuJoCo

### 1.1 出处与基本信息

- **标题**：LocoMuJoCo: A Comprehensive Imitation Learning Benchmark for Whole-Body Control
- **作者**：Firas Al-Hafez、Davide Tateo（TU Darmstadt）、Oleg Kaidanov、Aditya Bhatt、Artur Zelik（DFKI）、Zhengyi Luo（CMU）、Jan Peters（TU Darmstadt / DFKI / Hessian.AI）——论文第 1 页作者栏。
- **版本**：arXiv:2611.26293，2026-11-19 发布；本报告读的是 arXiv HTML 版。论文自述是 v1.0 以来第三代：第一版（Al-Hafez et al. 2023b）只有 CPU MuJoCo；v1.0 用 JAX 重建，加 GPU 并行 MJX 后端、重定向管线、新 embodiment、JAX 算法；当前版加可选 MjWarp 后端、更多算法与 loco-mujoco-real 部署管线（§1）。
- **许可与获取**：MIT 开源（摘要末句）；仓库 `github.com/robfiras/loco-mujoco`；模型另在 `loco-mujoco-models` 仓；数据集在 HuggingFace `robfiras/loco-mujoco-datasets`（§4）。
- **资助**：EU Horizon Europe ARISE（No. 101135959）、DFG SE1042/41-1、BMBF KIARA（13N16274）（Acknowledgments）。
- **一句话概括**：LocoMuJoCo 不是一个机器人、一个任务，而是一个"换机器人、换任务、换算法都不用重写环境"的 IL/RL 基准：同一套插件接口同时跑 16 个 embodiment（9 人形 + 3 肌骨 + 4 四足），同一条 JAX 训练循环同时跑 PPO/GAIL/DeepMimic/SuperMimic 等 10 种算法，评测用轨迹相似度（不是只看 return）。

### 1.2 核心结论与 headline 数字

| 指标 | 数值 | 出处 |
|---|---|---|
| embodiment 总数 | 9 人形 + 3（肌骨）骨骼 + 4 四足 = 16 | 摘要；§3.3 |
| 四足 | Unitree A1 / Go2、BD Spot、ANYmal C | §3.3 |
| 人形 | Unitree H1/H1v2/G1、Atlas、Talos、Apollo、Booster T1、ToddlerBot、Fourier GR1T2 | §3.3 |
| 后端 | MuJoCo CPU + MJX + MjWarp，同一 API | 摘要；§3.1 |
| MjWarp 吞吐 | 4.9×10⁵ env steps/s，单张消费级 GPU | §3.1；附录 G 表 3 |
| 算法数 | PPO、GAIL、AMP、DeepMimic、SuperMimic、BPTT-PPO、历史感知 PPO、SAC、TD3、S2PG、DAgger 蒸馏 | §3.4；附录 E |
| 评测实验量 | 715 次运行（附录 H 实验 1：11 embodiment × 13 clips × 5 seeds） | 附录 H.1 |
| 测试覆盖 | >500 个测试，93% 行覆盖 | §4；附录 F |
| 部署 | AgiBot X2 Ultra 43 s 全身运动，50 Hz 真机运行；SuperMimic 单策略 40 个 LAFAN1 clips，99.7% 帧完成 | §3.5；附录 H.4 表 4 |
| 重定向精度 | G1 形状拟合 15 点平均误差 112.1 → 21.8 mm | 附录 D.1，图 4 |

### 1.3 双后端：一次实现，NumPy 与 JAX 共用（§3.1、附录 C）

- 每个环境可走串行 NumPy/MuJoCo 后端，或向量化 JAX/MJX 后端；再加 `use_mjwarp=True` 走 MjWarp。观测、奖励、控制、终止等组件"针对一个数值后端参数实现一次"——因为 `jax.numpy` 与 NumPy 高度同构，"大多数代码逐字共用，只有后端特有的操作才分支"（§3.1）。
- 附录 C 的用法示例是全文最值得抄的一段：`ImitationFactory.make("MjxUnitreeH1", default_dataset_conf=dict(task="squat"))` 建一次环境；CPU 循环用 gymnasium 风格 `reset/step`；MJX 循环用 `jax.jit(jax.vmap(env.mjx_reset))` / `mjx_step`，1024 个并行 env；"两个循环共用同一观测、奖励、终止与随机化插件，只有 step/reset 调用点不同"（附录 C）。
- 对我们的意义见 §3 C-1：我们也有双路（CPU torch env + GPU mjlab），但两路是两套实现（`env_cpu.py` / `train_cpu.py` vs `tasks/mdp.py` 的 mjlab env），不是"一次实现、双后端"。LocoMuJoCo 证明了一次实现的可行性，但代价是全栈 JAX 化——见 §4 P2。

### 1.4 模块化插件架构（§3.2、附录 B）

附录 B 是全文方法论最密的一节，七个插槽（slots）：

1. **观测（Observations）**：建 env 时组合一个具名观测 spec 列表，每个 spec 返回固定形状的向量切片（关节位置/速度、基座朝向、mimic-site 位置、接触力……），env 串联成策略输入。"这就是同一条训练脚本能在所有 embodiment 上跑的原因，尽管它们的观测维数不同"（附录 B）。
2. **奖励（Rewards）**：从 `NoReward`（纯动力学）、速度类 shaping（`TargetXVelocityReward`、`TargetVelocityGoalReward`、`LocomotionReward`、`TargetVelocityTrajReward`）到模仿类 `MimicReward`；"从速度跟踪切换到动作模仿只改一个参数"（附录 B）。
3. **域随机化（Domain randomization）**：默认随机化器有独立开关：重力、逐 geom 摩擦/刚度/阻尼、link 质量、根 CoM 偏移、逐 DoF 关节摩擦损耗/阻尼/惯量（armature）、PD 增益、观测噪声；采样参数存在 Flax dataclass 里，"活在显式 JIT 可见的状态中，而不是可变的 Python 全局变量"（附录 B）。
4. **初始状态分布**：默认位形、用户指定位形、mocap clip 上的均匀采样（DeepMimic 式训练的标准做法）、用户自定义采样器；同一个 hook 让评估时能 reset 到 clip 的特定相位（附录 B）。
5. **终止条件**：按名选择：无终止、根高度范围、数据集导出的根高度+朝向范围、或根高度/倾斜相对当前参考帧的误差界。纯 RL 通常只因摔倒终止；模仿还因偏离参考姿态太远而终止，"这就产生了逐 clip 评估度量里可见的截断 rollout"（附录 B）。
6. **控制模式**：`DefaultControl`（动作直通执行器：电机或肌肉）与 `PDControl`（关节位置目标）。"力矩控制带宽高、刚性大；PD 控制提供低带宽的参考目标，是 sim-to-real 腿式系统常用的部署方式"（附录 B）。
7. **地形（Terrain）**：flat 或 reset 时重生成的粗糙 heightfield；与其它所有插槽解耦，"可以固定一切只换地面，让地形效应不与训练设置的改变混淆"（附录 B）。
- **为什么能 scale**：每个插件无状态，逐 env 状态全在 Flax dataclass 里，整个 env step 可在 `jax.jit` + `jax.vmap` 下干净编译；任务工厂按插件组装机器人-任务对，"这些用例的区别只在配置"（附录 B）。

### 1.5 Embodiment、数据集与重定向（§3.3、附录 D）

- 三个数据集源：原生 LocoMuJoCo 数据集（HF Hub）、LAFAN1、AMASS（后者需用户接受 SMPL 许可）。原生数据集为每个人形都提供（§3.3）。**注意：原生 mocap 数据集只覆盖人形；四足没有原生 mocap clip**——四足在库里是纯 RL 速度任务 embodiment，不是模仿任务 embodiment（§3.3 的措辞是 "ships a curated native motion-capture dataset for every humanoid"；`datasets/` 目录下也只有 `humanoids/` 与 `LAFAN1/`）。
- 重定向管线（附录 D，三阶段，全文技术含量最高的一节）：
  - **Stage 1**：每机器人拟合一次 SMPL 形状 β 与各向同性尺度 s：T-pose 下 15 个 mimic-site 位置对齐，Adam 1000 步、lr 0.05（`defaults.yaml` 的 `shape_iterations` / `shape_lr`）；G1 误差 112.1 → 21.8 mm；同时记录每 site 的刚体偏移 (R^s2r, t^s2r)、足高修正 Δz、头足高度比，存为 `shape_optimized.pkl`。
  - **Stage 2**：SMPL 运动 → 机器人轨迹：每帧把 15 个 site 的目标位姿 (τ, Q) 装进 MuJoCo mocap body，用 **软 `mjEQ_WELD` 等式约束**（SE(3)，不是只给位置的 `mjEQ_CONNECT`）连到机器人 site；`N_sub=25` 个子步、warm-start 上一帧积分出 q；软约束的 solref/solimp 表现为阻尼弹簧式阻抗，允许有限违反——"当运动学合理性与精确 site 匹配冲突时（例如人类参考会让足穿过地面），优化器可以偏向运动学合理性"，且同一物理步同时执行碰撞、关节限位与（可选）力矩限位，所以轨迹更可能是物理可行的；末尾再做一次逐轨迹的地面偏移（式 5），让最差帧恰好触地；速度用中心差分。
  - **Stage 3**：机器人→机器人：先把源机器人运动逆向优化回 SMPL（Adam 400 步、lr 0.005，λ_p=1.0、λ_r=0.1，1-D 高斯平滑核 3、σ=0.5），再走 Stage 2 到目标机器人。加一个机器人"只需要一个配置文件与模型里的 15 个 mimic site"（附录 A）。
- 对我们的意义见 §3 C-9：我们没有人类 mocap 可学（狗不是人），但 Stage 2 的"软约束 + 物理步同时执行限位/碰撞，生成物理可行的参考轨迹"这一机制思想，可直接对应到我们用 keyframe 生成 SIT/lie/getup 参考轨迹的问题。

### 1.6 算法套件（§3.4、附录 E）

- 每条算法一个模块 `<algo>_jax.py`（另有 `experimental/` 子包），共用小规模网络与基类；训练循环与策略更新融进一个 `lax.scan` 构建的 JIT 函数，"环境 stepping、优势计算、梯度更新全在设备上执行，没有 Python 开销"（附录 E）。
- **PPO**：默认纯 RL 主干，也是模仿算法的内环优化器；支持多 GPU 数据并行（每卡等分 env，梯度平均——与 loco-mujoco-real 的训练脚本同款做法）。
- **GAIL**：判别器输出的 log 概率作奖励，PPO 优化；适用于有专家演示、无手工任务奖励、不需要逐帧跟踪的场合。
- **AMP**：GAIL 式判别器（状态转移，编码运动风格）+ 用户任务奖励（速度跟踪）；"是 LocoMuJoCo 人形 env 上风格化 RL 的推荐算法"。
- **DeepMimic**：逐帧跟踪奖励（MimicReward：关节位置/速度 + mimic-site 的根相对位置/朝向/速度），PPO 训练；是原生 mocap clip 的默认 IL 训练信号。
- **SuperMimic**：世界模型法：拟合 simulator 转移的世界模型，确定性策略通过短想像 rollout 反向传播改进，TD(λ) critic 兜底；想像步就是普通 env 步，只是动力学换成学到的模型，"观测、奖励、终止都是 env 自己的函数，算法与机器人、观测、奖励无关"。
- 其余：SAC（off-policy，样本效率更高）、TD3（确定性策略对照）、S2PG（状态化策略、无 BPTT 的梯度不稳定）、BPTT-PPO（全 recurrent LSTM/GRU，长时记忆）、历史感知 PPO（固定长度 obs/action 栈 + MLP 或 Transformer 编码）、DAgger 蒸馏（每 env 独立以概率 β 选学生/教师、但所有 transition 都标教师动作，学生做监督 NLL；用于把 per-clip/per-embodiment 的 PPO 专家压缩成单个多任务通才）。

### 1.7 评估：轨迹相似度，而不只看 return（§3.4、附录 H.1）

- 评测度量模块基于 JAX 原生 `metrx` 库（Pompetzki et al. 2024），提供三种轨迹相似度：**Euclidean**（同时间步的均值欧氏距离，也惩罚"动作对但早到/迟到"）、**动态时间规整 DTW**（先单调时间对齐再求和，对时序偏移不敏感）、**离散 Fréchet 距离**（同样对齐、但取沿对齐的最大距离，报告最差偏差而非平均）。三者在 11 个关节/身体/site 量上计算（见 §2.11），训练时与 return 并排记录（图 8）。
- 论文明确指出动机："Return 与 episode 长度只说明策略活了多久，不说明它跟参考运动跟得多近"（附录 H.1）。这句话直接对应我们的 SIT/getup 评估现状（见 §3 C-7）。

### 1.8 实验结果（附录 H）

- **实验 1（PPO + DeepMimic，11 embodiment × 13 clips × 5 seeds = 715 运行）**：每 clip 的专家策略；最终误差按 embodiment 排序在三个度量下高度一致；少数失败种子的终步误差可达典型种子的 50–500 倍，所以图 8 用中位数/IQR 而不用均值/CI（附录 H.1）。
- **实验 2（H1 上 PPO vs SAC vs TD3，各 65 运行）**：PPO 最高 return（~1530），SAC ~1260，TD3 ~1080；episode 长度都收敛到 ~900–925。"这是 LocoMuJoCo 把 PPO + DeepMimic 作为默认训练配方的理由之一"（附录 H.2）。
- **实验 3（H1 上 PPO vs GAIL，12 clips × 5 seeds）**：PPO 收敛到约 2 倍训练 return、约 5–10 倍更低的关节位置验证误差；论文把差距归因于"在高维全身人形轨迹上训练判别器而没有显式逐帧 shaping 信号的众所周知的困难"；GAIL 主要作为对照基线与 AMP 式混合的起点（附录 H.3）。
- **实验 4（SuperMimic，真机 env，仿真内）**：每机器人一个策略训全部 40 个 LAFAN1 clips，1.53×10⁸ env steps，4096 个并行 MjWarp env，DR + 推力 + 自碰撞，单张 RTX 6000 Ada 约 50 分钟；X2 Ultra 归一化 return 0.776±0.004、失败 clips（3 seeds 各）1/0/0、完成帧 99.7%；G1 0.674±0.020、失败 2/0/1、完成帧 99.1%（表 4）。另：BONES-SEED 舞蹈 43 s  routine，4.56×10⁸ steps 后仿真内 99.9% clips 完成。
- **实验 4 的 sim-to-real DR（附录 H.4，全文最可直接抄的一段）**：只针对传感与扰动、不针对被仿植物：(i) 随机推力：根速度脉冲，水平 ≤0.5 m/s、yaw ≤0.78 rad/s，每 1–3 s；(ii) 每 episode 编码器偏置 ≤±0.02 rad；(iii) 观测均匀噪声：关节位置 ±0.01 rad、关节速度 ±0.5 rad/s、基座角速度 ±0.2 rad/s；(iv) 全部碰撞 geom 的摩擦系数 ∈ [0.3, 1.2]；(v) 骨盆 CoM 每轴偏移 ≤5 cm。**噪声与偏置只加在 actor 的观测上，critic 拿干净值**。评估时全部关掉，"测的是跟踪质量而非鲁棒性"。

### 1.9 真机部署（§3.5、附录 H.4）

- loco-mujoco-real 覆盖 Unitree G1 与 AgiBot X2 Ultra：加机器人 env、导出训练好的策略、做 sim-to-sim 检查、再上真机，用"同一套观测与控制代码"。
- 从 checkpoint 到机器人：导出成 bundle（网络权重 + 控制器设置 + 参考运动），只用 NumPy + MuJoCo 就能跑；上硬件前三道检查：验证脚本确认 bundle 逐项复现训练 env 的观测与动作；plain MuJoCo 里 rollout 一次（验导出、不验 sim-to-real gap）；sim-to-sim 里控制器与仿真机器人作为两个独立进程、走机器人自己的中间件（AgiBot AimDK over ROS 2 / Unitree SDK），"所以控制器已经作用在延迟后的状态上"。
- 控制循环：策略 50 Hz；观测只用 onboard 传感：编码器关节位置/速度、骨盆 IMU 的投影重力与角速度、上一动作、要跟踪的参考帧；**没有基座状态估计、没有外部跟踪**；动作是关节位置目标，由驱动器自己的 PD 环跟踪（用仿真位置伺服的刚度/阻尼）；X2 上一步策略约 2 ms、状态约 2 ms 旧；控制器在 stale state、关节超限位、骨盆倾斜超 ~60° 时会 damp 机器人。
- 图 2/图 11：X2 Ultra 上 43 s 动作，仿真与真机并排同一时刻。

### 1.10 论文的局限与本报告未读到的部分

- **相关工作表（表 1）的 mjlab 一行**：论文称 mjlab 是 GPU-only（无对等的串行 NumPy 后端）、无 curated 人类 mocap 数据集、无自动跨机器人重定向、无生物力学模型（§2、附录 A）。这与我们的实际使用一致（我们用 mjlab 的 GPU 路），但注意：**我们自己在 mjlab 之外另有一套 CPU torch env**（`env_cpu.py` / `train_cpu.py`），这是我们自己的双路，不是 mjlab 自带的——论文的批评针对 mjlab 本体，不针对我们的组合。
- **版本差**：附录 G 的吞吐对照用 LocoMuJoCo 1.2.0 vs mjlab 1.6.1、MuJoCo 3.11；我们训练钉版是 MuJoCo 3.8.1（`reference.py` 头注释）与 mjlab 1.2.0（`collie_mjlab_train.py` 注释"mjlab 1.2.0 has no such CLI flag"）。跨版本比吞吐时须注意版本差。
- **图内数值未读到**：图 6/7/8/9/10 的曲线、图 1 的 embodiment 渲染、图 4 的重定向渲染、图 5 的管线图，本报告只读了图注文字，未从图中提取数值；凡论文只在图中给出的数，本报告不引用。
- **没有四足的模仿实验**：全部 715 运行的实验 1–3 都是人形；四足只出现在 embodiment 列表与 RL 速度任务里，没有任何四足的 mocap 跟踪数字。把 LocoMuJoCo 的实验结论外推到四足时须打折扣（见 §3 C-8）。
- **没有"最佳实践清单"式结论**：论文是 benchmark 论文，不给"这样训一定成"的处方；§4 的建议全是我们从证据外推的工程判断，不是论文的原话。

---

## §2 仓库精读：loco-mujoco（HEAD `df9869e`）

### 2.1 结构与版本

- 顶层：`loco_mujoco/`（包）、`examples/`（tutorials + training_examples）、`docs/`、`imgs/`、`tests/`（论文称 >500 测试，未逐个读到）。`pyproject.toml` 的 version 是动态字段（`dynamic = ["version"]`，:44），本轮未在该文件里定位到字面版本号；论文附录 G 称对照版本为 LocoMuJoCo 1.2.0，README 称"新 release v1.2"，以论文/ README 为准。
- 包内结构（`loco_mujoco/`）：`core/`（仿真核：observations、reward、control_functions、domain_randomizer、terrain、initial_state_handler、terminal_state_handler、trajectory、wrappers、visuals、utils）+ `environments/`（base + humanoids + quadrupeds）+ `task_factories/`（base、rl_factory、imitation_factory、dataset_confs）+ `algorithms/`（ppo/gail/amp/supermimic + common + experimental）+ `smpl/`（重定向）+ `datasets/`（humanoids、LAFAN1、data_generation）+ `utils/`（dataset、metrics、logging、running_stats、video）。
- README 要点：Python ≥3.10；模型约 200 MB，首次建 env 自动下载、缓存在 `~/.loco-mujoco-caches/robot_models`；JAX 钉在 `<=0.10.1, !=0.10.0`（0.10.0 与 0.10.2 编译 SuperMimic 训练函数失败）；默认装 CPU 版 JAX，GPU 需 `jax[cuda12]`。

### 2.2 观测：具名 spec 列表 + 注册表（`core/observations/base.py`，1190 行）

- `Observation` 基类（:208）+ `ObservationType.register` 注册机制（:354–365）；`ObservationContainer`（:54）管名字、分组、随机化许可；`ObservationIndexContainer`（:21）记录每类观测在拼接向量里的下标；`get_randomizable_obs_indices`（:183）决定哪些下标允许被 DR 加噪声——这是 DR"只动允许动的通道"的实现点。
- 观测类（按 `grep "class "` 结果）：`BodyPos`（:433）、`BodyRot`（:456）、`BodyVel`（:479）、`FreeJointPos`（:502）/`EntryFromFreeJointPos`（:524）/`FreeJointPosNoXY`（:551，去掉世界 x/y，只留 z + 四元数）、`JointPos`（:573）、`JointPosArray`（:600）等；另有 `StatefulObservation`（:390，带状态的观测，如历史/滤波类）。`goals.py`（1217 行）放目标类观测，其中 `GoalRandomRootVelocity`（:251）默认 `max_x_vel = max_y_vel = max_yaw_vel = 1.0`（:266–270），状态是 `goal_vel_x/y/yaw` 三个浮点（:237–249）。
- 关键设计：观测是**声明式组合**，不是 env 硬编码——env 类只列一个 spec 表（见 §2.8 Go2 例），改观测不改 env 逻辑。

### 2.3 奖励：三类 + 两处"钳制"口径（`core/reward/`）

- `default.py`：`NoReward`（恒 0）；`TargetXVelocityReward`（`exp(-(v_x - v_target)²)`，无系数）；`TargetVelocityGoalReward`（:起）：跟踪奖励 = `exp(-w_exp_xy·mean((v_xy - g_xy)²))` 与 yaw 项之和，默认 `tracking_w_exp_xy = tracking_w_exp_yaw = 10.0`、`tracking_w_sum_xy = tracking_w_sum_yaw = 1.0`；它**断言** env 里必须有 `GoalRandomRootVelocity` 观测（"GoalRandomRootVelocity is the required goal for the reward"）——奖励与观测 spec 是显式耦合的，缺了就报错，而不是静默给 0。
- `LocomotionReward`（`default.py:223`，继承 `TargetVelocityGoalReward`）：在跟踪奖励上加 12 项惩罚/正则，默认系数（`__init__`，:255 起）：`z_vel 2.0`、`roll_pitch_vel 5e-2`、`roll_pitch_pos 2e-1`、`nominal_joint_pos 0.0`（默认关闭）、`joint_position_limit 10.0`、`joint_vel 0.0`（默认关闭）、`joint_acc 2e-7`、`joint_torque 2e-5`、`action_rate 1e-2`、`air_time_max 0.0` / `air_time 0.0`（默认关闭）、`symmetry_air 0.0`（默认关闭）、`energy 0.0`（默认关闭）。注意默认关闭的项不少——"locomotion 奖励"出厂只开速度跟踪 + 姿态/限位/加速度/力矩/动作率惩罚，air-time 与对称性要显式打开。状态是 `LocomotionRewardState`（:214）：`last_qvel`、`last_action`、`time_since_last_touchdown`（每足一个计时器）；air-time 奖励在足触地瞬间结算 `tslt[i] - air_time_max`，对称性惩罚统计"同侧双足同时离地"的次数（前对 [0,1]、后对 [2,3]）。**总奖励 `backend.maximum(total_reward, 0.0)`（:513）**——负值钳到 0。
- `trajectory_based.py`（648 行）：`MimicReward`（:150，DeepMimic 式）：五项 exp 核——关节位置（`qpos_w_exp 10.0`）、关节速度（`qvel_w_exp 2.0`）、根相对 site 位置（`rpos_w_exp 100.0`）、根相对 site 朝向（`rquat_w_exp 10.0`）、根相对 site 速度（`rvel_w_exp 0.1`）；求和权重默认 `qpos_w_sum 0.0`、`qvel_w_sum 0.0`、`rpos_w_sum 0.5`（:191）、`rquat_w_sum 0.3`、`rvel_w_sum 0.0`——**默认只计相对 site 的位置与朝向，关节空间两项权重是 0**。另有动作越界代价（`action_out_of_bounds_coeff 0.01`）、关节加速度/力矩/动作率惩罚（默认 0.0）；**惩罚项之和钳在 ≥ -1.0**（:417），**总奖励钳在 ≥ 0**（:429），NaN 置 0。`jax_rl_mimic/conf.yaml` 的实战权重与默认不同：`qpos_w_sum 0.4`、`qvel_w_sum 0.2`、`rpos_w_sum 0.5`、`rquat_w_sum 0.3`、`rvel_w_sum 0.1`，mimic sites 只取 5 个（upper_body、左/右手、左/右足）。另有 `SiteTrackingReward`（:449，BeyondMimic 式替代）与 `TargetVelocityTrajReward`（:47）。
- 相对 site 量的定义（附录 H.1 + `utils/metrics.py`）：mimic-site 的世界位姿表达到根坐标系下，得 `site_rpos`（位置）、`site_rrotvec`（朝向，轴角）、`site_rvel`（线速度）；"相对 site 量是 DeepMimic 奖励的主导信号：策略被奖励去匹配**相对于身体**的末端轨迹，这正是模仿能跨不同比例 embodiment 迁移的原因"。
- `SiteTrackingReward`（`trajectory_based.py:449`，BeyondMimic 式，Liao et al. 2025）：与 `MimicReward` 的三点区别——① **没有关节空间项**，只跟 site；② 主 site（第一个 mimic site）在世界系里跟；③ 其余 site 跟的是"搬到机器人当前位置与朝向"的参考（`ref_pos_moved`：取机器人主 site 的水平位置 + 参考自己的高度/横滚/俯仰，再按朝向差旋转），"这样漂移只降低两个主 site 项"。六项 exp 核（`exp(-dist/std²)`），std 默认：主位置 0.3、主朝向 0.4、site 位置 0.3、site 朝向 0.4、site 线速度 1.0、site 角速度 3.14；权重默认主位置/主朝向各 0.5、其余四项各 1.0；惩罚是动作率（-0.1）与**软关节限位**（超出关节 range 中央 90% 区间才罚，系数 -2.0，`soft_joint_pos_limit_factor 0.9`）；总奖励乘控制步长 `dt`（`scale_by_dt=True`，BeyondMimic 同款）；`max_reward` 属性 = `dt × 权重和`，即"在参考上走一步、无惩罚"的奖励是已知常数——这让归一化 return（附录 H.4 表 4 的 normalized return）有分母可算。

### 2.4 域随机化：显式状态 + 独立开关（`core/domain_randomizer/default.py`）

- `DefaultRandomizerState`（:18）是 Flax dataclass，字段：`gravity`、`geom_friction`、`geom_stiffness`、`geom_damping`、`base_mass_to_add`、`com_displacement`、`link_mass_multipliers`、`joint_friction_loss`、`joint_damping`、`joint_armature`。
- `reset()` 采样、`update()` 写回模型（`body_mass`、`body_ipos`、geom `solref`（刚度/阻尼进 solref 的两个分量）、`dof_frictionloss` / `dof_damping` / `dof_armature`（下标 6: 起，跳过 free joint）、`opt.gravity`、`geom_friction`）；`update_observation()` 给观测加噪声，但只写回 `get_randomizable_obs_indices()` 允许的下标；`update_action()` 原样返回动作。
- PD 增益噪声：只在控制函数是 `PDControl` 时才采样（`isinstance(env._control_func, PDControl)`），以 `p_gains_noise_scale × init_p_gain` 为幅度的均匀噪声。
- **本轮未定位到 `rand_conf` 各范围的默认值**：`default.py` 只读 `rand_conf["..._range"]` 键（如 :411 的 `geom_friction_tangential_range`），`base.py:32` 只做 `self.rand_conf = randomization_config`；默认值由调用方（env 配置 / hydra conf）传入，本轮读到的文件里没有一张完整的默认范围表。凡本报告引用 LocoMuJoCo 的 DR 范围，一律只引附录 H.4 的 sim-to-real 五条（那是论文白纸黑字的数），不引未定位的 `rand_conf` 默认值。

### 2.5 地形：flat / 粗糙 heightfield / 静态 heightmap（`core/terrain/`）

- `base.py`（336 行）：`Terrain`（:13）是 `StatefulObject`，`reset` / `update` / `modify_spec` / `get_height_at` / `has_env_origins` / `get_env_origin_xy` / `shift_to_env_origin` 等接口；地形可以带"每 env 原点"（多 env 平铺时各自原点不同，MimicReward 在比 qpos 前会减掉原点 x/y，见 `trajectory_based.py` 的 `free_x_in_qpos` 处理）。
- `rough.py`（304 行）：`RoughTerrain`（:32，`DynamicTerrain`）：reset 时重生成随机地形（`_np_random_uniform_terrain` / `_jnp_random_uniform_terrain`，:242/:258），`_reset_on_edge`（:277）处理边缘 reset。
- `heightmap.py`（469 行）：`HeightmapTerrain`（:16，`StaticTerrain`）：从文件加载 heightfield、分块（tile）、建 BVH（`_pool_bvh_nodes`）、`trajectories_in_world_frame` / `grounds_robot` 等；`static.py`（100 行）与 `dynamic.py`（19 行）是两类基类。

### 2.6 控制：直通 vs PD（`core/control_functions/`）

- `default.py`（77 行）：`DefaultControl`，动作直通执行器。
- `pd.py`（191 行）：`PDControl`（:28）：内部把动作空间归一化到 [-1, 1]，"但用关节位置限位作为 env 的限位"；`scale_action_to_jnt_limits=True` 时按关节 range 与 nominal 位置算 `norm_act_mean` / `norm_act_delta`，把 [-1, 1] 映射到关节限位区间；只支持 hinge/slide 关节（:67 断言）；状态 `PDControlState`（:18）：`p_gain_noise`、`d_gain_noise`、`pos_offset`、`ctrl_mult`；`generate_action`（:127）产出 PD 目标。
- 论文附录 B 的定性判断（§1.4 已引）：力矩控制高带宽、刚性大；PD 控制低带宽参考目标，是 sim-to-real 腿式系统常用部署方式。

### 2.7 终止与初始：按名选择（`core/terminal_state_handler/`、`core/initial_state_handler/`）

- 终止：`HeightBasedTerminalStateHandler`（`height.py`，144 行；RLFactory 默认）；`RootPoseTrajTerminalStateHandler`（`traj.py`，307 行；ImitationFactory 默认）：根高度裕度默认 0.3 m、根转角裕度默认 30°（:21–22），另有 `max_root_pos_deviation`（默认 1e6，即不启用）；软终止温度 `soft_temperature_m = 0.02`、`soft_temperature_rad = 0.05`（:17–18），供 SuperMimic 的 `mjx_absorbing_prob` 用；`no_terminal.py`（105 行）是什么都不终止。
- 初始：`DefaultInitialStateHandler`（`default.py`，110 行）；`TrajInitialStateHandler`（`traj_init_state.py`，123 行）：reset 时从轨迹采样初始 qpos/qvel（RSI），可加噪声：根位置/转角/线速度/角速度（默认全 0）与关节位置噪声（默认 0.0，超限位时 clip 到关节 range）。
- 工厂默认值（`task_factories/rl_factory.py` 全文 52 行）：`RLFactory.make` 默认 `init_state_type = "DefaultInitialStateHandler"`、`terminal_state_type = "HeightBasedTerminalStateHandler"`、`goal_type = "GoalRandomRootVelocity"`、`reward_type = "TargetVelocityGoalReward"`。`imitation_factory.py` 默认 `terminal_state_type = "RootPoseTrajTerminalStateHandler"`、`init_state_type = "TrajInitialStateHandler"`。

### 2.8 四足 embodiment：以 Go2 为例（`environments/quadrupeds/`）

- 四个四足各有一对文件：CPU 版（`unitreeA1.py`、`unitreeGo2.py`、`bd_spot.py`、`anymal_c.py`）与 MJX 版（`*_mjx.py`），共用 `base_robot_quadruped.py`（17 行，`BaseRobotQuadruped(LocoEnv)`，:5）。
- `UnitreeGo2`（`unitreeGo2.py`，323 行）：类 docstring 里印着完整的默认观测/动作表——观测 35-D：`q_root`（`FreeJointPosNoXY`，5-D：z + 四元数，去世界 x/y）+ 12 个关节位置 + `dq_root`（`FreeJointVel`，6-D）+ 12 个关节速度；动作 12-D、每维 [-1, 1]、`DefaultControl`（力矩直通，不是 PD）；`mjx_enabled = False`（:117）；`init_qpos` 根 z 0.27、thigh 0.9、calf -1.8（12 关节统一值）；`root_height_healthy_range` (0.25, 1.0)（:286–293）；足 geom 名 `RL_foot / RR_foot / FL_foot / FR_foot`；`grf_size` 12；`upper_body_xml_name` 就是根 body `"base"`。
- `MjxUnitreeGo2`（`unitreeGo2_mjx.py`）：`mjx_enabled = True`（:8）；`timestep=0.002`、`n_substeps=5`（:10）；solver `iterations=8`、`ls_iterations=12`、pyramidal cone、`impratio=1`、禁用 `EULERDAMP`（:12）；`_modify_spec_for_mjx`（:22 起）做两件事提速：① 全部 geom 的 contype/conaffinity 置 0；② 只保留 4 对接触（floor × 四足，:43–46）；注释明说"MJX 不擅长处理复杂接触"，足 mesh 换成每足两个 capsule。
- A1 与 Go2 同款结构（`unitreeA1.py:118` 同样 `mjx_enabled = False`、默认 `DefaultControl`）。**四足 env 的默认控制是力矩直通，不是 PD**——与 loco-mujoco-real 的 G1/X2（29-DoF 位置伺服）形成对照：人形部署用 PD，四足基准用 torque。

### 2.9 训练管线：PPO 与 SuperMimic（`algorithms/`、`examples/training_examples/`）

- `ppo_jax.py`（759 行）：单 JIT 训练函数（`lax.scan` 融合 env stepping、GAE、梯度更新）；支持 `pmap` 多卡（梯度跨卡平均，:439 起注释）；GAE（:332–335）、clip value loss（:362–368）、`clip_eps`、`vf_coef`、`ent_coef`、`max_grad_norm`、`anneal_lr`、`desired_kl` 等标准 PPO 件；奖励归一化（`update_reward_norm`，:300）。
- `jax_rl_mimic/conf.yaml`（DeepMimic 实战配置，G1）：`lr 1e-4`、`num_envs 2048`、`num_steps 200`、`total_timesteps 300e6`、`update_epochs 4`、`num_minibatches 32`、`gamma 0.99`、`gae_lambda 0.95`、`clip_eps 0.2`、`init_std 0.05`、`learnable_std false`、`ent_coef 0.0`、`vf_coef 0.5`、`max_grad_norm 0.5`、`activation tanh`、`anneal_lr false`、`desired_kl 0.01`、`hidden_layers [512, 256]`、`horizon 1000`、`normalize_env true`、`validation.active true`；控制是 `torque_control`（DefaultControl）；mimic sites 5 个（见 §2.3）。
- `supermimic_jax.py`（1767 行，文件头 docstring 是最好的算法说明）：世界模型的状态就是 MuJoCo 自己的 `(qpos, qvel)`；网络看 heading-invariant 特征，预测速度增量 + 位置修正（显式积分之上），末层零初始化（初始即恒速外推）；想像步 = 普通 `mjx_step`，只是仿真换成学到的动力学（`physics_fn`），随后走 `mjx.kinematics` / `com_pos` / `com_vel`；**接触（`data.contact`）与执行器力（`qfrc_actuator`）在想像 horizon 内取起始状态的值、不传梯度**（MjWarp 下则完全没有）；DR 开启时，逐 episode 的随机化器与控制函数状态展平成 context 向量 `xi` 喂给动力学与 critic；终止用 terminal handler 的 `mjx_absorbing_prob`（若有），否则学一个 head。
- 其余算法文件：`gail_jax.py`、`amp_jax.py`、`algorithms/common/`、`algorithms/experimental/`（BPTT-PPO、S2PG、历史感知 PPO、SAC、TD3、DAgger 在 `examples/training_examples/jax_rl_mimic_*` 各有 conf 目录）。

### 2.10 吞吐对照（附录 G 表 3，全文唯一一组硬数字）

- 硬件：单张 RTX 4090（24 GB）+ i9-13900KF；MuJoCo 3.11；50 Hz、4 个 5 ms 子步、Euler 积分器（mjlab 默认是 implicitfast）；协议：零动作、50 步 warm-up、200 步计时（CPU 2000 步）；G1 模型（loco-mujoco-real 版，34 碰撞 geom、自碰撞）。
- 全自碰撞：mjlab MjWarp 8192 env = 391,556 steps/s；LocoMuJoCo MjWarp 8192 env = 487,553（≈4.9×10⁵）；LocoMuJoCo MJX 8192 env = 8,974；LocoMuJoCo CPU 单 env = 2,147、8 进程 = 15,805。
- LocoMuJoCo MjWarp 相对 mjlab：1024 env 时 1.9 倍、4096 时 1.4 倍、8192 时 1.25 倍；但"mjlab 出厂状态（含 DR 与 implicitfast）"是 98k / 244k / 313k（1024/4096/8192）。
- 机理：MJX 每步评估全部 502 对候选几何对，MjWarp 的 broad phase 丢掉远的；只留足–地接触（14 对）时 MJX 快 4.8–9.3 倍，MjWarp/mjlab/CPU 变化 <3%；legacy G1（MJX 优化设置）MJX 可达 3.4×10⁵。
- 对我们的意义见 §3 C-2：数字本身不能直接搬（机器人不同、版本不同、任务逻辑重量不同），但"接触对数是 MJX 的主导成本"这一机理可直接借。

### 2.11 评估度量实现（`utils/metrics.py`，358 行）

- `SUPPORTED_QUANTITIES`（:16–18）：11 个量——JointPosition、JointVelocity、BodyPosition、BodyVelocity、BodyOrientation、SitePosition、SiteVelocity、SiteOrientation、RelSitePosition、RelSiteVelocity、RelSiteOrientation。
- `SUPPORTED_MEASURES`（:20）：EuclideanDistance、DynamicTimeWarping、DiscreteFrechetDistance（metrx 库）。
- `MetricsHandler`（:44）：按配置选量与度量；相对量需要 `rel_site_ids` 与第一 site 作参考帧；`QuantityContainer`（:23）与 `ValidationSummary`（:38）是三个度量 × 11 个量的容器；`get_relative_site_quantities`（:314）等 getter 直接读 env 状态。

### 2.12 loco-mujoco-real：部署库长什么样（README 全文，461 行）

- 定位："为 LocoMuJoCo 训练的策略提供真机部署协议；每台机器人给它需要的随机化、观测空间与奖励，附训练配置与上机工具"（README）。覆盖 G1 与 X2 Ultra；两个都是 **29-DoF 位置伺服模型、逐电机 armature、全自碰撞、anchored 运动跟踪**——"anchored"指肢体跟踪与全局漂移无关（README Robots 表下方）。
- 同一套 sim-to-real 随机化（两机器人相同）：推力、编码器偏置、足摩擦、CoM 偏移、观测噪声、可选观测延迟与历史。
- 训练：每机器人 `training/<robot>/` 下 `train.py`（PPO）与 `train_supermimic.py`；默认 19 个默认 clips；SuperMimic 默认 40 个 LAFAN1 clips、1.53 亿步、48 GB 显存；24 GB 卡要减半（`num_envs=2048`、`naconmax=70000`、`imag_batch_size=1024`）；G1 上 RTX 3090 约 30k steps/s、RTX A6000 约 44k steps/s。
- 部署五步（README Quickstart）：① 训；② `eval_viser.py` 在浏览器看策略；③ `export_bundle.py` 导出只用 numpy + mujoco 的 bundle + `validate_bundle.py` 逐项验证；④ `rollout_mujoco.py` 在 plain MuJoCo 里跑 bundle；⑤ 按机器人指南上真机（sim-to-sim rig、控制器、硬件流程各有指南）。README 明确警告：真机跑学习策略有危险，"先挂起来、清场、手握急停、先跑 sim-to-sim"。
- 结构（README Repository layout）：`lmj_real/` 是 LocoMuJoCo 文件夹结构的最小复制（`environments/`、`core/`、`models/`、`smpl_robot_confs/`），每件都 subclass LocoMuJoCo 接口、import 时自注册；"加一个机器人不需要改 LocoMuJoCo 核心"。X2 的 env subclass G1Real，只改模型路径与 body/geom/高度属性，其余插件原样复用。
- 致谢一节的关键句：该库的 env 设计"紧密跟随 mjlab 的 G1 运动跟踪配置，包括模型、奖励、终止与随机化"；mjlab 的跟踪任务延续 BeyondMimic；mjlab 本身基于 Isaac Lab 的 API 设计与 MuJoCo Warp。——也就是说，**这条部署链与我们的 mjlab 训练栈是同一血统**，不是平行宇宙。
- 运动数据：默认 clips、LAFAN1、自己的 `.lmj.npz`、BONES-SEED（142,220 个动作，已重定向到 G1，需申请访问；X2 的 BONES-SEED 例程"随后会有"，目前先用 LAFAN1）。

### 2.13 测试（附录 F）

- >500 个测试，覆盖双后端 env、观测/奖励/控制/DR/终止组件、轨迹与数据集管线、重定向管线、渲染与导出工具、每个算法的一次短训练；93% 行覆盖（coverage.py）；排除项（交互式 OpenGL viewer、需许可证的 MyoSkeleton、下载数据集/SMPL 的入口、包 `__init__`）在 `pyproject.toml` 里列明理由；需下载数据的测试用合成轨迹代替，不需联网；GitHub Actions 每次 push/PR 都跑（任务工厂测试除外，因其要加载托管数据集）。
- 本轮未运行测试套件（未安装），覆盖率数字以论文为准。

### 2.14 轨迹格式与重定向默认值（`core/trajectory/`、`smpl/robot_confs/defaults.yaml`）

- `core/trajectory/dataclasses.py`：`Trajectory`（:23，`concatenate` / `to_dict` / `save` / `load`）、`TrajectoryInfo`（:134，`add_joint` / `add_body` / `add_site` / `remove_*` / `reorder_*`）、`TrajectoryModel`（:398）。轨迹是自带模型描述（关节/body/site 表）的一等对象，不是裸数组——`save`/`load` 是 `.lmj.npz` 格式（loco-mujoco-real 的 README 称自己的轨迹文件就是 `.lmj.npz`）。`handler.py` 未细读（见附录 C）。
- `smpl/robot_confs/defaults.yaml` 的关键值（本轮 grep 核实，与论文附录 D 逐值一致）：`shape_iterations: 1000`（:66）、`shape_lr: 0.05`（:68）、`motion_iterations: 25`（:67）、`pose_iterations: 400`（:79）、`pose_lr: 0.005`（:78）、`smoothing_kernel_size: 3`（:80）、`max_height_adjustment: 1.0`（:84）。另有 `init_motion_iterations: 1000`（:75）。每机器人一个 yaml（`UnitreeG1.yaml`、`UnitreeH1.yaml` 等 12 个），共用 `defaults.yaml` 的 15-site 对应表（论文附录 D 表 2）。
- 对我们的意义见 §3 C-19：一个自带模型描述的轨迹文件格式，正是我们 `amp/reference_clips.json` 缺的——我们的参考 clip 是 JSON + 外挂的关节名约定，换模型就要人工对名字。

---

## §3 三方对照：LocoMuJoCo vs BorderCollieRobot（逐项）

对照口径：每一项写对方证据（论文章节 + 仓库文件行）→ 我们对应的文件/行 → 适用/不适用 → 理由 → 采纳它会改变我们什么。我们一侧的基线（本轮亲读，repo HEAD `00a0281`）：训练栈是 mjlab 1.2.0 + RSL-RL PPO（torch）+ MuJoCo Warp，MuJoCo 钉版 3.8.1；回归基线主集 145+1+4failed、bridge 34、parity 8、VSD `2d019d63…`/1971 B；B-1 现状见 C-10。

### C-1 插件化 env vs 我们的 mjlab 任务注册

- **对方**：附录 B 七插槽（观测/奖励/DR/初始/终止/控制/地形），全部无状态、状态进 Flax dataclass；任务工厂按名组装（`task_factories/rl_factory.py` 全文 52 行，默认 `HeightBasedTerminalStateHandler` + `GoalRandomRootVelocity` + `TargetVelocityGoalReward`）。
- **我们**：`tasks/mdp.py`（1822 行）是 GPU 训练 env 的唯一入口：REWARD_FNS 注册表（:161 起，约 80 项）、FN 字典（:1717 起，17 个事件/DR 函数）、事件表（:515 起，含 DR 与 push/impulse）、`SCENE_XML` 四地形（:109–117：flat/rough/stairs/slope）；`tasks/registry.py` 是任务表（kind、terrain、episode_s、weights）；训练入口 `collie_mjlab_train.py` 只是把任务注册进 mjlab registry 再交 `mjlab.scripts.train.main()`（其头注释明说 mjlab trainer 无插件 hook，不注册就报 invalid choice）。
- **适用（部分）**：思想适用，实现不适用。我们已有"注册表 + 事件表 + 任务表"三层，相当于手工版插件系统；但奖励、DR、终止、观测是**按任务写死在 mdp.py 事件表与 registry 权重里**的，换一个组合要改三处（且有四文件 checklist：`rewards.py` → `mdp_torch.py` → `mdp.py` → `tuning/objective.py`，`rewards.py:6` 头注释）。
- **采纳会改变什么**：若把"奖励/DR/终止"抽成按名组装的插槽（仍用 mjlab，不换 JAX），新任务的接线成本从"改四文件 + 同步 FN 字典"降到"改一行配置"；但这是中型重构，须先有回归基线兜底（145+1+4failed），且 `rl/AGENTS.md` 的"四文件 checklist"是血泪教训——重构本身就是 checklist 的消除，不是绕过。见 §4 P1-1。

### C-2 吞吐：MjWarp vs MJX vs 我们的 mjlab

- **对方**：附录 G 表 3：LocoMuJoCo MjWarp 8192 env 487,553 steps/s；mjlab MjWarp 同条件 391,556；LocoMuJoCo MJX 同条件 8,974；机理是 MJX 每步评估全部 502 对候选几何对，MjWarp broad phase 丢远对；只留足–地 14 对时 MJX 快 4.8–9.3 倍。
- **我们**：mjlab 1.2.0 + MuJoCo Warp，`collie_mjlab_train.py` 用法示例 `--env.scene.num-envs 4096`；CPU torch 路 `train_mujoco.py:451–453` 把 n_envs 封顶 512（"use the mjlab backend for 4096"）。我们没有自己测过 steps/s 对照表。
- **适用（机理适用，数字不适用）**："接触对数是 MJX/Warp 的主导成本"这一机理可直接借；但表 3 的绝对数不能搬（机器人不同：G1 34 碰撞 geom vs 我们的 23 体边牧；版本不同：MuJoCo 3.11 vs 我们 3.8.1；任务逻辑重量不同）。
- **采纳会改变什么**：一是给我们的训练加一张自己的吞吐对照表（flat/rough × 1024/4096 env），作为以后改 scene 或加 geom 时的回归尺；二是 `MjxUnitreeGo2._modify_spec_for_mjx` 的做法（foot mesh → capsule、只留足–地接触对）与我们 `rl/AGENTS.md` "walk 模型 strip trunk contacts、fall 不 catastrophic" 的做法同构——可互相印证，但**我们的 MJX 路不存在**（我们是 mjlab Warp，不是 MJX），所以"用 MJX 提速"这条对我们是空的。见 §4 P1-2。

### C-3 奖励聚合：exp 核 + 钳制口径

- **对方**：速度跟踪用 exp 核（`TargetVelocityGoalReward`，`w_exp 10.0`）；`LocomotionReward` 总奖励钳 ≥0（`default.py:513`）；`MimicReward` 惩罚和钳 ≥-1.0、总奖励钳 ≥0、NaN 置 0（`trajectory_based.py:417,429`）。
- **我们**：`rewards.py`（1598 行）同样以 `_exp_kernel`（:20）为公共底座：`track_lin_vel` sigma 0.25（:49）、`track_ang_vel` sigma 0.4（:54）、`base_height` sigma 0.05（:67）、`imitation` sigma 0.3（:405）、`sit_posture` sigma 0.3（:425）；聚合口径是 **only_positive_rewards（OPR）**：`collie_mjlab_train.py` 头注释——"非终止项求和钳 ≥0、termination 在钳后单独施加"，由 `COLLIE_ONLY_POSITIVE_REWARDS=1` 环境变量门控、`reward_opr.py` 实现，2026-10-23 定点修复。
- **适用（已对齐，无需新动作）**：两边在"跟踪用 exp 核、总奖励下钳 0、终止单独施加"三点上已经一致。LocoMuJoCo 的 `MimicReward` 多一处"惩罚和钳 ≥-1.0"，我们没有这一层——但 OPR 的总和钳 0 已覆盖其效果（惩罚再大也不会把总奖励拉穿 0 以下再叠加），多加一层属于重复钳制。
- **采纳会改变什么**：什么都不用改；本项的价值是**独立印证**——OPR 的口径与一个公开 benchmark 的出厂口径一致，2026-10-23 的定点修复不是孤例。见 §4 P2-1（已采纳项）。

### C-4 速度跟踪奖励的权重：我们的 sigma vs 他们的 w_exp

- **对方**：`TargetVelocityGoalReward` 的 exp 核是 `exp(-10·mean((v-g)²))`（xy 与 yaw 同为 10.0）；`TargetXVelocityReward` 是 `exp(-(v-vt)²)`（系数 1）。
- **我们**：`track_lin_vel` 用 sigma 0.25 的 exp 核（`rewards.py:49`），即 `exp(-(err/0.25)²)` = `exp(-16·err²)`；`track_ang_vel` sigma 0.4，即 `exp(-6.25·err²)`（:54）。另有 `cmd_gate_factor`（:38）做命令门控。
- **适用（可直接对）**：两者是同一族核，只差系数：我们的线速度核（16）比他们的（10）更紧，角速度核（6.25）比他们的（10）更松。这是**可计算的差异**，不是风格差异。
- **采纳会改变什么**：若把角速度核收紧到与线速度同量级（或反过来把线速度放宽到 10），会改变 yaw 跟踪与线速度跟踪的相对权重；但我们的 EUREKA HPO（`tuning/objective.py`，Phase 1 权重搜索）已经在搜这些权重——正确的做法不是照抄 10.0，而是把"10.0"作为 HPO 搜索空间的一个锚点。见 §4 P1-3。

### C-5 相对 site 跟踪（MimicReward 的主导信号）vs 我们的 imitation()

- **对方**：`MimicReward` 默认权重只计相对 site 位置（0.5）与朝向（0.3），关节空间两项权重默认是 0（`trajectory_based.py:184–197`）；实战 conf 才把关节项加回 0.4/0.2。论文附录 H.1 明说相对 site 量是主导信号、是跨 embodiment 迁移的原因。
- **我们**：`imitation()`（`rewards.py:405–409`）是**纯关节空间**的 exp 核：`exp(-‖q - q_ref‖/0.3)`，只比关节角；另有 `feet_to_hip_distance`（`mdp.py:161` 注册表内）与 `sit_posture`（:425，关节空间 exp 核）等任务项。**我们没有"根相对足端位置"这一项奖励**——`contract.py` 的 52-D 追加块里虽有 `foot_pos_body` 12-D（:39），那是观测，不是奖励。
- **适用（高度适用，正中当前卡点）**：SIT/getup 的卡点正在于"关节角对了、足相对身体的位置不对"（B-1：lift 已实现、到达 SIT 未达；终点是高蹲姿 root_z 217.1 vs SIT 245.03）。关节空间核对"足相对躯干"不敏感——髋/膝组合可以在关节误差不大时让足端差几厘米，而相对 site 核直接罚这个量。
- **采纳会改变什么**：给 SIT/getup 任务加一个根相对足端位置项（复用 `foot_pos_body` 的观测定义、对照 pins 目标），权重先按 MimicReward 的默认口径（rpos 主导）起步；这是一个**奖励项新增**，走四文件 checklist，不动 contract、不动观测。见 §4 P0-1。

### C-6 参考相对终止（偏离参考太远即截断）vs 我们的终止

- **对方**：`RootPoseTrajTerminalStateHandler`（`traj.py:21–22`）：根高度裕度 0.3 m、根转角裕度 30°，超了即 absorbing；附录 B 明说模仿设置"还因偏离参考姿态太远而终止，这就产生了逐 clip 评估度量里可见的截断 rollout"。
- **我们**：终止项在 `tasks/mdp.py`（:664–725）：各任务的 `terminations` 元组；`sustained_low_root`（0.15 m / 0.5 s，:697）是防塌项；其余是 mjlab 的通用终止（跌倒/超时）。**没有"偏离参考姿态"这一类终止**——SIT 训练里 policy 可以"直接塌掉拿 crouch_depth 分"（2026-10-08 记忆：sit v2 全零、base_low 0.958、bad_tilt 0.833；sit fitness = 3.0·crouch_depth + 0.1·feet_to_hip + 0.8·sit_posture，crouch_depth 是双刃剑）。
- **适用（高度适用）**：参考相对终止正是"塌掉拿分"这一 exploit 的对症药：rollout 一旦偏离 SIT 参考姿态超过阈值就截断，policy 拿不到塌掉后的 crouch_depth 分。
- **采纳会改变什么**：在 SIT/getup 任务配置里加一项参考相对终止（阈值先按 LocoMuJoCo 的 0.3 m / 30° 起步、再按狗的尺度缩放）；这是**任务配置级**的改动，不动 contract、不动奖励函数本体；但会改变训练动态（episode 变短、早期 return 下降），须做对照试验（同 seed、开关各一轮），不能直接当"修复"落地。见 §4 P0-2。

### C-7 评估度量：轨迹相似度 vs 我们的 return + 平滑度

- **对方**：`utils/metrics.py` 11 个量 × 3 种度量（Euclidean / DTW / Fréchet），训练时与 return 并排记录；动机是"return 只说明活了多久"（附录 H.1）。
- **我们**：评估是三件套——任务 return / 成功率（`eval_onnx.py`）、平滑度（`eval_smoothness.py`：action delta RMS/P95/max、rate rad/s、jerk RMS、torque RMS/P95/peak、slew）、以及 B-1 的分相门（max_sp 0.5 rad/s 只考核非 shuffle 相，2026-11-23 裁决 F-3 (i)）。**没有"rollout vs 参考轨迹"的相似度量**：SIT 的到达判断靠终点指标（root_z、trunk 角、support）与 pins（±2 mm），没有 DTW/Fréchet 这类"整条轨迹像不像"的量。
- **适用（高度适用，纯评估侧）**：DTW 对"动作对但时序偏移"不敏感——这正是我们 SIT/lift 评估需要的：S1 lift 的时序（t_move=1.0 不得放慢）与 SIT 到达的时序若差半拍，Euclidean 会罚、DTW 不会；Fréchet 报告最差偏差，对"中途有没有一次大偏离"敏感。
- **采纳会改变什么**：在 `eval_onnx.py` / SIT 评估里加 DTW + Fréchet（对 qpos 与根相对足端位置两个量），用 numpy 实现即可，不需 JAX、不需 metrx；不改变训练、不改变门，只多两个诊断数。见 §4 P0-3。

### C-8 四足 embodiment：Go2/A1 是纯 RL 速度任务，不是模仿任务

- **对方**：四足 env 默认 `DefaultControl`（力矩直通，`unitreeGo2.py` docstring 动作表 12-D [-1, 1]）、默认观测 35-D、默认奖励是 `TargetVelocityGoalReward`（RLFactory 默认）；原生 mocap 数据集只覆盖人形，四足没有任何 clip、没有任何模仿实验数字（附录 H 实验 1–3 全是人形）。
- **我们**：四足边牧，PD 位置伺服（19.0/0.2）+ 关节位置目标（训练 24-D / 部署 16-D）；任务是速度跟踪 + SIT/lie/getup/high-five 等姿态任务；参考来自**我们自己的狗**（`rl/AGENTS.md`："Imitation（optional）：add the phase-matched pose reward vs YOUR dog's mocap"）与手工 keyframe（pins）。
- **适用（框架适用，内容不适用）**：LocoMuJoCo 的四足部分能借的是 env 结构（观测 spec 表、足 geom 命名、健康根高范围），不是它的任务内容；把"人形 mocap 模仿"那套结论（PPO 2 倍 return、GAIL 5–10 倍误差）外推到四足上没有依据——实验里没有四足。
- **采纳会改变什么**：不改变任务设计；但 Go2 的 35-D 观测表是一个有用的**对照尺**：我们的 128-D 里 76-D 冻结前缀 + 52-D 追加块，远大于 Go2 的 35-D——多出来的主要是命令/时钟/相位/足端/地形/历史通道，这是"一机多技能热切换"（`rl/AGENTS.md` Golden rule 4：walk/stand/recover/trick 共用 obs）的代价，不是冗余。见 §4 P2-2。

### C-9 重定向管线：SMPL→机器人 vs 我们的 keyframe→参考轨迹

- **对方**：附录 D 三阶段：SMPL 形状拟合 → 软 `mjEQ_WELD` 约束 + 物理步生成轨迹（`N_sub=25`、warm-start、同时执行碰撞/限位/力矩限位）→ 机器人间迁移；关键机制是"软约束允许有限违反，优化器偏向运动学合理性而非精确匹配"。
- **我们**：参考轨迹来自手工 keyframe（SIT pins：足目标 ±2 mm、root_z 119.8±2、support 67.2±2，B-1 TASK）与 `amp/retarget.py`（视频→参考，`amp/reference_clips.json`）；没有 SMPL、没有人类 mocap——狗不是人，AMASS/LAFAN1 对我们没有内容价值。
- **适用（机制适用，数据源不适用）**：Stage 2 的机制思想可直接借：用 MuJoCo 软约束 + 物理步，把"手工 keyframe 插值出的参考轨迹"过一遍物理可行性（关节限位、足不穿地、自碰撞），而不是直接拿插值当参考。当前我们的 keyframe 是运动学量，物理可行性靠人工检查。
- **采纳会改变什么**：给 SIT/lie/getup 的参考轨迹加一道"软约束物理过筛"（离线脚本，不进训练）：keyframe 插值 → MuJoCo 软 weld 跟踪 → 输出物理可行参考 + 违反量报告；不改变训练、不改变 contract。见 §4 P1-4。

### C-10 B-1 SIT 现状：lift 已实现、到达未达——对方有什么对症的

- **我们现状**（2026-11-23 B-1 TASK + MEMORY）：S1 lift 已落为具名可复用实现（E8e shuffle 终点出发，drag lift + roll 反馈 k=2 mm/deg、±30 mm clamp，t_move=1.0 保留）；S1 在新模型上 n_viol=0、lift max|roll| 1.04°、min support 67.8 mm；终点是高蹲姿（root_z 217.1 vs SIT 245.03、trunk 20.29°、support +117.4）；诚实边界 binding：**"Lift 已实现；到达 SIT 未达"**，到达需要踝刹车硬件（设计轮并行）；刹车需求：后踝约 0.18 N·m（弹簧最大输出的 5.8 倍）、前踝约 0.11 N·m（1.7 倍）；后踝是 pitch ±108° + roll ±45° 双轴，前踝 pitch 单轴（G1 轮 F-5 订正）。
- **对方对症的三件**：
  1. **参考相对终止（C-6）**：SIT 训练的 exploit（塌掉拿 crouch_depth 分）与 LocoMuJoCo 设参考相对终止的动机完全相同。
  2. **相对 site 跟踪奖励（C-5）**：到达 SIT 的本质是"足相对躯干到 pins、躯干到 245.03"——关节核不够，相对 site 核对症。
  3. **RSI（C-11）**：SIT 的难点一半在初始状态分布——从站立出发坐下 vs 从高蹲姿出发坐下是两个难度；LocoMuJoCo 的做法是从 clip 均匀采样初始状态。
- **不适用的一件**：对方的"推力 DR + 编码器偏置 + 观测噪声"（附录 H.4）是针对**已训好策略的上机鲁棒性**，不是针对"学不会 SIT"——SIT 卡住是奖励语义与终止口径问题（2026-10-15 记忆已核实：sit fitness 有专用姿态项但 lie_down 没有；sit 比 lie_down 难在"往下 + 后腿平衡"），加 DR 只会让学习更难。DR 是 P1 的事，不是 SIT 的药。

### C-11 初始状态分布：RSI vs 我们的 reset

- **对方**：`TrajInitialStateHandler` 从轨迹均匀采样初始 qpos/qvel（RSI，DeepMimic 标准做法），可加根/关节噪声（默认 0）；附录 B 明说 reset 可以是默认位形、用户指定位形、clip 均匀采样、自定义采样器四档。
- **我们**：`tasks/mdp.py:515` 的 `reset_joints` 事件：`position_range ±0.1 rad`、`velocity_range ±0.5` 的小偏移（注释称"默认姿态 + 0.1 不会超出关节硬限位"）；没有从参考轨迹采样的 reset；`tasks/mdp.py:697` 的 `sustained_low_root` 是终止侧，不是初始侧。
- **适用（需适配）**：RSI 需要"轨迹"——我们没有 mocap clip，但有 SIT/lie/getup 的 keyframe 序列与 pins。可以做的适配是"keyframe RSI"：从 SIT 过程的关键帧（站立 → 高蹲 → SIT）均匀采样初始状态，而不是只从站立出发。这比纯 RSI 弱（没有连续轨迹），但比 ±0.1 rad 抖动强得多。
- **采纳会改变什么**：SIT/getup 任务的初始状态覆盖从"站立附近"扩到"全过程"，policy 不再只在"从站立坐下"这一条路径上学；实现是 `reset_joints` 的扩展（按 keyframe 列表采样 + 小抖动），不动 contract。见 §4 P1-5。

### C-12 域随机化对照表：我们缺哪几维

| 维度 | LocoMuJoCo（`default.py` 状态字段） | 我们（`tasks/mdp.py` 事件表） | 缺口 |
|---|---|---|---|
| 质量 | link 质量乘子（逐 body）+ 根附加质量 | `dr_mass` pseudo_inertia，alpha_range (-0.0527, 0.0477) → 质量比 [0.9, 1.1]，名义 v2 MJCF 9.22313 kg、带 [8.301, 10.145] kg（:525–541） | 无（且我们的 pseudo_inertia 同时缩放惯量，比对方只改质量更物理——2026-10-09 已修 `body_mass` 只改质量不改惯量的 bug） |
| CoM | 根 CoM 偏移 | `dr_com` ±0.025/±0.025/±0.015（:563） | 无（对方 sim-to-real 例是每轴 ≤5 cm，比我们宽——但那是人形骨盆，不是狗躯干） |
| 摩擦 | 逐 geom 摩擦（切向/扭转/滚动） | `dr_friction` (0.5, 2.0)（:566） | 无（范围一致：对方 H.4 例 [0.3, 1.2]，我们 [0.5, 2.0] 更宽） |
| 关节 | 摩擦损耗 / 阻尼 / armature（逐 DoF） | 无对应事件 | **缺**：关节阻尼、armature、摩擦损耗三维都没有 |
| PD 增益 | PD 增益噪声（相对 init 增益的比例噪声） | `dr_pd_gains` kp/kd ±30%（:575） | 无（口径一致） |
| 几何刚度/阻尼 | geom solref 刚度/阻尼 | 无（`dr_geom_size` 已 DISABLED，:573 注释：会尝试 resize 地面 plane） | **缺**：足底 TPU 的接触刚度/阻尼没有 DR |
| 重力 | 重力采样（只 z 分量） | 无 | **缺**（小） |
| 观测噪声 | 逐通道噪声（关节位置/速度、重力向量、线/角速度），只动允许下标 | 本轮 grep `mdp.py` 与 `collie_params.py` 未见 obs-noise 开关 | **缺**（待核实：噪声可能在别处，如 `adaptation.py`；本报告只说在这两个文件里未见） |
| 推力 | 附录 H.4：根速度脉冲 ≤0.5 m/s、yaw ≤0.78 rad/s，每 1–3 s | `push` ±0.5 m/s x/y，每 5–10 s；`impulse` ±1.0，每 7–12 s（:588 起） | **缺 yaw 推力**；频率比对方低（5–10 s vs 1–3 s） |
| 编码器偏置 | 每 episode ≤±0.02 rad（actor 观测） | `dr_encoder` ±0.02（backlash twin）/ ±0.01（:588） | 无（数值一致） |
| 努力限幅 | —（对方默认随机化器无此项） | `dr_effort` (0.9, 1.1)（:582） | 我们多一项 |

- **结论**：我们的 DR 在"质量/CoM/摩擦/PD/编码器/推力"六维上与对方等价或更强（pseudo_inertia 更物理、摩擦范围更宽）；缺口在**关节阻尼/armature/摩擦损耗、几何刚度/阻尼、重力、观测噪声、yaw 推力**五处。见 §4 P1-6。

### C-13 控制模式：PD 归一化 vs 我们的关节目标

- **对方**：`PDControl` 把动作归一化到 [-1, 1]，再按关节限位与 nominal 位置映射（`pd.py:28` 起，`norm_act_mean` / `norm_act_delta`）；四足默认却是 `DefaultControl` 力矩直通。
- **我们**：策略输出关节位置目标（训练 24-D / 部署 16-D），由 PD 19.0/0.2 跟踪；另有 `delta_action.py`（残差动作）与 `ressafe_mlp.py`（冻结基策略 D + 小残差，corr clamp）；执行器是 BAM QDD 模型 + Backlash twin（`rl/AGENTS.md` Golden rule 3："Actuator fidelity is most of sim2real"）。
- **适用（不适用照搬，值得核对）**：我们的动作已经是"位置目标 + PD"，与对方人形部署的 PD 路线同构；但**动作是否按关节限位归一化**这一点本轮未核实（`delta_action.py` 未细读）。对方的 `PDControl` 把"归一化到 [-1,1] 再映射到限位"做进了控制函数，而不是让策略自己学限位——这是一个值得核对的实现细节。
- **采纳会改变什么**：若核实发现我们的动作是"绝对关节角直通、无限位归一化"，补一层限位映射可减少策略输出越界的探索浪费；若已有，则无动作。见 §4 P1-7（核实项）。

### C-14 算法选择：PPO 默认 vs 我们的 RSL-RL PPO；SuperMimic / GAIL / SAC 的位置

- **对方实验结论**：PPO ~1530 > SAC ~1260 > TD3 ~1080（H1，65 运行/算法）；PPO 约 2 倍 GAIL return、5–10 倍更低关节位置误差（附录 H.2/H.3）；SuperMimic 是"一个策略训 40 clips"的规模化方案（附录 H.4）。
- **我们**：RSL-RL PPO（`collie_mjlab_train.py` 交 `mjlab.scripts.train.main()`）；AMP 已有实现（`amp/discriminator.py`、`train_disc.py`、`reward.py`、`retarget.py`、`reference_clips.json`，2026-11-16 已审计；`rewards.py:412` 的 `amp_style` 是 `-log(1-d)`）；残差策略（`residual_policy.py` / `ressafe_mlp.py`）与 obs 历史（`obs_history.py`）已有；DAgger 蒸馏有 `distill_rma.py`（RMA 蒸馏）。
- **适用（结论印证，不适用照搬算法）**：PPO 默认的结论与我们的选择一致，是印证不是新信息；GAIL 的 5–10 倍误差差距解释了为什么我们的 AMP 是"判别器 + 任务奖励"的混合式（AMP 式），而不是纯 GAIL——纯 GAIL 在全身跟踪上已被实验证明弱。SuperMimic（世界模型 + 想像 rollout）与我们的栈（torch + mjlab Warp + RSL-RL）不兼容：它要求 JAX 环境、想像步走 `mjx_step`、接触/力矩不传梯度；且我们没有 40 个 mocap clips 可供"一个策略训全部"。DAgger 蒸馏的思想（把 per-clip/per-skill 专家压成单个多任务学生）与我们的"一技能一 ONNX、热切换"（`rl/AGENTS.md` Golden rule 4）是**两条不同的部署路线**，不是上下级关系。
- **采纳会改变什么**：算法层无立即动作；SuperMimic 与 DAgger 单学生列为 P2（见 §4）。

### C-15 部署链：bundle 五步 vs 我们的导出验证

- **对方**：loco-mujoco-real 的五步——训 → 浏览器看策略 → 导出 bundle（numpy + mujoco）→ `validate_bundle.py` 逐项验证 → plain MuJoCo rollout → sim-to-sim（走机器人中间件，控制器吃延迟状态）→ 上机；策略 50 Hz、只用 onboard 传感、无基座状态估计、无外部跟踪；stale state / 超限位 / 倾斜超 ~60° 时 damp。
- **我们**：`export_onnx.py`（烘焙 normalizer、切片 16-D）→ `reference.py`（导出时录 64 帧 obs→act、验证时重放、容差 1e-5）→ `validate_deploy.py` / `verify_bundle.py` → `policy_runner.py`（ONNX 热切换）→ `deploy_sim.py`。另有一处 stale 文档：`reference.py` 的 docstring 示例仍写 `"act_dim": 14`（v0.5 值），代码本身从 `contract.py` 读 `ACT_POS_DIM`（现行 16），以代码为准、docstring 待修（小修，不在本轮范围）。
- **适用（高度适用，补齐一环）**：两边在"导出 → 逐项验证 → rollout"三环上已对齐；我们**缺的是 sim-to-sim 这一环**：让导出的 ONNX bundle 走 `policy_runner` + `deploy_loop` 的真实部署路径、驱动仿真里的机器人（含延迟与部署侧状态估计），而不是只在训练 env 里重放 torch。对方的 sim-to-sim 存在的理由正是"控制器已经作用在延迟后的状态上"——这一点我们的 `validate_deploy`（重放 64 帧）测不到。
- **采纳会改变什么**：加一个 sim-to-sim 评估脚本（ONNX bundle → policy_runner → 仿真机器人，含部署延迟），作为上机前的最后一道门；不改变训练、不改变 contract。见 §4 P1-8。

### C-16 历史/记忆：历史感知 PPO、BPTT-PPO、S2PG vs 我们的 obs_history

- **对方**：附录 E 有三档记忆方案——历史感知 PPO（固定长度 obs/action 栈 + MLP 或 Transformer 编码，"在标准 PPO 与 BPTT-PPO 之间务实的中间地带"）、BPTT-PPO（全 recurrent LSTM/GRU、每梯度步 unroll T_bptt 步）、S2PG（状态化策略拆成随机策略状态转移核 + 无状态策略，不用 BPTT、避免梯度不稳定；用于部分可观测的 locomotion 变体）。
- **我们**：`obs_history.py` 已有 `HistoryBuffer`（:67：定长历史、push/get/get_flat）与 `extend_obs` / `split_extended_obs`；`contract.py` 的 128-D 里含 walk_clock 8-D 与 phase_main 3-D（时钟/相位通道）；`residual_policy.py` 与 `distill_rma.py` 是记忆/适应的另一条线（RMA 式）。
- **适用（已对齐，P2 不再往上走）**：我们已经在用"固定长度历史栈"这一档（与对方的历史感知 PPO 同构），且是部署友好的前馈形式。BPTT-PPO 与 S2PG 的 recurrent 策略会堵住 ONNX 导出（50 Hz Jetson 上跑循环策略 + hidden state 管理），且我们的任务（速度跟踪、SIT、getup）尚未证明需要长时记忆——低速步态的卡点是参考与奖励，不是记忆。
- **采纳会改变什么**：什么都不用改；本项是"不要为记忆而记忆"的刹车片。见 §4 P2-7。

### C-17 评估统计：中位数/IQR vs 均值（附录 H.1 的方法论细节）

- **对方**：附录 H.1 图 8 用中位数/IQR 而不用均值/CI，理由是"少数失败模式种子的终步误差可达典型种子的 50–500 倍，会主导均值"；`jax_rl_mimic/conf.yaml:77` 还开 `vmap_across_seeds: true`（多 seed 向量化并行跑）。
- **我们**：`eval_smoothness.py` 已报告 RMS/P95/max（头注释 :3），即已有分位数意识；但任务 return 类指标的报告口径（均值 vs 中位数）本轮未逐项核实——SIT 的历史记录里有过"60 case-run 摔 59 个"这种极端偏态（B'-1 轮记忆），均值在这种分布下几乎没有信息量。
- **适用（高度适用，纯报告口径）**：在 SIT/getup/recovery 这类"成败二值 + 失败占多数"的评估里，并排报告中位数/IQR（或成功率 + 成功子集的中位数），而不是只报均值 return。
- **采纳会改变什么**：评估报告多两列数；不改变训练、不改变门。见 §4 P0-6。

### C-18 多卡数据并行 PPO vs 我们的单卡训练

- **对方**：附录 E 的 PPO"也在多 GPU 上数据并行训练：每设备 stepping 均分 env、梯度跨设备平均，与 loco-mujoco-real 的训练脚本同款做法"；`ppo_jax.py:439` 起有 `pmap` 下已跨设备平均的注释。
- **我们**：单卡训练（家里 3090 24 GB 或 AutoDL 3090 24 GB）；`collie_mjlab_train.py` 的用法示例是单卡 4096 env；没有多卡训练的实践。
- **适用（不适用）**：我们没有多卡可并行；且 LocoMuJoCo 的多卡是 JAX `pmap`，我们的 torch + RSL-RL 多卡是另一套机制（DDP），抄无可抄。
- **采纳会改变什么**：无。见 §4 P2-8。

### C-19 Anchored 跟踪（BeyondMimic 式）vs 我们的 pins 跟踪

- **对方**：`SiteTrackingReward`（`trajectory_based.py:449`）：参考被搬到机器人当前水平位置与朝向再比对，全局漂移只罚主 site 两项；软关节限位（range 中央 90% 之外才罚，-2.0）；奖励乘 `dt` 缩放；`max_reward` 已知（`dt × 权重和`），归一化 return 有分母。loco-mujoco-real 的 README 把这叫 "anchored motion tracking"（肢体跟踪与全局漂移无关）。
- **我们**：B-1 的 pins 是 FK 钉点（足目标钉 pins，`pins_from_qpos()`；足盒位置只作诊断）；`imitation()` 是关节空间核；`sit_posture` 是关节空间核。**我们没有"参考搬到机器人当前位姿再比对"这一层**——参考是固定在世界/身体帧的目标值，机器人整体漂移会直接进误差。
- **适用（需适配）**：SIT 是原地任务，漂移项权重本就不大；但 walk→SIT 的转移段与 getup（从躺到站，根位置大范围移动）正是漂移会污染跟踪误差的场合。Anchored 的思想（肢体相对位形 vs 全局漂移分离）与我们 B-1 的"pins 钉足目标、掌盒只作诊断"已部分同构——pins 本就是身体帧内的相对目标。
- **采纳会改变什么**：在 getup 与 walk→SIT 转移的跟踪项里试 anchored 口径（参考按机器人当前水平位置/朝向平移后再比足端），看漂移段的跟踪误差是否下降；同时给 SIT/getup 的 return 定一个 `max_reward` 式分母（在参考上走一步的奖励），让归一化 return 可算。见 §4 P1-10（对应 C-19）。

### C-20 轨迹文件格式：`.lmj.npz` 自带模型描述 vs 我们的 `reference_clips.json`

- **对方**：`core/trajectory/dataclasses.py` 的 `Trajectory` / `TrajectoryInfo` / `TrajectoryModel`（:23/:134/:398）把"关节/body/site 表 + 时序数据"存成一个 `.lmj.npz`；loco-mujoco-real 的自定义轨迹就是 `.lmj.npz` 文件（README Motion data 一节）；`TrajectoryInfo` 有 `reorder_joints` / `reorder_bodies` / `reorder_sites`（:357/:370/:383）——换关节顺序是格式内操作，不是人工改代码。
- **我们**：`amp/reference_clips.json` + `amp/retarget.py`（视频→参考）；关节顺序约定在 `contract.py` 与 `robot_constants.py` 里，换顺序要改代码 + 重导 ONNX（`rl/AGENTS.md` Golden rule 1："Never reorder joints"）。
- **适用（需适配）**：格式思想适用——把"参考轨迹 + 它用的关节/body/site 表"存成一个自描述文件，评估与重定向共用；但直接采用 `.lmj.npz` 须先验证它能否装我们的 128-D obs 定义与 keyframe pins（它装的是 qpos/qvel/site 量，不是我们的 obs 向量）。
- **采纳会改变什么**：先做格式评估（一个 SIT 参考 clip 转 `.lmj.npz` 往返，看 pins 误差是否保持 ±2 mm 内），再决定是否把 `amp/` 的参考 clip 迁过去；不改变训练。见 §4 P1-9。

---

## §4 应用建议（P0 立即可借 / P1 需适配 / P2 不适用）

分级标准：P0 = 不动 contract、不动 live XML、不训练就能试（配置、奖励项新增走既有 checklist、纯评估侧）；P1 = 需适配或先核实、或要一次对照试验；P2 = 不适用（附理由）。所有 P0/P1 都是建议，不是落地指令；落地须另开轮。

### P0 — 立即可借（6 项）

**P0-1 给 SIT/getup 加根相对足端位置奖励项（对应 C-5）**
- 做什么：新增一个奖励项，罚"根相对足端位置 vs pins 目标"的误差（复用 `contract.py` 52-D 块里 `foot_pos_body` 的定义与 B-1 的 pins 目标）；权重起步按 MimicReward 默认口径（相对位置主导，rpos_w_sum 0.5 级），关节空间核不动。
- 为什么现在：SIT 卡在"关节角对了、足相对身体的位置不对"；对方的实验证据是相对 site 量是 DeepMimic 的主导信号。
- 成本：一个奖励项，走四文件 checklist（`rewards.py` → `mdp_torch.py` → `mdp.py` → `tuning/objective.py`）；不动 contract、不动观测。
- 验收：SIT 任务上同 seed 对照（加项 vs 不加项），看终点 root_z / trunk / support 与 pins 误差，不只看 return。

**P0-2 给 SIT/getup 加参考相对终止（对应 C-6）**
- 做什么：在 SIT/getup 任务配置的 `terminations` 里加一项"偏离参考姿态超阈值即终止"；阈值先按 LocoMuJoCo 的 0.3 m / 30° 起步，再按狗的尺度（体长 ~0.42× 边牧）缩放。
- 为什么现在：直接对"塌掉拿 crouch_depth 分"这一已记录的 exploit；对方附录 B 明说这就是模仿设置加该终止的动机。
- 成本：任务配置级改动；但会改变训练动态（episode 变短、早期 return 下降），必须做开关对照试验，不能直接当修复落地。
- 注意：与 P0-1 联合试时要一次只动一个变量（`rl/AGENTS.md` 专家调试梯：一次一个变量、先预测再测量）。

**P0-3 评估侧加 DTW + Fréchet（对应 C-7）**
- 做什么：在 SIT/getup/lift 的 rollout 评估里，对 qpos 与根相对足端位置两个量，算 rollout vs 参考轨迹的 DTW 与离散 Fréchet 距离（与现有的终点指标、pins、max_sp 并排）；numpy 实现即可，不需 JAX、不需 metrx。
- 为什么现在：纯评估侧，不改变训练、不改变门；DTW 对"动作对但时序偏移"不敏感，正好补 Euclidean/终点指标的盲区（S1 的 t_move=1.0 时序要求）。
- 成本：一个评估脚本的加法；先在已有的 S1 轨迹 JSON 上离线算一次，看两个数是否与"lift 已实现、到达未达"的判断一致，再决定是否进门。

**P0-4 把 LocoMuJoCo 的 H.4 sim-to-real DR 五条写进我们的 DR 对照表（对应 C-12）**
- 做什么：文档动作——在 DR 文档里并列我们的现行范围与对方 H.4 的五条（推力 ≤0.5 m/s + yaw ≤0.78 rad/s 每 1–3 s；编码器偏置 ±0.02；观测噪声 qpos ±0.01 / qvel ±0.5 / 角速度 ±0.2；摩擦 [0.3, 1.2]；骨盆 CoM 每轴 ≤5 cm），并标出"噪声与偏置只加 actor、critic 拿干净值"这一不对称口径是我们**尚未核实**的。
- 为什么现在：零代码成本；为 P1-6 的 DR 缺口补齐提供书面锚点。
- 成本：只改文档；不改任何 DR 参数（我们的摩擦 [0.5, 2.0] 已比对方宽，不因对方的 [0.3, 1.2] 而收窄）。

**P0-5 修 `reference.py` docstring 的 stale act_dim（对应 C-15）**
- 做什么：`reference.py` 头注释示例里的 `"act_dim": 14` 改为 16（v0.6 现行值；代码本身已从 `contract.py` 读 `ACT_POS_DIM`，行为是对的，只有文档 stale）。
- 为什么现在：小修顺手改（措辞/stale 数字类低风险修改直接改、改完验真、改完报一声）；不改行为。
- 成本：一行；改完跑一次 `reference.py` 的自检（若有）或 grep 确认无他处 14。

**P0-6 评估报告并排中位数/IQR（对应 C-17）**
- 做什么：SIT/getup/recovery 的评估报告里，return 与关键终点指标（root_z、trunk 角、support、pins 误差）并排报告中位数与 IQR（或成功率 + 成功子集的中位数），不再只报均值。
- 为什么现在：附录 H.1 的理由直接适用——失败种子的误差可达典型种子的 50–500 倍，均值会被少数失败主导；我们的 SIT 历史里有过"60 case-run 摔 59 个"的偏态分布。
- 成本：报告格式的加法；`eval_smoothness.py` 已有 P95/max 的先例，做法一致。

### P1 — 需适配或先核实（10 项）

**P1-1 奖励/DR/终止按名组装的插槽化（对应 C-1）**
- 做什么：在 mjlab 之上（不换 JAX）把奖励、DR、终止抽成按名组装：任务配置只列名字与参数，`mdp.py` 不再为每个任务手写事件表；四文件 checklist 的同步点随之消除（不是绕过，是消除）。
- 为什么是 P1 不是 P0：中型重构；须先有回归基线兜底（145+1+4failed），且重构期间冻结新任务接入。
- 先决条件：先读完 `mdp_torch.py` 的 REWARD_ADAPTERS 与 `tuning/objective.py` 的权重搜索如何消费 REWARD_FNS，再定插槽边界。

**P1-2 我们自己的吞吐对照表（对应 C-2）**
- 做什么：测一张我们自己的表（flat/rough × 1024/4096 env，mjlab Warp，记录 steps/s 与显存），作为以后改 scene 或加 geom 时的回归尺；同时验证"接触对数是主导成本"在我们模型上是否成立（我们 23 体 vs G1 34 碰撞 geom）。
- 为什么是 P1：要占 GPU（3090/AutoDL），须排期；且表 3 的数字不能直接抄，必须实测。

**P1-3 速度跟踪核系数的 HPO 锚点（对应 C-4）**
- 做什么：把对方的 `w_exp 10.0`（即 sigma ≈ 0.22 的线速度核）作为 EUREKA HPO 搜索空间的一个锚点，与现行 sigma 0.25（线速度）/ 0.4（角速度）对照；不是直接改成 10.0。
- 为什么是 P1：改权重必须走 HPO，不能手拍；且角速度核（6.25）比对方松，收紧与否要看 yaw 跟踪的实测误差。

**P1-4 参考轨迹的软约束物理过筛（对应 C-9）**
- 做什么：离线脚本——keyframe 插值出的 SIT/lie/getup 参考轨迹，用 MuJoCo 软 `mjEQ_WELD` 约束跟踪一遍（`N_sub` 与 solref 参数按附录 D 起步），输出物理可行参考 + 违反量报告（足穿地量、关节超限位量）。
- 为什么是 P1：要先在一条 SIT 参考上验证"过筛前后 pins 误差变化"——若过筛把 pins 拉偏超过 ±2 mm，就不能直接用筛后轨迹当参考；须先有对照数据。
- 不改变训练、不改变 contract。

**P1-5 Keyframe RSI（对应 C-11）**
- 做什么：把 `reset_joints`（现行 ±0.1 rad / ±0.5 抖动）扩展为"从 SIT 过程 keyframe 列表均匀采样初始状态 + 小抖动"；先只用于 SIT/getup 任务。
- 为什么是 P1：需要先把 keyframe 序列整理成可采样的初始状态表（位置 + 速度），且初始状态覆盖扩大后，早期训练的 episode 长度与 return 口径会变，须重定基线。

**P1-6 DR 缺口补齐（对应 C-12）**
- 做什么：按优先级补五维——① yaw 推力（在 `push`/`impulse` 事件上加 yaw 分量，幅值按 H.4 的 0.78 rad/s 起步）；② 关节阻尼/armature/摩擦损耗（先核实 mjlab 的 `mdp.dr.*` 是否支持这三项，不支持则记为 GPU gap、不硬造）；③ 几何刚度/阻尼（足底 TPU 接触的 solref DR；先核实 `dr_geom_size` DISABLED 的同类过滤问题是否同样拦住这一项）；④ 重力（小）；⑤ 观测噪声（先核实噪声是否已在别处实现——本轮只 grep 了 `mdp.py` 与 `collie_params.py`，未见开关；若在 `adaptation.py` 或别处已有，则本项取消）。
- 为什么是 P1：每维都要先核实 mjlab 支持情况，再逐维加、逐维做消融（`rl/AGENTS.md`：单维消融、采样值记日志；全关只能排除、不能归因）。

**P1-7 动作限位归一化核实（对应 C-13）**
- 做什么：读 `delta_action.py` 与 `mdp_torch.py` 的动作路径，核实策略输出的关节位置目标是否经过"按关节限位归一化到 [-1,1] 再映射"；若无，评估补一层的收益（减少越界探索浪费）与风险（改变已训策略的动作语义——已训 ONNX 须重导）。
- 为什么是 P1：纯核实项；若涉及已训策略重导，成本立刻变大，须用户定。

**P1-8 Sim-to-sim 评估环（对应 C-15）**
- 做什么：加一个评估脚本——导出的 ONNX bundle 走 `policy_runner` + `deploy_loop` 的真实部署路径（含部署延迟与部署侧状态估计）驱动仿真机器人，而不是只在训练 env 里重放 torch；先在 walk 任务上跑通，再扩到 SIT/getup。
- 为什么是 P1：要先核实 `validate_deploy.py` / `verify_bundle.py` 是否已覆盖部署路径的一部分（本轮未细读这两文件），避免重复造轮子；且部署延迟的标定值（ms 级）须从实测来，不能拍。

**P1-9 参考 clip 的自描述格式评估（对应 C-20）**
- 做什么：拿一个 SIT 参考 clip 做往返试验——转成 `.lmj.npz`（含关节/body/site 表）再读回，对照 pins 误差是否保持在 ±2 mm 内、关节顺序是否无需人工对名字；通过后再议是否把 `amp/reference_clips.json` 的参考 clip 迁到自描述格式。
- 为什么是 P1：先验证格式能否装我们的 keyframe pins 与 128-D obs 定义（它原生装的是 qpos/qvel/site 量）；通不过就停在评估，不迁移。
- 不改变训练、不改变 contract。

**P1-10 Anchored 跟踪口径试验（对应 C-19）**
- 做什么：在 getup 与 walk→SIT 转移段试 anchored 口径——参考按机器人当前水平位置/朝向平移后再比足端（`SiteTrackingReward` 的 `ref_pos_moved` 做法）；同时给 SIT/getup 的 return 定一个 `max_reward` 式分母（在参考上走一步、无惩罚的奖励），让归一化 return 可算、可跨任务比。
- 为什么是 P1：SIT 原地段漂移权重本来不大，收益集中在转移段；须先在已有转移轨迹上离线算"anchored vs 非 anchored"的跟踪误差差，再决定是否进奖励。
- 不改变 contract；奖励项新增走四文件 checklist。

### P2 — 不适用（附理由，8 项）

**P2-1 奖励总和钳 0 / 惩罚和钳 -1.0（对应 C-3）——已采纳，无动作**
- 我们的 OPR（2026-10-23）已与对方口径一致；再加"惩罚和钳 -1.0"是重复钳制，不加。

**P2-2 全栈换 JAX / 一次实现双后端（对应 C-1、C-2）——不换**
- LocoMuJoCo 的双后端建立在全栈 JAX + Flax 之上；我们的栈是 torch + mjlab Warp + RSL-RL，全部训练资产（EUREKA HPO、reward 四文件、AMP、残差策略、ONNX 导出链）都长在 torch 上。换栈的收益（一次实现、吞吐数字）远小于迁移成本（重写 env、重训全部技能、重导全部 ONNX）；且附录 G 证明 mjlab Warp 本身已是同量级速度（1.25–1.9 倍差距是 env 级别、不是量级差距）。

**P2-3 SMPL/AMASS/LAFAN1 人类 mocap（对应 C-9）——内容不适用**
- 狗不是人；人类 mocap 重定向到边牧没有物理意义。我们的参考只能来自我们自己的狗（mocap/keyframe/视频），这是 `rl/AGENTS.md` 已定的路线。只借 Stage 2 的机制思想（P1-4），不借数据源。

**P2-4 肌肉/肌骨模型（§3.3 的 3 个 skeletal embodiment）——不适用**
- 我们的机器人是刚体 + 伺服，没有肌肉；MyoSuite 式肌肉驱动与我们的 BAM QDD 执行器模型不在同一层。S2PG/BPTT-PPO 等"肌肉探索"算法同样不适用。

**P2-5 SuperMimic 世界模型（对应 C-14）——不适用（当前）**
- 要求 JAX 环境 + `mjx_step` 想像 + 40 个 mocap clips 起步；我们三条全缺（torch 栈、无 mocap 语料、想像步的接触/力矩不传梯度会丢掉我们最关心的足–地接触信息）。若将来有了狗的 mocap 语料库再重议，在此之前列为 P2。

**P2-6 DAgger 单一多任务学生（对应 C-14）——路线不同，不适用**
- 对方用 DAgger 把 per-clip 专家压成单个多任务通才；我们是"一技能一 ONNX、运行期热切换"（`rl/AGENTS.md` Golden rule 4）。单学生会改变部署架构（热切换、回滚、单技能回归基线全要重定），收益（少几个 ONNX 文件）不抵成本。除非用户改部署路线，否则不议。

**P2-7 BPTT-PPO / S2PG 全 recurrent 策略（对应 C-16）——不适用**
- 我们已有部署友好的"固定长度历史栈"（`obs_history.py`），与对方的历史感知 PPO 同构；全 recurrent 策略会堵住 ONNX 导出（hidden state 管理 + 50 Hz Jetson），且当前卡点是参考与奖励、不是记忆。

**P2-8 多卡数据并行 PPO（对应 C-18）——无卡可并**
- 我们单卡训练（3090 24 GB）；对方的多卡是 JAX `pmap`，与 torch RSL-RL 的 DDP 不是同一套机制，抄无可抄。

---

## §5 结论（一页纸）

1. **LocoMuJoCo 是什么**：一个 16 embodiment（9 人形 + 3 肌骨 + 4 四足）的 IL/RL 基准，核心资产不是某个机器人或某个算法，而是**插件化 env + JAX 训练循环 + 轨迹相似度评测 + 部署链**四件套；最硬的数字是附录 G 的吞吐表与附录 H 的 715 运行对照（PPO 最优、GAIL 弱 5–10 倍）。
2. **对我们最有用的三件（全是 P0，前三项）**：① SIT/getup 加根相对足端位置奖励（对方 MimicReward 的主导信号，正中"关节对了、足不对"的卡点）；② 参考相对终止（对"塌掉拿分" exploit 的对症药）；③ 评估侧加 DTW + Fréchet（补终点指标的时序盲区）。其余三件 P0（DR 对照表进文档、修 `reference.py` stale docstring、评估报告并排中位数/IQR）是零代码或纯报告动作。六件都不动 contract、不动 live XML、不训练就能试，但每件都要做开关对照试验，不能直接当修复落地。
3. **已对齐、无需动作的三件**：奖励 exp 核 + 总和钳 0（OPR）、PPO 默认、AMP 混合式（判别器 + 任务奖励）。这三件的价值是独立印证——我们的口径与公开 benchmark 的出厂口径一致。
4. **缺口在 DR 的五维**（关节阻尼/armature/摩擦损耗、几何刚度/阻尼、重力、观测噪声、yaw 推力），列为 P1 逐维核实补齐；其中观测噪声是否已在别处实现，本轮未核实，先核实再动手。
5. **不借的三件**：全栈换 JAX、人类 mocap 数据源、SuperMimic 世界模型。理由分别是迁移成本、狗不是人、三条前置全缺。
6. **B-1 的诚实边界不变**：lift 已实现、到达 SIT 未达；到达需要踝刹车硬件。本报告的三件 P0 都是"让 SIT 学得更好"的训练侧建议，**不改变"到达需要刹车"这一结论**，也不把任何一项写成"SIT 已解决"。

---

## 附录 A 出处清单

| # | 来源 | 本轮读法 |
|---|---|---|
| A-1 | 论文 arXiv:2611.26293 HTML 全文 2230 行（§1–§3.5 + 附录 A–H） | `browser.open` 逐节读完；图内数值未读到；参考文献只扫标题 |
| A-2 | `robfiras/loco-mujoco` HEAD `df9869e8b1d94e7a4470f6fc42f67d3f7defe6a2` | `git clone --depth 1`；核心文件静态阅读，未运行、未安装 |
| A-3 | `robfiras/loco-mujoco-real` README（HEAD `dc9c4d0`） | `browser.open` 读 GitHub 页全文 461 行；其代码文件未逐行读到 |
| A-4 | 我们的 repo HEAD `00a0281`（`/home/hatch/workspace/border-collie-robot/repo`） | 亲读 `rl/collie_rl/` 下列文件（行号见 §3）：`contract.py`、`rewards.py`、`tasks/mdp.py`、`collie_params.py`、`collie_mjlab_train.py`、`train_mujoco.py`、`eval_smoothness.py`、`reference.py`、`residual_policy.py`、`obs_history.py`、`amp/` 目录、`tasks/registry.py`、`rl/AGENTS.md` |
| A-5 | B-1 TASK（`hidden_files/b1_lift_implementation_20261123/TASK.md`）与 MEMORY.md（2026-11-23 08:48 裁决、G1 轮 F-4/F-5） | 任务书与记忆原文 |

## 附录 B 数值核对表（本报告引用的关键数，逐项有出处）

| 数值 | 出处 |
|---|---|
| MjWarp 4.9×10⁵ steps/s（8192 env，全自碰撞） | 论文 §3.1、附录 G 表 3（487,553） |
| mjlab MjWarp 391,556 / LocoMuJoCo MJX 8,974（同条件） | 附录 G 表 3 |
| PPO ~1530 / SAC ~1260 / TD3 ~1080 return | 附录 H.2 |
| PPO 约 2 倍 GAIL return、5–10 倍更低关节位置误差 | 附录 H.3 |
| SuperMimic：1.53×10⁸ steps、4096 env、~50 min RTX 6000 Ada；X2 0.776±0.004、99.7% 帧；G1 0.674±0.020、99.1% 帧 | 附录 H.4 表 4 |
| H.4 DR：推力 ≤0.5 m/s + yaw ≤0.78 rad/s 每 1–3 s；编码器偏置 ±0.02；噪声 qpos ±0.01 / qvel ±0.5 / 角速度 ±0.2；摩擦 [0.3, 1.2]；CoM 每轴 ≤5 cm；噪声只加 actor | 附录 H.4 |
| G1 形状拟合误差 112.1 → 21.8 mm；Stage 1 Adam 1000 步 lr 0.05；Stage 2 N_sub=25；Stage 3 Adam 400 步 lr 0.005、λ_p=1.0、λ_r=0.1 | 附录 D.1–D.3 |
| MimicReward 默认：w_exp qpos 10 / qvel 2 / rpos 100 / rquat 10 / rvel 0.1；w_sum qpos 0 / qvel 0 / rpos 0.5 / rquat 0.3 / rvel 0 | `trajectory_based.py:184–197` |
| LocomotionReward 默认：z_vel 2.0、roll_pitch_vel 5e-2、roll_pitch_pos 2e-1、joint_limit 10.0、joint_acc 2e-7、joint_torque 2e-5、action_rate 1e-2；air_time/symmetry/energy 默认 0 | `default.py:255` 起 |
| Go2 默认观测 35-D、动作 12-D [-1, 1]、DefaultControl、init 根 z 0.27、根高健康范围 (0.25, 1.0) | `unitreeGo2.py`（docstring 表；:117；:286–293） |
| Mjx Go2：timestep 0.002、n_substeps 5、solver 8/12、pyramidal cone、只留 4 对足–地接触 | `unitreeGo2_mjx.py:10–12,43–46` |
| PPO mimic conf：lr 1e-4、2048 env、200 steps、300M、clip 0.2、gamma 0.99、gae 0.95、hidden [512, 256] | `examples/training_examples/jax_rl_mimic/conf.yaml` |
| 我们：obs 128-D、训练 action 24-D、部署 [1,128]→[1,16]；PD 19.0/0.2 | `contract.py:2`、`collie_params.py:62–63`、`rl/AGENTS.md` |
| 我们：dr_mass alpha_range (-0.0527, 0.0477) → [0.9, 1.1]、名义 9.22313 kg；dr_com ±0.025/±0.025/±0.015；dr_friction (0.5, 2.0)；dr_pd_gains ±30%；dr_effort (0.9, 1.1)；dr_encoder ±0.02/±0.01；push ±0.5 每 5–10 s；impulse ±1.0 每 7–12 s | `tasks/mdp.py:525–600` |
| 我们：S1 lift max\|roll\| 1.04°、min support 67.8 mm、n_viol=0；终点高蹲姿 root_z 217.1 vs SIT 245.03 | B-1 TASK |
| 我们：刹车需求后踝 ~0.18 N·m（5.8×）、前踝 ~0.11 N·m（1.7×）；后踝 pitch ±108° + roll ±45° 双轴 | MEMORY.md（G1 轮 F-4/F-5，2026-11-23） |
| 回归基线：主集 145+1+4failed、bridge 34、parity 8、VSD `2d019d63…`/1971 B | MEMORY.md（2026-11-23） |

## 附录 C 未确认清单（本轮没读到、不编数）

1. `rand_conf` 各 DR 范围的默认值：`default.py` 只读键，默认值由调用方传入，本轮读到的文件里没有完整默认表；凡涉 LocoMuJoCo DR 范围只引附录 H.4 的五条。
2. loco-mujoco-real `training/` 与 `deployment/`、`lmj_real/` 下代码文件的内部实现：只读了 README 文字，未逐行读到。
3. 论文图 1–11 的图内数值（曲线读数、渲染细节）：HTML 文本版只读到图注。
4. 我们一侧：观测噪声是否已在 `mdp.py` / `collie_params.py` 之外实现（`adaptation.py` 等未 grep）；`validate_deploy.py` / `verify_bundle.py` 是否已覆盖部署路径的一部分；`delta_action.py` 的动作是否经关节限位归一化；`mdp_torch.py` 的 REWARD_ADAPTERS 细节。以上四项是 P1-6/P1-7/P1-8 的先决核实项，本报告不预断。
5. LocoMuJoCo 的 `smpl/` 重定向代码：只读结构与 `defaults.yaml` 引用，未逐行读完；附录 D 的机制描述以论文文字为准。
6. 仓库测试套件（>500 测试、93% 覆盖）未运行，以论文附录 F 为准。
7. `core/trajectory/handler.py` 未细读；§2.14 的轨迹格式结论只基于 `dataclasses.py` 的类/方法签名与 loco-mujoco-real README 的 `.lmj.npz` 说法。

## 附录 D 勘误

（空。报告正文冻结；后续订正只追加到本附录，不得就地改正文。）
<!-- LINES: 557 -->
<!-- SELF: 6fdb9ec6718730b98cf661b6ec9e68ed FULL: 65d1e67d0a6010b89bd971bd97958152 -->
