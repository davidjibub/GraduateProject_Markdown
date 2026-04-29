---
type:
  - thesis
  - concept
status: done
thesis: "[[GLIA_Mode_masseffect]]"
module: 对GLIA masseffect模式的数学原理公式解析
tags:
  - thesis
  - paper
topic:
  - pathology simulation
  - math-learning
created: 2026-04-29
---
# reaction-diffusion PDE
细菌边随机游走扩散，边每个位置本地繁殖
$$\frac{∂c}{∂t}​=∇⋅(D*∇c)+f(c)$$

==未考虑 ”环境整体将细菌往某方向带走“==

# reaction-advection-diffusion PDE

^a13e8c

细菌边随机游走扩散，边每个位置本地繁殖，同时还被环境带着走$$\frac{∂c}{∂t}​= -∇(c \cdot \vec{v}) + ∇⋅(D*∇c)+f(c)$$
> 随机扩散 + 本地繁殖 + 整体搬运

## advection（对流运输）
某物质被一个背景速度场整体带着走，及背景流动导致的空间运输
$$ -∇(c \cdot \vec{v})$$
$c(x,t)$ : 肿瘤的密度
$\vec{v}(x,t)$ ：速度场，环境将物质外哪个方向搬运

守恒定律：  ^ecbf54
$$  
\frac{\partial c}{\partial t}+\nabla\cdot(cv)=0  
$$
$c\vec{v}$ 是一个向量场，表示通量，取散度，得到某个位置“流入/流出是否平衡”的标量
> 公式表示：局部密度变化 + 净流出量=0  
> 即：某点的密度随时间增加/减少=−该点附近通量的净流出​

### 公式解析
1. 通量
	单位时间穿过单位面积的物质量	$$c \cdot \vec{v}$$
2. 搬运速率
	 沿速度场的方向看，浓度被搬运的部分$$
