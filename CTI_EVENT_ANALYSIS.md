# CTI输入输出事件详细对比分析

## CTI概述

**CTI (Cross Trigger Interface)** 是ARM CoreSight调试架构中的关键组件，用于在不同调试组件之间进行事件同步和触发信号传递。

### CTI在调试系统中的位置
```
┌─────────────────────────────────────────────────────────────┐
│                     CoreSight Debug System                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  CPU Core ◄──► Debug Logic ◄──► CTI ◄──► External Trigger│
│                    │              │                         │
│                    │              ├──► ETM/PTM             │
│                    │              ├──► Breakpoint Unit     │
│                    │              └──► Other CTI           │
└─────────────────────────────────────────────────────────────┘
```

---

## 一、CTI事件架构

### 1.1 事件流向
```
外部触发源 → TRIGIN[n] → Channel Matrix → TRIGOUT[m] → 目标组件
    ↑                          ↓
    └────── 事件处理逻辑 ────────┘
```

### 1.2 CTI寄存器基地址示例（来自工作区配置）
| CPU/Domain | CTI Base Address | 配置文件 |
|-----------|-----------------|---------|
| AON M55 | `0xE0042000` | start-AON-M55.cmm |
| R82AE Core0 | `0x580CA000` | start-SI-RPU-R82AE.cmm |
| R82AE Core1 | `0x580D6000` | start-SI-RPU-R82AE.cmm |
| A78AE Core0 | `0x44020000` | EVB-Attach/SI_Jtag/start-APU-A78AE-Linux.cmm |
| SI HSM | `0x57300000+CTI_offset` | PLD-Attach/real-ddr/jtag_si_hsm.cmm |
| CVC DSP | `0x2f010000` | PLD-Attach/real-ddr/jtag_cvc_dsp.cmm |

---

## 二、CTI输入事件 (TRIGIN)

### 2.1 输入事件定义
**TRIGIN (Trigger Input)** 是CTI接收的外部触发信号，用于将外部事件引入到CTI通道矩阵中。

### 2.2 TRIGIN特性对比表

| 特性维度 | 详细说明 |
|---------|---------|
| **数量** | 通常8个输入通道：TRIGIN[7:0] |
| **信号方向** | 外部 → CTI（输入） |
| **信号类型** | 数字脉冲信号、电平信号 |
| **触发源** | CPU调试事件、外部引脚、其他CTI、ETM、Watchpoint等 |
| **响应延迟** | 通常1-2个时钟周期 |
| **编程控制** | 通过CTIINEN[n]寄存器映射到内部通道 |

### 2.3 TRIGIN事件来源分类

#### 2.3.1 CPU核心调试事件
```
TRIGIN来源            信号含义                    典型用途
─────────────────────────────────────────────────────────
TRIGIN[0]         → Debug Halt Request         外部断点触发
TRIGIN[1]         → Breakpoint Hit             断点命中事件
TRIGIN[2]         → Watchpoint Hit             数据观察点命中
TRIGIN[3]         → External Debug Request     外部调试请求
TRIGIN[4]         → PMU Overflow               性能计数器溢出
TRIGIN[5]         → ETM Trigger                ETM追踪触发
TRIGIN[6]         → Custom Event               自定义事件
TRIGIN[7]         → System Event               系统级事件
```

#### 2.3.2 Multi-Core调试链路
```c
// 典型场景：Core0触发导致所有Core停止
Core0_CTI (TRIGOUT) ──→ Core1_CTI (TRIGIN[0])
                   ──→ Core2_CTI (TRIGIN[0])
                   ──→ Core3_CTI (TRIGIN[0])
```

### 2.4 TRIGIN寄存器配置

#### CTIINEN[n] - 输入使能寄存器
- **地址偏移**: `0x020 + (n × 4)` (n=0-7)
- **功能**: 将TRIGIN[n]映射到内部通道[0-3]

```cmm
; 示例：将TRIGIN[0]映射到Channel 0
&cti_base=0x580CA000
Data.Set AXI:(&cti_base+0x020) %Long 0x1  ; CTIINEN0 = bit[0] = Channel 0

; 将TRIGIN[1]映射到Channel 1和Channel 2
Data.Set AXI:(&cti_base+0x024) %Long 0x6  ; CTIINEN1 = bit[1]|bit[2]
```

