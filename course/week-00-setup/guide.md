# Week 0：环境搭建

> **目标**：完成所有工具的安装和验证，为后续36周学习做好准备。
> **预计时间**：4-6小时（一次性投入）

---

## 工具全景

| 工具 | 用途 | 优先级 |
|------|------|--------|
| **RISC-V GNU Toolchain** | 编译RISC-V汇编/C程序 | 必须 |
| **Spike + pk** | RISC-V ISA级功能仿真 | 必须 |
| **Python 3.10+** (含numpy/matplotlib) | 自建仿真器 | 必须 |
| **Ripes** | 可视化流水线仿真（GUI） | 强烈推荐 |
| **gem5** | 全系统周期级仿真 | 强烈推荐 |
| **CUDA Toolkit** | GPU编程（Week 21起用） | Phase 5前安装 |
| **CACTI 7.0** | Cache/内存建模 | Phase 2前安装 |
| **Pin (Intel)** | 动态二进制插桩 | 可选 |

---

## Step 1：Python 环境

### 安装 Python 3.10+

```powershell
# 检查是否已安装
python --version

# 若未安装，从 https://www.python.org/downloads/ 下载安装
# 安装时勾选 "Add Python to PATH"
```

### 创建虚拟环境

```powershell
cd d:\LearningCAQA\course
python -m venv venv
.\venv\Scripts\Activate.ps1

# 若 PowerShell 报执行策略错误，先执行：
# Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser

pip install numpy matplotlib
```

### 验证

```powershell
python -c "import numpy; import matplotlib; print('Python env OK')"
```

---

## Step 2：RISC-V GNU Toolchain

### Windows 推荐方式：使用预编译版本

从 SiFive 或 Embecosm 下载 Windows 版 RISC-V 工具链：

```
https://github.com/riscv-collab/riscv-gnu-toolchain/releases
```

或使用 MSYS2：

```powershell
# 在 MSYS2 终端中
pacman -S mingw-w64-x86_64-riscv64-unknown-elf-gcc
```

### 验证

```powershell
riscv64-unknown-elf-gcc --version
```

### 编写 Hello World

创建 `hello.c`：
```c
#include <stdio.h>
int main() {
    printf("Hello, RISC-V!\n");
    return 0;
}
```

编译（注意：无OS环境需要newlib）：
```powershell
riscv64-unknown-elf-gcc -o hello hello.c
```

---

## Step 3：Spike + pk (Proxy Kernel)

### 安装

```powershell
# 克隆仓库
git clone https://github.com/riscv-software-src/riscv-isa-sim.git
git clone https://github.com/riscv-software-src/riscv-pk.git

# 编译 Spike
cd riscv-isa-sim
mkdir build && cd build
../configure --prefix=$HOME/riscv-tools
make -j$(nproc)
make install

# 编译 pk
cd ../../riscv-pk
mkdir build && cd build
../configure --prefix=$HOME/riscv-tools --host=riscv64-unknown-elf
make -j$(nproc)
make install
```

> **Windows 用户提示**：Spike 在原生 Windows 上编译较困难。建议使用 **WSL2 (Ubuntu)** 或 **MSYS2** 环境。

### WSL2 替代方案（推荐）

```bash
# 在 WSL2 Ubuntu 中
sudo apt update
sudo apt install build-essential device-tree-compiler

# 然后按上述步骤编译 Spike
```

### 验证

```bash
spike pk hello
# 输出: Hello, RISC-V!
```

---

## Step 4：Ripes（可视化流水线仿真器）

Ripes 是一个 Qt 图形化应用，可以直观展示5级流水线的运行过程。

### 安装

从 GitHub Releases 下载：
```
https://github.com/mortbopet/Ripes/releases
```

选择 Windows 版本（.exe 安装包）。

### 验证

1. 启动 Ripes
2. 在 Editor 中写入一段 RISC-V 汇编
3. 切换到 Processor 视图，观察流水线各级

---

## Step 5：gem5 全系统仿真器

gem5 是体系结构研究中最常用的周期级仿真器。

