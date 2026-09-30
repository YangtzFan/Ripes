# Linux 本机构建实录

本文记录 2026-09-30 在 Ubuntu 22.04.5 x86_64 上构建 Ripes 的实际过程。命令均在仓库根目录执行，即能看到 `CMakeLists.txt`、`src` 和 `docs_CN` 的目录。

## 1. 实测结果

- Qt 6.9.3 GCC 64 位已安装到用户目录：`~/.local/share/ripes-qt/6.9.3/gcc_64`。
- Release 主程序构建成功：`build-linux/Ripes`。
- 6 个 Qt Test 程序全部成功编译并通过：`tst_assembler`、`tst_expreval`、`tst_riscv`、`tst_cosimulate`、`tst_reverse`、`tst_stall`。
- CLI 使用 `RV32_5S`、`M,C` 扩展成功执行 `examples/assembly/factorial.s`，退出码为 0，并生成 `build-linux/cli-smoke.json`。
- Qt offscreen 模式下 GUI 进程成功初始化并持续运行 10 秒。
- 本机直接使用 X11 的 `xcb` 平台时提示缺少 `libxcb-cursor0`；由于当前账号没有免密码 `sudo` 权限，未安装系统包。这个限制不影响编译、CLI 或 offscreen 启动验证。

## 2. 工具和环境

本次实测环境如下：

| 项目 | 版本或路径 |
| --- | --- |
| 操作系统 | Ubuntu 22.04.5 LTS |
| 架构 | x86_64 |
| 编译器 | GCC 11.4.0 |
| CMake | 3.31.10（用户级安装） |
| Ninja | 1.10.1 |
| Qt | 6.9.3，GCC 64 位，包含 Charts |
| Python | 3.10.12 |
| 构建标准 | C++20 |

顶层 `CMakeLists.txt` 当前使用 `find_package(Qt6 6.8 COMPONENTS Core Widgets Svg Charts REQUIRED)`，因此 Qt 6.8 或更高版本是实际要求。Qt 5 和不含 Charts 的 Qt 6 安装不能满足配置条件。

Ubuntu 上通常还需要 Qt 的 OpenGL/XCB 运行依赖。若拥有管理员权限，可按 CI 使用的依赖安装：

```bash
sudo apt-get update
sudo apt-get install -y build-essential cmake ninja-build git \
  libgl1-mesa-dev libegl1-mesa-dev libxkbcommon-x11-0 libpulse-dev \
  libxcb-icccm4 libxcb-image0 libxcb-keysyms1 libxcb-render-util0 \
  libxcb-xinerama0 libxcb-composite0 libxcb-cursor0
```

`libxcb-cursor0` 主要影响 Qt 的 X11 `xcb` 平台插件，不影响 C++ 编译和链接。

## 3. 安装 Qt 到用户目录

如果系统没有符合要求的 Qt，可以使用 Python 工具 `aqtinstall` 安装。下面不修改系统 Qt，也不需要 `sudo`。当前终端的代理若不可用，应临时清除代理变量；不要把清除代理写入全局 shell 配置：

```bash
python3 -m virtualenv --python /usr/bin/python3 \
  ~/.local/share/ripes-build-tools

env -u http_proxy -u https_proxy -u HTTP_PROXY -u HTTPS_PROXY \
  ~/.local/share/ripes-build-tools/bin/python -m pip install aqtinstall

env -u http_proxy -u https_proxy -u HTTP_PROXY -u HTTPS_PROXY \
  ~/.local/share/ripes-build-tools/bin/python -m aqt install-qt \
  linux desktop 6.9.3 linux_gcc_64 -m qtcharts \
  -O ~/.local/share/ripes-qt
```

安装完成后检查：

```bash
QT_ROOT="$HOME/.local/share/ripes-qt/6.9.3/gcc_64"
test -f "$QT_ROOT/lib/cmake/Qt6/Qt6Config.cmake"
test -f "$QT_ROOT/lib/cmake/Qt6Charts/Qt6ChartsConfig.cmake"
```

如果 `python3 -m venv` 报告缺少 `ensurepip`，可以使用系统已有的 `virtualenv` 模块：

```bash
python3 -m virtualenv --python /usr/bin/python3 \
  ~/.local/share/ripes-build-tools
```

## 4. 准备源码和依赖

首次获取仓库时建议使用递归克隆：

```bash
git clone --recursive <仓库地址>
cd Ripes
```

已有仓库执行：

```bash
git submodule update --init --recursive
git submodule status
```

本次构建要求 VSRTL 为主仓库记录的固定提交 `8497dd14fe80e57efcff4c424a9a3b6363d93eb7`。如果 GitHub 的 Git smart protocol 在当前网络中长时间卡住，可以下载固定提交归档再解压到 `external/VSRTL`：