#### 寄存器位域映射
```
CTIINEN[n] Register (32-bit)
┌───────────────────────────────────────┐
│31            4   3   2   1   0        │
├─────────────────┬───┬───┬───┬─────────┤
│   Reserved      │CH3│CH2│CH1│CH0      │
└─────────────────┴───┴───┴───┴─────────┘
  写1：使能TRIGIN[n]映射到对应Channel
  写0：禁用映射
```

### 2.5 TRIGIN信号时序

```
外部事件       ─┐    ┌───────┐
                └────┘       └─────  (脉冲或电平)

TRIGIN[n]      ─┐    ┌───────┐
   (采样)       └────┘       └─────  (同步到CTI时钟)

Channel激活    ──┐   ┌──────┐
   (内部)         └───┘      └─────  (根据CTIINEN映射)

处理延迟：     ←─→ 1-2 CLK cycles
```

---

## 三、CTI输出事件 (TRIGOUT)

### 3.1 输出事件定义
**TRIGOUT (Trigger Output)** 是CTI发送给其他调试组件的触发信号，由内部通道矩阵驱动。

### 3.2 TRIGOUT特性对比表

| 特性维度 | 详细说明 |
|---------|---------|
| **数量** | 通常8个输出通道：TRIGOUT[7:0] |
| **信号方向** | CTI → 外部目标（输出） |
| **信号类型** | 脉冲信号（最小1个时钟周期） |
| **目标组件** | CPU Halt控制、ETM使能、其他CTI、外部引脚 |
| **响应延迟** | 通常0-1个时钟周期 |
| **编程控制** | 通过CTIOUTEN[n]寄存器从内部通道生成 |

### 3.3 TRIGOUT事件目标分类

#### 3.3.1 CPU控制输出
```
TRIGOUT目标           信号功能                    寄存器控制
─────────────────────────────────────────────────────────────
TRIGOUT[0]        → CPU Halt Request           CTIOUTEN0
TRIGOUT[1]        → CPU Resume Request         CTIOUTEN1
TRIGOUT[2]        → ETM Enable                 CTIOUTEN2
TRIGOUT[3]        → PMU Start/Stop             CTIOUTEN3
TRIGOUT[4]        → Custom Debug Action        CTIOUTEN4
TRIGOUT[5]        → External Debug Signal      CTIOUTEN5
TRIGOUT[6]        → Trace Start                CTIOUTEN6
TRIGOUT[7]        → System Trigger             CTIOUTEN7
```

#### 3.3.2 跨核心触发链
```c
// 主核心控制从核心调试状态
Master_Core_Event ──→ Master_CTI_Channel[0]
                            ↓
                    Master_CTI_TRIGOUT[0]
                            ↓
                    Slave_CTI_TRIGIN[0]
                            ↓
                    Slave_CTI_Channel[0]
                            ↓
                    Slave_Core_Halt
```

### 3.4 TRIGOUT寄存器配置

#### CTIOUTEN[n] - 输出使能寄存器
- **地址偏移**: `0x0A0 + (n × 4)` (n=0-7)
- **功能**: 将内部通道[0-3]映射到TRIGOUT[n]

```cmm
; 示例：从Channel 0生成TRIGOUT[0]（用于CPU Halt）
&cti_base=0x580CA000
Data.Set AXI:(&cti_base+0x0A0) %Long 0x1  ; CTIOUTEN0 = bit[0] = Channel 0

; 从Channel 1和Channel 2生成TRIGOUT[1]
Data.Set AXI:(&cti_base+0x0A4) %Long 0x6  ; CTIOUTEN1 = bit[1]|bit[2]
```

#### 寄存器位域映射
```
CTIOUTEN[n] Register (32-bit)
┌───────────────────────────────────────┐
│31            4   3   2   1   0        │
├─────────────────┬───┬───┬───┬─────────┤
│   Reserved      │CH3│CH2│CH1│CH0      │
└─────────────────┴───┴───┴───┴─────────┘
  写1：使能Channel映射到TRIGOUT[n]
  写0：禁用输出
```

### 3.5 TRIGOUT信号时序

