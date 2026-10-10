# S288 单舵机实验 → BAM 参数辨识 → Microduck 仿真接入

日期：2026-10-02。对象：5 个 S288、宇树 USB–单总线转接板、标称满充 12.6V / 2550mAh 电池；开发主控使用已有树莓派 5，也可以先用电脑。

## 0. 先读边界

本包是本次答复编写的**参考实现**，不是宇树或 Rhoban 官方发布的 S288 适配包。

已完成：Python 语法检查；9 项本地单元测试，覆盖负角度封包、CRC、反馈字段、两编码器差异拦截、数据处理与 M1 延迟逻辑。

未完成：真实 S288 / 宇树转接板 / 电池测试、在完整 BAM 环境中的真实数据拟合、MuJoCo 真运行、GPU mjlab 接入、Microduck 实机验证。工作环境无法从 GitHub 下载并安装完整依赖，因此**不能把这些脚本称为已联调工具**。没有提供任何冒充实测的 S288 摩擦或惯量结果。

硬件脚本默认预览，不开串口。`--execute` 才操作硬件，还须 `--single-servo-fixture-confirmed`。脚本的所有软件保护只能降低风险，不能替代物理断电、保险和人工监护。

先只接一颗舵机、无负载验证。模式0是官方定义的“锁定”，**不是承诺自由卸力**；状态读取也会发送模式0指令，不能对已承重的机器人随意使用。结束后物理断开舵机电源。没有实现自由落下实验。

## 1. 你最终要获得什么

```text
真实 S288 + 已知摆杆/配重
    ↓ capture.py（你执行实验）
data/raw/*.json：原始命令、反馈、时间戳
    ↓ prepare.py（清洗与固定积分网格）
data/train/*.json、data/validation/*.json
    ↓ s288_model.py + BAM fit（CPU离线优化）
results/run_001/m1.json：执行器参数
    ↓ 独立验证、MuJoCo / GPU 执行器适配
Microduck-S288 仿真
    ↓ PPO重新训练、规范导出
policy.onnx → 树莓派运行 → S288
```

**BAM 参数主要给训练端仿真使用，不是烧进舵机的参数，不是 ONNX。** 真实运行另存：关节ID/方向/零位、控制增益、限位、控制周期、安全参数。

## 2. 接线与仍需准备的物品

```text
电脑/Pi（自己正常供电）── USB ── 宇树转接板 ── S288 #0
                                      ↑
                     电池 ─ 保险/可断电连接 ─ 电源入口
```

第一阶段无需 IMU、15 个舵机、STM32 或整机结构。还需要：固定舵机的支架、直且刚性的轻摆杆、可靠固定的配重、尺子/卡尺、电子秤、万用表、正确极性的电源转接线。最好有能观察电流与母线瞬态的仪器。做台架时不要用手捏住舵机或摆杆。

电源注意：

- S288 官方输入上限 12.6V；请测量实际满电输出。2550mAh 是容量，不能推出持续电流或保护板阈值。
- 首先验证一颗，再考虑多颗；本实验不需要五颗一起运动。
- 电池、BMS、接口、导线、转接板都要满足电流要求。断电保险靠近电池正极，规格依最弱路径和实际电流确定。
- 树莓派不要直接接 12.6V；台架阶段独立正常供电，后期再单电池分路降压。
- 满电12.6V已在输入上界；不要立即做快速制动/抬起释放实验。了解电池/BMS能否吸收回灌及驱动过压处理。普通电源也未必能吸收回灌。
- 按电池厂商规定的停止电压停止，不能把舵机最低工作电压当电池放电下限。不要边充电边实验。

这些是本次工程建议；单舵机测试也不能保证任意未知电池与线束都安全。

## 3. 台架如何搭

固定舵机壳体，使输出轴水平、摆杆在竖直平面运动。杆正下方定义 q=0。运动空间加防护，配重不能甩脱。新增轴承会带来额外摩擦；第一版尽量结构简单，避免夹紧或偏心，安装摩擦不能全部误归为电机特性。

