---
type:
  - thesis
  - project
status: done
module: GLIA模式总览分析
tags:
  - thesis
  - project
created: 2026-04-25
topic:
  - pathology simulation
  - github code based
thesis: "[[GLIA Pathology Simulation Module Overview]]"
---

## forward.py：正向模拟，不是临床反推

你的理解“把肿瘤加入正常人 patient 图像中”基本接近，但更准确地说：
forward 的输入是一个健康/模板脑 atlas 的组织分割，包括 WM、GM、ventricles、CSF，可选 T1 MRI。然后你给定肿瘤种子位置 user_cms，以及反应/扩散/质量效应参数，例如 rho、k/kappa、gamma，GLIA 从这个种子开始模拟肿瘤生长和组织受压变形。官方脚本说明写得很明确：输入是 healthy/atlas brain image segmented into WM、GM、VT、CSF；输出包括每个时间步的 tissue、tumor、displacement、velocity、mass effect forcing function、Lamé coefficients、reaction/diffusion coefficients 等。

临床意义：
它回答的是：
- 如果某个健康脑中，从某个位置开始长 GBM，给定扩散、增殖、mass effect 参数，最后会长成什么样？
所以它适合做 synthetic data、模型验证、参数敏感性分析。不是你现在“已有患者影像，反推健康状态”的主流程。


## inverse.py / inverse_nomasseffect.py / inverse_til.py：反演肿瘤起点和生长参数，但不显式估计 mass effect

这个模式输入的是患者/病灶脑影像的分割，包括 WM、GM、CSF、ventricles，以及肿瘤标签，例如 enhancing tumor、necrosis、edema。官方说明中说，这类 inverse 输入是 diseased/patient brain segmentation，并假设图像已经 affine registered 到某个模板空间。输出包括 c0_rec 和 c1_rec：c0_rec 是反演出的肿瘤初始条件/起点，c1_rec 是用这个初始条件正向长到观察时刻的肿瘤浓度。

临床意义：
它回答的是：
- 这个肿瘤大概从哪里开始？它的扩散/增殖参数大概是多少？如果不考虑组织被挤压变形，模型能否解释当前肿瘤范围？
注意：这一步不是严格反推健康脑结构，因为它通常把患者脑中肿瘤区域用某种近似组织填充，先估计 tumor initiation location。GLIA 论文中的多阶段方法第一步就是先用 no-mass-effect 模型估计 tumor IC

| 输出                            | 含义                                          |
| ----------------------------- | ------------------------------------------- |
| `c0_rec.nii.gz` / `c0_rec.nc` | 反演的 tumor initial condition，近似“肿瘤起源位置/初始分布” |
| `c1_rec.nii.gz` / `c1_rec.nc` | 模型在观察时刻重建出的肿瘤浓度                             |
| `c_pred_at_t=1.2/1.5`         | 假设继续按模型生长，未来时刻的预测浓度                         |
| `reconstruction_info.dat`     | 参数、误差、收敛信息                                  |
| `solver_log.txt`              | 优化过程、误差、timing、收敛情况                         |

## inverse_masseffect.py：你最关心的模式，估计 tumor-induced deformation / mass effect

inverse_masseffect 才是和“肿瘤把脑组织挤压变形了多少”最相关的模式。官方脚本说明说，它是在 inverse_til.py 之后运行的，用更复杂的模型重建 mass effect deformation；它需要患者分割，还需要多个健康模板脑 atlas，并使用 CLAIRE 做配准；同时需要前一步 inverse_til 的 TIL 输出。

它的思想是：
- 你没有这个患者真正患病前的健康 MRI；
- 所以 GLIA 用多个正常人/atlas 作为“可能的健康脑”代理；
- 先把患者和 atlas 配准；
- 把前一步估计的 tumor initial condition 转移到每个 atlas；
- 在每个 atlas 上反演 rho、k、gamma；
- 最后用 ensemble/multi-atlas 的方式降低单个 atlas 偏差。
论文摘要也明确说：mass effect calibration 的难点是患者的 pre-cancer healthy anatomy 未知，所以他们用多个脑 atlas 作为健康前状态的 proxy。

临床意义：
它回答的是：
- 如果把某些健康 atlas 当作这个患者患病前的近似脑结构，那么当前肿瘤造成的组织位移场、最大位移、ventricle deformation、mass effect 强度大概是多少？

| 输出                                                                          | 含义                                                             |
| --------------------------------------------------------------------------- | -------------------------------------------------------------- |
| `displacement_rec_final.nc`                                                 | 观察时刻的重建位移场，最核心的 mass effect 输出                                 |
| `vel_X/Y/Z_rec_final.nc`                                                    | 速度/形变相关场                                                       |
| `gamma`                                                                     | mass effect forcing/intensity 参数                               |
| `max_disp`                                                                  | 最大位移，常用于量化 mass effect 强弱                                      |
| `norm_disp`                                                                 | 位移场范数/整体形变量                                                    |
| `wm_rec_final.nc`, `gm_rec_final.nc`, `vt_rec_final.nc`, `csf_rec_final.nc` | 模型重建出的受肿瘤影响后的组织分布                                              |
| `mri_rec_final.nc`                                                          | 用模板 MRI 生成的重建 MRI，用于和患者 MRI 定性比较                               |
| `stats.csv`                                                                 | 多个 atlas 的参数和统计结果，包括 `gam`, `rho`, `k`, `u`, `err`, `vt_err` 等 |
| `seg_rec_final.nc`                                                          | 最终估计的分割                                                        |
| `c_rec_final.nc` / `c1_rec.nc`                                              | 当前时刻重建的肿瘤浓度                                                    |
