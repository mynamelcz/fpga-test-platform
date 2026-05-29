# 配置文件 Schema 规范

## 章节定位

本章是平台四层配置文件的**权威 Schema 定义与校验规则**。

[[02-总体系统架构]]、[[04-扩展板与适配体系]]、[[07-上位机软件设计]]、[[05-MCU模拟外设测试方案]]、[[06-流片前FPGA原型验证方案]] 中都出现了 YAML 配置示例，但**字段名、层级、单位、ID 格式各不相同**，互相对不齐。这对一个把"配置驱动复用"作为核心架构的平台是致命的——上位机、FPGA 固件、AI 用例生成三方若各写各的，复用就无从谈起。

本章统一这一契约。它服务于：

- [[07-上位机软件设计]]：配置加载、资源映射、安全校验、AI 接口。
- [[03-核心FPGA主板设计]]：寄存器/资源与配置的对应。
- [[10-Pattern引擎时序规格]] / [[12-Capture引擎时序规格]]：Time-Set、Capture 窗口的配置承载。

> 纪律不变（[[09-风险与边界]]）：`安全性 > 架构复用性 > 可验证闭环 > 扩展余量 > 单点功能丰富度 > 成本极限优化`。Schema 的首要目标是**架构复用性与安全可校验**，因此约定从严：单位显式、ID 规范、安全字段必填。

---

## 一、四层配置文件总览

| 文件 | 描述对象 | 变化频率 | 来源/权威 |
|---|---|---|---|
| `core_board.yaml` | 核心板固定资源 | 低（随核心板版本） | 厂方随核心板发布 |
| `extension_board.yaml` | 扩展板能力与映射 | 中（随扩展板型号） | 随扩展板发布，板载 EEPROM ID 交叉校验 |
| `dut.yaml` | 待测芯片定义 | 高（每颗/每封装） | 用户/项目维护 |
| `test_plan.yaml` | 测试流程定义 | 高（每测试场景） | 用户/AI 生成 |

层级关系（与 [[02-总体系统架构]] 四层架构对应）：

```text
core_board.yaml      ← 核心板提供什么资源
  ↑ 引用
extension_board.yaml ← 扩展板把核心板资源映射/转换成什么
  ↑ 引用
dut.yaml             ← DUT 需要哪些引脚/电源/接口
  ↑ 引用
test_plan.yaml       ← 用上述资源跑什么流程
```

> 文件名统一为**全小写 + 下划线**：`core_board.yaml` / `extension_board.yaml` / `dut.yaml` / `test_plan.yaml`。
> 修订提示：[[02-总体系统架构]] 中写作 `DUT.yaml`（大写），应统一为 `dut.yaml`。

---

## 二、通用约定（消除现存不一致的根因）

现存示例的不一致主要来自单位、ID、命名三处。**全平台强制以下约定：**

### 2.1 单位约定（显式单位后缀，禁止裸数字）

现存冲突示例：[[07-上位机软件设计]] 写 `range: [1.2, 5.0]`、`current_limit: 0.5`（裸数字，单位靠猜）；[[04-扩展板与适配体系]] 写 `range: 0.8-3.6V`、`current_limit: 500mA`（字符串带单位）。两者都不利于机器校验。

统一规则：**数值字段用裸数字，单位由字段名后缀显式声明**：

| 物理量 | 字段后缀 | 示例 |
|---|---|---|
| 电压 | `_v` | `voltage_v: 3.3` |
| 电压范围 | `_v`（数组） | `range_v: [0.8, 3.6]` |
| 电流 | `_a` | `current_limit_a: 0.5` |
| 频率 | `_hz` / `_mhz` | `freq_mhz: 10` |
| 时间 | `_ns` / `_us` / `_ms` | `strobe_ns: 15.0` |
| 容量 | `_mb` / `_kb` | `size_mb: 512` |
| 容差(电压) | `_mv` | `tolerance_mv: 30` |
| 容差(LSB) | `_lsb` | `tolerance_lsb: 20` |

> 好处：单位无歧义、可机器校验、与 [[10-Pattern引擎时序规格]]（`_ns`）、[[05-MCU模拟外设测试方案]]（`tolerance_mv`/`tolerance_lsb`）已有用法一致。

### 2.2 ID 与命名约定

