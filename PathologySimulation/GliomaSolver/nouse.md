

患者影像数据
│
├─ 只有结构 MRI（如 T1/T2/FLAIR）
│   │
│   ├─ 先做肿瘤/异常区分割
│   ├─ 再做 WM/GM/CSF 分割
│   └─ 评估分割是否受占位/水肿明显影响
│       │
│       ├─ 如果分割不可信
│       │   └─ 先做 lesion masking / filling / tumor-aware segmentation
│       │
│       └─ 如果分割可信
│           ├─ 可进入 TumorGrowth
│           └─ 可进入 BrainDeformation / ICP
│
└─ 结构 MRI + FET-PET + 肿瘤分割 + WM/GM/CSF
    │
    ├─ 数据预处理并映射到 simulation grid
    ├─ Bayesian calibration / inference
    └─ 输出 MAP.nii（肿瘤细胞密度图）+ 报告

# TumorGrowth
## Input
WM / GM / CSF   --> **考虑使用Inpainting等技术得到分割质量较好的 WM/GM/CSF**
肿瘤初始位置和模型参数

## Output
tumor cell density
随时间演化的模拟结果 .dat/.vtu

## Clinical
科研上研究浸润模式
比较不同参数下肿瘤扩散范围
做方法学验证、灵敏度分析
作为其他下游模型的 forward simulator

## Limitation
更像“机制模拟”
与真实患者的一致性强依赖输入分割和参数设定

# BrainDeformation / ICP

## Input
WM / GM / CSF
PFF(1.二值化脑 mask，“脑内=1，脑外=0” 2.二值脑 mask 变成平滑过渡的标量场，命名为 *PFF.dat)
肿瘤参数和力学参数

## Output
组织位移场
颅内压
相关 pressure source / phase-field 通道

## Clinical
研究肿瘤导致的 mass effect
研究影像上中线移位、压迫、组织变形的机制解释
作为 image registration / biomechanical correction 的参考场
研究 ICP 空间分布的非侵入性估计
相关论文明确把这一点写成潜在临床意义：临床干预可能受益于患者脑实质内压力分布的非侵入性估计。

## Limitation
不是临床常规 ICP 测量替代品
力学参数和边界条件的不确定性很大

# Patient-specific Bayesian inference

## Input
MRI（T1c / FLAIR）
FET-PET
tumor masks
WM / GM / CSF

## Output
MAP.nii：患者空间下的肿瘤细胞密度图
参数后验 / 报告 / 不确定性信息

## Clinical
科研上估计影像可见病灶之外的潜在浸润区
分析肿瘤细胞密度与 MRI / PET 信号关系
个体化放疗靶区研究
研究 dose escalation 的候选区域
官方论文明确说：该框架整合 MRI 和 FET-PET，推断带可信区间的患者特异性肿瘤细胞密度；初步临床研究显示，它有助于减少健康组织照射并识别高密度、潜在放疗抗性区域。

## Limitation
对前处理要求最高
PET 不是所有中心都有
更接近“临床研究辅助”而不是“日常临床标准工具”