```
Channel激活    ──┐   ┌──────┐
   (内部)         └───┘      └─────  (软件/硬件触发)

CTIOUTEN使能   ──────────────────  (配置为1)

TRIGOUT[n]     ──┐   ┌──────┐
   (输出)         └───┘      └─────  (生成脉冲信号)

目标组件响应   ────┐ ┌─────┐
                   └─┘     └─────  (如CPU进入Halt)

输出延迟：     ←→ 0-1 CLK cycle
```

---

## 四、输入输出事件对比分析

### 4.1 核心差异对比表

| 对比维度 | TRIGIN（输入事件） | TRIGOUT（输出事件） |
|---------|-------------------|-------------------|
| **信号方向** | 外部 → CTI | CTI → 外部 |
| **寄存器基址** | `0x020` (CTIINEN) | `0x0A0` (CTIOUTEN) |
| **功能角色** | 事件接收器 | 事件发送器 |
| **触发源** | 外部组件、其他CTI、CPU事件 | 内部通道矩阵 |
| **信号特性** | 电平或脉冲（外部定义） | 脉冲信号（CTI生成） |
| **通道映射** | TRIGIN → Channel | Channel → TRIGOUT |
| **延迟特性** | 1-2个时钟周期（同步） | 0-1个时钟周期（直接） |
| **主要用途** | 接收断点、Watchpoint等事件 | 控制CPU Halt、ETM使能等 |
| **软件控制** | CTIINEN[n]、CTIGATE | CTIOUTEN[n]、CTIGATE |
| **级联能力** | 可接收多级CTI链路信号 | 可触发下游多个CTI |

### 4.2 信号流转完整路径

```
┌─────────────────────────────────────────────────────────────────┐
│                         CTI Internal Architecture                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  TRIGIN[n]  ──→  CTIINEN[n]  ──→  Channel Matrix  ──→  CTIOUTEN[n]  ──→  TRIGOUT[n]  │
│   (输入)          (输入映射)        (4通道逻辑)        (输出映射)           (输出)      │
│                                        │                                              │
│                                        │                                              │
│                                  CTIAPPPULSE                                         │
│                                  CTIAPPSET                                           │
│                                  CTIAPPCLEAR                                         │
│                                 (软件触发)                                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 通道矩阵工作原理

CTI内部通常有4个通道（Channel 0-3），作为TRIGIN和TRIGOUT之间的桥梁：

```
          [Channel Matrix]
TRIGIN[0] ──→ ╔═══╗
TRIGIN[1] ──→ ║CH0║ ──→ TRIGOUT[0]
TRIGIN[2] ──→ ║CH1║ ──→ TRIGOUT[1]
TRIGIN[3] ──→ ║CH2║ ──→ TRIGOUT[2]
TRIGIN[4] ──→ ║CH3║ ──→ TRIGOUT[3]
TRIGIN[5] ──→ ╚═══╝ ──→ TRIGOUT[4-7]
TRIGIN[6-7] ──→      (可配置多对多映射)
```

**关键特性**：
- 多对多映射：多个TRIGIN可以映射到同一个Channel
- 多个Channel可以驱动同一个TRIGOUT
- 逻辑OR关系：任何映射的输入激活都会激活Channel

---

## 五、实际应用场景

### 5.1 场景1：单核调试 - WFI唤醒

**问题**：CPU处于WFI低功耗状态无法Break（参见CORESIGHT_DEBUG_ANALYSIS.md）

**CTI解决方案**：
```cmm
; 使用CTI发送事件唤醒CPU
&cti_base=0x580CA000

; 1. 解锁CTI寄存器
Data.Set AXI:(&cti_base+0xFB0) %Long 0xC5ACCE55  ; CTILAR (Lock Access)

; 2. 使能CTI
Data.Set AXI:(&cti_base+0x000) %Long 0x1         ; CTICONTROL

; 3. 配置Channel 0映射到TRIGOUT[0]（连接到CPU事件输入）
Data.Set AXI:(&cti_base+0x0A0) %Long 0x1         ; CTIOUTEN0 bit[0]

; 4. 软件触发Channel 0（发送事件唤醒）
Data.Set AXI:(&cti_base+0x01C) %Long 0x1         ; CTIAPPPULSE bit[0]

