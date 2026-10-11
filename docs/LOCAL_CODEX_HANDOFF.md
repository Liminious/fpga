# FPGA 嵌赛 PCB 电路故障诊断项目：本地 Codex 接续说明

> 整理日期：2026-10-11（北京时间）。
> 本文件根据用户在云端对话中的最新决定整理，用于让本地 Codex 接续工作。
> **当前目标版本为 ICraft 3.36.1；3.39.0 / 3.39.1 是历史讨论版本。**
> 以下 Windows 路径来自用户操作记录。本地 Codex 必须检查它们是否仍然存在，不能把路径描述当作已经读取或执行的证据。

## 1. 给本地 Codex 的第一条任务

先读取本文件并检查实际文件，不要立刻训练或生成大量数据。

当前任务是：

1. 检查 ICraft 3.36.1 是否安装、独立 Python 3.8 环境是否存在，以及实际调用的版本。
2. 读取该版本随附文档和示例，确定正确的 ONNX 解析、转换、量化、编译配置。
3. 用现有 `joint_smoke.onnx` 验证最小联合模型的部署可行性。
4. 报告实际通过、失败与尚未运行的阶段；遇到不支持的算子，先提出并验证最小结构调整。
5. 在转换路线确认后，再安排悟净板端推理对比；正式数据生成和故障训练放在后面。

用户是新手。每次给一个能实际完成的小步骤，说明目的、软件、输入、修改文件、运行命令和验收标准。用户希望 Codex 在有本地权限时直接检查和执行可逆操作，不要反复要求用户人工列出你已经能读取的文件。

## 2. 项目背景与最终目标

用户参加 FPGA 嵌入式竞赛，参考 CircuitScout 开源项目，计划实现自动移动探针、测量 PCB 并诊断电气故障的系统。

CircuitScout 可参考的主要流程是：

```text
EDA 中选择信号 → 找到对应焊盘 → 获取坐标 → 移动探针接触
```

公开资料没有证明它已有完整的自动电气故障诊断、测量前端或 FPGA/NPU 实现。不能把参考项目的功能当作本项目已完成的功能。

现场待测电路不是固定的一张 RC 电路。用户希望上传：

- 正常设计网表。
- 元件标称值与允许容差。
- 支持器件所需的模型。
- 可用激励点、测量点。
- 实体测点坐标及机械定位所需信息。

目标闭环：

```text
正常设计
  ↓
悟净 CPU 解析电路结构、规划第一项测试
  ↓
机械定位 → 7020 激励与 ADC 采集
  ↓
正常电路结构 + 标称参数 + 测试动作 + 已有测量历史
  ↓
悟净 NPU 给出当前电路中的故障候选
  ↓
CPU 选择有区分能力的下一项测试
  ↓
增加测量历史、更新候选
  ↓
证据足够时给出结果，否则保留候选或报告无法唯一定位
```

期望研究：开路、短路、电阻错值/偏移、电容或 RC 响应异常、供电异常。阻值估计可由 CPU 根据模型与多次测量拟合，再计算偏移比例；不要求神经网络独自完成参数估计。

先限定小规模低压 R/C 网络，单故障是待正式定义的第一版建模范围，不承诺任意未知 PCB、多故障或有源器件通用诊断。

## 3. 硬件分工与最新采集方案

- **FMQL30TAI 悟净板**：CPU 做解析、测试规划、历史管理、参数拟合和网页服务；NPU 参与实际诊断推理。比赛要求实际使用 NPU。
- **另一块 Zynq 7020**：控制测试激励、外部 ADC 采集，并通过 TCP 网口向悟净传数据。
- **机械机构**：根据坐标定位与接触。控制器归属、完整到位/接触反馈能力仍需检查，不应自行假设。
- **当前明确方案**：一个 DDS 激励端、一个 ADC 采集通道；一支移动激励探针、一支移动采集探针。
- 用户说队友已经完成采集端，但本对话没有实际验证硬件性能或收到真实采集文件。

