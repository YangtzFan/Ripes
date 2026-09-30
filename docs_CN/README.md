# Ripes 工程接手指南

本文面向第一次接触 Ripes 源码的工程师，介绍仓库结构、构建和测试方法，以及从源代码到处理器模拟的主要运行链路。本文描述以当前分支代码为准；用户操作说明仍可参考 [`docs/`](../docs/) 下的英文文档。

## 1. 项目定位

Ripes 是一个基于 Qt 的 RISC-V 可视化计算机体系结构模拟器，同时提供汇编编辑器和命令行执行模式。它把“程序装载、汇编/反汇编、处理器模型执行、内存与外设交互、流水线可视化、缓存统计”组合在一个应用中。

这里的 Qt 是一套 C++ 应用开发库。Ripes 用它创建窗口、按钮、图表等界面，也使用它的文件、命令行和测试功能。Qt 不是编译器，不能自己把源码变成程序；Windows 上实际完成编译的是 Visual Studio 附带的 MSVC 编译器。CMake 则读取仓库中的 `CMakeLists.txt`，找到 Qt 和其他依赖，再为 MSVC 生成一套正确的编译规则。可以先记住这条关系：

```text
C++ 源码 + Qt/其他库 → CMake 组织构建规则 → MSVC 编译和链接 → Ripes.exe
```

工程的核心抽象是 `RipesProcessor` 接口：无论处理器是 VSRTL 描述的可视化模型，还是其他后端，只要实现该接口，就可以复用 Ripes 的汇编器、程序查看器、内存、系统调用、I/O、缓存模拟和 CLI。

## 2. 仓库结构

```text
Ripes/
├── main.cpp                  # 应用入口，解析 GUI/CLI 模式并初始化 Qt 资源
├── CMakeLists.txt            # 顶层构建配置、Qt 依赖、资源和可选构建项
├── src/                      # C++ 主体代码
│   ├── assembler/            # RISC-V 汇编器、伪指令、指令匹配、反汇编
│   ├── isa/                  # ISA 描述及 RV32/RV64 扩展定义
│   ├── processors/           # 处理器模型、模型注册、VSRTL 布局 JSON
│   ├── processorhandler.*    # 当前处理器/程序/汇编器的全局协调器
│   ├── io/                   # 内存映射 I/O、LED、开关、屏幕、时钟等外设
│   ├── syscall/              # ecall 和 Ripes 系统调用处理
│   ├── cachesim/             # 指令/数据缓存模拟与统计图表
│   ├── editor/               # 汇编/C 代码编辑器和语法高亮
│   ├── cli/                  # 命令行选项、输入处理和 telemetry 报告
│   ├── utilities/            # Qt/系统工具类
│   └── *.cpp, *.h, *.ui      # 主窗口、寄存器、内存、程序和各类 Qt 界面
├── resources/                # 图标、图片、字体等 Qt 资源
├── examples/                 # 内置汇编、C、ELF 示例，通过 examples.qrc 打包
├── test/                     # Qt Test 单元测试、RISC-V 指令和 C 测试样例
├── external/                 # VSRTL、ELFIO、libelfin、fancytabbar 等依赖
├── docs/                     # 面向使用者的英文文档
├── .github/workflows/        # Linux、Windows、macOS、WASM、测试和格式检查 CI
└── docker/                   # Docker 构建配置
```

`external/VSRTL`、`external/ELFIO` 和 `external/libelfin` 是 Git 子模块或由 CMake FetchContent 获取的依赖。首次克隆应使用 `--recursive`；已有工作副本可执行 `git submodule update --init --recursive`。

“依赖”是本工程需要、但由其他项目提供的代码或库。“Git 子模块”是在主仓库中记录另一个仓库的固定版本；只克隆 Ripes 而不初始化子模块时，相应目录可能是空的，编译自然无法继续。FetchContent 是 CMake 在首次配置时自动下载依赖的另一种方式。

### 2.1 CMake 目标关系

顶层 CMake 创建应用目标 `Ripes`，并先构建静态库 `ripes_lib`。`src/CMakeLists.txt` 将 ISA、汇编器、缓存、编辑器、系统调用、I/O、处理器和 CLI 等子库链接到 `ripes_lib`，最后由 `main.cpp` 链接 Qt 和 `ripes_lib` 生成桌面程序。开启测试时，`test/CMakeLists.txt` 会额外生成 `tst_riscv`、`tst_assembler`、`tst_expreval`、`tst_cosimulate`、`tst_reverse` 和 `tst_stall`。

