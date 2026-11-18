# CMake + Arm GCC + Ninja + OpenOCD 开发单片机：以本项目为例

> 目标芯片：STM32H723（Cortex-M7）
> 项目根目录：`/home/zhangjiayi/RL/26_AT_DOG_LOWER`
> 本文所有配置片段均取自本项目真实文件。

---

## 一、先弄清四个工具各自的角色

单片机开发和 PC 软件开发最大的区别是**交叉编译**：编译程序运行在 x86 PC 上，生成的机器码却要跑在 ARM 芯片上。围绕这件事，四个工具各管一段：

| 工具 | 角色 | 一句话职责 | 在本项目中的体现 |
|------|------|-----------|-----------------|
| **CMake** | 构建系统**生成器** | 用 `CMakeLists.txt` 描述"有哪些源文件、头文件路径、宏、链接选项"，生成底层构建工具的脚本 | 顶层 `CMakeLists.txt` + `cmake/stm32cubemx/CMakeLists.txt` |
| **Ninja** | 构建**执行器** | 拿着 CMake 生成的 `build.ninja`，按依赖图并行地调用编译器/汇编器/链接器 | `CMakePresets.json` 里 `"generator": "Ninja"` |
| **Arm GCC** | **交叉编译器/链接器** | `arm-none-eabi-gcc` 把 `.c/.s` 编译成 ARM 机器码，最终链接成 `.elf` | `cmake/gcc-arm-none-eabi.cmake` 工具链文件 |
| **OpenOCD** | 烧录/调试**服务端** | 通过调试探头（ST-Link/CMSIS-DAP/J-Link）的 **SWD** 接口把程序写进 Flash，并作为 GDB Server 供单步调试 | 本项目**尚未配置**（目前只有 MDK-ARM Keil 工程），见第五节 |

数据流可以串成一条线：

```
main.c / freertos.c / MyLib/*.c / MyTask/*.c   （源码）
        │  CMake 用 CMakeLists.txt 描述出来
        ▼
  cmake --preset Debug      →  生成 build/Debug/build.ninja
        │
        ▼
  cmake --build --preset Debug  →  Ninja 调度 arm-none-eabi-gcc
        │       .c → .obj → 链接（-T STM32H723xG_flash.ld）
        ▼
  build/Debug/Dog_lower.elf （可烧录/可调试的固件）
        │  OpenOCD 经 SWD 写入芯片 Flash
        ▼
  目标板上电运行 + GDB 在线调试
```

**为什么改用这套而不是 Keil？** 本项目同时保留了 `MDK-ARM/`（Keil 工程）和 CMake 两套构建。CMake + Ninja + Arm GCC 的优势是：跨平台（Linux/Windows/macOS 一致）、纯命令行可脚本化（适合 CI 自动化编译）、免费开源、编辑器（VS Code / CLion + clangd）索引体验好。Keil 的优势是图形化调试开箱即用。二者并不冲突，源码是同一份。

---

## 二、本项目关键文件逐个拆解

### 2.1 `CMakePresets.json` —— 把命令行参数固化成"预设"

预设（Preset）解决的是"每次敲一长串 cmake 参数太麻烦"的问题。本项目内容：

```json
{
  "version": 3,
  "configurePresets": [
    {
      "name": "default",
      "hidden": true,
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/${presetName}",
      "toolchainFile": "${sourceDir}/cmake/gcc-arm-none-eabi.cmake"
    },
    {
      "name": "Debug",
      "inherits": "default",
      "cacheVariables": { "CMAKE_BUILD_TYPE": "Debug" }
    },
    {
      "name": "Release",
      "inherits": "default",
      "cacheVariables": { "CMAKE_BUILD_TYPE": "Release" }
    }
  ],
  "buildPresets": [
    { "name": "Debug", "configurePreset": "Debug" },
    { "name": "Release", "configurePreset": "Release" }
  ]
}
```

要点：

- `default` 是隐藏的公共模板，`Debug` / `Release` 通过 `inherits` 继承它，只覆盖构建类型。
- `binaryDir` 决定产物放在 `${sourceDir}/build/${presetName}` —— 即 `build/Debug` 或 `build/Release`，两种构建互不干扰。
- `toolchainFile` 指向交叉编译工具链文件，这是"给 ARM 编译"而非"给 PC 编译"的关键开关。
- **下面这两个预设名字都叫 `Debug`，但属于不同类别**：`configurePresets` 里的 `Debug` 用于配置阶段，`buildPresets` 里的 `Debug` 用于构建阶段，并通过 `configurePreset` 关联到前者。

