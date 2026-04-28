---
type: project
status: active
module: GLIA在远程服务器测试文档及操作指南
period:
tags:
  - project
  - experiment
topic:
  - pathology simulation
  - testing
created: 2026-04-25
thesis: "[[GLIA Pathology Simulation Module Overview]]"
---

1. ## Vscode连接错误
    

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=NGZhYjE1YzM5MjdhOTA5ZjM3MWQ3OGZkMzRjZWU1ZmZfOGxhSWJiUEpOSERPbUR0bXZ1WWx4b29YeUZkOXB3Rm5fVG9rZW46V25GcGJ1OUU0b3hVOU94RFN0aGNLb2ZtbmNlXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=NzczYWVmOWU0YmUxN2E1Njc4ZGRhMjI5ZTJmMjNjZWNfUW51N1ZQT0FRNVFPV3hsUGtGSXY3NG94T25FeVB0YWJfVG9rZW46UEttamJuRjdWb3JXQVF4SUtRS2NhaUR0blZjXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

因为 VS Code 1.86 起提高了 Remote Server 对 glibc/libstdc++ 的要求，很多旧服务器就是从那之后开始连不上的

**使用****PowerShell**

  

2. ## 连接SSH
    

3. ### 上传数据
    

```Plain
ssh -p 39159 root@region-46.seetacloud.com
3IjBbx9qvGvK
```

用 SSH 连接到 `region-46.seetacloud.com` 这台服务器，端口是 `39159`，用户名是 `root`。

上传文件时，最简单就是用 **scp**。你本地是 Windows，远程是 Linux，所以流程是：

**在本地** **PowerShell** **里执行** **scp** **上传 → 再 ssh 登录服务器 → 在服务器里解压 zip**

```Plain
scp -P 39159 "D:\Learing_TANG\GraduateProject\Resources\GLIA-master.zip" root@region-46.seetacloud.com:/root/
```

> scp
> 
> 表示使用 SSH 协议复制文件，适合本地和服务器之间传文件。
> 
> -P 39159
> 
> 指定服务器 SSH 端口是 `39159`。
> 
> 你的用户名是 `root`，服务器地址是 `region-46.seetacloud.com`。
> 
> :/root/
> 
> 表示上传到远程服务器的 `/root/` 文件夹。
> 
> 因为你是 root 用户登录，默认家目录就是 `/root/`

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=NzBjMDQyOWE5YWMyMTM4MTZjMzMxMjA4MWIyZTc0YWVfTEhmSW1hc002Ym5najY5alRRREE5VWp5ZkNrRTJIUE9fVG9rZW46Snd3c2JPTmhKb0Q3bnR4SjFlNWNZeVRhbmhmXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=MmM4YTJiYWEzNzRhMTVjNDBkM2YwYjRmZjI1ZGE1ZGVfM0VGaUZwb0x0Q3ZGV3RmNnRCREJxR2V1QXNBQnkySDRfVG9rZW46QjMzSmJyNkxVb3U0Tzl4SW95dWNVM3RzbkJkXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

```Plain
ls -lh /root/
which unzip
cd /root
unzip GLIA-master.zip
```

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=ZWE5NjAwMjdkMThjM2JiZGJmNDQ2MWFiZDIwOGRhYTBfMWZLeUFnUmljcmF1YTQ1UXgyUzE3bXBEcVNiUHpuZklfVG9rZW46WDVTMWI3MThtb2I2Tzh4RjVMb2NvZVV4bjViXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

```Plain
mv GLIA-master GLIA
cd GLIA
ls
```

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=ZjQxOWJhMWMzNmFjOGE1YWU5ZjFiODE4OTg1OWNlYmZfRkhQZnhVZ0w2Y3RDQmRBQlpEdmg0M2Q0MjdZNGUzcHhfVG9rZW46UHpEa2JuRW1Cb1lIS0N4Z0NLZWNjZVZDbmRkXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

  

2. ### 依赖检查
    

```Plain
lsb_release -a
nvidia-smi
nvcc --version
gcc --version
mpicc --version
conda --version
```

> which = 查找某个命令所在路径
> 
> mpicc = MPI C 编译器
> 
> mpicxx = MPI C++ 编译器
> 
> nvcc = NVIDIA CUDA 编译器
> 
> scons = GLIA 使用的构建工具

```Plain
GPU：Tesla V100 32GB
CUDA 编译器：10.2
gcc：7.5.0
conda：4.10.3
mpicc：没有安装
```

```Plain
apt update
apt install -y openmpi-bin libopenmpi-dev
mpicc --version
```

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=YWEwNTZlNDMzZTI1MzYzYTc5OGJlMWFkMjZiZWEzNzFfTGFNU01obnBZU3ZkV1N3aXdHeTVVdE82clVOTUxIamJfVG9rZW46UzZ2RWJMeTlxb2V6ODV4RGd5TmNkdk81bmxiXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=MzM1NzY1ZDdiNzc5NTUzMWZkMjJkNzU4MDFmYjY5MWJfTlBxN0NkN2kzVlVFVFJsMm5rd2lRRWFkVmhnbWZjV2dfVG9rZW46VWh0T2J5Y1o4b21jSE94YmVFN2MyUzc5bmRsXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

3. ### 检查文件
    

```Plain
ls -lh build/last/tusolver
# scons = GLIA 使用的构建工具
pip install scons
```

