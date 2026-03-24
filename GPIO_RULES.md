# GPIO_RULES

## 1. 编码与寄存器映射
- **GPIO 号范围**：0–39。低 16 个存放在 `core_gpio_conf`，其余在 `core_gpio_conf_hi`。
- **地址计算**：`gpio_addr` 将 `temp`（GPIO 编号）与 `GPIO_NUMBER` 比较后确定使用哪个表，再将编号乘以 2 得到字节对齐地址并写入 `contw`。
- **打包参数 (`gpio_config_param`)**：`pdata[15:8]=功能码`，`pdata[7:0]=GPIO`。拆分后传递给 `gpio_config_function_int`，实现“外设 -> GPIO”一体映射。

## 2. ACTIVE_BIT 机制
- **定义**：GPIO 编号最高位（bit7）被称为 `GPIO_ACTIVE_BIT`，表示该 GPIO 的逻辑激活电平。
  - `bit7=0`：高电平为激活（Active High）。
  - `bit7=1`：低电平为激活（Active Low）。
- **作用**：
  1. `gpio_config_input`、`gpio_out`、`gpio_set_wake` 等函数会读取该位，自动在写寄存器前翻转电平或设置合适的上下拉。
  2. `gpio_get_bit` 会根据 ACTIVE_BIT 决定是否反相，从而让返回值始终代表“逻辑 active”状态。
- **使用方式**：构造参数时可用 `temp = (GPIO_ACTIVE_BIT ? 0x80 : 0x00) | gpio_number`；例如 `0x95 = 0x80 | 0x15` 表示 GPIO21 且低电平有效。

## 3. Active / Inactive 行为规则
- **输出设置**：
  - 调用 `gpio_out` 时 `temp.bit7`=期望逻辑状态（1=激活态、0=非激活态）。
  - 函数内部自动写 `gpcfg_output_high` 并根据 ACTIVE_BIT 决定最终电平。
  - `gpio_out_active`/`gpio_out_inactive` 仅负责设定 ACTIVE_BIT，实际电平由随后 `gpio_out` 控制。
- **状态检测**：
  - `gpio_check_active` 读取当前配置，若输出正处于激活态则 True=1。
  - 输入路径 (`gpio_get_bit`) 同样返回逻辑态，便于直接用 `nbranch ..., true` 判断“是否触发”。

## 4. 唤醒配置规则
- **Wake Table**：`mem_gpio_wakeup_cfg` 中每字节对应两个 GPIO（偶数=高 nibble，奇数=低 nibble）。`get_gpio_wakeup_index` 提供 `contw` 和 `alarm` 掩码。
- **Nibble 值语义**（常见约定）：
  - `0x0`：禁用唤醒。
  - `0x1`：低电平唤醒 + 需上拉 (`gpio_set_low_pullup`)。
  - `0x2`：单纯低电平唤醒。
  - `0x4`：高电平唤醒。
- **配置步骤**：
  1. 通过 `gpio_set_wake` 或 `gpio_set_low_pullup`，在写 nibble 前调用 `get_gpio_wakeup_index`，确保不会覆盖相邻 GPIO。
  2. `gpio_set_wake_by_current_state` 可先读取实时电平 (`gpio_get_bit`)，再写入合适的 nibble，防止极性错误。
  3. 进入低功耗前，`gpio_set_before_lpm` 会统一拉高关键 GPIO，随后 `p_lpm_write_gpio_wakeup` 把 `mem_gpio_wakeup_cfg` 镜像到硬件寄存器 `core_gpio_wakeup_cfg`。

## 5. 快速记忆
1. **“0x80 | gpio” 决定逻辑极性**：任何涉及 `temp`/GPIO 的 API 都遵循该编码。
2. **“函数不直接写电平，而是写配置值”**：输出控制通过写 `gpcfg_*` 组合实现，上下拉、方向、复用全部在同一字节完成。
3. **“唤醒依赖 nibble 表”**：修改唤醒条件一定要先取得 nibble 掩码，避免破坏邻居。
4. **“低功耗前统一上拉”**：`gpio_set_before_lpm` 是睡眠准备的最后一道默认防线。

> 将这些规则与 `peripherals.prog` 的函数名对应起来，就能在未来快速回忆整个 GPIO 体系的使用方式。只要记得 ACTIVE_BIT + nibble wake 表，即可复现我们这次的理解。
