# Google Breakpad 崩溃分析工具包

Google Breakpad 是一个用于生成和分析应用程序崩溃转储文件的跨平台库。本目录包含了 Breakpad 的完整工具集，用于符号提取、崩溃分析和调试。

## 目录结构

```
breakpad/
├── README.md          # 本文档
├── bin/               # 可执行工具目录
│   ├── core2md                # 核心转储转换工具
│   ├── dump_syms              # 符号提取工具（Linux/Windows）
│   ├── dump_syms_mac          # 符号提取工具（macOS 专用）
│   ├── microdump_stackwalk    # 微型转储堆栈遍历工具
│   ├── minidump_dump          # Minidump 内容查看工具
│   ├── minidump_stackwalk     # 堆栈遍历工具
│   ├── minidump_upload        # 转储上传工具
│   ├── minidump-2-core        # 转储转换工具
│   ├── pid2md                 # 进程ID转储工具
│   └── sym_upload             # 符号上传工具
├── depot_tools/       # Google depot_tools 工具集
├── include/           # 头文件目录
├── lib/               # 库文件目录
├── libexec/           # 辅助可执行文件目录
├── share/             # 共享资源目录
└── src/               # 源码符号链接（gclient 模式）
```

## Windows平台文件说明

在Windows系统中进行崩溃分析，主要涉及两种特定文件类型：

### Minidump文件（.dmp）

**概述**：
- Minidump是Windows系统上应用程序崩溃时生成的转储文件
- 包含崩溃时的内存状态、线程信息、调用堆栈和异常信息
- 文件大小通常比完整内存转储小得多，便于传输和存储
- 文件扩展名通常为`.dmp`

**特点**：
- 包含崩溃时所有线程的调用栈
- 包含引发崩溃的异常记录
- 包含加载的模块列表（DLL和EXE）
- 包含系统信息和进程信息
- 可能包含选定的内存区域内容

**生成方式**：
- Windows错误报告（WER）自动生成
- 通过程序内崩溃处理器（如Breakpad或CrashRpt）生成
- 使用任务管理器手动创建进程的Dump文件
- 使用DebugDiag、ProcDump等工具生成
- 通过Windows调试工具（WinDbg）生成

**DMP文件类型**：
- 迷你转储（MiniDump）：最常用，包含基本调试信息，体积小
- 标准转储（StandardDump）：包含更多内存信息
- 完整转储（FullDump）：包含进程的所有内存，体积大

**结构组成**：
- 头部信息（Header）：标识符、版本、流数量等
- 目录（Directory）：指向各数据流的指针
- 数据流（Streams）：
  * 系统信息流：操作系统版本、处理器类型等
  * 线程列表流：线程ID、寄存器状态等
  * 模块列表流：已加载模块信息（路径、大小、时间戳等）
  * 内存列表流：内存区域数据
  * 异常流：崩溃原因、异常代码、异常地址等
  * 其他辅助信息流

### 程序数据库文件（.pdb）

**概述**：
- PDB（Program Database）是Windows平台上的调试符号文件
- 包含源代码与二进制代码之间的映射关系
- 用于解析崩溃堆栈中的地址为可读的函数名和行号
- 由Visual Studio或其他编译器在构建时生成

**重要性**：
- 没有匹配的PDB文件，崩溃堆栈将只显示地址而非函数名
- PDB必须与对应的二进制文件（版本、编译时间）精确匹配
- 企业开发应建立中央符号存储库保存所有版本的PDB

**PDB文件内容**：
- 公共符号信息：函数名、变量名、类型定义等
- 私有符号信息：函数实现细节、本地变量等
- 源代码映射：将代码地址与源代码文件和行号关联
- 类型信息：结构体、类、枚举等类型的详细信息
- 编译单元信息：关于编译环境和编译选项的数据

**PDB生成方式**：
- **Visual Studio**：
  * 启用：项目属性 → C/C++ → 常规 → 调试信息格式 → "程序数据库 (/Zi)"或"编辑并继续 (/ZI)"
  * 位置：项目属性 → 链接器 → 调试 → 生成程序数据库文件 → 指定路径

- **命令行编译**：
  * MSVC编译器：`cl /Zi /Od /MDd source.cpp /link /DEBUG /PDB:output.pdb`
  * 链接器选项：`/DEBUG` 保留调试信息，`/PDB:filename.pdb` 指定PDB文件名

