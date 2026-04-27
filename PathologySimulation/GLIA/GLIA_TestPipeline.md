源数据：
`D:\Learing_TANG\GraduateProject\Data\Brats2024\BraTS_GLI\train\BraTS-GLI-00006-000`
工作路径：
`D:\Learing_TANG\GraduateProject\CodeProject\PathologySimulation\GLIA-master\patientdata`

GLIA数据要求：
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

```


## 肿瘤患者数据
```
patient_001_seg_aff2template.nii.gz
patient_001_vt_aff2template.nii.gz
patient_001_t1ce_aff2template.nii.gz
patient_001_flair_aff2template.nii.gz
```
### 1. Segment
使用nnunet完成肿瘤分割
这里直接使用ground truth
`D:\Learing_TANG\GraduateProject\CodeProject\PathologySimulation\GLIA-master\patientdata\BraTS-GLI-00006-000-seg.nii.gz`


### 2. Inpainting（可选）
使用fastwdm进行修复，使用Fastsurfer-neuroLIT进行修复
1. 二值化seg
生成`BraTS-GLI-00006-000-mask.nii.gz`

2. 自动化docker指令

生成docker指令

```
输入一个病例文件夹路径
自动查找其中唯一的 *-t1n.nii.gz 和 *-mask.nii.gz
自动把 Windows 路径转成 /mnt/d/... 这种 Docker/WSL 风格路径
打印完整的 docker run 指令
```
``` 
& 'D:\Software\envs\neuroLIT\python.exe' 'D:\Learing_TANG\GraduateProject\CodeProject\PathologySimulation\GLIA-master\patientdata\Code\generate_neurolit_docker_command.py' 'D:\Learing_TANG\GraduateProject\CodeProject\PathologySimulation\GLIA-master\patientdata\BraTS-GLI-00006-000' --shell bash

```
开启wsl运行docker
``` PS
wsl
mkdir -p ~/neurolit_weights

cp /mnt/c/Users/admin/Downloads/model_axial.pt ~/neurolit_weights/
cp /mnt/c/Users/admin/Downloads/model_coronal.pt ~/neurolit_weights/
cp /mnt/c/Users/admin/Downloads/model_sagittal.pt ~/neurolit_weights/

ls -lh ~/neurolit_weights
```
进行inpainting
```
docker run --gpus "device=all" --ipc=host --ulimit memlock=-1 --ulimit stack=67108864 --rm -v "/mnt/d/Learing_TANG/GraduateProject/CodeProject/PathologySimulation/GLIA-master/patientdata/BraTS-GLI-00006-000/BraTS-GLI-00006-000-t1n.nii.gz:/mnt/d/Learing_TANG/GraduateProject/CodeProject/PathologySimulation/GLIA-master/patientdata/BraTS-GLI-00006-000/BraTS-GLI-00006-000-t1n.nii.gz:ro" -v "/mnt/d/Learing_TANG/GraduateProject/CodeProject/PathologySimulation/GLIA-master/patientdata/BraTS-GLI-00006-000/BraTS-GLI-00006-000-mask.nii.gz:/mnt/d/Learing_TANG/GraduateProject/CodeProject/PathologySimulation/GLIA-master/patientdata/BraTS-GLI-00006-000/BraTS-GLI-00006-000-mask.nii.gz:ro" -v "/mnt/d/Learing_TANG/GraduateProject/CodeProject/PathologySimulation/GLIA-master/patientdata/BraTS-GLI-00006-000:/mnt/d/Learing_TANG/GraduateProject/CodeProject/PathologySimulation/GLIA-master/patientdata/BraTS-GLI-00006-000" -v "$HOME/neurolit_weights:/usr/local/share/LIT/weights:ro" -u "$(id -u):$(id -g)" neurolit-local:fixed -i "/mnt/d/Learing_TANG/GraduateProject/CodeProject/PathologySimulation/GLIA-master/patientdata/BraTS-GLI-00006-000/BraTS-GLI-00006-000-t1n.nii.gz" -m "/mnt/d/Learing_TANG/GraduateProject/CodeProject/PathologySimulation/GLIA-master/patientdata/BraTS-GLI-00006-000/BraTS-GLI-00006-000-mask.nii.gz" -o "/mnt/d/Learing_TANG/GraduateProject/CodeProject/PathologySimulation/GLIA-master/patientdata/BraTS-GLI-00006-000"

