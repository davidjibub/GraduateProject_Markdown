---
type:
  - project
  - thesis
status: done
thesis: "[[GLIA_Mode_General]]"
module: 对GLIA TIL模式的解析
tags:
  - thesis
  - project
topic:
  - pathology simulation
  - github code based
created: 2026-04-28
---

# Inverse.py 反演肿瘤起点与生长参数

### 输入GLIA
```
patient_seg
altas_seg

```
由`patient_seg`提供
1. 当前肿瘤观测值`d1`
2. Observation mask `O`
3. 候选 Gaussian TIL (tumor initial location) `φ(x)`

由`altas_seg`提供
1. K_altas(x)
2. R_altas(x)

### Inverse Pipeline
#### Step1： 参数准备
由`patient_seg`提供
1. 候选 Gaussian TIL (tumor initial location) `φ(x)`
    依据 Gaussian center确定候选中点，再结合tissue filter，smoothing，5*sigma截断，最大值归一化
    - 代码里如果 WM > 0.1 或 GM > 0.1 且 VT < 0.8，则 filter = 1，否则 filter = 0
    
    TIL 候选起点选定规则：
    PlanA
    1. 在规则网格上扫描候选中心
    2. 每隔固定space放一个候选中心，网格分辨率`n`
    3. 候选中心半径范围`sigma`内，超`gaussian_volume_fraction=90%`体素满足大于阈值`threshold_data_driven`，该中心保留为 Gaussian center

    PlanB
    直接提供pvec_path/phi，读文件中的 centers 作为Gaussian center

<font color=purple>超参数</font>：
```
n
sigma
gaussian_volume_fraction
threshold_data_driven
```
<font color=purple>依赖文件</font>：
```
patient_seg
```

#### Step2：初始肿瘤浓度
使用`p`作为权重组合候选种子
$$c_0(x) = \sum_{i=1}^{n} p_i*φ_i(x)$$
or
$$
c(0)=Φp
$$
待优化未知数为：
$$
x = (p, \rho, \kappa)
$$


#### Step3：reaction PDE （forward）
由`altas_seg`提供生长肿瘤的基准图像
1. K_altas(x), R_altas(x)：肿瘤开始生长的基准图，推得K(x), R(x)
    R(x) = rho * R_altas(x)
    K(x) = kappa * K_altas(x)
    默认模式中K_altas(x), R_altas(x) = WM(x), 即默认肿瘤只能在WM中生长 
    初始值`rho = 8`，`kappa = 0.01`
    ```
    atlas_seg
        ↓ splitSegmentation
    WM(x), GM(x), VT(x), CSF(x)
        ↓ 默认缩放
    K(x) = 0.01 * WM(x) + 0 * GM(x) + possible CSF term
    R(x) = 8    * WM(x) + 0 * GM(x) + possible CSF term
    ```
2. PDE：
[[GLIA PDE +elasticity]]
$$\frac{∂c}{∂t}​=∇⋅(kappa*K_{atlas​}(x)*∇c)+rho*R_{atlas}​(x)*c*(1−c)$$
or $$\frac{∂c}{∂t}​=∇⋅(K(x)*∇c)+R​(x)*c*(1−c)$$


  
4. 从当前的`c0(x)`开始，使用当前的`kappa`，`rho`生长，时间步`nt_inv = 40`，间隔`dt_inv = 0.025`，得到t=1时刻预测得到的肿瘤密度分布图`c1_pred(x)`

<font color=purple>超参数</font>：
```
nt_inv
dt_inv
```
<font color=purple>依赖文件</font>：
```
altas_seg
```
<font color=purple>迭代参数</font>：
```
kappa
rho
p
```

#### Step4：计算loss
对全脑进行PDE预测生长烟花，但是仅在 observation operator[^1] `O`区域计算loss
$$
    min_{p,rho,kappa}  \frac{1}{2} ||O*c_1^{pred}(p,rho,kappa) - d1|| + \beta||p||
$$
```
min over p, rho, kappa:
    1/2 || O c(1; Phi(p), rho, kappa) - d1 ||^2 + regularization
subject to:
    ∂c/∂t = diffusion(kappa, tissue) + reaction(rho, tissue)
    c(0) = Phi(p)
    p is sparse
```
observation operator `o`生成路径
1. 默认阈值
obs = 1[data > thr]
2. 带权重观测
obs = 1[TC] +lamba* 1[B/WT]
    默认配置`lamba=1`，即
    ```
    O(x) = 1, x in tumor core  
           0, x in ED
           1, x in else 
    ```
    核心：ED不确定tumor cell intensity故不强行进行拟合，否则model会强制使得 c1_pred覆盖整个区域
    
其作用不是比较全脑 MRI 灰度，而是在指定观测区域/权重下比较 predicted tumor concentration 与 d1
具体 ED 是否参与、权重是多少，由 obs_lambda / threshold / segmentation-derived data 配置决定