### 2.2 `cmake/gcc-arm-none-eabi.cmake` —— 工具链文件

CMake 默认认为在给本机编译；工具链文件把目标平台改成裸机 ARM：

```cmake
set(CMAKE_SYSTEM_NAME Generic)          # 裸机（无操作系统）
set(CMAKE_SYSTEM_PROCESSOR arm)

set(TOOLCHAIN_PREFIX arm-none-eabi-)
set(CMAKE_C_COMPILER   ${TOOLCHAIN_PREFIX}gcc)   # 编译 C
set(CMAKE_ASM_COMPILER ${CMAKE_C_COMPILER})      # 编译启动汇编
set(CMAKE_CXX_COMPILER ${TOOLCHAIN_PREFIX}g++)
set(CMAKE_OBJCOPY      ${TOOLCHAIN_PREFIX}objcopy)
set(CMAKE_SIZE         ${TOOLCHAIN_PREFIX}size)

set(CMAKE_EXECUTABLE_SUFFIX_C ".elf")   # 可执行文件后缀改为 .elf

# MCU 相关的编译选项（目标 CPU / FPU / 浮点 ABI）
set(TARGET_FLAGS "-mcpu=cortex-m7 -mfpu=fpv5-d16 -mfloat-abi=hard ")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${TARGET_FLAGS}")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -fdata-sections -ffunction-sections -fstack-usage")

set(CMAKE_C_FLAGS_DEBUG   "-O0 -g3")    # Debug：不优化 + 完整调试信息
set(CMAKE_C_FLAGS_RELEASE "-Os -g0")    # Release：体积优化 + 无调试信息

# 链接选项：链接脚本 + 精简 C 库 + 生成 map + 垃圾回收段 + 打印内存占用
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -T \"${CMAKE_SOURCE_DIR}/STM32H723xG_flash.ld\"")
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} --specs=nano.specs")
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,-Map=${CMAKE_PROJECT_NAME}.map -Wl,--gc-sections")
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,--print-memory-usage")
set(TOOLCHAIN_LINK_LIBRARIES "m")       # 链接 libm
```

几个关键点：

- `arm-none-eabi-` 必须在 `PATH` 中（本项目是 `/usr/bin/arm-none-eabi-gcc`）。
- `-mcpu=cortex-m7 -mfpu=fpv5-d16 -mfloat-abi=hard` 让编译器生成 Cortex-M7 硬件浮点指令；选项错了程序可能能编译但运行异常。
- `-T STM32H723xG_flash.ld` 是**链接脚本**，告诉链接器 Flash/RAM 的地址范围、入口复位向量、`.text`/`.data`/`.bss` 各段放在哪里——这是裸机程序和 Linux 程序最大的区别之一。
- `-fdata-sections -ffunction-sections` 配合 `-Wl,--gc-sections`，把没用到的函数/数据从最终固件里删掉，减小体积。
- `--specs=nano.specs` 使用精简版 newlib，printf/浮点格式化等体积更小。
- `CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY`（文件第 22 行）让 CMake 在检测编译器时只编译静态库而不尝试链接可执行程序——裸机没有 C 运行时，直接链接会失败。

### 2.3 顶层 `CMakeLists.txt` —— 项目自己的源码

```cmake
cmake_minimum_required(VERSION 3.22)
set(CMAKE_C_STANDARD 11)
set(CMAKE_PROJECT_NAME Dog_lower)
project(${CMAKE_PROJECT_NAME})
enable_language(C ASM)

add_executable(${CMAKE_PROJECT_NAME})          # 声明一个可执行目标（暂不列源码）
add_subdirectory(cmake/stm32cubemx)            # 引入 CubeMX 生成的 HAL/FreeRTOS/USB 等

# 用户自己的源码（我们在前面补全的部分）
target_sources(${CMAKE_PROJECT_NAME} PRIVATE
    MyLib/485_bus.c
    MyLib/bezier.c
    # ... 共 14 个 MyLib/*.c ...
    MyTask/app_main.c
    MyTask/run.c
    MyTask/ws2812.c
)

target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE
    MyLib
    MyTask
)

target_link_libraries(${CMAKE_PROJECT_NAME}
    stm32cubemx     # CubeMX 提供的 INTERFACE 库（头文件路径 + 全局宏）
)
```