```



### 3. SynthSeg
艾影PC上完成
```
conda activate SyhthSegEnv
cd C:\Users\AiYing212106001\Desktop\Tang_Data\BrainsTissue_SynthSeg\SynthSeg-master
python ./scripts/commands/SynthSeg_predict.py --i "D:\Tang_DATA\GraduateProject\GLIA_Test\BraTS-GLI-00006-000-t1n.nii.gz" --o "D:\Tang_DATA\GraduateProject\GLIA_Test\BraTS-GLI-00006-000-t1n-robust-parc.nii.gz" --robust --parc
```
数据保存在本机
`D:\Learing_TANG\GraduateProject\CodeProject\PathologySimulation\GLIA-master\patientdata\BraTS-GLI-00006-000\BraTS-GLI-00006-000-t1n-robust-parc.nii.gz`

提取其中的`WM=[2, 41] GM=[3, 42] CSF=[24 ] Ventricles=[4, 5, 14, 15, 43, 44]`
GLIA 对 seg 要求为 `[wm=6, gm=5, vt=7, csf=8, ed=2, nec=1, en=4]`
<font color="red">注意：这个病例的原始 BraTS-GLI-00006-000-seg.nii.gz 里增强瘤不是 4，而是 3</font>

得到文件：
`.\GLIA-master\patientdata\BraTS-GLI-00006-000\GLIA_patientdata`
`patient_001_seg.nii.gz`
`patient_001_vt.nii.gz`

## Altas数据
使用IXI数据集，需要进行HD-BET和组织分割


### 1. HD-BET

```
conda activate neuroLIT
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121
pip install hd-bet
```
脚本已经完成并放在 `.\GLIA-master\patientdata\Code\HD_BET\run_hd_bet_ixi.py`
它会在 neuroLIT 环境里调用 hd-bet，默认处理这 3 个文件：
IXI566-HH-2535-T1.nii.gz、IXI567-HH-2536-T1.nii.gz、IXI568-HH-2607-T1.nii.gz，并把剥离后的结果写到 `.\GLIA-master\patientdata\BraTS-GLI-00006-000\IXI_HD_BET`


输出：
```
IXI566-HH-2535-T1.nii.gz
IXI567-HH-2536-T1.nii.gz
IXI568-HH-2607-T1.nii.gz
```

### 2. SynthSeg
在艾影PC上完成SynthSeg分割，得到文件：
```
.\GLIA-master\patientdata\BraTS-GLI-00006-000\IXI_HD_BET\IXI566-HH-2535-t1n-robust-parc.nii.gz
...
```

整合label后得到文件：
```
\GLIA-master\patientdata\BraTS-GLI-00006-000\IXI_HD_BET\IXI568-HH-2607-T1_altas_seg.nii.gz
...
```
***
## Patients 数据与 Altas数据进行配准
以 patient 的空间作为固定空间，避免ce，flair再做调整
流程：
1. t1n.nii.gz 为 fixed image，atlas_001_t1.nii.gz 作为 moving image
  得到`atlas_001_t1_to_patient.nii.gz`
  <font color="red">注意：Image-Image之间进行配准，而不是label-label</font>
2. label 应用形变场，并用 linear interpolation，确保标签为整数
  得到`atlas_001_seg_to_patient.nii.gz`

目标形式
```
atlas_001_t1_to_patient.nii.gz    (应用形变)
atlas_001_seg_to_patient.nii.gz   (应用形变)

