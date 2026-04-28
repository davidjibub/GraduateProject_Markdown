---
type: summary
status: done
module: 远程服务器GLIA配置要求
topic: rule
created: 2026-04-25
thesis: "[[GLIA Pathology Simulation Module Overview]]"
---

下面是把我们前面所有排错过程整理成的一份 **“新服务器从零部署 GLIA 一次性流程”**。你可以把它保存成 `GLIA_deploy.md`，以后换新服务器时照着做。

<font color=red>testdata的测试流程见飞书 《GLIA 远程服务器SSH》
本地文件为：[[GLIA_RemoteServer_SSH]]
---

# GLIA 新服务器从零部署流程

适用环境：

```text
系统：Ubuntu 18.04 / Linux
用户：root
GPU：NVIDIA GPU，例如 Tesla V100
CUDA 编译器：CUDA 10.2
Conda：可有可无
目标：编译并运行 GLIA testdata
```

GLIA 仓库：

```text
https://github.com/ShashankSubramanian/GLIA/tree/master
```

---

# 0. 最终需要安装/配置的东西总览

从零开始，你需要准备：

```text
1. GLIA 源码
2. 基础 Linux 编译工具
3. MPI / OpenMPI
4. CUDA 10.2 环境变量
5. SCons 构建工具
6. pnetCDF 1.11.2
7. PETSc 3.11.4，必须单精度 single precision
8. GLIA 旧代码兼容性补丁：nvcc.py 里的 has_key
9. MPI 头文件路径配置
10. 最终 scons 编译 GLIA
11. 运行 testdata
```

---

# 1. 上传或下载 GLIA 源码

## 方案 A：服务器能联网，推荐直接 clone

```bash
cd /root
git clone https://github.com/ShashankSubramanian/GLIA.git
cd /root/GLIA
```

解释：

```text
cd = change directory，切换目录
git clone = 从 GitHub 下载源码仓库
```

---

## 方案 B：本地 Windows 已经有 zip，用 scp 上传

在 Windows PowerShell 里执行，不是在服务器里执行：

```powershell
scp -P 39159 "D:\你的路径\GLIA-master.zip" root@region-46.seetacloud.com:/root/
```

解释：

```text
scp = secure copy，通过 SSH 上传文件
-P 39159 = 指定 SSH 端口，scp 用大写 P
"D:\你的路径\GLIA-master.zip" = 本地 zip 文件
root@region-46.seetacloud.com = 服务器用户名和地址
:/root/ = 上传到服务器 /root 目录
```

然后登录服务器：

```powershell
ssh -p 39159 root@region-46.seetacloud.com
```

在服务器解压：

```bash
cd /root
apt update
apt install -y unzip
unzip GLIA-master.zip
mv GLIA-master GLIA
cd /root/GLIA
```

解释：

```text
apt = Ubuntu 软件包管理工具
unzip = 解压 zip 文件
mv = move，移动或重命名文件夹
```

---

# 2. 安装基础工具

```bash
apt update
apt install -y \
  build-essential \
  gcc g++ gfortran \
  make cmake git wget curl \
  python3 python3-pip \
  m4 gawk bison flex \
  zlib1g-dev \
  openmpi-bin libopenmpi-dev
```

解释：

```text
build-essential = Ubuntu 常用 C/C++ 编译工具集合
gcc/g++ = C/C++ 编译器
gfortran = Fortran 编译器，PETSc/pnetCDF 可能需要
make/cmake = 编译工具
git = 下载 GitHub 仓库
wget/curl = 下载文件
python3/python3-pip = Python 和 pip
m4/gawk/bison/flex = configure 阶段常用工具
zlib1g-dev = zlib 开发库
openmpi-bin/libopenmpi-dev = MPI 运行和开发库
```

检查 MPI：

```bash
which mpicc
which mpicxx
which mpif90
mpicc --version
```

解释：

```text
which = 查看命令所在路径
mpicc = MPI C 编译器
mpicxx = MPI C++ 编译器
mpif90 = MPI Fortran 编译器
```