现存冲突：[[07-上位机软件设计]] 扩展板用 `model: digital_adapter_v1` + `id: 0x1001`；[[04-扩展板与适配体系]] 用 `board_id: digital_adapter_v1` + `board_type: digital_test_adapter`。`board_id` 一会儿是字符串名一会儿是 hex 数字，混乱。

统一规则——**区分三个不同概念，各有专属字段**：

| 概念 | 字段 | 类型 | 示例 | 用途 |
|---|---|---|---|---|
| 人类可读型号名 | `model` | string | `digital_adapter_v1` | 显示、文档 |
| 板类型分类 | `board_type` | enum string | `digital_test_adapter` | 上位机选模板 |
| 机器/EEPROM 板卡 ID | `board_id` | hex | `0x1001` | 板载 EEPROM 烧录值，交叉校验 |
| 硬件版本 | `hardware_version` | string | `1.0` | 版本管理 |

> 关键：`board_id`（hex，EEPROM 中的值）与 `model`（字符串名）是**两个字段**，不可混用。上位机以 `board_id` 为准（板载 EEPROM 读出），并与 `model`/`board_type` 交叉校验（见第七节）。

### 2.3 资源命名约定

沿用 [[02-总体系统架构]] / [[07-上位机软件设计]] 已有的点分命名，固化为规范：

```text
io.<bank>.<ch>        例：io.bank_a.ch0
power.<name>          例：power.dut_vdd
clock.<name>          例：clock.dut_ref
trigger.<dir><n>      例：trigger.in0 / trigger.out0
<proto><n>            例：spi0 / i2c0 / uart0 / pwm0
analog.<type><n>      例：analog.dac0 / analog.adc0
capture.<group>       例：capture.group0
```

### 2.4 安全默认态枚举

统一取值（[[04-扩展板与适配体系]] 已用，固化）：

```text
default_io_state:    hiz | input | output_low | output_high   （默认 hiz）
default_power_state: off | on                                  （默认 off）
```

---

## 三、`core_board.yaml` Schema

描述核心板固定资源（对应 [[03-核心FPGA主板设计]]、[[11-FPGA选型与IO预算]]）。

```yaml
core_board:
  model: core_fpga_v1            # string, 必填
  board_id: 0x0001               # hex, 必填（EEPROM）
  hardware_version: "1.0"        # string, 必填
  firmware_version: "0.1.0"      # string, 必填（运行时回读交叉校验）
  fpga:                          # 必填
    part: TBD                    # string, 选型冻结后填（见 [[11-FPGA选型与IO预算]]）
    user_io_total: 400           # int, 目标 ≥400
  io_groups:                     # 必填, list
    - name: bank_a               # string
      channels: 32               # int
      voltage_v: configurable    # number | "configurable"
      timing_class: high         # enum: high | mid | low（见 [[10-Pattern引擎时序规格]]）
  ddr:                           # 必填
    size_mb: 512                 # int, 起步示例值；深 Capture 目标 1024（见 [[13-带宽与成本预算]]）
    type: ddr3                   # enum: ddr3 | ddr4
  protocols: [spi0, i2c0, uart0, gpio0, pwm0]   # list, 必填
  clock_trigger:                 # 必填
    clock_out: [clock.out0]
    clock_in:  [clock.in0]
    trigger_in:  [trigger.in0]
    trigger_out: [trigger.out0]
    sync: [sync.bus0]
  uplink:                        # 必填
    primary: usb3                # enum: usb3 | gige（MVP 建议 usb3，见 [[13-带宽与成本预算]]）
    debug: [uart, jtag]
```

> 修订提示：[[07-上位机软件设计]] 原示例缺 `board_id`/`hardware_version`/`firmware_version`/`timing_class`，且 `ddr.size_mb` 未标注为示例值——应按本 Schema 补全。

---

## 四、`extension_board.yaml` Schema

描述扩展板能力与到核心板资源的映射（对应 [[04-扩展板与适配体系]]、[[05-MCU模拟外设测试方案]]、[[06-流片前FPGA原型验证方案]]）。