第一版测试方式：固定激励探针在 VIN，采集探针先测 VIN，之后移动到 OUT，再重复激励测 OUT。两次均相对于固定参考节点（样例为 `0`）测电压。

```text
测试 1：激励 VIN 对地，ADC0 测 VIN 对地
恢复初始状态并重新定位
测试 2：重复激励 VIN 对地，ADC0 测 OUT 对地
```

**必须保留为两次顺序测试，不能伪装成同步双通道采集。** 第一次的 VIN 波形不是第二次的实时激励监测。

尚需确认：ADC/DAC/DDS 型号、模拟输出实现、采样率、量程、输入/输出阻抗、实际激励稳定性、接地/供电方式、时钟与触发、真实测点坐标、电流测量能力。

## 4. 时序和恢复初始状态

- DDS 与 ADC 最好来自共同参考时钟，频率不必相同，但需要稳定、已知的关系。
- 每次测试需要明确激励事件与采样索引的对应关系；共有时钟不等于自动完成起点对齐。
- DAC、ADC、模拟前端延迟需要记录或校准。
- TCP 传输时间不是采样时间基准；时间戳/样本索引应在 7020 本地确定。
- 对顺序测量，应保持激励设置与初始条件可重复。相位测量还需要可重复的初始相位和触发关系。

电容会储存能量。第一遍虽然只测 VIN，OUT 的电容也已经被激励充电。移探针不会清零电容，因此第二遍需要恢复规定的初始状态。

当前理想样例：R1=R2=10 kΩ，C1=100 nF。

- VIN 保持在理想低阻 0 V：放电时间常数为 `(R1 || R2) C = 0.5 ms`。
- VIN 断开而呈高阻，电容主要通过 R2 放电：时间常数为 `R2 C = 1 ms`。
- 等待 5τ 残留约 0.67%；等待 7τ 约 0.09%。
- 此样例可先试 10 ms 放电等待，并检查激励前电压；这不是任意电路通用的等待保证。
- 10 mV 曾作为可讨论的残余电压试验阈值，**没有被实板验证或确定为最终要求**。
- 故障可能切断放电路径，不能只等待固定时长就认为复位成功。

## 5. 已完成与未完成的边界

### 用户此前在 Windows 完成的原型（历史记录，当前接续应检查原文件）

- ngspice 仿真两个电阻加一个电容。
- 正常、R1 错值、C1 异常三类共 600 组波形。
- 使用 PyTorch 和 RTX 3050 Ti 跑通固定电路分类训练。
- 简单 R/C/V 网表解析为 JSON，并在原 RC 电路及两级 RC 电路上验证。
- 历史脚本：`normal.cir`、`rc_ladder.cir`、`template.cir`、`faults.json`、`generate.py`、`train.py`、`netlist_to_graph.py`。

不能仅凭历史记录认为这些原脚本已在当前目录存在。旧模型只适用于固定电路与固定类别，不能承担最终目标。

### 本轮用户实际确认完成

1. 正常理想 RC 样例仿真成功：801 点、OUT 末值约 1.649999 V，解析响应检查通过。
2. 单 ADC 顺序测试样例已运行完成。
3. 固定输入处理已推进到能运行最小联合模型；本地 Codex仍应核对具体 `.npz` 文件。
4. 最小随机权重联合模型在 Windows 完成 PyTorch 前向、ONNX 导出及 ONNX Runtime 对比。
5. 改变连接、测点、OUT 波形、激励设置或测量历史，输出变化；屏蔽的第二次测量不影响输出；补齐元件位置输出为零。

用户报告的对比结果：

```text
PASS baseline: ONNX error=2.38e-07, change=0
PASS changed_topology: ONNX error=2.38e-07, change=0.00281
PASS changed_measurement_mapping: ONNX error=1.19e-07, change=0.000391
PASS changed_out_waveform: ONNX error=1.19e-07, change=9.27e-05
PASS first_test_only: ONNX error=5.96e-08, change=0.746
PASS changed_stimulus_setting: ONNX error=1.19e-07, change=0.00174
PASS: masked test cannot affect output; unused component slots are zero.
```

