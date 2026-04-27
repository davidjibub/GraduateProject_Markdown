'''
26.4.24 分析GliomaSolver模式，判断如何接入毕设，详见David
'''

| 模式                 | 本质                                                      | 能做什么                        | 更像什么           |
| ------------------ | ------------------------------------------------------- | --------------------------- | -------------- |
| `TumorGrowth`      | **forward reaction-diffusion simulation**               | 在给定脑组织环境和肿瘤参数下，模拟肿瘤随时间生长    | “参数驱动的肿瘤生长演化”  |
| `BrainDeformation` | **forward tumor growth + mass effect / ICP simulation** | 在肿瘤生长基础上，再模拟脑组织位移、占位效应、颅内压  | “带力学挤压的肿瘤生长演化” |
| `PatientInference` | **Bayesian calibration / patient-specific inference**   | 利用患者影像数据反推更符合患者的肿瘤参数与细胞密度分布 | “基于患者观测的个体化推断” |

功能
| 模式                 | 主要功能                        | 是否考虑力学挤压 | 是否利用患者当前肿瘤观测做校准 |
| ------------------ | --------------------------- | -------: | --------------: |
| `TumorGrowth`      | 肿瘤生长 forward 模拟             |        否 |               否 |
| `BrainDeformation` | 肿瘤生长 + 脑组织形变/ICP forward 模拟 |        是 |               否 |
| `PatientInference` | 患者特异参数校准与肿瘤细胞密度推断           |   不是其主目标 |               是 |
输入
| 模式                 | 解剖输入                                  | 肿瘤输入                              | 其他要求                 |
| ------------------ | ------------------------------------- | --------------------------------- | -------------------- |
| `TumorGrowth`      | `WM.dat / GM.dat / CSF.dat`           | `icx, icy, icz` + `Dw, rho, Tend` | `.nii` 需先转 `.dat`    |
| `BrainDeformation` | `WM.dat / GM.dat / CSF.dat / PFF.dat` | `icx, icy, icz` + 生长参数            | 还需力学参数               |
| `PatientInference` | `WM.nii.gz / GM.nii.gz / CSF.nii.gz`  | 肿瘤相关 MRI/PET 及其分割                 | 标准流程需要 MRI + FET-PET |
输出

| 模式                 | 输出重点                      |
| ------------------ | ------------------------- |
| `TumorGrowth`      | 肿瘤生长时间序列                  |
| `BrainDeformation` | 肿瘤生长 + 脑位移/压力/形变结果        |
| `PatientInference` | 患者特异参数推断、肿瘤细胞密度 `MAP.nii` |


TumorGrowth
`给定脑组织和参数，生成一条可能的肿瘤生长轨迹。不是患者真实病史恢复。`

BrainDeformation

`在肿瘤生长基础上再加上占位挤压和脑组织形变，但仍然是 forward simulation，不是自动逆推患者历史。`

PatientInference

`三者里最接近“患者特异”的路线，但官方标准流程需要 MRI + FET-PET，输出是个体化参数/细胞密度推断，而不是唯一真实病程。`

# David
```
目前只有用BrainDeformation模式完成生物力学的仿真，再参考PatientInference模式的迭代优化，
如果你只有 MRI，还想更像“患者特异”，那就不是直接跑官方现成 tutorial 了，而是要做二次开发，例如：
用当前 MRI 的肿瘤分割作为目标
对 Dw/rho/Tend/icx/icy/icz，甚至力学参数，做优化/搜索
找到能最好匹配当前 MRI 肿瘤范围和形变的一组参数
再把这组参数对应的 forward trajectory 当作“最可能历史”。
```
26.4.24
[] 跑通BrainDeformation模式，得到生物力学的仿真挤压
[] 基于真实脑肿瘤影像为ground truth，自建pipeline得到生物仿真，参考PatientInference？



# Github仓库分析
## 1. TumorGrowth 模式
### 功能
在给定的脑组织环境中，从一个初始肿瘤种子出发，按照扩散参数和增殖参数，模拟肿瘤随时间的生长过程,
官方 tutorial 明确把它定义为在 patient anatomy 上进行 tumor growth simulation。

### 用途
- 看“在某个脑组织环境里，这组参数下肿瘤可能怎么长”
- 做参数敏感性分析，研究 WM/GM 异质性对肿瘤形态的影响
- 作为后续更复杂模型或 inference 的 forward 基础模型
注：
- 不能只靠当前一套患者 MRI，自动恢复该患者真实历史生长过程
- 默认不是从患者当前肿瘤 mask 继续往前/往后推
- 不包含 mass effect / 脑组织力学挤压。
### 数据要求
#### 输入
1. **解剖输入**:
- WM.dat
- GM.dat
- CSF.dat
它们是 .dat 三维矩阵文件，不是求解器直接读取 .nii。官方 tutorial 说明 .nii/.nii.gz 需要先经过工具链转换为 .dat。源码 Glioma_ReactionDiffusion::_ic(...) 也是直接拼接并读取 GM.dat / WM.dat / CSF.dat。