---

# 3. 安装 SCons

GLIA 使用 `scons` 编译。

```bash
python3 -m pip install --user scons
export PATH=$HOME/.local/bin:$PATH
```

解释：

```text
pip install = 安装 Python 包
--user = 安装到当前用户目录
PATH = Linux 查找命令的路径
```

检查：

```bash
scons --version
```

如果 `scons` 太新导致奇怪兼容问题，可以降级：

```bash
python3 -m pip uninstall -y scons
python3 -m pip install --user "scons==3.1.2"
export PATH=$HOME/.local/bin:$PATH
```

---

# 4. 配置 CUDA 10.2

先检查：

```bash
which nvcc
nvcc --version
nvidia-smi
ls -lh /usr/local
```

你之前的机器是：

```text
nvcc = /usr/local/cuda/bin/nvcc
/usr/local/cuda -> cuda-10.2
实际 CUDA 目录 = /usr/local/cuda-10.2
```

CUDA 库实际在：

```text
/usr/local/cuda-10.2/targets/x86_64-linux/lib
```

配置环境变量：

```bash
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

检查 CUDA 库：

```bash
ls -lh $CUDA_LIB_DIR/libcusparse*
ls -lh $CUDA_LIB_DIR/libcublas*
ls -lh $CUDA_LIB_DIR/libcufft*
ls -lh $CUDA_LIB_DIR/libcudart*
```

如果 GLIA 找不到 CUDA 库，可以创建兼容软链接：

```bash
ln -s /usr/local/cuda-10.2/targets/x86_64-linux/lib /usr/local/cuda-10.2/lib64
```

解释：

```text
ln = link，创建链接
-s = symbolic link，软链接
lib64 = 很多旧项目默认查找 CUDA 库的位置
```

---

# 5. 安装 pnetCDF 1.11.2

GLIA 的 `testdata` 是 `.nc` 文件，需要 pnetCDF。

```bash
mkdir -p /root/opt
cd /root/opt

wget https://parallel-netcdf.github.io/Release/pnetcdf-1.11.2.tar.gz
tar -xzf pnetcdf-1.11.2.tar.gz
cd pnetcdf-1.11.2
```

解释：

```text
mkdir -p = 创建目录，目录已存在也不报错
wget = 下载文件
tar -xzf = 解压 .tar.gz 文件
```

配置 pnetCDF：

```bash
./configure --prefix=/root/opt/pnetcdf-1.11.2-install CC=mpicc CXX=mpicxx --disable-fortran --enable-shared
```

解释：

```text
./configure = 检查系统环境并生成 Makefile
--prefix = 指定安装目录
CC=mpicc = 用 MPI C 编译器
CXX=mpicxx = 用 MPI C++ 编译器
--disable-fortran = 禁用 Fortran 接口，减少报错可能
--enable-shared = 生成动态库 libpnetcdf.so
```

编译安装：

```bash
make -j 8
make install
```

解释：

```text
make = 编译
-j 8 = 同时使用 8 个编译任务
make install = 安装到 --prefix 指定目录
```

配置环境变量：

```bash
export PNETCDF_DIR=/root/opt/pnetcdf-1.11.2-install
export LD_LIBRARY_PATH=$PNETCDF_DIR/lib:$LD_LIBRARY_PATH
export LIBRARY_PATH=$PNETCDF_DIR/lib:$LIBRARY_PATH
export CPATH=$PNETCDF_DIR/include:$CPATH
export CPLUS_INCLUDE_PATH=$PNETCDF_DIR/include:$CPLUS_INCLUDE_PATH
```

检查：

```bash
ls -lh $PNETCDF_DIR/lib/libpnetcdf*
ls -lh $PNETCDF_DIR/include/pnetcdf.h
```

---

# 6. 安装 PETSc 3.11.4，必须单精度

GLIA GPU 编译时使用 `float` 单精度 CUDA/CUBLAS 代码，所以 PETSc 也必须使用：

```text
--with-precision=single
```

否则会出现：

```text
cannot convert ‘ScalarType* {aka double*}’ to ‘const float*’
```

下载 PETSc：

```bash
mkdir -p /root/opt
cd /root/opt

