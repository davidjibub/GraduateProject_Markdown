
# 1.  WDM level channel变化

> level 1经过 4个 resblock后还是 64-64吗？ 还是有变为128？我不理解如果都想level 0一样，不对channel尺寸改变，那不是一直都是64吗？不符合unet网络channel逐层增加的特点啊

不是。**Level 1 经过 4 个 ResBlock 后不会一直是 64。**  
在 `ours_wnet_128` 里，encoder 的 channel 是按 `channel_mult=(1,2,2,4,4)` 逐层增加的，base channel 是 64，所以各 level 是：

```text
Level 0: 64
Level 1: 128
Level 2: 128
Level 3: 256
Level 4: 256
```

关键点是：**每个 level 的第一个 ResBlock 负责把上一层通道数变成当前 level 的目标通道数；后面的 ResBlock 保持该通道数。**

---

## 1. Level 0 为什么是 64→64？

网络最开始有一个 input conv：

```text
model_input [B,24,S,S,S]
    ↓ Conv3D 24→64
h [B,64,S,S,S]
```

Level 0 的目标通道数是：

```text
base_channels × channel_mult[0] = 64 × 1 = 64
```

所以 Level 0 的 4 个 ResBlock 都是：

```text
ResBlock 1: 64 → 64
ResBlock 2: 64 → 64
ResBlock 3: 64 → 64
ResBlock 4: 64 → 64
```

然后 wavelet down：

```text
Wavelet Down: [B,64,S,S,S] → [B,64,S/2,S/2,S/2]
```

再加上 progressive 分支：

```text
Progressive WaveletDownsample 24→64
```

最终进入 Level 1 的输入是：

```text
[B,64,S/2,S/2,S/2]
```

---

## 2. Level 1 第一个 ResBlock 会把 64 变成 128

Level 1 的目标通道数是：

```text
64 × channel_mult[1] = 64 × 2 = 128
```

所以 Level 1 的 4 个 ResBlock 实际是：

```text
ResBlock 1: 64  → 128
ResBlock 2: 128 → 128
ResBlock 3: 128 → 128
ResBlock 4: 128 → 128
```

也就是说，**不是 4 个都是 64→64。**

数据流：

```text
Level 1 input:
[B,64,S/2,S/2,S/2]

ResBlock 1:
[B,64,S/2,S/2,S/2] → [B,128,S/2,S/2,S/2]

ResBlock 2:
[B,128,S/2,S/2,S/2] → [B,128,S/2,S/2,S/2]

ResBlock 3:
[B,128,S/2,S/2,S/2] → [B,128,S/2,S/2,S/2]

ResBlock 4:
[B,128,S/2,S/2,S/2] → [B,128,S/2,S/2,S/2]
```

然后 Level 1 的 wavelet down：

```text
Wavelet Down:
[B,128,S/2,S/2,S/2]
→ low [B,128,S/4,S/4,S/4]
+ high-frequency skips
```

同时 progressive 分支：

```text
Progressive WaveletDownsample 64→128:
input_pyramid [B,64,S/2,S/2,S/2]
→ [B,128,S/4,S/4,S/4]
```

最后相加：

```text
Level 2 input:
[B,128,S/4,S/4,S/4]
```

---

## 3. 整个 encoder 的通道变化

以 `ours_wnet_128` 为例：

```text
Input:
[B,24,S,S,S]
    ↓ Conv3D 24→64

Level 0, target C=64:
ResBlock: 64→64
ResBlock: 64→64
ResBlock: 64→64
ResBlock: 64→64
Wavelet Down: 64→64
Progressive: 24→64
Output to Level 1: [B,64,S/2,S/2,S/2]

Level 1, target C=128:
ResBlock: 64→128
ResBlock: 128→128
ResBlock: 128→128
ResBlock: 128→128
Wavelet Down: 128→128
Progressive: 64→128
Output to Level 2: [B,128,S/4,S/4,S/4]

Level 2, target C=128:
ResBlock: 128→128
ResBlock: 128→128
ResBlock: 128→128
ResBlock: 128→128
Wavelet Down: 128→128
Progressive: 128→128
Output to Level 3: [B,128,S/8,S/8,S/8]

Level 3, target C=256:
ResBlock: 128→256
ResBlock: 256→256
ResBlock: 256→256
ResBlock: 256→256
Wavelet Down: 256→256
Progressive: 128→256
Output to Level 4: [B,256,S/16,S/16,S/16]

Level 4, target C=256:
ResBlock: 256→256
ResBlock: 256→256
ResBlock: 256→256
ResBlock: 256→256
Wavelet Down: 256→256
Progressive: 256→256
Output to Bottleneck: [B,256,S/32,S/32,S/32]
```