GLIA 官方文档说明，编译后的二进制程序是：build/last/tusolver

如果显示：`No such file or directory`说明还没有编译成功，需要先编译。

```Plain
scons build=release compiler=mpicxx niftiio=no platform=frontera single_precision=yes gpu=yes multi_gpu=no -j 8
```

> scons = 构建/编译工具
> 
> build=release = 编译发布版本，速度更快
> 
> compiler=mpicxx = 使用 MPI C++ 编译器
> 
> niftiio=no = 不使用 NIFTI 输入输出，这里 testdata 是 .nc 文件
> 
> platform=frontera = 使用官方默认平台配置之一
> 
> single_precision=yes = 使用单精度，官方推荐
> 
> gpu=yes = 编译 GPU 版本
> 
> multi_gpu=no = 不使用多 GPU，官方也说 multi_gpu 已废弃
> 
> -j 8 = 同时使用 8 个编译任务，加快编译

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=MGQxMmNhZGM4NGRiY2MyNjkyNzc3ZDRhYzhlYTQ2M2NfcUJ5Q1d5ZU5BMGJGSVUyMjFLY0F0WThlVkk0RGxTY21fVG9rZW46STl2UGJiQkxUb01aMzN4cTdqaGNDT2xLbmZjXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

**GLIA 仓库里的旧版 SCons 写法和你当前安装的新 SCons/Python 版本不兼容**。

意思是：GLIA 的 `nvcc.py` 里用了旧写法 `env.has_key(...)`，但你现在的 SCons 版本已经不支持 `has_key` 这个方法了。

  

```Plain
cp nvcc.py nvcc.py.bak
nl -ba nvcc.py | sed -n '1,80p'

# Python 3 / 新版 SCons 兼容写法
sed -i "s/if not env.has_key('NVCCCOM'):/if 'NVCCCOM' not in env:/" nvcc.py

# 检查整个 GLIA 项目里是否还有 has_key
grep -R "has_key" -n .

grep -R "has_key" -n . --exclude-dir=__pycache__ --exclude="*.bak"

```

> nl = number lines，给文件每一行加行号
> 
> -ba = number all，所有行都编号
> 
> sed = stream editor，文本流编辑器
> 
> -n '1,80p' = 只显示第 1 到第 80 行

> sed -i = 直接在原文件中修改内容
> 
> s/旧内容/新内容/ = substitute，替换文本
> 
> nvcc.py = 要修改的文件

> grep = 搜索文本
> 
> -R = recursive，递归搜索所有子目录
> 
> -n = 显示行号
> 
> . = 当前目录，也就是 /root/GLIA

has_key全部完成更改！！

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=ODY5ZGI3YjVmZWFlNzE4YjkyN2Q2NTUwMjlhODc3MzRfbExLOFN2SzM0cHhyQkd2NzNTejZjWWRTeHdWSWllZHJfVG9rZW46R3VmVGJsbXNJb2ZtN094NG1ITWM2SkF4blJoXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

意思是：**SCons 没找到 CUDA 的 cuSPARSE 库**。

`cusparse` 是 NVIDIA CUDA 里的一个稀疏矩阵计算库，GLIA 编译 GPU 版本时需要它。你的 `nvcc --version` 已经显示 CUDA 10.2 编译器存在，但 SCons 当前没有正确知道 CUDA 安装目录在哪里，因为它显示：

CUDA_DIR =也就是空的。

```Plain
ls -lh /usr/local
find /usr/local -name "libcusparse*"
```

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDFiZGNmZGY0YmEwZTJkYmFkYTg0ODI0YjhlYWI0NDRfZ3RKV1U4VXlqY1hxVDVNcjc4UFNja1ZvME05aVlDbkNfVG9rZW46TW9FYWJoTlgzb09vS3R4QlplUWNqc2RqbkhmXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

```Bash
export CUDA_DIR=/usr/local/cuda-10.2
export CUDA_HOME=/usr/local/cuda-10.2

export PATH=$CUDA_DIR/bin:$PATH

export CUDA_LIB_DIR=$CUDA_DIR/targets/x86_64-linux/lib
export CUDA_INC_DIR=$CUDA_DIR/targets/x86_64-linux/include

export LD_LIBRARY_PATH=$CUDA_LIB_DIR:$LD_LIBRARY_PATH
export LIBRARY_PATH=$CUDA_LIB_DIR:$LIBRARY_PATH
export CPATH=$CUDA_INC_DIR:$CPATH
export CPLUS_INCLUDE_PATH=$CUDA_INC_DIR:$CPLUS_INCLUDE_PATH
```

> CUDA_DIR = CUDA 安装目录
> 
> CUDA_HOME = 另一种常见 CUDA 目录变量名
> 
> PATH = Linux 查找命令的路径，比如 nvcc
> 
> CUDA_LIB_DIR = CUDA 库文件目录，比如 libcusparse.so
> 
> CUDA_INC_DIR = CUDA 头文件目录，比如 cusparse.h
> 
> LD_LIBRARY_PATH = 程序运行时查找动态库的路径
> 
> LIBRARY_PATH = 编译时查找库文件的路径
> 
> CPATH = 编译时查找 C/C++ 头文件的路径
> 
> CPLUS_INCLUDE_PATH = 编译 C++ 时查找头文件的路径