wget https://web.cels.anl.gov/projects/petsc/download/release-snapshots/petsc-3.11.4.tar.gz
tar -xzf petsc-3.11.4.tar.gz
cd petsc-3.11.4
```

设置变量：

```bash
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

配置 PETSc：

```bash
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
```

解释：

```text
--with-cc=mpicc = 使用 MPI C 编译器
--with-cxx=mpicxx = 使用 MPI C++ 编译器
--with-fc=mpif90 = 使用 MPI Fortran 编译器
--with-cuda=1 = 启用 CUDA
--with-cuda-dir = 指定 CUDA 路径
--with-precision=single = 使用单精度 float
--with-debugging=0 = 编译优化版本
--with-shared-libraries=1 = 生成动态库 libpetsc.so
--download-fblaslapack=1 = 自动下载 BLAS/LAPACK
--download-metis=1 = 自动下载 METIS
--download-parmetis=1 = 自动下载 ParMETIS
```

编译 PETSc：

```bash
make PETSC_DIR=$PETSC_DIR PETSC_ARCH=$PETSC_ARCH all
```

可选检查：

```bash
make PETSC_DIR=$PETSC_DIR PETSC_ARCH=$PETSC_ARCH check
```

确认 PETSc 是单精度：

```bash
grep -R "PETSC_USE_REAL_SINGLE" -n /root/opt/petsc-3.11.4/arch-linux-cuda-single/include
```

期望看到：

```text
#define PETSC_USE_REAL_SINGLE 1
```

配置 PETSc 环境变量：

```bash
export PETSC_DIR=/root/opt/petsc-3.11.4
export PETSC_ARCH=arch-linux-cuda-single

export LD_LIBRARY_PATH=$PETSC_DIR/$PETSC_ARCH/lib:$LD_LIBRARY_PATH
export LIBRARY_PATH=$PETSC_DIR/$PETSC_ARCH/lib:$LIBRARY_PATH
export CPATH=$PETSC_DIR/include:$PETSC_DIR/$PETSC_ARCH/include:$CPATH
export CPLUS_INCLUDE_PATH=$PETSC_DIR/include:$PETSC_DIR/$PETSC_ARCH/include:$CPLUS_INCLUDE_PATH
```

检查：

```bash
ls -lh $PETSC_DIR/$PETSC_ARCH/lib/libpetsc*
ls -lh $PETSC_DIR/include
ls -lh $PETSC_DIR/$PETSC_ARCH/include
```

---

# 7. 配置 MPI 头文件路径

后面 GLIA 编译 CUDA `.cu` 文件时可能找不到：

```text
mpi.h
```

先查找：

```bash
find /usr -name "mpi.h" 2>/dev/null
```

Ubuntu OpenMPI 常见位置：

```text
/usr/lib/x86_64-linux-gnu/openmpi/include/mpi.h
```

配置：

```bash
export MPI_DIR=/usr/lib/x86_64-linux-gnu/openmpi
export MPI_INC_DIR=$MPI_DIR/include
export MPI_LIB_DIR=$MPI_DIR/lib

export CPATH=$MPI_INC_DIR:$CPATH
export CPLUS_INCLUDE_PATH=$MPI_INC_DIR:$CPLUS_INCLUDE_PATH
export LIBRARY_PATH=$MPI_LIB_DIR:$LIBRARY_PATH
export LD_LIBRARY_PATH=$MPI_LIB_DIR:$LD_LIBRARY_PATH
```

解释：

```text
MPI_DIR = OpenMPI 安装目录
MPI_INC_DIR = MPI 头文件目录
MPI_LIB_DIR = MPI 库目录
CPATH = 编译时查找 C/C++ 头文件
CPLUS_INCLUDE_PATH = 编译 C++ 时查找头文件
LIBRARY_PATH = 编译链接时查找库
LD_LIBRARY_PATH = 程序运行时查找动态库
```