在 CMake 语境中，“目标（target）”就是一次构建要得到的东西，可能是可执行程序，也可能是供其他目标使用的库。“静态库”会在链接阶段合并进最终程序；“链接”是把各个已编译模块及其引用的库组合成 `Ripes.exe` 的步骤。

处理器子目录使用 `src/processors/CMakeLists.txt` 中的宏批量生成库。新增模型通常还要同时修改 `processorregistry.cpp/.h`、`layouts.qrc` 和处理器目录的 CMake 配置。

## 3. 构建环境

当前顶层 CMake 要求：

- C++20 编译器；Windows 实测使用 MSVC。编译器负责把 `.cpp` 源码翻译成机器代码，C++20 是这份源码使用的语言标准版本。
- CMake 3.13 或更高版本。CMake 负责配置和调度构建，本身不代替编译器。
- Qt 6，且包含 `Core`、`Widgets`、`Svg`、`Charts` 组件。它们分别提供基础功能、桌面控件、SVG 图形和图表。当前代码使用 `find_package(Qt6 6.8 ...)`，因此建议安装 Qt 6.8 或更高版本；仓库 README 中的“Qt >= 6.5”属于较早版本说明。
- Linux 通常还需要 OpenGL/EGL 开发包，例如 `libegl1-mesa-dev`（CI 还安装 `libgl1-mesa-dev` 等桌面依赖）。

Qt 的 CMake 包路径必须能被找到。`CMAKE_PREFIX_PATH` 相当于告诉 CMake“到这个目录寻找 Qt”。若 Qt 不在系统默认路径中，就把它指向 Qt 安装目录或其 `lib/cmake` 目录。它只是配置参数，不会移动或复制 Qt。Windows 上也可以直接使用 Qt 提供的开发者命令行环境。

## 4. 推荐构建流程

### 4.1 Linux/macOS

```bash
git clone --recursive <仓库地址>
cd Ripes
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_PREFIX_PATH=/path/to/Qt/6.8.x/<kit>
cmake --build build --parallel
```

构建调试版本时把 `Release` 改为 `Debug`。如果已配置好 Qt，也可以省略 `CMAKE_PREFIX_PATH`。生成的可执行文件通常位于 `build/Ripes`（具体位置取决于生成器）。

### 4.2 Windows PowerShell

本仓库已经在一台 Windows 11 主机上用 Visual Studio 2022、Qt 6.9.3 完成实际构建、部署和运行验证。首次搭建环境时，建议直接按照 [`WINDOWS_BUILD.md`](WINDOWS_BUILD.md) 操作；其中记录了本机使用的准确路径、Qt 安装方法、SSH 拉取依赖、测试结果和故障排查。下面的命令只作为通用写法。

“配置”和“编译”是两个不同阶段：第一条 `cmake -S ... -B ...` 检查工具和依赖并生成构建文件；第二条 `cmake --build ...` 才真正调用编译器。配置失败时不要继续编译，应先解决配置输出中的第一条错误。

```powershell
git clone --recursive <仓库地址>
Set-Location Ripes
cmake -S . -B build -G Ninja `
  -DCMAKE_BUILD_TYPE=Release `
  -DCMAKE_PREFIX_PATH="C:\Qt\6.8.x\msvc2022_64"
cmake --build build --parallel
```

也可以使用 Visual Studio 生成器；此时配置阶段不必设置 `CMAKE_BUILD_TYPE`，构建时使用 `--config Release`：

```powershell
cmake -S . -B build -DCMAKE_PREFIX_PATH="C:\Qt\6.8.x\msvc2022_64"
cmake --build build --config Release --parallel
```

运行发布程序时，若系统缺少 Qt DLL，可用 Qt 的 `windeployqt` 将运行时依赖复制到程序目录。Linux/macOS 发布流程分别使用 AppImage 和 macOS bundle，细节可参考 `.github/workflows/`。

### 4.3 常用 CMake 选项

```text
-DRIPES_BUILD_TESTS=ON                 构建 Qt Test 测试目标
-DRIPES_WITH_QPROCESS=OFF              关闭依赖 QProcess 的工具（默认 ON）
-DRIPES_BUILD_VERILATOR_PROCESSORS=ON  构建实验性的 Verilator 处理器
-DCMAKE_BUILD_TYPE=Debug               便于调试的构建类型
```

启用 Verilator 模型时必须先设置 `VERILATOR_ROOT`，否则配置阶段会直接失败。WASM 构建还需要 Emscripten 和 Qt WebAssembly 工具链，CI 示例见 `.github/workflows/wasm-release.yml`。

## 5. 运行方式

不带参数启动进入 GUI：