```bash
rm -rf external/VSRTL
mkdir -p /tmp/ripes-vsrtl
env -u http_proxy -u https_proxy -u HTTP_PROXY -u HTTPS_PROXY \
  curl -L --fail \
  https://codeload.github.com/mortbopet/VSRTL/tar.gz/8497dd14fe80e57efcff4c424a9a3b6363d93eb7 \
  -o /tmp/ripes-vsrtl.tar.gz
tar -xzf /tmp/ripes-vsrtl.tar.gz -C /tmp/ripes-vsrtl --strip-components=1
mv /tmp/ripes-vsrtl external/VSRTL
```

Ripes 和 VSRTL 的 CMake 配置还会通过 FetchContent 获取 cereal、Signals、magic_enum、libelfin 和 ELFIO。如果 Git 下载这些仓库超时，可使用对应提交的 `codeload.github.com` 归档，解压后通过 CMake 的 `FETCHCONTENT_SOURCE_DIR_*` 变量指定本地目录。libelfin 还需要它的 `cpp-mmaplib` 子模块；缺少该目录会出现：

```text
fatal error: external/cpp-mmaplib/mmaplib.h: No such file or directory
```

## 5. 配置和编译

使用独立的 `build-linux` 构建目录：

```bash
QT_ROOT="$HOME/.local/share/ripes-qt/6.9.3/gcc_64"
CMAKE="$HOME/.local/share/ripes-build-tools/bin/cmake"

"$CMAKE" -S . -B build-linux -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_PREFIX_PATH="$QT_ROOT" \
  -DRIPES_BUILD_TESTS=ON \
  -DRIPES_WITH_QPROCESS=ON

"$CMAKE" --build build-linux --parallel 16
```

配置成功的标志是最后出现：

```text
-- Configuring done
-- Generating done
-- Build files have been written to: .../build-linux
```

编译成功后主要产物为：

```text
build-linux/Ripes
build-linux/test/tst_assembler
build-linux/test/tst_expreval
build-linux/test/tst_riscv
build-linux/test/tst_cosimulate
build-linux/test/tst_reverse
build-linux/test/tst_stall
```

本次配置出现的 Vulkan 头文件缺失和第三方 CMake deprecation/dev warning 没有阻止构建：

```text
Could NOT find WrapVulkanHeaders
```

## 6. 测试和运行验证

CTest 当前没有被顶层 CMake 注册，因此直接执行 `ctest` 可能显示 `No tests were found`。应直接运行已经编译出的 Qt Test 程序：

```bash
QT_ROOT="$HOME/.local/share/ripes-qt/6.9.3/gcc_64"
export PATH="$QT_ROOT/bin:$PATH"
export LD_LIBRARY_PATH="$QT_ROOT/lib${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
export QT_QPA_PLATFORM=offscreen

for test in tst_assembler tst_expreval tst_riscv \
  tst_cosimulate tst_reverse tst_stall; do
  "build-linux/test/$test" \
    -o "build-linux/test/$test.txt,txt"
  echo "$test exit=$?"
done
```

本次 6 个测试的退出码全部为 0。

CLI 冒烟验证：

```bash
export QT_QPA_PLATFORM=offscreen
build-linux/Ripes --mode cli \
  --src examples/assembly/factorial.s \
  -t asm --proc RV32_5S --isaexts M,C \
  --cycles --iret --cpi --ipc --runinfo --json \
  --output build-linux/cli-smoke.json
```

本次输出包含 `Program exited with code: 0`，报告中的实测值为 125 条退休指令、189 个周期、CPI 1.512、IPC 约 0.6614。

GUI 启动：

```bash
export QT_QPA_PLATFORM=offscreen
timeout 10s build-linux/Ripes
```

返回码为 124 表示 `timeout` 主动结束了仍在运行的 GUI 进程，本次即为此结果，说明程序至少完成了 Qt 应用初始化并没有立即退出。若已安装 X11 依赖，也可以取消 `QT_QPA_PLATFORM=offscreen` 后直接运行 `build-linux/Ripes`。

## 7. 日常增量构建

Qt 和依赖已经配置好后，源码修改后的日常流程只需：

```bash
"$HOME/.local/share/ripes-build-tools/bin/cmake" \
  --build build-linux --parallel 16
```

切换 Qt 路径、修改 CMake 选项或更换生成器时，建议使用新的构建目录，或先删除并重新生成 `build-linux`。不要删除 `src`、`external` 或其他源码目录来清理构建缓存。

常见问题：

| 现象 | 处理方式 |
| --- | --- |
| `Could NOT find Qt6` | 设置 `-DCMAKE_PREFIX_PATH` 为 Qt 的 `gcc_64` 目录，并确认 `Qt6Config.cmake` 存在 |
| `Qt6ChartsConfig.cmake` 不存在 | 重新安装 Qt 的 `qtcharts` 模块 |
| `mmaplib.h` 不存在 | 初始化 libelfin 的 `external/cpp-mmaplib`，或按本文下载其归档 |
| X11 提示缺少 `libxcb-cursor0` | 安装 `libxcb-cursor0`，无管理员权限时使用 `QT_QPA_PLATFORM=offscreen` |
| GitHub 克隆超时 | 临时清除代理变量，或使用固定提交的 `codeload.github.com` 归档 |
