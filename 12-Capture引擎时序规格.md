# Capture 引擎时序规格（V1）

## 章节定位

本章是 [[10-Pattern引擎时序规格]] 的对偶文档，定义核心 FPGA 主板上 **Capture 引擎的第一版（V1）时序与数据规格**。

[[03-核心FPGA主板设计]] 已给出 Capture 的功能轮廓（多通道同步采样、Trigger 前后窗口、实时比较、Fail 地址记录、DDR 写入、上位机回读），但与 Pattern 一样**缺少量化时序指标和数据格式契约**。本章补齐这一半，使 [[01-需求与产品定位]] 提出的核心能力——

> 失败前后窗口 Capture、可观测性（IO 波形 / 协议交易 / 电源 / Trigger 前后窗口 / 失败上下文）、问题可复现

——成为可验收的工程目标。

> 设计纪律不变（[[09-风险与边界]]）：`安全性 > 架构复用性 > 可验证闭环 > 扩展余量 > 单点功能丰富度 > 成本极限优化`。Capture 的 V1 取舍原则是：**先保证"采得准、对得齐、回得来、复现得了"，再谈采样深度与高级分析。**

---

## 一、Capture 与 Pattern 的关系

二者是同一数字引擎的两面，必须**同源同步**，否则采样点相对激励漂移会导致误判。

```text
         共用 fine 时钟域 + 共用 Trigger（见 [[03-核心FPGA主板设计]] 时钟系统）
                              │
        ┌─────────────────────┴─────────────────────┐
   Pattern 引擎                                  Capture 引擎
   按 Time-Set 放置 drive 边沿                按 Time-Set 放置 compare strobe
        │                                          │
        └──────→ DUT ←───── 响应 ─────→ 采样/比较 ──┘
```

- **相位关系来自同一个 Time-Set**：drive edge 与 compare strobe 在同一 Time-Set 内定义（见 [[10-Pattern引擎时序规格]] 第六节），保证可复现。
- **两种采样口径**（V1 都要支持）：
  - **同步比较采样（functional compare）**：在 Time-Set 指定的 strobe 时刻采一次，与 expected 比较 → PASS/FAIL。对应 Pattern 向量测试。
  - **波形记录采样（waveform capture）**：按固定采样时钟连续记录通道电平，用于观测、协议交易、失败窗口。对应"逻辑分析仪"用途。

---

## 二、术语

| 术语 | 含义 |
|---|---|
| **Compare Strobe** | 同步比较采样时刻，落在 [[10-Pattern引擎时序规格]] 的 1 ns 细分栅格上 |
| **Sample Clock (Fsamp)** | 波形记录模式的采样时钟 |
| **Capture Depth** | 单次可记录的采样点数（按通道） |
| **Pre/Post Trigger Window** | 触发点前/后保留的采样窗口 |
| **Fail Capture** | 比较失败时冻结现场并保存上下文 |
| **Sample Skew** | 名义同时刻的采样，在不同通道实际采样点的时间差 |

---

## 三、采样架构（两条路径）

### 路径 1：同步比较采样（functional compare）

复用 [[10-Pattern引擎时序规格]] 模式 B 的 ISERDES 过采样：在 strobe 子位索引处取样，逐通道与 `expected[N-1:0]` 在 `compare_mask` 下比较。

- 分辨率：1 ns 细分栅格（strobe 可放在任意子位）。
- 输出：逐向量 PASS/FAIL + 首个失败地址/通道（实时比较，硬件完成）。
- 用途：功能向量、协议正常时序判定。

### 路径 2：波形记录采样（waveform capture）

按 `Fsamp` 连续采样多通道电平，写入 BRAM 或 DDR ring buffer。

- 用途：IO 波形、协议交易记录、Trigger 前后窗口、失败现场。
- 不做逐点 expected 比较，原始记录回放给上位机解码。

> 两条路径可**同时启用**：比较路径判 PASS/FAIL，记录路径同步抓波形，失败时记录路径冻结窗口 → 这就是 Fail Capture 的核心机制。

---

## 四、V1 时序与容量指标（目标值）

> ⚠️ 同 [[10-Pattern引擎时序规格]]：以下为目标值，须在 [[08-MVP实施计划]] 阶段 3 bring-up 实测冻结。容量项强依赖 FPGA BRAM 与 DDR 选型（见 [[11-FPGA选型与IO预算]]），并受回读带宽约束（建议见后续 `13-带宽与成本预算`）。