```bash
./build/Ripes
```

入口 `main.cpp` 先调用 `Q_INIT_RESOURCE` 注册图标、示例、处理器布局和字体，再解析 `--mode`。`--mode gui`（默认）创建 `Ripes::MainWindow`；`--mode cli` 创建 `Ripes::CLIRunner`，不依赖窗口交互即可汇编、运行并输出统计。

CLI 示例：

```bash
./build/Ripes --mode cli \
  --src examples/assembly/factorial.s \
  -t asm --proc RV32_5S --cycles --iret --cpi --ipc --json
```

完整选项以 `./build/Ripes --help` 为准。常用参数包括 `--src`（输入文件）、`-t`（`asm`/`c`/`bin`）、`--proc`（处理器模型）、`--isaexts`（扩展）、`--timeout`、`--output`、`--pipeline`、`--regs` 和 `--runinfo`。C 文件的编译器配置和限制见 [`docs/c_programming.md`](../docs/c_programming.md)。

## 6. 主要工作原理

### 6.1 初始化与处理器选择

`ProcessorRegistry` 是处理器模型的目录。每个条目包含模型 ID、显示名、描述、支持的 ISA、默认寄存器值和布局资源，并能构造一个 `RipesProcessor`。当前注册表包含 RV32/RV64 的：软件 ISA 解释器（ISS）、单周期、5 级流水线（含有无转发/冒险处理的变体）、多周期（单内存/分离内存）和 6 级双发射模型。

`ProcessorHandler` 负责销毁旧模型、按注册表构造新模型、选择 ISA 扩展、装载程序、复位/运行/单步/反向执行，并把处理器信号转发给 GUI、缓存模拟器和 CLI。理解功能时通常先从 `processorhandler.h/.cpp` 看状态如何流动，再进入具体模型。

### 6.2 从源代码到内存

1. 编辑器、CLI 或打开文件得到汇编/C/ELF 输入。
2. C 输入由 `CCManager` 调用外部编译器生成目标内容；汇编输入交给 `src/assembler`。汇编器按当前 ISA 解析标签、伪指令和 GNU directives，生成 `Program`（代码段、数据段、符号表和入口地址）。ELF 输入通过 ELFIO 读取。
3. `ProcessorHandler::loadProgram` 将 `Program` 写入当前处理器的地址空间，并设置程序计数器、栈指针等默认值。`IOManager` 将内存地址映射到外设，系统调用通过处理器的 `trapHandler` 回到 Ripes 环境。
4. 程序执行过程中，处理器每次 `clock()` 推进一个时钟周期；流水线模型在周期内更新各级寄存器、控制信号、存储器和寄存器堆，ISS 模型则解释并完成一条指令。处理器通过 `processorWasClocked` 等信号通知 UI 更新。
5. 当 PC 离开 `.text` 段或执行退出 ecall 时，`finalize` 让流水线排空；`finished()` 为真后运行结束。CLI 根据周期数、退休指令数、寄存器和流水线状态生成报告。

可以把执行链路概括为：

```text
输入文件 → 汇编/编译/ELF 加载 → Program → ProcessorHandler
        → 处理器 clock → 内存/I/O/系统调用 → UI、缓存统计或 CLI 报告
```

### 6.3 可视化和布局

VSRTL 模型在 C++ 中描述元件和连线，`*_standard_layout.json` 与 `*_extended_layout.json` 描述画布上的位置、端口、标签和连线。布局通过 `src/processors/layouts.qrc` 编译进资源系统，注册表使用 `:/layouts/...` 路径加载。标准布局用于教学时隐藏次要细节，扩展布局显示完整数据通路。

### 6.4 缓存、I/O 和系统调用

缓存模拟器位于 `src/cachesim`，通过 `ProcessorHandler` 获取当前 ISA、内存访问和周期信号，记录命中/未命中并提供图表视图。I/O 设备位于 `src/io`，通过内存映射地址访问；当前包含 LED、开关、屏幕、LED 矩阵和时钟等设备。`src/syscall` 根据 ABI 寄存器解释 ecall，处理控制台、文件和退出等行为。

## 7. 测试与验证

开启测试并构建：

```bash
cmake -S . -B build-tests -DRIPES_BUILD_TESTS=ON -DCMAKE_BUILD_TYPE=Debug
cmake --build build-tests --parallel
ctest --test-dir build-tests --output-on-failure
```

上面是常见的通用 CMake 测试流程。当前工程的 CMake 配置缺少 CTest 注册，在本机直接执行最后一条会显示 `No tests were found`，这不代表测试通过；Windows 请按照 [`WINDOWS_BUILD.md`](WINDOWS_BUILD.md) 的“测试”章节直接运行生成的 Qt Test 程序。

