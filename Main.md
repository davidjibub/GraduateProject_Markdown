# 1. Segmentation
使用nnunetv2完成肿瘤的自动分割

#### 开题前需完成
`验证segmentation管线可行`
- [x] nnunetv2 对brats数据集 train=1000 单折训练
- [ ] 对test=251 测试



#### 开题前可完成
- [ ] 使用OpenMind完成对私有数据集的微调，作为对比
- [ ] nnunetv2 对brats数据集 train=1000 5-fold 交叉验证。test=251，用于最终测试


#### 需考虑清楚
<font color="red">必做</font>
- [ ] Segmentation是否是主要工作量，主张是模型的泛化能力还是？ ->决定使用的模型（Swin UNETR）和数据集（是否加入私有） 

#### 后续安排



# 2. Inpainting
背景及意义
- 放射科划分脑功能区慢，且同样依靠对侧信息根据脑沟脑回划分，缺失自动化流程
- 让异常脑也能进入正常脑自动化计算脑区，皮层厚度，体积分割等自动化流程

方法
- 使用fastWDM完成肿瘤的生成修复，确保后续的脑区分割等自动化管线能正常运行，baseline -pix2pix

#### 开题前需完成
##### 验证inpainting管线可行（跨域）
详细结果见`Inpainting\fastWDM\fastwdm0410.md`
- [x] fastwdm 对BraTS 1200/51 single split 训练，验证损失选最优 checkpoint
- [x] fastwdm 训练过程稳定性及优化方向（train loss，valid loss）
- [x] fastwdm best checkpoint对私有Philips数据集测试 (模型跨域性能)
- [ ] fastwdm 与 baseline的对比
```
- 目前的问题是私有Philips数据无法通过指标很好的衡量
    eg:BraTS-GLI-00008-000	SSMI=0.9163020849227905	PSNR=19.60123062133789	MES=0.010961671359837055
       指标值好，是因为其healthy mask极小
       BraTS-GLI-00013-000	SSMI=0.875453531742096	PSNR=24.553646087646484	MES=0.0035045745316892862
       指标值好，但是其肿瘤区域有值=0的地方，未完全修复
```
**-> 缺乏好的修复评估指标**
####
##### 验证inpainting管线可行（同域）
- [ ] fastwdm 对BraTS 1000/51/200 single split 训练，验证损失选最优 checkpoint
- [ ] fastwdm 与 baseline的对比 

####
#### 开题前可完成
- [ ] 添加模块或者修改network
- [ ] 与成熟的Freesurfer-neuroLIT对比
####
#### 需考虑清楚
<font color="red">必做</font>
- [ ] 核心主张是对后续的自动化脑区分割有效，如何证明？
- [ ] 数据集太少，diffusion model能够学习到有效信息？


####
## 后续安排
- [ ] 考虑与其他的模型做对比（pseudo）？
- [ ] 添加模块或者修改network

####
# 3. Pathology Simulation
GLIA


####
## 待考虑
1. GliomaSolver
    需要先跑通`BrainDeformation`,再参考`PatientInference`完成迭代设计
    详见`PathologySimulation\GliomaSolver\GliiomaSolver_.md`
####
# 4. NeuroImage Analysis
- [x] Brain Regions (VBM) -> MarchingCubes
- [ ] Brain Regions (SBM) -> 临床意义待确定
- [x] Fiber tract (Regions)
- [ ] Fiber tract (CSF...) -> JHU Altas or TractSeg or RapidTract
####
## 待考虑
接入本地LLM，给出初步的分析判断 -> 临床意义待确定

####
# 5. XR Display
- [x] Unity 完成AR场景下的可视化
