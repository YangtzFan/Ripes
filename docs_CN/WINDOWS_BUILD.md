# Windows 本机构建实录

本文记录 2026-09-21 在 Windows 11 上从环境检查、安装 Qt 到构建、测试、部署和运行 Ripes 的完整过程。文中的命令已经在当前工作副本实际执行。路径不同的机器只需调整 `$QtRoot`、`$CMake` 和 Visual Studio 安装路径。命令都应在 Ripes 仓库根目录的 PowerShell 中执行；根目录就是能够看到 `CMakeLists.txt`、`src` 和 `docs_CN` 的目录。

## 1. 最终结果

- Release 主程序构建成功：`build-windows/Release/Ripes.exe`，大小约 6.4 MiB。
- `windeployqt` 已将 Qt DLL 和 `platforms/qwindows.dll` 等插件放入 `build-windows/Release`。
- CLI 使用 `RV32_5S` 成功执行 `examples/assembly/factorial.s`，退出码为 0。
- GUI 进程成功完成加载，持续运行 5 秒后由验证脚本关闭。
- 6 个 Qt Test 程序全部成功编译；3 个全部通过，另外 3 个存在当前代码或测试环境相关失败，详见“测试结果”。
- Qt 6.9.3 安装在 `D:\Qt\6.9.3\msvc2022_64`，该版本目录约占 2.05 GiB。
- 构建目录 `build-windows` 约占 0.28 GiB，已被仓库现有 `build*` 忽略规则排除，不会进入 Git。

本机原有 `D:\Python\anaconda3\Library` 中的 Qt 5.15.2 没有删除。它属于 Anaconda 环境的一部分，直接删除会破坏依赖 Qt 的 Python 包，而且 Qt 5 也不能替代本工程要求的 Qt 6.8+。新版本采用独立目录安装，二者不会互相覆盖。

当前这台机器已经完成 Qt 安装、CMake 配置和 Release 构建。日常修改代码后直接执行第 12 节的“后续增量构建”即可；只有 Qt 目录被删除、换电脑或重装开发环境时，才需要重新执行安装章节。第一次操作时建议一次只执行一个代码块，看到该步骤的成功标志后再继续，避免多条错误叠在一起。

## 2. 构建工具与实测版本

第一次接触 C++ 工程时，最容易混淆的是这些工具的分工：

- **源代码**是仓库里的 `.cpp` 和 `.h` 文件，人可以阅读，但 Windows 不能直接运行。
- **MSVC** 是 Visual Studio 附带的 C/C++ 编译器和链接器。编译器把每个源文件变成机器代码，链接器再把机器代码和库组合成 `.exe`。安装 Visual Studio IDE 的主要目的之一，就是取得这套工具和 Windows SDK；构建时不要求一直打开 IDE。
- **Qt** 是 Ripes 使用的 C++ 库。窗口、按钮、图表、SVG、事件循环和 Qt Test 都来自 Qt。Qt 不是编译器，同一套 Qt 二进制文件还必须与编译器类型匹配，所以这里安装 `msvc2022_64`，不能换成 MinGW 套件。
- **CMake** 是构建配置工具。它读取 `CMakeLists.txt`，查找 Qt 和依赖，然后生成 Visual Studio 能执行的工程与编译规则。它不负责编辑代码，也不取代 MSVC。
- **Visual Studio 生成器/MSBuild** 是本次实际执行构建规则的后端。CMake 的 `-G 'Visual Studio 17 2022'` 选择它。Ninja 是另一种构建后端，本次没有使用它。
- **Git** 管理源码版本，也负责下载子模块。**PowerShell** 只是输入并组合这些命令的终端。

整个过程可以理解为：

```text
Git 准备完整源码
    ↓
CMake 读取 CMakeLists.txt，并找到 Qt
    ↓
Visual Studio/MSVC 编译、链接
    ↓
得到 Ripes.exe
    ↓
windeployqt 把运行所需的 Qt DLL 放到 exe 附近
```

“库”是别人已经写好的可复用代码；“SDK”是开发某个平台所需的一组头文件、库和工具；“DLL”是程序运行时动态加载的库文件。即使 `Ripes.exe` 已经编译成功，缺少所需 DLL 仍然无法在 Windows 上启动。