; 5. CPU被唤醒，可以响应Break命令
```

**事件流**：`软件触发 → Channel[0] → TRIGOUT[0] → CPU WFE输入 → CPU唤醒`

---

### 5.2 场景2：多核同步调试

**需求**：Core0断点命中时，自动停止Core1-Core3

#### 方法A：使用Central CTI（推荐）
```cmm
; Central CTI配置（APU示例）
&cent_cti=0x0FE50000

; 1. 解锁并使能Central CTI
Data.Set EZAXI:(&cent_cti+0xFB0) %Long 0xC5ACCE55
Data.Set EZAXI:(&cent_cti+0x000) %Long 0x1

; 2. 配置TRIGIN映射：所有Core的Halt事件 → Channel 0
Data.Set EZAXI:(&cent_cti+0x020) %Long 0x1     ; Core0 Halt → CH0
Data.Set EZAXI:(&cent_cti+0x024) %Long 0x1     ; Core1 Halt → CH0
Data.Set EZAXI:(&cent_cti+0x028) %Long 0x1     ; Core2 Halt → CH0
Data.Set EZAXI:(&cent_cti+0x02C) %Long 0x1     ; Core3 Halt → CH0

; 3. 配置TRIGOUT映射：Channel 0 → 所有Core的Halt请求
Data.Set EZAXI:(&cent_cti+0x0A0) %Long 0x1     ; CH0 → Core0 Halt
Data.Set EZAXI:(&cent_cti+0x0A4) %Long 0x1     ; CH0 → Core1 Halt
Data.Set EZAXI:(&cent_cti+0x0A8) %Long 0x1     ; CH0 → Core2 Halt
Data.Set EZAXI:(&cent_cti+0x0AC) %Long 0x1     ; CH0 → Core3 Halt

; 4. 清除CTI门控（允许传播）
Data.Set EZAXI:(&cent_cti+0x140) %Long 0x0     ; CTIGATE = 0

; 5. 使能Channel 0
Data.Set EZAXI:(&cent_cti+0x014) %Long 0x1     ; CTIAPPSET CH0
```

**事件流**：
```
Core0 断点 → Core0_CTI_TRIGOUT → Central_CTI_TRIGIN[0] → Channel[0] → 
    ├→ Central_CTI_TRIGOUT[0] → Core1_CTI_TRIGIN → Core1 Halt
    ├→ Central_CTI_TRIGOUT[1] → Core2_CTI_TRIGIN → Core2 Halt
    └→ Central_CTI_TRIGOUT[2] → Core3_CTI_TRIGIN → Core3 Halt
```

#### 方法B：CTI级联链路
```cmm
; Core0 CTI → Core1 CTI → Core2 CTI → Core3 CTI
; (适用于没有Central CTI的系统)

; Core0 CTI配置
&core0_cti=0x44020000
Data.Set AXI:(&core0_cti+0x0A0) %Long 0x1      ; CH0 → TRIGOUT[0]

; Core1 CTI配置
&core1_cti=0x44120000
Data.Set AXI:(&core1_cti+0x020) %Long 0x1      ; TRIGIN[0] → CH0
Data.Set AXI:(&core1_cti+0x0A0) %Long 0x1      ; CH0 → TRIGOUT[0]

; 依次配置Core2、Core3...
```

---

### 5.3 场景3：ETM追踪触发

**需求**：特定断点命中时启动ETM追踪

```cmm
&core_cti=0x44020000

; 1. 配置断点事件 → Channel 1
;    (假设CPU断点事件连接到TRIGIN[1])
Data.Set AXI:(&core_cti+0x024) %Long 0x2       ; TRIGIN[1] → CH1

; 2. Channel 1 → ETM启动信号 (TRIGOUT[2])
Data.Set AXI:(&core_cti+0x0A8) %Long 0x2       ; CH1 → TRIGOUT[2]

; 3. 使能CTI
Data.Set AXI:(&core_cti+0x000) %Long 0x1
```

**事件流**：`断点命中 → TRIGIN[1] → Channel[1] → TRIGOUT[2] → ETM.Start`

---

### 5.4 场景4：外部引脚触发调试

**需求**：硬件按钮按下时触发所有CPU停止

```cmm
; 外部引脚 → CTI TRIGIN[7] (通常连接到芯片外部引脚)
&cti_base=0x580CA000