CUDA 已经识别成功了，但 pnetCDF 没有被找到！！！

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=YjYyMjI3ZjFhZDg4OTQ1YjgxMDNiZDVjMzg2NzI4YTJfUHJlakZpNXd3V053YmZVV1NYZkJIaUdOWHIyODVBTVFfVG9rZW46WGtzT2JzTThVb1NESTJ4VHNoSWNMdXF5bmpaXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

GLIA 的测试数据是 `.nc` 文件，`.nc` 是 netCDF 格式。GLIA 读取这些测试数据时需要 **pnetCDF**。

所以这个报错不是代码坏了，而是 GLIA 编译时没有找到：libpnetcdf.so pnetcdf.h

```Plain
find / -name "pnetcdf.h" 2>/dev/null

# /root/opt = 建议放第三方软件的目录
mkdir -p /root/opt
cd /root/opt
wget https://parallel-netcdf.github.io/Release/pnetcdf-1.11.2.tar.gz

tar -xzf pnetcdf-1.11.2.tar.gz
cd pnetcdf-1.11.2
```

```Plain
./configure --prefix=/root/opt/pnetcdf-1.11.2-install CC=mpicc CXX=mpicxx FC=mpif90
```

> ./configure = 检查系统环境，生成 Makefile
> 
> --prefix=/root/opt/pnetcdf-1.11.2-install = 指定安装到哪里
> 
> CC=mpicc = 用 MPI C 编译器
> 
> CXX=mpicxx = 用 MPI C++ 编译器
> 
> FC=mpif90 = 用 MPI Fortran 编译器

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=ODQwYzViNjhkYjhjODBlNjc3NmUxMDJlN2ZiNjM1Y2Jfa3c3Vk1HUkVSdmNmZUo0enZXQk1pTTg4a2llSEh6Rk1fVG9rZW46UzlyRGIxWFVqb2tsRVV4Nzh1d2NiSXZqbkhoXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

pnetCDF 配置阶段需要 `m4` 工具，但你的系统没有安装。

`m4` 是一个宏处理工具，很多 C/C++/Fortran 科学计算库在 `configure` 阶段都会用到。

```Plain
apt update
apt install -y m4 gawk gfortran
```

> apt = Ubuntu 的软件包管理工具
> 
> update = 更新软件源索引
> 
> install = 安装软件
> 
> -y = 自动回答 yes
> 
> m4 = 宏处理工具，pnetCDF configure 必需
> 
> gawk = GNU awk，文本处理工具
> 
> gfortran = GNU Fortran 编译器

```Plain
# 清理上一次失败的配置
cd /root/opt/pnetcdf-1.11.2
make distclean 2>/dev/null || true


./configure --prefix=/root/opt/pnetcdf-1.11.2-install CC=mpicc CXX=mpicxx FC=mpif90
```

> make distclean = 清理 configure 生成的文件
> 
> 2>/dev/null = 隐藏错误输出
> 
> || true = 即使前面失败，也不要中断

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=MzM4MDRmNWM2ZDhkY2ZhMjc1ZmU2YzRmNjQwY2JmMGNfdUFHN0g3SW81UzQ4ZGRVRUxiSjRKa1RCeUp3Z2ZwaUZfVG9rZW46U1gyYWJGamR2b1o3Zmt4Qmo2SGNnSmtUbkxjXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

**configure 配置成功**，意思是：pnetCDF 已经检查完环境，并生成了后续编译需要的 `Makefile` !!!

```Plain
make -j 8
make install
```

> make = 根据 Makefile 编译源码
> 
> -j 8 = 同时开 8 个编译任务，加快速度

> make install = 把编译好的库、头文件、工具安装到 --prefix 指定的目录

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=YjQwOGUxMjk4NzVlY2IyNzRkZmQ2ODk5N2VkMDg5NzdfZkd5aUJPSndSM3dEaUNNSkdxRmxYdEFSSndQTTR5NUlfVG9rZW46R3ozUWI4NWdMb2ZHdmR4R21CZGNrYThrbmJiXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=NWNhY2UxMzU2ZjRmMTQ5ZGNmNjBkZGY5MTI3ZjNjZDNfUVFMNVN4QkFHc3o3b2JWN0k4UXFISUE5dXFIbDRUeWVfVG9rZW46Wmk3d2JTSEJ1bzZ0and4TWtTQ2NrMzhhbmhkXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

```Plain
ls -lh /root/opt/pnetcdf-1.11.2-install
ls -lh /root/opt/pnetcdf-1.11.2-install/lib
ls -lh /root/opt/pnetcdf-1.11.2-install/include
```

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=MTRkYWU5ZjBiNmRjOWVlMjQwZjNhN2ExNWMwNzVmMGNfYnN5cHZSU3M3RmlCSDhLSUs3WnlQa2RtZ0t6RmtGSU1fVG9rZW46UTliNmJqSlF3b2luOTB4dmJBSWNOdUx0bk9nXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

  

```Bash
export PNETCDF_DIR=/root/opt/pnetcdf-1.11.2-install
export LIBRARY_PATH=$PNETCDF_DIR/lib:$LIBRARY_PATH
export CPATH=$PNETCDF_DIR/include:$CPATH
export CPLUS_INCLUDE_PATH=$PNETCDF_DIR/include:$CPLUS_INCLUDE_PATH
```