- **CMake构建系统**：
  ```cmake
  set(CMAKE_CXX_FLAGS_DEBUG "/Zi /Od /MDd")
  set(CMAKE_SHARED_LINKER_FLAGS_DEBUG "/DEBUG /PDB:${PROJECT_NAME}.pdb")
  set(CMAKE_EXE_LINKER_FLAGS_DEBUG "/DEBUG /PDB:${PROJECT_NAME}.pdb")
  ```

**PDB文件管理**：
- **版本控制**：
  * 不建议将PDB文件加入版本控制（体积大且频繁变化）
  * 应使用符号服务器或专门的存储解决方案
  * 与构建版本关联的唯一标识符（如构建号或Git提交哈希）

- **符号服务器设置**：
  * 微软推荐使用SymStore工具管理符号存储库
  * 目录结构：`symbol_server/product_name/GUID/filename.pdb`
  * 添加PDB：`symstore add /f path\to\file.pdb /s \\server\symbols /t product_name /v version`

- **匹配验证**：
  * PDB与二进制文件通过GUID匹配
  * 使用`dumpbin /headers file.exe`查看时间戳和GUID
  * PDB调试信息头部包含相同的GUID

**调试环境设置**：
- **Visual Studio**：
  * 调试 → 选项 → 符号 → 添加符号服务器路径
  * 支持本地路径和网络UNC路径
  * 可设置符号缓存目录减少网络传输

- **WinDbg**：
  * `.sympath+ \\server\symbols`
  * `.symfix+ C:\SymbolCache`
  * `.reload`加载所有模块的符号

## 工具介绍

### bin/ 目录下的核心工具

#### 1. dump_syms - 符号提取工具（Linux/Windows）

**功能**: 从二进制文件中提取调试符号信息，生成 Breakpad 格式的符号文件

**用法**:
```bash
./bin/dump_syms <binary_file> > symbols.sym
./bin/dump_syms /path/to/your/app > app.sym
```

**支持的文件格式**:
- ELF 可执行文件和共享库（Linux）
- Mach-O 文件（macOS）
- PE/COFF 文件（Windows，通过交叉编译）

**输出格式**:
- Breakpad 符号格式 (.sym 文件)
- 包含函数名、源文件路径、行号信息
- 用于后续的崩溃堆栈解析

**示例**:
```bash
# 提取应用程序符号
./bin/dump_syms /usr/bin/myapp > myapp.sym

# 提取动态库符号
./bin/dump_syms /usr/lib/libmylib.so > libmylib.so.sym

# 批量处理
find /path/to/binaries -name "*.so" -exec ./bin/dump_syms {} \; > all_symbols.sym
```

**Windows平台特殊用法**:
```bash
# 提取Windows可执行文件符号（需要PDB文件）
./bin/dump_syms C:/Path/To/app.exe > app.sym

# 直接从PDB文件提取符号
./bin/dump_syms C:/Path/To/app.pdb > app.sym

# 批量处理Windows DLL文件
for %f in (C:\Windows\System32\*.dll) do ./bin/dump_syms "%f" > symbols\%~nf.sym
```

#### 2. minidump_stackwalk - 堆栈遍历工具

**功能**: 分析 minidump 文件，结合符号文件生成可读的崩溃报告

**用法**:
```bash
./bin/minidump_stackwalk <minidump_file> <symbols_directory>
./bin/minidump_stackwalk crash.dmp ./symbols/
```

**输出信息**:
- 崩溃时的调用堆栈
- 寄存器状态
- 内存映射信息
- 线程信息
- 异常详情

**符号目录结构**:
```
symbols/
├── module_name/
│   ├── version_id/
│   │   └── module_name.sym
│   └── another_version/
│       └── module_name.sym
└── another_module/
    └── version_id/
        └── another_module.sym
```

**示例**:
```bash
# 基本分析
./bin/minidump_stackwalk crash.dmp ./symbols/

# 详细输出到文件
./bin/minidump_stackwalk crash.dmp ./symbols/ > crash_report.txt

# 分析特定线程
./bin/minidump_stackwalk -t 0 crash.dmp ./symbols/
```

#### 3. minidump-2-core - 转储转换工具

**功能**: 将 minidump 格式的崩溃转储文件转换为 Linux core dump 格式

**用法**:
```bash
./bin/minidump-2-core <minidump_file>
./bin/minidump-2-core crash.dmp
```

**输出**:
- 生成标准的 Linux core 文件
- 可以用 gdb 等调试器分析
- 保持原有的内存布局和寄存器状态