### 安装（WSL2 / Linux）

```bash
# 安装依赖
sudo apt install build-essential scons python3-dev python3-pip \
  zlib1g-dev m4 libprotobuf-dev protobuf-compiler libgoogle-perftools-dev

# 克隆 gem5
git clone https://github.com/gem5/gem5.git
cd gem5

# 编译 RISCV 目标
scons build/RISCV/gem5.opt -j$(nproc)
```

> **注意**：首次编译需要 30-60 分钟，取决于机器性能。建议晚上挂着编译。

### 验证

```bash
# 用之前编译的 hello 程序测试
build/RISCV/gem5.opt configs/example/se.py -c /path/to/hello

# 查看输出统计
# 应在输出末尾看到 "Hello, RISC-V!" 和统计信息
```

### 快速测试：配置不同 Cache

```bash
# 带 L1 Cache
build/RISCV/gem5.opt configs/example/se.py -c hello \
  --caches --l1d_size=32kB --l1i_size=32kB

# 带 L1+L2 Cache
build/RISCV/gem5.opt configs/example/se.py -c hello \
  --caches --l2cache --l1d_size=32kB --l1i_size=32kB --l2_size=256kB
```

---

## Step 6：CACTI 7.0（Cache 建模）

CACTI 是 HP 实验室开发的 Cache/内存访问时间和功耗建模工具。

### 安装

```bash
# 下载 CACTI 7.0
# https://github.com/HewlettPackard/cacti

git clone https://github.com/HewlettPackard/cacti.git
cd cacti
make
```

### 验证

```bash
./cacti -infile sample.cfg
# 查看输出中的 Access time, Energy per access 等参数
```

---

## Step 7：CUDA Toolkit（Week 21 前完成即可）

### 安装

从 NVIDIA 官网下载 CUDA Toolkit：
```
https://developer.nvidia.com/cuda-downloads
```

### 验证

```powershell
nvcc --version
```

创建 `hello.cu`：
```cpp
#include <stdio.h>
__global__ void kernel() { printf("Hello from GPU!\n"); }
int main() {
    kernel<<<1,1>>>();
    cudaDeviceSynchronize();
    return 0;
}
```

```powershell
nvcc -o hello_cuda hello.cu
.\hello_cuda.exe
```

---

## Step 8：Intel Pin（可选）

用于动态二进制插桩，分析程序行为。

```
https://www.intel.com/content/www/us/en/developer/articles/tool/pin-a-dynamic-binary-instrumentation-tool.html
```

下载 Windows 版本，解压到 `d:\LearningCAQA\tools\pin\`。

```powershell
# 示例：统计指令数
.\pin.exe -t source\tools\ManualExamples\obj-ia32\inscount0.dll -- .\hello.exe
```

---

## 安装自查清单

完成以下所有项后，方可进入 Week 1：

- [ ] `python --version` 输出 3.10+
- [ ] `riscv64-unknown-elf-gcc --version` 正常输出
- [ ] `spike pk hello` 输出 "Hello, RISC-V!"
- [ ] Ripes 启动成功，能加载 RISC-V 汇编程序
- [ ] gem5 编译成功，SE 模式能运行 hello 程序
- [ ] `python -c "import numpy, matplotlib"` 无错误
- [ ] (可选) `nvcc --version` 正常输出
- [ ] (可选) `pin` 能正常运行

---

## 常见问题

### Q：Windows 上 gem5 编译失败？
**A**：gem5 主要支持 Linux。使用 WSL2 是最佳方案。运行 `wsl --install` 安装 WSL2。

### Q：Spike 找不到 pk？
**A**：确保 pk 和 Spike 安装在同一 prefix 下，或将 pk 路径加入 PATH。

### Q：RISC-V 工具链太大，只想快速上手？
**A**：可以使用在线 RISC-V 模拟器 [VENUS](https://venus.cs61c.org/) 作为临时替代。

---

## 下一步

环境搭建完成后，请进入 **[Phase 1 Week 1：性能的量化语言](../phase-01/week-01/reading-guide.md)**。