检查：

```bash
mpicxx --showme:compile
mpicxx --showme:link
```

---

# 8. 修复 GLIA 旧 SCons 代码兼容问题

进入 GLIA：

```bash
cd /root/GLIA
```

备份：

```bash
cp nvcc.py nvcc.py.bak
```

解释：

```text
cp = copy，复制文件
.bak = backup，备份文件
```

修改旧写法：

```bash
sed -i "s/if not env.has_key('NVCCCOM'):/if 'NVCCCOM' not in env:/" nvcc.py
```

解释：

```text
sed -i = 直接修改文件
has_key = Python/SCons 旧写法，新版 SCons 不支持
```

删除 Python 缓存：

```bash
rm -rf __pycache__
```

解释：

```text
rm = remove，删除
-r = recursive，递归删除目录
-f = force，强制删除
__pycache__ = Python 自动生成的缓存目录
```

检查：

```bash
grep -R "has_key" -n . --exclude-dir=__pycache__ --exclude="*.bak"
```

如果没有输出，说明修好了。

---

# 9. 把所有环境变量写入 `.bashrc`

为了避免下次登录丢失配置，写入：

```bash
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

export PATH=$CUDA_DIR/bin:$HOME/.local/bin:$PATH

export LD_LIBRARY_PATH=$PETSC_DIR/$PETSC_ARCH/lib:$PNETCDF_DIR/lib:$CUDA_LIB_DIR:$MPI_LIB_DIR:$LD_LIBRARY_PATH
export LIBRARY_PATH=$PETSC_DIR/$PETSC_ARCH/lib:$PNETCDF_DIR/lib:$CUDA_LIB_DIR:$MPI_LIB_DIR:$LIBRARY_PATH
export CPATH=$PETSC_DIR/include:$PETSC_DIR/$PETSC_ARCH/include:$PNETCDF_DIR/include:$CUDA_INC_DIR:$MPI_INC_DIR:$CPATH
export CPLUS_INCLUDE_PATH=$PETSC_DIR/include:$PETSC_DIR/$PETSC_ARCH/include:$PNETCDF_DIR/include:$CUDA_INC_DIR:$MPI_INC_DIR:$CPLUS_INCLUDE_PATH

EOF
```

解释：

```text
cat >> ~/.bashrc = 把下面内容追加写入 bash 启动配置
EOF = 多行输入结束标记
```

立即生效：

```bash
source ~/.bashrc
```

解释：

```text
source = 重新读取配置文件
```

---

# 10. 编译 GLIA

进入 GLIA：

```bash
cd /root/GLIA
```

清理旧编译结果：

```bash
scons -c
rm -rf build/release
```

解释：

```text
scons -c = clean，清理 SCons 编译产物
rm -rf build/release = 删除 release 编译目录
```

正式编译：

```bash
scons build=release compiler=mpicxx niftiio=no platform=frontera single_precision=yes gpu=yes multi_gpu=no \
  CUDA_DIR=/usr/local/cuda-10.2 \
  MPI_DIR=/usr/lib/x86_64-linux-gnu/openmpi \
  PNETCDF_DIR=/root/opt/pnetcdf-1.11.2-install \
  PETSC_DIR=/root/opt/petsc-3.11.4 \
  PETSC_ARCH=arch-linux-cuda-single \
  -j 8
```

解释：

```text
scons = GLIA 使用的构建工具
build=release = 编译 release 版本
compiler=mpicxx = 使用 MPI C++ 编译器
niftiio=no = 不启用 NIFTI，先跑 .nc testdata
platform=frontera = 使用 GLIA 自带平台配置
single_precision=yes = GLIA 使用单精度
gpu=yes = 启用 GPU/CUDA
multi_gpu=no = 不启用多 GPU
CUDA_DIR = CUDA 路径
MPI_DIR = MPI 路径
PNETCDF_DIR = pnetCDF 路径
PETSC_DIR = PETSc 路径
PETSC_ARCH = PETSc 编译架构
-j 8 = 8 线程并行编译
```