```yaml
extension_board:
  model: digital_adapter_v1            # string, 必填（人类可读）
  board_type: digital_test_adapter     # enum, 必填
                                       #   digital_test_adapter
                                       #   mcu_analog_adapter
                                       #   fpga_proto_adapter
                                       #   batch_fixture
  board_id: 0x1001                     # hex, 必填（EEPROM 烧录值）
  hardware_version: "1.0"              # string, 必填
  bom_version: "1.0"                   # string, 可选（见 [[04-扩展板与适配体系]] 版本管理）
  calibration_version: "1.0"           # string, 模拟板必填
  supported_dut: [qfn48, qfp64]        # list, 可选
  io_groups:                           # 必填
    - name: bank_a
      channels: 32
      voltage_v: 3.3                   # 显式电压（非 configurable，扩展板已定域）
      default_io_state: hiz            # 必填（安全）
  io_mapping:                          # 必填：核心板资源 → DUT 引脚
    io.bank_a.ch0: dut.SPI_CLK
    io.bank_a.ch1: dut.SPI_MOSI
  power_channels:                      # 必填
    - name: dut_vdd
      range_v: [0.8, 3.6]              # 数组, 显式单位后缀
      current_limit_a: 0.5             # 显式单位后缀
      default_power_state: off         # 必填（安全）
  analog:                              # 可选（仅模拟板，见 [[05-MCU模拟外设测试方案]]）
    dac: [analog.dac0, analog.dac1]
    adc: [analog.adc0, analog.adc1]
    switch_matrix: true
  clock:                               # 可选
    supports_clock_out: true
    supports_clock_in: true
  safety:                              # 必填
    default_io_state: hiz
    default_power_state: off
    overcurrent_protect: true
```

> 修订提示（统一 [[04]] 与 [[07]] 的两套写法）：
> - [[04-扩展板与适配体系]] 原示例用顶层 `board_id`（值是字符串名）——应改为 `model`（字符串）+ `board_id`（hex），并整体置于 `extension_board:` 顶层键下。
> - [[07-上位机软件设计]] 原示例 `range: [1.2, 5.0]`/`current_limit: 0.5` → `range_v`/`current_limit_a`；`id: 0x1001` → `board_id: 0x1001`。
> - [[04]] 原 `range: 0.8-3.6V`/`current_limit: 500mA`（字符串带单位）→ `range_v: [0.8, 3.6]`/`current_limit_a: 0.5`。

---

## 五、`dut.yaml` Schema

描述待测芯片（对应 [[07-上位机软件设计]]）。

```yaml
dut:
  name: demo_chip                # string, 必填
  version: "A0"                  # string, 可选（样片版本）
  package: BGA256                # string, 必填
  power:                         # 必填
    - name: vdd
      voltage_v: 3.3             # 显式单位
      max_current_a: 0.2
      power_seq: 1               # int, 上电顺序
    - name: iovdd
      voltage_v: 1.8
      max_current_a: 0.1
      power_seq: 2
  pins:                          # 必填
    - name: SPI_CLK
      type: input                # enum: input | output | bidir | analog | power | gnd
      voltage_v: 3.3
    - name: SPI_MISO
      type: output
      voltage_v: 3.3
  boot:                          # 可选
    reset_active: low            # enum: low | high
    boot_pins: {BOOT0: 0}
  interfaces: [spi0, i2c0]       # list, 可选
```

---

## 六、`test_plan.yaml` Schema

描述测试流程（对应 [[07-上位机软件设计]] 脚本引擎、[[10-Pattern引擎时序规格]] Time-Set、[[06-流片前FPGA原型验证方案]] 用例）。

```yaml
test_plan:
  name: smoke_test               # string, 必填
  dut: demo_chip                 # string, 必填（引用 dut.yaml）
  time_sets:                     # 可选（见 [[10-Pattern引擎时序规格]]）
    ts_nrz_50m:
      tcyc_ns: 20.0
      format: nrz
      drive_edge_ns: 0.0
      strobe_ns: 15.0
  steps:                         # 必填, 有序 list
    - power_on: dut_vdd
    - set_voltage: {rail: dut_vdd, voltage_v: 3.3}
    - set_current_limit: {rail: dut_vdd, current_limit_a: 0.2}
    - spi_write: {bus: spi0, addr: 0x00, data: 0x01}
    - spi_read:  {bus: spi0, addr: 0x04, expect: 0x55}
    - load_pattern: basic_io_test
    - run_pattern: {name: basic_io_test, time_set: ts_nrz_50m}
    - start_capture: {group: capture.group0, pre_ns: 1000, post_ns: 1000}
    - assert_range: {signal: analog.adc0, min_mv: 1600, max_mv: 1700}
  on_fail: {save_capture: true, power_off: true}   # 失败动作（安全）
  report: {format: [md, html, json]}
```

