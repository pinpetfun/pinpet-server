# Windows 交叉编译指南

本文档说明如何在 Linux 环境下交叉编译 spin-server 项目到 Windows 平台。

## 前置要求

- Linux 开发环境（已安装 Rust）
- 足够的磁盘空间（约 2GB）
- 稳定的网络连接

## 1. 安装交叉编译工具链

### 1.1 添加 Windows 目标平台

```bash
# 64位 Windows
rustup target add x86_64-pc-windows-gnu

# 或者使用 MSVC 工具链（推荐，但需要更多配置）
# rustup target add x86_64-pc-windows-msvc
```

### 1.2 安装 MinGW-w64 交叉编译器

```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y mingw-w64

# CentOS/RHEL
sudo yum install -y mingw64-gcc mingw64-gcc-c++
```

### 1.3 配置 Cargo 交叉编译

创建或编辑 `~/.cargo/config.toml`：

```toml
[target.x86_64-pc-windows-gnu]
linker = "x86_64-w64-mingw32-gcc"
ar = "x86_64-w64-mingw32-ar"
```

## 2. 处理 RocksDB 依赖

RocksDB 是本项目最具挑战性的依赖，因为它是 C++ 库。有以下几种方案：

### 方案 A：使用预编译的 RocksDB（推荐）

1. 下载 Windows 版本的 RocksDB 库
2. 设置环境变量指向库文件：

```bash
export ROCKSDB_LIB_DIR=/path/to/rocksdb/lib
export ROCKSDB_INCLUDE_DIR=/path/to/rocksdb/include
```

### 方案 B：禁用 RocksDB 功能（用于测试）

如果只是测试编译，可以考虑暂时使用内存存储替代：

1. 在 `Cargo.toml` 中将 RocksDB 改为可选依赖
2. 编译时使用 `--no-default-features`

### 方案 C：使用交叉编译 Docker 容器（最简单）

使用 `cross` 工具自动处理交叉编译环境：

```bash
# 安装 cross
cargo install cross

# 使用 cross 编译
cross build --release --target x86_64-pc-windows-gnu
```

## 3. 编译命令

### 3.1 使用 cross（推荐）

```bash
# 开发模式
cross build --target x86_64-pc-windows-gnu

# 发布模式
cross build --release --target x86_64-pc-windows-gnu
```

编译产物位于：`target/x86_64-pc-windows-gnu/release/spin-server.exe`

### 3.2 使用原生 cargo

```bash
# 设置环境变量
export CC_x86_64_pc_windows_gnu=x86_64-w64-mingw32-gcc
export CXX_x86_64_pc_windows_gnu=x86_64-w64-mingw32-g++
export AR_x86_64_pc_windows_gnu=x86_64-w64-mingw32-ar

# 编译
cargo build --release --target x86_64-pc-windows-gnu
```

## 4. 打包部署

### 4.1 收集必要文件

```bash
# 创建部署目录
mkdir -p windows-release

# 复制可执行文件
cp target/x86_64-pc-windows-gnu/release/spin-server.exe windows-release/

# 复制配置文件
cp -r config windows-release/

# 复制 DLL 依赖（如果使用 MinGW）
# 通常需要以下 DLL：
# - libgcc_s_seh-1.dll
# - libstdc++-6.dll
# - libwinpthread-1.dll
# 这些文件通常在 /usr/x86_64-w64-mingw32/lib/ 目录
cp /usr/x86_64-w64-mingw32/lib/libgcc_s_seh-1.dll windows-release/ 2>/dev/null || true
cp /usr/x86_64-w64-mingw32/lib/libstdc++-6.dll windows-release/ 2>/dev/null || true
cp /usr/x86_64-w64-mingw32/lib/libwinpthread-1.dll windows-release/ 2>/dev/null || true
```

### 4.2 创建打包脚本

可以创建 `build-windows.sh` 脚本自动化这个过程：

