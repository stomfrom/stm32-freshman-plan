# P08 手把手教程：ADC 与数据采集（滤波与标定）

> 配套任务卡：`tasks/P08-ADC与数据采集.md`
> 芯片：**STM32F103C8T6**｜LED = **PC13**（低电平点亮）｜按键 = **PA0**（按下为高电平）
> 环境：STM32CubeMX + Keil MDK（HAL 库）｜串口：USART1，115200
> 外设：**电位器**（或电阻分压）接 ADC 输入｜可选：复用 P07 的 OLED

---

## 目录

| 任务点 | 教程 |
|---|---|
| A1 ADC 单通道采样 | [跳转](#p08-a1) |
| A2 换算成电压打印 | [跳转](#p08-a2) |
| A3 万用表对比验证 | [跳转](#p08-a3) |
| A4 记录 5 个输入点 | [跳转](#p08-a4) |
| A5 封装成采集模块 | [跳转](#p08-a5) |
| B1 ADC + DMA 多通道 | [跳转](#p08-b1) |
| B2 两种数字滤波对比 | [跳转](#p08-b2) |
| B3 两点标定 | [跳转](#p08-b3) |
| B4 数据存 Flash | [跳转](#p08-b4) |
| B5 OLED 显示实时电压 | [跳转](#p08-b5) |

---

<a id="p08-a0"></a>
## 0 · 开工前的准备

### 0.1 ADC 是什么

**ADC（模数转换器）**把**模拟电压**变成**数字值**。单片机的引脚只能读到"高/低"，ADC 能读出"多少伏"。

```
模拟世界                     数字世界
 0V ~ 3.3V  ────[ ADC ]────>  0 ~ 4095（12 位）
```

**三个核心参数**：

| 参数 | 本项目取值 | 说明 |
|---|---|---|
| 分辨率 | **12 位** | 输出 0~4095 |
| 参考电压 Vref | **3.3V** | 换算的基准 |
| 采样时间 | 55.5 cycles 左右 | 源阻抗大时要加长 |

**⭐ 核心换算公式**：

```
电压(mV) = ADC原始值 × 3300 / 4095
```

**1 个计数等于多少伏**：`3300mV / 4095 ≈ 0.806 mV`。所以 ADC 的理论精度约 0.8mV。

### 0.2 引脚选择

F103 的 ADC1 通道：

| 通道 | 引脚 |
|---|---|
| ADC1_IN0 | **PA0** ⚠️ 和板载按键冲突 |
| ADC1_IN1 | **PA1** ✅ 推荐用这个 |

**⚠️ 注意引脚冲突**：PA0 在本方案里是按键。所以**本项目用 PA1** 作为 ADC 输入。

### 0.3 硬件连接

用电位器（推荐，可连续调）：

```
        3.3V
         │
        ┌┴┐
        │ │ 电位器（10kΩ）
        │ │
        ├─┼──> PA1（中间抽头接 ADC 输入）
        │ │
        └┬┘
         │
        GND
```

**没有电位器**：用两个电阻分压产生一个固定电压（如 1kΩ + 2kΩ 得到 2.2V）。

---

<a id="p08-a1"></a>
## A1 · ADC 单通道采样

**任务**：配置一个 ADC 通道（注意参考电压与分辨率设置），用轮询方式单次采样并通过串口打印原始值。

### 知识点

**轮询 vs DMA**：

| 方式 | 特点 | 适用 |
|---|---|---|
| 轮询 | 简单，但采样时要等 | 单次/低速采样 |
| 中断 | 采样完成通知 | 中等速率 |
| **DMA** | 硬件自动搬运，不占 CPU | 多通道连续采样（B1 用）|

A 档先用轮询理解流程，B1 再上 DMA。

**采样时间的意义**：ADC 内部有个采样电容，需要时间充电。**信号源内阻越大，充电越慢**，采样时间设太短会导致读数偏低。

| 源阻抗 | 建议采样时间 |
|---|---|
| 小（运放输出） | 1.5~7.5 cycles |
| 中（分压电阻几十 kΩ） | 55.5 cycles ✅ |
| 大（高阻传感器） | 239.5 cycles |

### 操作步骤

1. CubeMX → `Analog` → **`ADC1`**
2. `IN1` 后面勾 **`IN1`**（引脚图里 PA1 变绿）
3. `Parameter Settings` 设：

   | 参数 | 值 |
   |---|---|
   | `Mode` | Independent mode |
   | `Data Alignment` | Right alignment |
   | `Scan Conversion Mode` | Disabled（单通道） |
   | `Continuous Conversion Mode` | Disabled（要单次） |
   | `Number Of Conversion` | 1 |
   | `Rank 1 → Channel` | Channel 1 |
   | `Rank 1 → Sampling Time` | **55.5 Cycles** |
   | `Resolution` | 12 bits |

4. `GENERATE CODE`

采样代码：

```c
/* USER CODE BEGIN 0 */
#include <stdio.h>

/* 读一次 ADC1_IN1 的原始值（0~4095） */
static uint16_t adc_read_raw(void)
{
    HAL_ADC_Start(&hadc1);                        /* 启动转换 */
    HAL_ADC_PollForConversion(&hadc1, 10);        /* 等转换完成，超时 10ms */
    uint16_t v = (uint16_t)HAL_ADC_GetValue(&hadc1);  /* 取值 */
    HAL_ADC_Stop(&hadc1);
    return v;
}
/* USER CODE END 0 */
```

主循环：

```c
while (1)
{
    uint16_t raw = adc_read_raw();
    printf("ADC raw = %u\r\n", raw);
    HAL_Delay(500);
}
```

### 验证方法

- 串口打印出 0~4095 之间的数值
- **转动电位器，读数跟着连续变化**（这是 ADC 通了的关键证据）
- 电位器转到两端时，读数接近 0 和 4095

### 常见问题

| 现象 | 原因 | 怎么办 |
|---|---|---|
| 读数一直是 4095 | 输入电压超过 3.3V，或引脚配成了输出 | 量输入电压；检查引脚配置 |
| 读数一直是 0 | 输入没接好，或接了 GND | 量输入电压 |
| 读数不动 | 电位器没接中间抽头，或引脚不是 PA1 | 确认接线与 CubeMX 配置 |
| 读数跳动几十个计数 | 内阻大 + 采样时间短，或地线噪声 | 加长采样时间（改 239.5）；检查接地 |

---

<a id="p08-a2"></a>
## A2 · 换算成电压打印

**任务**：把 ADC 原始值按公式换算成实际电压（单位 mV）并打印，同时打印原始值便于对照。

### 知识点

**整数运算避免浮点**：单片机上浮点运算慢且占空间。换算用整数：

```c
/* 电压(mV) = raw × 3300 / 4095 */
uint32_t mv = (uint32_t)raw * 3300UL / 4095UL;
```

**澄清一个常被写错的"溢出"说法**：`raw * 3300 / 4095` 里 `raw` 是 `uint16_t`，但它会**先做整型提升（integer promotion）变成 `int`** 再参与运算。Cortex-M3（F103）上 `int` 是 **32 位**，`4095 × 3300 = 13,513,500 < 2^31`，所以按上面的式子算，**实际不会溢出**，结果是对的。

| 写法 | 结论 |
|---|---|
| `raw * 3300 / 4095`（raw 是 `uint16_t`） | ✅ **不会溢出**：`uint16_t` 先整型提升为 32 位 `int`，`4095×3300 = 13,513,500` 远小于 `2^31` |
| `(uint32_t)raw * 3300 / 4095` | ✅ 推荐这么写：理由**不是"防溢出"**，而是**显式表达意图**——防止将来换到 16 位 `int` 的平台、或把表达式改成 `float` 时出错 |

**记住结论**：Cortex-M3 上不写强制转换也不会算错；但把类型写清楚是"让代码自己说明意图"，是好习惯，不是救命技巧。

### 操作步骤

```c
/* USER CODE BEGIN 0 */
/* raw(0~4095) → 电压(mV)；显式转 uint32_t 是为了表达意图（Cortex-M3 上不转也不会溢出） */
static uint32_t adc_to_mv(uint16_t raw)
{
    return (uint32_t)raw * 3300UL / 4095UL;
}
/* USER CODE END 0 */
```

主循环：

```c
uint16_t raw = adc_read_raw();
uint32_t mv  = adc_to_mv(raw);

printf("raw=%4u  ->  %4lu mV\r\n", raw, (unsigned long)mv);
HAL_Delay(500);
```

**手算验证一组**：`raw = 2048` → `2048 × 3300 / 4095 = 1650 mV`（约为 3.3V 的一半 ✅）

### 验证方法

- 同时打印原始值与电压值
- 电位器转到中间时电压约 1650mV
- 转到两端时分别为 ~0mV 与 ~3300mV
- 能解释"为什么 Cortex-M3 上不写 `(uint32_t)` 也不会溢出，但仍建议显式写出来"

---

<a id="p08-a3"></a>
## A3 · 万用表对比验证

**任务**：用万用表测量同一路输入的实际电压，与程序算出的电压值对比，记录偏差。

### 知识点

**"程序算的"和"实际有多少伏"是两件事**。验证方法：

- 万用表黑笔接 **GND**、红笔接 **PA1**（也就是电位器中间抽头）
- 同一时刻读万用表读数，与串口打印的电压值对比

**为什么会有偏差**：

| 来源 | 影响 |
|---|---|
| 参考电压实际不是精确 3.3V | 系统性偏差（按比例） |
| 万用表自身精度 | ±(0.5%~1%) |
| ADC 的偏移误差与增益误差 | 小偏差（B3 标定能修） |
| 输入阻抗分压 | 读数偏低 |

### 操作步骤

1. 万用表转到 **V⎓** 档（量程 20V），黑笔接 GND，红笔接 PA1
2. 调电位器到某个位置，**保持不动**
3. 同时（先后几秒内）记录：
   - 万用表读数
   - 串口最近一次打印的 mV 值

**在 3 个不同位置各测一次**：

| # | 万用表实测 | 程序算出 | 绝对偏差 | 相对偏差 | 是否 ≤3% |
|---|---|---|---|---|---|
| 1 | ____ mV | ____ mV | ____ | ____% | ✅/❌ |
| 2 | ____ mV | ____ mV | ____ | ____% | ✅/❌ |
| 3 | ____ mV | ____ mV | ____ | ____% | ✅/❌ |

**验收**：在 0.3V~3.0V 范围内，相对偏差 ≤3%。

### 验证方法

- 3 个点都有记录
- 偏差在 3% 以内
- 若超差，能说出至少一种原因（并尝试用 B3 的标定修正）

### 常见问题

| 现象 | 原因 | 怎么办 |
|---|---|---|
| 万用表读数与程序差很多 | 万用表红笔没接在 ADC 输入脚上 | 确认接的是 PA1 |
| 偏差是固定比例（如都差 5%） | 参考电压不是准 3.3V | B3 标定修正 |
| 偏差随机跳动 | 两边不是同一时刻读的 | 调好后 10 秒内读完两边 |
| 程序读数始终偏低 | 输入阻抗分压 | 加长采样时间，或加缓冲运放 |

---

<a id="p08-a4"></a>
## A4 · 记录 5 个输入点

**任务**：转动电位器（或用电阻分压）改变输入电压，观察程序读数随之连续变化，记录 5 个不同输入点的读数。

### 知识点

**"连续变化"是重点**：要确认读数**单调变化**（电压升高读数就升高），而不是"跳变"或"反向"。后者说明接线或配置有问题。

### 操作步骤

1. 从电位器一端开始，缓慢转到另一端，观察串口输出
2. 均匀取 5 个位置，记录：

| # | 电位器位置 | 万用表读数(mV) | ADC 原始值 | 程序电压(mV) | 单调？ |
|---|---|---|---|---|---|
| 1 | 最左（或最低） | | | | — |
| 2 | 约 1/4 | | | | ✅ |
| 3 | 约 1/2 | | | | ✅ |
| 4 | 约 3/4 | | | | ✅ |
| 5 | 最右（或最高） | | | | ✅ |

3. 在 README 里写一句结论，如："5 个点读数随输入电压单调上升，无跳变、无负值，最大值 3.28V、最小值 0.01V。"

### 验证方法

- 5 个点读数**单调变化**（不出现跳变或负值）
- 原始值与电压值同步变化（比例一致）
- 两端读数接近 0 与 3300

---

<a id="p08-a5"></a>
## A5 · 封装成采集模块

**任务**：把采样与换算封装成独立模块（`.c` / `.h`），对外提供"读一次电压"的接口，主程序只调用接口。

### 操作步骤

**`adc_app.h`**：

```c
#ifndef __ADC_APP_H
#define __ADC_APP_H

#include <stdint.h>

/* 初始化（确保 ADC 已由 CubeMX 初始化，这里只做应用层准备） */
void     adc_app_init(void);

/* 读一次电压（单位 mV），内部完成采样 + 换算 */
uint32_t adc_app_read_mv(void);

/* 读一次原始值（调试用） */
uint16_t adc_app_read_raw(void);

#endif
```

**`adc_app.c`**：

```c
#include "adc_app.h"
#include "adc.h"        /* CubeMX 生成的句柄 */

#define VREF_MV      3300UL
#define ADC_FULL     4095UL

uint16_t adc_app_read_raw(void)
{
    HAL_ADC_Start(&hadc1);
    HAL_ADC_PollForConversion(&hadc1, 10);
    uint16_t v = (uint16_t)HAL_ADC_GetValue(&hadc1);
    HAL_ADC_Stop(&hadc1);
    return v;
}

uint32_t adc_app_read_mv(void)
{
    return (uint32_t)adc_app_read_raw() * VREF_MV / ADC_FULL;
}

void adc_app_init(void)
{
    HAL_ADCEx_Calibration_Start(&hadc1);   /* 校准：部分芯片需要 */
}
```

主程序：

```c
#include "adc_app.h"

adc_app_init();
while (1)
{
    printf("%lu mV\r\n", (unsigned long)adc_app_read_mv());
    HAL_Delay(500);
}
```

### 验证方法

- 功能与 A2 一致
- **主程序里不出现 `hadc1` 或 `HAL_ADC_`**（全封装在模块里）
- `adc_app.h` 的接口清晰（读电压 / 读原始值）

---

<a id="p08-b1"></a>
## B1 · ADC + DMA 多通道

**任务**：配置 ADC + DMA 做多通道连续采样（至少 2 个通道），在串口上按通道分组打印数据。

### 知识点

**为什么要 DMA**：多通道连续采样时，如果靠 CPU 一个个搬运，根本忙不过来。**DMA（直接存储器访问）是硬件搬运工**，ADC 转换完自动把数据搬到内存数组，CPU 完全不用管。

```
ADC 转换完成 → DMA 自动把值写进 buf[] → 写满一轮 → 中断通知 CPU
                                        ↓
                              CPU 只在写满时才处理
```

**规则组序列**：多通道时，ADC 按 `Rank 1 → Rank 2 → ...` 的顺序轮流转换，DMA 也按这个顺序放进数组。

### 操作步骤

1. CubeMX → `ADC1` → `Parameter Settings`：
   - `Scan Conversion Mode` 改 **`Enabled`**（多通道必须开）
   - `Continuous Conversion Mode` 改 **`Enabled`**
   - `Number Of Conversion` 改 **`2`**
   - `Rank 1`：Channel 1，Sampling Time 55.5 Cycles
   - `Rank 2`：Channel 2（PA2），Sampling Time 55.5 Cycles
2. 引脚图里确认 **PA2** 也变绿
   > **⚠️ PA2 也要有输入源**：PA2 必须接 **第二个电位器**，或直接接 GND / 3.3V 做固定参考（0 / 4095）；浮空引脚读数会乱跳，别误判成"串道"。
3. `DMA Settings` 标签 → `Add`：
   - `DMA Request` 选 `ADC1`
   - `Mode` 选 **`Circular`**（循环，连续采样）
   - `Data Width`：Peripheral = `Half Word`，Memory = `Half Word`
4. `GENERATE CODE`

代码：

```c
/* USER CODE BEGIN 0 */
#define ADC_CH_CNT   2
static volatile uint16_t adc_buf[ADC_CH_CNT];   /* DMA 目标缓冲区 */
/* USER CODE END 0 */

/* USER CODE BEGIN 2 */
HAL_ADCEx_Calibration_Start(&hadc1);
HAL_ADC_Start_DMA(&hadc1, (uint32_t *)adc_buf, ADC_CH_CNT);   /* 启动 DMA 采样 */
/* USER CODE END 2 */

while (1)
{
    /* 分组打印：buf[0]=CH1(PA1)  buf[1]=CH2(PA2) */
    printf("CH1(PA1)=%4u (%4lu mV)  |  CH2(PA2)=%4u (%4lu mV)\r\n",
           adc_buf[0], (unsigned long)((uint32_t)adc_buf[0] * 3300UL / 4095UL),
           adc_buf[1], (unsigned long)((uint32_t)adc_buf[1] * 3300UL / 4095UL));
    HAL_Delay(500);
}
```

### 验证方法

- 两个通道都在打印，且数值随各自的输入变化
- **改 CH1 的输入，只有 CH1 的值变**（证明没串道）
- 连续运行不卡顿（DMA 不占 CPU）

### 常见问题

| 现象 | 原因 | 怎么办 |
|---|---|---|
| 两个通道数值一样 | 串道：DMA 顺序与 Rank 顺序不一致，或采样间隔太短 | 核对 Rank 顺序；加长采样时间 |
| 某个通道数值乱跳 | 该通道引脚**没接输入源**（浮空） | PA2 接第二个电位器，或直接接 GND / 3.3V 做固定参考 |
| DMA 不搬运 | 没调 `HAL_ADC_Start_DMA`，或 DMA 没配 | 检查启动调用与 DMA Settings |
| 只采一轮就停 | DMA Mode 配成了 Normal | 改成 `Circular` |
| 缓冲区数据乱 | 数组类型与 DMA 数据宽度不匹配 | 用 `uint16_t`（Half Word）|

---

<a id="p08-b2"></a>
## B2 · 两种数字滤波对比

**任务**：实现两种数字滤波（滑动平均、中值滤波），把滤波前后的数据都记录下来，画出或列表对比两条曲线的差别。

### 知识点

**两种滤波的适用场景**：

| 滤波 | 原理 | 优点 | 缺点 | 适合 |
|---|---|---|---|---|
| **滑动平均** | 取最近 N 个值的平均 | 平滑随机噪声 | 响应变慢（跟随突变差） | 缓慢变化的信号 |
| **中值滤波** | 取最近 N 个值的中间值 | **抗尖峰干扰**（毛刺） | 对连续噪声效果一般 | 有突发毛刺的信号 |

**组合使用**：很多场合先中值去毛刺、再平均平滑，效果最好。C 档可以试。

### 操作步骤

```c
/* USER CODE BEGIN 0 */
#define FILT_N   8

/* 滑动平均 */
typedef struct {
    uint16_t buf[FILT_N];
    uint8_t  idx;
    uint32_t sum;
} movavg_t;

static uint16_t movavg_update(movavg_t *m, uint16_t v)
{
    m->sum -= m->buf[m->idx];        /* 减去最老的 */
    m->buf[m->idx] = v;
    m->sum += v;                     /* 加上最新的 */
    m->idx = (m->idx + 1) % FILT_N;
    return (uint16_t)(m->sum / FILT_N);
}

/* 中值滤波（对 N 个值排序取中间） */
static uint16_t median_filter(const uint16_t *src, uint8_t n)
{
    uint16_t tmp[FILT_N];
    for (uint8_t i = 0; i < n; i++) tmp[i] = src[i];
    /* 冒泡排序（n 小，够用） */
    for (uint8_t i = 0; i < n - 1; i++)
        for (uint8_t j = 0; j < n - 1 - i; j++)
            if (tmp[j] > tmp[j + 1]) { uint16_t t = tmp[j]; tmp[j] = tmp[j+1]; tmp[j+1] = t; }
    return tmp[n / 2];               /* 取中间 */
}
/* USER CODE END 0 */
```

采集一组数据并**量化对比波动幅度**：

```c
static movavg_t g_movavg;
static uint16_t g_median_win[FILT_N];
static uint8_t  g_median_idx = 0;

while (1)
{
    uint16_t raw  = adc_app_read_raw();
    uint16_t avg  = movavg_update(&g_movavg, raw);

    g_median_win[g_median_idx] = raw;
    g_median_idx = (g_median_idx + 1) % FILT_N;
    uint16_t med  = median_filter(g_median_win, FILT_N);

    printf("%u,%u,%u\r\n", raw, avg, med);   /* CSV 格式，方便导 Excel 画图 */
    HAL_Delay(50);
}
```

**量化方法**（比肉眼比曲线更可靠）：各采 100 个点，算**最大最小值之差**（波动幅度）：

| 信号 | 最大值 | 最小值 | 波动幅度 |
|---|---|---|---|
| 滤波前（raw） | | | |
| 滑动平均后 | | | |
| 中值滤波后 | | | |

把串口 CSV 粘进 Excel，画三条折线，截图放进交付物。

### 验证方法

- 两种滤波都实现了
- **有量化数据**：滤波后波动幅度明显小于滤波前（写出具体数值）
- 能讲清"什么信号该用哪种滤波"
- （推荐）人为制造一个尖峰（快速猛转电位器），观察中值滤波的抗毛刺效果

---

<a id="p08-b3"></a>
## B3 · 两点标定

**任务**：做两点标定：用两个已知的输入值（如 0.5V 与 2.5V）求出线性换算系数，用第三个已知点验证换算误差。

### 知识点

**为什么要标定**：理论公式假设"参考电压正好 3.3V、ADC 完全线性"。实际会有偏差。**标定就是用两个已知点反推真实的换算关系**：

```
理论：  实际电压 = raw × 3300 / 4095
标定：  实际电压 = k × raw + b      ← k、b 由两个已知点求出
```

**两点求 k、b**：

```
已知两个点 (raw1, V1)、(raw2, V2)：

k = (V2 - V1) / (raw2 - raw1)
b = V1 - k × raw1
```

**为什么用两点而不是一点**：一点只能修"偏移"，两点能同时修"偏移 + 增益"。

### 操作步骤

**1. 取两个已知点**

用万用表测出实际电压（这就是"已知"的标准），同时记录 ADC 原始值：

| 点 | 万用表实测电压 | ADC 原始值 |
|---|---|---|
| 点1（低） | V1 = ____ mV | raw1 = ____ |
| 点2（高） | V2 = ____ mV | raw2 = ____ |

> 用整数运算避免浮点：把 k 和 b **都**放大 1000 倍存成整数：
> ```
> k_x1000 = (V2 - V1) * 1000 / (raw2 - raw1)
> b_x1000 = V1 * 1000 - k_x1000 * raw1
> ```
> 然后 `电压(mV) = (raw * k_x1000 + b_x1000) / 1000`。
> **注意 `b` 也被放大了 1000 倍**——`b` 的单位是 mV，写成 `-2` 相当于把"毫伏"当成了"原始码值"，整条标定线会平移错位。

**2. 求系数**

```c
/* USER CODE BEGIN 0 */
/* 两点标定的系数（用你的实测值填入，或运行时先测再算） */
static int32_t cal_k_x1000 = 806;    /* k × 1000，默认用理论上 3300/4095 ≈ 806 */
static int32_t cal_b_x1000 = 0;      /* b × 1000 */

static uint32_t adc_to_mv_calibrated(uint16_t raw)
{
    int32_t mv = (cal_k_x1000 * (int32_t)raw + cal_b_x1000) / 1000;
    return (mv < 0) ? 0 : (uint32_t)mv;
}
/* USER CODE END 0 */
```

**在纸上/Excel 里算出你实测的 k、b**，填进代码，例如：

```
raw1=100, V1=80mV
raw2=4000, V2=3260mV
k = (3260-80)/(4000-100) = 3180/3900 = 0.8154  → k_x1000 = 815
b = 80 - 0.8154×100 = 80 - 81.5 = -1.5        → b_x1000 = -1500   （不是 -2！）
```

**为什么 `b_x1000` 是 −1500 而不是 −2**：`b = V1 − k × raw1 = 80 − 81.5 = −1.5`，单位是 **mV**。因为 `k` 被放大了 1000 倍，`b` 也必须同样放大 1000 倍，才能和 `raw * k_x1000` 直接相加：`b_x1000 = −1.5 × 1000 = −1500`。写成 `−2` 等于把 −1.5 mV 当成了 −2 个原始码值，标定出来的电压会整体偏掉。

**真值校验**：`raw = 100` → `(815 × 100 − 1500) / 1000 = (81500 − 1500) / 1000 = 80.0 mV` ✅ 正好回到 V1，说明 k、b 这一对填对了。

**3. 验证（第三点）**

取第三个点，比较标定前后的误差：

| 第三点 | 万用表实测 | 理论公式算出 | 标定后算出 | 标定前误差 | 标定后误差 |
|---|---|---|---|---|---|
| ____ | ____ mV | ____ mV | ____ mV | ____% | ____% |

**验收**：标定后第三点误差 ≤5%（且应优于标定前）。

### 验证方法

- 有 k、b 的推导过程（写在 README 里）
- 第三点验证的误差数据
- 能解释"两点标定修正了什么"（偏移 + 增益）

---

<a id="p08-b4"></a>
## B4 · 数据存 Flash

**任务**：把采样数据保存到单片机内部 Flash：存一段数据、断电、重新上电后能读回，并说明写入前为什么要先擦除。

### 知识点

**⭐ 为什么必须先擦除**：Flash 的物理特性——**擦除后所有位变成 1，写入只能把 1 变成 0**。

```
擦除后：  1111 1111
能写入：  1111 0000   （把部分 1 变成 0）
不能写：  1111 1111 ← 想从 0000 变回 1111 不行！必须擦除
```

所以想改一个已经写过的位置，**必须先擦除整个扇区**。

**F103 的 Flash 结构**：

| 项 | 值 |
|---|---|
| 页大小 | **1KB**（F103 中容量） |
| 起始地址 | `0x08000000` |
| 最后 1 页 | 建议用 `0x0800FC00`（避开程序区）|
| 擦写寿命 | 约 1 万次（**不要频繁写**）|

**⭐ 地址是怎么算出来的**：F103C8T6 Flash = **64KB = 0x10000**，范围 `0x08000000`~`0x0800FFFF`；最后一页（1KB）起点 = `0x08000000 + 0x10000 − 0x400 =` **`0x0800FC00`**。若用 128KB 器件（CBT6）才是 `0x0801FC00`——**换了板子还用老地址，写进去就落到程序区，跑飞或者写失败**。

**⭐ 换板防越界的运行时保护**：写入前用 `FLASH_SIZE`（`FLASHSIZE_BASE` = `0x1FFFF7E0` 处读出，单位 KB）或 `#if defined(FLASH_SIZE)` 断言地址 ≤ `0x08000000 + FLASH_SIZE - 0x400`，避免换板后再次越界。例如：

```c
/* F103 的 Flash 容量存放在 0x1FFFF7E0（单位 KB），运行时读出来更保险 */
#define FLASHSIZE_BASE   0x1FFFF7E0UL
static uint32_t flash_kb = *(volatile uint32_t *)FLASHSIZE_BASE;

if (FLSH_DATA_ADDR + 0x400 > 0x08000000UL + flash_kb * 1024UL)
{
    printf("FLASH ADDR OUT OF RANGE (chip flash = %lu KB)\r\n",
           (unsigned long)flash_kb);
    return -1;      /* 地址越界，宁可不写也不要把程序擦掉 */
}
```

**⚠️ 地址要避开程序区**！F103C8 有 64KB Flash，程序一般占前几十 KB。用**最后一页**存数据最安全。

### 操作步骤

```c
/* USER CODE BEGIN 0 */
#define FLSH_DATA_ADDR   0x0800FC00UL      /* 最后一页（1KB），避开程序 */
#define FLSH_MAGIC       0x5A5AA5A5UL      /* 用于判断数据是否有效 */

typedef struct {
    uint32_t magic;           /* 魔数：判断是否为有效数据 */
    uint16_t samples[8];      /* 8 个采样值 */
    uint32_t crc;             /* 简单校验 */
} flash_data_t;

/* 写一组采样值到 Flash（先擦后写） */
static int flash_save(const uint16_t *samples, uint8_t n)
{
    /* 1. 解锁 */
    HAL_FLASH_Unlock();

    /* 2. 擦除：必须整页擦！ */
    FLASH_EraseInitTypeDef er = {0};
    uint32_t sector_err = 0;
    er.TypeErase   = FLASH_TYPEERASE_PAGES;
    er.PageAddress = FLSH_DATA_ADDR;
    er.NbPages     = 1;
    if (HAL_FLASHEx_Erase(&er, &sector_err) != HAL_OK)
    {
        HAL_FLASH_Lock();
        printf("ERASE FAIL\r\n");
        return -1;
    }

    /* 3. 写入：按"半字（16 位）"写 */
    flash_data_t d = {0};
    d.magic = FLSH_MAGIC;
    for (uint8_t i = 0; i < n && i < 8; i++) d.samples[i] = samples[i];
    d.crc = d.magic ^ d.samples[0];

    uint32_t addr = FLSH_DATA_ADDR;
    uint16_t *p = (uint16_t *)&d;
    for (uint32_t i = 0; i < sizeof(flash_data_t) / 2; i++)
    {
        if (HAL_FLASH_Program(FLASH_TYPEPROGRAM_HALFWORD, addr, p[i]) != HAL_OK)
        {
            HAL_FLASH_Lock();
            printf("WRITE FAIL @%lu\r\n", (unsigned long)addr);
            return -1;
        }
        addr += 2;
    }

    /* 4. 上锁 */
    HAL_FLASH_Lock();
    printf("SAVE OK (%u samples)\r\n", n);
    return 0;
}

/* 从 Flash 读回 */
static int flash_load(flash_data_t *out)
{
    /* Flash 可直接按地址读（不需要解锁） */
    const flash_data_t *p = (const flash_data_t *)FLSH_DATA_ADDR;
    *out = *p;
    if (out->magic != FLSH_MAGIC) return -1;      /* 无效数据 */
    if (out->crc != (out->magic ^ out->samples[0])) return -2;   /* 校验失败 */
    return 0;
}
/* USER CODE END 0 */
```

使用：

```c
/* USER CODE BEGIN 2 */
/* 上电先试着读回上次保存的数据 */
flash_data_t loaded;
if (flash_load(&loaded) == 0)
{
    printf("LOADED from Flash: ");
    for (uint8_t i = 0; i < 8; i++) printf("%u ", loaded.samples[i]);
    printf("\r\n");
}
else
{
    printf("No valid data in Flash\r\n");
}

/* 采样 8 个点并保存 */
uint16_t s[8];
for (uint8_t i = 0; i < 8; i++) { s[i] = adc_app_read_raw(); HAL_Delay(200); }
flash_save(s, 8);
/* USER CODE END 2 */
```

**测试流程**：

1. 上电 → 打印 `No valid data` → 采样保存 → `SAVE OK`
2. **断电**（拔电源）
3. **重新上电** → 应打印 `LOADED from Flash:` 加那 8 个值
4. 记录证据（串口两次上电的日志截图）

### 验证方法

- 断电重上电后能读回数据，且**内容与写入时一致**
- 有两次上电的串口日志作为证据
- 能讲清"为什么写之前必须擦除"（Flash 擦除后为 1，写入只能 1→0）
- **能讲清"为什么用最后一页"**（避开程序区）
- 能讲清最后一页地址 `0x0800FC00` 是怎么由 64KB 算出来的，并知道 128KB 器件要用 `0x0801FC00`

### 常见问题

| 现象 | 原因 | 怎么办 |
|---|---|---|
| 写入失败 | 地址落在程序区被保护，或没解锁 | 用最后一页；确认 `HAL_FLASH_Unlock` 调了 |
| 读回的 magic 不对 | 没擦除就写（旧数据挡住） | 确保先擦后写 |
| 程序跑飞 | 擦写时关中断太久 / 地址覆盖了程序 | 用最后一页；擦写期间避免中断干扰 |
| 每次上电都是 No valid data | 写入失败或地址不对 | 打印写入返回值定位 |
| 换了 CBT6 板子后写入失败 | 地址按 C8T6（64KB）算的，落到别处 | 用 `FLASH_SIZE` 运行时判断或改宏 |

---

<a id="p08-b5"></a>
## B5 · OLED 显示实时电压

**任务**：用 OLED（P07 的驱动模块）把实时电压值显示出来，实现"采集 + 显示"联动，刷新不闪屏。

### 知识点

**两个关键要求**：

1. **不闪屏**：用 P07 的"局部更新 + 定长显示"技巧，不要每帧清屏
2. **两个任务都要跑**：采集（ADC）和显示（OLED）不能互相阻塞——ADC 用轮询没问题（很快），但显示不要每轮都全屏重绘

### 操作步骤

```c
/* USER CODE BEGIN Includes */
#include "adc_app.h"
#include "oled.h"
/* USER CODE END Includes */

/* USER CODE BEGIN 2 */
oled_init();
oled_clear();
oled_set_pos(0, 0);                          /* 第 0 页第 0 列（接口见 P07 附录 A） */
oled_print("Voltage Monitor", 15);
oled_set_pos(4, 0);                          /* 第 4 页：从第 32 行起 */
oled_print("mV:", 3);
/* USER CODE END 2 */

uint32_t t_refresh = 0;
while (1)
{
    uint32_t now = HAL_GetTick();

    if (now - t_refresh >= 200)          /* 每 200ms 刷新一次（5Hz，肉眼够流畅） */
    {
        t_refresh = now;

        uint32_t mv = adc_app_read_mv();

        /* 定长显示，前面补零 → 避免残影、不闪屏 */
        oled_set_pos(4, 40);                         /* 第 4 页第 40 列 */
        oled_printf("%05lu", (unsigned long)mv);     /* 宽 5 位，如 "01650" */

        /* 顺便在屏上画一个简单的条形指示（0~3300mV 映射到 0~100 格） */
        uint8_t bar_len = (uint8_t)((mv > 3300 ? 3300 : mv) * 100UL / 3300UL);
        for (uint8_t x = 0; x < 100; x++)
            oled_pixel(x, 6, (x < bar_len) ? 1 : 0);   /* 需要驱动支持画点 */

        printf("%lu mV\r\n", (unsigned long)mv);       /* 串口同步 */
    }
}
```

> `oled_pixel` 需要你的驱动支持画点。若没有，可以跳过条形图，只显示数字（任务只要求"实时电压值显示出来"）。

### 验证方法

- 屏上电压值随电位器连续变化
- **连续观察 30 秒无明显闪屏**（这是任务要求）
- 串口打印与屏上显示一致
- 采集与显示互不干扰（转动电位器，屏上跟着变）

---

## 附：ADC 关键参数与公式速查

| 项 | 值/公式 |
|---|---|
| 分辨率 | 12 位 → 0~4095 |
| 参考电压 | 3.3V（3300mV） |
| 理论换算 | `mV = raw × 3300 / 4095` |
| 1 计数 | ≈ 0.806 mV |
| **类型提升** | 建议写 `(uint32_t)raw × 3300 / 4095`（表达意图；Cortex-M3 上不写也不会溢出） |
| 两点标定 | `k = (V2-V1)/(raw2-raw1)`，`b = V1 - k×raw1`；整数化时 k、b 同乘 1000 |

---

## 下一步

- 回到任务卡 `tasks/P08-ADC与数据采集.md` 逐条勾选
- 交付物：源码（采集/滤波/Flash 模块）、5 个输入点记录表、万用表对比数据、滤波对比图、标定推导与验证、断电读回证据
- 照 `REVIEW_TEMPLATE.md` 准备 10 分钟演示