### 尚未确认完成

- ICraft 3.36.1 安装与实际版本检查。
- `icraft3361` Python 3.8 环境创建、Python API 安装。
- ICraft 对当前 ONNX 的解析、量化、编译。
- 悟净 NPU 实际推理及输出对比。
- 图结构故障诊断正式训练、主动测试规划器与参数拟合闭环。
- 真实 PCB 故障数据、机械/采集/诊断完整闭环验证。

**当前模型是随机权重部署试验，不是训练完成的诊断模型。**

## 6. 实际本地目录与训练环境

当前工作目录来自用户终端：

```text
E:\ngspice\rc_sequential_demo
```

主要文件/目录应检查：

```text
rc_sequential_demo\
├─ normal.cir
├─ circuit_graph.json
├─ run_sequential.py
├─ prepare_sequential_inputs.py
├─ model_smoke.py
├─ sample_rerun_sequential\
├─ inputs_two_tests.npz
├─ inputs_first_test.npz          名称以实际文件为准
├─ smoke_output\
│  ├─ joint_smoke.onnx
│  ├─ random_weights.pth
│  ├─ verification_inputs.npz
│  ├─ pytorch_output.npy
│  ├─ onnx_output.npy
│  └─ report.json
└─ ICraft资料\
   ├─ v3.36.1\
   └─ v3.39.0\                 用户计划清理，是否仍存在需检查
```

历史上解压路径曾是 `E:\ngspice\fpga-main\downloads\FPGA_RC_simulation_example\rc_measurement_demo`，不要误用为当前默认目录。

已实测的训练 Python：

```text
H:\anaconda_2024.06\envs\yolo\python.exe
Python 3.11（安装wheel信息及用户环境记录）
PyTorch 2.5.0+cu118
NumPy 2.3.5
CUDA True
NVIDIA GeForce RTX 3050 Ti Laptop GPU
ONNX 1.23.2
ONNX Runtime 1.31.0
```

用户的 `(base)` 没有 torch。另有 `H:\Anaconda\envs\torchcpu` 安装了 torch，但版本没有检查。不要默认 `python` 指向 yolo。

## 7. 顺序数据与固定输入规格

两次 ngspice 仿真使用同一正常设计、同一理想源与各自的新直流工作点。每次只导出一个电压通道，不是旧双通道数据拆列。

```text
sample_rerun_sequential\
├─ normal.cir
├─ circuit_graph.json
├─ measurement_history.json
├─ label.json
├─ validation.json
├─ test_01_vin\
│  ├─ wave.csv
│  ├─ test_action.json
│  ├─ measurement_record.json
│  ├─ simulation.cir
│  ├─ raw_wave.txt
│  └─ ngspice.log
└─ test_02_out\               同样结构，ADC0测OUT
```

CSV：`time_from_trigger_s, voltage_V`。
时间窗：各自激励上升沿前 1 ms 到后 7 ms，间隔 10 μs，801 点。负时间是激励前数据。
激励：理想电压源 VIN 对地，0→3.3 V，上升时间 1 μs，仿真开始后 1 ms 触发。

固定张量：

| 名称 | 形状 | 含义 |
|---|---|---|
| X | [1,16,8] | 最多8元件+8电气节点的属性 |
| A | [1,16,16] | 元件-节点关联、自身连接，CPU行归一化 |
| M | [1,16,1] | 有效顶点标记 |
| P | [1,2,2,16] | 两次测试；每次激励节点对和测量节点对，正端+1负端−1 |
| W | [1,2,1,128] | 两次测试，每次单通道，重采样128点 |
| S | [1,2,10] | 每次激励设置与时间信息 |
| H | [1,2] | 已完成测试标记，[1,0]或[1,1] |