```Plain
cd /root/GLIA
scons build=release compiler=mpicxx niftiio=no platform=frontera single_precision=yes gpu=yes multi_gpu=no CUDA_DIR=/usr/local/cuda-10.2 PNETCDF_DIR=/root/opt/pnetcdf-1.11.2-install -j 8
```

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=YzVkNGRkYmJmYzg2NmYzNDAzODNhMmEyMTFmM2IzODVfcVdRQkhua2c1azdwaTRuSW96eFJXQUw0eTBOV0JWeXpfVG9rZW46TnJjNWJuVU5mb29ON2J4MFB3RmNPU01hbjRjXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

`PETSc` 全称是 **Portable, Extensible Toolkit for Scientific Computation**，是科学计算求解器库。GLIA 依赖 PETSc，所以必须安装并配置：

```Plain
apt install -y wget make python3 gfortran bison flex
cd /root/opt

wget https://web.cels.anl.gov/projects/petsc/download/release-snapshots/petsc-3.11.4.tar.gz
tar -xzf petsc-3.11.4.tar.gz
cd petsc-3.11.4
```

```Bash
export CUDA_DIR=/usr/local/cuda-10.2
export CUDA_HOME=/usr/local/cuda-10.2
export CUDA_LIB_DIR=$CUDA_DIR/targets/x86_64-linux/lib
export CUDA_INC_DIR=$CUDA_DIR/targets/x86_64-linux/include

export PNETCDF_DIR=/root/opt/pnetcdf-1.11.2-install

export PATH=$CUDA_DIR/bin:$PATH
export LD_LIBRARY_PATH=$CUDA_LIB_DIR:$PNETCDF_DIR/lib:$LD_LIBRARY_PATH
export LIBRARY_PATH=$CUDA_LIB_DIR:$PNETCDF_DIR/lib:$LIBRARY_PATH
export CPATH=$CUDA_INC_DIR:$PNETCDF_DIR/include:$CPATH
export CPLUS_INCLUDE_PATH=$CUDA_INC_DIR:$PNETCDF_DIR/include:$CPLUS_INCLUDE_PATH
```

```Bash
which nvcc
ls -lh $CUDA_LIB_DIR/libcusparse*
ls -lh $PNETCDF_DIR/lib/libpnetcdf*
```

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=MzIzNTVjNDI2MWM0NjI2NTQzMWU5ODJkM2QxZDlkY2VfSHgzN3VyWjVZTjRtYzlFRmNrSjYyRjBmNVNWWk1WTnBfVG9rZW46Tk1EM2JyZkxIb1NTcDJ4QmJjamNyMzJObktoXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

```Plain
export PETSC_DIR=/root/opt/petsc-3.11.4
export PETSC_ARCH=arch-linux-cuda

./configure \
  --with-cc=mpicc \
  --with-cxx=mpicxx \
  --with-fc=mpif90 \
  --with-cuda=1 \
  --with-cuda-dir=/usr/local/cuda-10.2 \
  --with-debugging=0 \
  --with-shared-libraries=1 \
  --download-fblaslapack=1 \
  --download-metis=1 \
  --download-parmetis=1
```

> ./configure = 检查系统环境并生成编译配置
> 
> --with-cc=mpicc = 使用 MPI C 编译器
> 
> --with-cxx=mpicxx = 使用 MPI C++ 编译器
> 
> --with-fc=mpif90 = 使用 MPI Fortran 编译器
> 
> --with-cuda=1 = 启用 CUDA 支持
> 
> --with-cuda-dir = 指定 CUDA 目录
> 
> --with-debugging=0 = 编译优化版本
> 
> --with-shared-libraries=1 = 生成动态库 libpetsc.so
> 
> --download-fblaslapack=1 = 自动下载/编译 BLAS/LAPACK
> 
> --download-metis=1 = 自动下载/编译 METIS
> 
> --download-parmetis=1 = 自动下载/编译 ParMETIS

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=ZjUxZjBkMGU5OGJjNDhlMmNmOWNkNWZjNjA3ZjhjMWRfQWZ4UG1ORmpUSE1Dd1JDdjhWd29xazgzNU1ZWnc2OEdfVG9rZW46SDU4MGJUUU9qbzJpV2F4Y0FWWGMzZTJCbjViXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

```Plain
make PETSC_DIR=$PETSC_DIR PETSC_ARCH=$PETSC_ARCH all
make PETSC_DIR=$PETSC_DIR PETSC_ARCH=$PETSC_ARCH check
```

> make = 编译
> 
> PETSC_DIR = PETSc 目录
> 
> PETSC_ARCH = PETSc 当前编译配置名称
> 
> all = 编译全部目标

> check = 运行 PETSc 自带的小测试，确认库可用

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=MDgyMDAzOTFiOTNkMjY0MGZlMmZhYWVmYWVmOTQ2NGNfazNLb25ldWJ6UDkyTldFTk5OeGc3b0hlSlFDWUl5VnpfVG9rZW46WDZudWI1ZFd0b2tJZEd4cE5PaGN5TjdTbkFmXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