2. **初始肿瘤条件**:
- icx
- icy
- icz
这是 3 个归一化坐标标量，不是一个初始肿瘤 NIfTI 掩膜。源码中会用这三个坐标生成一个小球形初始肿瘤种子。

3. **模型参数**
- Dw：白质扩散参数
- rho：增殖参数
- Tend：仿真终止时间
在普通 run.sh 用法里，这些参数是命令行标量；在 UQ 模式中，也可以从文本文件里读取，但本质仍然是数值参数，不是影像。

#### 输出数据
官方 tutorial 说明输出为一系列时间序列结果，例如：
Data_000*.vtu
这些文件表示不同时间点的肿瘤生长结果，可用于可视化肿瘤演化。

### 总结
TumorGrowth 的本质是：
给定患者或模板脑的 WM/GM/CSF + 一组肿瘤参数 + 一个初始种子位置，生成一条“可能的”肿瘤生长轨迹。

它更适合做：
- forward simulation
- 参数探索
- 脑组织环境影响分析

它不等于：
- 患者真实病史重建
- 基于当前肿瘤影像的自动反演
***
## 2. BrainDeformation 模式
### 功能
BrainDeformation 是在肿瘤生长基础上，再加入 mass effect / brain deformation / intracranial pressure 的 forward model。官方 tutorial 把它定义为 tumor mass-effect and intracranial pressure model。
适合的用途
- 模拟肿瘤占位效应对脑组织的挤压
- 分析脑组织位移场
- 分析颅内压相关分布
- 比 TumorGrowth 更接近“占位性病变”的物理表现。
不能直接做的事
- 不能只靠当前 MRI 自动恢复患者真实历史
- 不是现成的 patient-specific inverse solver
- 仍然需要你给定初始肿瘤位置和模型参数。

### 数据要求

#### 输入：
1. **解剖输入**
- WM.dat
- GM.dat
- CSF.dat
- PFF.dat
其中 PFF.dat 是这个力学/形变模式额外需要的输入。

2. **初始肿瘤条件**
- icx
- icy
- icz
仍然是种子位置坐标，不是直接输入初始肿瘤 mask。

3. **生长参数**
- Dw
- rho
- Tend

4. **力学参数**
- kCSF
- kWM
- kGM
- 以及 mobility 等参数
这些参数控制脑组织在肿瘤作用下的变形与压力响应。

#### 输出：
虽然 tutorial 重点强调输入和模型含义，但这一模式的输出核心可理解为：
- 肿瘤时序演化
- 脑组织位移 / 形变场
- 压力相关分布结果。

### 总结
- BrainDeformation 的本质是：
- 给定脑组织、肿瘤参数和初始种子，模拟“肿瘤怎么长 + 脑怎么被挤压”。
- 相比 TumorGrowth：
- 它更接近临床上“占位效应”的现象
- 但仍然是 forward simulation
- 仍然不能直接从单次 MRI 唯一恢复该患者真实病程。

## 3. PatientInference 模式
### 功能
PatientInference 是官方提供的 Bayesian calibration / patient-specific prediction 路线。
它的目标不是单纯 forward 模拟，而是结合患者影像观测，去推断更符合该患者的肿瘤参数和肿瘤细胞密度分布。官方 tutorial 明确写的是：
Bayesian calibration for patient-specific predictions about tumor cell density。

适合的用途
- 做 patient-specific parameter calibration
- 得到最可能的肿瘤细胞密度分布
- 进行个体化预测
- 研究参数后验分布。
- 它比前两种模式更接近你的目标

如果你的目标是：
- “尽量贴近这个患者自己的肿瘤状态”
- “不是随便模拟一条可能轨迹，而是希望受患者当前观测约束”
那么 PatientInference 是三者里最接近这个目标的。
但它也不是“真实历史完全恢复”
官方强调的是 posterior pdf、Bayesian calibration 和 patient-specific prediction，而不是“唯一真实生长史重建”。单时间点观测通常不能唯一确定完整历史。这个结论是对官方方法学定位的合理推断。

### 数据要求
#### 输入
官方 tutorial 列出的输入包括：
1. **MRI / PET 影像**
- FLAIR.nii.gz
- T1c.nii.gz
- FET.nii.gz
2. **对应肿瘤分割**
- Tum_FLAIR.nii.gz
- Tum_T1c.nii.gz
- Tum_FET.nii.gz
3. **脑组织分割**
- WM.nii.gz
- GM.nii.gz
- CSF.nii.gz

也就是说，官方标准 PatientInference 流程需要 MRI + FET-PET，不是只靠 MRI。

#### 输出
官方 tutorial 中指出结果包括：
MAP.nii，即映射回患者解剖空间的、最可能的 tumor cell density 分布。
同时，这一模式也会给出参数 posterior / calibration 相关结果。

#### 总结
PatientInference 的本质是：
利用患者 MRI/PET 观测，对肿瘤模型做个体化校准，得到更符合患者的参数和肿瘤细胞密度预测。
它最接近：
患者特异分析
当前状态约束下的肿瘤分布推断
但它不等于：
自动、唯一地恢复完整真实历史轨迹。