| 工具 | 本机版本或路径 | 用途 |
| --- | --- | --- |
| PowerShell | 7.6.6 | 执行本文命令 |
| Git | 2.54.0.windows.1 | 获取仓库和子模块 |
| Visual Studio | Community 2022 17.10.4 | 提供 MSVC 和 Windows SDK |
| MSVC | 19.40.33812，x64 | C++20 编译器 |
| Windows SDK | 10.0.22621.0 | Windows API 和链接库 |
| CMake | 3.28.3-msvc11 | Visual Studio 自带版本 |
| Qt | 6.9.3，MSVC 2022 64-bit | GUI、Charts、SVG 和 Qt Test |

Visual Studio 自带的 CMake 和 Ninja 未加入普通 PowerShell 的 `PATH`，但文件已经存在，无需重复安装：

```text
D:\Microsoft Visual Studio\2022\Community\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin\cmake.exe
D:\Microsoft Visual Studio\2022\Community\Common7\IDE\CommonExtensions\Microsoft\CMake\Ninja\ninja.exe
D:\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvars64.bat
```

`PATH` 是 Windows 用来搜索命令的目录列表。一个程序“不在 PATH 中”只表示输入短名称（例如 `cmake`）时找不到它，并不表示没有安装；使用完整路径仍可运行。本文把完整路径保存到 PowerShell 变量中，既明确又不会永久修改系统配置。

## 3. 环境检查

在仓库根目录打开 PowerShell，先确认工具和子模块状态：

```powershell
$VSRoot = 'D:\Microsoft Visual Studio\2022\Community'
$CMake = "$VSRoot\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin\cmake.exe"
$CTest = "$VSRoot\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin\ctest.exe"
$QtRoot = 'D:\Qt\6.9.3\msvc2022_64'

git --version
& $CMake --version
Test-Path "$VSRoot\VC\Auxiliary\Build\vcvars64.bat"
Test-Path "$QtRoot\lib\cmake\Qt6\Qt6Config.cmake"
Test-Path "$QtRoot\lib\cmake\Qt6Charts\Qt6ChartsConfig.cmake"
git submodule status
```

上面几种 PowerShell 写法需要先认识：

- `$VSRoot = '...'` 创建变量，后面可用 `$VSRoot` 代替长路径。这些变量只在当前 PowerShell 窗口中有效，关闭窗口后不会保留。
- 单引号字符串基本按原样使用；双引号字符串会展开其中的 `$变量`，所以 `$CMake` 能由 `$VSRoot` 拼出完整路径。
- `& $CMake --version` 中的 `&` 是“调用运算符”，表示运行变量中保存的程序路径。路径含空格时尤其需要这种写法。
- `Test-Path` 只检查文件或目录是否存在，不会修改任何内容。
- 命令中的 `#` 开头部分是注释，不会执行；文档中的提示符或输出内容也不要当作命令输入。
- `<目录>`、`<仓库地址>` 这类尖括号内容表示需要替换的占位符，不要连同尖括号原样输入；本文给出的 `D:\...` 实际路径则可以直接使用。

Qt 检查项应返回 `True`，VSRTL 子模块状态前不应有 `-`。如果 `cmake` 命令在普通终端中找不到，直接使用上面的绝对路径即可，也可以改用“x64 Native Tools Command Prompt for VS 2022”。PowerShell 中 `Get-Location` 可以查看当前目录，`Set-Location <目录>` 用于切换目录，`Get-ChildItem` 用于列出目录内容。

## 4. 安装 Qt 6.9.3 到 D 盘

当前 `CMakeLists.txt` 使用：

```cmake
find_package(Qt6 6.8 COMPONENTS Core Widgets Svg Charts REQUIRED)
```

因此 Qt 5 或 Qt 6.5 均不满足当前源码要求。此次使用 `aqtinstall` 从 Qt 官方发布仓库安装 MSVC 2022 64 位套件和 `qtcharts`；`qtsvg` 会作为依赖自动安装。

这里的 `Core`、`Widgets`、`Svg`、`Charts` 是 Qt 的模块：基础能力通常在 Core，传统桌面界面在 Widgets，矢量图标在 Svg，性能曲线等图表在 Charts。CMake 会逐项检查这些模块，只缺一个也会在配置阶段停止。

下面使用 Python 只是为了运行 Qt 下载工具，Ripes 本身仍是 C++ 工程。先运行 `python --version`，当前机器应显示 Python 3.12.7；若提示找不到命令，就不能继续运行 `venv`，需要先安装 Python 3 或改用 Qt 官方图形化安装器。