**与 GDB 配合使用**:
```bash
# 转换 minidump
./bin/minidump-2-core crash.dmp

# 使用 GDB 分析
gdb /path/to/binary core
```

**示例**:
```bash
# 转换单个文件
./bin/minidump-2-core app_crash.dmp

# 批量转换
for dump in *.dmp; do
    ./bin/minidump-2-core "$dump"
done
```

#### 4. core2md - 核心转储转换工具

**功能**: 将 Linux core dump 转换为 minidump 格式（反向转换）

**用法**:
```bash
./bin/core2md <core_file> <executable> <output_minidump>
./bin/core2md core.12345 /usr/bin/myapp crash.dmp
```

**应用场景**:
- 将传统 core dump 转换为 Breakpad 格式
- 统一崩溃分析流程
- 跨平台崩溃数据交换

#### 5. dump_syms_mac - macOS 符号提取工具

**功能**: 从 macOS 的 Mach-O 文件中提取调试符号，专门针对 macOS 平台优化

**用法**:
```bash
./bin/dump_syms_mac <binary_file> > symbols.sym
./bin/dump_syms_mac /Applications/MyApp.app/Contents/MacOS/MyApp > MyApp.sym
```

**支持的文件格式**:
- Mach-O 可执行文件
- .app 包内的二进制文件
- dylib 动态库文件
- 通用二进制格式（Universal Binary）

**示例**:
```bash
# 提取应用程序符号
./bin/dump_syms_mac /Applications/MyApp.app/Contents/MacOS/MyApp > MyApp.sym

# 提取框架符号
./bin/dump_syms_mac /System/Library/Frameworks/CoreFoundation.framework/CoreFoundation > CoreFoundation.sym

# 提取动态库符号
./bin/dump_syms_mac /usr/lib/libSystem.dylib > libSystem.dylib.sym
```

#### 6. microdump_stackwalk - 微型转储堆栈遍历工具

**功能**: 分析更轻量级的微型崩溃转储（microdump），适用于资源受限环境

**用法**:
```bash
./bin/microdump_stackwalk <microdump_file> <symbols_directory>
./bin/microdump_stackwalk micro_crash.dmp ./symbols/
```

**特点**:
- 处理更小的转储文件，通常不包含完整内存映像
- 适用于移动设备和嵌入式系统
- 更快的传输和分析速度
- 占用更少的存储空间

**示例**:
```bash
# 基本分析
./bin/microdump_stackwalk micro_crash.dmp ./symbols/ > micro_crash_report.txt

# 导出为 JSON 格式
./bin/microdump_stackwalk --output-format=json micro_crash.dmp ./symbols/ > micro_crash.json
```

#### 7. minidump_dump - Minidump 内容查看工具

**功能**: 查看和导出 minidump 文件的详细内容结构

**用法**:
```bash
./bin/minidump_dump [options] <minidump_file>
./bin/minidump_dump crash.dmp
```

**选项**:
- `--hexdump`: 以十六进制格式显示数据块
- `--print-all`: 打印所有可用信息
- `--print-memory`: 打印内存数据
- `--print-threads`: 打印线程信息

**应用场景**:
- 调试崩溃分析工具问题
- 检查转储文件完整性
- 提取特定内存区域数据
- 详细查看转储内部结构

**示例**:
```bash
# 显示基本信息
./bin/minidump_dump crash.dmp

# 显示完整信息
./bin/minidump_dump --print-all crash.dmp > crash_details.txt

# 只查看线程信息
./bin/minidump_dump --print-threads crash.dmp
```

#### 8. minidump_upload - 转储上传工具

**功能**: 将 minidump 文件上传到崩溃报告服务器

**用法**:
```bash
./bin/minidump_upload [options] <minidump_file>
./bin/minidump_upload --url="https://crashes.example.com/submit" crash.dmp
```

**选项**:
- `--url=URL`: 指定上传服务器 URL
- `--product=NAME`: 设置产品名称
- `--version=VERSION`: 设置产品版本
- `--auth=TOKEN`: 身份验证令牌
- `--timeout=SECONDS`: 设置上传超时时间

**应用场景**:
- 自动化崩溃报告收集
- CI/CD 流程中的崩溃监控
- 远程技术支持和问题排查