### 4.1 采样时序

| 指标 | V1 目标 | 说明 |
|---|---|---|
| 同步比较 strobe 分辨率 | 1.0 ns（与 drive 共栅格） | ISERDES 过采样 |
| 波形采样率 Fsamp（BRAM 模式） | ≥ 200~250 MSa/s（目标） | 受 FPGA IO/SERDES 限 |
| 波形采样率 Fsamp（高速子集 stretch） | 更高，依赖过采样倍数 | 仅重点通道 |
| 采样通道数 | 与 Pattern 通道一致（160~192 用 / 256 预留） | 见 [[11-FPGA选型与IO预算]] |
| 通道间采样 skew（校准后） | < ±0.5 ns（目标） | 与 [[10-Pattern引擎时序规格]] 共用 deskew 校准 |
| compare 与 capture 相位对齐 | 同 Time-Set / 同时钟域，相位确定 | |

### 4.2 采样深度（两档）

| 模式 | 深度量级 | 用途 | 限制 |
|---|---|---|---|
| BRAM 模式 | 浅（数 K~数十 K 采样点/通道，依 BRAM 量） | 短窗口、低延迟、Trigger 前后窗口 | FPGA BRAM/URAM 容量 |
| DDR 模式 | 深（受 DDR 容量限） | 长向量回放比对、长时间回归、大失败窗口 | DDR 带宽 + 回读带宽 |

> V1 实现节奏（呼应 [[09-风险与边界]]"分阶段启用 DDR"）：**先打通 BRAM 模式 Capture，再扩展 DDR ring buffer**。bring-up 顺序见 [[03-核心FPGA主板设计]]（先 BRAM Capture 再 DDR Capture）。

### 4.3 触发与窗口

| 指标 | V1 目标 |
|---|---|
| Trigger 源 | 内部（比较失败/Pattern 事件）、外部 Trigger In、上位机软触发 |
| 触发模式 | 边沿触发、电平/掩码匹配触发、比较失败触发 |
| Pre-Trigger 窗口 | 可配（受 buffer 深度限） |
| Post-Trigger 窗口 | 可配 |
| 触发后动作 | 冻结窗口 / 继续滚动 / 停止 Pattern |

---

## 五、Fail Capture（平台区别于普通开发板的关键能力）

承接 [[03-核心FPGA主板设计]] 的 Fail Capture 要求，给出 V1 必须记录的字段契约：

```text
Fail Record（失败现场）:
  test_item_id        测试项 ID
  pattern_addr        失败时 Pattern 地址/向量号
  fail_channels[]     失败通道掩码
  expected / actual   期望值 / 实际采样值
  timestamp_fine      fine 栅格时间戳（与 Pattern 同源）
  pre_post_window     失败前后波形窗口（指向 buffer 区间）
  power_state         DUT 电源/电流快照（来自管理 MCU，见 [[03-核心FPGA主板设计]]）
  protocol_context    最近协议交易上下文（若适用）
```

- 时间戳以 **fine 栅格**为基准 → 上位机可与 Pattern 激励精确对齐回放。
- 与测试脚本版本、DUT 编号、扩展板版本、固件版本绑定（见 [[02-总体系统架构]] Capture 数据流），保证可复现、可回归。

---

## 六、数据打包格式（回读契约）

Capture 数据要能被上位机无歧义解析，并导出标准格式（[[07-上位机软件设计]] Capture 管理依赖此契约）。

### 6.1 板上原始打包

```text
Capture Frame:
  header:
    capture_id
    mode            (compare / waveform / fail)
    channel_count
    sample_rate_or_strobe
    start_timestamp_fine
    sample_format   (bit-packed N 通道/采样点)
  payload:
    bit-packed 采样数据（每采样点 N bit，按通道顺序）
  footer:
    sample_count
    overflow_flag   (buffer 溢出/丢点标志)
    crc
```

- **必须有 `overflow_flag`**：当采样率×通道数超过 DDR 写入或回读带宽时如实标注丢点，绝不静默截断（呼应 [[09-风险与边界]]"不能静默丢数据"的工程诚实原则）。

### 6.2 上位机导出

| 格式 | 用途 |
|---|---|
| VCD | 波形查看器/逻辑分析仪工具链 |
| CSV | 通用分析、脚本处理 |
| JSON | 原始元数据 + Fail Record |

---

## 七、与 Capture 相关的 skew 与对齐

