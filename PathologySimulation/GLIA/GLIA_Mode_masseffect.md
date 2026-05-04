---
type:
  - project
  - thesis
status: done
thesis: "[[GLIA_Mode_General]]"
module: 对GLIA masseffect模式的解析
tags:
  - thesis
  - project
topic:
  - pathology simulation
  - github code based
created: 2026-04-28
---
# Inverse_masseffect.py 考虑展位效应反演肿瘤起点与生长参数
### 输入GLIA
```
atlas_seg  
atlas_t1  
patient_seg  
patient_vt  
d0 初始肿瘤  
d1 终末肿瘤观测
```
代码不再只关心“肿瘤从哪里开始长到现在”，而是同时关心：
- 肿瘤如何生长  
- 肿瘤如何推动组织变形  
- atlas 组织如何被 transport 到接近 patient 状态

<font color=orange>关键改变：</font>
t1, seg, vt等文件更多的参与理解和输出形变后组织结构的重要输入
```
inverse_TIL:
    主要围绕 tumor concentration c 的反演
inverse_masseffect:
    同时围绕 tumor concentration c 和 tissue deformation / displacement 的反演
```

### Inverse masseffect Pipeline
#### Step1：参数准备
不再默认先生成一批 Gaussian TIL 候选，而是直接读取一个初始肿瘤 c0
<font color=orange>关键改变：</font>
我已经有 c0，d1
现在我要找一组生长/形变参数，使 c0 经过生长和 mass effect 后接近 d1
```
读取初始肿瘤 c0  
读取终末肿瘤 d1  
读取 atlas/patient 组织图  
准备 reaction-diffusion + elasticity 所需材料
```

#### Step2：初始肿瘤浓度
直接读取初始肿瘤文件`c0(x)`，不再通过`p`去组合Gaussian center得到`c0(x)`
<font color=orange>关键改变：</font>
反演对象从：`p + rho + kappa`
变成：`gamma + rho + kappa`
其中 `gamma` 是 mass-effect forcing factor，也就是控制肿瘤推动组织变形强度的参数
```
inverse_TIL:
    c0 是未知的，需要通过 p 反演出来
inverse_masseffect:
    c0 默认是已知的，直接作为 forward simulation 的初始条件
```
对比：TIL 待优化未知数为：
$$  
x = [p, \rho, \kappa]  
$$

mass effect 待优化未知数为：
$$x=[γ,ρ,κ]$$
- $\gamma$ 控制 tumor-induced force / mass effect 强度；  
- $\rho$ 控制肿瘤增殖；  
- $\kappa$ 控制肿瘤扩散

#### Step3：reaction PDE（forward）

>注意：Step3 是一次 forward solve，在优化器看来，只用于计算当前参数 xk 下的预测结果，参数不在 Step3 内更新

`reaction-advection-diffusion PDE coupled with linear elasticity`
- 肿瘤自己会增殖  
- 肿瘤会扩散  
- 肿瘤会因为组织运动而被 advect  
- 组织也会被肿瘤产生的力推动

```
在每个时间步 tn -> tn+1：
1. 用当前速度场 advect tumor/tissue
2. 处理 diffusion
3. 处理 reaction
4. 更新总肿瘤负荷
5. 由 tumor burden 产生 mechanical force
6. 解 screened linear elasticity 得到 displacement
7. 由 displacement 得到 velocity，供下一时间步 advection 使用
```

forward solve 的输入是：
	$\gamma_k, \rho_k, \kappa_k, altastissue$

输出是：
	$c_1^pred​, VT^pred, WM^pred, GM^pred, CSF^pred, u, v$

#### Step4：计算 loss / objective

>forward simulation 结束后，GLIA 得到当前参数下的：  
> - 终态肿瘤预测图 `c1_pred`  
> - 形变后的组织结构图，尤其是 `VT_pred`

mass-effect 模式的 objective 包括两部分：  
$$  
J(x)  
=  
J_{\text{tumor}}(x)  
+  
J_{\text{brain}}(x)  
$$
肿瘤项：  
$$  
J_{\text{tumor}}  
=  
\frac12 |\Omega|_h  
\|O c(1;x)-d_1\|_2^2  
$$ 
脑组织项，当前代码主要使用 ventricle mismatch：  
  
$$  
J_{\text{brain}}  
=  
\frac12 |\Omega|_h  
\|VT^{pred}(x)-VT^{patient}\|_2^2  
$$
因此：  
  
$$  
J(x)  
=  
\frac12 |\Omega|_h  
\|O c(1;x)-d_1\|_2^2  
+  
\frac12 |\Omega|_h  
\|VT^{pred}(x)-VT^{patient}\|_2^2  
$$  
注意：GM/WM/CSF mismatch 在代码中有相关实现痕迹，但当前被注释掉，实际 objective 主要使用 VT mismatch。


#### Step5：计算 gradient

> 这是 mass-effect 和 TIL 最大的不同点

mass-effect 用的是 **forward finite difference gradient** 而非 adjoint