所以 channel 是会逐层增加的，只是**增加发生在每个 level 的第一个 ResBlock**。

---

## 4. 为什么 Wavelet Down 本身不改 channel？

Wavelet Down 的主要职责是：

```text
空间下采样 + 保存高频 skip
```

也就是：

```text
[B,C,S,S,S]
→ low [B,C,S/2,S/2,S/2]
+ 7 个 high-frequency skips
```

它主要改变空间尺寸，不负责把 64 改成 128。  
通道改变由下一个 level 的第一个 ResBlock 完成。

这其实很常见。很多 U-Net 写法也是：

```text
Downsample: H,W 减半
Next level first conv/resblock: C 增加
```

而不是必须在 downsample 那一步同时改 channel。

---

## 5. 为什么 ResBlock 能 64→128？

因为 ResBlock 有两条分支：

主分支：

```text
GroupNorm → SiLU → Conv3D 64→128
...
ZeroConv3D 128→128
```

skip 分支：

```text
Conv3D 1×1×1, 64→128
```

最后：

```text
output = main_branch [B,128,...] + skip_branch [B,128,...]
```

所以 `ResBlock(64→128)` 是合法的。

---

## 6. 你图中应该怎么改？

你画 Level 1 时，不要写：

```text
ResBlock ×4 (64→64)
```

而应该写：

```text
Level 1, target channels=128

ResBlock ×4:
    第 1 个：64→128
    后 3 个：128→128

Wavelet Down:
    128→128，空间减半

Progressive WaveletDownsample:
    64→128

Add:
    [B,128,S/4,S/4,S/4]
```

更简洁可以写成：

```text
ResBlock ×4: 64→128, then 128→128 ×3
```

Level 3 同理：

```text
ResBlock ×4: 128→256, then 256→256 ×3
```

---

最短回答就是：

**Level 1 的第一个 ResBlock 会把 64 channel 变成 128 channel；后面三个 ResBlock 保持 128。channel 不是由 Wavelet Down 改的，而是由每个 level 的第一个 ResBlock 根据 `channel_mult` 改的。**


# 2. Decoder Resblock Wavelet Up

>这里经过resblock后，high4是没有变化吗？没有变化的话为什么还要放进resblock？还是有变化？


![[WDM 疑问-1778126046095.png]]
直面回答：**经过 decoder 的普通 ResBlock 时，`high4` 基本没有变化；它被放进 ResBlock，不是为了让 ResBlock 更新它，而是为了让它在 decoder level 内部被“携带”到最后的 wavelet up / IDWT 模块。**

更准确地说：

```text
(h, high4) 进入 ResBlock
ResBlock 只处理 h
high4 原样带出来
输出仍然是 (h_processed, high4)
```

---

## 1. 为什么说 high4 在普通 ResBlock 中不变？

源码 `ResBlock.forward()` 里，如果输入是 tuple：

```python
if isinstance(x, tuple):
    hSkip = x[1]
else:
    hSkip = None
```

也就是先把高频 skip 拿出来，记作 `hSkip`。

然后如果不是 up/down ResBlock，代码会做：

```python
if isinstance(x, tuple):
    x = x[0]

h = self.in_layers(x)
...
out = self.skip_connection(x) + h
out = out, hSkip
return out
```

也就是说：

```text
x[0] = 当前 decoder 低频特征 h
x[1] = high4 skip

ResBlock 只对 x[0] 做 GroupNorm、SiLU、Conv、time embedding、Conv
x[1] high4 不参与这些卷积
最后又和 out 一起打包返回
```

所以普通 ResBlock 的效果是：

```text
ResBlock((h, high4)) = (ResBlock(h), high4)
```