```bash
#!/bin/bash
set -e

echo "Building for Windows..."
cross build --release --target x86_64-pc-windows-gnu

echo "Packaging..."
rm -rf windows-release
mkdir -p windows-release
cp target/x86_64-pc-windows-gnu/release/spin-server.exe windows-release/
cp -r config windows-release/

echo "Creating archive..."
cd windows-release
zip -r ../spin-server-windows.zip .
cd ..

echo "Done! Package: spin-server-windows.zip"
```

## 5. 常见问题

### 5.1 RocksDB 编译失败

**错误信息**：`error: failed to run custom build command for rocksdb`

**解决方案**：
- 确保安装了 MinGW-w64 的 C++ 编译器
- 使用 `cross` 工具，它会自动处理这些依赖
- 考虑使用静态链接：设置 `ROCKSDB_STATIC=1`

### 5.2 OpenSSL 相关错误

**错误信息**：`could not find native static library 'ssl'`

**解决方案**：
```bash
# 安装 OpenSSL for MinGW
sudo apt-get install libssl-dev:i386

# 或使用 vendored 特性
# 在 Cargo.toml 中修改 reqwest 和 tokio-tungstenite：
# reqwest = { version = "0.11", features = ["json", "native-tls-vendored"] }
```

### 5.3 运行时缺少 DLL

**错误信息**：Windows 上运行时提示缺少 `.dll` 文件

**解决方案**：
- 将必要的 MinGW DLL 文件与 exe 放在同一目录
- 或使用静态链接：添加到 `.cargo/config.toml`：
  ```toml
  [target.x86_64-pc-windows-gnu]
  rustflags = ["-C", "target-feature=+crt-static"]
  ```

### 5.4 文件路径问题

**问题**：Windows 和 Linux 的路径分隔符不同

**解决方案**：
- 确保代码中使用 `std::path::PathBuf` 而不是硬编码路径
- 配置文件中的路径使用相对路径

## 6. 性能优化

### 6.1 减小可执行文件大小

在 `Cargo.toml` 中添加：

```toml
[profile.release]
opt-level = "z"     # 优化大小
lto = true          # 启用链接时优化
codegen-units = 1   # 更好的优化
strip = true        # 移除符号信息
```

### 6.2 启用特定 CPU 特性

```bash
RUSTFLAGS="-C target-cpu=native" cross build --release --target x86_64-pc-windows-gnu
```

## 7. 测试建议

1. **本地测试**：如果有 Windows 环境，先在虚拟机中测试
2. **Wine 测试**：在 Linux 上使用 Wine 进行初步测试
   ```bash
   sudo apt-get install wine64
   wine target/x86_64-pc-windows-gnu/release/spin-server.exe --help
   ```
3. **依赖检查**：使用 `ldd` 检查依赖（在 Linux 上）或 Dependency Walker（在 Windows 上）

## 8. CI/CD 集成

如果使用 GitHub Actions，可以添加 Windows 交叉编译任务：

```yaml
name: Build Windows
on: [push]

jobs:
  build-windows:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions-rs/toolchain@v1
        with:
          toolchain: stable
          target: x86_64-pc-windows-gnu
      - uses: actions-rs/cargo@v1
        with:
          use-cross: true
          command: build
          args: --release --target x86_64-pc-windows-gnu
      - uses: actions/upload-artifact@v2
        with:
          name: spin-server-windows
          path: target/x86_64-pc-windows-gnu/release/spin-server.exe
```

## 总结

推荐使用 `cross` 工具进行交叉编译，这是最简单且最可靠的方法。对于生产环境，建议：

1. 使用 `cross` 进行编译
2. 启用静态链接减少 DLL 依赖
3. 在真实 Windows 环境中进行充分测试
4. 使用 CI/CD 自动化构建流程

如有问题，请参考：
- [cross 项目文档](https://github.com/cross-rs/cross)
- [Rust 交叉编译指南](https://rust-lang.github.io/rustup/cross-compilation.html)