```powershell
python -m venv D:\Qt\aqt-venv
D:\Qt\aqt-venv\Scripts\python.exe -m pip install --upgrade pip aqtinstall

# 可选：查询仓库中实际存在的版本和模块
D:\Qt\aqt-venv\Scripts\python.exe -m aqt list-qt windows desktop --spec 6.9
D:\Qt\aqt-venv\Scripts\python.exe -m aqt list-qt windows desktop `
  --modules 6.9.3 win64_msvc2022_64

# 安装基础套件、Qt SVG 和 Qt Charts
D:\Qt\aqt-venv\Scripts\python.exe -m aqt install-qt `
  windows desktop 6.9.3 win64_msvc2022_64 `
  -m qtcharts -O D:\Qt
```

`python -m venv` 创建的是隔离的 Python 工具环境，只用于安装和运行 `aqtinstall`，它不是 Qt 本体。PowerShell 行末的反引号 `` ` `` 表示“下一行仍属于同一条命令”；反引号后不能再有空格。觉得多行命令难以输入时，可以把同一段合并成一行并删除反引号。`-O D:\Qt` 指定安装根目录，最终 Qt 位于其下的版本和工具链子目录。

验证安装：

```powershell
& D:\Qt\6.9.3\msvc2022_64\bin\qmake.exe -v
Test-Path D:\Qt\6.9.3\msvc2022_64\lib\cmake\Qt6\Qt6Config.cmake
Test-Path D:\Qt\6.9.3\msvc2022_64\lib\cmake\Qt6Charts\Qt6ChartsConfig.cmake
Test-Path D:\Qt\6.9.3\msvc2022_64\bin\windeployqt.exe
```

这里的 `qmake -v` 仅用于显示已安装的 Qt 版本。本工程采用 CMake，不使用 qmake 生成构建规则；后续仍应执行第 6 节的 CMake 命令。

实测输出为 Qt 6.9.3，三个路径均存在。安装完成后，`D:\Qt\aqt-venv` 只是下载工具环境，已在本次构建结束后删除，释放约 27.4 MiB；删除它不会影响 `D:\Qt\6.9.3` 中已安装的 Qt。以后升级 Qt 时重新执行创建虚拟环境和安装 `aqtinstall` 的命令即可。

## 5. 通过 SSH 获取 GitHub 依赖

当前网络环境访问 `https://github.com` 会超时，但 GitHub SSH 可正常认证：

```powershell
ssh -o BatchMode=yes -o StrictHostKeyChecking=accept-new -T git@github.com
```

GitHub 正常会返回“successfully authenticated, but GitHub does not provide shell access”，并以状态码 1 结束；这是 SSH 认证成功，不是错误。

SSH 是一种加密连接协议。GitHub 可以用本机已有的 SSH 密钥识别账号，从而拉取仓库，不需要在命令中填写密码。这里仅用它绕过当前网络对 GitHub HTTPS 的连接问题，不会改变 Ripes 源码。

让当前仓库的 GitHub 依赖使用 SSH：

Git 子模块可以理解为“主仓库中嵌入的另一个 Git 仓库”。Ripes 只记录 VSRTL 应使用的固定提交，不会把 VSRTL 的全部文件重复保存进主仓库，因此首次使用必须额外下载。

```powershell
# 仅写入当前仓库的 .git/config，不修改全局 Git 配置
git config --local 'url.git@github.com:.insteadOf' 'https://github.com/'
git config --local submodule.external/VSRTL.url `
  'git@github.com:mortbopet/VSRTL.git'

git submodule update --init --recursive
git submodule status
```

本次检出的 VSRTL 提交为：

```text
8497dd14fe80e57efcff4c424a9a3b6363d93eb7 external/VSRTL
```

URL 重写也让 CMake FetchContent 使用 SSH 获取 VSRTL 的 cereal 依赖、ELFIO 和 libelfin。FetchContent 是 CMake 在首次配置时自动下载指定版本依赖的机制。若所在网络能直接访问 GitHub HTTPS，可以省略两条 `git config`，直接执行 `git submodule update --init --recursive`。

命令成功后，`external/VSRTL` 中应出现源码文件。子模块状态开头的空格表示已经检出正确提交；`-` 表示尚未初始化，`+` 表示当前提交与主仓库记录的不一致。遇到后两种情况时不要直接开始 CMake 配置。

## 6. 配置 CMake

使用 Visual Studio 多配置生成器，以便同一个 `build-windows` 同时支持 Release 和 Debug。“生成器”决定 CMake 要生成哪种构建文件；“多配置”表示无需重新配置，就能从同一构建目录选择 Release 或 Debug：

```powershell
$VSRoot = 'D:\Microsoft Visual Studio\2022\Community'
$CMake = "$VSRoot\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin\cmake.exe"
$QtRoot = 'D:\Qt\6.9.3\msvc2022_64'

