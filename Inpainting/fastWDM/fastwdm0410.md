# 4.10 BraTS 1200/51 single split 训练 + 验证损失选最优 checkpoint
## 训练
用 BraTS2023 Local Synthesis Challenge Training 数据集训练 FastWDM / ours_wnet_128
BraTS 数据划成了 1200 个训练病例和 51 个验证病例来训练与验证 
(读取固定好的 split manifest：`train_val_split_david.json`,`D:\Learing_TANG\TaskInpainting\fastWDM3D-main\fastWDM3D-main\WDM3D\David_eval\split1200_51\train_val_split_david.json`)
```
python WDM3D\run_david.py --mode train --model ours_wnet_128 --data_dir "D:\Learing_TANG\TaskInpainting\ASNR-MICCAI-BraTS2023-Local-Synthesis-Challenge-Training" --batch_size 1 --lr 2e-5 --diffusion_steps 2 --gpu 0 --num_workers 0 --split_manifest "WDM3D\David_eval\split1200_51\train_val_split_david.json" --val_interval 5000 --max_val_batches 0 --experiment_label ours_wnet_128_single_split_seed42
```

## 训练结果
这次训练已经完成并产生了规范化结果
(当前最优模型在：`best_checkpoint_meta.json`,`D:\Learing_TANG\TaskInpainting\fastWDM3D-main\fastWDM3D-main\WDM3D\runs\ours_wnet_128_single_split_seed42\checkpoints\best_checkpoint_meta.json`)
记录的是：
`
best_step = 460000
best_val_loss = 0.1430448524507822
best checkpoint = brats3dimage460000.pt
`

### 训练结果分析

```
D:\micromamba\root\envs\fastwdm3d\python.exe WDM3D\David_eval\training_analysis_david\plot_training_curves_david.py --run_dir "D:\Learing_TANG\TaskInpainting\fastWDM3D-main\fastWDM3D-main\WDM3D\runs\ours_wnet_128_single_split_seed42"
```

- 模型有没有学会
- 模型什么时候学到最好
- 后期训练是在继续变好，还是开始过拟合/浪费时间

1. loss_rec_train_val.png
画训练 reconstruction loss 和验证 reconstruction loss。红色竖线是 best checkpoint，也就是你现在选中的最优模型步数
=> 模型收敛，best checkpoint 在 460000 step
![[loss_rec_train_val.png]]
2. loss_full_plus_masked_train_val.png
当前 best checkpoint 就是按 loss/val_rec_full_plus_masked 选出来的
注：在你当前这版代码里，val_rec 和 val_rec_full_plus_masked 实际上是一样的数值来源，所以这两条 validation 指标现在可以近似看成同一个 validation loss 的两个命名版本。也就是说，这两张图在 validation 部分的信息量很接近，但“full_plus_masked”那张更符合你现在的 best model 选择逻辑
![[loss_full_plus_masked_train_val.png]]

3. val_loss_zoom.png
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
![[val_loss_zoom.png]]


4. timing_metrics.png
看训练效率和瓶颈：
- time/load：每步花多少时间在数据加载
- time/forward：每步花多少时间在前向/计算
- time/total：每步总耗时
怎么看：
- time/load 很高：说明 IO 或 DataLoader 是瓶颈
- time/forward 很高：说明 GPU 计算是瓶颈
- time/total 逐渐升高：说明训练过程可能越来越慢，可能有资源问题
结论：
- time/load 很小
- time/forward 占比很高
- 所以当前瓶颈主要是计算，不是读数据
- 想提速，优先考虑模型计算、显存策略、batch、mixed precision，不需要优先怀疑 DataLoader
![[timing_metrics.png]]

5. schedule_indicator.png
训练过程状态指标，回答“训练过程有没有异常”，主要看：
- 是否稳定
- 是否突然异常跳变
- 是否出现训练调度行为异常
怎么看：
- 平稳波动：一般正常
- 突然尖峰/断崖：可能要回头检查采样步使用、训练状态、数值稳定性
![[schedule_indicator.png]]


## 对Philips私有数据集进行测试
生成结果在`D:\Learing_TANG\TaskInpainting\ASNR-Private-Testing-SampleResults\FastWDM\0423\sampling_output\best_checkpoint_by_val`
private test 的汇总指标是：
- SSIM mean = 0.620522
- PSNR mean = 17.183422
- MSE mean = 0.022207
```
D:\micromamba\root\envs\fastwdm3d\python.exe WDM3D\David_eval\validate_test_david.py --test_data_dir "D:\Learing_TANG\TaskInpainting\ASNR-Private-Testing" --checkpoint "D:\Learing_TANG\TaskInpainting\fastWDM3D-main\fastWDM3D-main\WDM3D\runs\ours_wnet_128_single_split_seed42\checkpoints\best_checkpoint_by_val.pt" --output_dir "D:\Learing_TANG\TaskInpainting\ASNR-Private-Testing-SampleResults\FastWDM\0423" --dataset_label private_test --model ours_wnet_128 --batch_size 1 --diffusion_steps 2 --gpu 0
```