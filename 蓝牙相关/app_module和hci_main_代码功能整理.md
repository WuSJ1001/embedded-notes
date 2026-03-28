# app_module.prog 与 hci_main.prog 代码理解与功能整理

> 说明
>
> - 本文基于工程内现有代码行为进行梳理，不修改任何代码。
> - 重点围绕：这两个文件各自“在系统里扮演什么角色”、如何使用 UART、以及两者的差异与注意事项。
> - 代码定位以文件路径 + 行号为准（行号可能随文件版本变化而略有偏移）。

---

## 1. 背景概念：HCI 是什么

HCI（Host Controller Interface，主机控制器接口）可以理解为蓝牙系统中 **Host（主机：MCU/PC）** 与 **Controller（控制器：蓝牙芯片）** 之间的一套标准命令/事件通信协议。

在 UART 承载上常见两种形态：

- **H4**：最常见的 UART HCI 承载格式。包头第 1 字节通常是 `HciType`（例如 `0x01` 代表 HCI Command）。
- **H5**：在 H4 之上增加了可靠传输/滑窗等机制（工程中常见 `h5` 命名的缓冲区，暗示可能走 H5）。

在本工程里，`app_module.prog` 有明确的“命令格式”注释，并检查首字节 `HciType==0x01`（见 `D:\YiChip_workspace\Stack\program\app_module.prog:163`），说明 UART 上跑的是“类似 HCI 的帧格式/协议”。

---

## 2. app_module.prog 是做什么的

### 2.1 总体定位（模块模式/业务桥接）

`app_module.prog` 更像“**模块模式（Module）**”的上层逻辑：把蓝牙协议栈的事件与数据，通过一条 UART（HCI-like 格式）对接到外部主机（通常是 MCU）。

它的典型特征是：

- 有较多 **业务状态**（连接/断开/配对等）处理；
- 有 **低功耗（LPM）** 相关的 wake/lock 管理；
- UART 既用于 **接收主机命令**，也用于 **向主机上报事件/数据**；
- 在 `module_init` 中把各种回调函数写入 `mem_cb_*`，让系统在不同场景回调到模块逻辑（见 `D:\YiChip_workspace\Stack\program\app_module.prog:4`）。

### 2.2 初始化流程要点

#### (1) 回调注册

`module_init` 会设置：

- idle 回调：`mem_cb_idle_process`（见 `D:\YiChip_workspace\Stack\program\app_module.prog:7`）
- BB（基带/协议栈事件）回调：`mem_cb_bb_event_process`（见 `D:\YiChip_workspace\Stack\program\app_module.prog:9`）
- wakelock 检查回调：`mem_cb_check_wakelock`（见 `D:\YiChip_workspace\Stack\program\app_module.prog:11`）
- BLE 发送/ATT 写入等回调：`mem_cb_ble_transmit`、`mem_cb_att_write` 等（见 `D:\YiChip_workspace\Stack\program\app_module.prog:13`）

这些回调决定了“系统在何时进入模块逻辑”，而不是像 `hci_main.prog` 那样自己死循环轮询。

#### (2) 串口初始化（LPM 场景）

`module_lpm_uart_init` 是 `app_module` 中 UART 初始化的核心入口（见 `D:\YiChip_workspace\Stack\program\app_module.prog:37`）。

它的大致步骤：

1. **先关闭 UART 使能位**（避免配置过程中乱收发）
2. **配置 UART DMA/环形缓冲区**（`uarta_init_dma_mem`）
3. **配置波特率**：从 `mem_module_uarta_baud_rate` 取“已经算好的配置值”，直接写 `core_uart_baud`
4. **选择 UART 主时钟源**：`uart_clock_select_main_freq_crystal`
5. **配置 GPIO 复用**：用固定宏 `HCI_UART_*_GPIO_NUM`，直接 `jam` 到 `core_gpio_conf+GPIO号`
6. **使能 UART**：写 `core_uart_ctrl`，并按 `mem_module_flag` 的标志位选择是否打开硬件流控

> 关键点：这里的引脚选择是“宏固定”的（例如 `HCI_UART_TX_GPIO_NUM`），因此它倾向于“模块默认脚位”。

### 2.3 UART 收包与协议处理

`module_process_idle` 会进入 `module_process_check_hci_command_complete`（见 `D:\YiChip_workspace\Stack\program\app_module.prog:88`）。

`module_process_check_hci_command_complete` 做的事情可以概括为：

