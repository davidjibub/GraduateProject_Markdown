![[Pasted image 20260506102527.png|637|626x443]]
![[WDM3D 方法分析-1778116606335.png]]

# 2024亚军 Brain Tumor Removing and Missing Modality Generation using 3D WDM
>论文：[2411.04630](https://arxiv.org/pdf/2411.04630)

文章设置了 4 个inpainting版本

## WDM3D Framework
> github：[pfriedri/wdm-3d: [DGM4MICCAI'24] PyTorch implementation for "WDM: 3D Wavelet Diffusion Models for High-Resolution Medical Image Synthesis"](https://github.com/pfriedri/wdm-3d)

![[WDM3D-1778044966506.png]]

### WDM

#### 1. DWT

![[WDM3D 方法分析-1778129771411.png]]

原始 3D 图像尺寸
`y_gt, y_masked, y_mask: [B, 1, H, W, D]`

> 记`h = H/2, w = W/2, d = D/2`

对`y_gt, y_masked, y_mask` 分别进行
1. 经过一次 3D Haar DWT 后，每个子带空间尺寸减半
	`LLL, LLH, LHL, LHH, HLL, HLH, HHL, HHH:
	`[B, 1, H/2, W/2, D/2]`

2. 把 8 个子带沿 channel 维拼接
	`x_dwt: [B, 8, h, w, d]` 

3. 对每个子带除以√8，进行幅值归一化
	`x_dwt_norm[c, i, j, k] = x_dwt[c, i, j, k] / √8`
	
即
```
y_gt      : [B,1,H,W,D] → DWT → [B,8,h,w,d] → ÷√8 → x_gt_dwt
y_masked  : [B,1,H,W,D] → DWT → [B,8,h,w,d] → ÷√8 → x_masked_dwt
y_mask    : [B,1,H,W,D] → DWT → [B,8,h,w,d] → ÷√8 → x_mask_dwt
```

示意
```
完整图像 y_gt
[B,1,H,W,D]
   │
   ▼
DWT_3D
   │
   ▼
8 个子带 concat
[B,8,h,w,d]
   │
   ▼
÷√8
   │
   ▼
x_gt_dwt
[B,8,h,w,d]
```

#### 2. concat

将DWT后的结果拼成 24ch 输入 U-Net
`model_input = concat(x_t, x_masked_dwt, x_mask_dwt)`
```
x_t          : [B, 8, h,w,d]
x_masked_dwt : [B, 8, h,w,d]
x_mask_dwt   : [B, 8, h,w,d]
--------------------------------
model_input  : [B,24, h,w,d]
```

示意
```
x_t             [B, 8,h,w,d] ┐
x_masked_dwt    [B, 8,h,w,d] ├─ concat channel ─→ [B,24,h,w,d] ─→ WavUNet
x_mask_dwt      [B, 8,h,w,d] ┘
timestep t      [B] ─────────────────────────────→ time embedding ─┘
```
注意：`t` 不拼到 24ch 里。`t` 是单独进入 timestep embedding，然后注入 ResBlock

##### 2.1 time embedding

`t` 就是 diffusion 的时间步编号

- 如果这个仓库 fastWDM3D 默认 `diffusion_steps=2`，那训练时采样到的 `t` 可能是 `t = 0 或 1`
- 若扩散步数是 `T`，那 `t` 就从`0, 1, 2, ..., T-1` 里随机采样一个


```
离散时间步 t（例如 0,1）
    ↓
sinusoidal timestep embedding
    ↓
得到一个长度为 64 的向量
    ↓
Linear(64→256)
    ↓
SiLU
    ↓
Linear(256→256)
    ↓
得到 t_emb / emb（长度 256）
    ↓
送入每个 ResBlock
    ↓
每个 ResBlock 再用一个小线性层，把 256 维 emb 映射到当前 block 所需通道数
    ↓
reshape 成 [B, C, 1, 1, 1]
    ↓
加到该 ResBlock 的 feature map 上
```

1. sinusoidal timestep embedding
	这是把一个“整数时间步”变成一个“连续向量表示”的方法
	可以把它类比成：
	`把“第几步”这个离散编号,翻译成一个神经网络更容易用的实数向量`
	它和 Transformer 里的 positional embedding 很像，通常形式是:
	`[cos(ω1 t), cos(ω2 t), ..., sin(ω1 t), sin(ω2 t), ...]`

	也就是说：
	- 不同频率的正弦和余弦共同编码 `t`
	- 这样相近的时间步会有相近但不相同的表示
	- 网络更容易学会“不同噪声等级之间的连续变化关系”

这个 `sinusoidal timestep embedding` 一般是一个**固定公式**，不是学习到的查表 embedding
- ==本身通常没有可训练参数==  
- 只是把整数 t 变成一个固定规则生成的向量，对batch而言 `[B, 64]`

2. Linear
	`sinusoidal timestep embedding` 输出`[B, 64]`
	第一层线性层
	- 输入 `[B, 64]`
	- 权重矩阵大小`[64, 256]`
	- 输出`[B, 256]`

这层**有可训练参数**：
- 权重：`64 × 256`
- 偏置：`256`

3. SiLU
	`SiLU(x) = x · sigmoid(x)`

4. Linear
	得`emb: [B, 256]`

- ==sinusoidal embedding：负责“把编号变成向量”==
- ==后面的两层 Linear：负责“把这个向量学成更适合当前任务的表示”==

#### 3. Unet输出

U-Net 输出：
`model_output: [B,8,h,w,d]` 

表示：
```
预测的 clean wavelet 子带，且仍然是 normalized wavelet coefficient
```
即
`model_output ≈ DWT(y_gt) / √8`

##### 3.1 Resblock

这部分只是提特征，不改变空间尺寸
```
h [B,64,S,S,S]  
↓ ResBlock 64→64  
h [B,64,S,S,S]  
↓ ResBlock 64→64  
h [B,64,S,S,S]  
↓ ResBlock 64→64  
h [B,64,S,S,S]  
↓ ResBlock 64→64  
h [B,64,S,S,S]
```
![[WDM3D 方法分析-1778122769510.png]]

##### 3.2  Wavelet Down ResBlock

ResBlock 内部带 wavelet downsample
```
输入 h
[B, C, S, S, S]
      │
      ▼
GroupNorm → SiLU → Conv3D 3×3×3
      │
      ▼
DWT_3D downsample
      │
      ├── LLL / 3.0 作为低频主路径
      │       shape: [B, C, S/2, S/2, S/2]
      │
      └── LLH,LHL,LHH,HLL,HLH,HHL,HHH
              7 个高频子带，保存给 decoder
              每个 shape: [B, C, S/2, S/2, S/2]
```

```
Wavelet Down ResBlock =
    普通 ResBlock 的计算
    + 在 block 内部用 DWT 做降采样
    + 顺便保存 7 个高频子带作为 decoder skip
```

```
输出给下一层的主路径:
    h_low: [B, C, S/2, S/2, S/2]

保存给 decoder 的 skip:
    skip_high = (LLH, LHL, LHH, HLL, HLH, HHL, HHH)
```

Wavelet Down ResBlock 用 DWT 完成空间下采样；  
- 主路径 channel 数不因为 DWT 本身增加；  
- 空间尺寸减半；  
- 高频信息不丢掉，而是作为 skip 保存。

```
普通下采样：
    只留下一个下采样 feature，细节可能被压掉

Wavelet down：
    低频继续往下
    高频细节保存给 decoder
```

##### 3.3 Progressive WaveletDownsample

把“输入图像/输入特征金字塔”也同步下采样到当前 encoder 分辨率，  
再用 Conv3D 投影成和主路径 h 一样的通道数，  
然后加到主路径上
```
每往下一层，不只依赖上一层 feature，
还把原始输入信息下采样后补充进来。
```


##### 3.4 多次DWT意义

第一次 DWT：
    分离较细尺度的低频/高频信息

第二次 DWT：
    在更粗尺度上继续分离低频/高频信息

第三次 DWT：
    在更抽象、更低分辨率的 feature 上继续做多尺度分解


#### 4. Loss 

##### 训练时：
```
model_output *= √8
model_output_idwt = IDWT_3D(model_output)
```

`model_output_idwt` 和 `y_gt` 算 L1 loss

```
pred_x0_dwt_norm
[B,8,h,w,d]
   │
   ▼
×√8
[B,8,h,w,d]
   │
   ▼
IDWT_3D
   │
   ▼
pred_image
[B,1,H,W,D]
   │
   ├──────────────→ L_full = L1(pred_image, y_gt)
   │
   └──────────────→ L_masked = L1(pred_image[mask==1], y_gt[mask==1])
```

1. 全局Loss：整幅预测图像和完整真实图像，每个 voxel 都比较一次，取平均绝对误差
```
L_full = mean(|pred_image - y_gt|)
```
2. mask Loss：只在被挖掉 / voided 的区域里比较预测值和真实值，非 mask 区域完全不参与这一项
```
L_masked =
1 / |M| · Σ_{p∈M} | pred_image(p) - y_gt(p) |
```

==这会让 mask 区域被“重复关注”==：
既参与 `L_full`，又参与 `L_masked`。所以 mask 区域的误差权重大于非 mask 区域

##### 采样时：

```
A. 训练 loss 分支：

pred_x0_dwt_norm [B,8,h,w,d]
    ↓ ×√8
wavelet coeff [B,8,h,w,d]
    ↓ IDWT
pred_image [B,1,H,W,D]
    ↓
L_full + L_masked


B. 采样 process_xstart 分支：

pred_x0_dwt_norm [B,8,h,w,d]
    ↓ ×√8
image [B,1,H,W,D]
    ↓ clamp[-1,1]
image_clamped [B,1,H,W,D]
    ↓ DWT
wavelet_clamped [B,8,h,w,d]
    ↓ ÷√8
pred_x0_dwt_clamped [B,8,h,w,d]
    ↓
用于 q(x_{t-1} | x_t, pred_x0)，继续反向采样
```



## 1. Default (Repaint)

#### Train Phase：

unconditional training
![[WDM3D-1778045545866.png]]

#### Sample Phase：

Conditional sampling
- Input：xt 是 wavelet coeff
- Sample conditional：$$x_{t-1} =(1-m) ⊙ x_{t-1}^{known}+m⊙x_{t-1}^{unknown}$$
- Loss：全局Loss

可参考 [[Inpainting 方法分析#^56ec7b]]

![[WDM3D-1778045579475.png]]

## 2. Default conditional

#### Train Phase:
- Input：xt 是 wavelet coeff
	- mask 不加噪声

$$  
\begin{matrix}  
x & mask(hlth+unhlth) & timestep \\  
\downarrow & \downarrow & \downarrow \\  
\operatorname{concat}(x,y_t) & & t \\  
\searrow & & \swarrow \\  
& \operatorname{UNet} & \\  
& \downarrow & \\  
& f_\theta(x,y_t,t) &  
\end{matrix}  
$$

- Loss：全局 MSE + mask 内部MSE（healthy+unhealthy）
$$
\begin{aligned}
Loss= MSE(\tilde{x},x) + \lambda MSE(\tilde{x_{roi}},x_{roi})\\
=\left\|  \tilde{x}_0 - x_0  \right\|_2^2  +  \lambda_1  \left\|  \tilde{x}_0 \cdot m - x_0 \cdot m  \right\|_2^2
\end{aligned}
$$
![[WDM3D-1778045886748.png]]

## 3. Always known

#### Train Phase:

- Input：只对mask内加噪声，mask外为干净（未加noise）的voided image
$$x_{t} =(1-m) ⊙ x_{t}^{known}+m⊙\tilde{x_{t}}$$
	![[WDM3D-1778047200031.png|158]]

$$  
\begin{matrix}  
x & mask(hlth+unhlth) & timestep \\  
\downarrow & \downarrow & \downarrow \\  
\operatorname{concat}(x,y_t) & & t \\  
\searrow & & \swarrow \\  
& \operatorname{UNet} & \\  
& \downarrow & \\  
& f_\theta(x,y_t,t) &  
\end{matrix}  
$$

- Loss：全局 MSE + mask 内部MSE（healthy）
$$
\begin{aligned}
Loss= MSE(\tilde{x},x) + \lambda MSE(\tilde{x_{healthy}},x_{healthy})\\
=\left\|  \tilde{x}_0 - x_0  \right\|_2^2  +  \lambda_1  \left\|  \tilde{x}_0 \cdot m_h - x_0 \cdot m_h  \right\|_2^2
\end{aligned}
$$
![[WDM3D-1778046763277.png]]
==model 不再浪费容量重建本就已知的区域（voided image）而是集中能力重建ROI==

## 4. Always known healthy (best)

#### Train Phase：

- Input ：xt只对healthy mask加噪，unhealthy mask保持为0

	![[WDM3D-1778047303688.png|183]]

$$  
\begin{matrix}  
x & mask(hlth+unhlth) & timestep \\  
\downarrow & \downarrow & \downarrow \\  
\operatorname{concat}(x,y_t) & & t \\  
\searrow & & \swarrow \\  
& \operatorname{UNet} & \\  
& \downarrow & \\  
& f_\theta(x,y_t,t) &  
\end{matrix}  
$$

只对healthy mask加噪，==让model只能根据健康组织修复healthy mask==
model 不学会生成肿瘤，而是只学 healthy mask修复