```Bash
export PETSC_DIR=/root/opt/petsc-3.11.4
export PETSC_ARCH=arch-linux-cuda
export LD_LIBRARY_PATH=$PETSC_DIR/$PETSC_ARCH/lib:$LD_LIBRARY_PATH
export LIBRARY_PATH=$PETSC_DIR/$PETSC_ARCH/lib:$LIBRARY_PATH
export CPATH=$PETSC_DIR/include:$PETSC_DIR/$PETSC_ARCH/include:$CPATH
export CPLUS_INCLUDE_PATH=$PETSC_DIR/include:$PETSC_DIR/$PETSC_ARCH/include:$CPLUS_INCLUDE_PATH
```

```Plain
cd /root/GLIA
scons build=release compiler=mpicxx niftiio=no platform=frontera single_precision=yes gpu=yes multi_gpu=no \
  CUDA_DIR=/usr/local/cuda-10.2 \
  PNETCDF_DIR=/root/opt/pnetcdf-1.11.2-install \
  PETSC_DIR=/root/opt/petsc-3.11.4 \
  PETSC_ARCH=arch-linux-cuda \
  -j 8
```

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=ZGRhOWI0YmM0N2ZhNzc3ZjQ1MTdlOWIyYjQwYjQxMDhfdmRSVEVrR2YyMHJDcEE1NHNlVWpTOE9LY2pUSGs5bktfVG9rZW46TWVyaWJ6aGY5b2UzNnl4VEtvN2NNWUVVblViXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

这个报错的本质是：**GLIA 这次实际在用 double 双精度编译，但 CUDA GPU 代码是按 float 单精度写的**，所以类型对不上

需要重新编译 PETSc 为单精度

```Bash
cd /root/opt/petsc-3.11.4

export PETSC_DIR=/root/opt/petsc-3.11.4
export PETSC_ARCH=arch-linux-cuda-single

export CUDA_DIR=/usr/local/cuda-10.2
export CUDA_HOME=/usr/local/cuda-10.2
export CUDA_LIB_DIR=$CUDA_DIR/targets/x86_64-linux/lib
export CUDA_INC_DIR=$CUDA_DIR/targets/x86_64-linux/include

export PATH=$CUDA_DIR/bin:$PATH
export LD_LIBRARY_PATH=$CUDA_LIB_DIR:$LD_LIBRARY_PATH
export LIBRARY_PATH=$CUDA_LIB_DIR:$LIBRARY_PATH
export CPATH=$CUDA_INC_DIR:$CPATH
export CPLUS_INCLUDE_PATH=$CUDA_INC_DIR:$CPLUS_INCLUDE_PATH
```

```SQL
./configure \
  --with-cc=mpicc \
  --with-cxx=mpicxx \
  --with-fc=mpif90 \
  --with-cuda=1 \
  --with-cuda-dir=/usr/local/cuda-10.2 \
  --with-precision=single \
  --with-debugging=0 \
  --with-shared-libraries=1 \
  --download-fblaslapack=1 \
  --download-metis=1 \
  --download-parmetis=1
 make PETSC_DIR=$PETSC_DIR PETSC_ARCH=$PETSC_ARCH all
```

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=YmYyNTVmNTFkOWJhMjZlMWEyNjNjMmJmZDJiNzY4MDlfWmp0UmwyRnN1N0Qyb250ZWQxYUVoMFNqcnB2U3dxV2hfVG9rZW46S2pqS2JsRUtsbzZrZVd4WFE3TGNBWm8xbnNkXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

```Plain
grep -R "PETSC_USE_REAL_SINGLE" -n /root/opt/petsc-3.11.4/arch-linux-cuda-single/include
```

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=OGFjNzhkOTIyYmFkNGM4OWQ0Yzk5NWM4NTNkOWZmY2ZfY0dMUDY3YzAyb3BxbUdwcDdoNkhOZ015QUhMb3kySEtfVG9rZW46U0JqT2JTT3Rhb2Y1Sm54NkxUYWNVR3M0bmdmXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

```Plain
cd /root/GLIA
scons -c
rm -rf build/release
```

```Bash
export CUDA_DIR=/usr/local/cuda-10.2
export CUDA_HOME=/usr/local/cuda-10.2
export CUDA_LIB_DIR=$CUDA_DIR/targets/x86_64-linux/lib
export CUDA_INC_DIR=$CUDA_DIR/targets/x86_64-linux/include

export PNETCDF_DIR=/root/opt/pnetcdf-1.11.2-install

export PETSC_DIR=/root/opt/petsc-3.11.4
export PETSC_ARCH=arch-linux-cuda-single

export PATH=$CUDA_DIR/bin:$PATH
export LD_LIBRARY_PATH=$PETSC_DIR/$PETSC_ARCH/lib:$PNETCDF_DIR/lib:$CUDA_LIB_DIR:$LD_LIBRARY_PATH
export LIBRARY_PATH=$PETSC_DIR/$PETSC_ARCH/lib:$PNETCDF_DIR/lib:$CUDA_LIB_DIR:$LIBRARY_PATH
export CPATH=$PETSC_DIR/include:$PETSC_DIR/$PETSC_ARCH/include:$PNETCDF_DIR/include:$CUDA_INC_DIR:$CPATH
export CPLUS_INCLUDE_PATH=$PETSC_DIR/include:$PETSC_DIR/$PETSC_ARCH/include:$PNETCDF_DIR/include:$CUDA_INC_DIR:$CPLUS_INCLUDE_PATH
```

