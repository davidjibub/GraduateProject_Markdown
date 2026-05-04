# 标题

基于多模态 MRI与肿瘤生长建模的胶质瘤个体化术前辅助决策系统研究

# 背景与意义

## 医学临床背景
> 胶质瘤手术的关键矛盾在于：既要最大化安全切除肿瘤，又要尽量保护运动、语言、认知等关键脑功能区
### 脑胶质瘤治疗流程

1. 术前影像评估
	基于多模态 MRI、功能影像与弥散影像，评估肿瘤范围、功能区位置和关键白质纤维束走行

2. 手术方案制定
	神经外科根据肿瘤边界、功能区和纤维束空间关系，制定手术入路、切除边界和风险控制策略

3. 术后辅助治疗
	放疗科结合术前/术后 MRI、残余病灶、术腔范围和潜在浸润区域，确定放疗靶区
	
4. 团队协同与医患沟通
	医生团队和患者家属需要理解肿瘤、功能区、纤维束和潜在风险之间的空间关系

### 对应痛点
1. 医生对肿瘤边界识别依赖经验且逐层勾画耗时久
2. 肿瘤破坏正常脑结构，导致脑区配准、皮层指标计算和后续病理模拟输入不稳定
3. 影像可见边界不能完全代表真实肿瘤细胞浸润范围
4. 纤维束与肿瘤空间关系复杂难直观评估手术风险

## 科研技术难点

### 临床痛点对应技术难点

1. 基于多模态影像自动化划分胶质瘤不同子区域
	解决传统人工勾画依赖医生经验，存在耗时长、主观性强、不同医生间一致性不足等问题
	
2. 占位性病变导致神经影像自动化分析管线失效
	Freesurfer等神经影像分析工具能对正常人进行脑功能划分，但脑肿瘤改变了脑组织结构，导致正常脑膜板无法与肿瘤患者进行影像配准，影响自动化脑区标签、皮层指标计算、功能区定位和病理模拟输入的稳定性

3. 结合MRI影像与解剖结构推断细胞密度分布
	胶质瘤具有浸润性生长特征，潜在侵袭性细胞可能超出常规影像可见边界，导致手术切除边界和放疗靶区确定存在不确定性

4. 多模态影像信息的融合与展示
	纤维束，肿瘤，脑功能区等信息难以通过二维切片的形式进行信息融合，且难以支持直观沟通


## PPT
### PPT 1：胶质瘤术前规划的临床需求

同一张PPT中，并列展示
> 治疗流程 -> 临床痛点 -> 技术难点

**术前影像评估**  
多模态 MRI / 功能区 / 关键纤维束  
↓  
**手术方案制定**  
手术入路 / 切除边界 / 功能保护  
↓  
**辅助放疗规划**  
残余病灶 / 潜在浸润 / 放疗靶区  
↓  
**多学科协同与医患沟通**  
三维空间理解 / 风险解释 / 决策支持

胶质瘤不是简单“看见肿瘤就切掉”，而是要在**切除范围、功能保护、术后放疗和风险沟通**之间做综合决策。你的系统正是服务于这个决策链条

### PPT 2：临床痛点与技术难点

|临床痛点|技术难点|对应研究内容|
|---|---|---|
|肿瘤边界和子区域依赖人工勾画，耗时且一致性不足|多模态 MRI 中肿瘤异质性强、边界模糊|胶质瘤多模态 MRI 分割|
|肿瘤破坏正常脑结构，影响脑区标签和皮层指标分析|占位性病变导致配准、皮层重建和自动化分析不稳定|肿瘤区域影像修复|
|影像可见边界不能完全代表真实细胞浸润范围|需要结合影像、脑区标签和生长模型推断细胞密度分布|肿瘤生长病理模拟|
|肿瘤、功能区、纤维束关系复杂，二维切片难以直观理解|多源医学信息融合、三维空间展示和临床解释困难|XR 三维可视化与结构化报告|


# 内容与方法

## 系统框架

![[系统框架图.png]]

## 胶质瘤多模态 MRI 分割

