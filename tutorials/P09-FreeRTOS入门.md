# P09 手把手教程：FreeRTOS 入门（多任务数据采集器）

> 配套任务卡：`tasks/P09-FreeRTOS入门.md`
> 芯片：**STM32F103C8T6**（20KB RAM，注意堆栈配置）
> 环境：STM32CubeMX + Keil MDK（HAL 库）｜串口：USART1，115200
> 前置：P08 的 ADC 采集模块可复用

---

## 目录

| 任务点 | 教程 |
|---|---|
| A1 启用 FreeRTOS 生成工程 | [跳转](#p09-a1) |
| A2 创建 2 个任务 | [跳转](#p09-a2) |
| A3 vTaskDelay 与 HAL_Delay 的区别 | [跳转](#p09-a3) |
| A4 加第 3 个任务 | [跳转](#p09-a4) |
| A5 三个任务执行次数核对 | [跳转](#p09-a5) |
| B1 队列传数据 | [跳转](#p09-b1) |
| B2 信号量/互斥锁保护共享资源 | [跳转](#p09-b2) |
| B3 栈水位检查与调优 | [跳转](#p09-b3) |
| B4 移植 P08 采集功能，30 分钟测试 | [跳转](#p09-b4) |
| B5 任务关系图与优先级说明 | [跳转](#p09-b5) |

---

<a id="p09-a0"></a>
## 0 · 开工前的准备

### 0.1 RTOS 解决什么问题

裸机主循环的写法：

```c
while (1) {
    read_keys();        /* 想做这个 */
    update_display();   /* 也想做这个 */
    report_data();      /* 还想做这个 */
    /* 谁快了谁慢了都互相影响，任何一个慢操作会拖住其他所有 */
}
```

**RTOS（实时操作系统）**让每个功能变成**独立的任务**，各自按自己的节奏跑：

```
任务1 (LED)     ──每500ms翻转一次──
任务2 (上报)    ──────每2000ms打印一次──────
任务3 (按键)    ─每50ms检查一次─
        ↑ 它们"同时"在跑（其实是调度器快速轮流切换）
```

**好处**：一个任务卡住（比如 `vTaskDelay`）**不会拖死别的任务**——它会主动让出 CPU。

### 0.2 F103C8 的资源约束（重要）

| 资源 | 大小 | 影响 |
|---|---|---|
| Flash | 64 KB | FreeRTOS 占 6~10 KB，够 |
| **RAM** | **20 KB** | **这是瓶颈**！每个任务要独立栈，堆也要分 |

**20KB RAM 下的经验值**：

| 项 | 建议 |
|---|---|
| 每个任务的栈 | **128~256 words**（512B~1KB） |
| `configTOTAL_HEAP_SIZE` | **4096~8192 字节** |
| 任务数量 | 3~5 个，别贪多 |

**栈给小了会溢出**（破坏别的内存，出现各种奇怪现象）；给大了 RAM 不够。

### 0.3 概念对照表（裸机 → RTOS）

| 裸机做法 | RTOS 做法 |
|---|---|
| `while(1)` 里轮流调用各功能 | 每个功能一个独立任务 |
| `HAL_Delay()` 等时间 | `vTaskDelay()` 让出 CPU |
| 全局变量传数据 | 队列（queue） |
| 靠标志位协调 | 信号量（semaphore）/ 互斥锁（mutex） |

---

<a id="p09-a1"></a>
## A1 · 启用 FreeRTOS 生成工程

**任务**：在 CubeMX 里启用 FreeRTOS，生成一个可编译可下载的 RTOS 工程。

### 知识点

**CubeMX 里的两个 FreeRTOS 选项**：

| 选项 | 说明 | 建议 |
|---|---|---|
| `CMSIS_V1` | 较旧，接口封装层 | 教程多 |
| `CMSIS_V2` | 较新 | **推荐** |

两者本质都是 FreeRTOS，只是包装层不同。本教程用 **CMSIS_V2**。

**⚠️ 与 HAL 库的时钟源冲突**：FreeRTOS 需要自己的时基（通常用 SysTick），而 HAL 也用 SysTick。CubeMX 会提示你改 HAL 的时基源——**必须改**，否则会冲突。

### 操作步骤

1. CubeMX → `Middleware and Software Packs` → **`FREERTOS`**
2. `Interface` 选 **`CMSIS_V2`**
3. 出现"建议修改 HAL 时基源"的提示 → 点 **`Yes`**（会把 HAL 时基改到 TIM 上）
4. 到 `System Core` → `SYS` 确认 `Timebase Source` 已变成 **`TIM1`**（或别的 TIM，不再是 SysTick）
5. 切到 `FREERTOS` → `Tasks and Queues` 标签 → 里面已有一个 `defaultTask`
6. `GENERATE CODE`

### 验证方法

- 工程生成成功，编译 **0 错误**
- `main.c` 里能看到 `osKernelInitialize()`、`osKernelStart()` 的调用
- `Core/Src/freertos.c` 文件存在，里面有 `StartDefaultTask` 函数
- 下载后程序能跑（LED 若已配置会闪）

### 常见问题

| 现象 | 原因 | 怎么办 |
|---|---|---|
| 编译报 SysTick 相关错误 | HAL 时基没改 | 在 SYS 里把 Timebase 改成 TIM |
| 编译报 FreeRTOS 头文件找不到 | 中间件没启用 | 确认 `FREERTOS` 已启用并重新生成 |
| 下载后卡住 | 栈/堆太小 | 加大 `configTOTAL_HEAP_SIZE` 与任务栈 |
| 编译报 heap 不足 | `configTOTAL_HEAP_SIZE` 太小 | 改大到 8192 |

---

<a id="p09-a2"></a>
## A2 · 创建 2 个任务

**任务**：创建 2 个任务：一个让 LED 以 1Hz 闪烁，一个每 2 秒往串口打印一行信息。

### 知识点

**任务函数长这样**（永不返回）：

```c
void MyTask(void *argument)
{
    /* 初始化（只执行一次） */

    for (;;)                  /* 死循环，任务不能退出 */
    {
        /* 干活 */

        vTaskDelay(pdMS_TO_TICKS(1000));   /* 让出 CPU 指定时间 */
    }
}
```

**`pdMS_TO_TICKS(ms)`**：把毫秒换算成 RTOS 的 tick 数。**凡是要延时就用它**，别直接用 tick 数。

**头文件别漏**：`cmsis_os2.h` **不含** `vTaskDelay` 与 `pdMS_TO_TICKS`（它们属于 FreeRTOS 原生 API）。只要你在 CMSIS-RTOS2 工程里直接写 `vTaskDelay` / `pdMS_TO_TICKS`，就必须补上：

```c
#include "FreeRTOS.h"     /* pdMS_TO_TICKS、configUSE_TIME_SLICING 等 */
#include "task.h"         /* vTaskDelay / vTaskDelayUntil 的声明 */
```

> **⚠️ 栈单位：CMSIS-RTOS2 的 `stack_size` 单位是字节，CubeMX 界面里填的是 words（字），两者差 4 倍**，这是最常见的栈溢出原因。跑 `printf` 的任务至少给 **1024 字节**。

**为什么必须显式给 `osThreadAttr_t`**：`osThreadNew(task, NULL, NULL)` 用的是默认属性（默认栈很小、名字为空），一旦任务里有 `printf` 或深层函数调用就会溢出。显式指定 `name`、`stack_size`、`priority`，出问题时才能通过栈水位（B3）定位到具体任务。

### 操作步骤

**方式一（推荐，用 CubeMX 图形界面）**

1. `FREERTOS` → `Tasks and Queues` → `Add`：
   - `Task Name`: `LedTask`，`Priority`: `osPriorityNormal`，`Stack Size`: **256**（**界面单位是 words**，256 words = 1024 字节）
   - 再加一个 `ReportTask`，同上配置
2. 生成的代码在 `Core/Src/freertos.c` 里

> 记住这个换算：**CubeMX 界面填 256（words）× 4 = 运行时 1024 字节**；而代码里 `osThreadAttr_t.stack_size` 填的是**字节数**，填 256 就只有 256 字节，连一次 `printf` 都不够。

**方式二（自己写，更灵活）**

先补头文件与任务函数声明（放在 `/* USER CODE BEGIN 0 */`）：

```c
#include "cmsis_os2.h"    /* CMSIS_V2 接口 */
#include "FreeRTOS.h"     /* pdMS_TO_TICKS 在这里（cmsis_os2.h 不含） */
#include "task.h"         /* vTaskDelay 的声明在这里 */

/* 任务函数的声明 */
void LedTask(void *argument);
void ReportTask(void *argument);

/* 显式属性：name / stack_size（字节！）/ priority */
static const osThreadAttr_t led_attr = {
    .name       = "led",
    .stack_size = 1024,              /* 1024 字节 = 256 words */
    .priority   = osPriorityNormal,
};

static const osThreadAttr_t rep_attr = {
    .name       = "report",
    .stack_size = 1024,              /* 有 printf，至少 1024 字节 */
    .priority   = osPriorityNormal,
};
```

> Keil 若报"designated initializer"相关错误，把 `Options for Target → C/C++ → C99 Mode` 打开；若必须用 C90，就改成先 `osThreadAttr_t led_attr = {0};` 再逐项 `led_attr.name = "led";` 赋值（此时不能加 `const`）。

在 `main()` 里创建任务（`/* USER CODE BEGIN 2 */`，且**必须在 `osKernelStart()` 之前**）：

```c
osThreadId_t led_id = osThreadNew(LedTask, NULL, &led_attr);
osThreadId_t rep_id = osThreadNew(ReportTask, NULL, &rep_attr);

if (led_id == NULL || rep_id == NULL)
{
    printf("task create failed: heap too small?\r\n");
}
```

在 `/* USER CODE BEGIN 4 */`（或独立文件）里实现：

```c
/* LED 任务：1Hz 翻转（半周期 500ms） */
void LedTask(void *argument)
{
    for (;;)
    {
        HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin);
        vTaskDelay(pdMS_TO_TICKS(500));      /* 让出 CPU 500ms */
    }
}

/* 上报任务：每 2 秒打印一行 */
void ReportTask(void *argument)
{
    uint32_t cnt = 0;
    for (;;)
    {
        cnt++;
        printf("[Report] alive #%lu\r\n", (unsigned long)cnt);
        vTaskDelay(pdMS_TO_TICKS(2000));
    }
}
```

> **本文统一用 CMSIS-RTOS2 API**（`osThreadNew` / `osDelay` / `osMessageQueuePut` 等）；如果你要混用原生 API（比如在 CMSIS 任务里直接调 `vTaskDelay`），必须自己 include `FreeRTOS.h` 与 `task.h`，且**不要混用 `osThreadId_t` 与 `TaskHandle_t`**——它们属于两层封装，类型不兼容，混着传句柄会编译报错或运行异常。

> **用 CubeMX 方式时**：任务函数框架已生成在 `freertos.c`，你只需在 `for(;;)` 里填内容，并加 `printf` 重定向（注意要在任务启动前初始化串口）。

### 验证方法

- LED 以 1Hz 闪
- 串口每 2 秒打印一行，编号递增
- 两者**同时**工作、互不影响
- 任务名能在调试器/`osThreadGetName()` 里看到（说明用的是显式属性，不是默认值）

### 常见问题

| 现象 | 原因 | 怎么办 |
|---|---|---|
| 只有 LED 闪，串口没输出 | 串口没初始化，或任务没创建成功 | 检查 `MX_USART1_UART_Init()` 调用顺序 |
| 任务完全没跑 | 任务创建失败（堆不足） | 检查 `osThreadNew` 返回值是否为 `NULL`；加大堆 |
| 程序卡在启动 | 栈太小 | 加大任务栈（注意 `stack_size` 是**字节**） |
| 编译报 `vTaskDelay` / `pdMS_TO_TICKS` 未定义 | 只 include 了 `cmsis_os2.h` | 补 `#include "FreeRTOS.h"`、`#include "task.h"` |
| `printf` 输出交叉乱码 | 多任务同时打印（B2 解决） | 见 B2 |

---

<a id="p09-a3"></a>
## A3 · vTaskDelay 与 HAL_Delay 的区别

**任务**：用 `vTaskDelay` 控制两个任务的节拍，在 README 里说明它与裸机 `HAL_Delay` 的本质区别（一个是让出 CPU，一个是原地空等）。

### 知识点

**⭐ 这是 RTOS 最核心的一个概念**：

```
HAL_Delay(500)：      while (uwTick 没到) { /* 原地忙等，不让出 CPU */ }
                      ↑ 任务一直占着 CPU、不进入阻塞态；但调度器并没有被关掉

osDelay(500)：        把本任务标记为"阻塞"，交还 CPU 给调度器
                      调度器立刻去跑别的就绪任务
                      500ms 后本任务重新变"就绪"，等调度器安排
                      ↑ CPU 在这 500ms 里被别的任务充分使用
```

**一张对比表**：

| | `HAL_Delay()` | `osDelay()` / `vTaskDelay()` |
|---|---|---|
| 等待时 CPU | **忙等空转**，占满 CPU | **让给其他任务** |
| 是否进入阻塞态 | 否（一直在就绪/运行） | 是 |
| 调度器是否关闭 | **不关闭**，SysTick 照常运行 | 调度器正常运行 |
| 高优先级任务 | 仍能抢占（不会被完全挡住） | 正常抢占 |
| 同优先级任务 | 只能靠时间片轮流（`configUSE_TIME_SLICING=1`），**时序抖动** | 正常调度 |
| 低优先级任务 | 会被**饿死** | 正常跑 |
| 真实危害 | 占满 CPU、浪费电、时序抖动 | 无 |

**准确的说法**：`HAL_Delay` 是**忙等**——它不让出 CPU、不进入阻塞态，但**不会关闭调度器**。SysTick 照常在跑，所以**高优先级任务仍会抢占它**；同优先级任务靠时间片轮流（`configUSE_TIME_SLICING=1`），所以你会发现打印节拍被拉长且不可预测；**低优先级任务会被饿死**。真正的问题是它**占满 CPU、浪费电，而且同级任务的时间片轮转导致时序抖动**。

> 想稳定复现"ReportTask 间隔变差"的现象：把 ReportTask 的优先级设为 **`osPriorityBelowNormal`**（比 LedTask 低），就能观察到它被饿（打印次数明显低于理论值）。反过来把 ReportTask 设成更高优先级，你会看到它照样能跑——**因为调度器没关**，这正是"忙等 ≠ 关调度器"的直接证据。

正确的延时是 **`osDelay(500)`**（CMSIS-RTOS2，单位 ms）；用原生 API 时是 `vTaskDelay(pdMS_TO_TICKS(500))`（需 include `FreeRTOS.h` 与 `task.h`）。

### 操作步骤

在 README 里写一段对比说明（照上面表格组织），并**做一个小实验证明区别**：

```c
/* 实验：把 LedTask 里的延时换成 HAL_Delay(500)，观察 ReportTask */
void LedTask(void *argument)
{
    for (;;)
    {
        HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin);
        HAL_Delay(500);        /* ⚠️ 实验用：忙等，不让出 CPU */
    }
}
```

观察现象：`ReportTask` 的打印**间隔会变得不稳定**（被 LED 任务的时间片轮转拖着），节拍被拉长且不可预测；把 ReportTask 优先级降到 `osPriorityBelowNormal` 后，它会明显**被饿**（打印次数远低于理论值）。改回 `osDelay(500)` 后恢复正常。

| 用的延时 | ReportTask 打印间隔 | 是否均匀 |
|---|---|---|
| `HAL_Delay(500)` | 实测 ____ ms | 否 |
| `osDelay(500)` | 实测 ____ ms | **是** |

### 验证方法

- README 里有清晰的对比说明
- 有实验数据（换用 HAL_Delay 后打印间隔变得不均匀；降优先级后能被饿）
- 能口头回答："为什么 RTOS 里不能用 HAL_Delay？"（答：它是忙等，不让出 CPU、占满 CPU，同级任务只能靠时间片轮转导致时序抖动，低优先级任务会被饿死）

---

<a id="p09-a4"></a>
## A4 · 加第 3 个任务

**任务**：创建第 3 个任务：周期性读取按键状态并打印，确认三个任务互不干扰地各自运行。

### 操作步骤

```c
/* 按键任务：每 50ms 检查一次按键（复用 P03/P04 的读法） */
void KeyTask(void *argument)
{
    GPIO_PinState last = GPIO_PIN_RESET;
    uint32_t cnt = 0;

    for (;;)
    {
        GPIO_PinState now = HAL_GPIO_ReadPin(KEY_GPIO_Port, KEY_Pin);

        if (now != last)                   /* 只在变化时打印，避免刷屏 */
        {
            last = now;
            cnt++;
            printf("[Key] %s  #%lu\r\n",
                   (now == GPIO_PIN_SET) ? "PRESSED" : "RELEASED",
                   (unsigned long)cnt);
        }

        vTaskDelay(pdMS_TO_TICKS(50));     /* 50ms 采样一次 */
    }
}
```

创建：

```c
static const osThreadAttr_t key_attr = {
    .name       = "key",
    .stack_size = 1024,              /* 字节 */
    .priority   = osPriorityNormal,
};
osThreadId_t key_id = osThreadNew(KeyTask, NULL, &key_attr);
```

> **注意采样周期 50ms**：按键机械抖动典型 0.5~5ms，50ms 采样本身就相当于粗去抖了。要做精细去抖可复用 P04 的状态机。

### 验证方法

- 三个任务**都在工作**：LED 闪 + 报告打印 + 按键响应
- 按按键能立刻打印（≤100ms 响应）
- **按住 LED 闪不停、报告照常打印**（证明三者互不干扰）
- 长时间运行不乱

---

<a id="p09-a5"></a>
## A5 · 三个任务执行次数核对

**任务**：用串口分别打印三个任务的执行次数，连续运行 10 分钟后核对每个任务的实际执行频率是否与设定一致。

### 知识点

**为什么要核对频率**：如果某个任务没达到设定频率，说明：

- 它被更高优先级的任务饿死了
- 或者有阻塞操作拖慢了它
- 或者栈溢出导致行为异常

**这是验证"调度正常"的最直接手段。**

### 操作步骤

给每个任务加计数：

```c
volatile uint32_t led_cnt = 0, rep_cnt = 0, key_cnt = 0;

void LedTask(void *argument)
{
    for (;;) { led_cnt++; HAL_GPIO_TogglePin(...); vTaskDelay(pdMS_TO_TICKS(500)); }
}
void ReportTask(void *argument)
{
    for (;;) { rep_cnt++; printf(...); vTaskDelay(pdMS_TO_TICKS(2000)); }
}
void KeyTask(void *argument)
{
    for (;;) { key_cnt++; /* 读按键 */ vTaskDelay(pdMS_TO_TICKS(50)); }
}
```

**加一个统计任务**（或让 ReportTask 兼职）每 60 秒打印一次：

```c
printf("== STAT t=%lus  led=%lu rep=%lu key=%lu ==\r\n",
       (unsigned long)(HAL_GetTick()/1000),
       (unsigned long)led_cnt, (unsigned long)rep_cnt, (unsigned long)key_cnt);
```

**核对表**（连续跑 10 分钟）：

| 任务 | 设定周期 | 理论 600 秒内次数 | 实测次数 | 偏差 | 是否 ≤5% |
|---|---|---|---|---|---|
| LedTask | 500ms | 1200 | | | ✅/❌ |
| ReportTask | 2000ms | 300 | | | ✅/❌ |
| KeyTask | 50ms | 12000 | | | ✅/❌ |

**理论次数怎么算**：`600秒 ÷ 周期`。例：LedTask 每 500ms 一次 → 600/0.5 = 1200 次。

### 验证方法

- 三个任务的实际次数都接近理论值（偏差 ≤5%）
- 连续运行 10 分钟**不死机、不重启**
- 有完整的核对表

### 常见问题

| 现象 | 原因 | 怎么办 |
|---|---|---|
| 某任务次数远低于理论 | 被高优先级任务饿死，或有阻塞 | 检查优先级设置；确认没用 `HAL_Delay` |
| 次数远高于理论 | 延时算错了 | 确认 `pdMS_TO_TICKS` 用得对 |
| 跑一会儿就重启 | 栈溢出或硬件异常 | 检查栈水位（B3）|
| 次数一直不动 | 任务创建失败 | 检查 `osThreadNew` 返回值 |

---

<a id="p09-b1"></a>
## B1 · 队列传数据

**任务**：用队列在两个任务之间传递数据（一个任务生产采集数据、另一个任务消费并打印），说明不用全局变量的原因。

### 知识点

**为什么不用全局变量**：

| 问题 | 全局变量 | 队列 |
|---|---|---|
| 读写竞争 | 生产者写一半被消费者读 → 数据错乱 | 队列内部有保护 |
| 数据丢失 | 消费者慢了，新数据覆盖旧的 | 队列缓存，满了可选择阻塞或丢弃 |
| 数据错序 | 无保证 | 先进先出（FIFO）|
| 满/空判断 | 要自己写 | 队列 API 自带 |

**简言之**：全局变量是"一个共享白板"，队列是"有管理的传送带"。

### 操作步骤

**1. 创建队列**（CubeMX 里 `FREERTOS` → `Queues` → `Add`，或在代码里建）

```c
/* 定义消息类型 */
typedef struct {
    uint16_t adc_raw;
    uint32_t tick;
} sensor_msg_t;

/* 代码里创建（在 main 的任务创建区） */
osMessageQueueId_t sensor_q = osMessageQueueNew(8, sizeof(sensor_msg_t), NULL);
/*                                              ^ 深度 8 条            */
```

**2. 生产者任务**

```c
void ProducerTask(void *argument)
{
    sensor_msg_t msg;
    for (;;)
    {
        msg.adc_raw = adc_app_read_raw();
        msg.tick    = HAL_GetTick();

        /* 放入队列，若满则最多等 10ms */
        if (osMessageQueuePut(sensor_q, &msg, 0, pdMS_TO_TICKS(10)) != osOK)
        {
            printf("[Prod] queue full, dropped\r\n");
        }

        vTaskDelay(pdMS_TO_TICKS(100));      /* 10Hz 采集 */
    }
}
```

**3. 消费者任务**

```c
void ConsumerTask(void *argument)
{
    sensor_msg_t msg;
    for (;;)
    {
        /* 从队列取，永久等待；取到就处理 */
        if (osMessageQueueGet(sensor_q, &msg, NULL, osWaitForever) == osOK)
        {
            printf("[Cons] raw=%u  t=%lu  -> %lu mV\r\n",
                   msg.adc_raw, (unsigned long)msg.tick,
                   (unsigned long)((uint32_t)msg.adc_raw * 3300UL / 4095UL));
        }
    }
}
```

### 验证方法

- 生产者产出与消费者处理**数量一致**（记录两者计数对比）
- 无数据错乱、无错序（时间戳单调不减）
- 故意让消费者变慢（`vTaskDelay` 加大），观察队列满时的行为（丢弃并计数）
- 能讲清"为什么不用全局变量"（列出竞争/丢失/错序三个问题）

---

<a id="p09-b2"></a>
## B2 · 信号量 / 互斥锁保护共享资源

**任务**：用信号量或互斥锁保护一个被多个任务访问的共享资源（如共享的串口或共享的显示缓存），并说明不加保护时会出现什么现象。

### 知识点

**不加保护会怎样**——以共享串口为例：

```
任务A: printf("HELLO")   →  开始逐字符发送 H E L ...
任务B: 抢过来 printf("WORLD") → ...  O W O R L D
                                ↑ 输出变成 "HELWOORLDLD" 混成一团
```

**这就是"交叉输出"**——现象是串口日志里两个任务的输出互相穿插，完全不可读。

**两种同步原语的区别**：

| | 信号量（Semaphore） | 互斥锁（Mutex） |
|---|---|---|
| 用途 | **同步**（通知"有事发生了"）| **互斥**（保护共享资源） |
| 所有权 | 无（谁都能给/取）| **有**（谁锁定谁解锁） |
| 优先级继承 | 无 | **有**（防优先级反转）|
| 本项目用途 | 中断通知任务 | **保护串口/OLED** |

**保护共享资源用互斥锁**（它有优先级继承，能防优先级反转）。

### 操作步骤

**1. 创建互斥锁**

```c
osMutexId_t uart_mtx = osMutexNew(NULL);
```

**2. 封装成带保护的打印函数**

```c
/* 线程安全的打印（串口是共享资源） */
void safe_printf(const char *fmt, ...)
{
    char buf[128];
    va_list ap;
    va_start(ap, fmt);
    vsnprintf(buf, sizeof(buf), fmt, ap);
    va_end(ap);

    osMutexAcquire(uart_mtx, osWaitForever);    /* 锁定 */
    HAL_UART_Transmit(&huart1, (uint8_t *)buf, strlen(buf), 100);
    osMutexRelease(uart_mtx);                   /* 解锁 */
}
```

**3. 所有任务改用 `safe_printf`**

**4. 做对照实验**（这是任务要求"说明不加保护会出现什么现象"）：
- 先让两个任务都用**裸 `printf`**，观察输出交叉
- 再改成 `safe_printf`，观察输出整齐

| 版本 | 现象 | 截图 |
|---|---|---|
| 裸 printf | 两任务输出交叉混乱 | |
| safe_printf | 每次输出完整不交叉 | |

### 验证方法

- 有一段"交叉混乱"的对照证据（截图）
- 加锁后输出整齐、无交叉
- 能讲清互斥锁与信号量的区别与各自用途
- 能回答"为什么这里用互斥锁而不是信号量"（答：保护资源要用有所有权的锁，且有优先级继承）

> **性能提示**：不要为了保护打印而频繁加锁解锁（增加开销）。也可以改成"由一个任务统一负责输出"的架构——这也是一种常见做法。

---

<a id="p09-b3"></a>
## B3 · 栈水位检查与调优

**任务**：用栈水位检查接口查看每个任务的栈使用情况，把栈大小调整到合理值（既不浪费也不溢出），记录调整前后的数值。

### 知识点

**每个任务有独立的栈**，用来存局部变量和函数调用信息。**栈不够会溢出**——后果是**破坏相邻内存**，表现为：

- "我没改代码，但行为变了"
- 随机死机 / 重启
- 某个变量莫名其妙被改

**栈水位（High Water Mark）**：记录该任务**历史上栈最多用到过多少**。用法：

```c
UBaseType_t hw = uxTaskGetStackHighWaterMark(task_handle);
/* hw = 剩余的栈空间（words），越小说明用得越多 */
```

**判断标准**：
- 剩余 < 20% → 栈偏小，有溢出风险
- 剩余 40%~80% → 合理
- 剩余 > 85% → 浪费，可以改小

### 操作步骤

**1. 在 CubeMX 里确保 `configCHECK_FOR_STACK_OVERFLOW` 已开启**（`FREERTOS` → `Advanced settings` 或 `Config parameters`）

**2. 获取每个任务的栈水位**

需要任务的句柄。用 CubeMX 创建的任务，句柄在 `freertos.c` 里（如 `defaultTaskHandle`）；自己创建的用 `osThreadNew` 的返回值。

```c
/* 打印所有任务的栈水位 */
void vTaskList_print(void)   /* 可放在一个低优先级监控任务里 */
{
    printf("Task         StackHW(words)  Status\r\n");
    printf("LedTask      %4u\r\n", (unsigned)uxTaskGetStackHighWaterMark(led_id));
    printf("ReportTask   %4u\r\n", (unsigned)uxTaskGetStackHighWaterMark(rep_id));
    printf("KeyTask      %4u\r\n", (unsigned)uxTaskGetStackHighWaterMark(key_id));
    printf("ProdTask     %4u\r\n", (unsigned)uxTaskGetStackHighWaterMark(prod_id));
    printf("ConsTask     %4u\r\n", (unsigned)uxTaskGetStackHighWaterMark(cons_id));
}
```

**3. 记录数据并调优**

先跑 5 分钟（让各任务都跑到过最坏情况），记录水位：

| 任务 | 初始栈大小(words) | 剩余水位(words) | 使用率 | 调整后栈大小 | 调整后水位 |
|---|---|---|---|---|---|
| LedTask | 256 | | | | |
| ReportTask | 256 | | | | |
| KeyTask | 256 | | | | |
| ProdTask | 256 | | | | |
| ConsTask | 256 | | | | |

**使用率 = (初始 - 剩余) / 初始**，目标落在 40%~80%。

调整：改 CubeMX 里的 Stack Size，或改 `osThreadNew` 的参数。

### 验证方法

- 5 个任务的水位都有记录
- 调整后使用率落在 40%~80%
- 调整后功能仍正常（连续跑 10 分钟）
- 能讲清"栈溢出会有什么后果"（破坏相邻内存，出现"代码没改但行为变了"）

### 常见问题

| 现象 | 原因 | 怎么办 |
|---|---|---|
| 水位返回 0 或异常大 | 句柄无效（任务创建失败）| 检查 `osThreadNew` 返回值 |
| 任务老崩 | 栈溢出 | 加大栈；开启 overflow 检查钩子函数 |
| RAM 不够 | 总栈占用太多 | 把任务栈都压到合理值；减小堆 |

---

<a id="p09-b4"></a>
## B4 · 移植 P08 采集功能，30 分钟连续运行

**任务**：把 P08 的 ADC 采集功能移植成一个独立任务，与显示任务、上报任务组成完整的数据采集器，做一次 30 分钟连续运行记录。

### 操作步骤

**任务划分建议**：

| 任务 | 职责 | 周期 | 优先级 |
|---|---|---|---|
| `AdcTask` | 采集 ADC，投入队列 | 100ms | Normal |
| `ReportTask` | 从队列取数据，串口上报 | 事件驱动 | BelowNormal |
| `LedTask` | 心跳灯 | 500ms | Low |
| `MonitorTask` | 每 60s 打印统计（栈水位、队列深度、各任务计数）| 60s | Low |

**代码要点**：

```c
/* AdcTask：生产者 */
void AdcTask(void *argument)
{
    sensor_msg_t msg;
    for (;;)
    {
        msg.adc_raw = adc_app_read_raw();
        msg.tick    = HAL_GetTick();
        osMessageQueuePut(adc_q, &msg, 0, pdMS_TO_TICKS(5));
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}

/* MonitorTask：运维监控（便于发现长期运行的问题） */
void MonitorTask(void *argument)
{
    for (;;)
    {
        safe_printf("\r\n---- t=%lus ----\r\n", (unsigned long)(HAL_GetTick()/1000));
        safe_printf("queue depth  = %lu\r\n", (unsigned long)osMessageQueueGetCount(adc_q));
        safe_printf("led HW=%u  adc HW=%u  rep HW=%u\r\n",
                    (unsigned)uxTaskGetStackHighWaterMark(led_id),
                    (unsigned)uxTaskGetStackHighWaterMark(adc_id),
                    (unsigned)uxTaskGetStackHighWaterMark(rep_id));
        safe_printf("free heap    = %lu\r\n", (unsigned long)xPortGetFreeHeapSize());
        safe_printf("adc=%lu rep=%lu err=%lu\r\n",
                    (unsigned long)adc_cnt, (unsigned long)rep_cnt, (unsigned long)err_cnt);
        vTaskDelay(pdMS_TO_TICKS(60000));
    }
}
```

**30 分钟连续运行记录表**（每 5 分钟记一次）：

| 时间 | 采集数 | 上报数 | 丢包 | 错帧 | 剩余堆 | 最小栈水位 | 异常现象 |
|---|---|---|---|---|---|---|---|
| 5 min | | | | | | | |
| 10 min | | | | | | | |
| … | | | | | | | |
| 30 min | | | | | | | |

**重点观察三个指标**：

| 指标 | 正常表现 | 异常含义 |
|---|---|---|
| **剩余堆**（`xPortGetFreeHeapSize`）| 基本稳定 | 持续下降 = **内存泄漏** |
| **栈水位** | 基本稳定 | 持续下降 = 栈不够 |
| **采集数 vs 上报数** | 差值稳定 | 差值持续变大 = 消费跟不上 |

### 验证方法

- 30 分钟连续运行**不死机、不重启、无任务卡死**
- 三个关键指标（堆、栈、队列）**全程稳定**
- 有完整的 10 行记录表
- 串口日志完整保留作为证据

---

<a id="p09-b5"></a>
## B5 · 任务关系图与优先级说明

**任务**：在 README 里画出任务与队列的关系图（哪个任务生产、哪个任务消费、哪个任务上报），并说明每个任务的优先级设定理由。

### 知识点

**优先级怎么定**——三条原则：

| 原则 | 说明 | 本项目体现 |
|---|---|---|
| **越"急"越高** | 错过就有损失的（采集、控制）优先级高 | AdcTask 高于 ReportTask |
| **越"慢"越低** | 慢也无所谓的（打印、灯）优先级低 | MonitorTask 最低 |
| **别都用同一个** | 全同优先级会导致调度不确定 | 分层设置 |

**⚠️ 不要乱设紧急优先级**：低优先级任务可能被永久饿死。

### 操作步骤

画关系图（ASCII 或手绘）：

```
                    ┌─────────────┐
                    │   AdcTask   │  Prior=Normal (osPriorityNormal)
                    │  (生产者)   │  周期 100ms
                    └──────┬──────┘
                           │ osMessageQueuePut
                           v
                    ┌─────────────┐
                    │   adc_q     │  深度 8，元素 sensor_msg_t
                    └──────┬──────┘
                           │ osMessageQueueGet
                           v
                    ┌─────────────┐
                    │ ReportTask  │  Prior=BelowNormal
                    │  (消费者)   │  事件驱动 + safe_printf
                    └─────────────┘

     独立任务（不与队列交互）：
     ┌─────────────┐   ┌──────────────┐
     │  LedTask    │   │ MonitorTask  │  Prior=Low
     │ 心跳灯500ms │   │ 每60s统计    │
     └─────────────┘   └──────────────┘
```

README 里写清：

```markdown
## 任务优先级设定理由

| 任务 | 优先级 | 理由 |
|---|---|---|
| AdcTask | Normal | 采集有时效性，错过就丢数据，需要及时运行 |
| ReportTask | BelowNormal | 上报稍慢可接受，且它有队列缓冲，不会丢数据 |
| LedTask | Low | 心跳灯纯指示，晚一点无所谓 |
| MonitorTask | Low | 每 60s 才跑一次，且执行时间短 |

**为什么不让 AdcTask 用更高优先级**：100ms 周期本身很宽松，
Normal 优先级已足够保证不丢；设太高反而可能饿死上报任务。

**为什么 ReportTask 不放 Lowest**：它要消费队列，太低会导致
队列积压（生产者 10Hz，队列深度 8，积压超过 0.8 秒就开始丢包）。
```

### 验证方法

- 关系图清楚标出生产者/消费者/队列
- 每个任务的优先级都有理由（不是"随便设的"）
- 能回答："如果把 AdcTask 优先级设成最高会怎样？"（答：可能饿死上报任务，导致队列积压丢包）
- 能回答："为什么 MonitorTask 可以设最低？"（答：60s 才跑一次，晚点无所谓）

---

<a id="p09-irq"></a>
## 把 P04 / P06 的中断搬进 FreeRTOS

P04 的 EXTI 按键中断、P06 的串口接收中断，在裸机里"中断里直接改全局变量 / 打印"就行；搬进 RTOS 之后，中断里**不能再直接操作任务级的东西**（不能 `osDelay`、不能等队列、不能 `printf`），必须遵守下面三条。

### 1. NVIC 优先级：数值 ≥ 5 才能在中断里调 FreeRTOS API

```
STM32F103 + CubeMX 默认：NVIC 优先级分组 = 4（4 位抢占优先级，0~15，数值越小越急）
FreeRTOSConfig.h 默认：configMAX_SYSCALL_INTERRUPT_PRIORITY = 5
```

- FreeRTOS 用 `BASEPRI` 把"抢占优先级数值 ≥ 5"的中断挡在临界区之外，所以这些中断**可以**调用 `...FromISR` 版 API；
- 抢占优先级数值 **0~4** 的中断**永远不会**被 FreeRTOS 屏蔽，它们**绝对不能**调用任何 FreeRTOS API（包括 `...FromISR`）；
- 结论：**任何调用 FreeRTOS API 的 ISR，其抢占优先级数值必须 ≥ 5**（数值越大 = 优先级越低）；
- P04 里把 EXTI 配成 **6**，就是为了满足这条规则。如果按默认 0 配，运行时要么被 `configASSERT` 抓到，要么直接静默 HardFault。

| 中断 | 抢占优先级 | 能调 FreeRTOS API 吗 |
|---|---|---|
| SysTick | 15（最低）| 由内核 / RTOS 自己管，**不要改** |
| EXTI（P04 按键）| 6 | ✅ 可以，用 `...FromISR` |
| USART1（P06 串口）| 6 | ✅ 可以，用 `...FromISR` |
| 优先级 0~4 的 ISR | 0~4 | ❌ 绝对不行 |

> 配置位置：CubeMX 的 `System Core → NVIC` 里逐项设抢占优先级；`configMAX_SYSCALL_INTERRUPT_PRIORITY` 在 `FreeRTOSConfig.h` 里（CubeMX 生成的默认值 5，不要为了"调高响应速度"去改小它，除非你完全清楚后果）。

### 2. 中断里只能用 `...FromISR` 版本

| 场景 | 任务里用 | 中断里用 |
|---|---|---|
| 发消息到队列 | `osMessageQueuePut(q, &msg, 0, 0)` | `osMessageQueuePutFromISR(q, &msg, 0, &woken)` + `portYIELD_FROM_ISR(woken)` |
| 释放信号量 | `osSemaphoreRelease(sem)` | `...FromISR` 版本（或 `xSemaphoreGiveFromISR`）|
| 延时 | `osDelay(ms)` | ❌ 中断里绝对不能延时 |
| 打印 | `printf(...)` | ❌ 中断里不要打印（慢、不可重入）|

核心原则：**中断里只做"投递/通知"，把耗时的活留给任务**。

**示例：P06 的 UART 接收回调，把收到的字节投进队列**

```c
/* 队列与接收缓冲（任务里创建/初始化） */
osMessageQueueId_t uart_q;
uint8_t            g_rx_byte;

/* 任务侧：普通 Put，允许阻塞等待 */
osMessageQueuePut(uart_q, &byte, 0, 0);

/* 中断侧：必须用 FromISR 版本，且不能阻塞（超时只能填 0） */
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance != USART1) return;

    uint32_t woken = 0;
    osMessageQueuePutFromISR(uart_q, &g_rx_byte, 0, &woken);

    /* 如果有更高优先级任务因这次投递变为就绪，请求一次上下文切换 */
    portYIELD_FROM_ISR(woken);

    HAL_UART_Receive_IT(&huart1, &g_rx_byte, 1);   /* 重新挂上接收 */
}
```

> **封装差异提醒**：`...FromISR` 是 FreeRTOS 原生风格的 API。CMSIS-RTOS2 的各家封装对它的导出情况不完全一致（有的版本只提供 `osMessageQueuePut`，而 STM32 的 FreeRTOS 封装里提供的可能是 `osMessageQueuePutFromISR`）。**如果你的工程里编译报"未定义"，就用原生版本，语义完全一样**：
>
> ```c
> /* include "FreeRTOS.h"、"task.h"、"queue.h" */
> BaseType_t woken = pdFALSE;
> xQueueSendFromISR((QueueHandle_t)uart_q, &g_rx_byte, &woken);
> portYIELD_FROM_ISR(woken);
> ```
>
> 判断标准只有一条：**中断里绝不调用会阻塞、或会等超时的任务级 API**。

### 3. 打开 configASSERT / USE_FULL_ASSERT

**现象**：`configASSERT` 没定义时，在优先级 0~4 的 ISR 里调用 FreeRTOS API **不会报错**，表现为随机的死机、HardFault 或"莫名其妙的行为"——最难查的一类问题。

**做法**：

1. CubeMX → `Project Manager → Code Generator` 勾选 **`Enable Full Assert`**（生成 `USE_FULL_ASSERT`）；
2. 确认 `FreeRTOSConfig.h` 里 `configASSERT` 有定义（CubeMX 默认生成的形式类似）：

```c
#define configASSERT(x)  if ((x) == 0) { taskDISABLE_INTERRUPTS(); for (;;); }
```

3. 同时把 `assert_param()` 接到你自己的死循环/打印上；出错时能停在断言处，比随机死机好查一百倍。

### 验证方法

- 在 `NVIC` 里能看到 EXTI / USART1 的抢占优先级都是 6
- 故意把一个会调 FreeRTOS API 的 ISR 优先级改成 4，运行时能被 `configASSERT` 抓住（验证完改回去）
- 串口中断投递数据 → 任务从队列取出并打印，长时间运行不乱
- 能讲清"为什么中断里不能用 `osMessageQueuePut`（阻塞版）、只能用 `...FromISR`"

### 常见问题

| 现象 | 原因 | 怎么办 |
|---|---|---|
| 中断一触发就死机 / HardFault | ISR 里调了非 FromISR API，或优先级 < 5 | 改成 `...FromISR`；优先级改成 ≥ 5 |
| 断言停在 `vPortValidateInterruptPriority` | ISR 抢占优先级数值太小（0~4）| 把该中断优先级改到 5~15 |
| 中断里 `printf` 导致乱码/卡死 | 打印不可重入且耗时 | 中断里只投队列，任务里打印 |
| 投递了但任务没反应 | 忘了 `portYIELD_FROM_ISR`（或没在任务里等队列）| 补上，并检查任务的 `osMessageQueueGet` |
| 中断里等队列超时 | 用了阻塞版 Put / 超时非 0 | 中断里超时必须填 0 |

---

## 附：FreeRTOS 常见坑速查

| 现象 | 首查项 |
|---|---|
| 任务创建失败但编译通过 | `configTOTAL_HEAP_SIZE` 够不够；检查 `osThreadNew` 返回值 |
| 行为随机变化 / 莫名重启 | **栈溢出**（用栈水位检查） |
| 任务得不到执行 | 优先级被饿死；或有 `HAL_Delay` 阻塞 |
| 串口输出交叉乱码 | 共享串口没加互斥锁 |
| 内存持续减少 | 泄漏（任务/队列创建后未释放，或多处重复创建） |
| 中断里调用普通 API 崩溃 | 必须用带 `FromISR` 后缀的版本 |

---

## 下一步

- 回到任务卡 `tasks/P09-FreeRTOS入门.md` 逐条勾选
- 交付物：源码（含任务划分与队列）、任务关系图、优先级说明、栈水位记录（调整前后）、10 分钟三任务核对表、30 分钟连续运行记录
- 照 `REVIEW_TEMPLATE.md` 准备 10 分钟演示，重点能讲：任务划分思路、优先级取舍、长期运行指标