成功标志：

```text
scons: done building targets.
```

检查主程序：

```bash
ls -lh build/release/tusolver
```

---

# 11. 检查动态库

```bash
ldd build/release/tusolver | grep "not found"
```

解释：

```text
ldd = list dynamic dependencies，列出程序依赖的动态库
grep "not found" = 只显示找不到的库
```

如果没有任何输出，说明动态库正常。

---

# 12. 运行 GLIA testdata：forward 测试

先确认 testdata：

```bash
cd /root/GLIA
ls -lh testdata
```

生成 forward 配置文件：

```bash
python3 scripts/forward.py
```

解释：

```text
python3 = 使用 Python 3
scripts/forward.py = GLIA 官方 forward 测试脚本
```

检查生成结果：

```bash
ls -lh results/forward
```

运行：

```bash
export CUDA_VISIBLE_DEVICES=0

mpirun --allow-run-as-root -np 1 ./build/release/tusolver -config ./results/forward/solver_config.txt 2>&1 | tee ./results/forward/run_terminal.log
```

解释：

```text
CUDA_VISIBLE_DEVICES=0 = 使用第 0 块 GPU
mpirun = MPI run，启动 MPI 程序
--allow-run-as-root = 允许 root 用户运行 MPI
-np 1 = number of processes，启动 1 个进程
./build/release/tusolver = GLIA 主程序
-config = 指定配置文件
2>&1 = 错误输出合并到普通输出
| = 管道，把前一个命令输出传给后一个命令
tee = 一边显示，一边保存日志
```

查看日志：

```bash
tail -n 80 results/forward/run_terminal.log
```

解释：

```text
tail = 查看文件末尾
-n 80 = 显示最后 80 行
```

---

# 13. 运行 inverse 测试

forward 成功后再运行：

```bash
cd /root/GLIA
python3 scripts/inverse.py
ls -lh results/inverse_til

export CUDA_VISIBLE_DEVICES=0
mpirun --allow-run-as-root -np 1 ./build/release/tusolver -config ./results/inverse_til/solver_config.txt 2>&1 | tee ./results/inverse_til/run_terminal.log

tail -n 80 results/inverse_til/run_terminal.log
```

---

# 14. 运行 inverse_masseffect 测试

最后再跑更复杂的：

```bash
cd /root/GLIA
python3 scripts/inverse_masseffect.py
ls -lh results/inverse_masseffect

export CUDA_VISIBLE_DEVICES=0
mpirun --allow-run-as-root -np 1 ./build/release/tusolver -config ./results/inverse_masseffect/solver_config.txt 2>&1 | tee ./results/inverse_masseffect/run_terminal.log

tail -n 80 results/inverse_masseffect/run_terminal.log
```

---

# 15. 监控 GPU

另开一个 SSH 窗口：

```powershell
ssh -p 39159 root@region-46.seetacloud.com
```

服务器里执行：

```bash
watch -n 1 nvidia-smi
```

解释：

```text
watch = 重复执行命令
-n 1 = 每 1 秒执行一次
nvidia-smi = 查看 NVIDIA GPU 使用情况
```

退出：

```text
Ctrl + C
```

---

# 16. 常见错误对应解决方法

## 16.1 `has_key not found`

错误：

```text
AttributeError: Builder or other environment method 'has_key' not found
```

解决：

```bash
cd /root/GLIA
cp nvcc.py nvcc.py.bak
sed -i "s/if not env.has_key('NVCCCOM'):/if 'NVCCCOM' not in env:/" nvcc.py
rm -rf __pycache__
```

---

## 16.2 `cusparse not found`

错误：

```text
ERROR: Library 'cusparse' not found!
```

原因：`CUDA_DIR` 没设置，或者 CUDA 库实际在 `targets/x86_64-linux/lib`。

解决：

