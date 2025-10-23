# Google Breakpad 崩溃分析工具包

Google Breakpad 是一个用于生成和分析应用程序崩溃转储文件的跨平台库。本目录包含了 Breakpad 的完整工具集，用于符号提取、崩溃分析和调试。

## 目录结构

```
breakpad/
├── README.md          # 本文档
├── bin/               # 可执行工具目录
│   ├── dump_syms      # 符号提取工具
│   ├── minidump_stackwalk     # 堆栈遍历工具
│   ├── minidump-2-core        # 转储转换工具
│   └── core2md        # 核心转储转换工具（可选）
├── breakpad/          # Breakpad 源码目录
├── depot_tools/       # Google depot_tools 工具集
└── src/               # 源码符号链接（gclient 模式）
```

## 工具介绍

### bin/ 目录下的核心工具

#### 1. dump_syms - 符号提取工具

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

#### 4. core2md - 核心转储转换工具（可选）

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