先计算当前参数的 objective：  
$$  
J_b = J(x)  
$$
然后对每个参数分别做小扰动：  
$$  
x_i^{+}=x+h e_i  
$$
重新执行一次完整 forward simulation，并计算：  
$$  
J_f = J(x+h e_i)  
$$
于是：  
$$  
\frac{\partial J}{\partial x_i}  
\approx  
\frac{J_f-J_b}{h}  
$$
其中：  
  
$$  
x_i \in \{\gamma,\rho,\kappa\}  
$$
  
因此，若当前反演三个参数 `[gamma,rho,kappa]`，一次 gradient evaluation 大约需要：  
- 1 次 baseline forward solve；  
- 3 次 perturbed forward solve；  
  
总共约 4 次 forward solve

#### Step6：TAO 更新参数

>这一步和 TIL 在优化器层面是类似的：  
>GLIA 把 `J(x)` 和 `∇J(x)` 返回给 TAO，TAO 决定下一步怎么走

GLIA 将当前参数下的：  
$$  
J(x_k), \quad \nabla J(x_k)  
$$
返回给 TAO

TAO 根据内部优化方法，例如 line search / quasi-Newton / bound-constrained optimizer，决定更新方向：  
$$  
s_k  
$$
并更新参数：  
$$  
x_{k+1}  
=  
x_k+\alpha_k s_k  
$$  
其中：  
$$  
x_k=[\gamma_k,\rho_k,\kappa_k]  
$$  
更新后得到：  
  
$$  
x_{k+1}  
=  
[\gamma_{k+1},\rho_{k+1},\kappa_{k+1}]  
$$

#### （Step3：重新forward ）

```
新的 gamma、rho、kappa
        ↓
重新 forward simulation
        ↓
重新计算 loss
        ↓
重新计算 finite-difference gradient
        ↓
TAO 再更新
```

#### Step7：输出反演结果

当 TAO 满足收敛条件后，得到最终参数：
$$
x^*=[\gamma^*,\rho^*,\kappa^*]
$$
这组参数代表：
- $\gamma^*$：最能解释 patient mass effect 的组织推挤强度；
- $\rho^*$：最能解释 tumor growth 的增殖强度；
- $\kappa^*$：最能解释 tumor spread 的扩散强度。

最终 forward simulation 会给出：
- 终态肿瘤预测图；
- 形变后的组织结构；
- 位移场；
- 速度场；
- 与 patient data 的 mismatch。



# masseffect Step3 forward 详细计算顺序
GLIA 不是把所有方程一次性组成一个巨大的完全耦合非线性系统同时解，而是用 **operator splitting**
在一个时间步：$t^n→t^{n+1}$ 里面，把不同物理过程拆开依次计算


## Step 0：当前时刻已知
在 $t^n$，你已经知道：
$$p^n, i^n, n^n, o^n $$
$$m_G^n, m_W^n, m_F^n$$
$$u^n,v^n$$
其中：
- $p,i,n$：肿瘤各组分 ( prolifeactive 增殖，invasive 侵袭， necrotic坏死 )；
- $o$：氧气；
- $m_G, m_W, m_F$：灰质、白质、脑脊液等组织；
- $u$：组织位移；
- $\vec{v}$：组织速度。

