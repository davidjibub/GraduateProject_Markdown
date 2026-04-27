# Inverse_masseffect
## 1. 把 NIfTI 转成 GLIA 更稳定的 .nc
可以先用 SimpleITK / nibabel 把标签或概率图转为 3D float32 array，再写成 NetCDF。
这样避免 NIfTI header、spacing、orientation 和 GLIA 网格解释不一致的问题。

## GLIA
### 1. 输入
#### 1. 患者肿瘤分割
`patient_001_tumor_seg.nii.gz`
GLIA 标签映射为 `[wm=6,gm=5,vt=7,csf=8,ed=2,nec=1,en=4]`

即需要Brats的分割标签
- 1 = necrotic / non-enhancing tumor core
- 2 = edema
- 4 = enhancing tumor

#### 2. 患者脑组织分割
`patient_001_seg_aff2template.nii.gz`
GLIA 标签映射为 `[wm=6,gm=5,vt=7,csf=8,ed=2,nec=1,en=4]`

即需要分割结果
- WM  = white matter
- GM  = gray matter
- CSF = cerebrospinal fluid
- VT  = ventricles

#### 3. 患者 ventricle mask
`patient_001_vt_aff2template.nc`
ventricle 对 mass effect 很重要，因为肿瘤压迫常常表现为脑室变形、移位、压窄。GLIA 输出里也有 vt_err、vt_l2、vt_change 等指标。

inverse_masseffect.py test script 里有：
`p['p_vt_path'] = os.path.join(code_dir, 'testdata/patient_vt.nc')`

#### 4. 健康Altas
`atlas_001_seg_aff2template.nii.gz`
`atlas_001_t1_aff2template.nii.gz`
atlas segmentation 也用同样标签：
- 0 = background
- 5 = GM
- 6 = WM
- 7 = ventricles
- 8 = CSF
可复用`p['atlas_labels'] = "[wm=6,gm=5,vt=7,csf=8,ed=2,nec=1,en=4]"`
官方说明中，mass effect inversion 需要多个 healthy template brain images，推荐使用 ADNI 正常对照；这些模板也要被分割，并且和患者一样 affine registered 到模板空间
建议准备 5 到 20 个健康 atlas。数量越多，ensemble 结果越稳，但计算量也越大。

目录建议：
```
/project/GLIA_USER_DATA/
  patients/
    patient_001/
      patient_001_seg_aff2template.nii.gz
      patient_001_vt_aff2template.nii.gz
      patient_001_t1ce_aff2template.nii.gz
      patient_001_flair_aff2template.nii.gz

  atlases/
    atlas_001/
      atlas_001_seg_aff2template.nii.gz
      atlas_001_t1_aff2template.nii.gz
    atlas_002/
      atlas_002_seg_aff2template.nii.gz
      atlas_002_t1_aff2template.nii.gz
    atlas_003/
      atlas_003_seg_aff2template.nii.gz
      atlas_003_t1_aff2template.nii.gz

  atlas-list.txt
```
atlas-list.txt 内容例如：
```
atlas_001
atlas_002
atlas_003
```
官方输出说明中也提到 atlas-list.txt 是用于 ensemble inversion 的 template 文件名列表。


### 2. 预处理
最重要的预处理：所有图像必须在同一空间、同一尺寸、同一 orientation
GLIA 对输入空间非常敏感。

你要确保：
```
patient_seg
patient_vt
atlas_seg
atlas_t1
tumor masks
```
满足：
```
同一 physical space
同一 affine
同一 voxel spacing
同一 matrix size
同一 orientation
同一 label convention
```
官方文档说 inverse_til 假设 scans 已经 affine registered 到 template。
推荐你统一到：
```
256 x 256 x 256
1 mm isotropic
template / atlas common space
```
GLIA 官方目标就是医学影像 256³ 分辨率，并说明 GPU/CPU 支持这种分辨率
`p['n'] = 256`


## GLIA 数据准备Pipeline
### Step 1：患者 MRI 配准到模板空间

例如你有：
```
patient_001_t1ce.nii.gz
patient_001_flair.nii.gz
patient_001_seg.nii.gz
```
你需要做：
```
patient native space
        ↓ affine registration
common template space / Jakob / MNI-like space
```
官方示例路径里使用了 aff2jakob 这样的结构：

`/path/to/all_patients/patient_name/aff2jakob/patient_name_seg_ants_aff2jakob.nii.gz`

这是 README 中明确提到的 BraTS 风格输入路径。

你自己的目录可以这样设计：
```
patients/patient_001/aff2template/
  patient_001_t1ce_aff2template.nii.gz
  patient_001_flair_aff2template.nii.gz
  patient_001_seg_aff2template.nii.gz
  patient_001_vt_aff2template.nii.gz
```

### Step 2：生成 GLIA 合并分割

你要把 tissue segmentation 和 tumor segmentation 合成一个 label image。

最终类似：
`patient_001_seg_aff2template.nii.gz`

标签：
```
0 background
1 necrosis
2 edema
4 enhancing tumor
5 GM
6 WM
7 ventricles
8 CSF
```
伪代码逻辑：
```
combined = tissue_seg.copy()
combined[tumor_seg == 1] = 1  # necrosis
combined[tumor_seg == 2] = 2  # edema
combined[tumor_seg == 4] = 4  # enhancing
```
**注意：肿瘤标签应该覆盖 WM/GM/CSF 标签，否则 GLIA 会把肿瘤区域误认为正常组织。**
### Step 3：先跑 inverse_til / inverse_nomasseffect
这是 inverse_masseffect 的前置步骤。