patient_001_seg.nii.gz            (不应用形变)
patient_001_vt.nii.gz             (不应用形变)
patient_001_t1ce.nii.gz           (不应用形变)
patient_001_flair.nii.gz          (不应用形变)
```



脚本在 `.\GLIA-master\patientdata\Code\Register\register_ixi_atlas_to_patient.py`
- 在 IXI_HD_BET 中自动配对 atlas T1 和 *-t1n-robust-parc.nii.gz
- 重映射生成仅含 0/5/6/7/8 的 altas_seg
- 用患者 BraTS-GLI-00006-000-t1n.nii.gz 作为 fixed，使用 ants.registration(..., - type_of_transform="SyN")
- 对 altas_seg 用同一 forward transform 做最近邻变换
- 输出 warped atlas T1、warped atlas_seg、transform 文件和日志
- 做标签检查与几何/affine 一致性检查
- 实际输出已经生成在 GLIA_altasdata：
```
IXI566-HH-2535-T1_to_patient_warped.nii.gz
IXI566-HH-2535-T1_to_patient_altas_seg.nii.gz
IXI566-HH-2535-T1_to_patient_0GenericAffine.mat
IXI566-HH-2535-T1_to_patient_1Warp.nii.gz
...
```
对另外两例 IXI567...、IXI568... 也生成了同类文件
中间生成的 atlas 标签文件保存在 IXI_HD_BET：
```
IXI566-HH-2535-T1_altas_seg.nii.gz
IXI567-HH-2536-T1_altas_seg.nii.gz
IXI568-HH-2607-T1_altas_seg.nii.gz
```
汇总日志在 `register_summary.json`。这次运行结果是 3/3 成功，所有 warped altas_seg 标签都只包含 {0,5,6,7,8}，并且 shape / spacing / origin / direction / affine 与患者 fixed image 的 QC 全部通过。

##### 待Codex脚本运行完毕后手动完成以下过程
1. 手动流程
手动整合进`.\GLIA-master\patientdata\BraTS-GLI-00006-000\GLIA_Input`
选择`\GLIA-master\patientdata\BraTS-GLI-00006-000\GLIA_altasdata`：
```
IXI566-HH-2535-T1_to_patient_warped.nii.gz
IXI566-HH-2535-T1_to_patient_altas_seg.nii.gz
```
选择`GLIA_patientdata`和`BraTS-GLI-00006-000`
```
BraTS-GLI-00006-000-t1c.nii.gz
BraTS-GLI-00006-000-t2f.nii.gz
patient_001_seg.nii.gz
patient_001_vt.nii.gz
```
2. 汇总

```
D:.
├─atlases
│  ├─atlase_001
│  │      IXI566-HH-2535-T1_to_patient_altas_seg.nii.gz
│  │      IXI566-HH-2535-T1_to_patient_warped.nii.gz
│  │
│  ├─atlase_002
│  │      IXI567-HH-2536-T1_to_patient_altas_seg.nii.gz
│  │      IXI567-HH-2536-T1_to_patient_warped.nii.gz
│  │
│  └─atlase_003
│          IXI568-HH-2607-T1_to_patient_altas_seg.nii.gz
│          IXI568-HH-2607-T1_to_patient_warped.nii.gz
│
└─patients
    └─patient_001
            BraTS-GLI-00006-000-t1c.nii.gz
            BraTS-GLI-00006-000-t2f.nii.gz
            patient_001_seg.nii.gz
            patient_001_vt.nii.gz
```

3. data.nc 
`data.nc`从你的 patient_001_seg.nii.gz 里提取肿瘤标签：
  `1 = necrosis
  2 = edema
  4 = enhancing tumor`
生成一个 tumor concentration 图：
`
enhancing / necrosis = 1.0
edema = 0.2 
其他 = 0
`
Codex完成生成
- `convert_glia_input_to_nc.py`将 GLIA_Input 下的 .nii.gz 在转 .nc 前都会重采样，强度图用线性插值，标签图（seg、vt、altas_seg）用最近邻插值，避免标签污染
- `datanc_get.py`生成 data.nc 时也会先把 patient_001_seg.nii.gz 重采样
保存为：
`./patients/patient_001/data.nc`

4. c0_rec.nc
`c0_rec.nc`这个不能直接从当前 segmentation 得到。它应该来自 GLIA 的 inverse_til / inverse_nomasseffect。
官方说明里 inverse_til.py 的输出包括 c0_rec.nii.gz 和 c1_rec.nii.gz，其中 c0_rec 是 reconstructed tumor initial condition，最终分辨率结果通常保存在 output_dir/inversion/nx256/obs-1.0/。
需要先跑 no-mass-effect inverse，得到：`c0_rec.nc`

<font color="red">所有nii转nc文件时，重采样至128* 128* 128</font>

***
## 输入GLIA
1. 用 patient_001_seg.nii.gz 生成 data.nc
2. 先跑 inverse_til 得到 c0_rec.nc
3. 用 3 个 atlas 的 warped 文件分别跑 inverse_masseffect
### <font color=red>患者数据的测试流程见飞书 《GLIA 远程服务器SSH》</font>

## GLIA结果分析
相对有潜在临床/科研解释价值的是：
| 文件                          | 价值                       |
| --------------------------- | ------------------------ |
| `c0_rec.nc`                 | 估计肿瘤初始位置，研究肿瘤起源和生长轨迹     |
| `c_rec_final.nc`            | 检查模型是否能重建当前肿瘤            |
| `seg_rec_final.nc`          | 查看模型重建后的组织结构             |
| `displacement_rec_final.nc` | 观察 mass effect 造成的组织位移强度 |
| `disp_x/y/z_rec_final.nc`   | 定量分析三维形变场                |
| `vt_rec_final.nc`           | 观察脑室受压/变形                |
| `mri_rec_final.nc`          | 可视化变形后的 atlas 形态         |