<font color=purple>超参数</font>：
```
lamba
```
<font color=purple>依赖文件</font>：
```
patient_seg -> d1
```
<font color=purple>迭代参数</font>：
```
kappa
rho
p
```

#### Step5：反向传播
基于计算的loss进行参数迭代，得到更新后的参数值

##### PDE-constrained optimization
限制状态变量 c 不能随便取，必须满足 PDE

普通优化问题是：
$$
\min_x J(x)
$$
比如深度学习里：
$$
\min_\theta L(f_\theta(X), Y)
$$
其中 $\theta$ 是神经网络参数。

PDE-constrained optimization 是：
$$
\min_m J(u(m), m)
$$
subject to：
$$
F(u, m) = 0
$$
这里：
- $m$：你要反演的参数，比如 GLIA 里的 $p, \rho, \kappa, \gamma$
- $u$：PDE 状态变量，比如肿瘤浓度 $c(t)$、位移场、脑组织分布
- $F(u, m)=0$：PDE 约束，比如 reaction-diffusion 方程、mass-effect 力学方程
- $J$：loss / objective，比如预测肿瘤图和病人分割图的差异

```
在 GLIA TIL 中，它的作用是：
1. 定义问题 
不是任意找一张肿瘤图拟合病人数据，而是找一组p,ρ,κ，使 PDE 生长出的 c(1)拟合 d1

2. 连接参数和 loss
参数 p,ρ,κ 通过 PDE solver 影响 c(1)，再影响 loss

3. 提供梯度计算理论
因为 c 受 PDE 约束，所以要用 adjoint PDE / sensitivity equation 得到 ∇J

4. 让 TAO 能求解问题  
GLIA 把 PDE-constrained optimization 化成 TAO 能接受的形式：
    x↦J(x),∇J(x)
然后 TAO 负责更新 x
```


 TAO
 >PDE-constrained optimization = 问题建模框架  
> TAO = 求解这个优化问题的优化器
 
PETSc 是高性能科学计算库，TAO 是 PETSc 里的优化器模块，全称可以理解为 **Toolkit for Advanced Optimization**
TAO 本身不懂肿瘤、不懂 MRI、不懂 GLIA。它只懂这种通用接口：

```
你给我一个参数向量 x
我需要你告诉我：
1. 当前 objective value 是多少
2. 当前 gradient 是多少
3. 如果需要，Hessian 或 Hessian-vector product 是什么
然后我帮你决定下一步 x 应该怎么更新
```

即：TAO ➡ 拿到 J(x) 和 ∇J(x) 后做参数更新。
	TAO 不负责解 PDE。TAO 只负责优化参数
	
```
TAO:
给你一组参数 x = [p, rho, kappa]，
请告诉我当前 loss 和 gradient。

GLIA:
好的。
我先用 p 生成 c(0)。
然后用 rho, kappa 解 PDE 得到 c(1)。
再计算 loss = 1/2 ||O c(1)-d1||^2 + beta R(p)。
再解 adjoint PDE 得到 gradient。
返回给你。

TAO:
我根据 loss 和 gradient 决定下一步参数。
比如用 quasi-Newton / line search。
新的参数是 x_new。

GLIA:
继续用 x_new 解 PDE。
```


#### Step6：更新参数
更新 `kappa`,`rho`,`p`
```
p:决定TIL在哪里
rho：决定局部增殖
kappa：决定空间浸润
```


# TIL Step5 时间步详细计算顺序

$$
[p, \rho, \kappa] \rightarrow \Phi p \rightarrow PDEsolve \rightarrow c_{final} \rightarrow loss
$$
> 第 k 次TAO迭代流程
> n: PDE推演的时间步 0: 初始时刻   1：最终时刻

### Step 1：TAO 给当前参数
$$
x_k = (p_k, \rho_k, \kappa_k)
$$
### Step 2：GLIA 构造初始条件
$$
c_0 = \Phi p_k
$$
### Step 3：GLIA 解 forward PDE

$$
\frac{\partial c}{\partial t}
=
\nabla \cdot (\kappa_k \nabla c)
+
\rho_k c(1-c)
$$

得到：
$c_k(1)$：第 k 次TAO迭代下得到的最终时刻的肿瘤密度分布图

#类比神经网络

> Step 3：GLIA 解 forward PDE 可以理解为 [神经网络] 的前向传播

每个时间点可以当成神经网络的一层
$$
 p_0 → c_0 →c_1→...c_n→c_{n+1}...→c_N→loss
$$
$$
\begin{aligned}
c_{n+1} = A(c_n; p, \rho, \kappa) \\
c_n = A(t_n,x,y,z)
\end{aligned}
$$
这里的 $A$ 其实就是 PDE 差分形式

### Step 4：计算 residual

$$
r_k = O c_k(1) - d_1
$$