## Step 1：advection
参考 [[GLIA PDE +elasticity#^ecbf54]] （守恒定律）
先用当前速度场 $v^n$ 搬运所有会随组织运动的量
$$
\begin{aligned}
\frac{\partial p}{\partial t}+\nabla\cdot(pv)=0 \\
\frac{\partial i}{\partial t}+\nabla\cdot(iv)=0 \\
\frac{\partial n}{\partial t}+\nabla\cdot(nv)=0
\\
\frac{\partial m_G}{\partial t}+\nabla\cdot(m_Gv)=0
\\
\frac{\partial m_W}{\partial t}+\nabla\cdot(m_Wv)=0.
\end{aligned}
$$
==代表”组织被tumor推着移动，细胞浓度和组织标签也一起移动“==

## Step 2：diffusion
然后处理会扩散的变量，例如 invasive cells 和 oxygen

$$
\begin{aligned}
\frac{\partial i}{\partial t}  
=  
\nabla\cdot(D_i\nabla i)  
 \\  
\frac{\partial o}{\partial t}  
=  
\nabla\cdot(D_o\nabla o).
\end{aligned}
$$
- i 侵袭型细胞向外浸润 
- o 氧气/营养在组织中扩散

## Step 3：reaction
处理局部反应项，在每个空间点上，根据当前：
$$p, i, n, o, m_G​, m_W​$$
计算：
- p 增殖多少 
- p→i 转换多少
- i→p 转换多少 
- o 被消耗多少
这一步==没有空间导数==，可以理解为每个 voxel 自己内部发生生物反应

## Step 4：更新总肿瘤负荷
反应、扩散、搬运之后，你得到新的：
$$p^{n+1}, i^{n+1}, n^{n+1}$$
进行计算
$$
c_{\mathrm{tumor}}^{n+1}=p^{n+1}+i^{n+1}+n^{n+1}.
$$

## Step 5：由肿瘤总量产生机械力
计算肿瘤诱导力：
$$
f^{n+1}=\gamma c_{\mathrm{tumor}}^{n+1}\nabla c_{\mathrm{tumor}}^{n+1}.
$$
肿瘤总量越大，边界梯度越强 ⇒ 越强的组织推挤力
>这一步把 reaction–advection–diffusion PDE 的结果传给 elasticity PDE

## Step 6：解 screened linear elasticity
然后解：
$$
-\nabla\cdot\sigma(u^{n+1})+\eta u^{n+1}=f^{n+1}.
$$
其中：
$$
\sigma(u)=\lambda \operatorname{tr}(\varepsilon(u))I+2\mu\varepsilon(u)
$$
$$
\varepsilon(u)=\frac{1}{2}\left(\nabla u+\nabla u^T\right).
$$
这一步得到新的组织位移场：
$$
u^{n+1}.
$$

## Step 7：由位移得到速度
$$
v^{n+1}=\frac{u^{n+1}-u^n}{\Delta t}.
$$
这个新速度会用于下一轮时间步的 advection：
$$
t^{n+1}\to t^{n+2}.
$$
所以耦合闭环是：
$$
p,i,n,o
\Rightarrow c_{\mathrm{tumor}}
\Rightarrow f
\Rightarrow u
\Rightarrow v
\Rightarrow \text{transport } p,i,n,\text{组织}
\Rightarrow p,i,n,o.
$$

## （Step 1：advection） 

用当前速度场 $v^n$ 搬运所有会随组织运动的量


## 问题解析：

在学习过程中遇到的问题

1. 为什么reaction-advection-diffusion PDE拆分成step 1 2 3？
	总的PDE在实际数值计算时，常常不会“一口气”直接解这个完整方程，而是把它拆成几个更简单的子问题
	$advection→diffusion→reaction$
	这叫 operator splitting，算子分裂
$$\frac{\partial c}{\partial t}
=
\underbrace{-\nabla\cdot(cv)}_{\text{advection}}
+
\underbrace{\nabla\cdot(D\nabla c)}_{\text{diffusion}}
+
\underbrace{R(c)}_{\text{reaction}}.
$$
	==它们不是代数拼接，而是**时间推进拼接**==
	- advection
		描述的是：细胞浓度被速度场 v 搬运$$  
\frac{\partial c}{\partial t}+\nabla\cdot(cv)=0  
$$ 
	- Diffusion
		描述的是：肿瘤细胞从高浓度区域向低浓度区域扩散$$\frac{\partial c}{\partial t}  
=  
\nabla\cdot(D\nabla c)  $$
	- reaction 
		描述的是：局部细胞增殖、死亡、转换$$
		\frac{\partial c}{\partial t}  = R(c)$$
			$$ c^n ⟶ c^* ⟶ c^{**} ⟶ c^{n+1}$$$$ advection ⟶ diffusion ⟶ reaction$$
		eg: 假设某个 voxel 里当前肿瘤密度是：
			$c^n=100$
			一个时间步 Δt 内：
				- advection 把 8 个单位搬走；
				- diffusion 净流入 3 个单位；
				- reaction 增殖 10 个单位。
			如果直接从总 PDE 的角度看：
			$c^{n+1} = 100-8+3+10 = 105$
			这就是：搬运+扩散+反应  一起作用的结果
		


## 涉及公式
肿瘤浓度 c 产生力 f， 力 f 造成位移 u， 位移 u 产生速度 v， 速度 v 搬运组织和肿瘤
[[GLIA PDE +elasticity#^ec7277]] （应变张量）
$$  
u  
\;\xrightarrow{\nabla}\;  
\varepsilon(u)  
=  
\frac{1}{2}  
\left(  
\nabla u+\nabla u^{T}  
\right)  
$$
[[GLIA PDE +elasticity#^00f701]] (应力张量)
$$ \varepsilon(u)  
\;\xrightarrow{\lambda,\mu}\;  
\sigma(u)  
=  
\lambda \operatorname{tr}(\varepsilon)I  
+  
2\mu\varepsilon  
\ $$
[[GLIA PDE +elasticity#^f77af1]] (力平衡)
$$  
\sigma(u)  
\;\xrightarrow{\nabla\cdot}\;  
-\nabla\cdot\sigma(u)+\eta u=f$$
[[GLIA PDE +elasticity#^293ce9]] (GLIA假设)
$$c  
\;\longrightarrow\;  
f=\gamma c\nabla c$$
对位移场求偏导得到速度场
$$u  
\;\longrightarrow\;  
v=\frac{\partial u}{\partial t}$$
[[GLIA PDE +elasticity#^a13e8c]] (reaction-advection PDE)
$$v  
\;\longrightarrow\;  
\frac{\partial c}{\partial t}  
+  
\nabla\cdot(cv)  
=  
\nabla\cdot(D\nabla c)  
+  
R(c)$$