建议起点（不是官方测试标准）：两种有效长度 60mm / 100mm，两种配重 20g / 50g。先最短最轻，小幅运动正常后才增加。杆最好质量均匀；配重质心到轴心距离作为 L。

均匀细杆 + 末端点质量的模型：

- J_load = m*L² + m_arm*L²/3
- gravity(q) = −g*L*(m + m_arm/2)*sin(q)

例：50g、100mm 的配重，水平时自身重力力矩约0.049N·m，另加杆重和加减速力矩。这不是负载安全认证。

`mass` 是末端配重；`arm_mass` 是杆质量；**不能把舵机壳体质量填成配重**。大块配重、非均匀杆、随轴转动的舵盘/螺丝质量不可忽略时，使用 CAD 或测量计算完整质心和惯量并扩展 Testbench；不要硬塞进默认均匀杆公式。

## 4. 五颗舵机怎样分工

给实体贴 S0～S4 标签，记录固件和总线ID。逐颗接线设置唯一 ID，先不要将出厂同ID的设备一起挂上。

- S0：做完整第一轮辨识。
- S1～S4：用相同轻载轨迹做重复测试，检验个体差异。
- 不要把五颗串起来共同驱动一个摆杆。
- 数据目录分别留存每一颗。先拟合S0，不把全部个体混在一起拟出一个“平均舵机”。

## 5. 安装环境

BAM 的离线识别使用 CPU，不要求先有 CUDA；Microduck 的 MuJoCo Warp 并行RL训练是后续GPU任务。以下给 Linux/Ubuntu 命令；Windows原生或Mac的串口路径、Python可执行文件位置需要对应改变。WSL需要USB透传，首次硬件采集不优先用WSL。

在本包目录执行：

```bash
mkdir -p third_party
git clone https://github.com/Rhoban/bam.git third_party/bam
cd third_party/bam
uv python install 3.12
uv sync --extra identification
uv pip install pyserial
git rev-parse HEAD > ../../BAM_COMMIT.txt
cd ../..
PY="$PWD/third_party/bam/.venv/bin/python"
"$PY" -c "import bam; print(bam.__file__)"
"$PY" -m unittest discover -s tests -v
```

以上 `uv sync --extra identification` 是仓库的安装方式。本环境未实际完成此安装；如遇依赖错误，保留日志、Python版本和commit排查，不要随意全量升级与训练仓库混装。采集端仅需Python+pyserial；拟合端需要BAM及辨识依赖。为了减少移动文件，第一版可用同一台电脑。

每次实验记录BAM commit、宇树参考代码commit、自己的脚本版本；不要一边采数据一边无记录升级。

## 6. 先核对协议与两路编码器

参考来源是宇树的 `stm32/firmware/App/protocol.h/.c` 和 `python/servo_demo.py`。本包没有照搬SDK交互主循环，按C结构体重新封装了小范围通信层。

本次核对发现官方不同文本存在不一致：Python例程把有符号目标位置用 `struct.pack('<hhIhh',...)` 中的大写I编码；C头文件则明确是int32。负角度会受影响。本包用 `<hhihh`。反馈C头文件的电压是uint8、绕组温度是uint8；本包采用 `<bBBhhiIHBB` 共19字节。不要关闭CRC来掩盖错误。

每次命令20字节、反馈26字节，ID0～14，6Mbps。CRC按官方每4字节逆序计算，不能直接用zlib默认CRC。首次务必与官方电机助手/参考程序交叉核对状态。

状态查询（模式0锁定，不是静默读）：

```bash
"$PY" capture.py status --port /dev/ttyUSB0 --id 0
# 上一行只预览。确认单颗固定台架后才实际执行：
"$PY" capture.py status --port /dev/ttyUSB0 --id 0 \
  --execute --single-servo-fixture-confirmed
```