; 1. TRIGIN[7] (外部引脚) → Channel 3
Data.Set AXI:(&cti_base+0x03C) %Long 0x8       ; TRIGIN[7] → CH3

; 2. Channel 3 → TRIGOUT[0] (CPU Halt)
Data.Set AXI:(&cti_base+0x0A0) %Long 0x8       ; CH3 → TRIGOUT[0]

; 3. 使能CTI
Data.Set AXI:(&cti_base+0x000) %Long 0x1
```

**硬件连接**：
```
按钮 → GPIO → SoC TRIGIN[7]引脚 → CTI TRIGIN[7] → CPU Halt
```

---

## 六、关键寄存器完整列表

### 6.1 CTI寄存器映射表

| 偏移地址 | 寄存器名称 | 访问权限 | 功能描述 |
|---------|-----------|---------|---------|
| `0x000` | CTICONTROL | RW | CTI使能控制（bit[0]=1启用） |
| `0x010` | CTIINTACK | WO | 中断确认（清除激活的触发器） |
| `0x014` | CTIAPPSET | WO | 软件设置通道激活 |
| `0x018` | CTIAPPCLEAR | WO | 软件清除通道激活 |
| `0x01C` | CTIAPPPULSE | WO | 软件脉冲触发通道 |
| `0x020-0x03C` | CTIINEN[0-7] | RW | TRIGIN到Channel映射（8个寄存器） |
| `0x0A0-0x0BC` | CTIOUTEN[0-7] | RW | Channel到TRIGOUT映射（8个寄存器） |
| `0x0F0` | CTITRIGINSTATUS | RO | TRIGIN状态（当前电平） |
| `0x0F4` | CTITRIGOUTSTATUS | RO | TRIGOUT状态（当前输出） |
| `0x0F8` | CTICHINSTATUS | RO | Channel状态（当前激活） |
| `0x140` | CTIGATE | RW | 门控寄存器（bit[n]=0允许Channel[n]传播） |
| `0xFB0` | CTILAR | WO | 锁定访问寄存器（写0xC5ACCE55解锁） |
| `0xFB4` | CTILSR | RO | 锁定状态寄存器 |

### 6.2 重要寄存器详解

#### CTICONTROL (0x000)
```
Bit[31:1] - Reserved
Bit[0]    - GLBEN: Global Enable
            0 = CTI禁用
            1 = CTI使能
```

#### CTIGATE (0x140)
```
Bit[31:4] - Reserved
Bit[3]    - Gate Channel 3 (0=使能, 1=门控)
Bit[2]    - Gate Channel 2
Bit[1]    - Gate Channel 1
Bit[0]    - Gate Channel 0

说明：门控位为1时，阻止该通道的事件传播
应用：临时禁用某些触发路径
```

#### CTIAPPPULSE (0x01C)
```
Bit[31:4] - Reserved
Bit[3]    - Pulse Channel 3
Bit[2]    - Pulse Channel 2
Bit[1]    - Pulse Channel 1
Bit[0]    - Pulse Channel 0

写入：写1产生一个时钟周期的脉冲（自动清零）
用途：软件手动触发CTI事件
```

---

## 七、调试技巧与常见问题

### 7.1 CTI状态检查

```cmm
; CTI诊断脚本
LOCAL &cti_base
&cti_base=0x580CA000

PRINT "=== CTI Status Check ==="

; 1. 检查CTI是否使能
&ctrl=Data.Long(AXI:&cti_base+0x000)
PRINT "CTICONTROL: 0x" CONV.Long(&ctrl)
IF (&ctrl&0x1)==0
    PRINT "  [WARNING] CTI is DISABLED!"

; 2. 检查锁定状态
&lsr=Data.Long(AXI:&cti_base+0xFB4)
IF (&lsr&0x1)==1
    PRINT "  [WARNING] CTI is LOCKED!"

; 3. 检查当前触发状态
&trigin=Data.Long(AXI:&cti_base+0x0F0)
&trigout=Data.Long(AXI:&cti_base+0x0F4)
&channel=Data.Long(AXI:&cti_base+0x0F8)
PRINT "TRIGIN Status:  0x" CONV.Long(&trigin)
PRINT "TRIGOUT Status: 0x" CONV.Long(&trigout)
PRINT "Channel Status: 0x" CONV.Long(&channel)

