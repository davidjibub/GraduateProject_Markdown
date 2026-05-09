
# 1. FreeSurfer 分割“准不准”如何评估

> 公开的“肿瘤患者专家逐脑区 FreeSurfer-style 标注数据集”非常少，可以让专家标注关键 ROI

可以，有指标。但要先区分两件事：

**FreeSurfer 分割准确性**通常不能只靠 FreeSurfer 自己判断；最可靠的是和人工标注、专家修正标签、或高质量无损影像上的参考分割比较。FreeSurfer 官方工具 `mri_seg_overlap` 可以比较两个分割体，输出每个 label 的 Dice、Jaccard、体积差等指标；默认主要算 Dice，也支持 `--measures dice jaccard voldiff`。([FreeSurfer](https://surfer.nmr.mgh.harvard.edu/fswiki/mri_overlap "mri_overlap - Free Surfer Wiki"))

\mathrm{Dice}(A,B)=\frac{2|A\cap B|}{|A|+|B|}

## 1. 衡量 FreeSurfer 分割是否准确的常用指标

### A. 有 ground truth / 人工标注时：最推荐

对 `aseg.mgz`、`aparc+aseg.mgz` 或你关心的 ROI mask 做逐标签比较：

|指标|含义|适合场景|
|---|---|---|
|**Dice / DSC**|区域重叠程度，越高越好|最常用，适合海马、杏仁核、脑室、皮层/白质等|
|**Jaccard**|交并比，越高越好|和 Dice 类似，更严格|
|**HD95 / Hausdorff 95**|边界最坏 5% 误差，越低越好|关心边界质量时|
|**ASSD / ASD**|平均表面距离，越低越好|适合边界/表面评价|
|**Volume Difference / Relative Volume Error**|体积偏差，越接近 0 越好|下游是脑区体积分析时很重要|

FreeSurfer 自带的 `mri_seg_overlap` 明确支持 Dice、Jaccard 和 volume difference，并且可以按每个 label 输出结果。([FreeSurfer](https://surfer.nmr.mgh.harvard.edu/fswiki/mri_overlap "mri_overlap - Free Surfer Wiki"))

示例思路：

```bash
mri_seg_overlap \
  --measures dice jaccard voldiff \
  --seg \
  --out overlap.json \
  reference_aparc_aseg.mgz \
  inpainted_aparc_aseg.mgz
```

### B. 没有人工 ground truth 时：做“替代性验证”

没有人工标注时，不能严格说“准确性提升”，只能说“与参考结果更一致”或“FreeSurfer 输出质量更好”。可以用：

|指标|用法|
|---|---|
|**Euler number / topological defects**|FreeSurfer 皮层重建质量常用自动 QC 指标；ABCD 数据文档中也把 FreeSurfer 拓扑缺陷数定义为由 Euler number 计算的自动 QC 指标。([AbcdStudy Docs](https://docs.abcdstudy.org/v/6_0_0/documentation/imaging/type_qc.html "MRI Quality Control"))|
|**QC pass rate**|修复前后人工检查 FreeSurfer 输出，统计通过率|
|**surface defect 数量**|`recon-all.log`、`?h.orig.nofix`、Euler 相关统计|
|**ROI 体积分布异常值**|看修复后 ROI volume 是否更接近健康/同组分布|
|**左右对称性指标**|对非病灶侧和对侧同名结构比较，适合单侧病灶场景|
|**重测一致性 ICC / CV**|如果有 test-retest 或多扫描，修复后同一被试结果应更稳定|
|**专家盲评**|最实用：让医生/影像专家盲评修复前后 FreeSurfer 分割边界|

视觉 QC 仍然很重要。VisualQC 论文强调 FreeSurfer parcellation 的准确性需要可靠的人工/视觉 QC，并提供了专门用于检查 FreeSurfer 分区质量的流程。

## 2. 你的 inpainting 对 FreeSurfer 下游任务提升，应该这样评估

你的实验最好设计成 **paired comparison**：

```text
同一受试者影像
├── 原始受损/含伪影/含病灶图像 → FreeSurfer → segmentation_before
└── inpainting 修复后图像       → FreeSurfer → segmentation_after
```

然后根据有没有参考标准选择评估方式。

---

## 方案一：有真实健康图像或人工标签，最有说服力

这是最强实验设计。

### 情况 1：你有“修复前损坏图像 + 原始干净图像”

例如你人为 mask 掉健康 T1 的某一区域，再做 inpainting。此时原始干净图像的 FreeSurfer 结果可作为 pseudo-reference：

```text
clean T1 → FreeSurfer → reference segmentation
corrupted T1 → FreeSurfer → before segmentation
inpainted T1 → FreeSurfer → after segmentation
```

然后比较：

```text
Dice(after, reference) > Dice(before, reference)
HD95(after, reference) < HD95(before, reference)
Volume error(after, reference) < Volume error(before, reference)
```

这是最推荐的验证 inpainting 是否改善下游分割的方式。

### 情况 2：你有人工标注

比如专家标注海马、脑室、肿瘤周围正常组织、皮层/白质边界。直接比较：

```text
FreeSurfer before vs manual label
FreeSurfer after  vs manual label
```

核心结果就是每个 ROI 的 Dice、HD95、体积误差，以及全脑/加权平均。

---

## 方案二：没有 ground truth，但有病灶/缺损区域 mask

这时建议分区域报告：

### 1. lesion / defect mask 内

看 inpainting 是否让 FreeSurfer 能合理恢复结构，但要谨慎，因为病灶区本来可能没有真实正常解剖结构。

可报告：

```text
病灶区内 FreeSurfer label 合理性
病灶区内灰质/白质/脑脊液比例
修复前后分割失败率
专家盲评分
```

### 2. lesion / defect mask 外

更重要。因为 inpainting 理论上不应该破坏健康区域。

可报告：

```text
mask 外 Dice(before, after)
mask 外 ROI volume change
mask 外皮层厚度变化
非病灶侧结构体积变化
```

如果 inpainting 后 mask 外变化很大，说明修复可能引入了非局部影响，FreeSurfer 结果不稳定。

---

## 方案三：用 FreeSurfer 自身 QC 指标证明“可处理性提升”

这适合没有人工标签时作为辅助证据。

可以比较修复前后：

```text
recon-all 成功率
recon-all 运行失败率
Euler number / topological defects
manual QC pass rate
pial surface 错误数
white surface 错误数
aseg 明显错误率
ROI volume outlier rate
```

ABCD 文档中就把 FreeSurfer 的 topological defects 作为自动 post-processing QC 指标，并说明它由 Euler number 计算。([AbcdStudy Docs](https://docs.abcdstudy.org/v/6_0_0/documentation/imaging/type_qc.html "MRI Quality Control"))

注意：**Euler number 变好不等于分割一定准确**，但可以说明皮层表面重建更少拓扑错误，是一个很有用的辅助指标。

---

## 3. 我建议你的主指标和辅指标这样设定

### 主指标

如果你评估的是 FreeSurfer 下游分割：

```text
Primary outcome:
1. ROI-wise Dice
2. ROI-wise HD95 / ASSD
3. ROI-wise relative volume error
```

重点 ROI 可以包括：

```text
Cerebral Cortex
Cerebral White Matter
Hippocampus
Amygdala
Thalamus
Caudate
Putamen
Pallidum
Lateral Ventricle
Lesion-adjacent cortical ROIs
```

FreeSurfer 的 `mri_seg_overlap --seg` 正好支持若干主要结构，例如 cerebral white matter、cerebral cortex、hippocampus、caudate、putamen、pallidum、amygdala、thalamus、lateral ventricle 等。([FreeSurfer](https://surfer.nmr.mgh.harvard.edu/fswiki/mri_overlap "mri_overlap - Free Surfer Wiki"))

### 辅指标

```text
Secondary outcome:
1. recon-all success rate
2. Euler number / topological defects
3. Visual QC pass rate
4. ROI volume outlier rate
5. cortical thickness reliability
6. downstream statistical effect stability
```

如果你的最终任务是组间分析，比如患者 vs 对照，还可以看：

```text
inpainting 后是否提高组间效应的稳定性
是否减少异常体积值
是否降低批次/伪影相关偏差
是否提升分类/回归任务性能
```

---

## 4. 一个比较完整的实验表述可以这样写

你可以在论文/实验中这样定义：

```text
We evaluated whether MRI inpainting improves downstream FreeSurfer segmentation by comparing FreeSurfer outputs obtained from corrupted and inpainted T1-weighted images against reference segmentations derived from clean images or manual annotations. Segmentation accuracy was quantified using ROI-wise Dice coefficient, Jaccard index, HD95, ASSD, and relative volume error. FreeSurfer processing quality was further assessed using recon-all success rate, Euler number/topological defect counts, ROI volume outlier rate, and blinded visual QC.
```

中文就是：

```text
我们通过比较修复前后 T1 图像经 FreeSurfer 得到的分割结果与参考分割之间的一致性，评估 inpainting 对下游分割任务的影响。主要指标包括各脑区 Dice、Jaccard、HD95、ASSD 和相对体积误差；辅助指标包括 recon-all 成功率、Euler number/拓扑缺陷数、ROI 体积异常值比例和盲法视觉 QC 通过率。
```

---

## 5. 最关键的注意点

不要只比较：

```text
FreeSurfer 原始图像分割 vs FreeSurfer inpainted 图像分割
```

然后说 Dice 高或低。因为这只能说明“两个 FreeSurfer 输出是否一致”，不能说明谁更准确。

更合理的是：

```text
修复前 FreeSurfer vs 参考标准
修复后 FreeSurfer vs 参考标准
```

或者至少：

```text
修复前后 FreeSurfer 输出质量指标 + 专家盲评 + ROI 体积合理性
```

我的建议是：**主实验用模拟缺损数据建立 clean reference，真实病灶数据用人工 QC 和 FreeSurfer QC 指标做外部验证**。这样最容易证明你的 inpainting 不是只让图像“看起来更好”，而是真的改善了 FreeSurfer 下游分割。