```Plain
scons build=release compiler=mpicxx niftiio=no platform=frontera single_precision=yes gpu=yes multi_gpu=no \
  CUDA_DIR=/usr/local/cuda-10.2 \
  PNETCDF_DIR=/root/opt/pnetcdf-1.11.2-install \
  PETSC_DIR=/root/opt/petsc-3.11.4 \
  PETSC_ARCH=arch-linux-cuda-single \
  -j 8
```

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=NDI3NDUzNDk0OTY0YjM5MmVlNGFhMmUwNDg2MzNlZWZfWEtKRmw1Tk9nSnc5QWtYdHJrVHNhazlWUUozTGhKVHFfVG9rZW46VlgwZGJtc0pkb2x0RFZ4NDBEYmMyZjBmbkJlXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

**SCons 编译 CUDA 文件时没有自动加入 OpenMPI 的 include 路径**

```Bash
export MPI_DIR=/usr/lib/x86_64-linux-gnu/openmpi
export MPI_INC_DIR=$MPI_DIR/include
export MPI_LIB_DIR=$MPI_DIR/lib

export CPATH=$MPI_INC_DIR:$CPATH
export CPLUS_INCLUDE_PATH=$MPI_INC_DIR:$CPLUS_INCLUDE_PATH
export LIBRARY_PATH=$MPI_LIB_DIR:$LIBRARY_PATH
export LD_LIBRARY_PATH=$MPI_LIB_DIR:$LD_LIBRARY_PATH
```

```Bash
cd /root/GLIA
scons -c
rm -rf build/release
scons build=release compiler=mpicxx niftiio=no platform=frontera single_precision=yes gpu=yes multi_gpu=no \
  CUDA_DIR=/usr/local/cuda-10.2 \
  MPI_DIR=/usr/lib/x86_64-linux-gnu/openmpi \
  PNETCDF_DIR=/root/opt/pnetcdf-1.11.2-install \
  PETSC_DIR=/root/opt/petsc-3.11.4 \
  PETSC_ARCH=arch-linux-cuda-single \
  -j 8
```

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=MDk5N2Y1MDJlOGYzYjNlM2RhNTRlMDQ4ZThjODRjNmZfT3ZTNXE2N1Y4VU85eXB5SmpBRGViYWE1QXluaVlwUU1fVG9rZW46TXJsZ2JhVEdLb1FXb1Z4VGNJamNuc2Y4bkJmXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

`SCons` 已经完成编译，GLIA 的主程序应该已经生成了！！！

```Plain
ls -lh build/release/tusolver
ls -lh build/last/tusolver
```

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=NjUzMzk2NmU0M2VlMzExOWM3YmZhYWE4YzQ0NjU3ZTlfVDl4eWhCcENvVldLb3VkRk5zS3VXZGhJV2ltVVhqU1dfVG9rZW46Q0U1UmJTd1hRb3ZFWTl4VGZMb2MzTlpwbmVXXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

### 小结

```Bash
cat >> ~/.bashrc << 'EOF'

# GLIA environment
export CUDA_DIR=/usr/local/cuda-10.2
export CUDA_HOME=/usr/local/cuda-10.2
export CUDA_LIB_DIR=$CUDA_DIR/targets/x86_64-linux/lib
export CUDA_INC_DIR=$CUDA_DIR/targets/x86_64-linux/include

export PNETCDF_DIR=/root/opt/pnetcdf-1.11.2-install

export PETSC_DIR=/root/opt/petsc-3.11.4
export PETSC_ARCH=arch-linux-cuda-single

export MPI_DIR=/usr/lib/x86_64-linux-gnu/openmpi
export MPI_INC_DIR=$MPI_DIR/include
export MPI_LIB_DIR=$MPI_DIR/lib

export PATH=$CUDA_DIR/bin:$PATH
export LD_LIBRARY_PATH=$PETSC_DIR/$PETSC_ARCH/lib:$PNETCDF_DIR/lib:$CUDA_LIB_DIR:$MPI_LIB_DIR:$LD_LIBRARY_PATH
export LIBRARY_PATH=$PETSC_DIR/$PETSC_ARCH/lib:$PNETCDF_DIR/lib:$CUDA_LIB_DIR:$MPI_LIB_DIR:$LIBRARY_PATH
export CPATH=$PETSC_DIR/include:$PETSC_DIR/$PETSC_ARCH/include:$PNETCDF_DIR/include:$CUDA_INC_DIR:$MPI_INC_DIR:$CPATH
export CPLUS_INCLUDE_PATH=$PETSC_DIR/include:$PETSC_DIR/$PETSC_ARCH/include:$PNETCDF_DIR/include:$CUDA_INC_DIR:$MPI_INC_DIR:$CPLUS_INCLUDE_PATH
EOF


source ~/.bashrc
```

> cat >> ~/.bashrc = 把下面的内容追加写入 ~/.bashrc
> 
> ~/.bashrc = 每次打开终端时自动执行的配置文件
> 
> EOF = 多行输入的结束标记

> source = 重新读取配置文件，使环境变量马上生效

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=MjBlMzE5NzI5MWEwMDE5ZjI1OGZlYWQzYWU0NzQ5NzlfR1BwMGZsWU5FeVd6TnN3Q091VmJuZjl0ck9sMEUzVFZfVG9rZW46VkpFdGJTVUp1bzVnZlh4M2Nkb2NWT2d3bnVkXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

4. ### 测试数据运行
    

5. #### forward
    

