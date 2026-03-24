# API_INDEX（GPIO 相关函数速查）

> 约定：输入参数按照常见寄存器命名（temp/pdata/rega…）。副作用描述写明主要寄存器或内存区。

## 基础寻址 / 复用

### gpio_addr / gpio_addr_next / gpio_addr_ext
- **功能**：根据 GPIO 号计算 `core_gpio_conf` 或 `_hi` 中的配置地址。
- **输入**：`temp`=GPIO 编号（0-39）。
- **副作用**：`contw` 指向目标寄存器；读取 `core_gpio_conf(_hi)`。

### gpio_config_param
- **功能**：分解打包参数并调用 `gpio_config_function_int` 设置复用。
- **输入**：`pdata` 高 8 位为功能码，低 8 位为 GPIO；`temp` 临时保存 GPIO。
- **副作用**：写 `core_gpio_conf(_hi)`。

### gpio_config_function / gpio_config_function_int (+next/+ext)
- **功能**：把 GPIO 绑定到指定功能（输出、高阻、外设复用等）。
- **输入**：`temp`=功能码，`pdata`=GPIO（bit7=ACTIVE_BIT）。
- **副作用**：写 `core_gpio_conf(_hi)`。

### gpio_get_config (+next/+ext)
- **功能**：读取目标 GPIO 当前配置。
- **输入**：`temp`=GPIO 编号。
- **副作用**：返回值放入 `contr`，读取 `core_gpio_conf(_hi)`。

## 输入 / 输出行为

### gpio_config_input / gpio_config_input_without_wake
- **功能**：把 GPIO 设为输入，可选连接唤醒逻辑。
- **输入**：`temp`=GPIO，bit7 表示低/高有效。
- **副作用**：`gpio_config_input` 会调用 `gpio_set_wake`；均写 `core_gpio_conf(_hi)`。

### gpio_config_input_nowake
- **功能**：无唤醒输入配置。
- **输入**：同上。
- **副作用**：调用 `gpio_clr_wake`，再写 `core_gpio_conf(_hi)`。

### gpio_get_bit / gpio_get_bit_reverse
- **功能**：读取 GPIO 电平并按 ACTIVE_BIT 输出逻辑态。
- **输入**：`temp`=GPIO；`GPIO_ACTIVE_BIT` 决定是否反相。
- **副作用**：读 `core_gpio_in`，更新 `pdata` 与 Zero/True 标志。

### gpio_out / gpio_out_active / gpio_out_inactive / gpio_out_flag
- **功能**：驱动 GPIO 输出高/低并设置极性。
- **输入**：`temp` bit7=目标电平，bit6-0=GPIO。
- **副作用**：写 `core_gpio_conf(_hi)`。

### gpio_check_active / gpio_check_active_high
- **功能**：检测当前输出是否处于“激活”电平。
- **输入**：`temp`=GPIO。
- **副作用**：读取 `core_gpio_conf(_hi)`，更新 Zero/True。

### gpio_set_analog / gpio_set_high_impedance / gpio_write
- **功能**：分别写入 `gpcfg_no_ie`、`gpcfg_high_impedance` 或任意值。
- **输入**：`temp`=GPIO（前两者），`pdata`=最终配置。
- **副作用**：写 `core_gpio_conf(_hi)`。

## 唤醒与上下拉

### get_gpio_wakeup_index
- **功能**：根据 GPIO 编号取得 `mem_gpio_wakeup_cfg` 的字节和半字节掩码。
- **输入**：`temp`=GPIO（低 5 位有效）。
- **副作用**：`contw` 指向 `mem_gpio_wakeup_cfg`，`alarm` 保存掩码。

### gpio_set_wake / gpio_set_wake_high / gpio_set_wake_low4bit
- **功能**：启用 GPIO 唤醒并设置极性。
- **输入**：`temp`=GPIO，bit7=0 表示低有效，bit7=1 表示高有效；`debug`=写入 nibble 的数值。
- **副作用**：写 `mem_gpio_wakeup_cfg` 相应 nibble。

### gpio_set_low_pullup / gpio_set_low_pullup_low4bit
- **功能**：低电平唤醒并自动配置上拉。
- **输入**：`temp`=GPIO。
- **副作用**：写 `mem_gpio_wakeup_cfg` nibble=1。

### gpio_clr_wake
- **功能**：禁用指定 GPIO 唤醒。
- **输入**：`temp`=GPIO。
- **副作用**：清除 `mem_gpio_wakeup_cfg` nibble。

### gpio_set_wake_by_current_state
- **功能**：读取实时电平并自动匹配唤醒极性。
- **输入**：`temp`=GPIO。
- **副作用**：调用 `gpio_get_bit`，再写 `mem_gpio_wakeup_cfg`。

## 低功耗准备

### gpio_set_before_lpm / gpio_set_before_lpm_ext
- **功能**：进入 LPM 前扫描所有 GPIO，将指定功能改为上拉。
- **输入**：循环内部使用 `loopcnt` 遍历。
- **副作用**：写 `core_gpio_conf`、`core_gpio_conf_hi`。

### setgpio_pullup / setgpio_pulldown
- **功能**：辅助函数，把当前 `contw` 指向的 GPIO 写上拉或下拉。
- **输入**：无显式外部参数，使用全局状态。
- **副作用**：写 `core_gpio_conf(_hi)`。

## 其他 GPIO 相关辅助

### gpio_set_analog
- **功能**：配置 GPIO 为模拟（禁用输入 buffer）。
- **输入**：`temp`=GPIO。
- **副作用**：写 `core_gpio_conf(_hi)`。

### gpio_config_param（PWM/AC 等）
- **功能**：允许外设通过打包参数配置引脚。
- **输入**：同上。
- **副作用**：写 `core_gpio_conf(_hi)`。

> 若需更具体的调用示例，可参考 `spi_gpio_init`、`iicd_init_pin_scl_sda`、`pwm_gpio_select_process` 等模块，它们均严格遵循上述 API 约定。