; 4. 检查门控状态
&gate=Data.Long(AXI:&cti_base+0x140)
PRINT "CTIGATE: 0x" CONV.Long(&gate)
IF &gate!=0
    PRINT "  [WARNING] Some channels are GATED!"

; 5. 检查输入映射
PRINT "=== TRIGIN Mappings ==="
LOCAL &i
&i=0
REPEAT 8
(
    &inen=Data.Long(AXI:&cti_base+0x020+(&i*4))
    IF &inen!=0
        PRINT "  TRIGIN[" CONV.Long(&i) "] -> Channels 0x" CONV.Long(&inen)
    &i=&i+1
)

; 6. 检查输出映射
PRINT "=== TRIGOUT Mappings ==="
&i=0
REPEAT 8
(
    &outen=Data.Long(AXI:&cti_base+0x0A0+(&i*4))
    IF &outen!=0
        PRINT "  TRIGOUT[" CONV.Long(&i) "] <- Channels 0x" CONV.Long(&outen)
    &i=&i+1
)
```

### 7.2 常见问题排查

#### 问题1：CTI配置后无效果

**可能原因**：
1. CTI未解锁：检查CTILAR是否已写入`0xC5ACCE55`
2. CTI未使能：检查CTICONTROL[0]是否为1
3. 通道被门控：检查CTIGATE寄存器
4. 映射配置错误：检查CTIINEN/CTIOUTEN

**解决方法**：
```cmm
; 完整初始化序列
&cti=0x580CA000
Data.Set AXI:(&cti+0xFB0) %Long 0xC5ACCE55  ; 1. 解锁
Data.Set AXI:(&cti+0x140) %Long 0x0         ; 2. 清除门控
Data.Set AXI:(&cti+0x000) %Long 0x1         ; 3. 使能CTI
; 4. 配置映射...
```

#### 问题2：多核同步不工作

**检查点**：
- Central CTI地址是否正确
- 所有Core的CTI是否都已配置
- CoreSight时钟域是否已上电
- TRIGIN/TRIGOUT物理连接是否正确

**验证方法**：
```cmm
; 手动触发测试
&cent_cti=0x0FE50000
Data.Set EZAXI:(&cent_cti+0x01C) %Long 0x1  ; 脉冲触发CH0

; 检查所有Core是否停止
system.mode.show
```

#### 问题3：ETM不能被CTI触发

**检查**：
- ETM的TRIGIN连接是否正确（查看ETM文档）
- CTI TRIGOUT信号是否连接到ETM TRIGIN
- ETM是否已配置为外部触发模式

```cmm
; ETM配置为外部触发（示例）
ETM.TRIGger.Mode EXTernal
ETM.TriggerMode EXT
```

---

## 八、性能与时序考虑

### 8.1 延迟分析

```
事件传播延迟 = TRIGIN采样 + 通道处理 + TRIGOUT生成
                (1-2 CLK)   (0 CLK)    (0-1 CLK)
                = 1-3个时钟周期
```

**在200MHz系统中**：延迟 = 5-15 ns

### 8.2 多级CTI级联延迟

```
Level1_CTI → Level2_CTI → Level3_CTI → Target
  (2 CLK)      (2 CLK)      (2 CLK)     (响应)
总延迟 ≈ 6-9个时钟周期
```

**建议**：
- 使用Central CTI减少级联层级
- 关键路径避免超过2级CTI

### 8.3 同步考虑

- **异步时钟域**：CTI通常在APB时钟域，需要考虑与CPU时钟域的CDC（Clock Domain Crossing）
- **亚稳态**：输入信号需要经过同步器，增加1-2个时钟周期延迟
- **背压处理**：某些SoC支持CTI背压机制，防止事件丢失

---

## 九、高级应用

### 9.1 条件触发网络

**场景**：仅当Core0和Core1同时命中断点时，才停止所有Core

```cmm
; 使用2个Channel实现AND逻辑
&cent_cti=0x0FE50000

; Core0断点 → CH0, Core1断点 → CH1
Data.Set EZAXI:(&cent_cti+0x020) %Long 0x1  ; TRIGIN[0] → CH0
Data.Set EZAXI:(&cent_cti+0x024) %Long 0x2  ; TRIGIN[1] → CH1

