
# 4.6 私有Philips 整理成 BraTS inpainting 格式
## 概述
先做私有数据预处理，也就是把 Philips 数据进行mask，并整理成 BraTS inpainting 格式
（Philips整理后的数据在`D:\Learing_TANG\TaskInpainting\ASNR-Private-Testing`）
```
cd D:\Learing_TANG\TaskInpainting\fastWDM3D-main\fastWDM3D-main
python WDM3D\David_eval\Test_dataprepare_david.py
```
如果你想显式指定路径，推荐这样跑：
```
python WDM3D\David_eval\Test_dataprepare_david.py `
  --source-root "D:\Learing_TANG\TaskInpainting\philips-预处理薄层高质量\philips-预处理薄层高质量" `
  --output-root "D:\Learing_TANG\TaskInpainting\ASNR-Private-Testing" `
  --brats-train-root "D:\Learing_TANG\TaskInpainting\ASNR-MICCAI-BraTS2023-Local-Synthesis-Challenge-Training"
```

Test_dataprepare_david.py 输出预处理后的私有测试集
```
D:\Learing_TANG\TaskInpainting\ASNR-Private-Testing
├─ case_name_mapping.txt
├─ BraTS-GLI-00000-000
│  ├─ BraTS-GLI-00000-000-t1n.nii.gz
│  ├─ BraTS-GLI-00000-000-mask-unhealthy.nii.gz
│  ├─ BraTS-GLI-00000-000-mask-healthy.nii.gz
│  ├─ BraTS-GLI-00000-000-mask.nii.gz
│  └─ BraTS-GLI-00000-000-t1n-voided.nii.gz
├─ BraTS-GLI-00001-000
│  ├─ ...
```
case_name_mapping.txt 记录原 Philips 病例号和新 BraTS 编号的对应关系




## 步骤说明
### 1. Mask库生成
healthy-mask “候选形状池mask库” 从 BraTS 数据里建，当前默认路径是：
`D:\Learing_TANG\TaskInpainting\ASNR-MICCAI-BraTS2023-Local-Synthesis-Challenge-Training`
这个目录下每个病例至少要有：
```
BraTS-GLI-xxxxx-000-t1n.nii.gz
BraTS-GLI-xxxxx-000-mask-unhealthy.nii.gz
```
如果你换成原始 BraTS 训练集，则要求：
```
BraTS-GLI-xxxxx-000-t1n.nii.gz
BraTS-GLI-xxxxx-000-seg.nii.gz
```
最终保存在：
```
binarySegmentationMasks.gz  (D:\Learing_TANG\TaskInpainting\fastWDM3D-main\fastWDM3D-main\WDM3D\David_eval\brats_official_cache\binarySegmentationMasks.gz)
tumorCompartments.gz  (D:\Learing_TANG\TaskInpainting\fastWDM3D-main\fastWDM3D-main\WDM3D\David_eval\brats_official_cache\tumorCompartments.gz)
```
``` 详细步骤
1. 构建 binarySegmentationMasks.gz
对每个 BraTS 病例：读入 t1n,取 t1n != 0 的区域作为脑区候选,取最大连通域并填洞，得到 brainMask,如果是 Local-Synthesis 数据：,读 mask-unhealthy,二值化,去掉小连通域,填洞,与 brainMask 相交,得到 tumorSegmentation,如果是原始 BraTS：,读 seg,取标签 1,2,3,去掉小连通域,填洞,得到 tumorSegmentation
这个文件里保存的是每个病例的：
- brainFolder
- brainMask
- brainMask_V
- tumorSegmentation
- tumorSegmentation_p
2. 构建 tumorCompartments.gz
对 binarySegmentationMasks.gz 里的每个 tumorSegmentation：做 3D 连通域分解对每个连通域取最小包围盒裁出单个肿瘤块压缩保存
这个文件里保存的是：
- brainFolder
- p
- packedCompartment
- compartmentShape
也就是说，tumorCompartments.gz 就是 healthy-mask 采样时真正用到的“mask 库”。
```

### 2. 采样Mask库生成healthy mask
对某个 Philips 病例，脚本会：从私有 ROI 生成当前病例自己的 tumor_mask,从私有 T1W 生成 brain_mask,根据当前病例肿瘤体积比例，从 tumorCompartments.gz 里采样一个肿瘤连通域形状,对该形状做随机翻转、随机旋转,在当前病例脑区中采样放置位置,检查是否满足：,不越界,不和肿瘤重叠,与肿瘤距离足够远,和脑区重叠比例足够高,通过后，这个形状就成为当前病例的 mask-healthy

然后：
```
mask = mask-unhealthy ∪ mask-healthy
t1n-voided = t1n 在 mask 区域置零
```

| 步骤                        | 规则                                                                                                             |
| ------------------------- | -------------------------------------------------------------------------------------------------------------- |
| 构建 unhealthy / tumor mask | 原始 BraTS seg 中 label 1、2、3 合并；去除 <800 voxels 小连通分量；填洞；因此包含 edema                                               |
| 构建候选形状池                   | 对 whole-tumor 二值 mask 做 3D connected components；每个 disconnected compartment crop 成一个候选形状                       |
| 选择候选形状                    | 根据当前病例 tumor 占 brain 的比例 `tumorSegmentation_p`，在候选大小分布的“相反侧”取样：大 tumor 配小 healthy mask，小 tumor 配大 healthy mask |
| 形状增强                      | 随机 X/Y/Z 翻转；在 X-Y 和 Y-Z 平面随机 0°–360° 旋转                                                                        |
| 选择位置                      | 在 brain 内且离 tumor 足够远的区域采样；随机取若干点，选距离 tumor 最远的点作为中心                                                           |
| 有效性检查                     | 与 tumor 至少 5 voxel 欧氏距离；与 zero-T1/background 重叠不超过 25%；不满足则重试                                                  |
| healthy mask 是否必须全在脑实质内   | 否。可重叠脑室、CSF、脑外背景，但 background overlap 被限制                                                                      |

# 4.7 BraTS 1251 single split 训练
把整个训练目录当作训练集使用
- 输入
训练数据集：`D:\Learing_TANG\TaskInpainting\ASNR-MICCAI-BraTS2023-Local-Synthesis-Challenge-Training`
- 输出
权重：`D:\Learing_TANG\TaskInpainting\fastWDM3D-main\fastWDM3D-main\WDM3D\runs\Apr01_21-03-26_DESKTOP-5H`
采样后的结果：`D:\Learing_TANG\TaskInpainting\fastWDM3D-main\fastWDM3D-main\WDM3D\runs\Apr01_21-03-26_DESKTOP-5H\checkpoints\brats3dimage180000`
完整信息：`D:\Learing_TANG\TaskInpainting\fastWDM3D-main\fastWDM3D-main\WDM3D\validation_output_david\ckpt180000\validation_summary_david.json`


## （错误）对训练集中提取一部分进行测试
训练完成后，拿brats3dimage180000.pt对 BraTS train 目录里按 10% 切出来的 125 例做离线验证
- SSIM mean = 0.957222
- PSNR mean = 28.365020
- MSE mean = 0.001833

## 对Philips私有数据集进行测试
做过独立 private test，数据目录是：`D:\Learing_TANG\TaskInpainting\ASNR-Private-Testing`
- SSIM mean = 0.622123
- PSNR mean = 17.537721
- MSE mean = 0.020979

summary:
`D:\Learing_TANG\TaskInpainting\ASNR-Private-Testing-SampleResults\FastWDM\0406\independent_test_summary_david.txt`

```
python WDM3D\David_eval\validate_test_david.py --test_data_dir "D:\Learing_TANG\TaskInpainting\ASNR-Private-Testing" --checkpoint "D:\Learing_TANG\TaskInpainting\fastWDM3D-main\fastWDM3D-main\WDM3D\runs\Apr01_21-03-26_DESKTOP-5H\checkpoints\brats3dimage180000.pt" --output_dir "D:\Learing_TANG\TaskInpainting\ASNR-Private-Testing-SampleResults\FastWDM\0406" --dataset_label private_test --model ours_wnet_128 --batch_size 1 --diffusion_steps 2 --gpu 0

```

### 运行流程参考
D:\Learing_TANG\TaskInpainting\fastWDM3D-main\fastWDM3D-main\WDM3D\FastWDM.md