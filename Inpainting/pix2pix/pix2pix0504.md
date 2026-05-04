
# 与 fastWDM用相同数据集划分进行训练

> 将pix2pix的结果作为 baseline，方便进行与wdm3d的对比

- 直接复用已经存在的 WDM3D split：  
    `D:\Learing_TANG\TaskInpainting\fastWDM3D-main\fastWDM3D-main\WDM3D\David_eval\split1000_val51_test200\train_val_test_split_david.json`
- 源训练数据仍来自：  
    `D:\Learing_TANG\TaskInpainting\ASNR-MICCAI-BraTS2023-Local-Synthesis-Challenge-Training`
- Pix2Pix 保持原始 128×128×96 输入裁剪，不尝试改成 128³。
- 主比较口径以 WDM3D 的 SSIM / PSNR / MSE 为准；MAE 和可视化只是附加诊断输出

评估脚本会按 manifest 的 val 或 test 子集逐例推理，输出和 WDM3D 对齐的 SSIM / PSNR / MSE，再附加 MAE、worst-case 排序和 samples_david 可视化

1. 激活环境
```
$Env:MAMBA_ROOT_PREFIX = "D:\micromamba\root" 
Set-Location "D:\Learing_TANG\TaskInpainting\2023_challenge_main\2023_challenge_main\baseline"
```


```
D:\micromamba\bin\micromamba.exe run -r D:\micromamba\root -p D:\micromamba\root\envs\infillgpu python train_pix2pix_single_split_david.py --dataset "D:\Learing_TANG\TaskInpainting\ASNR-MICCAI-BraTS2023-Local-Synthesis-Challenge-Training" --split-manifest "D:\Learing_TANG\TaskInpainting\fastWDM3D-main\fastWDM3D-main\WDM3D\David_eval\split1000_val51_test200\train_val_test_split_david.json" --batch-size 1 --crop-shape 128 128 96 --gpus 0 --experiment-label Pix2Pix3D_split1000_val51_test200_seed42

```

```
D:\micromamba\bin\micromamba.exe run -r D:\micromamba\root -p D:\micromamba\root\envs\infillgpu python evaluate_pix2pix_single_split_david.py --dataset "D:\Learing_TANG\TaskInpainting\ASNR-MICCAI-BraTS2023-Local-Synthesis-Challenge-Training" --split-manifest "D:\Learing_TANG\TaskInpainting\fastWDM3D-main\fastWDM3D-main\WDM3D\David_eval\split1000_val51_test200\train_val_test_split_david.json" --checkpoint "<best.ckpt>" --subset test
```