把摆杆人工对齐竖直向下，记录输出：

- `q_rotor_equiv`：转子编码器除减速比后的输出侧等效角，记为 ZERO_ROTOR。
- `q_output`：输出端编码器角，记为 ZERO_OUTPUT。

这两个数一般不相同，不能互换。重新上电后先检查，不要假定绝对转子多圈值永远连续。不要在低增益未启用时强行掰齿轮；按厂家允许的校准方式定位。

实际控制单位统一为rad、rad/s、关节侧等效N·m及相应PD增益；上层代码不再额外乘减速比。底层封包才换算。Kp、Kd涉及减速比平方，而且协议有量化台阶；本包保存实际量化后的增益。

同时观察 (q_output−ZERO_OUTPUT) 和 (q_rotor_equiv−ZERO_ROTOR)。零位对齐后仍存在的差异不全是齿隙，也可能有柔性、采样不同步和标定误差。

## 7. 首次小幅记录

先无负载验证状态，再接最轻摆杆。增益没有可以保证安全通用的固定值；以下0.2与0.005仅是本次给出的低幅起步示例，不是宇树标定值。先确认单位与方向，必要时在官方工具下验证控制行为。

本包要求先填真实测量量；下面带 `替换` 的变量不能直接复制运行：

```bash
ZERO_ROTOR=替换为竖直时q_rotor_equiv
ZERO_OUTPUT=替换为竖直时q_output
ARM_MASS=替换为实测摆杆质量_kg
MIN_V=替换为电池厂家要求的停止电压

"$PY" capture.py record --port /dev/ttyUSB0 --id 0 \
  --zero-rotor "$ZERO_ROTOR" --zero-output "$ZERO_OUTPUT" \
  --kp 0.2 --kd 0.005 \
  --mass 0.020 --arm-mass "$ARM_MASS" --length 0.060 \
  --amplitude-deg 3 --frequency 0.2 --hz 100 --duration 12 \
  --min-voltage "$MIN_V" --pd-budget 0.05 \
  --firmware "填写版本" --out data/raw/S0_check_rep1.json
```

这行是预览。核对后增加 `--execute --single-servo-fixture-confirmed`。`0.020/0.060`也必须与实际配重及长度相符；不能照抄。

程序先保持零位1s、中间运动12s、最后保持1s。发生CRC/ID/故障/温度/越界/超时错误会中止并标记 `complete=false`，不得拿坏日志去拟合。

**`pd-budget`只是利用上一反馈估算PD并在超限时停止，不是电机内部的硬力矩限制。tau_ff=0也不限制PD力矩。** 旧反馈、Linux调度和内部控制仍有延迟，物理保护不可省略。程序例示50°C壳温/60°C绕组温度停止，为保守初始实验阈值，不是厂家绝对安全限值。

舵机不动时不要直接把增益加十倍。先检查模式、单位、供电、关节方向、机械卡滞，再小步调整。轻载数据无法识别最大力矩，不需要故意堵转。

采完先检查，不用等拟合：

```bash
"$PY" inspect_log.py data/raw/S0_check_rep1.json --out reports/S0_check_rep1
```

它会导出原始CSV、两路编码器/目标角对比图、采样时间图以及p50/p95/p99统计。只有角度真实运动、反馈有更新、时间可靠，才扩展实验矩阵；原始曲线几乎不动的数据不能当成丰富辨识数据。

## 8. 正式第一轮实验矩阵

先找到能稳定、小幅跟踪的一组增益（没有明显持续振荡、不靠饱和顶住）。每条正式记录中固定增益，随后可另外采一组已验证增益作为外部检验。

第一輪建议：

- 2杆长：60/100mm。
- 2配重：20/50g。
- 3平滑运动：例如10°@0.2Hz、20°@0.2Hz、10°@0.5Hz；必须逐步验证再进入。
- 每种3次，每次12s正式片段。

