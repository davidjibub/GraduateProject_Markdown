
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

脚本文件路径在
`D:\Learing_TANG\TaskInpainting\fastWDM3D-main\fastWDM3D-main\neuroLIT`

参考 `README_same_domain_test.md` 完成测试