```Plain
cd /root/GLIA
python3 scripts/forward.py

ls -lh results/forward
```

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=NzEzZDI0NzQ2Y2QzNGM1NWE3Yzg0Y2IzMzQwODU0YzNfcFFVNG5aMnVGczJLYTRzNk00YTNsWUxmbnIyVEpaQ3dfVG9rZW46Um1XcmI3MzNTb0QyUkl4ZHFuR2NiaFQ1bk9kXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

```Plain
export CUDA_VISIBLE_DEVICES=0

mpirun --allow-run-as-root -np 1 ./build/release/tusolver -config ./results/forward/solver_config.txt 2>&1 | tee ./results/forward/run_terminal.log
```

> export CUDA_VISIBLE_DEVICES=0 = 只使用第 0 块 GPU
> 
> mpirun = MPI run，用 MPI 启动程序
> 
> --allow-run-as-root = 允许 root 用户运行 MPI
> 
> -np 1 = number of processes，启动 1 个 MPI 进程
> 
> ./build/release/tusolver = GLIA 主程序
> 
> -config = 指定配置文件
> 
> ./results/forward/solver_config.txt = forward.py 生成的配置文件
> 
> 2>&1 = 把错误输出也合并到普通输出
> 
> | = pipe，管道，把前一个命令输出交给后一个命令
> 
> tee = 一边在屏幕显示，一边保存到文件
> 
> run_terminal.log = 保存终端日志

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=NTU0ZTI0NTJhYmE3MjBmZWU0ODM0ZjRjZTI4OTE3OTlfOWl5OVlwTXY3eHV2dVJPVUN0dkM2aXBEMHpvVlJQZWpfVG9rZW46VUxQNmJjM25mb0pveWt4NGVFRmM1T3gwbkRlXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDE1ZGY4NWRiZDNlNGUyZjQzZTIyZDEyYWZkODIzMDhfSHhqS0ZoMVhvSGp0ZnhlUGs4OE5WZmNQeWdLMzlScllfVG9rZW46TGY2TmJmMUh3b3JsbWd4TDhudWNqUTRlbkVlXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

2. #### inverse
    

```Plain
cd /root/GLIA
python3 scripts/inverse.py
export CUDA_VISIBLE_DEVICES=0

mpirun --allow-run-as-root -np 1 ./build/release/tusolver -config ./results/inverse_til/solver_config.txt 2>&1 | tee ./results/inverse_til/run_terminal.log
```

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=NjdmMzk4ZmIzNjViNjA4OWQ5OTk2NDRjYjJhOTAwZDhfaWVlSHRDN1ZCNndETlA4QTNUc0JTenI0TGJiTm90cVZfVG9rZW46SzJCUGJoc003b1JSckp4N0FaN2NYamZjbm9mXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

3. #### inverse_masseffect
    

```Plain
python3 scripts/inverse_masseffect.py
export CUDA_VISIBLE_DEVICES=0

mpirun --allow-run-as-root -np 1 ./build/release/tusolver -config ./results/inverse_masseffect/solver_config.txt 2>&1 | tee ./results/inverse_masseffect/run_terminal.log
```

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=MjVjODc0NDEyYTM2YjQxODM4MjU2OGM5NWVjMGYzZWZfSFBzelVzY05SbERMZ2swb3lSVUxkTTF0Y3ZjTmdncGFfVG9rZW46QU41MmJxQzZCbzhJMXp4SDdDUGNQeTE5bmJiXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

  

3. ## 患者数据测试
    

```Plain
scp -P 39159 -r D:\Learing_TANG\GraduateProject\CodeProject\PathologySimulation\GLIA-master\patientdata\BraTS-GLI-00006-000\GLIA_Input_nc root@region-46.seetacloud.com:/root/GLIA/input/
```

1. #### 创建 no-mass-effect inverse 脚本
    

```Plain
cp scripts/inverse.py scripts/inverse_patient001_no_mass_effect.py
```

```Python
############### === define parameters
p['n'] = 128                           # grid resolution in each dimension
p['output_dir'] = '/root/GLIA/results/patient_001/no_mass_effect/'                         # results path
p['atlas_labels'] = "[wm=6,gm=5,vt=7,csf=8,ed=2,nec=1,en=4]"                               # example (brats): '[wm=6,gm=5,vt=7,csf=8,ed=2,nec=1,en=4]'
p['a_seg_path'] = '/root/GLIA/input/GLIA_Input_nc/patients/patient_001/patient_001_seg.nc'
p['d1_path'] = '/root/GLIA/input/GLIA_Input_nc/patients/patient_001/data.nc'
```

**创建** **`inverse_patient001_no_mass_effect.py`** **的本质就是：保留官方流程，只把官方** **`testdata`** **的路径替换成你自己的数据路径**。

不是改 GLIA 算法，也不是自创流程。它等价于官方教程中的这一步：

python scripts/inverse.py

只是官方脚本默认读取：

testdata/patient.nc

testdata/patient_tumor.nc

而你现在要读取：

/root/GLIA/input/GLIA_Input_nc/patients/patient_001/patient_001_seg.nc

/root/GLIA/input/GLIA_Input_nc/patients/patient_001/data.nc