`tst_assembler` 和 `tst_expreval` 覆盖汇编与表达式求值；`tst_riscv` 验证指令行为；`tst_reverse` 和 `tst_stall` 覆盖反向执行及流水线停顿；`tst_cosimulate` 用参考模型对比目标模型的寄存器变化，是新增处理器模型时最有价值的回归测试。测试样例位于 `test/riscv-tests*` 和 `test/*.cpp`。

CI 还会执行 Clang 格式检查，并分别验证 Linux、Windows、macOS 和 WASM 构建。提交前至少完成对应平台的 CMake 配置、编译和测试；Windows 当前应直接运行 Qt Test 程序。修改 Qt UI、资源或处理器布局时要额外启动 GUI 手工检查。

## 8. 常见开发任务

### 新增处理器模型

1. 在 `src/processors/<ISA>/<model>/` 编写继承 `RipesProcessor`（VSRTL 模型通常使用现有基类）的头文件和实现，明确时钟、复位、PC、寄存器、内存访问、完成条件和 ISA 支持。
2. 编写标准/扩展布局 JSON，并加入 `src/processors/layouts.qrc`。
3. 在 `src/processors/CMakeLists.txt` 用现有宏加入模型库。
4. 在 `processorregistry.h/.cpp` 增加 `ProcessorID`、描述、默认寄存器值、布局资源和 `ProcInfo` 注册项。枚举顺序决定 GUI 选择框的显示顺序。
5. 在 `test/tst_riscv.cpp` 和 `test/tst_cosimulate.cpp` 增加测试；优先用 ISS 或单周期模型作为参考模型进行协模拟。

具体布局编辑和 Verilator 集成说明可参考 [`docs/README.md`](../docs/README.md) 的对应章节。Verilator 路径是实验功能，必须同时维护 CMake 选项、`VERILATOR_ROOT` 和 `RipesProcessor` 封装。

### 修改指令或 ISA

ISA 元数据和指令定义在 `src/isa`，汇编/反汇编匹配逻辑在 `src/assembler`。修改后需要同时考虑：指令编码与解码、立即数和重定位、伪指令、RV32/RV64 差异、处理器控制逻辑，以及汇编器和处理器测试。不要只修改 UI 中的指令列表。

### 修改 GUI 或资源

Qt Designer 文件是 `*.ui`，图标、字体和示例通过 `*.qrc` 编译进可执行文件。新增资源后必须更新相应 qrc，并确认 `main.cpp` 中存在对应的 `Q_INIT_RESOURCE`（处理器布局由资源文件统一加载）。

## 9. 调试建议与注意事项

- 首先确认当前选择的 `ProcessorID`、ISA 扩展和布局；同一条程序在“无冒险检测”模型上出现错误可能是模型设计意图，而非汇编器故障。
- 调试执行错误时，先用 RV32/RV64 ISS 作为参考，比较 PC、退休指令和寄存器变化，再到具体流水级查看控制信号和数据通路。
- 处理器模型通过回调访问 `isExecutableAddress` 和 `trapHandler`，不要在模型内部复制一套系统调用或程序边界逻辑。
- `RIPES_BUILD_TESTS` 默认关闭；只编译主程序并不能证明处理器或汇编器行为正确。
- 资源路径使用 Qt 资源前缀（例如 `:/layouts/...`），文件系统路径在源码树中能找到并不代表运行时已打包。
- `src/CMakeLists.txt` 使用 `file(GLOB ...)` 收集部分源文件。新增 `.cpp/.h/.ui` 后通常会被发现，但新增处理器仍需显式调用处理器构建宏和注册流程。
- 修改外部依赖或子模块版本时，同时检查 CI 工作流和发布脚本，避免本地能构建而打包失败。

## 10. 建议的接手顺序

先按本文件的命令完成一次 Release 构建，再运行 GUI 打开一个 `examples/assembly` 示例；随后阅读 `main.cpp`、`processorhandler.*`、`processorregistry.*` 和 `processors/interface/ripesprocessor.h`。接着用 `src/assembler`、`src/io`、`src/syscall` 追踪一次程序装载和 ecall，再选择 `rvss` 或 `rv5s` 阅读一个完整处理器模型及其布局 JSON。最后运行协模拟测试，并根据要负责的领域深入 `cachesim`、`editor` 或具体处理器目录。

版本、发布和面向用户的功能说明会随项目更新，接手时应以当前分支的 CMake、CI 工作流和测试结果为最终依据。