注意：`add_executable()` 时**没有列源文件**，源码是后面用多个 `target_sources()` 逐步追加的——CubeMX 生成的那部分在子目录里用 `target_sources(${CMAKE_PROJECT_NAME} ...)` 加进来（见 `cmake/stm32cubemx/CMakeLists.txt` 第 137 行）。这是 CubeMX 工程模板的常见写法。

### 2.4 `cmake/stm32cubemx/CMakeLists.txt` —— CubeMX 生成部分

这个文件由 STM32CubeMX 生成，不要手改（重新生成 .ioc 时会被覆盖）。它做了几件事：

1. 定义全局宏 `USE_HAL_DRIVER`、`STM32H723xx`、`DEBUG`（Debug 配置下）——见第 5-10 行；
2. 列出所有头文件路径 `MX_Include_Dirs`（Core/Inc、HAL、CMSIS、FreeRTOS、USB...）——第 13-26 行；
3. 把源码分组为若干 **OBJECT 库**：`STM32_Drivers`（HAL 驱动）、`USB_Device_Library`、`FreeRTOS`——第 121-134 行；
4. 用一个 `INTERFACE` 库 `stm32cubemx` 统一向外传播头文件路径和宏——第 116-118 行；
5. 把应用源码（`main.c`、`freertos.c`、各外设 `.c`、启动文件）加到主目标——第 137 行；
6. 最后把这些库链接到主目标——第 143 行。

**这也是为什么 `freertos.c` 里 `#include "app_main.h"` 之前会报错**：CubeMX 提供的 include 路径里没有 `MyTask/`，必须在顶层 `target_include_directories` 里补上。

### 2.5 与 Keil 构建的差异

`MDK-ARM/` 下的 `.uvprojx` 是 Keil 的工程描述，编译器是 `armclang`，链接脚本是 `RAM_DMA.sct`（ARM 的散射加载文件格式）。而 CMake 这套用的是 GCC 的 `.ld` 链接脚本（`STM32H723xG_flash.ld`）。两者是**两套独立的构建描述**，共用同一份 C 源码，因此源码里要注意可移植性（例如前面遇到的大小写敏感问题：Windows 下 `pid_old.h` 能匹配 `PID_old.h`，Linux 下不行）。

---

## 三、`cmake --build --preset Debug` 背后具体发生了什么

这是本文的重点。我们把这条命令一层层剥开。

### 3.1 第一层：`cmake --build` 是"构建前端"

`cmake` 本身并不编译任何 `.c` 文件。它有两个主要工作模式：

- **配置模式**：`cmake -S . -B build/Debug -G Ninja ...`，读取 `CMakeLists.txt`，生成构建脚本。
- **构建模式**：`cmake --build <dir>`，切换到构建模式，**调用配置阶段选定的底层构建工具**（这里是 Ninja）去真正编译。

`cmake --build` 的好处是统一了接口：不管底层是 Ninja、Make、Visual Studio 还是 Xcode，用户都敲同一条命令。所以：

```
cmake --build --preset Debug   ≈   ninja -C build/Debug
```

### 3.2 第二层：`--preset Debug` 如何解析成构建目录

`--build --preset Debug` 引用 `CMakePresets.json` 中 `buildPresets` 里名为 `Debug` 的条目：

```json
{ "name": "Debug", "configurePreset": "Debug" }
```

它只有 `configurePreset` 一个字段——意思是"我去找配置预设 `Debug` 对应的构建目录"。于是再查 `configurePresets` 里的 `Debug`：

```json
{ "name": "Debug", "inherits": "default", ... }
```

它继承 `default` 的 `binaryDir: ${sourceDir}/build/${presetName}` 和 `generator: Ninja`，所以最终得到：

- 构建目录：`/home/zhangjiayi/RL/26_AT_DOG_LOWER/build/Debug`
- 底层工具：Ninja

完整等价链条：

```
cmake --build --preset Debug
        │  --preset Debug → buildPresets[Debug] → configurePreset = "Debug"
        │  configurePresets[Debug] → binaryDir = build/Debug, generator = Ninja
        ▼
cmake --build build/Debug
        ▼
ninja -C build/Debug
```

### 3.3 第三层：构建前的"自动重新配置"检查

这是很多人忽略的机制。Ninja 在执行前会检查 `build.ninja` 是否过期——它依赖 `CMakeLists.txt`、`CMakePresets.json`、工具链文件等。如果这些文件比 `build.ninja` 新（比如你刚改了 `CMakeLists.txt`），Ninja 会**自动重跑一次 CMake 配置**来重新生成 `build.ninja`，然后再继续构建。