**示例**:
```bash
# 基本上传
./bin/minidump_upload --url="https://crashes.example.com/submit" crash.dmp

# 带附加信息的上传
./bin/minidump_upload --url="https://crashes.example.com/submit" \
                     --product="MyApp" \
                     --version="1.2.3" \
                     --comment="测试版本崩溃" \
                     crash.dmp

# 批量上传
for dump in *.dmp; do
    ./bin/minidump_upload --url="https://crashes.example.com/submit" "$dump"
done
```

#### 9. pid2md - 进程ID转储工具

**功能**: 通过进程 ID 生成指定进程的 minidump 文件

**用法**:
```bash
./bin/pid2md <pid> <output_minidump>
./bin/pid2md 12345 process_dump.dmp
```

**应用场景**:
- 对挂起或性能问题进程进行调试
- 分析运行中程序的内存状态
- 在不终止进程的情况下获取诊断信息
- 系统管理和监控

**示例**:
```bash
# 为特定进程创建转储
./bin/pid2md 12345 app_running.dmp

# 结合 ps 查找并转储进程
pid=$(ps aux | grep "myapp" | grep -v grep | awk '{print $2}')
./bin/pid2md $pid myapp_running.dmp

# 转储并立即分析
./bin/pid2md 12345 app_dump.dmp && ./bin/minidump_stackwalk app_dump.dmp ./symbols/
```

#### 10. sym_upload - 符号上传工具

**功能**: 将符号文件上传到符号服务器

**用法**:
```bash
./bin/sym_upload [options] <symbol_file> <upload_url>
./bin/sym_upload --api-key=KEY symbols.sym https://symbols.example.com/
```

**选项**:
- `--api-key=KEY`: API 密钥用于身份验证
- `--version=VERSION`: 符号版本信息
- `--platform=PLATFORM`: 指定平台（如 linux, mac, windows）
- `--timeout=SECONDS`: 设置上传超时时间
- `--compress`: 压缩符号文件再上传

**应用场景**:
- 自动化构建和部署流程
- CI/CD 中的符号管理
- 分布式开发团队的符号共享
- 维护集中式符号服务器

**示例**:
```bash
# 基本上传
./bin/sym_upload app.sym https://symbols.example.com/

# 带认证的上传
./bin/sym_upload --api-key="your-secret-key" app.sym https://symbols.example.com/

# 批量上传
find ./symbols -name "*.sym" -exec ./bin/sym_upload --api-key=KEY {} https://symbols.example.com/ \;
```

## 编译环境介绍

本工具集已在本地环境成功编译。以下是编译 Breakpad 工具集所需的环境和步骤。

### 支持的操作系统

- **Linux**: Ubuntu 20.04/22.04, Debian 10/11, CentOS 7/8
- **macOS**: 10.15 (Catalina) 及更高版本
- **Windows**: 通过 WSL (Windows Subsystem for Linux) 或 MinGW 支持

### 编译依赖项

#### 基本依赖项
```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y build-essential git python3 curl cmake
sudo apt-get install -y libcurl4-openssl-dev zlib1g-dev libdistorm64-dev

# CentOS/RHEL
sudo yum install -y gcc gcc-c++ make git python3 curl cmake
sudo yum install -y libcurl-devel zlib-devel
```

#### 特定平台依赖
```bash
# macOS (使用 Homebrew)
brew install cmake ninja git python3 curl

# 交叉编译 Windows 工具 (在 Linux 上)
sudo apt-get install -y mingw-w64
```

### 获取源码

Breakpad 源码可以通过两种方式获取：

1. **直接克隆**:
```bash
git clone https://github.com/google/breakpad.git
cd breakpad
git clone https://chromium.googlesource.com/linux-syscall-support src/third_party/lss
```

2. **使用 depot_tools** (推荐):
```bash
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
export PATH=$PATH:$(pwd)/depot_tools

mkdir breakpad && cd breakpad
fetch breakpad
gclient sync
```

### 编译步骤

#### Linux 平台编译
```bash
cd breakpad
./configure
make -j$(nproc)
```

#### macOS 平台编译
```bash
cd breakpad
./configure
make -j$(sysctl -n hw.logicalcpu)
```

#### 交叉编译 Windows 工具
```bash
cd breakpad
./configure --host=i686-w64-mingw32
make -j$(nproc)
```

### 编译选项

`configure` 脚本支持以下主要选项:

- `--prefix=/path/to/install`: 指定安装目录
- `--enable-shared`: 构建共享库 (.so/.dylib) 而非静态库
- `--disable-tools`: 仅构建库，不包含工具
- `--disable-processor`: 不构建崩溃处理器库
- `--disable-sender`: 不构建崩溃报告发送器

### 验证编译结果

编译完成后，bin 目录下应该包含所有工具:

```bash
# 检查工具是否可用
./bin/dump_syms --help
./bin/minidump_stackwalk --help
```

### 常见编译问题与解决方案

1. **缺少依赖库**
   ```
   错误: 找不到 -lcurl
   解决: sudo apt-get install libcurl4-openssl-dev
   ```

2. **Python 版本问题**
   ```
   错误: 需要 Python 3
   解决: sudo apt install python3 python3-pip
   ```

3. **权限问题**
   ```
   错误: 无法写入目录
   解决: chmod +x configure && chmod -R u+w .
   ```

4. **构建日志过多**
   ```
   解决: make -j$(nproc) V=0
   ```

### 持续集成支持

Breakpad 工具可以集成到 CI/CD 流程中:

```yaml
# .github/workflows/build.yml 示例片段
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Install Dependencies
        run: sudo apt-get install -y build-essential libcurl4-openssl-dev
      - name: Configure
        run: ./configure
      - name: Build
        run: make -j$(nproc)
      - name: Test
        run: make check
```

## 使用工作流程

### 1. 符号提取阶段
```bash
# 为应用程序和所有依赖库提取符号
./bin/dump_syms /usr/bin/myapp > symbols/myapp/version/myapp.sym
./bin/dump_syms /usr/lib/libssl.so > symbols/libssl.so/version/libssl.so.sym
./bin/dump_syms /usr/lib/libcrypto.so > symbols/libcrypto.so/version/libcrypto.so.sym
```

### 2. 崩溃分析阶段
```bash
# 分析崩溃转储
./bin/minidump_stackwalk crash_20231023.dmp ./symbols/ > crash_report.txt

# 查看详细报告
cat crash_report.txt
```

### 3. 深度调试阶段
```bash
# 转换为 core dump 进行 GDB 调试
./bin/minidump-2-core crash_20231023.dmp

# 使用 GDB 深度分析
gdb /usr/bin/myapp core
```

### 4. Windows平台特有的崩溃分析流程

#### 准备阶段
```powershell
# 为所有构建创建符号目录
mkdir C:\SymbolStore

# 确保构建时启用PDB生成
# Visual Studio项目属性 -> C/C++ -> 常规 -> 调试信息格式 -> "程序数据库 (/Zi)"
# 链接器 -> 调试 -> 生成调试信息 -> "是 (/DEBUG)"
```

#### 符号提取与管理
```bash
# Windows平台提取符号（从可执行文件提取）
./bin/dump_syms path\to\app.exe > symbols\app.sym

# Windows平台提取符号（从PDB文件提取）
./bin/dump_syms path\to\app.pdb > symbols\app.sym

# 处理符号文件
$module_line = Get-Content .\symbols\app.sym | Select-Object -First 1
$parts = $module_line -split " "
$name = $parts[1]
$id = $parts[3]

# 创建符号服务器标准目录结构
New-Item -ItemType Directory -Path ".\symbols\$name\$id\" -Force
Move-Item .\symbols\app.sym ".\symbols\$name\$id\$name.sym"
```

#### DMP文件解析
```bash
# 基本崩溃报告生成
./bin/minidump_stackwalk path\to\crash.dmp .\symbols > crash_report.txt

# 详细分析包括所有线程
./bin/minidump_stackwalk --verbose path\to\crash.dmp .\symbols > crash_detailed.txt

# JSON格式输出（便于程序化处理）
./bin/minidump_stackwalk --output-format=json path\to\crash.dmp .\symbols > crash.json
```

#### 与Windows调试工具集成
```powershell
# 使用WinDbg分析minidump
$env:_NT_SYMBOL_PATH = "srv*C:\SymbolCache*https://msdl.microsoft.com/download/symbols;C:\MySymbols"
Start-Process "C:\Program Files\Windows Kits\10\Debuggers\x64\windbg.exe" -ArgumentList "-z path\to\crash.dmp"

# WinDbg内部命令
# .sympath+ C:\MySymbols  # 添加本地符号路径
# !analyze -v             # 详细分析崩溃原因
# kp                      # 显示带参数的调用堆栈
# .reload                 # 重新加载符号
```