```bash
export CUDA_DIR=/usr/local/cuda-10.2
export CUDA_LIB_DIR=$CUDA_DIR/targets/x86_64-linux/lib
export LD_LIBRARY_PATH=$CUDA_LIB_DIR:$LD_LIBRARY_PATH
export LIBRARY_PATH=$CUDA_LIB_DIR:$LIBRARY_PATH
```

必要时：

```bash
ln -s /usr/local/cuda-10.2/targets/x86_64-linux/lib /usr/local/cuda-10.2/lib64
```

---

## 16.3 `pnetcdf not found`

错误：

```text
ERROR: Library 'pnetcdf' not found!
```

解决：安装 pnetCDF，然后编译 GLIA 时加：

```bash
PNETCDF_DIR=/root/opt/pnetcdf-1.11.2-install
```

---

## 16.4 `petsc not found`

错误：

```text
ERROR: Library 'petsc' not found!
```

解决：安装 PETSc，并编译 GLIA 时加：

```bash
PETSC_DIR=/root/opt/petsc-3.11.4
PETSC_ARCH=arch-linux-cuda-single
```

---

## 16.5 `ScalarType* {aka double*} cannot convert to float*`

错误：

```text
cannot convert ‘ScalarType* {aka double*}’ to ‘const float*’
```

原因：PETSc 是 double，GLIA GPU 代码需要 single。

解决：重新编译 PETSc，必须加：

```bash
--with-precision=single
```

---

## 16.6 `mpi.h: No such file or directory`

错误：

```text
fatal error: mpi.h: No such file or directory
```

解决：

```bash
export MPI_DIR=/usr/lib/x86_64-linux-gnu/openmpi
export MPI_INC_DIR=$MPI_DIR/include
export CPATH=$MPI_INC_DIR:$CPATH
export CPLUS_INCLUDE_PATH=$MPI_INC_DIR:$CPLUS_INCLUDE_PATH
```

编译时加：

```bash
MPI_DIR=/usr/lib/x86_64-linux-gnu/openmpi
```

---

# 17. 一键编译 GLIA 前的最终环境检查

在正式 `scons` 前，建议执行：

```bash
which nvcc
nvcc --version

which mpicc
which mpicxx
which mpif90

echo $CUDA_DIR
echo $PNETCDF_DIR
echo $PETSC_DIR
echo $PETSC_ARCH
echo $MPI_DIR

ls -lh $CUDA_LIB_DIR/libcusparse*
ls -lh $PNETCDF_DIR/lib/libpnetcdf*
ls -lh $PETSC_DIR/$PETSC_ARCH/lib/libpetsc*
find /usr -name "mpi.h" 2>/dev/null
```

如果这些都正常，再执行 GLIA 编译命令。

---

# 18. 最终核心命令总结

新服务器上完整部署成功后，最终 GLIA 编译命令是：

```bash
cd /root/GLIA

scons build=release compiler=mpicxx niftiio=no platform=frontera single_precision=yes gpu=yes multi_gpu=no \
  CUDA_DIR=/usr/local/cuda-10.2 \
  MPI_DIR=/usr/lib/x86_64-linux-gnu/openmpi \
  PNETCDF_DIR=/root/opt/pnetcdf-1.11.2-install \
  PETSC_DIR=/root/opt/petsc-3.11.4 \
  PETSC_ARCH=arch-linux-cuda-single \
  -j 8
```

运行 forward testdata 的核心命令是：

```bash
cd /root/GLIA

python3 scripts/forward.py

export CUDA_VISIBLE_DEVICES=0

mpirun --allow-run-as-root -np 1 ./build/release/tusolver -config ./results/forward/solver_config.txt 2>&1 | tee ./results/forward/run_terminal.log
```

成功编译的标志：

```text
scons: done building targets.
```

成功运行的初步判断：

```bash
tail -n 80 results/forward/run_terminal.log
```

里面没有：

```text
ERROR
Segmentation fault
CUDA error
MPI error
not found
```