目标：得到：
```
c0_rec.nc
c0_rec_*.nii.gz
c1_rec.nc
reconstruction_info.dat
solver_log.txt
```
官方说明中，inverse_til.py 输出 c0_rec.nii.gz 和 c1_rec.nii.gz，其中 c0_rec 是 reconstructed tumor IC，c1_rec 是最终肿瘤浓度重建。

你可以新建一个自己的脚本：

`cp scripts/inverse_masseffect.py scripts/my_inverse_til_patient001.py`

但更合理是基于 inverse / inverse_til 的模式改。由于你的仓库里可能脚本名称不同，你先看：

`ls scripts | grep inverse`

如果你只有 inverse.py，就用它作为第一步。

配置核心参数应该类似：
```
import os, sys
import params as par

p = {}
r = {}

submit_job = False
use_gpu = True

code_dir = "/path/to/GLIA"
patient_dir = "/project/GLIA_USER_DATA/patients/patient_001/aff2template"
out_dir = "/project/GLIA_RESULTS/patient_001/inverse_til"

p['n'] = 64   # 先用 64 跑通；最终改 256
p['output_dir'] = out_dir

p['solver'] = 'sparse_til'
p['model'] = 1
p['regularization'] = 'L1'

p['syn_flag'] = 0
p['patient_labels'] = "[wm=6,gm=5,vt=7,csf=8,ed=2,nec=1,en=4]"
p['p_seg_path'] = os.path.join(patient_dir, "patient_001_seg_aff2template.nii.gz")

# 如果你的脚本需要 d1_path，建议准备 tumor target map
# 或使用 p_seg_path + patient_labels 让 GLIA 从 segmentation 生成 data
# 具体以 solver_log 中是否成功读取 data 为准

p['sparsity_level'] = 5
p['gaussian_selection_mode'] = 1
p['threshold_data_driven'] = 0.1

p['prediction'] = 1
p['pred_times'] = [1.0, 1.2, 1.5]

r['code_path'] = code_dir
r['compute_sys'] = 'local'
r['mpi_tasks'] = 1

par.submit(p, r, submit_job, use_gpu)
```
运行：
```
cd /path/to/GLIA/scripts
python3 my_inverse_til_patient001.py
```
它会生成：
```
/project/GLIA_RESULTS/patient_001/inverse_til/
  solver_config.txt
  job.sh
```
然后运行：
```
cd /path/to/GLIA
mpirun -np 1 build/last/tusolver \
  -config /project/GLIA_RESULTS/patient_001/inverse_til/solver_config.txt
```
如果你用 GPU，通常单 GPU：
```
CUDA_VISIBLE_DEVICES=0 mpirun -np 1 build/last/tusolver \
  -config /project/GLIA_RESULTS/patient_001/inverse_til/solver_config.txt
```
官方 README 也说明，脚本会生成 solver_config.txt，然后用 mpirun/ibrun -np $NUMPROCS build/last/tusolver -config /path/to/solver_config.txt 运行；GPU 单卡通常 NUMPROCS=1。

### inverse_masseffect 的核心输入

完成 inverse_til 后，你进入 mass effect 反演。

inverse_masseffect.py test 脚本里关键参数是：
```
p['d0_path'] = 'testdata/patient_tumor_IC.nc'
p['d1_path'] = 'testdata/patient_tumor.nc'
p['atlas_labels'] = "[wm=6,gm=5,vt=7,csf=8,ed=2,nec=1,en=4]"
p['a_seg_path'] = 'testdata/atlas.nc'
p['mri_path'] = 'testdata/atlas_t1.nc'
p['patient_labels'] = "[wm=6,gm=5,vt=7,csf=8,ed=2,nec=1,en=4]"
p['p_seg_path'] = 'testdata/patient.nc'
p['p_vt_path'] = 'testdata/patient_vt.nc'
p['solver'] = 'mass_effect'
p['model'] = 4
```
这些来自官方 scripts/inverse_masseffect.py。

对应到你的数据：
| GLIA 参数          | 你的文件                                       | 含义                             |
| ---------------- | ------------------------------------------ | ------------------------------ |
| `d0_path`        | `inverse_til/.../c0_rec.nc`                | 上一步估计的 tumor initial condition |
| `d1_path`        | 当前肿瘤 target map，或由患者 segmentation 得到       | 当前观测肿瘤浓度/肿瘤区域                  |
| `p_seg_path`     | `patient_001_seg_aff2template.nii.gz`      | 患者当前分割                         |
| `p_vt_path`      | `patient_001_vt_aff2template.nii.gz`       | 患者脑室 mask                      |
| `a_seg_path`     | `atlas_xxx_seg_aff2template.nii.gz`        | 健康 atlas 分割                    |
| `mri_path`       | `atlas_xxx_t1_aff2template.nii.gz`         | 健康 atlas MRI，用于重建可视化           |
| `atlas_labels`   | `"[wm=6,gm=5,vt=7,csf=8,ed=2,nec=1,en=4]"` | atlas 标签映射                     |
| `patient_labels` | 同上                                         | 患者标签映射                         |