#### 自动化处理脚本
```powershell
# PowerShell自动处理脚本示例
$dumpFiles = Get-ChildItem -Path "C:\CrashReports" -Filter "*.dmp"
$symbolPath = "C:\SymbolStore"
$outputPath = "C:\AnalysisReports"

foreach ($dump in $dumpFiles) {
    $outputFile = Join-Path $outputPath "$($dump.BaseName).txt"
    ./bin/minidump_stackwalk $dump.FullName $symbolPath > $outputFile
    Write-Host "Processed: $($dump.Name)"
}
```

### Windows版本的预编译工具路径

如果您需要在不自行编译的情况下获取Windows版本的工具（如dump_syms.exe），可以在以下位置找到预编译版本：

1. **在Chromium仓库中**：
   * `src/tools/windows/dump_syms/` - 包含Windows平台的dump_syms工具源码和预编译文件
   * 仓库地址：https://chromium.googlesource.com/breakpad/breakpad

2. **在Android预构建库中**：
   * `prebuilts/google-breakpad/windows-x86/` - 包含为Windows x86平台预编译的breakpad工具
   * 仓库地址：https://android.googlesource.com/platform/prebuilts/google-breakpad/windows-x86/

3. **在Windows SDK中**：
   * Windows SDK安装目录下的调试工具集也包含一些类似功能的工具
   * 典型路径：`C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\`

下载预编译工具后，您可以将其放置在方便访问的位置（如项目的bin目录），然后按照本文档中的说明使用这些工具进行崩溃分析和调试。

## 高级用法

### 自动化符号提取脚本
```bash
#!/bin/bash
# extract_symbols.sh

BINARY_DIR="/usr/bin"
SYMBOLS_DIR="./symbols"