\begin{aligned}
-\nabla(c \cdot \vec{v}) \\
\nabla(c \cdot \vec{v})
&= \vec{v} \cdot \nabla c + c \cdot \nabla \vec{v} \end{aligned}
$$ 
	**==沿速度场方向浓度被搬移的量 + ==局部体积变化导致的浓度变化==**
	
	若流体不可被压缩，则 *散度* =0$$∇\vec{v} =0$$
	 对向量场的梯度称为 *散度*，表示某点是像”源“向外发散 or 像”汇“向内聚集[[微分算符#^dc7fd1]]
	 故：$$∇(c \cdot \vec{v}) = \vec{v} \cdot ∇c $$
	 但是真实肿瘤世界，不一定 $\nabla \vec{v} = 0$ ，因为肿瘤生成会导致组织的局部体积变化
	 

# linear elasticity
^d813b1
将脑组织看作 “受力后会产生微小形变，撤去力后倾向恢复” 的弹性材料

## 1D
$$ F = k \Delta x$$
位移向量场:  $u(x,t)$
应变：局部形变 $\epsilon$，位移场的空间导数 $\frac{d u}{d x}$
应力：$F$
==位移向量场 u(x,t) 描述 “去哪儿”，应变 $\epsilon$ 描述 “变形了多少”==

## 3D
1. 位移向量场 $u(x,t)$
> 表示在位置 x 的组织，被tumor挤压后移动至 x+u(x,t)
$$u(x,t) = \begin{bmatrix} u1(x,t) \\ u2(x,t) \\u3(x,t)
\end{bmatrix}$$
2. 位移梯度 $\Delta u(x,t)$
   ==向量场的空间变化率矩阵==
    >当空间位置沿 xj​ 方向移动一点时，i 方向的位移分量变化多快
    > $$ \nabla u = \frac{\partial u_i}{\partial x_j}$$$$\nabla u= \begin{bmatrix} \frac{\partial u1}{\partial x} &.&\frac{\partial u1}{\partial z}\\ .&.&. \\ \frac{\partial u3}{\partial x}&.&\frac{\partial u3}{\partial z}
\end{bmatrix}$$
3. 应变张量
	衡量内部的形变$$ε(u)=\frac{1}{2} ​(∇u+∇u^T)$$这里的位移梯度 $\Delta u$ 包括了 “真正的形变” + “刚体旋转”
	==应变张量 $\varepsilon(u)$ 通过计算 $\Delta u$ 的对称部分，从而衡量 “真正的形变”== 
 ^ec7277
4. 应力张量 stress tensor
    >由 [[张量]] 可知：应力张量是二阶张量场，即每个点是二阶张量 [ 矩阵 ]，且应该是3x3，因为有每个面不是只有拉伸，而是可能有拉伸旋转
	
	 根据应变张量计算内部形变所造成的 “内部反抗力”，==单位面积==上的力，单位为Pa（$\frac{N}{m^2}$）$$\sigma = C \varepsilon $$
	 若材料各向同性，各方向的弹性一致，则
	 $$\sigma(u) = \lambda \cdot tr(\varepsilon(u)) \cdot I + 2 \mu \varepsilon(u)$$
	 $tr(\varepsilon)$ : 代表应变张量矩阵的迹，为 $\varepsilon_{11} + \varepsilon_{22}+ \varepsilon_{33}$
		 用于描述局部体积变大 $tr >0$ or 变小 $tr <0$
	 $\lambda$ : 材料参数，代表体积压缩，体积压缩/膨胀 难度越高，$\lambda$ 越大
	 $\mu$ : 材料参数，代表剪切刚度，组织越硬，$\mu$ 越大  ^00f701


# 静态弹性平衡
==脑组织内部弹性恢复力密度= 肿瘤施加的外部推力密度==
对应力张量场计算散度得到向量场
$$-\nabla \sigma(u) = f$$
>为什么要求散度？
   $\sigma(u)$ 是单位面积上的力（$\frac{N}{m^2}$），$f$ 通常是体力密度（$\frac{N}{m^3}$）
   对 $\sigma$ 做散度，单位才能和 $f$ 一致
- σ是张量场 / 矩阵场 
- ∇⋅σ是向量场
- f也是向量场



# GLIA
## linear + screening 弹性平衡
$$-\nabla \sigma(u) + \eta u= f$$
### linear+ screening

^f77af1

为了限制远场效应，相比 *静态弹性平滑*， 加入screening elasticity
- $\eta$ ： 与肿瘤密度相关，更好的使mass effect局部化
- $\eta u$ ： 将组织拉回原位的虚拟弹簧，实现越远离tumor，越不允许组织出现大位移
		- 理想情况模拟：将 skull，meninges... 进行作为刚体/流体约束，但是会带来过多的参数，以及接触约束造成的难以计算
	
	$\eta u$ 本质：将 meninges, skull, CSF... 边界简单化，==用局部位移惩罚替代复杂边界==

### GLIA Hypothesis
使用多物种假设
^293ce9

$$f = \gamma C_{tumor} \cdot \nabla C_{tumor}$$
$$C_{tumor} = p+i+n$$
- p : prolifeactive tumor cells 增殖
- i : invasive tumor cells  侵袭
- n : necrotic   坏死
- o : oxygen  氧气

Oxygen 决定 p, i , n 如何进行演化
	- O 高 -> p 增加
	- O 中低 -> p 向 i 转化
	- O 低 -> p, i 向 n转化

p, i, n, o 有自己的演化方程
增殖型细胞 p
$$
\frac{\partial p}{\partial t}+\nabla\cdot(pv)
=R_p(p,o)-T_{p\to i}(o)p+T_{i\to p}(o)i-D_p(o)p.
$$
- $\nabla \cdot (pv)$  : 表示组织位移速度 $v$ 对 $p$ 的搬运
- $R_p(p,o)$ : 表示氧气充足时的有丝分裂增殖
- $T_{p\to i}(o)p$ : 表示缺氧时 $p$ 转成 $i$
- $T_{i\to p}(o)i$ : 表示环境改善后 $i$ 转回 $p$
- $D_p(o)p$ : 这里的 $D_p$ 我用作 death rate，不是 diffusion coefficient，表示缺氧死亡进入坏死

侵袭型细胞 i
$$
\frac{\partial i}{\partial t}+\nabla\cdot(iv)
=\nabla\cdot(D_i\nabla i)+R_i(i,o)+T_{p\to i}(o)p-T_{i\to p}(o)i-D_i^{death}(o)i
$$
- invasive cells 的核心特点就是迁移、浸润，所以有==扩散项==

坏死细胞 n
$$
\frac{\partial n}{\partial t}+\nabla\cdot(nv)
=
D_p^{death}(o)p+D_i^{death}(o)i+D_h^{death}(o)h.
$$
氧气 / 营养 o
$$
\frac{\partial o}{\partial t}
=
\nabla\cdot(D_o\nabla o)+S_o(h)-C_o(p,i,o).
$$

- $S_o(h)$ ：表示健康组织供应氧气
- $C_o(p,i,o)$ ： 表示肿瘤细胞消耗氧气
- 也有==扩散项==