源码中这个“pipe skip connections”的逻辑就是为了把 skip connection 透传下去。([GitHub](https://raw.githubusercontent.com/AliciaDurrer/fastWDM3D/main/WDM3D/guided_diffusion/wunet.py "raw.githubusercontent.com"))

---

## 2. 那为什么还要把 high4 放进 ResBlock？

因为 decoder level4 不是只有一个 IDWT，它前面有多个 ResBlock。

结构是：

```text
进入 decoder level4:
(h_bottleneck, high4)
        ↓
ResBlock
        ↓
ResBlock
        ↓
ResBlock
        ↓
ResBlock
        ↓
Wavelet Up / IDWT
```

如果不把 `high4` 放进 tuple 里一路传递，那么前面的 ResBlock 输出就只剩下：

```text
h_processed
```

到最后的 IDWT 时，就找不到 high-frequency skip 了。

所以 high4 放进 ResBlock 的目的不是“被 ResBlock 修改”，而是：

```text
作为 tuple 的第二项被一路携带，
直到最后的 Upsample / IDWT 模块真正使用它。
```

你可以把它理解成：

```text
ResBlock 是“处理 h 的加工站”
high4 是“随车携带的包裹”
包裹不在加工站打开
一直带到最后的 IDWT 站才使用
```

---

## 3. high4 什么时候真正被使用？

在 decoder level 最后的 `Upsample.forward()` 里。

源码中 `Upsample.forward()` 如果收到 tuple，会做：

```python
skip = x[1]
x = x[0]
```

然后在 `use_freq=True` 时：

```python
x = self.idwt(
    3.0 * x,
    skip[0], skip[1], skip[2],
    skip[3], skip[4], skip[5], skip[6]
)
```

也就是说：

```text
当前 decoder 低频特征 h
    ↓ 乘 3.0
3h

encoder 保存的 high4 七个高频子带
    ↓ 原样作为 IDWT 的高频输入

IDWT_3D(3h, high4[0], ..., high4[6])
    ↓
上采样后的 feature
```

所以 high4 真正参与计算是在 **IDWT 上采样** 的时候。([GitHub](https://raw.githubusercontent.com/AliciaDurrer/fastWDM3D/main/WDM3D/guided_diffusion/wunet.py "raw.githubusercontent.com"))

---

## 4. high4 是否永远完全不变？

默认理解：**在普通 ResBlock 中不变。**

但还有一个细节：`Upsample` 模块本身支持对 high-frequency skip 做 grouped convolution：

```python
if self.use_conv:
    skip = self.conv(th.cat(skip, dim=1) / 3.0) * 3.0
    skip = tuple(th.chunk(skip, 7, dim=1))
```

也就是说，如果 `Upsample` 的 `use_conv=True`，那么 high4 在进入 IDWT 前会经过一个 grouped conv 被调整。这个修改发生在 **Upsample 内部**，不是发生在前面的普通 ResBlock 里。([GitHub](https://raw.githubusercontent.com/AliciaDurrer/fastWDM3D/main/WDM3D/guided_diffusion/wunet.py "raw.githubusercontent.com"))

对于你现在问的“经过 ResBlock 后 high4 有没有变化”，答案仍然是：

```text
没有，普通 ResBlock 不改 high4。
```

---

## 5. 用 level4 数据流直观表示

假设：

```text
h_bottleneck : [B,256,s,s,s]
high4        : 7 × [B,256,s,s,s]
```

进入 decoder level4：

```text
(h_bottleneck, high4)
```

经过第一个 ResBlock：

```text
ResBlock 1:
    输入:  (h_bottleneck, high4)
    处理:  h_bottleneck → h1
    输出:  (h1, high4)
```

第二个 ResBlock：

```text
ResBlock 2:
    输入:  (h1, high4)
    处理:  h1 → h2
    输出:  (h2, high4)
```

第三、第四个同理：

```text
ResBlock 3: (h2, high4) → (h3, high4)
ResBlock 4: (h3, high4) → (h4, high4)
```

最后 wavelet up：

```text
Upsample / IDWT:
    输入: (h4, high4)

    h_up = IDWT_3D(
        3.0 * h4,
        high4[0],
        high4[1],
        high4[2],
        high4[3],
        high4[4],
        high4[5],
        high4[6]
    )

    输出:
        h_up [B,256,2s,2s,2s]
```

---

## 6. 为什么设计成这样，而不是直接在 decoder 开头 IDWT？

因为作者想让 decoder 在最低分辨率处先对低频语义特征做若干 ResBlock 处理，再恢复空间分辨率。

也就是：

```text
先在低分辨率上处理语义信息：
    h_bottleneck → ResBlock ×4 → h4

再用 encoder 的高频 skip 恢复空间细节：
    IDWT(3h4, high4)
```

如果一开始就 IDWT：

```text
IDWT(h_bottleneck, high4)
→ 再 ResBlock
```

那计算会更早回到高分辨率，显存和计算量更大，而且结构上也不符合代码当前的 decoder block 构造。

---

## 7. 最简结论

你的问题可以直接回答为：

**high4 经过普通 decoder ResBlock 时没有变化。它之所以放进 ResBlock，是因为代码把 decoder 的数据表示成 `(低频主特征 h, 高频 skip)` 这个 tuple，ResBlock 只更新 `h`，同时把 `high4` 原样透传到后面的模块。等到最后的 wavelet up / IDWT 时，才真正使用 `high4`，即 `IDWT_3D(3.0 * h_processed, high4)`。**