合计2×2×3×3=36条S0记录。每个组合rep1/rep2用于拟合，共24条；rep3用于验证，共12条。这是起步数据量建议，不是收敛或可辨识保证。

之后S1～S4各采相同轻载轨迹3次，不参与S0拟合，用于跨个体验证。若差异大，先排除装配、固件、温度、电压差异，再决定逐颗参数或域随机化。

不要直接复用BAM示例的大幅±90°和较重配重。`lift_and_drop`暂不做，模式0不是自由释放证明。

实验温度、电压尽量相近，记录每条实验前后万用表读数。S288协议电压反馈量化较粗，不能用其独立测毫秒压降或毫欧内阻。不同电压的数据先分组比较，V0并没有电压电流环模型。

## 9. 日志里有哪些数据

原始JSON保存：

- 发送/接收时间，另保存二者中点作为粗略采样时间估计；它不等于精确传感器采样时刻。
- 原计划目标与实际量化后发送目标。
- 两路角度、转子等效速度、转子等效力矩估计、电压、两类温度、错误、原始反馈hex。
- 实际量化后的Kp/Kd、杆重、配重、长度、固件、舵机ID。

反馈中的力矩是估计量，不能直接当作经过传动摩擦后的净轴端力矩；2550mAh电池电流也不是FOC相电流。本V0拟合用位置轨迹作为目标，不用反馈力矩当真值。

记录硬件周期的中位数、p95/p99、最长缺口。先100Hz稳定，再争取200Hz；6Mbps不能保证内部传感器刷新或Linux循环准时。计划控制周期和实际时间必须都保存。

## 10. 处理数据

```bash
"$PY" prepare.py data/raw/S0_check_rep1.json data/train/S0_check_rep1.json \
  --dt 0.005 --start 1 --seconds 12
```

处理器：拒绝中止记录/故障/非单调时间/较长丢样；反馈线性插值到固定积分网格；实际发送目标采用零阶保持；输出统一BAM字段。

`max-encoder-difference-deg`默认0.5°是本包对单自由度V0假设的**检查阈值**，不是S288厂家精度指标。超过后先检查，不要只增大阈值强行拟合。

100Hz反馈插值成5ms网格只是仿真积分整理，不能声称采到了200Hz新信息。也不要把10ms控制跳变先线性平滑，再误判为真实舵机延迟。

官方 `bam.process` 是通用处理入口，但所核对实现对多数数字字段统一线性插值；本包单独处理命令，避免把保持目标变成斜坡。不要先后重复重采样。

所有用于一次BAM批处理的轨迹要同dt、同长度。所核对的Logs.make_batch会按最短轨迹截断，本包拟合入口会拒绝不一致长度。

## 11. 模型与待辨识参数

`s288_model.py` 定义新的 `s288_v0`：

```text
tau_drive = clip(torque_gain * (Kp*(delayed_target-q)-Kd*dq), ±max_torque)
(J_load + armature)*qdd = tau_drive + gravity(q) - friction
```

V0假设输出角≈转子角/N；固定增益、tau_ff=dq_des=0；忽略高速下的电压限幅、电流环动态和柔性。它是小幅、低速、固定控制模式下的等效模型，不是S288完整数字孪生。

第一轮优化 armature、friction_base、friction_viscous、command_delay；物理零位已对准时把q_offset固定为0。torque_gain先固定1，max_torque暂固定规格峰值0.6N·m且**不得把它当成连续额定能力或本次拟合结果**。初始数值/搜索范围只是数值设置，出现贴边要检查模型与实验，不要无限放宽。

由于控制阻尼和黏性摩擦可能互相补偿，第一轮不同时自由拟合两者。若要校准有效增益，另用独立实验，再把结论纳入模型。

M1先合格，再试M2等。M1/M2不能解释真实齿隙；更高M5/M6也不是自动齿隙模型。真实台架夹紧、杆柔性和附加支承摩擦可能污染拟合。