- **采样 skew 与 drive skew 同源同治**：复用 [[10-Pattern引擎时序规格]] 模式 C 的 IDELAY 通道去偏斜校准，采样侧用 IDELAY 对齐到位。校准后通道间采样 skew 目标 < ±0.5 ns（FPGA→连接器段）。
- **端到端口径同样区分**（重要）：DUT 输出经扩展板电平转换器回到 FPGA，其传播延迟差是采样侧的校准盲区。报告须区分"平台采样 skew"与"DUT 真实时序"，避免误判 DUT（与 [[10-Pattern引擎时序规格]] 4.4、第八节一致，也与 [[05-MCU模拟外设测试方案]] 的误差区分纪律一致）。

---

## 八、V1 边界（做 / 不做）

### V1 做

- 同步比较采样（1 ns strobe）+ 波形记录采样（≥200 MSa/s 目标）
- BRAM 模式 Capture（先）+ DDR ring buffer（后）
- 边沿/掩码/失败触发 + Pre/Post 窗口
- Fail Capture 全字段记录 + fine 时间戳对齐
- overflow_flag 丢点标注
- VCD/CSV/JSON 导出
- 采样侧 deskew 校准（复用 Pattern 校准）

### V1 不做（推迟 V2+）

- 板上实时协议解码（V1 回读后由上位机解码 SPI/I2C/UART）
- 高速串行/眼图采样（明确不在范围，见 [[09-风险与边界]]）
- 板上复杂触发序列机（多级状态触发）
- 板上波形压缩/降采样智能筛选
- 实时流式无限深 Capture（V1 受 buffer + 带宽限）

---

## 九、对 FPGA / DDR / 带宽的反向要求（输出给下游）

| 能力 | 要求 | 去向 |
|---|---|---|
| ISERDES 过采样 | ≥ 8:1，与 Pattern 共用 | [[11-FPGA选型与IO预算]]（已列 P0） |
| BRAM/URAM 容量 | 满足 BRAM 模式浅 Capture + Fail 窗口 | [[11-FPGA选型与IO预算]] |
| DDR 写入带宽 | ≥ 采样率×通道数×位宽（同时承载 Pattern 回放） | 需量化 → 建议 `13-带宽与成本预算` |
| 上行回读带宽 | 决定深 Capture 可用性（GigE ~100 MB/s 可能成瓶颈） | 需量化 → 建议 `13-带宽与成本预算` |

> 关键提醒：**Capture 最终的"实测墙"在带宽**。160~256 通道 × ≥200 MSa/s 的原始数据率，会迅速超过 GigE/USB3 的回读能力。V1 必须靠 **触发窗口 + 掩码 + 按需 Capture**（不无脑全量采集）来匹配带宽——这条直接需要一份量化预算坐实，是下一份文档的核心任务。

---

## 十、验收测试（对接 MVP 阶段 6）

为 [[08-MVP实施计划]] 补充 Capture 专项验收项：

| 验收项 | 方法 | 通过标准 |
|---|---|---|
| 同步比较 | 已知激励→strobe 采样比对 | PASS/FAIL 判定正确，失败地址准确 |
| 波形记录 | 已知波形环回采样 | 回读波形与输入一致 |
| 采样 skew（校准后） | 多通道同时沿采样 | < ±0.5 ns（FPGA→连接器段） |
| Pre/Post 窗口 | 触发后检查窗口完整 | 前后窗口数据完整、对齐 |
| Fail Capture | 注入失败并回读现场 | 全字段记录 + 时间戳可与激励对齐 |
| overflow 标注 | 故意超带宽采集 | overflow_flag 正确置位，不静默丢点 |
| 导出 | 导出 VCD/CSV/JSON | 第三方工具可正确打开 |

---

## 当前结论

```text
Capture V1 与 Pattern 同源同步：
  · 同步比较（1 ns strobe，判 PASS/FAIL）+ 波形记录（≥200 MSa/s）
  · BRAM 先、DDR 后，分阶段启用
  · Fail Capture 全字段 + fine 时间戳，问题可精确复现
  · overflow_flag 如实标丢点，端到端 skew 分口径标注
  · 实测墙在带宽——必须靠触发/掩码/按需采集匹配，需量化预算坐实
```

至此数字链路的两半（Pattern 激励 + Capture 采集）规格成对完成，并与 [[11-FPGA选型与IO预算]] 形成闭环。三份文档共同指向同一个下游缺口——**带宽与成本量化预算**，这是验证平台"实测墙"与"低成本"两个核心卖点的最后一块拼图。
