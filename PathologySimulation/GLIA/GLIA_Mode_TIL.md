# Inverse.py 反演肿瘤起点与生长参数

### 输入GLIA
```
patient_seg
altas_seg
altas_t1
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


#### Step3：reaction PDE
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
$$\frac{∂c}{∂t}​=∇⋅(kappa*K_{atlas​}(x)*∇c)+rho*R_{atlas}​(x)*c*(1−c)$$
or $$\frac{∂c}{∂t}​=∇⋅(K(x)*∇c)+R​(x)*c*(1−c)$$


  
3. 从当前的`c0(x)`开始，使用当前的`kappa`，`rho`生长，时间步`nt_inv = 40`，间隔`dt_inv = 0.025`，得到t=1时刻预测得到的肿瘤密度分布图`c1_pred(x)`

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
对全脑进行PDE预测生长烟花，但是仅在 observation operator `O`区域计算loss
$$
    min_{p,rho,kappa}  \frac{1}{2} ||O*c_1^{pred} - d1|| + \beta||p||
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
    核心：ED不确定tumor cell intensity故不强行进行你和，否则model会强制使得 c1_pred覆盖整个区域
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

#### Step5：更新参数
更新 `kappa`,`rho`,`p`
```
p:决定TIL在哪里
rho：决定局部增殖
kappa：决定空间浸润
```