全部为 float32。电压除以固定3.3；S中的时间除以1 ms。不能逐条波形按自己的最大值归一化。

X字段：R类型、C类型、电气节点、地节点、标称值对数、容差、容差已知标记、顶点有效标记。
标称值对数：电阻使用 log10(R/10000)，电容使用 log10(C/1e-7)。容差未知保留已知标记为0，不能当作真实零容差。
S字段：低/高电压、时间窗起点/终点、原采样间隔、相对触发的激励起点（本例0）、上升/下降时间、脉宽、周期。
未测位置补零必须结合H；补齐的顶点结合M。

这些尺寸仅用于最小试验。当前处理脚本与敏感性测试限定本RC样例，不是通用解析器或任意电路输入方案。

## 8. 最小联合模型究竟验证了什么

模型结构：

```text
X → Linear+ReLU → A矩阵聚合 → M屏蔽无效顶点
                              ↓
                  P提取激励/测量节点对特征
                              ↓
W → 小卷积+池化+Linear → 与位置特征和S拼接
                              ↓
                  每次测试特征 → H屏蔽 → 求和
                              ↓
                  与各元件图特征融合 → 演示输出
```

输出 `[1,8,3]`：八个元件位置各三个演示分数。没有绑定故障类别，没有节点对短路输出、供电端口输出或参数回归头。
权重随机，不计算准确率。改变输入后有响应只说明计算路径没有被忽略。

导出为固定尺寸 ONNX，opset13。PyTorch和ONNX Runtime对照容差 rtol=1e-4、atol=1e-5。
算子清单：

```text
MatMul 5; Add 4; Relu 6; Mul 4; Constant 15; Reshape 6;
Conv 1; AveragePool 1; Gemm 1; Concat 1; ReduceSum 1; Slice 2.
```

重点检查：A和P是运行时输入。工具支持普通固定权重矩阵乘，不代表支持这里的动态输入矩阵乘；还需验证具体shape和参数。
ONNX Runtime运行成功只证明电脑端导出一致，不证明ICraft或NPU兼容。

## 9. 已知可用的重跑命令

在PowerShell中，先进入实际项目目录：

```powershell
cd E:\ngspice\rc_sequential_demo
```

仿真（标准库脚本；必须选择新的输出目录）：

```powershell
python .\run_sequential.py --ngspice "E:\ngspice\Spice64\bin\ngspice.exe" --out sample_check_sequential
```

固定输入：

```powershell
& "H:\anaconda_2024.06\envs\yolo\python.exe" .\prepare_sequential_inputs.py --sample .\sample_check_sequential --out .\inputs_check.npz
```

最小模型：

```powershell
& "H:\anaconda_2024.06\envs\yolo\python.exe" .\model_smoke.py --inputs .\inputs_check.npz --out .\smoke_check
```

输出存在时脚本拒绝覆盖。没有新的问题时，不必重新跑全部流程；可直接使用已验证的smoke_output。

## 10. ICraft 3.36.1 文件与README的明确要求

资料目录：

```text
E:\ngspice\rc_sequential_demo\ICraft资料\v3.36.1\ICraft_V3.36.1安装包
```

已发现：

- Icraft_Setup_v3.36.1.exe
- IcraftMdz_ThirdParty_v0.2.exe
- CustomOp_Setup_v3.36.1.exe
- icraft-3.36.1-cp38-none-win_amd64.whl
- Icraft_3.36.1_amd64.deb / Icraft_3.36.1_onchip.deb
- readme.md / tags_3.36.1.txt

上传的README明确：

1. Windows主程序双击安装，安装目录固定 `C:\Icraft`，不可修改。
2. 安装目录中有 `uninst.exe`。
3. CustomOp可选，必须先安装主程序。
4. Python API可选，仅支持Python3.8，必须先安装主程序。
5. 安装后 `icraft docs` 打开完整文档。