& $CMake -S . -B build-windows `
  -G 'Visual Studio 17 2022' `
  -A x64 `
  -DCMAKE_PREFIX_PATH="$QtRoot" `
  -DRIPES_BUILD_TESTS=ON `
  -DRIPES_WITH_QPROCESS=ON
```

各参数的含义如下：

- `-S .`：源码目录是当前目录；`.` 在 PowerShell 和多数命令行工具中表示“当前目录”。
- `-B build-windows`：把生成文件、中间文件和下载的依赖放在 `build-windows`，避免污染源码目录。这个目录不是最终安装目录，里面的文件由工具生成，不应手工修改。
- `-G 'Visual Studio 17 2022'`：选择 Visual Studio 2022 生成器。
- `-A x64`：生成 64 位程序，并与 `msvc2022_64` 版 Qt 保持一致。
- `-D名称=值`：设置一个 CMake 配置变量。`ON`/`OFF` 相当于打开/关闭功能；`CMAKE_PREFIX_PATH` 指出 Qt 的位置，`RIPES_BUILD_TESTS=ON` 表示同时生成测试程序，`RIPES_WITH_QPROCESS=ON` 允许 Ripes 使用 Qt 的 QProcess 启动外部程序。

这一步称为“配置”，不会得到最终的 `Ripes.exe`。成功标志是最后出现 `Configuring done`、`Generating done` 和 `Build files have been written to...`。如果失败，应从输出中向上寻找第一条包含 `Error` 或 `Could NOT find` 的关键错误；最后一行常常只是前面错误的结果。

首次配置实测耗时 633.2 秒，主要用于通过 FetchContent 下载和配置依赖。最终识别结果为：

```text
MSVC 19.40.33812.0
Windows SDK 10.0.22621.0
Qt 6.9.3
Configuring done
Generating done
Build files have been written to: .../build-windows
```

配置时出现的以下信息没有阻止构建：

- `Could NOT find WrapVulkanHeaders`：本工程的 Widgets/SVG/Charts 路径不需要 Vulkan。
- libelfin 的 `CMP0148`、`CMP0071`：第三方项目的 CMake 开发者警告。

不要为消除这两类提示额外安装 Vulkan SDK 或修改本机 Python。

## 7. 编译 Release

```powershell
& $CMake --build build-windows --config Release --parallel 4
```

`--build build-windows` 使用刚才生成的构建规则；`--config Release` 选择发布配置；`--parallel 4` 最多并行处理四项任务，以缩短编译时间。Release 会进行优化，适合日常运行和发布。Debug 包含更多调试信息、运行较慢，适合在调试器里定位源码问题。两者产物和 Qt DLL 不能混用。

构建成功后主要产物为：

```text
build-windows/Release/Ripes.exe
build-windows/test/Release/tst_assembler.exe
build-windows/test/Release/tst_cosimulate.exe
build-windows/test/Release/tst_expreval.exe
build-windows/test/Release/tst_reverse.exe
build-windows/test/Release/tst_riscv.exe
build-windows/test/Release/tst_stall.exe
```

MSVC 会输出较多 C4267、C4244、C4805 等警告，主要是 `size_t`/整数转换以及现有处理器模板中的布尔与无符号值混用。本次没有把警告当错误，最终编译和链接均成功。修改相关代码时仍应逐项判断是否存在真实的截断风险。

编译输出中的 `warning` 是潜在问题提示，本次不一定阻止生成程序；`error` 会导致当前目标失败。判断是否真正成功，应查看命令退出码并确认末尾出现 `Ripes.vcxproj -> ...\Ripes.exe`。在 PowerShell 中，外部程序运行后可输入 `$LASTEXITCODE`：通常 `0` 表示成功，非 `0` 表示失败。

## 8. 测试

### 8.1 CTest 当前不能直接发现测试

执行：

```powershell
$env:Path = "$QtRoot\bin;$env:Path"
& $CTest --test-dir build-windows -C Release --output-on-failure
```

当前会输出 `No tests were found!!!`。原因是 `test/CMakeLists.txt` 虽然调用了 `add_test`，顶层 CMake 当前没有调用 `enable_testing()` 或 `include(CTest)`，因此没有生成 `CTestTestfile.cmake`。状态码 0 只代表 CTest 自身正常结束，不能视为测试通过。