## 12. 用BAM拟合与验证

`bam_entry.py`在当前Python进程内注册 `s288_v0`，然后调用真正的 `bam.fit`。不用往系统库乱改文件。直接在干净BAM中执行 `--actuator s288`仍然不成立。

```bash
"$PY" bam_entry.py fit --actuator s288_v0 --model m1 \
  --logdir data/train --output results/run_001/m1.json \
  --trials 3000 --workers 1 --set '{"q_offset": 0.0}'
```

3000次只是首轮工具验证，不是次数越多就能拟合真值。优化器在CPU上反复模拟，不会再驱动实物。真实数据好、参数可辨识后，再增加评估次数、做多随机起点。

验证：

```bash
"$PY" evaluate.py --params results/run_001/m1.json \
  --logdir data/validation --out reports/run_001_validation
```

输出逐条MAE/RMSE/最大误差、预测/实测/目标角CSV以及PNG。这里显式采用 `simulate_control=True`、不定期重置真实状态，防止只是用实测状态喂给控制器而高估预测能力。还应对比简单PD基线，报告新负载/新频率/新舵机的误差。

发现 `--eval` 在所核对上游fit.py中仍有参数路径硬编码，故本包不用它。也不使用只支持整数选值的`--validation_kp`来划分小数关节增益，而是分目录保存验证记录。

频繁reset可用于局部诊断，但不能把有重置的误差与自由长回放误差混作同一指标。

重要限制：本环境未安装完整BAM/cmaes，未真实执行以上拟合命令；这些接口依据已读取源码编写。运行时仍需对BAM版本和依赖做一次集成检查。

## 13. 你需要保存的成果

```text
results/run_001/m1.json        参数，不是策略
reports/run_001_validation/   独立验证曲线和误差
BAM_COMMIT.txt               所用工具版本
experiment_log.csv           每次配重/增益/电压/温度/异常记录
characterization.json         人工整理的参数来源和有效工况说明
```

参数来源区分：datasheet（规格）、readback（寄存器/反馈）、measured（独立实测）、fitted（轨迹拟合）、assumption（假设）。任何没被实验激励的饱和/热/高转速性能都不能标为已辨识。

## 14. 得到参数后放哪里、怎么用

### 14.1 单关节与MuJoCo CPU回放

M1的物理映射：

- `armature` → 对应关节的dof_armature（不是舵机壳体惯性张量）。
- `friction_base` → dof_frictionloss。
- `friction_viscous` → dof_damping。
- `command_delay` → 执行器目标缓冲；单位是秒，再按物理步长换算。
- Kp/Kd/torque_gain/max_torque → 执行器控制律。
- `q_offset` → 仅台架零位误差，不直接覆盖整机每关节零位。

本包 `m1_runtime.py` 是CPU接入参考。应使用输出端 `gear=1` 的 motor actuator，在每个物理步调用，不要额外乘288.35或再叠一层XML position PD。

控制Kd已在主动PD力矩中计算，不要又把同一Kd加到被动damping里重复计入。高阶BAM摩擦预算随力矩/负载变化，不能只复制其中两个数字到静态XML就声称用了M5。

### 14.2 Microduck训练仓库

建议自己的fork中保存：

```text
microduck_rl/
├── actuator_params/s288/m1.json                   # 自定义推荐目录
├── src/mjlab_microduck/actuator/s288_actuator.py   # 需要新增的GPU执行器实现
├── src/mjlab_microduck/robot/microduck_constants.py
└── src/mjlab_microduck/robot/microduck/robot_walk.xml
```

以上前两个路径是建议新增，**不是当前开源库已经存在的S288功能**。`microduck_constants.py`要选择S288执行器配置，而非保留飞特类只改名字。机器人XML另外更新连杆质量、质心、惯量、关节轴线、限位和碰撞；执行器JSON不会自动更新CAD。

