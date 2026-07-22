# 2026_AT_DogLower

河北科技大学 四足机器人 STM32H723 下位机代码。

MCU: STM32H723VGT6 (Cortex-M7, 550 MHz, 1 MB Flash, 560 KB RAM)

---

## 工具链要求

| 工具 | 版本 | 安装方式 (Ubuntu) |
|------|------|-------------------|
| CMake | >= 3.22 | `sudo apt install cmake` |
| Ninja | >= 1.10 | `sudo apt install ninja-build` |
| arm-none-eabi-gcc | >= 10.3 | `sudo apt install gcc-arm-none-eabi` |

验证安装：

```bash
cmake --version       # >= 3.22
ninja --version       # >= 1.10
arm-none-eabi-gcc --version  # >= 10.3
```

## 构建

项目使用 CMake Presets，提供 Debug 和 Release 两种配置。

```bash
# 配置 + 编译 (Debug)
cmake --preset Debug
cmake --build build/Debug

# 配置 + 编译 (Release)
cmake --preset Release
cmake --build build/Release
```

编译完成后，`build/<preset>/` 目录下会生成：

| 文件 | 说明 |
|------|------|
| `Dog_lower.elf` | ELF 格式固件（含调试符号，用于 GDB 调试） |
| `Dog_lower.bin` | 二进制固件（用于烧录） |
| `Dog_lower.hex` | Intel HEX 格式固件 |
| `Dog_lower.map` | 内存映射文件 |

### 清理构建

```bash
rm -rf build/Debug      # 删除 Debug 构建
rm -rf build/Release    # 删除 Release 构建
rm -rf build/           # 删除全部构建
```

## 烧录

使用 OpenOCD + J-Link (SWD) 烧录。项目根目录已包含 `openocd.cfg` 配置文件。

### 方式一：CMake flash 目标（推荐）

```bash
cmake --preset Debug && cmake --build build/Debug
cmake --build build/Debug --target flash
```

该命令会自动编译并烧录 `Dog_lower.elf`，含 verify + reset。

### 方式二：直接调用 OpenOCD

```bash
# 烧录 .elf (推荐，含断点信息)
openocd -f openocd.cfg -c "program build/Debug/Dog_lower.elf verify reset exit"

# 烧录 .bin (需指定起始地址)
openocd -f openocd.cfg -c "program build/Debug/Dog_lower.bin 0x08000000 verify reset exit"
```

### 方式三：STM32CubeProgrammer

```bash
STM32_Programmer_CLI -c port=swd -w build/Debug/Dog_lower.bin 0x08000000 -v -rst
```

### `openocd.cfg` 说明

```tcl
source [find interface/jlink.cfg]    # J-Link 调试器
transport select swd                  # SWD 接口
source [find target/stm32h7x.cfg]     # STM32H7 目标
adapter speed 4000                    # 4 MHz SWD 速率
reset_config srst_only srst_nogate    # 复位配置
```

## 调试 (VS Code)