CTest 是 CMake 配套的“测试调度器”，负责发现并依次启动测试；Qt Test 是实际编写测试用例的 C++ 测试框架。两者不是同一个工具。当前只是 CTest 注册缺失，测试可执行文件本身已经生成，所以接下来直接运行它们。

### 8.2 直接运行 Qt Test

在 PowerShell 中执行：

```powershell
$env:Path = "$QtRoot\bin;$env:Path"
$env:QT_QPA_PLATFORM = 'offscreen'

$tests = @(
  'tst_assembler',
  'tst_expreval',
  'tst_riscv',
  'tst_cosimulate',
  'tst_reverse',
  'tst_stall'
)

foreach ($test in $tests) {
  & ".\build-windows\test\Release\$test.exe" `
    -o ".\build-windows\test\Release\$test.txt,txt"
  Write-Host "$test exit code: $LASTEXITCODE"
}
```

`$env:Path = ...` 只为当前 PowerShell 进程临时增加 Qt DLL 搜索目录，不会永久修改 Windows 环境变量。`$env:QT_QPA_PLATFORM = 'offscreen'` 让界面测试在不显示窗口的模式下运行。`@(...)` 创建列表，`foreach` 对列表中的每个测试重复执行同一组命令；`$LASTEXITCODE` 是刚执行的测试程序返回的状态码。

本机实测结果：

| 测试 | 结果 | Qt Test 汇总或失败点 |
| --- | --- | --- |
| `tst_assembler` | 通过 | 20 passed，0 failed |
| `tst_expreval` | 通过 | 3 passed，0 failed |
| `tst_stall` | 通过 | 5 passed，0 failed |
| `tst_riscv` | 失败 | 2 passed，12 failed；失败集中在 `brk_syscall.S` 和 `ecall_file.s` 的系统调用用例 |
| `tst_cosimulate` | 部分失败 | 6 passed，1 failed；`testRV6SDual()` 报 `No program was loaded!`，其他输出案例均为 PASS |
| `tst_reverse` | 部分失败 | 3 passed，1 failed；`tst_reverse_mem()` 期望 43，实际为 1304 |

三个失败程序均返回状态码 1。日志还重复出现 `QObject::startTimer: Timers cannot have negative intervals`；反向测试在 offscreen 模式下还提示缺少字体和未注册的测试资源图标。这些是当前代码在 Qt 6.9.3/Windows 下的真实基线，不能将本次结果表述为“测试全部通过”。

## 9. 部署 Qt 运行库

只编译出的 `Ripes.exe` 依赖 Qt DLL。先初始化 MSVC x64 环境，再调用 `windeployqt`：

```powershell
$VCVars = 'D:\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvars64.bat'
$DeployQt = 'D:\Qt\6.9.3\msvc2022_64\bin\windeployqt.exe'

& cmd.exe /d /c "call `"$VCVars`" && `"$DeployQt`" --release --compiler-runtime --no-translations `"build-windows\Release\Ripes.exe`""
```

`windeployqt` 会分析 `Ripes.exe` 使用了哪些 Qt 模块，然后复制对应 DLL 和插件。这里借助 `cmd.exe` 调用 Visual Studio 的 `vcvars64.bat`，是因为批处理文件会设置 MSVC 运行库位置；这条命令的引号和转义较复杂，建议整行复制执行，不要拆开改写。部署是复制运行依赖，并不会重新编译程序。

部署后目录包含 `Qt6Core.dll`、`Qt6Gui.dll`、`Qt6Widgets.dll`、`Qt6Svg.dll`、`Qt6Charts.dll`、`Qt6Network.dll`、OpenGL 相关 DLL，以及 `platforms/qwindows.dll` 等插件。若目标机仍提示缺少 `VCRUNTIME140.dll` 或 `MSVCP140.dll`，安装 Microsoft Visual C++ 2015-2022 x64 Redistributable。

`windeployqt` 可能提示找不到 `dxcompiler.dll` 和 `dxil.dll`。当前 QWidget 应用在本机能够启动，该提示没有阻止本次 GUI/CLI 验证；只有使用依赖 DirectX Shader Compiler 的 Qt Quick/RHI 功能时才需要额外处理。

## 10. CLI 冒烟验证

使用五级流水线、M/C 扩展运行内置阶乘程序，并将报告写入 JSON：

CLI（命令行界面）适合自动测试和脚本调用；GUI（图形界面）适合交互使用。PowerShell 中以 `.\` 开头表示运行当前目录或其子目录里的文件。Windows 出于安全考虑通常不会自动在当前目录搜索程序，因此不要省略 `.\`。

```powershell
& .\build-windows\Release\Ripes.exe `
  --mode cli `
  --src .\examples\assembly\factorial.s `
  -t asm `
  --proc RV32_5S `
  --isaexts M,C `
  --cycles --iret --cpi --ipc --runinfo --json `
  --output .\build-windows\cli-smoke.json

Get-Content .\build-windows\cli-smoke.json
```

