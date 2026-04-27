# 4.27 BraTS 1000/51/200 single split 训练 + 验证损失选最优 checkpoint
## 训练
用 BraTS2023 Local Synthesis Challenge Training 数据集训练 FastWDM / ours_wnet_128
在现有 1000 train / 51 val / 200 test manifest 基础上，保持模型结构、训练超参数、train loss / val loss 记录、best checkpoint 选择逻辑全部不变，只补“同域 test 评估链路”和“结果目录组织”
1. 训练流程保持不变，仅换 split manifest 与 run 输出目录,继续使用现有 run_david.py 训练，不修改任何训练参数或训练记录逻辑。
    训练输入改为：
    - --data_dir 仍指向 1251 例 BraTS 根目录
    - --split_manifest 改为：`train_val_test_split_david.json`

    训练输出改为：
    - `WDM3D/runs/ours_wnet_128_split1000_val51_test200_seed42`

2. 扩展 manifest 子集选择能力，但不动训练核心
在 common_david.py 中把 prepare_dataset_from_manifest(...) 扩展为显式支持子集选择
    - 训练入口不改，仍只读 train_ids / val_ids
    - 评估入口在同域 test 时显式要求 subset="test"
这样不会影响现有 1200/51 single-split 实验

3. 扩展统一 launcher，支持 train 后直接做同域 test
扩展 launch_single_split_david.py
训练：`run_david.py 或 launcher --mode train`
同域最终 test：`launcher --mode eval --eval_subset test`
如需一条龙：`launcher --mode all --eval_subset test`

5. 结果目录组织固定到同一个 run 下
新的 run 目录内应形成如下稳定结构：
WDM3D/runs/ours_wnet_128_split1000_val51_test200_seed42/

训练指令
```
$Env:MAMBA_ROOT_PREFIX = "D:\micromamba\root"
$Env:MAMBA_EXE = "D:\micromamba\bin\micromamba.exe"
& $Env:MAMBA_EXE shell hook -s powershell | Out-String | Invoke-Expression
micromamba activate D:\micromamba\root\envs\fastwdm3d
cd D:\Learing_TANG\TaskInpainting\fastWDM3D-main\fastWDM3D-main

python WDM3D\run_david.py --mode train --model ours_wnet_128 --data_dir "D:\Learing_TANG\TaskInpainting\ASNR-MICCAI-BraTS2023-Local-Synthesis-Challenge-Training" --batch_size 1 --lr 2e-5 --diffusion_steps 2 --gpu 0 --num_workers 0 --split_manifest "WDM3D\David_eval\split1000_val51_test200\train_val_test_split_david.json" --val_interval 5000 --max_val_batches 0 --experiment_label ours_wnet_128_split1000_val51_test200_seed42
```