> 步骤动词与 [[07-上位机软件设计]] 脚本引擎基础动作保持一致（power_on/off、set_voltage、spi_*/i2c_*/uart_*、load/run_pattern、start_capture、assert_equal/range、save_capture）。

---

## 七、跨文件校验规则（安全校验器据此实现）

[[07-上位机软件设计]] 的安全校验器与 [[02-总体系统架构]] 的控制流要求"未识别扩展板不上电、未加载 DUT 配置不驱动 IO"。本节给出**机器可执行的交叉校验规则**：

| # | 规则 | 失败处理 |
|---|---|---|
| C1 | `extension_board.board_id` 必须等于板载 EEPROM 读出值 | 拒绝上电 |
| C2 | `firmware_version` 运行时回读须与 `core_board.yaml` 兼容 | 告警/拒绝 |
| C3 | `dut.power[*].voltage_v` 必须落在所映射 `extension_board.power_channels[*].range_v` 内 | 拒绝上电 |
| C4 | `dut.power[*].max_current_a` ≤ 对应 `current_limit_a` | 拒绝上电 |
| C5 | `dut.pins[*].voltage_v` 必须与映射到的 `io_groups.voltage_v` 电压域匹配 | 拒绝驱动 IO |
| C6 | `io_mapping` 引用的 `io.<bank>.<ch>` 必须存在于 `core_board.io_groups` | 配置错误 |
| C7 | `test_plan` 引用的 bus/pattern/capture/analog 资源必须在上游文件中声明 | 配置错误 |
| C8 | `time_sets[*]` 的 `*_ns` 必须落在 1 ns 细分栅格上（见 [[10-Pattern引擎时序规格]]） | 告警并对齐 |
| C9 | 所有 `default_io_state`/`default_power_state` 必须存在且为安全值 | 拒绝执行 |
| C10 | `run_pattern` 不得驱动 `dut.yaml` 中标记为 output/power/gnd 的引脚 | 拒绝驱动 |

> 这套规则把 [[09-风险与边界]] 的"安全优先控制原则"从口号变成可代码化的断言。**不通过校验，禁止上电/驱动**——软件不能绕过（呼应 [[02-总体系统架构]] 安全链路）。

---

## 八、版本与演进

- 每个配置文件建议带 `schema_version` 顶层字段（本规范为 `schema_version: 1`），便于上位机向后兼容。
- 字段**只增不改语义**：新增字段走可选，废弃字段保留过渡期。
- 单位后缀约定（第 2.1 节）是硬约束，未来新增物理量沿用同规则。

```yaml
schema_version: 1
core_board: { ... }
```

---

## 九、需同步修订的现存文档

本 Schema 落地后，以下示例应对齐（建议作为一次文档一致性修订）：

| 文档 | 需修订点 |
|---|---|
| [[02-总体系统架构]] | `DUT.yaml` → `dut.yaml`；配置示例补 `schema_version` |
| [[04-扩展板与适配体系]] | 顶层 `board_id`(字符串) → `model`+`board_id`(hex)；`range`/`current_limit` 加单位后缀并置于 `extension_board:` 下 |
| [[07-上位机软件设计]] | `id` → `board_id`；`range`/`current_limit` → `range_v`/`current_limit_a`；core_board 补必填字段 |
| [[05-MCU模拟外设测试方案]] | `vref`/`tolerance_*` 已基本合规，补 `voltage_v` 后缀统一 |
| [[06-流片前FPGA原型验证方案]] | `clock: 50MHz`/`freq: 10MHz` → `clock_mhz`/`freq_mhz` |

---

## 当前结论

```text
· 四层配置文件统一为：core_board / extension_board / dut / test_plan（全小写）
· 三大根因约定：单位显式后缀（_v/_a/_mhz/_ns…）、ID 三分（model/board_type/board_id）、点分资源名
· 安全字段（default_io_state/default_power_state）必填，缺则拒绝执行
· 10 条跨文件校验规则把"安全优先"变成可代码化断言
· schema_version 支撑向后兼容；现存文档示例按第九节对齐
```

这份 Schema 把散落在五份文档里、互相对不齐的 YAML 片段，收敛成一份机器可校验的契约，是上位机、FPGA 固件、AI 用例生成三方协作的地基。下一步建议补 **寄存器映射文档**（[[08-MVP实施计划]] 阶段 4 已列为交付物但缺失），把配置契约进一步落到固件与驱动的寄存器级接口上。
