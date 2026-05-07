
> github 仓库：[Deep-MI/neurolit: Neuro Lesion Inpainting Tool (neuroLIT / FastSurfer-LIT) for lesion inpainting e.g. for whole brain segmentation and cortical surface reconstruction in the presence of tumor, surgical cavities or other abnormalities](https://github.com/Deep-MI/neurolit)

# 1. 部署

## 修复
在wsl中使用
```Plain
cd /mnt/d/Learing_TANG/FreesurferInpainting/neurolit-main
chmod +x ./neurolit/scripts/run_lit_containerized.sh
./neurolit/scripts/run_lit_containerized.sh --input_image /mnt/d/Learing_TANG/FreesurferInpainting/neurolit-main/neurolit/BratsData/sub001/BraTS-GLI-00000-000-t1n.nii.gz --mask_image /mnt/d/Learing_TANG/FreesurferInpainting/neurolit-main/neurolit/BratsData/sub001/BraTS-GLI-00000-000-mask.nii.gz --output_directory /mnt/d/Learing_TANG/FreesurferInpainting/neurolit-main/neurolit/BratsData/sub001
```

![[Pasted image 20260505160632.png]]

> 真正更像问题根源的是：你拉到的镜像还是 `deepmi/lit:0.5.0`，而仓库后来的修复补丁明确改了 Dockerfile，把代码改成显式 `COPY lit /inpainting/lit`，这正好对应你报错里缺失的路径 `/inpainting/lit/utils/download_checkpoints.py`。同时，仓库公开 tag 里仍然只有 `v0.5.0`，说明你从 Docker Hub 拉到的公开镜像很可能就是修复前的那版。

本地重新 build 一个修复后的镜像，然后让脚本用你本地镜像

```Bash
cd /mnt/d/Learing_TANG/FreesurferInpainting/neurolit-main
docker build -t neurolit-local:fixed -f containerization/Dockerfile .
./neurolit/scripts/run_lit_containerized.sh --tag neurolit-local:fixed --input_image /mnt/d/Learing_TANG/FreesurferInpainting/neurolit-main/neurolit/BratsData/sub001/BraTS-GLI-00000-000-t1n.nii.gz --mask_image /mnt/d/Learing_TANG/FreesurferInpainting/neurolit-main/neurolit/BratsData/sub001/BraTS-GLI-00000-000-mask.nii.gz --output_directory /mnt/d/Learing_TANG/FreesurferInpainting/neurolit-main/neurolit/BratsData/sub001
```

![[Pasted image 20260505160714.png]]

```Bash
 ./neurolit/scripts/run_lit_containerized.sh --tag neurolit-local:fixed --input_image /mnt/d/Learing_TANG/FreesurferInpainting/neurolit-main/neurolit/BratsData/sub001/BraTS-GLI-00000-000-t1n.nii.gz --mask_image /mnt/d/Learing_TANG/FreesurferInpainting/neurolit-main/neurolit/BratsData/sub001/BraTS-GLI-00000-000-mask.nii.gz --output_directory /mnt/d/Learing_TANG/FreesurferInpainting/neurolit-main/neurolit/BratsData/sub001
```

![[Pasted image 20260505160721.png]]

> 不是挂载脚本坏了，而是**容器里无法把模型从 Zenodo 下载到** **`/usr/local/share/LIT/weights`**。 Zenodo 记录里也明确说了，权重会自动下载，文件名就是这三个。

https://github.com/Deep-MI/neurolit/releases/tag/v0.5.0

![[Pasted image 20260505160730.png]]

```Plain
mkdir -p ~/neurolit_weights
cp /mnt/c/Users/admin/Downloads/model_axial.pt ~/neurolit_weights/
cp /mnt/c/Users/admin/Downloads/model_coronal.pt ~/neurolit_weights/
cp /mnt/c/Users/admin/Downloads/model_sagittal.pt ~/neurolit_weights/
ls -lh ~/neurolit_weights
```

![[Pasted image 20260505160737.png]]

![[Pasted image 20260505160742.png]]

```Bash
docker run --gpus "device=all" --ipc=host \
  --ulimit memlock=-1 --ulimit stack=67108864 --rm \
  -v "/mnt/d/Learing_TANG/FreesurferInpainting/neurolit-main/neurolit/BratsData/sub001/BraTS-GLI-00000-000-t1n.nii.gz:/mnt/d/Learing_TANG/FreesurferInpainting/neurolit-main/neurolit/BratsData/sub001/BraTS-GLI-00000-000-t1n.nii.gz:ro" \
  -v "/mnt/d/Learing_TANG/FreesurferInpainting/neurolit-main/neurolit/BratsData/sub001/BraTS-GLI-00000-000-mask.nii.gz:/mnt/d/Learing_TANG/FreesurferInpainting/neurolit-main/neurolit/BratsData/sub001/BraTS-GLI-00000-000-mask.nii.gz:ro" \
  -v "/mnt/d/Learing_TANG/FreesurferInpainting/neurolit-main/neurolit/BratsData/sub001:/mnt/d/Learing_TANG/FreesurferInpainting/neurolit-main/neurolit/BratsData/sub001" \
  -v "$HOME/neurolit_weights:/usr/local/share/LIT/weights:ro" \
  -u "$(id -u):$(id -g)" \
  neurolit-local:fixed \
  -i "/mnt/d/Learing_TANG/FreesurferInpainting/neurolit-main/neurolit/BratsData/sub001/BraTS-GLI-00000-000-t1n.nii.gz" \
  -m "/mnt/d/Learing_TANG/FreesurferInpainting/neurolit-main/neurolit/BratsData/sub001/BraTS-GLI-00000-000-mask.nii.gz" \
  -o "/mnt/d/Learing_TANG/FreesurferInpainting/neurolit-main/neurolit/BratsData/sub001"
```

# 2. 测试

^2777dc

脚本文件路径在
`D:\Learing_TANG\TaskInpainting\fastWDM3D-main\fastWDM3D-main\neuroLIT`

参考 `README_same_domain_test.md` 完成测试

eg: 对fastwdm效果最好和最坏的10例进行测试
![[Pasted image 20260506092940.png]]

Powershell中进行指标评估
```
cd D:\Learing_TANG\TaskInpainting\fastWDM3D-main\fastWDM3D-main

& 'D:\micromamba\root\envs\fastwdm3d\python.exe' neuroLIT\evaluate_neurolit_same_domain_test.py `
  --split-txt neuroLIT\fastwdm_best5_worst5_by_ssim.txt `
  --output-root neuroLIT\runs\fastwdm_best5_worst5_by_ssim_neurolit `
  --overwrite-converted

```
![[Pasted image 20260506095024.png]]

没有问题，你这 10 例 neuroLIT 已经成功评估完了。结论比较清楚：**fastWDM 总体更适合这个 BraTS 同域测试集，尤其在强度保真度上明显优于 neuroLIT；neuroLIT 在 fastWDM 最差病例中有一定结构修复优势，但不稳定。**

##### 总体对比（10 例）

| 方法                 |    SSIM ↑ |    PSNR ↑ |     MSE ↓ |
| ------------------ | --------: | --------: | --------: |
| fastWDM            |  0.752862 | 23.163808 |  0.013531 |
| neuroLIT           |  0.753229 | 14.748702 |  0.037742 |
| neuroLIT - fastWDM | +0.000366 | -8.415106 | +0.024211 |

总体 SSIM 几乎持平，但 **PSNR 低 8.42 dB、MSE 高约 2.8 倍**，说明 neuroLIT 虽然结构相似性指标不差，但 voxel intensity 的数值误差明显更大。

##### fastWDM 最差 5 例

| 方法 | SSIM ↑ | PSNR ↑ | MSE ↓ |
|---|---:|---:|---:|
| fastWDM | 0.508360 | 15.918341 | 0.026094 |
| neuroLIT | 0.546839 | 13.888962 | 0.046138 |

这组里 neuroLIT 的平均 SSIM 更高，说明它在 fastWDM 容易失败的病例上，有时能给出更合理的结构连续性。尤其 `BraTS-GLI-01063-000`，neuroLIT 三个指标都优于 fastWDM。但其他病例中 neuroLIT 虽提升 SSIM，PSNR/MSE 仍变差，说明结构看起来可能更顺，但强度不够准确。

##### fastWDM 最好 5 例

| 方法 | SSIM ↑ | PSNR ↑ | MSE ↓ |
|---|---:|---:|---:|
| fastWDM | 0.997365 | 30.409275 | 0.000969 |
| neuroLIT | 0.959618 | 15.608442 | 0.029347 |

这组 fastWDM 明显碾压。说明 fastWDM 在同域数据、同划分测试集上，一旦模型适配成功，重建结果非常接近 GT；neuroLIT 作为通用预训练/外部工具，无法达到这种同域拟合精度。

##### 严谨解释

- fastWDM 的优势是：针对该 BraTS local synthesis 数据和相同 split 训练，输出空间和强度分布天然对齐，因此 PSNR/MSE 非常强，尤其在简单或模型熟悉的病例上表现极好。

- neuroLIT 的优势是：它不是针对你的 split 训练的，但在 fastWDM 最差病例中，4/5 个病例 SSIM 高于 fastWDM，说明它可能具有更强的通用结构先验，对某些异常形态能生成更连续的脑组织结构。

- neuroLIT 的劣势也明显：输出经过 256 conform 后再重采样回 BraTS 原始网格，且强度尺度和 BraTS GT 不是原生一致，所以强度保真度差，PSNR/MSE 普遍落后；另外采样时间很长，不适合作为大规模测试主方法。

##### 最终判断

如果你的目标是这套 BraTS 测试集上的定量性能，**fastWDM 是主方法，neuroLIT 更适合作为对照基线或失败病例补充分析**。可以在论文/报告里写：neuroLIT 在 fastWDM 低 SSIM 病例中表现出一定结构鲁棒性，但同域训练的 fastWDM 在整体强度保真和高质量病例重建上显著更优。