for binary in "$BINARY_DIR"/*; do
    if file "$binary" | grep -q "ELF.*executable"; then
        name=$(basename "$binary")
        mkdir -p "$SYMBOLS_DIR/$name/latest"
        ./bin/dump_syms "$binary" > "$SYMBOLS_DIR/$name/latest/$name.sym"
        echo "提取符号: $name"
    fi
done
```

### 批量崩溃分析
```bash
#!/bin/bash
# analyze_crashes.sh

DUMPS_DIR="./crashes"
SYMBOLS_DIR="./symbols"
REPORTS_DIR="./reports"

mkdir -p "$REPORTS_DIR"

for dump in "$DUMPS_DIR"/*.dmp; do
    name=$(basename "$dump" .dmp)
    ./bin/minidump_stackwalk "$dump" "$SYMBOLS_DIR" > "$REPORTS_DIR/$name.txt"
    echo "分析完成: $name"
done
```

## 符号文件管理

### 符号服务器结构
```
symbols/
├── myapp/
│   ├── 1.0.0-debug-abc123/
│   │   └── myapp.sym
│   └── 1.0.1-debug-def456/
│       └── myapp.sym
├── libssl.so/
│   └── openssl-1.1.1-xyz789/
│       └── libssl.so.sym
└── libc.so.6/
    └── glibc-2.31-uvw012/
        └── libc.so.6.sym
```

### 版本标识符获取
```bash
# 获取模块的唯一标识符
objdump -h binary_file | grep "\.note\.gnu\.build-id"
readelf -n binary_file | grep "Build ID"
```

## 集成开发环境配置

### CMake 集成
```cmake
# 在发布构建中启用调试信息
set(CMAKE_BUILD_TYPE RelWithDebInfo)
set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "-O2 -g -DNDEBUG")

# 添加符号提取目标
add_custom_target(extract_symbols
    COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/tools/bin/dump_syms 
            $<TARGET_FILE:your_app> > 
            ${CMAKE_CURRENT_BINARY_DIR}/your_app.sym
    DEPENDS your_app
)
```

### 持续集成（CI）配置
```yaml
# .github/workflows/extract-symbols.yml
- name: Extract Debug Symbols
  run: |
    ./tools/platforms/unix/crash/breakpad/bin/dump_syms ./build/myapp > myapp.sym
    
- name: Upload Symbols
  uses: actions/upload-artifact@v3
  with:
    name: debug-symbols
    path: "*.sym"
```

## 故障排除

### 常见问题

1. **符号提取失败**
   ```bash
   # 检查文件格式
   file binary_file
   
   # 检查调试信息
   objdump -h binary_file | grep debug
   readelf -S binary_file | grep debug
   ```

2. **堆栈解析不完整**
   - 确保符号文件路径正确
   - 检查模块版本匹配
   - 验证符号文件完整性

3. **minidump 文件损坏**
   ```bash
   # 检查文件头
   hexdump -C crash.dmp | head -10
   
   # 验证 minidump 格式
   ./bin/minidump_stackwalk crash.dmp 2>&1 | grep -i error
   ```

### 调试选项

```bash
# 启用详细输出
export BREAKPAD_VERBOSE=1

# 调试符号解析
./bin/minidump_stackwalk -v crash.dmp ./symbols/

# 检查符号文件格式
head -20 symbols/myapp/version/myapp.sym
```

## 性能优化

### 符号文件压缩
```bash
# 压缩符号文件以节省空间
gzip symbols/myapp/version/myapp.sym

# minidump_stackwalk 支持压缩的符号文件
./bin/minidump_stackwalk crash.dmp ./symbols/
```

### 并行处理
```bash
# 并行提取符号
find /usr/lib -name "*.so" | xargs -P4 -I{} sh -c '
    name=$(basename {})
    mkdir -p symbols/$name/latest
    ./bin/dump_syms {} > symbols/$name/latest/$name.sym
'
```

## 相关资源

- [Google Breakpad 官方文档](https://chromium.googlesource.com/breakpad/breakpad/)
- [符号文件格式规范](https://chromium.googlesource.com/breakpad/breakpad/+/master/docs/symbol_files.md)
- [崩溃分析最佳实践](https://chromium.googlesource.com/breakpad/breakpad/+/master/docs/)

## Linux平台处理Windows的PDB和DMP文件

在跨平台开发环境中，经常需要在Linux系统上分析来自Windows平台的崩溃数据。以下是在Linux平台上处理Windows的PDB文件和解析DMP文件的方法。

### 安装依赖

```bash
# Ubuntu/Debian系统
sudo apt-get update
sudo apt-get install -y wine wine64 winetricks
sudo apt-get install -y cabextract

# 安装Windows调试工具依赖
winetricks msxml6 vcrun2015 corefonts
```

### 配置Windows调试工具

```bash
# 创建工作目录
mkdir -p ~/windbg_workspace

# 下载Windows调试工具SDK (可选，如果只使用breakpad工具则不需要)
# 注意：需要从Microsoft网站下载Windows SDK安装程序
# 使用wine运行安装程序，只选择"Debugging Tools for Windows"组件
wine ~/Downloads/winsdksetup.exe
```

### 处理PDB文件

#### 方法1：直接使用Breakpad提取符号

```bash
# 从Windows PDB文件提取符号信息（Linux系统上运行）
./bin/dump_syms /path/to/windows_app.pdb > windows_app.sym

# 如果PDB路径包含空格，需要正确引用
./bin/dump_syms "/path/to/Program Files/MyApp/app.pdb" > app.sym

# 创建符号目录结构
head -n1 windows_app.sym > module_info
MODULE=$(cut -d ' ' -f 2 module_info)
ID=$(cut -d ' ' -f 3 module_info)
mkdir -p symbols/$MODULE/$ID/
mv windows_app.sym symbols/$MODULE/$ID/$MODULE.sym
```

#### 方法2：使用PDB到SYM转换脚本

```bash
#!/bin/bash
# pdb_to_sym.sh - 批量处理PDB文件转换为SYM

PDB_DIR="./windows_pdbs"  # Windows PDB文件目录
SYM_DIR="./symbols"       # 输出符号目录

mkdir -p $SYM_DIR

for pdb_file in $PDB_DIR/*.pdb; do
  echo "处理: $pdb_file"
  base_name=$(basename "$pdb_file" .pdb)
  
  # 提取符号
  ./bin/dump_syms "$pdb_file" > temp.sym
  
  if [ $? -eq 0 ]; then
    # 获取模块信息
    module_line=$(head -n1 temp.sym)
    module=$(echo $module_line | cut -d ' ' -f 2)
    id=$(echo $module_line | cut -d ' ' -f 3)
    
    # 创建目录结构
    mkdir -p $SYM_DIR/$module/$id/
    mv temp.sym $SYM_DIR/$module/$id/$module.sym
    echo "符号已保存至: $SYM_DIR/$module/$id/$module.sym"
  else
    echo "处理 $pdb_file 失败"
  fi
done
```

### 解析Windows的DMP文件

```bash
# 基本解析
./bin/minidump_stackwalk /path/to/windows_crash.dmp ./symbols > crash_report.txt

# 详细输出
./bin/minidump_stackwalk --verbose /path/to/windows_crash.dmp ./symbols > crash_detailed.txt

# 指定CPU类型（如果需要）
./bin/minidump_stackwalk --cpu-info /path/to/windows_crash.dmp ./symbols > crash_with_cpu.txt
```

### 解析DMP时的常见问题与解决方案

1. **符号不匹配**
   ```
   错误: No symbol file
   解决: 确保PDB文件与崩溃的可执行文件完全匹配（版本、时间戳、编译ID）
   ```

2. **路径问题**
   ```
   错误: Cannot find symbol file
   解决: 检查符号目录结构是否符合 ./symbols/module_name/id/module_name.sym
   ```

3. **Windows与Linux路径格式**
   ```
   错误: 路径解析错误
   解决: 注意转换Windows风格路径为Linux风格路径（反斜杠改为正斜杠）
   ```

### 交叉平台工作流程示例

```bash
#!/bin/bash
# windows_crash_analysis.sh - Windows崩溃在Linux上的分析工作流

DMP_FILE=$1
PDB_DIR=$2
OUT_DIR="./crash_analysis"

if [ -z "$DMP_FILE" ] || [ -z "$PDB_DIR" ]; then
  echo "用法: $0 crash.dmp pdb目录"
  exit 1
fi

# 创建输出目录
mkdir -p $OUT_DIR
mkdir -p ./symbols

# 处理所有PDB文件
echo "正在处理PDB文件..."
for pdb_file in $PDB_DIR/*.pdb; do
  base_name=$(basename "$pdb_file" .pdb)
  echo "  提取符号: $base_name"
  
  ./bin/dump_syms "$pdb_file" > temp.sym
  
  if [ $? -eq 0 ]; then
    module_line=$(head -n1 temp.sym)
    module=$(echo $module_line | cut -d ' ' -f 2)
    id=$(echo $module_line | cut -d ' ' -f 3)
    
    mkdir -p ./symbols/$module/$id/
    mv temp.sym ./symbols/$module/$id/$module.sym
  else
    echo "  警告: 无法处理 $pdb_file"
  fi
done

# 分析DMP文件
echo "正在分析崩溃转储..."
./bin/minidump_stackwalk $DMP_FILE ./symbols > $OUT_DIR/crash_report.txt
./bin/minidump_dump $DMP_FILE > $OUT_DIR/crash_dump.txt

# 输出模块列表
echo "正在提取加载模块信息..."
grep "Loaded modules:" -A 100 $OUT_DIR/crash_report.txt | grep -E "^[0-9]" > $OUT_DIR/loaded_modules.txt

echo "分析完成。报告保存在: $OUT_DIR/"
```

### 在CI/CD环境中的自动化分析

```yaml
# .github/workflows/analyze-windows-crashes.yml 示例
name: Analyze Windows Crashes
on: [workflow_dispatch]

jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Install Dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y build-essential libcurl4-openssl-dev
      
      - name: Build Breakpad Tools
        run: |
          ./configure
          make -j$(nproc)
      
      - name: Download PDBs and DMPs
        uses: actions/download-artifact@v3
        with:
          name: windows-crash-data
          path: ./crash-data
      
      - name: Process PDBs
        run: |
          mkdir -p ./symbols
          for pdb in ./crash-data/*.pdb; do
            ./bin/dump_syms "$pdb" > temp.sym
            if [ $? -eq 0 ]; then
              module_line=$(head -n1 temp.sym)
              module=$(echo $module_line | cut -d ' ' -f 2)
              id=$(echo $module_line | cut -d ' ' -f 3)
              mkdir -p ./symbols/$module/$id/
              mv temp.sym ./symbols/$module/$id/$module.sym
            fi
          done
      
      - name: Analyze Crashes
        run: |
          mkdir -p ./reports
          for dmp in ./crash-data/*.dmp; do
            base=$(basename "$dmp" .dmp)
            ./bin/minidump_stackwalk "$dmp" ./symbols > ./reports/$base.txt
          done
      
      - name: Upload Analysis Reports
        uses: actions/upload-artifact@v3
        with:
          name: crash-analysis-reports
          path: ./reports
```

## 技术支持

如果遇到问题，请检查：
1. 工具版本兼容性
2. 符号文件格式正确性
3. 系统依赖库完整性
4. 文件权限设置

更多详细信息请参考源码文档和示例。
