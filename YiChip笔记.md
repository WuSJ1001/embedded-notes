# YiChip BL ASM 精简笔记

## 使用策略
- 先锁定寄存器数据来源：`fetch/fetcht`→`pdata/temp`→逻辑/算术→`store` 类。
- 条件判断优先用标志位组合：`blank` 判空、`true` 判逻辑、`positive` 判符号。
- 循环与分支牢记配套寄存器：`loopcnt`、`queue`、`mark`、`user*`。
- 任何写硬件寄存器的指令（`hjam/hfetch/hstore`）都会连带修改 `contw`，操作前确认目标地址。

## 常用寄存器速查
- `pdata`：RW 64b；所有 `fetch/ifetch` 默认落点；`blank` 依据其值。
- `temp`：RW 64b；配合 `fetcht` 存放辅助数据。
- `rega/regb/regc/regab`：RW；常规指针/中转寄存器，`ifetch` 时解释为地址。
- `alarm`：RW；定时触发相关指针，`ifetch` 会把其值当地址。
- `regext`：RW；扩展指针，常与 `regext_index` 配合读取表项。
- `queue`：RW；位操作偏移寄存器，驱动 `qset*`、`qisolate*`。
- `contr/contw/contru/contwu`：RW；读写指针族，`fetch/store/istore` 自动更新。
- `loopcnt`：RW；`loop` 自减计数，需预设循环次数。
- `mark`：RW；位标记缓存，供 `bmark*`/`rtnmark*` 判断。
- `clkn/clke` 及 `_bt/_rt`：时钟寄存器；`deposit` 取值后写入内存做时间戳。

## 标志位速查
- `blank`：`pdata` 为 0 或空置 1；常配 `rtn blank`、`branch ...,blank`。
- `positive`：算术结果 ≥0 置 1；与 `sub/isub` 配合判断大小。
- `true`：逻辑/比较命中置 1；由 `compare/icompare/isolate` 等更新。
- `zero`：计数归零使用，多与 `branch ... ,zero`、`ncall ...,zero`。
- `user/user2/user3`：可编程工作流标记，适合流程切换。
- `match/timeout/sync/le`：协议状态标志，作为高层流程判据。

## 指令族速查
- **逻辑**：`and/and_into`（立即数遮罩）、`iand/iand_into`（`pdata` 与寄存器）、`ior/ixor`、`or/or_into`、`nop`（可填充等待）。
- **算术**：`add/iadd`、`increase/pincrease`（快速 ±1）、`sub/isub`、`mul32/imul32`、`div/idiv`+`quotient/remainder`（记得 `call wait_div_end`）、`random`。
- **位操作**：`lshift*/rshift*`（按位移）、`setflag/qsetflag/nsetflag/nqsetflag`（控制寄存器某 bit）、`isolate*/qisolate*`（抽取指定位到 `true`）、`setflip/set0/set1/qset0/qset1`、`byteswap/reverse/invert`、`compare/icompare`（掩码比较）。
- **寻址与数据搬运**：`jam/hjam`（写内存/硬件寄存器）、`fetch/fetcht`、`ifetch/ifetcht`（间接取数会更新指针并影响 `blank`）、`hfetch/hfetcht`（0x8000 起硬件窗口）、`store/storet`、`istore/istoret`、`hstore/hstoret`、`force/iforce`、`arg/setarg`（设置立即数）、`copy/icopy`、`deposit`（把寄存器送回 `pdata`）、`disable/enable`（标志位 0/1）、`parse`（截取 bit 串至 `pwindow` 等）。
- **跳转与流程控制**：`call/ncall`＋`rtn/nrtn`（函数式跳转）、`rtn*`/`rtnbit*`/`rtnmark*`（条件返回）、`branch/nbranch`（传统跳转）、`bbit*/bmark*`（按位跳转）、`beq/bne`（立即数判断）、`loop`（基于 `loopcnt`）、`bpatch`（热补丁位控制）。

## 常见代码骨架
- **条件返回**：`fetch` → 自动更新 `pdata`/`blank` → `rtn blank` 快速退出。
- **多分支**：`iadd` 或 `compare` 处理后连用多条 `beq`；位判断使用 `bbit*/bmark*`。
- **循环搬运**：预设 `loopcnt`、地址寄存器与队列指针，循环体通常 `ifetch`→`istore`→`loop label`。
- **硬件访问套路**：`hfetch`/`hstore` 前确认地址落在 0x8000+；操作后 `contw` 被改写，若后续还要写普通 SRAM，需重置 `contw`。
- **位图设置**：写入偏移用 `arg <bit>,queue`，随后 `qset1/qset0` 或 `qisolate*` 处理目标寄存器。

## 复用提示
- 与 C 代码互查时关注 `.format` 文件：变量尺寸需匹配指令的 `num_bytes`。
- 任何 `ifetch/istore` 族指令都会改变 `contr/contw`，嵌套操作前后应保存或重置地址寄存器。
- 线上调试常见 bug：忘记 `call wait_div_end`、错误复用 `queue`、`loopcnt` 未预设、硬件寄存器地址写错导致 `contw` 残留。
