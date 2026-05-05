---
type:
  - project
  - experiment
status: active
thesis: "[[GLIA_Mode_General]]"
module: 使用 fastwdm 完成训练及测试
tags:
  - project
  - experiment
topic:
  - github code based
  - inpainting
created: 2026-04-27
period: 2026-05-05
---

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

### 训练结果分析

绘图数据保存在 `D:\Learing_TANG\TaskInpainting\fastWDM3D-main\fastWDM3D-main\WDM3D\runs\ours_wnet_128_split1000_val51_test200_seed42\training_analysis_david`

1. loss_rec_train_val.png
画训练 reconstruction loss 和验证 reconstruction loss。红色竖线是 best checkpoint，也就是你现在选中的最优模型步数
=> 模型收敛，best checkpoint 在 325000 step
![[Inpainting/fastWDM/trainres_0427/loss_rec_train_val.png]]
2. val_loss_zoom.png
专门把 validation loss 后期放大来看。
为什么它重要：
- 大图里后期小波动不容易看清，但真正决定“该不该停训练”的往往就是后期走势。
怎么看：
- 如果红线前后差不多横着走，说明已经平台了。
- 如果红线后明显上升，说明继续训开始不划算，甚至过拟合。
- 如果红线后还在缓慢下降，说明还可以继续训。
- 你这次的结果是：
结论：
- best 在 460000，到 490000 只有非常轻微回升
- 所以更像是“已经接近训练上限，后面收益很小”，而不是“后面突然坏掉”
![[Inpainting/fastWDM/trainres_0427/val_loss_zoom.png]]
4. timing_metrics.png
看训练效率和瓶颈：
- time/load：每步花多少时间在数据加载
- time/forward：每步花多少时间在前向/计算
- time/total：每步总耗时
怎么看：
- time/load 很高：说明 IO 或 DataLoader 是瓶颈
- time/forward 很高：说明 GPU 计算是瓶颈
- time/total 逐渐升高：说明训练过程可能越来越慢，可能有资源问题

![[Inpainting/fastWDM/trainres_0427/timing_metrics.png]]
5. schedule_indicator.png
训练过程状态指标，回答“训练过程有没有异常”，主要看：
- 是否稳定
- 是否突然异常跳变
- 是否出现训练调度行为异常
怎么看：
- 平稳波动：一般正常
- 突然尖峰/断崖：可能要回头检查采样步使用、训练状态、数值稳定性
![[Inpainting/fastWDM/trainres_0427/schedule_indicator.png]]



## 测试

## 基于先前划分的测试集

基于现有训练结果，直接对 manifest 中预留的 test_ids=200 子集做采样评估，不改模型、不改参数，沿用当前默认“最佳验证权重”逻辑。结果默认写回本次训练 run 目录下的 same_domain_test_eval，与训练记录放在同一个实验目录中

- 使用训练根目录 `D:\Learing_TANG\TaskInpainting\ASNR-MICCAI-BraTS2023-Local-Synthesis-Challenge-Training`
- 使用三划分 manifest  
    `train_val_test_split_david.json`
- 使用训练 run  
    `ours_wnet_128_split1000_val51_test200_seed42`
- 默认评估权重为  
    `best_checkpoint_by_val.pt`

```
python WDM3D\David_eval\launch_single_split_david.py --mode eval --train_data_dir "D:\Learing_TANG\TaskInpainting\ASNR-MICCAI-BraTS2023-Local-Synthesis-Challenge-Training" --split_manifest "WDM3D\David_eval\split1000_val51_test200\train_val_test_split_david.json" --model ours_wnet_128 --experiment_label ours_wnet_128_split1000_val51_test200_seed42 --gpu 0 --eval_subset test

```

结果生成在 `D:\Learing_TANG\TaskInpainting\fastWDM3D-main\fastWDM3D-main\WDM3D\runs\ours_wnet_128_split1000_val51_test200_seed42\same_domain_test_eval`

- Brats MRI采样结果在 `sampling_output\best_checkpoint_by_val`
- 逐例采样指标（SSMI，PSNR，MSE） `same_domain_test_cases_david.csv`  `same_domain_test_summary_david.json`


这次同域 test 的汇总指标是：

- SSIM mean = 0.789818
- PSNR mean = 20.383899
- MSE mean = 0.011836


## 单例分析（适配 肿瘤生长模拟）

已经完成单病例采样，使用的是这次 1000/51/200 训练的最优权重：  best_checkpoint_by_val.pt  


最终结果已经放到你指定目录： `D:\Learing_TANG\GraduateProject\CodeProject\PathologySimulation\GLIA-master\patientdata\BraTS-GLI-00006-000\inpainting_volumes`

这次我是用原始的 BraTS-GLI-00006-000-t1n.nii.gz 和 BraTS-GLI-00006-000-mask.nii.gz 临时整理成 FastWDM 采样输入后跑的，采样本身耗时大约 8 分钟。因为这个病例目录没有 FastWDM 评估所需的 mask-healthy 真值对，所以这次只生成了采样结果，没有计算 SSIM / PSNR / MSE。

临时运行产物和摘要还在这里，后面如果你想复查命令或继续批量化会很方便：

- `D:\Learing_TANG\TaskInpainting\fastWDM3D-main\fastWDM3D-main\WDM3D\David_eval\tmp_single_case_fastwdm_output`

运行指令
```
D:\micromamba\root\envs\fastwdm3d\python.exe WDM3D\David_eval\validate_test_david.py --test_data_dir "D:\Learing_TANG\TaskInpainting\fastWDM3D-main\fastWDM3D-main\WDM3D\David_eval\tmp_single_case_fastwdm_input" --checkpoint "D:\Learing_TANG\TaskInpainting\fastWDM3D-main\fastWDM3D-main\WDM3D\runs\ours_wnet_128_split1000_val51_test200_seed42\checkpoints\best_checkpoint_by_val.pt" --output_dir "D:\Learing_TANG\TaskInpainting\fastWDM3D-main\fastWDM3D-main\WDM3D\David_eval\tmp_single_case_fastwdm_output" --dataset_label single_case_fastwdm --model ours_wnet_128 --batch_size 1 --diffusion_steps 2 --gpu 0

```