- 先检查 `core_uart_status` 的 RX 是否为空（`UART_STATUS_RX_FIFO_EMPTY`）
- `uarta_prepare_rx` 获取当前 RX 读指针
- 读取首字节 `HciType`，要求为 `0x01`，否则认为异常并走异常处理/丢弃路径
- 根据长度判断是否收齐完整命令帧，否则等待更多字节
- 收齐后分发到 `module_hci_cmd_control` 做 opcode 级别的命令处理

这意味着：如果系统在跑 `app_module` 模式，你从外部发“随便的字符串/裸数据”，很可能会被当成非法包而丢弃或触发异常流程。

### 2.4 BLE 事件上报（通过 UART 发回主机）

`module_process_bb_event` 会根据不同 `BT_EVT_*` 分发并触发 `module_hci_event_*`（见 `D:\YiChip_workspace\Stack\program\app_module.prog:94`），典型包括：

- 连接/断开
- 配对成功/失败
- PHY 更新
- 需要输入 passkey 等

其发送路径常见模式是：

- `module_hci_prepare_tx` 组织包头/长度/opcode
- `uart_copy_tx_bytes_fast` 搬运 payload（如有）
- `uarta_send_register_pop` 提交 TX 写指针并发送

---

## 3. hci_main.prog 是做什么的

### 3.1 总体定位（纯 HCI 主循环/调试入口）

`hci_main.prog` 更像“**HCI 模式的主程序**”：上电后初始化 UART，然后进入死循环，不断从 UART 收命令、解析、回事件。

从开头结构就非常明显（见 `D:\YiChip_workspace\Stack\program\hci_main.prog:5`）：

- `hci_init`：清栈、关 WDT、配置时钟、配置 UART 默认参数、初始化 UART、初始化 PWM
- `hci_process_loop`：不停调用 `hci_process_check_uart_rx` 轮询收包

### 3.2 UART 引脚默认值与可配置性

`hci_init_uart_default_config_*` 这一组函数会：

- 如果 `mem_hci_uart_tx_gpio` 等变量当前为空，就写入默认的 `HCI_UART_*_GPIO_NUM`
- 这让 UART 引脚具备“可在运行前通过内存变量改写”的可能（见 `D:\YiChip_workspace\Stack\program\hci_main.prog:25`）

### 3.3 串口初始化（hci_init_uart_config）

`hci_init_uart_config`（见 `D:\YiChip_workspace\Stack\program\hci_main.prog:54`）做的事情可概括为：

1. 组织 `mem_h5rx_buf/_end`、`mem_h5tx_buf/_end` 形成 DMA 缓冲区配置块
2. 调 `uarta_init_dma_mem` 写入 `core_uart_*` 并打开 UART 时钟
3. 选 UART 主时钟：`uart_clock_select_main_freq_crystal`
4. 配波特率：使用十进制 `115200` 调 `uarta_calc_baud_rate_config` 计算并写入 `core_uart_baud`
5. 配 GPIO 复用：从 `mem_hci_uart_*_gpio` 读取 GPIO 号，调用 `gpio_config_function_int` 配成 UART TXD/RXD/RTS/CTS
6. 使能 UART：写 `core_uart_ctrl`

> 关键点：这里的引脚选择来自 `mem_hci_uart_*_gpio`，不是写死宏；因此更“可变”。

### 3.4 主循环收包解析

`hci_process_check_uart_rx` 会：

- 查看 `core_uart_rxitems` 是否达到最小长度（示例里先判断是否至少有 4 字节）
- `uarta_prepare_rx` 取 RX 指针
- 读取 `HciType` 并分支处理（示例里主要处理 CMD）
- 判断是否收齐完整包，否则等待
- 收齐后进入 `hci_parse_complete_packet` / vendor 分组等处理
- 必要时丢弃当前包并推进 RX 指针（`uarta_rxdone`）

这就是“纯 HCI 控制器”模式的典型收包流程。

---

## 4. 两者的相同点与不同点（重点：UART 配置）

### 4.1 相同点

- 两者最终都在操作同一套 UARTA 相关底层能力（`uarta_init_dma_mem`、`core_uart_baud`、`core_uart_ctrl`、`core_uart_rxitems` 等）。
- 都会选择 UART 主时钟（crystal 路径）。
- 都会配置 TX/RX/RTS/CTS 的 GPIO 复用，并使能 UART。

### 4.2 不同点（非常关键）

#### (1) 波特率配置方式不同