README没有明确说明不同CLI版本能否共存，也没有给出这里的模型转换命令。不要凭空编造安装子目录、CLI标志或JSON字段。
README对Linux包命名与实际下载清单有差异，板端安装要继续核实，不能简单把onchip包当作已确认的对应项。

用户最新选择：清理3.39.0下载资料，并为3.36.1创建独立Python3.8环境。**目前未收到这些操作已完成的结果。**

下载文件夹与已安装程序要区别处理。删除下载资料不会卸载已安装CLI。
不要手动删除整个C:\Icraft；如果确需卸载已安装版本，先识别正确卸载器和用户配置，保留可恢复资料。

## 11. 本地Codex现在应执行的步骤

### 步骤A：只读检查现状

- 检查项目与ONNX产物是否存在，读取report.json。
- 检查C:\Icraft实际目录结构。
- 检查命令解析路径，避免PATH调用旧版。
- 检查Conda环境列表、icraft3361是否已经存在及Python版本。
- 不打印许可证、密钥、密码或整个环境变量值。

可用检查命令：

```powershell
Test-Path "E:\ngspice\rc_sequential_demo\smoke_output\joint_smoke.onnx"
Get-ChildItem "C:\Icraft" -Directory | Select-Object -ExpandProperty FullName
Get-Command icraft -All -ErrorAction SilentlyContinue | Select-Object -ExpandProperty Source
conda env list
```

### 步骤B：按实际缺项安装

如果尚未安装主程序，按README运行Icraft_Setup_v3.36.1.exe。
如果安装器要求卸载/覆盖已有版本，先确定受影响范围；不盲目覆盖用户配置。
主程序安装与Python环境创建是两件事，Conda环境隔离不会隔离Windows全局CLI。

若独立环境不存在，创建：

```powershell
conda create -n icraft3361 python=3.8 pip -y
conda activate icraft3361
python -c "import sys; print(sys.executable); print(sys.version)"
```

激活成功后，按README安装API：

```powershell
python -m pip install "E:\ngspice\rc_sequential_demo\ICraft资料\v3.36.1\ICraft_V3.36.1安装包\icraft-3.36.1-cp38-none-win_amd64.whl"
python -m pip show icraft
```

若无法激活，可使用 `conda run -n icraft3361 python ...`，先确认调用正确Conda安装。
不要在icraft3361里重复安装PyTorch。yolo负责导出，icraft3361负责API，两者用ONNX文件交接。
不要为了安装成功关闭TLS验证、签名验证或修改校验值。

### 步骤C：读取本机完整文档，做最小解析

确认实际使用3.36.1，再执行官方已知命令 `icraft docs`。
本地Codex应直接读取随安装包落地的文档、配置和示例，而不是仅让用户打开浏览器后反复截图。
根据官方示例创建独立项目配置，填入joint_smoke.onnx和7个输入的固定shape。
记录实际命令、工作目录、日志、退出码及产物。先检查解析/转换支持性，再做代表性校准和量化。

若失败：区分ONNX版本/shape/算子参数/动态MatMul/平台依赖/目标配置等原因。
只修改新建的模型试验文件，保留原模型和验证产物。修改后重跑PyTorch与ONNX对比及输入敏感性检查。
如果动态图计算受限，可研究CPU承担部分聚合、NPU承担学习变换与评分，但必须如实说明分工并核对比赛要求。

### 步骤D：悟净NPU验证

在模型编译通过后再推进：

- 向队友确认悟净系统、运行时、驱动、位流和可用推理示例。
- 确认CLI产物与板端运行时版本兼容，不默认3.36.1主机包能直接配任意系统。
- 检查网络/SSH连接；网口传文件只是部署传输，不等于运行模型。
- 依据实际运行示例传入相同固定输入，在NPU执行并比较输出和耗时。
- 不擅自刷BOOT、替换系统或升级驱动；板端变更需结合队友现有配置。
- 量化后的误差阈值不能直接照搬浮点ONNX对比阈值，需制定合适标准。

