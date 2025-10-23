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

## 技术支持

如果遇到问题，请检查：
1. 工具版本兼容性
2. 符号文件格式正确性
3. 系统依赖库完整性
4. 文件权限设置

更多详细信息请参考源码文档和示例。