所以改完 `CMakeLists.txt` 后直接 `cmake --build --preset Debug` 就能生效，不必手动重新 `cmake --preset Debug`。

但注意**边界**：如果构建目录根本不存在（第一次用），`cmake --build` 不会帮你创建它，而是直接报错：

```
Error: .../build/Debug is not a directory
```

这就是之前遇到的问题。正确顺序是**先配置、后构建**：

```bash
cmake --preset Debug            # 第一次：生成 build/Debug 和 build.ninja
cmake --build --preset Debug    # 之后：重复执行即可，改了 CMakeLists 也会自动重配
```

### 3.4 第四层：Ninja 真正执行了什么

Ninja 读取 `build/Debug/build.ninja`，按依赖图并行调度命令。以本项目真实日志为例，它做的事依次是：

**1）编译每个 `.c` 为 `.obj`**（真正干活的是 `arm-none-eabi-gcc`）：

```bash
/usr/bin/arm-none-eabi-gcc \
  -DDEBUG -DSTM32H723xx -DUSE_HAL_DRIVER -DUSE_PWR_LDO_SUPPLY \
  -I/home/.../MyLib -I/home/.../MyTask \
  -I/home/.../Core/Inc -I/home/.../Drivers/STM32H7xx_HAL_Driver/Inc \
  -I/home/.../Middlewares/Third_Party/FreeRTOS/Source/include \
  -mcpu=cortex-m7 -mfpu=fpv5-d16 -mfloat-abi=hard \
  -Wall -fdata-sections -ffunction-sections -fstack-usage \
  -O0 -g3 -std=gnu11 \
  -MD -MT CMakeFiles/Dog_lower.dir/MyTask/run.c.obj \
  -MF CMakeFiles/Dog_lower.dir/MyTask/run.c.obj.d \
  -o CMakeFiles/Dog_lower.dir/MyTask/run.c.obj \
  -c /home/zhangjiayi/RL/26_AT_DOG_LOWER/MyTask/run.c
```

逐项对应：
- `-D...`：来自 CubeMX 的 `MX_Defines_Syms` 和 Debug 预设的 `CMAKE_BUILD_TYPE=Debug`；
- `-I...`：来自 `target_include_directories`（MyLib/MyTask）和 CubeMX 的 `MX_Include_Dirs`；
- `-mcpu/-mfpu/-mfloat-abi`：来自工具链文件的 `TARGET_FLAGS`；
- `-O0 -g3`：来自 `CMAKE_C_FLAGS_DEBUG`；
- `-MD -MF`：生成 `.d` 依赖文件，让 Ninja 知道这个 `.c` 依赖了哪些头文件——改了某个 `.h`，只重新编译受影响的 `.c`；
- 启动文件 `startup_stm32h723xx.s` 由同一编译器以汇编模式处理。

**2）链接为 `.elf`**：

```bash
/usr/bin/arm-none-eabi-gcc \
  -mcpu=cortex-m7 -mfpu=fpv5-d16 -mfloat-abi=hard \
  -O0 -g3 \
  -T "/home/zhangjiayi/RL/26_AT_DOG_LOWER/STM32H723xG_flash.ld" \
  --specs=nano.specs \
  -Wl,-Map=Dog_lower.map -Wl,--gc-sections -Wl,--print-memory-usage \
  <所有 .obj 文件> \
  -o Dog_lower.elf -lm
```

链接器按链接脚本把各段放到 Flash/RAM 的正确地址，解析所有符号（函数调用、变量引用）。**任何 `.c` 里调用了不存在的函数都会在这里报 `undefined reference`**——例如之前 `taskEX15IT_CRITICAL()` 拼错，就是这一步暴露的。

**3）打印内存占用**（`-Wl,--print-memory-usage` 的效果）：

```
         DTCMRAM:       30616 B       128 KB     23.36%
          RAM_D1:          0 B       320 KB      0.00%
           FLASH:       95408 B         1 MB      9.10%
```

### 3.5 产物清单（`build/Debug/` 下）