- `module_lpm_uart_init`：从 `mem_module_uarta_baud_rate` 取“已经算好的配置值”，直接写 `core_uart_baud`（见 `D:\YiChip_workspace\Stack\program\app_module.prog:42`）。
- `hci_init_uart_config`：用十进制 `115200` 调 `uarta_calc_baud_rate_config` 计算后写（见 `D:\YiChip_workspace\Stack\program\hci_main.prog:66`）。

#### (2) 引脚配置方式不同

结合 `format\hci.format` 的默认定义：

- `HCI_UART_TX_GPIO_NUM=0x07`
- `HCI_UART_RX_GPIO_NUM=0x06`
- `HCI_UART_RTS_GPIO_NUM=0x09`
- `HCI_UART_CTS_GPIO_NUM=0x0a`  
（见 `D:\YiChip_workspace\Stack\format\hci.format:31`）

`module_lpm_uart_init`：

- 直接把这些宏作为索引，写 `core_gpio_conf+GPIO号`（固定脚位倾向）

`hci_init_uart_config`：

- 先把默认宏写入 `mem_hci_uart_*_gpio`（如未设置）
- 再从 `mem_hci_uart_*_gpio` 取 GPIO 号去配置（可通过内存变量改变）

#### (3) 硬件流控开关策略不同

- `module_lpm_uart_init` 会根据 `mem_module_flag` 的 `MODULE_FLAG_UART_FLOW_CONTROL` 决定是否置位 `BIT_UART_CONTROL_FLOW_CONTROL`（见 `D:\YiChip_workspace\Stack\program\app_module.prog:51`）。
- `hci_init_uart_config` 在该函数片段中未见类似“按标志位开关流控位”的逻辑（至少在这段代码中没有）。

---

## 5. “全映射（所有引脚都能当串口脚）”在工程里的体现与注意事项

### 5.1 道理（PinMux / 交换矩阵）

所谓“全映射”，本质是芯片内部存在 **引脚复用矩阵**：UART 的 TXD/RXD/RTS/CTS 并不是固定绑在某几个脚上，而是可以通过 GPIO 配置寄存器把“UART 信号线”路由到任意 GPIO PAD。

在本工程里，“把某个 GPIO 变成 UART 引脚”的动作体现为：

- 对 `core_gpio_conf+GPIO号` 写入 `gpcfg_uart_txd/rxd/rts/cts` 等功能配置值  
（见 `D:\YiChip_workspace\Stack\format\regs.format:512`）

并且工程中确实出现过将 UART 配到不同 GPIO 的例子：

- `D:\YiChip_workspace\Stack\program\sim.prog:629` 使用过 `core_gpio_conf+16`
- `D:\YiChip_workspace\Stack\program\mesh_protocol_stack\mesh_chip_peripherals.prog:359` 使用过 `core_gpio_conf+11/12`

### 5.2 注意事项（全映射不等于随便选）

- **封装是否引出**：有些 GPIO 编号可能未引出到封装/板级不可用。
- **启动/下载/Flash 等关键脚**：如果某些脚在上电早期用于启动选择、调试、外部存储等，改成 UART 可能导致不可启动或冲突。
- **电气与上拉**：RX 常见会配置 `pullup`；若外部电路已有强上拉/下拉，注意不要对抗。
- **流控一致性**：开了 RTS/CTS，就必须把线接全、对端也开启，否则容易“卡住不发/不收”。
- **协议占用**：若 UART 被 `app_module` 的 HCI-like 协议栈占用，发送裸数据可能被当成非法包丢弃。

---

## 6. 如何快速判断当前系统“到底跑哪种模式”

可用的判断思路：

- 代码编译开关：`app_module.prog` 顶部有 `ifdef COMPILE_MODULE`（见 `D:\YiChip_workspace\Stack\program\app_module.prog:2`）。若该宏启用，模块模式相关逻辑更可能生效。
- 主入口差异：
  - `hci_main.prog`：`hci_init` 后进入 `hci_process_loop` 的死循环（见 `D:\YiChip_workspace\Stack\program\hci_main.prog:17`）。
  - `app_module.prog`：通过 `mem_cb_*` 回调被动进入各种处理函数，不表现为单一死循环。

---

## 7. 总结

- `app_module.prog`：偏“蓝牙模块业务层”，把 BLE 事件/状态与 UART（HCI-like）命令/事件桥接，并且包含低功耗/唤醒等逻辑。
- `hci_main.prog`：偏“纯 HCI 控制器模式/调试模式”，初始化 UART 后进入轮询收包解析的主循环。
- 两者底层都使用 UARTA 的同一套寄存器与 `uarta_*` 例程，但引脚选择与波特率配置方式存在差异，且存在相互覆盖的可能（谁后执行谁生效）。

