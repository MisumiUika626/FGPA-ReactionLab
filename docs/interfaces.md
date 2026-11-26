# ReactionLab 公共接口规范

> 基础版本候选规范，双方确认后生效。公共接口不得擅自修改，模块内部实现自行决定。

## 1. 分工与公共约定

- A：`rtl/core/reaction_core.v`，负责状态机、等待、计时、犯规、超时及 LED；负责 `rtl/top.v` 集成。
- B：`rtl/peripheral/reaction_peripheral.v`，负责按键、历史、统计、查询及显示驱动。
- `clk`：统一系统时钟，上升沿更新；`rst_n`：同步低有效全局复位，优先级最高。
- `CLK_FREQ`：单位 Hz，正整数且可被 1000 整除，两个模块参数一致。50 MHz 仅为开发默认值，实际频率待确认。
- 全局复位清空全部数据，回到 IDLE 和 TEST；Clear 只清除当前测试并回到 TEST，保留历史与统计。

## 2. 公共通信信号

| 信号 | 位宽 | 方向 | 约定 |
|---|---:|---|---|
| `start_pulse` | 1 | B → A | TEST 模式且 IDLE 时开始测试 |
| `stop_pulse` | 1 | B → A | TEST 模式下的停止事件 |
| `clear_pulse` | 1 | B → A | 任意模式下清除当前测试 |
| `test_state` | 3 | A → B | 当前状态，持续有效 |
| `reaction_time` | 10 | A → B | 无符号整数，0～999 ms |
| `save_pulse` | 1 | A → B | 有效成绩保存事件 |

所有 `*_pulse` 均为高有效、持续一个系统时钟周期的脉冲，接收端在上升沿采样，不得将其用作时钟。Core 进入 RESULT 时同时更新最终成绩并置 `save_pulse=1`，下一上升沿 Peripheral 保存一次，Core 撤销脉冲；脉冲有效期间成绩保持稳定。

## 3. Core 状态与行为

| 编码 | 状态 | 计时输出 | LED | TEST 显示 |
|---|---|---|---|---|
| 000 | IDLE | 0 | 灭 | 000 |
| 001 | WAIT | 0 | 灭 | 全灭 |
| 010 | TIMING | 当前时间 | 亮 | 实时计时 |
| 011 | RESULT | 最终成绩 | 灭 | 最终成绩 |
| 100 | FOUL | 0 | 灭 | Err（近似段码） |
| 101 | TIMEOUT | 999 | 灭 | 999 |

- IDLE：Start 进入 WAIT；单独 Stop 无效，Start 与 Stop 同时有效不开始。
- WAIT：等待 1～6 s，结束时在同一上升沿点亮 LED、启动计时并进入 TIMING；提前 Stop 判犯规，若与等待结束同沿，优先犯规。
- TIMING：从亮灯起每满 1 ms 加 1。Stop 时锁定已完成毫秒数（含该沿刚满的毫秒，允许 0）；达到 999 ms 优先超时，包括 Stop 与截止同沿的情况。
- 仅进入 RESULT 时产生一次 `save_pulse`；犯规、超时、中断不保存。
- RESULT、FOUL、TIMEOUT 保持状态与结果直到 Clear，不响应 Start/Stop；110、111 保留，非法状态回 IDLE。
- 任意状态 Clear 优先于测试事件，回 IDLE；复位优先于 Clear。

## 4. Peripheral 行为

- `key_*` 为高电平按下的逻辑输入，B 负责同步、消抖；板级适配负责实际极性。长按不得重复触发。
- Stop 捕获同步后的首次按下边沿，锁定抖动，稳定释放后重新使能；不附加固定数十毫秒的确认延迟。计时以 Core 采样 Stop 为准。Stop 未重新使能或与 Start 同时按下时，不发送 Start。
- Mode 循环切换 TEST → HISTORY → MIN → MAX → TEST；WAIT、TIMING 及 Start 已发出但尚未被采样时禁止切换。
- TEST：Start/Stop 用于测试；HISTORY：Start 查看较新、Stop 查看较旧；MIN/MAX：Start/Stop 无效。查询模式不向 Core 发送 Start/Stop。
- Clear 在任何模式有效并返回 TEST。非测试进行期间按键优先级为 Clear > Mode > 其他；测试进行期间忽略 Mode，仍处理 Stop。模式提示方式待硬件确认。
- 保存最近 10 次有效成绩，写满覆盖最旧。进入 HISTORY 默认查看最新记录，索引 0 为最新，两端饱和，无记录显示 `---`。
- MIN/MAX 统计全局复位以来的所有有效成绩，无记录显示 `---`；仅在上升沿采样到 `rst_n=1` 且 `save_pulse=1` 时保存并更新统计，即使同沿 Clear 也要接收已发出的保存事件。

## 5. Verilog 端口合同

```verilog
module reaction_core #(parameter CLK_FREQ = 50_000_000)(
    input wire clk, rst_n, start_pulse, stop_pulse, clear_pulse,
    output wire [2:0] test_state,
    output wire [9:0] reaction_time,
    output wire save_pulse, stimulus_led
);
endmodule

module reaction_peripheral #(parameter CLK_FREQ = 50_000_000)(
    input wire clk, rst_n, key_start, key_stop, key_clear, key_mode,
    output wire start_pulse, stop_pulse, clear_pulse,
    input wire [2:0] test_state,
    input wire [9:0] reaction_time,
    input wire save_pulse,
    output wire [20:0] display_segments
);
endmodule
```

以上仅为端口合同，输出可按实现改为 `reg`。LED 与段码均逻辑高有效；`display_segments` 每位按 `{a,b,c,d,e,f,g}` 排列，[20:14] 百位、[13:7] 十位、[6:0] 个位，全灭 `0000000`、横线 `0000001`、数字 0 `1111110`。

实际时钟、复位来源、按键数量、段线/位选/扫描方式、I/O 极性与引脚待确认；B 提供显示驱动，A 顶层连接并提供符合时序的复位（异步物理输入须适当同步）。硬件配置以 `docs/design.md` 为准，不得猜测；按键不足时双方另定映射。

## 6. 验收与变更

- A：验证正常、犯规、超时、Clear 优先级、同沿冲突、保存脉冲与数据稳定性。
- B：验证单次保存、十条记录及覆盖、空记录、统计、Clear 保留历史、模式隔离与长按。
- 集成：验证完整测试与查询流程、异常及按键冲突；未执行的测试不得标记 PASS。
- 平均值、评级、双人竞技暂不纳入接口。基础功能通过后，经双方确认方可扩展，同步更新接口文档、实现及测试。