所以才复制一个新脚本，避免破坏官方原始脚本。

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=YjQ4NDhjYWNmM2FhMjY2YzIwYTI4NDZmMDQ5NTZlOGZfdmQwM2xxQjlCZnE5dmJpZ293aTVuWVAyNjdYZk5OaDNfVG9rZW46TkRHTGI3eUNQb29kZmx4UE5lYWNmcWhGbmtiXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

```Plain
python3 scripts/inverse_patient001_no_mass_effect.py
ls -lh /root/GLIA/results/patient_001/no_mass_effect/
```

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=NWRlZjI0YzhiMmEzM2Y3ODE1MTcyYzRjMjVjZjgzNzNfWFVsWHlnTWJCVWZBN2JwdnQ3aXhYM05ad21aNkhDY1FfVG9rZW46VkdHMGIxZmRHbzdCQWF4S3FiUGN3ZEc3bktmXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

```Plain
mpirun --allow-run-as-root -np 1 build/last/tusolver -config /root/GLIA/results/patient_001/no_mass_effect/solver_config.txt
```

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=NjM5ZDJjM2ZkOWY1OWU0N2YzNjIzMWYzNjQ0NjIzNmRfQW40cFVXdHg3SUU5MUhEWnp4WkgwSjRaVWM1T3ZrdU5fVG9rZW46SDRnRWJMNWtHb21ad1V4dkp4V2NKVE1nblFjXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

```Plain
find /root/GLIA/results/patient_001/no_mass_effect -name "*c0*"
```

  

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=ZjEzYzQyNjU0M2JmNGFiMTBjZGFiYTkxNDFmYTQ2OWRfYnR4MEJWUVJGcUFOM1k2VFR1MlZkdURaZzhJY0c4eUlfVG9rZW46WFpGbmI2ZnJzb3h0WmJ4anlQcGN3N1VPbnVkXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

2. #### 创建 mass-effect inverse 脚本
    

```Plain
cp scripts/inverse_masseffect.py scripts/inverse_patient001_masseffect_atlas001.py
```

```Python
p['n'] = 128                                                                                       # grid resolution in each dimension
p['output_dir']     = '/root/GLIA/results/patient_001/mass_effect_atlas_001/'                      # results path
p['d0_path']        = '/root/GLIA/results/patient_001/no_mass_effect/c0_rec.nc'
p['d1_path']        = '/root/GLIA/input/GLIA_Input_nc/patients/patient_001/data.nc'
p['atlas_labels']   = "[wm=6,gm=5,vt=7,csf=8,ed=2,nec=1,en=4]"                                   
p['a_seg_path']     = '/root/GLIA/input/GLIA_Input_nc/atlases/atlase_001/IXI566-HH-2535-T1_to_patient_altas_seg.nc'
p['mri_path']       = '/root/GLIA/input/GLIA_Input_nc/atlases/atlase_001/IXI566-HH-2535-T1_to_patient_warped.nc'
p['patient_labels'] = "[wm=6,gm=5,vt=7,csf=8,ed=2,nec=1,en=4]"
p['p_seg_path']     = '/root/GLIA/input/GLIA_Input_nc/patients/patient_001/patient_001_seg.nc'
p['p_vt_path']      = '/root/GLIA/input/GLIA_Input_nc/patients/patient_001/patient_001_vt.nc'
```

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=MWEwMWFkZDI2NzI2YTEwNzE3YWVhZWYwNTg3NTVhMWVfOXFVOFcwYU1oR3RQUDBkT0lybjJXc2l1Y2M2Tkt4UHdfVG9rZW46RDFHcmJabjByb2Y3Z2N4VkpTMWMzNzQwbkRmXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

```Plain
python3 scripts/inverse_patient001_masseffect_atlas001.py
ls -lh /root/GLIA/results/patient_001/mass_effect_atlas_001/
```

```Plain
mpirun --allow-run-as-root -np 1 build/last/tusolver -config /root/GLIA/results/patient_001/mass_effect_atlas_001/solver_config.txt
```

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=MWQ3NjZhZTBiZDEwMDdmZGRjYmJhNTZjMTRmZjg4YzlfeWZTc05TalNxRHhxb2lkWERNdWRzNUxLNkJXUUVCUjBfVG9rZW46RWgzdGJoMXlibzJwSU54Rm0zSmNEanlIbmtoXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

![](https://bw23mdjeqeo.feishu.cn/space/api/box/stream/download/asynccode/?code=ODRlMGQxMjA5ZmI3YzE1YWNhYjg3MTI2NDMyNTNjZWRfNHdjWVd4aGZQSjY1VXpzWU1rV3d5ZFlYdUliakIyY2ZfVG9rZW46WEFUZGJOYVFIb2ZpVzF4QkthcWNtYkNabnBiXzE3NzczNTQ5NzY6MTc3NzM1ODU3Nl9WNA)

拷贝文件

```Plain
scp -P 39159 -r root@region-46.seetacloud.com:/root/GLIA/results/patient_001/mass_effect_atlas_001 "D:\Learing_TANG\GraduateProject\CodeProject\PathologySimulation\GLIA-master\patientdata\BraTS-GLI-00006-000\GLIA_Results\"

scp -P 39159 -r root@region-46.seetacloud.com:/root/GLIA/results/patient_001/no_mass_effect "D:\Learing_TANG\GraduateProject\CodeProject\PathologySimulation\GLIA-master\patientdata\BraTS-GLI-00006-000\GLIA_Results\"
```