### 模块规划
1. 输入输出
	Input:  
	- t1n
	- t1c 
	- t2w 
	- t2f 
	
	Output：
	肿瘤子区域及标签
	- NCR（label1）
	- ED（label2）
	- ET（label4）
	
2. 公开数据集
	Brats
3. 网络框架图
	

### 现有结果

完成 baseline nnunet 的训练及测试
1. Loss
	损失函数
	train loss / valid loss
2. 模型超参数

|内容|示例|
|---|---|
|optimizer|AdamW / SGD|
|learning rate|1e-4|
|scheduler|cosine annealing / poly LR|
|batch size|2 / 4|
|patch size|128×128×128|
|epochs|300 / 1000|
|early stopping|是否使用|
|hardware|GPU 型号|

3. 评估指标
	Dice / HD95
4. 效果示意图
	(best / worst / median)
	原图 / Ground Truth / Prediction

>本研究首先利用 nnU-Net 建立多模态 MRI 胶质瘤子区域分割基线模型，获得坏死/非增强肿瘤区、水肿区和增强肿瘤区的自动化分割结果，为后续影像修复、肿瘤生长模拟和三维可视化提供结构化输入。

## 肿瘤区域影像修复

>论文参考  [fastWDM3D: Fast and Accurate 3D Healthy Tissue Inpainting | Springer Nature Link](https://link.springer.com/chapter/10.1007/978-3-032-05472-2_17)
>github [AliciaDurrer/fastWDM3D](https://github.com/AliciaDurrer/fastWDM3D)
>Brats github [BraTS-inpainting/2025_challenge: BraTS 2025 Inpainting Challenge (Local Synthesis Challenge)](https://github.com/BraTS-inpainting/2025_challenge)
### 模块规划
1. 输入输出
	Input:  
	- t1n  
	- t1n-voided
	- mask-healthy
	- mask-unhealthy
	- mask
	
	Output：
	- Tumor-inpainted pseudo-healthy T1 image
	
2. 公开数据集
	Brats（train：1000 valid：51 test：200）
3. 构建mask
	- mask-unhealthy：肿瘤区域（包含水肿区）
	- mask-healthy：从候选形状池mask库中构建 [[fastwdm0406]]
		- 依据 mask-unhealthy 构建候选形状池
		- 依据当前影像选择候选形状 mask-healthy
		- 对候选形状进行处理（随机反转，选择远离tumor的位置作为中心）
		- mask-healthy 检验（与 tumor 至少 5 voxel 欧氏距离，与 background 重叠不超过 25%）

|mask|作用|
|---|---|
|mask-unhealthy|表示真实肿瘤及水肿区域，用于真实病例修复|
|mask-healthy|在健康脑组织中构造合成缺损，用于监督训练和定量评价|

4. 网络框架图
	模块最后加上：修复影像 → FreeSurfer / registration / simulation

### 现有结果

完成 baseline pix2pix 和 fastwdm 的训练及测试
1. Loss
	损失函数
	train loss / valid loss
2. 模型超参数 [ pix2pix fastwdm ]

|内容|示例|
|---|---|
|optimizer|AdamW / SGD|
|learning rate|1e-4|
|scheduler|cosine annealing / poly LR|
|batch size|2 / 4|
|patch size|128×128×128|
|epochs|300 / 1000|
|early stopping|是否使用|
|hardware|GPU 型号|

3. 评估指标
	- PSNR / SSIM / MAE
	- 下游任务成功率（Freesurfer异常值，指标异常？）
4. 效果示意图
	(best / worst / median)
	原图 / Ground Truth / Prediction
5. 下游任务验证图
	- Original tumor T1 → FreeSurfer failed / unstable  
	- Inpainted T1 → FreeSurfer success / better cortical metrics
6. Baseline对比

| 方法                  | MAE ↓ | SSIM ↑ | PSNR ↑ | FreeSurfer success ↑ |
| ------------------- | ----: | -----: | -----: | -------------------: |
| Original tumor T1   |     - |      - |      - |                    低 |
| Pix2pix             |       |        |        |                      |
| FastWDM             |       |        |        |                      |
| Fastsurfer-neuroLIT |       |        |        |                      |


> 恢复被肿瘤破坏区域的合理脑结构，为 FreeSurfer、脑区配准、皮层指标计算和病理模拟提供稳定输入


## 肿瘤生长病理模拟

> 论文参考：
> 1. Forward tumor growth models: S. Subramanian, A. Gholami & G. Biros. **Simulation of glioblastoma growth using a 3D multispecies tumor model with mass effect**. Journal of Mathematical Biology 2019 [[arxiv](https://arxiv.org/abs/1810.05370), [jomb](https://link.springer.com/article/10.1007/s00285-019-01383-y)].
>2. TIL inversion algorithms: S. Subramanian, K. Scheufele, M. Mehl & G. Biros. **Where did the tumor start? An inverse solver with sparse localization for tumor growth models**. Inverse Problems 2020 [[arxiv](https://arxiv.org/abs/1907.06564), [ip](https://iopscience.iop.org/article/10.1088/1361-6420/ab649c/meta)]; K. Scheufele, S. Subramanian & G Biros. **Fully-automatic calibration of tumor-growth models using a single mpMRI scan**. IEEE Transactions in Medical Imaging 2020 [[arxiv](https://arxiv.org/abs/2001.09173), [tmi](https://ieeexplore.ieee.org/abstract/document/9197710)].
>3. Mass effect inversion algorithms: S. Subramanian, K. Scheufele, N. Himthani & G. Biros. **Multiatlas calibration of brain tumor growth models with mass effect**. MICCAI 2020 [[arxiv](https://arxiv.org/abs/2006.09932), [miccai](https://link.springer.com/chapter/10.1007/978-3-030-59713-9_53)].
> github参考：[ShashankSubramanian/GLIA: GLioblastoma Image Analysis for integrating brain tumor growth models with medical imaging](https://github.com/ShashankSubramanian/GLIA/tree/master)
> 相似文章参考：[Individualizing Glioma Radiotherapy Planning by Optimization of Data and Physics-Informed Discrete Loss](https://arxiv.org/pdf/2312.05063)

### 模块规划

1. 输入输出
	Input:  
	- patient
		- t1c
		- t2f
		- seg `[wm=6, gm=5, vt=7, csf=8, ed=2, nec=1, en=4]`
		- vt
	- altas
		- seg `[wm=6, gm=5, vt=7, csf=8]`
		- t1n_warp (配准至患者空间)
	Output：
	- 肿瘤细胞分布图
	- mass effect 造成的脑组织位移强度
	- 基于altas重建的伪健康脑组织图（与inpainting区别？结合形变场解释？） #Todo 待确定？
	- 脑室受压变形图
	
2. 公开数据集
	Brats
3. 网络框架图
	- reaction-advection-diffusion PDE
	- linear-screening elasticity
	- ... [[GLIA_Mode_TIL]][[GLIA_Mode_masseffect]]
	- 加入前向？？ #Todo 待确定？

### 现有结果

完成 单例数据 的运行测试
1. Loss
	#Todo 待确定？
2. 模型超参数
	#Todo 待确定？
3. 评估指标
	- 多体素MRS
	#Todo 待确定？
4. 效果示意图
	MRS
	c0/c_final, mri #Todo 待确定？
5. Baseline对比
	#Todo 待确定？

## XR 三维可视化与结构化报告

### 模块规划
1. 输入输出
	Input:  
	- t1n + regions（ freesurfer分割得到的脑区）
	- DTI（nii + .bvecs + .bvals）
	- tumor seg（肿瘤子区域及标签 NCR ED ET）
	- tumor cell （肿瘤密度分布图）
	Output：
	- 纤维束streamline三维模型
	- 脑区三维模型
	- 肿瘤三维模型（tumor cell + tumor seg）
	- 结构化报告
	
2. 网络框架图
	XR Display（model - Unity - HMD）
	Report （value）
	
### 现有结果
1. 效果示意图
	- （纤维束+肿瘤+肿瘤密度分布）

# 总结与展望
## 现有工作

已完成进度总结

## 未来展望

未来工作的时间安排
- 解决泛化问题