本机报告为：

```json
{
  "# instructions retired": 16973,
  "CPI": 1.4619100924998527,
  "IPC": 0.6840365937210333,
  "cycles": 24813,
  "runinfo": {
    "ISA extensions": ["M", "C"],
    "processor": "RV32_5S",
    "source file": ".\\examples\\assembly\\factorial.s"
  }
}
```

程序输出包含 `Program exited with code: 0`，证明汇编、程序装载、处理器执行和 telemetry（周期数、CPI 等运行统计）报告链路均能工作。

## 11. GUI 冒烟验证

正常人工启动：

```powershell
& .\build-windows\Release\Ripes.exe
```

本次自动验证使用下面的脚本启动进程，等待 5 秒，确认进程没有异常退出，然后关闭窗口：

```powershell
$proc = Start-Process `
  -FilePath '.\build-windows\Release\Ripes.exe' `
  -WorkingDirectory '.\build-windows\Release' `
  -PassThru

Start-Sleep -Seconds 5
$proc.Refresh()
if ($proc.HasExited) {
  throw "Ripes GUI 启动失败，退出码：$($proc.ExitCode)"
}
$null = $proc.CloseMainWindow()
```

实测进程在 5 秒后仍正常运行，GUI 动态链接和窗口初始化验证通过。

这只是“冒烟验证”，即快速确认程序能启动、不会立刻崩溃；它不能代替完整功能测试。人工启动后还应至少装载一个示例、单步执行并查看处理器视图是否更新。

## 12. 后续增量构建

依赖已经下载、CMake 已配置后，日常只需：

```powershell
$CMake = 'D:\Microsoft Visual Studio\2022\Community\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin\cmake.exe'
& $CMake --build build-windows --config Release --parallel 4
```

修改 CMake 文件或切换 Qt 路径时重新运行“配置 CMake”命令。需要调试时执行：

```powershell
& $CMake --build build-windows --config Debug --parallel 4
```

多配置生成器的 Debug 程序位于 `build-windows/Debug`，运行前应使用 `windeployqt --debug`，不能混用 Release DLL。

修改普通 `.cpp`/`.h` 文件后通常只需再次执行构建命令，MSVC 会只重新编译受影响的文件，这叫“增量构建”。新增文件、修改 `CMakeLists.txt` 或更换 Qt 后应重新执行配置命令。CMake 会把上次选择的路径和选项保存在构建目录的 `CMakeCache.txt` 中，这就是“CMake 缓存”。若旧缓存造成异常，可以新建另一个构建目录（例如 `build-windows-clean`）重新配置；不要为了清理构建缓存而删除 `src`、`external` 或整个仓库。

新手日常开发可以遵循这个循环：修改 `src` 中的源码，执行 Release 或 Debug 增量构建，运行相关测试，再启动 Ripes 检查功能。`git status --short` 用于查看哪些源码或文档被修改，`git diff` 用于查看具体差异；两条命令都只读取状态，不会修改文件。

常见中断可以先按下表检查：

| 现象 | 最先检查 |
| --- | --- |
| 提示找不到 `cmake` | 使用 `$CMake` 保存的完整路径，并运行 `Test-Path $CMake` |
| CMake 提示找不到 Qt6 或 Charts | 检查 `$QtRoot`，以及第 3 节中的两个 Qt CMake 路径是否为 `True` |
| CMake 提示 VSRTL 文件不存在 | 运行 `git submodule status`；开头为 `-` 时重新初始化子模块 |
| PowerShell 把下一行当成新命令 | 检查上一行末尾反引号后是否存在空格，或把多行命令合并成一行 |
| 启动时提示缺少 Qt6 DLL 或平台插件 | 重新执行第 9 节的 `windeployqt`，确认 `platforms/qwindows.dll` 存在 |
| 编译突然失败，但之前能成功 | 从输出中找第一条 `error`；若刚改过 CMake/Qt 路径，用新构建目录重新配置 |
