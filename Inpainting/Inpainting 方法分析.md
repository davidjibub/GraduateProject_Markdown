# Diffusion
## unconditional diffusion

Input
- 纯噪声

Output
- 整张图

学习到 $P(x_0)$ -> ==整个图像的真实数据分布==

> 什么样的东西**长得像真实的** brain MRI


## conditional diffusion

以 Inpainting diffusion 为例

Input
- 已知周围区域（有缺失区域mask）

学习到 $P(x_{missing} | x_{known},mask)$ -> ==已知周围区域和mask，缺失区域长什么样==

> 给定有缺失区域的 brian MRI影像，如何填充缺失的区域，才能使其**和周围一致**，并且**长得像真实的** brain MRI


# Inpainting

## 局部信息传入

> 局部 Inpainting 如何将周围的结构信息传进去？

route A：输入
- Train
	- voided image + mask + time step 直接送进Unet，让model学习到  $P(x_{missing} | x_{known},mask)$ 
	eg： Palette，WDM3D

route B：采样
- Train
	- model 本身是unconditional diffusion，学习到 $P(x_0)$
- Sample
	- 在 reverse 时每一时间步都将生成的图像中的已知区域换为已知区域（同noise水平），相当于只让noise在mask内自由生成
	eg：Repaint

route C：Loss
- 强制 model 利用周围上下文恢复局部
	不再是全局loss，而是对 mask，边界环带... 进行重点计算

## 2022 Repaint : Inpainting using Denoising Diffusion Probabilistic Models

^56ec7b

> 论文：[[2201.09865] RePaint: Inpainting using Denoising Diffusion Probabilistic Models](https://arxiv.org/abs/2201.09865)

### Train Phase：

	unconditional DDPM

### Sample Phase

需要将 voided image（ground truth）加噪到**同一time step水平（逐级同步加nosie）**
对于 Sample 得到的图像只保留mask区域内的
$$x_{t-1} =(1-m) ⊙ x_{t-1}^{known}+m⊙x_{t-1}^{unknown}$$

![[Inpainting-1778036856776.webp|506]]

model 预测 $x_{t-1}$ 只依靠 $x_t$
故：只要将 $x_t$ 已知区域修复值与真实观测一致且噪声级正确

> 将条件信息（voided image）写进当前状态 xt

## 2022 Paltte：Image-to-Image Diffusion Models
> 论文：[2111.05826v1](https://arxiv.org/pdf/2111.05826v1)

$x$：voided image，挖去mask后洞内填充高斯noise
	**x 不随 diffusion step t 做逐级加噪**
	
$$x=(1−M)⊙y+M⊙η,η∼N(0,I)$$
$y$：完整真图
	**Palette 的扩散过程只发生在目标图 y 上**，也就是完整真图
	
$$  
\tilde{y}_t = \sqrt{\gamma_t}y + \sqrt{1-\gamma_t}\epsilon  
$$

### Train Phase
![[Inpainting-1778039518175.webp]]

1. **Forward：**

$$y0 \rightarrow y1 \rightarrow... \rightarrow yt \rightarrow ...   \rightarrow yT$$

2. **Reverse:**

$$y0 \leftarrow y1 \leftarrow... \leftarrow yt \leftarrow ...   \leftarrow yT$$

每个 time step，输入 Unet 的为：
$$  
\begin{matrix}  
x & y_t & timestep \\  
\downarrow & \downarrow & \downarrow \\  
\operatorname{concat}(x,y_t) & & t \\  
\searrow & & \swarrow \\  
& \operatorname{UNet} & \\  
& \downarrow & \\  
& f_\theta(x,y_t,t) &  
\end{matrix}  
$$  
Loss
$$  
\mathcal{L}  
=  
\left\|  
m \odot f_\theta(x,y_t,t) - \varepsilon  
\right\|^2  
$$


model 学习到的是
$$  
\epsilon_\theta(x,\tilde{y}_t,\gamma_t) \approx \epsilon  
$$

### Sample Phase

与 Train Phase 相同，x 仅对mask内施加固定的noise

$$  
\begin{matrix}  
x & y_t & timestep \\  
\downarrow & \downarrow & \downarrow \\  
\operatorname{concat}(x,y_t) & & t \\  
\searrow & & \swarrow \\  
& \operatorname{UNet} & \\  
& \downarrow & \\  
& f_\theta(x,y_t,t) &  
\end{matrix}  
$$

### 原理解析

> Platte 为何能让model学会利用周围的结构？

#### 1. Concat

- 通过concat解决信息的可用性，让mask修复有上下文信息能用
	==让model再同一前向传播中能同时访问”证据“与”当前待恢复状态“==

#### 2. Network

- ==**Unet** : 侧重局部 -> 边缘连续，形状补全==
	conv的感受野，使得mask能看到周围的结构

- ==**Self-attention**：侧重全局 -> 远处对称结构，全局语义==
	使得mask内部能与远处相关区域直接建立联系

#### 3. Noise

- 在不同 noise level 中学习如何从 noise target中恢复正确方向


# 2025 Hierarchical Diffusion Framework for Pseudo-Healthy Brain MRI Inpainting with Enhanced 3D Consistency
> 论文：[Hierarchical Diffusion Framework for Pseudo-Healthy Brain MRI Inpainting with Enhanced 3D Consistency](https://arxiv.org/html/2507.17911)