## 12. 资料位置与参考实现

官方目录：

- https://university-program.fdwmy.com/data/2026%E5%B5%8C%E8%B5%9B/04_ICraft/
- https://university-program.fdwmy.com/data/2026%E5%B5%8C%E8%B5%9B/04_ICraft/v3.39.0/

第二个是历史3.39.0链接，当前不要用它替代3.36.1资料。3.36.1下载目录应以本地实际记录和网页链接为准。

本地教程TXT实际上是网页链接，不是教程正文：

- 开发教程：https://blog.csdn.net/qq_36840004/article/details/158010956
- PLIN教程：https://blog.csdn.net/qq_36840004/article/details/166363151

这些网页正文此前未在云端读取。不得把未读内容作为已验证依据。

本地3.36.1参考实现包含：

- FMQL30TAI参考实现 / 悟净开发板（完整版&LITE版均适用）。
- 完整版与LITE版各自的BOOT位流ZIP，不能混用管脚约束。
- 单路PLIN+pHDMI参考runtimeapp：fpai_demo_package_26040701.zip、对应txt和md5。
- 单路PLIN+VPU编码参考runtimeapp：fpai_demo_package_26040702.zip、对应txt和md5。

具体选择、启动方式、兼容性需要阅读包内文件；本项目网页交互不意味着必须先启用HDMI或视频编码。

## 13. 正式训练前必须坚持的原则

- 模型输入只含正常设计、标称值、允许的测试动作和已取得的测量。
- 故障真实参数、故障后的真实网表、标签和带故障名称的文件路径不进入输入。
- 同一诊断过程中的真实故障和参数保持不变。
- 正常容差、测量误差、激励漂移与接触误差需要建模；范围应尽可能来自实测。
- 按拓扑分组划分训练/验证/测试；等价拓扑、参数变体和历史切片不能跨组泄漏。
- 允许证据不足、候选集合和无法唯一定位；模型高分不等于唯一证据。
- 单ADC先后测VIN/OUT的数据不能默认同步；相位诊断需要额外验证时序和重复性。
- 旧600组固定RC分类数据保留作对照，不继续盲目扩大。
- 优先验证部署，再设计多拓扑数据和正式网络输出。

测试激励策略仍处于规划：先阶跃/直流，再增加少量离散正弦频点，暂不全面连续扫频。
当前样例τ=0.5ms、截止频率约318Hz，30/300/3000Hz仅是该样例的候选频点，尚未生成或实板验证。
扫频不能自动消除参数不可辨识：当前理想电路仅凭增益和时间常数，R1/R2/C全未知时可能不能唯一估计。

## 14. 云端已交付资料

仓库：https://github.com/Liminious/fpga

已有ZIP：

- downloads/FPGA_RC_simulation_example.zip：早期双波形理想样例，只作历史对照。
- downloads/FPGA_RC_sequential_single_ADC.zip：当前单ADC两次顺序测量及固定输入。
- downloads/FPGA_RC_joint_model_smoke.zip：最小联合网络、输入、ONNX和电脑端检查报告。

云端文件链接在用户界面无法下载，因此此前改由GitHub传文件。
GitHub文件页面可用Download raw file下载。
本地若已有文件，优先检查并复用，不要重复解压覆盖用户改动。

## 15. 接续时的完成标准

当前阶段完成应有证据：

1. 实际使用的CLI/API版本和Python路径已确认。
2. 当前模型被ICraft成功解析/转换，7个运行时输入及图结构信息未被错误固化。
3. 量化、编译的目标明确，日志与产物可复现。
4. 对应板端运行时加载成功，并有实际NPU执行及对比记录。

若卡在不支持的算子、缺失许可证、板端条件或安装兼容性，明确指出具体阻塞并继续完成不依赖它的工作。
只完成安装、ONNX导出、CPU运行或文件拷贝，都不能声称NPU部署完成。
