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

### 操作步骤

**方式一（推荐，用 CubeMX 图形界面）**

1. `FREERTOS` → `Tasks and Queues` → `Add`：
   - `Task Name`: `LedTask`，`Priority`: `osPriorityNormal`，`Stack Size`: **256**（words）
   - 再加一个 `ReportTask`，同上配置
2. 生成的代码在 `Core/Src/freertos.c` 里

**方式二（自己写，更灵活）**

在 `main.c` 的 `/* USER CODE BEGIN 2 */` 里创建任务：

```c
#include "cmsis_os2.h"    /* CMSIS_V2 接口 */

/* 任务函数的声明 */
void LedTask(void *argument);
void ReportTask(void *argument);

/* 创建任务 */
osThreadId_t led_id = osThreadNew(LedTask, NULL, NULL);
osThreadId_t rep_id = osThreadNew(ReportTask, NULL, NULL);
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

> **用 CubeMX 方式时**：任务函数框架已生成在 `freertos.c`，你只需在 `for(;;)` 里填内容，并加 `printf` 重定向（注意要在任务启动前初始化串口）。

### 验证方法

- LED 以 1Hz 闪
- 串口每 2 秒打印一行，编号递增
- 两者**同时**工作、互不影响

### 常见问题

| 现象 | 原因 | 怎么办 |
|---|---|---|
| 只有 LED 闪，串口没输出 | 串口没初始化，或任务没创建成功 | 检查 `MX_USART1_UART_Init()` 调用顺序 |
| 任务完全没跑 | 任务创建失败（堆不足） | 检查 `osThreadNew` 返回值是否为 `NULL`；加大堆 |
| 程序卡在启动 | 栈太小 | 加大任务栈 |
| `printf` 输出交叉乱码 | 多任务同时打印（B2 解决） | 见 B2 |

---

<a id="p09-a3"></a>
## A3 · vTaskDelay 与 HAL_Delay 的区别

**任务**：用 `vTaskDelay` 控制两个任务的节拍，在 README 里说明它与裸机 `HAL_Delay` 的本质区别（一个是让出 CPU，一个是原地空等）。

### 知识点

**⭐ 这是 RTOS 最核心的一个概念**：

```
HAL_Delay(500)：         while (计数没到) { /* 原地转圈空等 */ }
                         ↑ CPU 被这一个任务独占，其他任务全停

vTaskDelay(500ms)：      把本任务标记为"阻塞"，交还 CPU 给调度器
                         调度器立刻去跑别的就绪任务
                         500ms 后本任务重新变"就绪"，等调度器安排
                         ↑ CPU 在这 500ms 里被别的任务充分使用
```

**一张对比表**：

| | `HAL_Delay()` | `vTaskDelay()` |
|---|---|---|
| CPU 在等待时 | **空转浪费** | **让给其他任务** |
| 其他任务能否跑 | ❌ 全部阻塞 | ✅ 正常运行 |
| 适用场景 | 纯裸机 | RTOS 里的任何延时 |

**在 RTOS 里用 `HAL_Delay` 会怎样**：它不会让出 CPU，会**拖住整个系统**（其他任务在此期间完全得不到执行）。个人项目可能"看起来还能跑"，但实时性完全没了。

### 操作步骤

在 README 里写一段对比说明（照上面表格组织），并**做一个小实验证明区别**：

```c
/* 实验：把 LedTask 里的 vTaskDelay 换成 HAL_Delay(500)，观察 ReportTask */
void LedTask(void *argument)
{
    for (;;)
    {
        HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin);
        HAL_Delay(500);        /* ⚠️ 实验用：会阻塞整个系统 */
    }
}
```

观察现象：`ReportTask` 的打印**间隔会变得不稳定**（被 LED 任务拖着），甚至打印节奏明显变慢/不均匀。

改回 `vTaskDelay` 后恢复正常。

| 用的延时 | ReportTask 打印间隔 | 是否均匀 |
|---|---|---|
| `HAL_Delay(500)` | 实测 ____ ms | 否 |
| `vTaskDelay(500ms)` | 实测 ____ ms | **是** |

### 验证方法

- README 里有清晰的对比说明
- 有实验数据（换用 HAL_Delay 后打印间隔变得不均匀）
- 能口头回答："为什么 RTOS 里不能用 HAL_Delay？"（答：不让出 CPU，会阻塞其他任务）

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
osThreadId_t key_id = osThreadNew(KeyTask, NULL, NULL);
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
