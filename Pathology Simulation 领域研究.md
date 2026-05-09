
# 领域从“生物物理真实性”转向“临床可用反演”
过去 10 年大致有 4 条主线：

1. **GLIA 路线：高保真生物物理模型**  
    重点是多组分、坏死、缺氧、mass effect、组织形变，解释病灶形态很强，但参数反演昂贵、单次临床 MRI 下病前健康脑未知，临床工作流较难

2. **临床影像个体化路线：用 MRI/PET/DWI 给模型加约束**  
    重点从“模拟像不像”变为“能不能预测复发、治疗反应、放疗靶区”

3. **快速反演路线：Bayesian / ensemble / deep learning / PINN / ODIL**  
    目标是解决 GLIA 这类 PDE-constrained optimization 太慢、太病态的问题

4. **临床转化路线：从 tumor Dice 到 recurrence coverage / survival stratification / radiotherapy planning**  
    最新 Nature Communications 级别的工作已经直接把模型输出用于放疗靶区个体化评估，而不只是模拟肿瘤形状

| 论文                                                                            | 建模方式                                                                                                             | 相比 GLIA 的亮点                                                                                                                    | 验证方式                                                                                                       | 临床有效性判断                                                                                                                                                                                                                                                         |
| ----------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **1. Lipková et al., 2019, TMI — Personalized Radiotherapy Design**           | 反应–扩散模型 + 多模态影像 + Bayesian inference，用于个体化放疗设计                                                                   | 不追求 GLIA 那种多组分 mass effect，而是把模型直接嵌入 **radiotherapy planning**；重点是估计不可见浸润区                                                     | 多模态扫描 + Bayesian 框架，输出个体化放疗方案                                                                              | 临床意义强，但更像 retrospective / planning framework，尚不是前瞻性临床决策工具。([PMC](https://pmc.ncbi.nlm.nih.gov/articles/PMC7170051/?utm_source=chatgpt.com "Personalized Radiotherapy Design for Glioblastoma - PMC"))                                                           |
| **2. Hormuth et al., 2021, Scientific Reports — Image-based personalization** | 单/双组分 reaction–diffusion + chemoradiation response + DWI/ADC 估计细胞密度                                              | GLIA 强在病理生长结构，Hormuth 强在 **治疗响应预测**；引入 DWI 定量影像，模型参数更贴近患者局部细胞密度                                                                | 9 名 high-grade glioma 患者，预测 3 月和 5 月 MRI；两组分模型体积预测误差中位数低于 2.5%                                             | 有临床潜力，但样本很小；作者也明确需要更大前瞻性验证。([Nature](https://www.nature.com/articles/s41598-021-87887-4 "Image-based personalization of computational models for predicting response of high-grade glioma to chemoradiation \| Scientific Reports"))                            |
| **3. Tunç et al., 2021, IEEE TBME — Longitudinal MRI with mass effect**       | 比较 RD、RDA、RDAM 三类模型，其中 RDAM 加入 mass effect                                                                       | 与 GLIA 最接近：同样强调 mass effect，但更偏向 **纵向 MRI 反演与模型比较**，而不是 GLIA 的完整多组分病理机制                                                        | 用 longitudinal MRI 评估不同模型对 tumor growth 和 mass effect 的捕获                                                  | 证明 mass effect 建模有价值，但主要是影像预测/模型评估层面，距离治疗决策仍有距离。([PubMed](https://pubmed.ncbi.nlm.nih.gov/34061731/?utm_source=chatgpt.com "Modeling of Glioma Growth With Mass Effect by Longitudinal ..."))                                                                   |
| **4. Subramanian et al., 2023, TMI — Ensemble inversion with mass effect**    | PDE tumor growth + mass effect；用多个正常脑模板近似病前健康脑；单次 mpMRI 反演 tumor initiation、proliferation、migration、mass effect  | 这是 GLIA 方向的关键升级：GLIA inverse_masseffect 依赖 atlas/patient mismatch，但单 scan 下“病前健康脑未知”很难；ensemble inversion 用多模板缓解 ill-posedness | synthetic 验证 + 216 名 GBM 临床数据；显著 mass effect 患者中加入 mass effect 后平均 Dice 提高约 10%；还做 survival stratification | 是 GLIA 系列最接近临床 biomarker 的进展，但 survival 分析仍是 preliminary，尚未证明能改变治疗结局。([PMC](https://pmc.ncbi.nlm.nih.gov/articles/PMC10201550/ "Ensemble inversion for brain tumor growth models with mass effect - PMC"))                                                      |
| **5. Ezhov et al., 2023, Medical Image Analysis — Learn-Morph-Infer**         | 深度学习替代传统 PDE 反演：从 T1Gd/FLAIR 推断患者 tumor cell density，可适配 RD 和 RDA 模型                                             | GLIA 的瓶颈是每次参数更新都要多次 forward solve；LMI 的亮点是把“反演”变为近实时推断，计算时间可降到分钟级                                                              | synthetic + clinical MRI；重点验证反演速度和 tumor density reconstruction                                            | 临床工作流友好很多，但对真实低密度浸润的 ground truth 仍不足；更像“快速反演引擎”，不是完整病理模型。([科学直通车](https://www.sciencedirect.com/science/article/abs/pii/S1361841522003000?utm_source=chatgpt.com "A new way of solving the inverse problem for brain tumor ..."))                              |
| **6. Martens et al., 2022, Cancers — DL for reaction–diffusion modeling**     | DCNN 从 MRI contour 重建全脑 tumor cell density，并估计 diffusivity/proliferation                                         | 相比 GLIA 的 PDE-constrained inverse，它用神经网络解决两个老问题：初始条件未知、参数估计病态                                                                  | 1200 个 synthetic tumors，真实脑几何；另展示真实 GBM 患者适用性                                                              | 临床有效性偏早期；价值在于证明 DL 可补 GLIA 反演短板。([PMC](https://pmc.ncbi.nlm.nih.gov/articles/PMC9139770/?utm_source=chatgpt.com "Deep Learning for Reaction-Diffusion Glioma Growth Modeling"))                                                                                 |
| **7. Zhang et al., 2025, Medical Image Analysis — PINN for GBM infiltration** | Physics-Informed Neural Network，把 reaction–diffusion PDE 和 MRI 数据共同放进 loss；从单次 3D MRI 估计患者参数                     | 相比 GLIA 的显式数值求解 + finite-difference gradient，PINN 把 PDE residual 当软约束，理论上更适合单 scan、复杂几何和快速反演                                   | synthetic + patient datasets；使用 nondimensional pretraining + patient fine-tuning                           | 很适合做 GLIA 的“反演替代模块”，但目前主要面向 infiltration prediction，尚未覆盖 GLIA 的多组分、oxygen、necrosis、elasticity。([科学直通车](https://www.sciencedirect.com/science/article/abs/pii/S1361841524003487?utm_source=chatgpt.com "Personalized predictions of Glioblastoma infiltration")) |
| **8. Balcerak et al., 2025, Nature Communications — GliODIL**                 | GliODIL：MRI/FET-PET + Fisher–Kolmogorov physics model + ODIL，软同化数据项和物理约束，输出全空间 tumor cell concentration 和个体化放疗靶区 | 这是当前最值得你重点读的前沿：它不追求 GLIA 的复杂病理组分，而是把物理模型、MRI/PET 和 recurrence validation 直接连到 **radiotherapy target individualization**        | 152 名 GBM 测试集，其中 58 例有 pre-treatment FET-PET；用 1–12 月 post-treatment MRI follow-up 验证 recurrence coverage  | 临床相关性最强：直接挑战 uniform margin 放疗策略；但仍需前瞻性试验验证是否改善生存或毒性。([Nature](https://www.nature.com/articles/s41467-025-60366-4 "Individualizing glioma radiotherapy planning by optimization of a data and physics-informed discrete loss \| Nature Communications"))        |


# 1. GLIA（forward）  2019
>Simulation of glioblastoma growth using a 3D multispecies tumor model with mass effect

## Input 

- **健康脑组织图 / atlas / segmentation**  
    用健康脑作为初始空间环境，包括 WM、GM、CSF 等组织

- **BraTS / GLISTR 等真实 MRI 数据作为影像形态参考**  
    论文展示了真实 GBM MRI 中的 FLAIR、T1、T1-Gd、T2 和 segmentation，用来说明 GBM 典型结构：水肿、增强肿瘤、坏死核心、mass effect。作者还用 GLISTR / BraTS 数据中的 MRI 强度分布，把模拟出的组织和肿瘤结构转换成类似 MRI 的 synthetic images
    

## Model

multispecies reaction–advection–diffusion PDE+linear elasticity
[[GLIA_Mode_masseffect]]

## Valid
> 这篇的验证不是临床预测验证，而是**模型合理性验证**

- **视觉/形态验证**  
    模拟结果能产生 GBM MRI 中常见的结构：增强 rim、坏死核心、水肿、mass effect。作者展示了模拟 MRI 图像，并说 multispecies mass-effect 模型能捕获 GBM 的多组分结构和周围组织形变

- **与 single-species model 对比**  
    单组分 reaction–diffusion 模型只能得到一个 tumor concentration，不能自然区分增强肿瘤、坏死核心、水肿等影像结构。multispecies 模型的优势是不用简单 threshold 就能产生不同病理区域

- **参数敏感性分析**  
    作者分析了 reaction coefficient、hypoxia threshold、death rate、oxygen consumption、transition rate、diffusion coefficient 等参数对不同肿瘤组分的影响。结论是 reaction coefficient 和 hypoxia threshold 对多个肿瘤组分都很重要

注意这里不是去除参数的消融实验，而是参数敏感性

> 固定其他参数，只改变某个参数的取值，看最终模拟出来的肿瘤形态、各组分体积、坏死区域、水肿、mass effect 等输出怎么变	

这和“去除参数”不同。去除参数相当于说“没有 hypoxia 机制”，那是模型结构消融；敏感性分析是说“这个参数在合理范围内变化，结果变化有多大”

eg: 假设模型输出 4 张图：
1. 增殖肿瘤 ppp；
2. 侵袭肿瘤 iii；
3. 坏死核心 nnn；
4. 脑组织位移 uuu。

然后你把 hypoxia threshold 从 0.2 改到 0.4：
- 如果 threshold 低：只有很缺氧时才坏死，坏死核心可能小；
- 如果 threshold 高：稍微缺氧就坏死，necrotic core 会更早出现、更大；
- 这说明 hypoxia threshold 对 necrosis 和 tumor morphology 很敏感。


- **mesh convergence / 数值收敛性**  
    他们做了网格收敛实验，证明数值方案大致有一阶收敛

mesh convergence 是数值 PDE 论文常做的验证，目的是回答：

> 我模拟出来的肿瘤形状，是模型真实结果，还是网格太粗造成的数值假象？


通俗说，你用电脑求 PDE 时，连续大脑被离散成体素/网格：
- 粗网格：64
- 中等网格：128
- 细网格：256

如果模型是可靠的，那么网格越细，结果应该逐渐稳定

eg: 假设你模拟同一个初始肿瘤，跑到 t=1，得到最终肿瘤总体积：

| 网格  | 最终肿瘤体积   |
| --- | -------- |
| 64  | 42 cm³   |
| 128 | 39 cm³   |
| 256 | 38.5 cm³ |

这说明从 128到 256 已经很接近，模型在收敛
但如果是：

| 网格  | 最终肿瘤体积 |
| --- | ------ |
| 64  | 42 cm³ |
| 128 | 30 cm³ |
| 256 | 55 cm³ |

那说明数值方法不稳定或分辨率不够，模拟结果不可信

2019 论文摘要明确提到他们提出了 operator-splitting scheme，包括 semi-Lagrangian advection 和 elliptic solvers；这类数值方法需要通过 mesh convergence 证明离散化是可靠的

所以 mesh convergence 不是临床验证，而是**数值正确性验证**

## 临床用途

这篇本身还不是临床工具，但作者的临床目标很明确：

1. **帮助理解 GBM 影像结构**  
    为什么增强 rim 在外面？为什么中心坏死？为什么水肿包围肿瘤？为什么脑室被挤压？

2. **为患者级参数反演做 forward model**  
    作者明确说最终目标是结合 patient MRI 做参数估计，用于诊断、预后、图像分割等

3. **合成数据生成 / segmentation augmentation**  
    文章提到该模型也可用于生成 synthetic data，辅助医学图像分割模型训练

所以它的临床价值是**基础模型层面的**，不是已经证明能改善治疗结局


# 2. GLIA （single-specie mass effect inverse）2023
>Ensemble Inversion for Brain Tumor Growth Models With Mass Effect

## Input

- 单次术前 mpMRI；
- tumor segmentation；
- brain tissue segmentation / atlas；
- 多个 normal brain templates，作为患者病前健康脑的 proxy

数据集
- **Synthetic dataset**  
    用来验证 solver：因为 synthetic tumor 有 ground truth，可以知道反演出来的 ρ,κ,γ\rho,\kappa,\gammaρ,κ,γ、tumor origin 和 displacement 是否接近真值

先用 forward model 人工生成一个病例
$健康脑模板+设定好的 seed, ρ,κ,γ→模拟出的 tumor + mass effect$

然后再把这个模拟结果当作“观测 MRI”，让 inverse solver 去反推参数。这样可以比较：
$反演出来的参数vs生成时设定的 ground truth 参数$


- **216 名 GBM 患者临床 mpMRI 数据**  
    作者在 216 名 glioblastoma 患者上做 retrospective analysis，并用模型重建结果提取 biophysics-based features，再做 survival stratification / survival prediction



## Model

single-species reaction–advection–diffusion + mass effect

## Valid

- **Synthetic validation**  
    因为 synthetic case 有 ground truth，可以评估反演误差

- **Clinical reconstruction quality**  
    在 216 名 GBM 患者上，看模型反演后是否能匹配 observed tumor pattern

- **Mass effect 是否真的有帮助**  
    作者比较加入 mass effect 与不加入 mass effect 的模型。结果显示，对 significant mass effect 患者，加入 mass effect 后 average Dice coefficient 提高约 10%

- **Survival analysis**  
    作者提取 biophysics-based features，用于患者分层和 overall survival prediction。结果是 preliminary，但显示这些特征可能改善 survival stratification

## 临床用途

这篇的临床用途比 2019 GLIA forward paper 更直接：

1. **量化 mass effect**  
    mass effect 不只是“看起来脑室被压了”，而是可以输出 displacement field、局部形变、力学相关指标。
2. **提取 biophysical biomarkers**  
    例如：
    - 肿瘤更偏“扩散型”还是“增殖型”；
    - mass effect 强不强；
    - tumor origin 位置；
    - infiltration pattern；
    - displacement / deformation features。
3. **预后分层**  
    作者尝试用这些 biophysics-based features 做 survival prediction。这个方向很重要，因为它把 PDE 模型从“图像重建工具”推进到“影像生物标志物”。

但它仍然是 retrospective，不是前瞻性临床试验，因此不能说已经证明能改善患者治疗结局

# GliODIL Balcerak et al., 2025
> Individualizing glioma radiotherapy planning by optimization of a data and physics-informed discrete loss

## Input

- **152 名成人 GBM 患者**  
    都是 WHO-CNS grade 4 IDH-wildtype glioblastoma

- **术前和术后 MRI + 复发随访 MRI**  
    MRI 包括 3D T1w-MPRAGE、增强 T1、FLAIR、T2，1 mm isotropic resolution

- **58 名患者有术前 FET-PET**  
    FET-PET 提供肿瘤代谢活性信息

- **post-treatment follow-up MRI**  
    用于观察 1–12 个月内复发区域，从而验证模型预测的高风险区域是否覆盖真实复发。论文明确说测试集包括 152 名 GBM，其中 58 名有 FET-PET，且有 post-treatment MRI follow-ups 用于 recurrence monitoring 和 validation

## Model

reaction–diffusion PDE

| 变量                            | 通俗解释                                      |
| ----------------------------- | ----------------------------------------- |
| ($u(x,t)$)                    | tumor cell density：每个 voxel 里“肿瘤细胞浓度”     |
| ($D(x)$)                      | 空间变 diffusion：肿瘤沿不同组织扩散的容易程度              |
| ($\rho$)                      | proliferation：肿瘤细胞原地增殖速度                  |
| ($R$)                         | WM/GM diffusion ratio，通常白质中更容易沿结构扩散       |
| ($x_0,y_0,z_0$)               | 初始 tumor seed 位置                          |
| ($\theta_{down},\theta_{up}$) | 把 tumor cell density 映射到 edema/core 的影像阈值 |
| PET scaling/background 参数     | 把 tumor density 映射到 FET-PET 代谢信号          |

### ODIL

真正的创新是 **ODIL：Optimizing a Discrete Loss**

GLIA / 传统 PDE 反演通常是：
> 给参数 → forward solve PDE → 比较终态 → 更新参数

GliODIL 是：
> 直接把整条 4D tumor evolution u(x,y,z,t)u(x,y,z,t)u(x,y,z,t)、PDE residual、MRI/PET 数据项一起放入 loss，用 automatic differentiation 优化

loss 包括：
- $L_{PDE}$：是否满足 reaction–diffusion 方程
- $L_{IC}$​：是否来自一个 focal Gaussian initial condition
- $L_{CORE}$​：是否匹配 tumor core
- $L_{EDEMA}$​：是否匹配 edema
- $L_{PET}$​：是否匹配 FET-PET
- $L_{PARAMS}$​：参数是否在合理范围

GLIA 更像“我相信 PDE 是硬规则，参数调到让终态像 MRI”； 
==GliODIL 更像“PDE 是强先验，但真实病人可能不完全遵守它，所以我同时允许数据项修正 PDE”==

## Valid

- **Synthetic data validation**  
    用 single-focal 和 localized multi-focal synthetic tumors 校准 LPDEL_{PDE}LPDE​ 权重，测试噪声、PET partial volume effect、坏死无代谢信号等真实问题。
- **Dice score**  
    比较模型 inferred tumor cell density 阈值化后，与 MRI segmentation 的重叠。
- **FET-PET signal correlation**  
    看 inferred tumor cell density 是否与 PET 代谢强度相关。
- **Recurrence Coverage**  
    这是最重要的临床验证指标：  
    模型给出的放疗靶区是否覆盖后续 MRI 上真实复发区域。论文把 recurrence coverage 定义为 follow-up MRI 检测到的复发中，有多少比例被某个 radiation target 覆盖。
- **与 standard clinical plan 比较**  
    标准方案是围绕术前可见肿瘤做 15 mm uniform margin，并排除非脑组织和 CSF。GliODIL 生成的 CTV 与 standard plan 保持相同总体积，以公平比较覆盖率

## 临床用途

这篇的临床用途非常明确：**个体化放疗靶区设计**。

传统 GBM 放疗常用 uniform margin：比如可见肿瘤外扩 15 mm。问题是 GBM 侵袭不是球形均匀扩散，它受白质/灰质结构、代谢活性、坏死、周围组织限制等影响。

GliODIL 的目标是：

> 根据患者自己的 MRI + PET + PDE 物理先验，推断不可见 tumor cell distribution，然后用它设计更个体化的 CTV。

论文结果显示，GliODIL 是唯一 consistently outperform Standard Plan 的模型；作者强调它在保持治疗总体积与标准方案相当的情况下，对复杂肿瘤形状的高风险复发区域有更好覆盖。

## Conclusion

1. 严格遵守 PDE 反而可能不如“PDE + data-driven correction”
	GliODIL discussion 明确指出，strict PDE solutions 即使加入额外 imaging input，也可能无法覆盖完整复发；GliODIL 因为有 adaptive/data-driven component，能更好利用 FET-PET 信号修正预测

2. FET-PET
	作者强调 FET-PET 明显提升预测准确性，因为它提供了 MRI 看不到的代谢信息

# PET

> PET 已经告诉我哪里代谢高，那我直接照 PET 高信号区域放疗不就行了吗？为什么还要 GliODIL？

PET 很有用，但 PET 不等于“完整肿瘤细胞密度图”，也不等于“未来复发风险图”

- FET-PET 测的是氨基酸摄取/代谢活性。高信号区域通常更可疑，但它不是直接的 tumor cell density ground truth

| 情况        | PET 可能表现   | 问题                    |
| --------- | ---------- | --------------------- |
| 高代谢活跃肿瘤   | PET 高      | 很有价值                  |
| 坏死核心      | PET 低或无代谢  | 但仍属于肿瘤核心结构            |
| 低密度浸润细胞   | PET 可能低或模糊 | 可能未来复发                |
| 炎症/治疗反应   | PET 可能有信号  | 不一定全是肿瘤               |
| 小灶 / 边界区域 | PET 受分辨率限制 | partial volume effect |

## 放疗

放疗靶区规划不是只问：
> 今天哪里 PET 高？

而是问：
> 哪些区域现在看不见、但未来最可能复发，应该被 CTV 覆盖？

==GBM 的复发常常发生在 MRI/PET 可见边界之外==
GliODIL 的核心是把 PET 当前信息和 PDE migration path 结合起来。论文明确说，GliODIL 通过整合 PET 和 PDE regularization，能够捕获一些位于术前可见肿瘤边界之外、标准 1.5 cm margin 捕获不到的远处复发

这就是为什么不能直接“PET 高信号外扩一下”完事

### 靶区限制

临床放疗必须平衡：
- 覆盖复发风险；
- 保护正常脑组织；
- 控制 CTV 体积；
- 避免认知损伤、放射性坏死等毒性

GliODIL 的比较方式很重要：它把 GliODIL-derived CTV 和标准 15 mm margin CTV 控制在**相同总体积**下比较 recurrence coverage。论文说明，standard CTV 是围绕术前 enhancing/necrotic tumor 做 15 mm uniform margin，并排除非脑组织和 CSF；各种 radiotherapy plan 保持相同总体积，再比较 recurrence coverage。

所以 GliODIL 问的是：

> 在同样大小的放疗体积里，怎样把剂量区域放得更聪明？

GliODIL 不是“用 PET 替代模型”，而是“让 PET 修正模型”
$MRI+PET+tissueanatomy+PDEgrowthprior→tumorcelldensity/recurrenceriskmap→个体化CTV$

