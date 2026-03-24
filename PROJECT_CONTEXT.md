# PROJECT_CONTEXT

## 1. 项目整体结构（从 `TEST/program/peripherals.prog` 视角）
- **指令补丁入口 (`patch.prog`)**：针对芯片固件的 Hook，集中在 `p_my_feature` 等标签处接入自定义逻辑。
- **外设驱动集 (`peripherals.prog`)**：提供 GPIO、I²C、SPI、PWM、ADC、WDT、NEC 等模块的底层流程，由宏汇编构成，直接操控 `core_*` 与 `mem_*` 寄存器区。
- **OTP/Loadcode 管线**：负责从 OTP、EEPROM、Flash 加载固件块，期间会配置 SPI/I²C GPIO 并进行 CRC 检查。
- **LPM（低功耗管理）与 Wakeup**：封装在 `gpio_set_before_lpm`、`lpm_write_gpio_wakeup` 等逻辑，睡眠前快照 GPIO 配置并设置唤醒源。

## 2. 各模块职责概览

| 模块 | 关键职责 | 依赖/关联 |
| --- | --- | --- |
| **GPIO** | 地址计算 (`gpio_addr`)、功能复用 (`gpio_config_function_int`)、输入/输出配置、唤醒控制 (`mem_gpio_wakeup_cfg`) | 大量被 I²C/SPI/PWM/AC 检测等模块调用 |
| **I²C (IICD)** | EEPROM/GPIO 初始化 (`iicd_load_gpio_init`)、主机时序参数 (`iic_init_600khz`)、WP 引脚保护 (`iicd_eeprom_write_enable`) | GPIO 用于 SDA/SCL/WP 复用；loadcode 依赖 I²C 读取固件 |
| **SPI (SPID)** | Flash/寄存器读写 (`spid_write_reg`)、Flash 初始化 (`spid_init_flash`)、GPIO 绑定 (`spi_gpio_init`) | loadcode SPI 路径、twspi 驱动 |
| **PWM** | 时钟选择 (`pwm_clk_set`)、通道/引脚一体配置 (`pwm_gpio_select`)、同步模式 | 通过 `gpio_config_param` 选择 PWM 输出脚 |
| **ADC** | SADC 校准、GPIO/VDCDC 采样、参考电压切换 | GPIO 需配置为模拟输入或输出 GND |
| **WDT/Timer** | 看门狗使能/禁用 (`wdt_set_enable`)、延时/计时器配置 | 用于系统可靠性与键盘扫描节奏 |
| **Low Power** | `gpio_set_before_lpm` 统一切换上拉，`gpio_set_wake*` 配置唤醒条件，`p_lpm_save_context` 备份 GPIO 配置 | 与 `mem_saved_gpio`、`core_gpio_wakeup_cfg` 共享数据 |
| **用户逻辑 (例：p_my_feature)** | 利用上述 API 实现具体业务（LED、按键等） | 依赖 GPIO API 与轮询循环 |

## 3. 核心设计思想

1. **ACTIVE_BIT 机制**  
   - `temp`/GPIO 号的 bit7 表示“逻辑 active”极性；低电平有效则 bit7=1。所有输入、输出、唤醒函数都按该位自动翻转电平，无需每处都记住实际拉高/拉低。
2. **配置与行为分层**  
   - 先通过 `gpio_addr`/`gpio_config_function_int` 确定寄存器地址与复用，再调用 `gpio_out`、`gpio_config_input` 之类写入行为位 (上拉、方向、激活状态)。
3. **统一的唤醒表 (`mem_gpio_wakeup_cfg`)**  
   - 每个 GPIO 占半字节，记录唤醒极性/上下拉。`get_gpio_wakeup_index` 负责定位 nibble，使任何函数都能无冲突更新。
4. **低功耗预处理**  
   - 进入 LPM 前统一调用 `gpio_set_before_lpm` 等函数把关键管脚拉到可预测状态，并把唤醒配置镜像到 `core_gpio_wakeup_cfg`。
5. **模块自洽**  
   - I²C/SPI/PWM/AC 等模块均通过 GPIO API 复用同一套机制，不直接硬编码寄存器，确保服务端函数彼此兼容、易于组合。

> 记忆提示：只要看到 ACTIVE_BIT/`gpio_*` 函数，就回想到这一整套“GPIO 地址计算 + 极性抽象 + 唤醒表”的设计；所有外设初始化都会从这里起步。