1. 安装扩展：`Cortex-Debug`
2. 项目根目录已有 `.vscode/` 配置（或自行创建 `launch.json`）：

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug (OpenOCD)",
            "type": "cortex-debug",
            "request": "launch",
            "servertype": "openocd",
            "cwd": "${workspaceFolder}",
            "executable": "${workspaceFolder}/build/Debug/Dog_lower.elf",
            "configFiles": [
                "openocd.cfg"
            ],
            "runToEntryPoint": "main"
        }
    ]
}
```

## 项目结构

```
26_AT_DOG_LOWER/
├── CMakeLists.txt              # 顶层 CMake 文件（用户源码在此添加）
├── CMakePresets.json            # CMake 预设（Debug / Release）
├── STM32H723XG_FLASH.ld        # GCC 链接脚本
├── Dog_lower.ioc                # STM32CubeMX 工程文件
├── startup_stm32h723xx.s        # 启动文件 (GCC)
│
├── cmake/
│   ├── gcc-arm-none-eabi.cmake  # ARM GCC 工具链配置
│   ├── starm-clang.cmake        # ST ARM Clang 工具链配置（可选）
│   └── stm32cubemx/
│       └── CMakeLists.txt       # CubeMX 自动生成的源码/库（勿手动改）
│
├── Core/                       # CubeMX 生成的应用核心代码
│   ├── Inc/                    #   头文件 (main.h, FreeRTOSConfig.h ...)
│   └── Src/                    #   源文件 (main.c, freertos.c, gpio.c ...)
│
├── Drivers/                    # HAL 驱动 + CMSIS
│   ├── CMSIS/
│   └── STM32H7xx_HAL_Driver/
│
├── Middlewares/                 # 中间件
│   ├── ST/STM32_USB_Device_Library/
│   └── Third_Party/FreeRTOS/
│
├── USB_DEVICE/                  # USB Device 应用层 (CDC)
│
├── MyLib/                      # ★ 用户库（电机驱动、通信、滤波等）
│   ├── FDCANDriver.{c,h}       #   FDCAN 总线驱动
│   ├── motor.{c,h} / motorEx.{c,h}  # 电机控制
│   ├── dm_h6215.{c,h}          #   达妙电机驱动
│   ├── go_motor.{c,h}          #   GO 电机驱动
│   ├── 485_bus.{c,h}           #   RS485 总线
│   ├── JY61.{c,h}              #   IMU 驱动
│   ├── kalman.{c,h}            #   卡尔曼滤波
│   ├── bezier.{c,h}            #   贝塞尔曲线
│   ├── usb_trans.{c,h}         #   USB 通信
│   └── ...
│
├── MyTask/                     # ★ 用户任务（FreeRTOS 任务入口）
│   ├── app_main.{c,h}          #   应用主任务
│   ├── run.{c,h}               #   运行控制
│   └── ws2812.{c,h}            #   WS2812 灯带控制
│
├── MDK-ARM/                    # 旧 MDK-ARM 工程（已弃用，可删除）
└── build/                      # 构建输出（gitignore）
```

## 开发指南

### 添加新的源文件

在顶层 `CMakeLists.txt` 的 `target_sources` 和 `target_include_directories` 中添加：

```cmake
target_sources(${CMAKE_PROJECT_NAME} PRIVATE
    MyLib/new_driver.c        # 新增源文件
)

target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE
    MyLib                     # 如果新增了头文件目录
)
```

然后重新编译即可，无需重新运行 `cmake --preset`（Ninja 会自动检测 CMakeLists.txt 变化）。

### STM32CubeMX 重新生成代码

1. 打开 `Dog_lower.ioc`，修改外设配置
2. 在 Project Manager → Toolchain/IDE 中选择 **CMake**
3. 生成代码（会更新 `cmake/stm32cubemx/CMakeLists.txt` 和 `Core/` 下的文件）
4. **CubeMX 会覆盖 `CMakeLists.txt` 顶层文件**，生成后需重新添加 `MyLib`/`MyTask` 源码和 post-build 命令
5. 重新配置并编译：`cmake --preset Debug && cmake --build build/Debug`

> **注意**：CubeMX 生成的代码块以 `/* USER CODE BEGIN */` 和 `/* USER CODE END */` 标记，在这些标记之间的用户代码不会被覆盖。

### 从 MDK-ARM 迁移注意事项

本项目原为 MDK-ARM (Keil) 工程，迁移到 CMake + GCC 后需注意：

1. **文件名大小写**：Linux 文件系统区分大小写，`#include "freertos.h"` 必须改为 `#include "FreeRTOS.h"`。已修复的文件：
   - `MyLib/motorEx.h`: `pid_old.h` → `PID_old.h`
   - `MyTask/app_main.c`: `freertos.h` → `FreeRTOS.h`

2. **编译器差异**：GCC 比 ARMCC/ARMClang 更严格，可能暴露更多 warning（如指针类型不匹配）。可逐步修复或使用 `-Wno-<warning>` 临时忽略。

3. **链接脚本**：MDK 使用 `.sct` 文件，GCC 使用 `.ld` 文件。项目根目录下的 `STM32H723XG_FLASH.ld` 是 GCC 链接脚本，由 CubeMX 生成。

4. **启动文件**：MDK 使用 `MDK-ARM/startup_stm32h723xx.s`，GCC 使用根目录的 `startup_stm32h723xx.s`。两者格式不同，不可混用。

5. **MDK-ARM 目录**：旧的 MDK 工程目录已不再使用，可在确认无需回退后删除。

## 编译器优化等级

| 配置 | CFLAGS | 适用场景 |
|------|--------|----------|
| Debug | `-O0 -g3` | 开发调试，无优化，完整调试信息 |
| Release | `-Os -g0` | 发布固件，最小体积优化 |