; 仅当CH0和CH1都激活时，软件检测后触发CH2
; (需要软件轮询CTICHINSTATUS[0:1]，然后写CTIAPPSET[2])
```

**注意**：CTI硬件只支持OR逻辑，AND/XOR需要软件辅助。

### 9.2 性能监控触发

**场景**：PMU计数器溢出时自动停止CPU并启动ETM

```cmm
; PMU Overflow → TRIGIN[4] → CH2 → TRIGOUT[0] (Halt)
;                                 → TRIGOUT[2] (ETM Start)
&cti=0x44020000
Data.Set AXI:(&cti+0x030) %Long 0x4         ; TRIGIN[4] → CH2
Data.Set AXI:(&cti+0x0A0) %Long 0x4         ; CH2 → TRIGOUT[0]
Data.Set AXI:(&cti+0x0A8) %Long 0x4         ; CH2 → TRIGOUT[2]
```

### 9.3 故障注入测试

**场景**：通过外部信号模拟CPU异常，测试系统容错能力

```cmm
; 外部引脚 → TRIGIN[7] → 软件NMI/异常
&cti=0x580CA000
Data.Set AXI:(&cti+0x03C) %Long 0x8         ; TRIGIN[7] → CH3

; 软件轮询CTICHINSTATUS[3]，检测到后触发NMI Handler
```

---

## 十、参考资料

### 10.1 ARM官方文档
- **ARM CoreSight Architecture Specification v3.0**
- **ARM Cross Trigger Interface (CTI-600) Technical Reference Manual**
- **ARM Debug Interface Architecture Specification v5.2 (ADIv5)**

### 10.2 工作区相关文件
| 文件 | 描述 |
|-----|------|
| [CORESIGHT_DEBUG_ANALYSIS.md](CORESIGHT_DEBUG_ANALYSIS.md) | CoreSight调试问题分析 |
| [start-SI-RPU-R82AE.cmm](start-SI-RPU-R82AE.cmm) | R82AE CTI配置示例 |
| [start-AON-M55.cmm](start-AON-M55.cmm) | M55 CTI配置示例 |
| [EVB-Attach/SI_Jtag/start-APU-A78AE-Linux.cmm](EVB-Attach/SI_Jtag/start-APU-A78AE-Linux.cmm) | A78AE Central CTI示例 |

### 10.3 寄存器快速查询

```
TRIGIN相关:
  CTIINEN[0-7]       0x020 - 0x03C
  CTITRIGINSTATUS    0x0F0

TRIGOUT相关:
  CTIOUTEN[0-7]      0x0A0 - 0x0BC
  CTITRIGOUTSTATUS   0x0F4

Channel相关:
  CTIAPPSET          0x014
  CTIAPPCLEAR        0x018
  CTIAPPPULSE        0x01C
  CTICHINSTATUS      0x0F8
  CTIGATE            0x140

控制:
  CTICONTROL         0x000
  CTILAR             0xFB0
```

---

## 十一、总结

### 输入输出核心差异
```
┌─────────────────────┬──────────────────────┐
│     TRIGIN          │       TRIGOUT        │
├─────────────────────┼──────────────────────┤
│ 接收外部事件         │ 发送控制信号          │
│ 被动响应            │ 主动驱动              │
│ 需要同步处理         │ 直接输出              │
│ 支持多源OR合并       │ 可广播到多个目标      │
│ CTIINEN映射         │ CTIOUTEN映射          │
└─────────────────────┴──────────────────────┘
```

### 使用建议
1. **单核调试**：利用TRIGOUT唤醒WFI状态的CPU
2. **多核调试**：通过Central CTI实现同步断点
3. **性能分析**：PMU事件触发ETM追踪
4. **系统验证**：外部引脚注入测试信号

### 最佳实践
- 始终检查CTI锁定状态和使能状态
- 使用CTIGATE进行运行时触发路径控制
- 多核系统优先使用Central CTI而非级联
- 定期读取状态寄存器验证配置正确性

---

**文档版本**: 1.0  
**创建日期**: 2026-02-06  
**工作区路径**: `/xpsdv/usergb/wanghw3/code/cornerstone/tools/emu_tools/trace32-scripts`