### Step 5：计算 objective

$$
J_k
=
\frac{1}{2}\lVert r_k \rVert^2
+
\beta R(p_k)
$$

### Step 6：解 adjoint PDE

从终点误差 $r_k$ 反向传播误差信息，得到伴随变量 $\lambda(t)$
终点条件（符号可能因具体推导约定有正负差异）：
$$
\lambda(1) = O^T(Oc(1) - d_1)
$$
伴随变量 $\lambda(t,x)$ ：
	代表 loss对 PDE状态变量 $c(t,x)$ 的敏感性
	$$
	\lambda_n = \frac{\partial J}{\partial c_n}
	$$
	可类比于[神经网络]： Loss对 L 层激活值的敏感性
	$$
	\frac{\partial Loss}{\partial z_l}
	$$

#类比神经网络

> Step 3：GLIA 解 forward PDE 可以理解为 [神经网络] 的前向传播
> Setp 6：GLIA 解 adjoint PDE 可以理解 [神经网络] 的反向传播

终点 loss 是：  
$$  
\begin{aligned}
J = \frac{1}{2}\lVert Oc_N - d_1 \rVert^2  \\
\lambda_N  
=  
\frac{\partial J}{\partial c_N}  
=  
O^T(Oc_N - d_1)
\end{aligned}
$$
进行反向传播得到每层的伴随变量
$$
\begin{aligned}
c_{n+1} = A(c_n; p, \rho, \kappa) \\
\lambda_n =  
\left(  
\frac{\partial A}{\partial c_n}  
\right)^T  
\lambda_{n+1}
\\ \\ or \\
\lambda_n =  
\left(  
\frac{\partial c_{n+1}}{\partial c_n}  
\right)^T  
\lambda_{n+1}

\end{aligned}
$$
即由终点状态反向传播得到每个时间点的伴随变量
$$
loss → \lambda_N →\lambda_{N-1} →...\lambda_{n+1} →\lambda_n ...
$$
==**注意**：
这时候得到的 $\lambda_n$ 是得到参数梯度的中间变量最终需要的是  $p,\rho,\kappa$ 参数==

### Step 7：由伴随变量计算梯度

1. 对 $p$ 梯度：
$$
\begin{aligned}
\frac{\partial J}{\partial p}  
=  
\Phi^T  
\frac{\partial J}{\partial c_0}  
+  
\beta  
\frac{\partial R}{\partial p} \\
\nabla_p J  
=  
\Phi^T \lambda_0  
+  
\beta \nabla R(p)


\end{aligned}
$$

2. 对 $\rho , \kappa$ 梯度：
$ρ$ 控制 reaction 项， $κ$ 控制 diffusion 项
$$
\frac{\partial c}{\partial t}=\nabla \cdot (\kappa_k \nabla c)
+
\rho_k c(1-c)
$$
==它们的梯度来自整个时间区间上的累计影响==

$$  
\frac{\partial J}{\partial \rho}  
=  
\int_0^1  
\lambda(t,x)  
\frac{\partial}{\partial \rho}  
\left[  
\rho c(t,x)(1-c(t,x))  
\right]  
\, dx\, dt  
$$
代入化简后
$$  
\frac{\partial J}{\partial \rho}  
\sim  
\int_0^1  
\lambda(t,x)c(t,x)(1-c(t,x))  
\, dx\, dt  
$$
对 $\kappa$ 也是类似，只不过==涉及空间梯度项==：
$$  
\frac{\partial J}{\partial \kappa}  
\sim  
\int_0^1  
\nabla c(t,x) \cdot \nabla \lambda(t,x)  
\, dx\, dt  
$$


最终得到：

$$
\begin{aligned}
\nabla_p J \\
\nabla_\rho J \\
\nabla_\kappa J
\end{aligned}
$$

### Step 8：TAO 更新参数

$$
x_{k+1} = x_k + \alpha_k s_k
$$
其中 $s_k$ 由 TAO 的优化方法决定
1. 如果是最简单的梯度下降：
$$
s_k = -\nabla J(x_k)
$$
2. 如果是 quasi-Newton：
$$
s_k = -B_k^{-1}\nabla J(x_k)
$$
3. 如果是 Newton：
$$
s_k = -H_k^{-1}\nabla J(x_k)
$$

### （Step1：TAO 给当前参数）

将上一次迭代求得的 $x_{k+1}=(p_{k+1}, \rho_{k+1}, \kappa_{k+1})$ 作为下一次的初值 $x_k=(p_k, \rho_k, \kappa_k)$ 

==注意区分迭代次数 $k$ 是不断递增，所以是 $k$ ➡ $k+1$==
==而由最终时刻 $n$ 反向传播的时间是递减的，所以是 $\lambda_{n+1}$ ➡ $\lambda_{n}$==





[^1]: 如何根据loss优化参数，迭代遍历还是反向传播类似？
	