| 文件 | 来源 | 用途 |
|------|------|------|
| `Dog_lower.elf` | 链接产物 | 含符号和调试信息，用于 GDB 调试、OpenOCD 烧录 |
| `Dog_lower.map` | `-Wl,-Map` | 符号/段地址映射表，排查 Flash/RAM 溢出时看它 |
| `*.obj` | 编译产物 | 中间目标文件 |
| `*.obj.d` | `-MD` | 依赖文件，供 Ninja 做增量编译 |
| `build.ninja` | CMake 生成 | Ninja 的构建脚本 |
| `compile_commands.json` | `CMAKE_EXPORT_COMPILE_COMMANDS` | 供 clangd/VS Code 做代码跳转和补全 |

其中 `.elf` 已经被 `.gitignore` 忽略（第 15 行），整个 `build/` 目录也已被忽略，所以构建产物不会污染 Git 仓库。

### 3.6 一条命令总结

> `cmake --build --preset Debug` = 读取预设 → 定位到 `build/Debug` → 必要时重新生成构建脚本 → 调用 Ninja → Ninja 调用 `arm-none-eabi-gcc` 编译所有改动过的源码并链接 → 输出 `Dog_lower.elf`。

---

## 四、完整的开发闭环

```bash
# 1. 首次配置（只需一次；改了 .ioc 重新生成代码后也建议重跑）
cmake --preset Debug

# 2. 编译（日常反复执行）
cmake --build --preset Debug

# 3. 烧录（需先配置 OpenOCD，见第五节）
openocd -f interface/stlink-v2.cfg -f target/stm32h7x.cfg \
        -c "program build/Debug/Dog_lower.elf verify reset exit"

# 4. 调试：一个终端开 OpenOCD（GDB Server），另一个终端开 GDB
openocd -f interface/stlink-v2.cfg -f target/stm32h7x.cfg
arm-none-eabi-gdb build/Debug/Dog_lower.elf \
  -ex "target extended-remote :3333" \
  -ex "monitor reset halt" -ex "load" -ex "continue"
```

清理构建：`cmake --build --preset Debug --target clean`，或干脆删掉 `build/Debug` 重新配置。

---

## 五、OpenOCD 配合 SWD 接口：本项目的下一个方向

### 5.1 SWD 是什么，为什么用它

**SWD（Serial Wire Debug）** 是 ARM 推出的两线调试接口，相对于传统 JTAG（4~5 根线）引脚更少，是 Cortex-M 系列最常用的调试/烧录接口：

| 信号 | 方向 | 说明 |
|------|------|------|
| `SWDIO` | 双向 | 串行数据线 |
| `SWCLK` | PC → 芯片 | 时钟线 |
| `GND` | — | 共地，**必须接** |
| `VCC` | 可选 | 调试器给目标板供电或检测目标电压（3.3V） |
| `NRST` | 可选 | 硬件复位线，便于可靠复位 |
| `SWO` | 可选 | 跟踪输出，配合 ITM 做 printf 调试 |

PC 端不能直连 SWD，需要一个**调试探头**把 USB 转成 SWD 信号：ST-Link V2/V3、CMSIS-DAP、J-Link 都常见。接线时至少接 `SWDIO`、`SWCLK`、`GND` 三根，必要时加 `NRST`。注意目标板供电与调试器电压匹配（本项目 3.3V）。

### 5.2 OpenOCD 的工作模型

OpenOCD 跑在 PC 上，配置由两类文件拼成：

1. **interface 配置**：描述调试探头，如 `interface/stlink-v2.cfg`、`interface/cmsis-dap.cfg`；
2. **target 配置**：描述目标芯片，STM32H7 系列用 `target/stm32h7x.cfg`。

启动后它同时提供：

- **Flash 编程能力**（`program` 命令），用于烧录；
- **GDB Server**（默认 TCP 3333 端口），供 `arm-none-eabi-gdb` 连接做单步、断点、查看变量；
- **Telnet 管理口**（4444 端口），可直接发 TCL 命令。

### 5.3 把 OpenOCD 接进本项目的两种方式

**方式一：手动命令（验证阶段最直观）**

```bash
# 烧录并复位运行
openocd -f interface/stlink-v2.cfg -f target/stm32h7x.cfg \
        -c "program build/Debug/Dog_lower.elf verify reset exit"
```

**方式二：在顶层 `CMakeLists.txt` 加自定义目标（推荐，和构建流程一体化）**