也可以把适配模块与参数放入JoyandAI的`vendor/bam`，但其快照不一定与独立Rhoban主线一致。不要整目录覆盖，需比对API；启动时打印`bam.__file__`确认加载的是哪个版本。

**当前Rhoban的 `bam.mjlab.BamActuator` 会拒绝非DCMotorActuator子类。** 本包V0继承基础Actuator，因此不能把结果JSON直接交给该GPU类。还需要实现力矩型S288的向量化执行器或改造该接口，并保证与CPU离线模型一致。不要为了通过类型检查而假装电压控制模型。

本包没有提供已联调GPU适配器。推荐先M1，GPU版只需固定摩擦+同样PD/限幅/延迟，但仍要做1个关节、1个环境、相同目标轨迹的CPU/GPU对照，再增加并行数。没有验证这步，不开始长时间PPO训练。

### 14.3 实机部署

PPO训练完成，经项目exporter生成包含正确归一化的ONNX。树莓派上部署ONNX、joint_map和runtime控制配置，不需要把BAM摩擦JSON烧进S288。

硬件侧的增益、方向、单位、控制频率、读取策略、零位要与训练约定一致。IMU提供机体角速度/重力方向等观测。驱动目标变成舵机协议时再做底层换算。

不要把拟合到的command_delay再当成真实控制循环的sleep：它是描述真实延迟的仿真量，硬件侧应实测并尽量降低延迟，不是人为添加延迟。

## 15. 按验收推进

1. 单颗状态稳定、两路角度方向与单位核对；物理断电和错误停止可用。
2. 小幅无负载/轻载记录，CSV/JSON中的目标与反馈可信。
3. 36条S0矩阵记录＋独立验证，排除坏记录与参数贴边。
4. S1～S4交叉验证，决定一致性范围。
5. 单关节MuJoCo CPU验证，再GPU执行器对照。
6. Microduck整机动力学更新、重新训练、ONNX受保护上机。

第一天只以“状态核对＋一条可信小幅记录”为目标，不以生成最终参数文件为目标。

## 资料依据

- [Rhoban BAM](https://github.com/Rhoban/bam)
- [BAM台架](https://bam.readthedocs.io/en/latest/identification/setup.html)
- [BAM数据采集](https://bam.readthedocs.io/en/latest/identification/acquisition.html)
- [BAM拟合](https://bam.readthedocs.io/en/latest/identification/fitting.html)
- [Actuator基础接口](https://github.com/Rhoban/bam/blob/main/bam/actuator.py)
- [摩擦模型](https://github.com/Rhoban/bam/blob/main/bam/model.py)
- [fit实现](https://github.com/Rhoban/bam/blob/main/bam/fit.py)
- [数据批处理](https://github.com/Rhoban/bam/blob/main/bam/logs.py)
- [MuJoCo CPU接入](https://github.com/Rhoban/bam/blob/main/bam/mujoco.py)
- [mjlab接入](https://github.com/Rhoban/bam/blob/main/bam/mjlab.py)
- [宇树S288参数](https://www.unitree.com/DigitalServo)
- [宇树C结构体](https://github.com/unitreerobotics/digital_servo/blob/main/stm32/firmware/App/protocol.h)
- [宇树换算实现](https://github.com/unitreerobotics/digital_servo/blob/main/stm32/firmware/App/protocol.c)
- [宇树Python例程](https://github.com/unitreerobotics/digital_servo/blob/main/python/servo_demo.py)
- [Microduck训练项目](https://github.com/pollen-robotics/microduck_rl)
- [HD1910参数对照](https://github.com/JoyandAI/microduck_rl/tree/develop/vendor/bam/bam/params/hd1910)
- [MuJoCo XML参数](https://mujoco.readthedocs.io/en/stable/XMLreference.html)

链接内容会更新；以记录的commit与自己的测量数据为准。本包提出的载荷、轨迹、增益、阈值与文件组织为本次建议，不是上述厂商的认证实验流程。