```cmake
find_program(OPENOCD openocd)
if(OPENOCD)
  add_custom_target(flash
    COMMAND ${OPENOCD}
            -f interface/stlink-v2.cfg
            -f target/stm32h7x.cfg
            -c "program ${CMAKE_BINARY_DIR}/${CMAKE_PROJECT_NAME}.elf verify reset exit"
    DEPENDS ${CMAKE_PROJECT_NAME}
    COMMENT "Flashing ${CMAKE_PROJECT_NAME}.elf via OpenOCD/SWD"
  )
endif()
```

之后就能：

```bash
cmake --build --preset Debug --target flash
```

`DEPENDS` 保证每次烧录前先编译出最新的 `.elf`。

### 5.4 调试工作流

```bash
# 终端 1：启动 GDB Server（保持运行，不要退出）
openocd -f interface/stlink-v2.cfg -f target/stm32h7x.cfg

# 终端 2：GDB 连接
arm-none-eabi-gdb build/Debug/Dog_lower.elf
(gdb) target extended-remote :3333
(gdb) monitor reset halt
(gdb) load
(gdb) break app_main
(gdb) continue
```

VS Code 用户可安装 **Cortex-Debug** 扩展，在 `launch.json` 里配置 `device: STM32H723xx`、`interface: swd`、`gdbPath` 和 OpenOCD 配置文件路径，即可图形化单步调试，配合已有的 `compile_commands.json` 获得代码补全。

### 5.5 本项目现状

当前仓库里**没有任何 OpenOCD 配置文件或 CMake flash 目标**：调试走的是 `MDK-ARM/` 下的 Keil 工程 + 商业调试器。若要把 OpenOCD 纳入本项目：

1. 安装 OpenOCD（如 `sudo apt install openocd`），确认 `interface/` 和 `target/` 配置目录路径；
2. 按 5.4 的方式把 `flash` 目标加入顶层 `CMakeLists.txt`（注意该文件是用户维护的，改它不会被 CubeMX 覆盖）；
3. 若芯片 Flash 是双 Bank 或需要特定擦除算法，可在 `target/stm32h7x.cfg` 基础上调整；
4. 把 `openocd.cfg` 等自定义配置纳入版本管理，但保留调试器相关的本地差异。

---

## 六、常见坑（均来自本项目真实经历）

1. **构建目录不存在**：报 `.../build/Debug is not a directory` → 先 `cmake --preset Debug` 配置，再 build。
2. **Linux 大小写敏感**：`#include "pid_old.h"` 找不到实际文件 `PID_old.h`；`#include "freertos.h"` 找不到 `FreeRTOS.h`。Windows/Keil 下不区分大小写所以能过，Linux 下必须严格一致。
3. **`undefined reference`**：链接阶段才暴露，例如 `taskEX15IT_CRITICAL()` 是 `taskEXIT_CRITICAL()` 的笔误。这类错误不是编译器问题，是符号定义缺失或拼写不一致。
4. **头文件找不到但文件确实存在**：检查是否把目录加进了 `target_include_directories`，CubeMX 生成的 include 路径不会自动包含用户自己的 `MyLib/`、`MyTask/`。
5. **改了源码但没重新编译**：确认改动的文件是否被列进了某个 `target_sources()`；CMake 只会编译它知道的源文件。
6. **裸机链接报错 "undefined reference to main" 之类**：检查链接脚本和启动文件是否参与构建（本项目启动文件在 `cmake/stm32cubemx/CMakeLists.txt` 的 `MX_Application_Src` 列表里）。
7. **修改反复丢失**：如果 `CMakeLists.txt`、头文件的修复在 `git checkout` / `git restore` / `git stash` 后消失，说明这些改动还没提交。及时 `git add` + `git commit` 才能固化下来。

---

## 七、速查表

| 想做的事 | 命令 |
|---------|------|
| 首次配置 Debug | `cmake --preset Debug` |
| 编译 Debug | `cmake --build --preset Debug` |
| 编译 Release | `cmake --preset Release && cmake --build --preset Release` |
| 只编译某个目标 | `cmake --build --preset Debug --target Dog_lower` |
| 清理 | `cmake --build --preset Debug --target clean` |
| 看详细编译命令 | `cmake --build --preset Debug --verbose` |
| 查看构建目录内容 | `ls build/Debug` |
| 烧录（配置 OpenOCD 后） | `openocd -f interface/stlink-v2.cfg -f target/stm32h7x.cfg -c "program build/Debug/Dog_lower.elf verify reset exit"` |
| GDB 调试 | `arm-none-eabi-gdb build/Debug/Dog_lower.elf` + `target extended-remote :3333` |
