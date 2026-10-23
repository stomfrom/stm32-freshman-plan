# P03 手把手教程：GPIO 与按键（轮询去抖）

> 配套任务卡：`tasks/P03-GPIO与按键.md`
> 芯片：**STM32F103C8T6**｜LED = **PC13**（低电平点亮）｜按键 = **PA0**（按下为高电平）
> 环境：STM32CubeMX + Keil MDK（HAL 库）
>
> ⚠️ 若你的板子按键/ LED 引脚不同，以板子原理图为准；本文会标出需要相应修改的地方。

---

## 目录

| 任务点 | 教程 |
|---|---|
| A1 配置下拉输入读取按键电平 | [跳转](#p03-a1) |
| A2 按下点亮 / 松开放下 | [跳转](#p03-a2) |
| A3 加 20ms 去抖，一次按下只翻转一次 | [跳转](#p03-a3) |
| A4 打印电平序列，连续按 20 次 | [跳转](#p03-a4) |
| A5 读原理图确认按键接法与电平 | [跳转](#p03-a5) |
| B1 改成非阻塞去抖 | [跳转](#p03-b1) |
| B2 100 次按压统计表 | [跳转](#p03-b2) |
| B3 记录抖动过程 | [跳转](#p03-b3) |
| B4 用位运算直接读寄存器 | [跳转](#p03-b4) |

---

<a id="p03-a0"></a>
## 0 · 开工前的准备（所有任务点共用）

### 0.1 确认板子与接线

| 你要确认的事 | 怎么做 |
|---|---|
| LED 接在哪个引脚 | 看板子丝印（蓝板一般是 `PC13`），或查原理图 |
| LED 是高电平还是低电平点亮 | 看板子原理图（LED 阳极接 VCC 还是接 GND）；或**断电后**用万用表二极管档测 LED 的方向。⚠️ **不要给已经配成输出的引脚外加 3.3V**——那等于对 IO 灌电流，可能把引脚或 LED 烧坏。蓝板 PC13 是**接 GND 才亮**（低电平点亮） |
| 按键接在哪个引脚 | 蓝板上标 `KEY` / `WK_UP` 的，一般是 `PA0`；看丝印 |
| 按键按下是什么电平 | 蓝板 PA0 按键**按下接 3.3V**（按下为高）；另一种接法（按下接地）很常见，**必须确认** |

> **为什么必须先确认这些**：GPIO 的所有流程都一样，但"按下是高还是低"决定了后面代码里的判断方向。搞反了代码看起来完全正确却毫无反应。

### 0.2 你需要的最少背景知识

**GPIO 是"通用输入输出"引脚**，每个引脚可以被配置成：

- **输出**：单片机控制引脚电平（点灯、驱动蜂鸣器）
- **输入**：单片机读取引脚电平（按键、传感器）

一个引脚同一时刻只能选一种。配置错了（该配输入却配成输出），现象是"读到的值永远是 0"或"电平完全不受外部影响"。

### 0.3 新建工程（与 P01/P02 相同流程，快速过）

1. 打开 **STM32CubeMX** → `File` → `New Project`
2. 在 `Part Number` 搜索框输入 `STM32F103C8` → 选中 `STM32F103C8Tx` → 点 `Start Project`
3. 左侧 `System Core` → `SYS` → `Debug` 选 **Serial Wire**（否则下次烧不进去）
4. 左侧 `System Core` → `RCC` → `High Speed Clock (HSE)` 选 **Crystal/Ceramic Resonator**
5. 顶部 `Clock Configuration` 标签 → 把 `HCLK` 填 **72**，回车，让工具自动算 PLL（出现红色说明有冲突，按提示调）
6. 左侧 `Connectivity` → `USART1` → `Mode` 选 **Asynchronous**（本教程要用串口打印）
   - 展开 `Parameter Settings`：`Baud Rate` = **115200**，其余默认
   - 右侧引脚图上 `PA9`（TX）、`PA10`（RX）会自动变绿
7. `Project Manager` 标签：
   - `Project Name` 填 `P03_GPIO_Key`
   - `Project Location` 选**纯英文无空格**的路径
   - `Toolchain / IDE` 选 **MDK-ARM**，版本选你装的
   - 左侧 `Code Generator` 里勾上 **Generate peripheral initialization as a pair of .c/.h files**
8. 点右上角 **GENERATE CODE** → 完成后点 `Open Project` 打开 Keil
9. 在 Keil 里打开 `Options for Target`（工具栏的"魔术棒"图标）→ `C/C++` 标签 → 勾选 **C99 Mode**
10. 同一个窗口 → `Target` 标签 → 勾选 **Use MicroLIB**

> ⚠️ **这两个勾必须现在就勾上，否则后面会卡住**：
> - 漏勾 **C99 Mode** → 编译报 `declaration may not appear after executable statement`（本教程的示例会在初始化函数调用之后声明变量）
> - 漏勾 **Use MicroLIB** → `printf` 重定向（`fputc`）的常见现象是**卡死或串口一个字都不输出**，而且不报编译错误，最难查

---

<a id="p03-a1"></a>
## A1 · 配置下拉输入读取按键电平

**任务**：配置 **PA0** 为**下拉输入**，读取板载按键电平，用串口打印出来。

### 知识点

**GPIO 四种输入模式**（CubeMX 里 `GPIO mode` 的下拉选项）：

| 模式 | 含义 | 什么时候用 |
|---|---|---|
| `Input mode` | 浮空输入：引脚电平完全由外部决定 | 外部已有上拉/下拉电阻时 |
| `Input mode` + `Pull-up` | 内部上拉：默认高电平，外部拉低才变低 | 按键一端接地（按下为低） |
| `Input mode` + `Pull-down` | 内部下拉：默认低电平，外部拉高才变高 | 按键一端接电源（按下为高） |
| `Analog` | 模拟输入 | ADC 采样时用，**数字读取时不要选这个** |

**上拉/下拉为什么必需**：引脚悬空时电平是"飘"的，读到的值随机跳动（可能受手靠近影响）。上拉/下拉电阻给它一个确定的默认电平。

**关键判断**（本项目最容易错的一步）：

```
按键另一端接 GND  → 按下为低电平 → 配 Pull-up（上拉）
按键另一端接 3.3V → 按下为高电平 → 配 Pull-down（下拉）
```

蓝板 PA0 按键通常是**按下接 3.3V**，所以配 **Pull-down**。

**为什么是下拉**：按下 → PA0 被接到 3.3V → 读 **1**；松开 → 内部下拉电阻把 PA0 拉到 GND → 读 **0**。这就是本方案「PA0 + 内部下拉 + 按下为高（读 1）」的来源。若你的板子按键接地（按下为低），则要配 `Pull-up`，并把后面所有对高/低电平的判断整个反过来。

### 操作步骤

1. 回到 CubeMX，在右侧芯片引脚图上**左键点击 `PA0`** → 弹窗选 **`GPIO_Input`**
2. 此时左侧 `System Core` 下会多出 **`GPIO`** 菜单 → 点进去
3. 在列表里找到 `PA0`，按下表设置：

   | 参数 | 设为 |
   |---|---|
   | `GPIO mode` | `Input mode` |
   | `GPIO Pull-up/Pull-down` | **`Pull-down`**（若你的按键接地，改选 `Pull-up`） |
   | `User Label` | `KEY`（可选，填了以后代码里会生成 `KEY_Pin`，更好读） |

4. 顺便把 LED 也配上（后面任务要用）：点 `PC13` → 选 **`GPIO_Output`** → 在 GPIO 列表里把 PC13 的 `User Label` 填 `LED`
5. 点 **GENERATE CODE** 重新生成

**CubeMX 会为你生成什么**（在 `main.c` 的 `MX_GPIO_Init` 里）：

```c
/*Configure GPIO pin : PA0 */
GPIO_InitStruct.Pin = KEY_Pin;
GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
GPIO_InitStruct.Pull = GPIO_PULLDOWN;   /* 你选的下拉 */
HAL_GPIO_Init(KEY_GPIO_Port, &GPIO_InitStruct);
```

### 写代码：读按键并打印

在 `main.c` 里找到这几处（**只写在 `USER CODE BEGIN/END` 之间**）：

**第一步**：文件顶部 `/* USER CODE BEGIN Includes */` 之间加：

```c
#include <stdio.h>
```

**第二步**：`/* USER CODE BEGIN 0 */` 之间加 `printf` 重定向（P02 已学过，这里直接抄）：

```c
/* printf 重定向到 USART1（用于串口调试） */
int fputc(int ch, FILE *f)
{
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, 0xFFFF);
    return ch;
}
```

**第三步**：在 `/* USER CODE BEGIN WHILE */` 之后的 `while (1)` 里写主循环：

```c
while (1)
{
    /* 读一次按键电平 */
    GPIO_PinState key = HAL_GPIO_ReadPin(KEY_GPIO_Port, KEY_Pin);

    printf("KEY = %d\r\n", key);   /* 1=高(按下)  0=低(松开) */

    HAL_Delay(200);   /* 先每 200ms 打一次，看得清 */
}
```

> 注意：如果你的板子引脚没有配 `User Label`，代码里用 `GPIOA, GPIO_PIN_0` 代替 `KEY_GPIO_Port, KEY_Pin`。

### 验证方法

1. Keil 里按 `F7` 编译，确认 **0 Error**
2. 连上 ST-Link，按 `F8` 下载
3. 打开串口助手（波特率 **115200**，8 数据位、1 停止位、无校验）
4. 观察输出：**不按按键时打 `0`，按住时打 `1`**（蓝板下拉配置）

### 常见问题

| 现象 | 原因 | 怎么办 |
|---|---|---|
| 串口无任何输出 | 串口助手选错端口 / 波特率不对 | 拔插设备看哪个 COM 口消失，确认 115200 |
| 一直打 `0`，按下也不变 | 上下拉配反了，或按键没接到这个引脚 | **先量 PA0 电压**（不按/按下各量一次）。按本文标准接法应配 `Pull-down`（按下为高）；只有确认按键确实接地（按下为低）才改成 `Pull-up`，并把电平判断整个反过来 |
| 一直打 `1`，按下也不变 | 配成了 `Pull-up`，而按键实际接 3.3V（按下为高） | 改成 `Pull-down`（本文标准接法） |
| 读数一直在 `0` `1` 之间乱跳 | 引脚悬空（没配上下拉），或按键引脚没接 | 确认 `GPIO Pull-up/Pull-down` 不是 `No pull` |
| 编译报 `FILE` 未定义 | 没加 `#include <stdio.h>` | 补上 |
| 编译很卡 / 报 MicroLIB | 未启用 MicroLIB | Keil 里 `Options for Target` → `Target` 勾 **Use MicroLIB** |

---

<a id="p03-a2"></a>
## A2 · 按下点亮、松开放下

**任务**：实现「按下按键点亮 LED、松开熄灭」，确认按下与松开各触发一次正确结果。

### 知识点

**低电平点亮**：蓝板 LED 是 `PC13 → 电阻 → LED → VCC`，所以**输出低电平（0）时电流流过、LED 亮**。很多人写反了，导致"按下反而灭"。

```
HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_RESET);  // 输出低 → 点亮
HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_SET);    // 输出高 → 熄灭
```

**为什么要判断电平而不是"翻转"**：这个任务是"状态跟随"——按键按着灯就亮，松开就灭。正确的做法是把按键电平**映射**到 LED 上，而不是每次检测到就翻转。

### 操作步骤

把 A1 的主循环改成：

```c
/* 放在 while(1) 之前 */
GPIO_PinState last_key = GPIO_PIN_RESET;   /* 上一次的按键电平 */

while (1)
{
    GPIO_PinState key = HAL_GPIO_ReadPin(KEY_GPIO_Port, KEY_Pin);

    if (key == GPIO_PIN_SET)   /* 按下（高电平） */
    {
        HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_RESET);  /* 点亮 */
    }
    else                        /* 松开（低电平） */
    {
        HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_SET);    /* 熄灭 */
    }

    /* ⭐ 只在电平"变化"时打印：115200 下一帧 printf 要 1~2ms，
     *    每轮都打会刷满串口、还会拖慢按键响应 */
    if (key != last_key)
    {
        last_key = key;
        printf("KEY=%d  LED=%s\r\n", key,
               (key == GPIO_PIN_SET) ? "ON" : "OFF");
    }
}
```

> 如果你的按键是**按下接地（低电平）**，把条件反过来写：`if (key == GPIO_PIN_RESET)`。

### 验证方法

- 按住按键不放 → LED **持续亮**；松手 → **立即灭**。
- 串口打印的 `LED=ON/OFF` 与眼睛看到的一致。
- **按住不放时不应该闪烁**——如果闪烁，说明你的板子按键有硬件/软件重复触发问题，记下来，A3 会解决。

---

<a id="p03-a3"></a>
## A3 · 加 20ms 去抖，一次按下只翻转一次

**任务**：在轮询逻辑里加入 20ms 软件延时去抖，实现「一次按下只翻转一次 LED 状态」。

### 知识点

**机械抖动**：按键内部的金属弹片在接触瞬间会**弹跳**多次，让引脚电平在几毫秒内高高低低抖好几下。所以"按一次"在单片机看来可能是"按了七八次"。

```
理想情况：  ‾‾‾‾‾‾‾‾‾‾‾‾    一次干净的下降
实际情况：  ‾_‾_‾‾_‾_______  前几毫秒在抖动
            ↑ 这段叫抖动期，典型 0.5~5ms
```

**软件去抖的思路**：检测到电平变化后**等 20ms 再读一次**。抖动期早就过去了，读到的就是稳定值。

**为什么 20ms**：抖动典型不超过 5ms，20ms 有充足余量；同时远小于人手最快动作（约 100ms），用户感觉不到延迟。

**为什么用"翻转"**：这个任务是"按一次切换一次灯"（像电灯开关），所以用 `HAL_GPIO_TogglePin`。

### 操作步骤

```c
while (1)
{
    /* 检测到"按下"（低→高的那一刻） */
    if (HAL_GPIO_ReadPin(KEY_GPIO_Port, KEY_Pin) == GPIO_PIN_SET)
    {
        HAL_Delay(20);   /* ① 等抖动过去 */

        /* ② 再确认一次，仍然是按下才算数 */
        if (HAL_GPIO_ReadPin(KEY_GPIO_Port, KEY_Pin) == GPIO_PIN_SET)
        {
            HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin);   /* 翻转一次 */
            printf("KEY PRESSED -> toggle\r\n");

            /* ③ 等松手，防止按住不放被当成连续多次按下 */
            while (HAL_GPIO_ReadPin(KEY_GPIO_Port, KEY_Pin) == GPIO_PIN_SET)
            {
                HAL_Delay(10);
            }
            HAL_Delay(20);   /* 松手也要去抖 */
        }
    }
}
```

**三步缺一不可**：
1. **等 20ms** —— 跳过抖动期
2. **再确认** —— 确认是稳定按下，不是抖动
3. **等松手** —— 否则按住期间每轮循环都翻转一次，灯会疯狂闪

### 验证方法

- 按一次 → 灯翻转一次；再按一次 → 再翻转一次。
- **按住不放，灯应该只翻转一次**，不能持续闪。
- 用串口看 `KEY PRESSED` 的行数：按 10 次应该正好打印 10 行。

### 常见问题

| 现象 | 原因 | 怎么办 |
|---|---|---|
| 按一次灯翻转好几次 | 漏了"再确认"或"等松手" | 对照上面三步检查 |
| 按住不放灯疯狂闪 | 没有第 ③ 步的等松手循环 | 补上 |
| 反应迟钝 | 延时给太长（比如 200ms） | 改回 20ms |
| 完全不翻转 | 第一次判断的条件与你的按键电平不符 | 若按键按下为低，条件改成 `== GPIO_PIN_RESET` |

---

<a id="p03-a4"></a>
## A4 · 打印电平序列，连续按 20 次

**任务**：把按键原始电平序列打印到串口（每次电平变化打一行，带序号），连续按 20 次。

### 知识点

**为什么要打"变化"而不是"每次采样"**：如果每轮循环都打印，一秒能刷几百行，根本看不清；只在**电平发生变化时**打印一行，日志就变成了一条清晰的"事件流"，一眼能数出按了几次。

**这行日志的用途**：它记录的是**原始**电平变化（本步**不做**去抖），是后面 B2、B3 的基础素材。评审时会看你这份日志。**A3 去抖到底有没有效果，用 B2 的 100 次统计来证明，不靠本步的日志行数。**

### 操作步骤

```c
/* 放在 while(1) 之前 */
GPIO_PinState last_key = GPIO_PIN_RESET;   /* 上一次的电平 */
uint32_t      seq      = 0;                /* 变化序号 */

while (1)
{
    GPIO_PinState key = HAL_GPIO_ReadPin(KEY_GPIO_Port, KEY_Pin);

    /* 只在电平变化时打印 */
    if (key != last_key)
    {
        seq++;
        printf("#%03lu  t=%lu ms  KEY=%d\r\n",
               (unsigned long)seq,
               (unsigned long)HAL_GetTick(),
               key);
        last_key = key;
    }

    HAL_Delay(5);   /* 5ms 采样一次，比抖动周期稍大，便于观察 */
}
```

**`HAL_GetTick()`** 返回开机以来的毫秒数，用来给每行打时间戳——**这是后面数抖动次数的关键**。

### 验证方法：做一张记录表

连续按 20 次，从串口日志里填这张表（放进你的 README）：

| 按压序号 | 日志行数 | 是否有多余行 | 备注 |
|---|---|---|---|
| 1 | | | |
| 2 | | | |
| … | | | |
| 20 | | | |

**合格标准**：
- 一次按压通常对应 **2 行**（一次按下 + 一次松开），20 次共 40 行左右
- **抖动期多打几行是正常的**——本步就是故意不去抖；抖动有什么样、持续多久，正好用这些多余的行走查（B3 会用到）
- 同一按压的按下与松开时间戳之差 ≈ 你按住的实际时长
- **如果一次按压恰好只剩 2 行**，说明这次的抖动没被 5ms 采样捕捉到，不能据此认为"去抖有效"——去抖有效性看 B2

> 想让 A4 的日志"一次按压正好 2 行"也能做到：把本步的判据从"原始电平变化"换成 A3 的"稳定 20ms 后才确认的变化"，再跑一遍对比两份日志。这是可选的进阶做法，任务卡不强制。

### 常见问题

- **行数远超 40**：本步不去抖，抖动期多打几行属正常；若你想区分"真抖动"和"重复触发"，把采样间隔从 5ms 改成 1ms 再对比，或参照 A3 的写法加 20ms 确认后再打印。
- **序号跳号**：正常，序号是"变化次数"不是"按压次数"。
- **中文乱码**：串口助手编码设为 UTF-8，或把日志里的中文改成英文。

---

<a id="p03-a5"></a>
## A5 · 读原理图确认按键接法与电平

**任务**：打开开发板原理图，找到按键接法，确认引脚、上下拉电阻，在 README 里说明按下/松开的电平。

### 知识点

**原理图怎么看按键部分**：

1. 先搜/keyword：在原理图 PDF 里搜 `KEY`、`BUTTON`、`SW`、`WK_UP`
2. 找到按键符号（一般是两个触点加一个斜杠），看它**哪一端接地/接电源**
3. 顺着引脚找 `PA0` 之类的网络标号

**两种典型接法对照**：

| 接法 | 电路 | 按下时引脚 | 应配 |
|---|---|---|---|
| 一端接地 | `PA0 ──按键── GND`，另配 **上拉电阻到 3.3V** | **低电平** | Pull-up |
| 一端接电源 | `PA0 ──按键── 3.3V`，另配 **下拉电阻到 GND** | **高电平** | Pull-down |

**上拉/下拉电阻的作用**：按键没按时，引脚不能悬空——电阻给它一个确定的默认电平。阻值常见 **10kΩ**。

### 操作步骤

1. 找到你的板子对应的原理图 PDF（买板子时卖家一般会提供，或搜"STM32F103C8T6 最小系统板 原理图"）
2. 定位按键与 LED 两个部分，截图
3. 在截图/文字里标出：
   - 按键接在哪个引脚
   - 另一端接的是 GND 还是 3.3V
   - 有没有独立的上拉/下拉电阻，阻值多少
   - LED 接在哪个引脚，是低电平点亮还是高电平点亮
4. 写进 README（复制这张**空模板**，自己填）：

```markdown
## 硬件确认（对照原理图）

| 项 | 结论（自己填） | 依据（自己填） |
|---|---|---|
| 按键引脚 |  |  |
| 按键另一端接的是 |  |  |
| 上下拉电阻 |  |  |
| 按下时电平 |  |  |
| 松开时电平 |  |  |
| LED 引脚 |  |  |
| LED 点亮条件 |  |  |
```

> **填法示例**（只示范格式，不是你的答案）：
> `| 按键引脚 | PA0 | 原理图网络标号 KEY，截图见 evidence/key-sch.png |`
>
> ⚠️ **结论必须来自你自己查的原理图和万用表实测**，照抄示例、照抄同学、照抄网上的"标准答案"都**不算完成**——评审会追问"你是在原理图第几页/哪个网络标号看到的"，答不上来这一条就不过。

5. 用万用表验证（可选但推荐）：**不按**时量 PA0 对地电压 ≈ 0V，**按住**时 ≈ 3.3V

### 验证方法

评审时会被要求**对着原理图讲清**：为什么按下是高电平、上拉电阻在哪、LED 为什么是低电平点亮。**讲得出来才算过关**。

---

<a id="p03-b1"></a>
## B1 · 改成非阻塞去抖

**任务**：把阻塞式 20ms 延时去抖改写成"两次采样间隔确认"的非阻塞写法，主循环里不再有任何延时。

### 知识点

**阻塞 vs 非阻塞**：

```c
HAL_Delay(20);   /* 阻塞：CPU 在这 20ms 里什么都不干，干等 */
```

如果一个系统要同时做"按键 + 刷屏 + 上报数据"，`HAL_Delay` 会让其他任务全部卡住。

**非阻塞的做法**：把"等 20ms"改成"记下时间戳，下次循环再来看够不够 20ms"。CPU 在等待期间可以处理别的事。

```
阻塞式：  按键 → 死等20ms → 处理 → 按键 → 死等20ms → ...
非阻塞式：按键 → 记时间 → 干别的 → 按键 → 时间够了吗? → 处理 → ...
              └────────── 这期间 CPU 是自由的 ──────────┘
```

### 操作步骤

```c
/* 状态变量（放在 while(1) 之前） */
uint8_t  key_state   = 0;        /* 0=松开确认态  1=按下确认态 */
uint32_t key_tick    = 0;        /* 上次电平变化的时间戳 */
GPIO_PinState last_raw = GPIO_PIN_RESET;

while (1)
{
    GPIO_PinState raw = HAL_GPIO_ReadPin(KEY_GPIO_Port, KEY_Pin);
    uint32_t now = HAL_GetTick();

    /* 电平刚发生变化 → 只记时间，不做动作 */
    if (raw != last_raw)
    {
        last_raw = raw;
        key_tick = now;
    }

    /* 电平已稳定保持 20ms → 才认为是有效动作 */
    if ((now - key_tick) >= 20)
    {
        if (key_state == 0 && raw == GPIO_PIN_SET)   /* 松开→按下：确认一次按下 */
        {
            key_state = 1;
            HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin);
            printf("PRESS  t=%lu\r\n", (unsigned long)now);
        }
        else if (key_state == 1 && raw == GPIO_PIN_RESET)   /* 按下→松开 */
        {
            key_state = 0;
            printf("RELEASE t=%lu\r\n", (unsigned long)now);
        }
    }

    /* ↓ 这里可以自由地做别的事，没有任何 HAL_Delay ↓ */
}
```

**要点**：
- 用 `now - key_tick` **不要**用 `now > key_tick + 20`——时间戳溢出时会出错（`HAL_GetTick` 是 32 位，约 49 天回绕一次）
- `key_state` 用来记住"当前是否已判定按下"，避免重复触发

### 验证方法

- 功能与 A3 完全一致（按一次翻转一次、按住只翻转一次）
- **在 `main` 的 `while(1)` 里搜 `HAL_Delay`，应该搜不到**（这是验收标准之一）
- 非阻塞带来的好处可以顺手验证：在循环里加一个心跳打印，会发现按键响应和心跳打印**互不影响**——这就是非阻塞的意义。注意**不要**写成 `if ((now % 500) == 0)`：同一毫秒内可能重复打印，循环稍慢又会整个毫秒跳过去，心跳时间不均匀。要用"差值判断"：

  ```c
  static uint32_t last_hb = 0;
  if (now - last_hb >= 500) { last_hb = now; printf("alive\r\n"); }
  ```

---

<a id="p03-b2"></a>
## B2 · 100 次按压统计表

**任务**：连续按 100 次，记录 LED 实际翻转次数与误触发次数，算出正确率。

### 知识点

**"误触发"指什么**：一次按压导致 LED 翻转了 **2 次或以上**（多翻等于白按），或者按了却没反应。两者都算误触发。

**为什么要做 100 次**：单次测试可能恰好蒙对。100 次能暴露偶发的抖动漏网——这才是"去抖真的有效"的证明。

### 操作步骤

1. 在代码里加计数器，**让单片机自己数**（比人眼数可靠得多）：

```c
uint32_t toggle_cnt = 0;    /* LED 实际翻转次数 */
uint32_t press_cnt  = 0;    /* 检测到的按下次数 */

/* 在"确认按下"的分支里： */
press_cnt++;
HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin);
toggle_cnt++;
printf("PRESS #%lu  ticks=%lu\r\n", (unsigned long)press_cnt, (unsigned long)now);

/* 每隔一段时间（或用按键触发）打印汇总： */
printf("== STAT: press=%lu toggle=%lu ==\r\n",
       (unsigned long)press_cnt, (unsigned long)toggle_cnt);
```

2. 有节奏地按 100 次（每次按实、松开彻底），结束后记录两个数字
3. 填表并计算：

| 指标 | 计算方式 | 你的数值 |
|---|---|---|
| 实际按压次数（人记） | 你数的 | 100 |
| 单片机检测到按下 | `press_cnt` | |
| LED 翻转次数 | `toggle_cnt` | |
| **误触发次数** | `|toggle_cnt - 100|` | |
| **正确率** | `(1 - 误触发/100) × 100%` | |

**验收标准**：误触发 ≤ 2 次（正确率 ≥ 98%）。

4. 把表放进 README，并写一句结论（例如"100 次按压中 LED 翻转 100 次，误触发 0 次，正确率 100%"）

### 常见问题

- **`toggle_cnt` 比 100 大很多**：去抖没生效，或"等松手"逻辑缺失——回头看 A3/B1
- **`toggle_cnt` 比 100 小**：按太快，两次按压间隔小于 20ms 的确认时间；放慢节奏重测
- **两个数差 1~2**：正常，属于手速与采样时序的边界情况，在记录里注明即可

---

<a id="p03-b3"></a>
## B3 · 记录抖动过程

**任务**：用示波器（或串口打点日志）记录一次按键从松开到稳定的完整电平变化过程，数出抖动次数。

### 知识点

**什么是抖动**：机械触点闭合瞬间的多次弹跳。表现出来就是"按下"的那几毫秒里，电平在高低之间跳了好几下。

**抖动的典型特征**：
- 持续时间 **0.5~5ms**
- 跳出的是**不规则的**几次高低跳变
- 松手时同样会有抖动

### 方法一：示波器（有示波器就用这个）

1. 探头接到**按键引脚**（如 `PA0`），夹子接到板子的 **GND**
2. 探头衰减设为 **×1**（测 3.3V 不需要 ×10）
3. 触发设置：
   - `Trigger Mode` = **Normal**
   - `Trigger Edge` = **上升沿**（按键按下是高电平）
   - `Trigger Level` ≈ **1.6V**（3.3V 的一半）
4. 时基（`Time/Div`）调到 **1ms/格**，这样 5ms 抖动大约占 5 格，看得清
5. 按一次按键，示波器会停住（单次捕获）。用 `Single` 模式最好
6. 数一数：**在电平从低到高稳定的过程中，中间跳了几次**

记录格式：

```
通道：PA0（按键）   触发电平：1.6V 上升沿   时基：1ms/div
抖动持续时长：约 ____ ms
抖动次数：约 ____ 次
去抖设计（20ms）是否足够覆盖：是/否
```

### 方法二：串口打点（没有示波器）

用**尽可能小的采样间隔**打印，把抖动期"拍"下来：

```c
/* 专门用于观察抖动的循环：理论采样间隔约 1ms，
 * 但实际受 printf 耗时影响会明显大于 1ms（115200 下每行要 1~2ms） */
while (1)
{
    printf("%lu,%d\r\n", (unsigned long)HAL_GetTick(),
           HAL_GPIO_ReadPin(KEY_GPIO_Port, KEY_Pin));
    HAL_Delay(1);
}
```

把串口输出粘进 Excel：A 列时间、B 列电平，画成折线图——**抖动会表现为一小段锯齿**。

> **注意**：串口打印本身有耗时，实际采样间隔 ≥1ms 而且不均匀，所以这份日志只能看出"抖了、大概多久、大形状是什么样"，**数不准确切的抖动次数**。要数准确次数，用**定时器中断里按固定间隔采样存进数组，事后统一打印**（定时器在 P05 会系统讲；现在也可以先照 `hints/P05-定时器与PWM.md` 的 1ms 中断跑通）。

### 验证方法

产出一张抖动波形图（示波器截图或 Excel 折线图）+ 上面那张记录表，放进交付物。评审会问：**"你的去抖时间 20ms 是怎么定的？"** —— 答案要能指向你测出来的抖动时长。

---

<a id="p03-b4"></a>
## B4 · 用位运算直接读寄存器

**任务**：用位运算宏直接操作 GPIO 输入数据寄存器读按键电平（不调用 HAL 读函数），确认与 HAL 方式读到的值一致。

### 知识点

**HAL 函数底下发生了什么**：`HAL_GPIO_ReadPin` 内部其实就是**读一个寄存器的一位**。看懂这一层，你才算真的懂 GPIO。

**F103 的 GPIO 寄存器**（每个端口一组，如 GPIOA、GPIOC）：

| 寄存器 | 全称 | 作用 |
|---|---|---|
| `IDR` | Input Data Register | **输入数据**，每 1 位对应 1 个引脚，读它得到引脚电平 |
| `ODR` | Output Data Register | 输出数据，写它设置引脚输出电平 |
| `BSRR` | Bit Set/Reset Register | 原子地置位/复位某一位（HAL 的 WritePin 用它） |

**读法**：`GPIOA->IDR` 的第 0 位就是 PA0 的电平。

```c
GPIOA->IDR & (1 << 0)    /* 结果非 0 → PA0 是高电平；为 0 → 低电平 */
```

### 操作步骤

1. 在 `main.c` 的 `/* USER CODE BEGIN 0 */` 里写宏：

```c
/* 直接读 GPIO 输入寄存器的某一位：port 是 GPIOA/GPIOB/...，pin 是 0~15 */
#define GPIO_READ_BIT(port, pin)   ( ((port)->IDR >> (pin)) & 0x01U )

/* 直接写输出寄存器某一位（顺带把 A2 的操作再过一遍） */
#define GPIO_WRITE_BIT(port, pin, v)  do {                        \
        if (v) (port)->BSRR = (1U << (pin));                      \
        else   (port)->BSRR = (1U << ((pin) + 16));               \
    } while (0)
```

2. 在循环里**同时**用两种方式读，对比打印：

```c
uint8_t by_reg = GPIO_READ_BIT(GPIOA, 0);                                  /* 寄存器直读 */
uint8_t by_hal = (HAL_GPIO_ReadPin(KEY_GPIO_Port, KEY_Pin) == GPIO_PIN_SET) ? 1 : 0;

printf("reg=%d  hal=%d  %s\r\n", by_reg, by_hal,
       (by_reg == by_hal) ? "OK" : "MISMATCH!");

HAL_Delay(200);
```

> **`(1 << (pin + 16))` 是什么**：BSRR 的低 16 位用于置位、高 16 位用于复位。写高 16 位等于把对应位清零。这是 F1 系列的标准做法。

### 验证方法

- 连续运行观察：**永远只出现 `OK`，一次 `MISMATCH` 都不能有**
- 把两种读法的对比日志（至少 20 行）作为交付证据
- 评审会问：`IDR` 和 `ODR` 有什么区别？BSRR 为什么能做到"原子操作"？

### 常见问题

| 现象 | 原因 | 怎么办 |
|---|---|---|
| 编译报 `GPIOA` 未定义 | 没包含头文件 | 确认 `#include "main.h"`（CubeMX 已生成） |
| 一直 `MISMATCH` | `pin` 号码写错，或读的不是按键所在的端口 | 核对按键到底在 GPIOA 还是别的端口 |
| 宏编译报错 | 宏定义里少了括号 | 照抄上面的完整写法，参数都带括号 |

---

## 附：本项目完整代码骨架

把以下内容对照 `USER CODE` 区域填入，就是一份可运行的 P03 成品（含 B 档要点）：

```c
/* USER CODE BEGIN Includes */
#include <stdio.h>
/* USER CODE END Includes */

/* USER CODE BEGIN 0 */
int fputc(int ch, FILE *f)
{
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, 0xFFFF);
    return ch;
}

#define GPIO_READ_BIT(port, pin)   ( ((port)->IDR >> (pin)) & 0x01U )

/* USER CODE END 0 */

/* USER CODE BEGIN PV */
/* 变量统一放这里（文件作用域，在 main() 之前）：
 * ① 不会在 CubeMX 重新生成代码时被清掉；
 * ② 不会出现"在 HAL_Init() 等语句之后再声明变量"的 C90 报错
 *    （declaration may not appear after executable statement）。 */
uint8_t       key_state  = 0;
uint32_t      key_tick   = 0;
GPIO_PinState last_raw   = GPIO_PIN_RESET;
uint32_t      press_cnt  = 0, toggle_cnt = 0, seq = 0;
/* USER CODE END PV */

int main(void)
{
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_USART1_UART_Init();

    /* USER CODE BEGIN 2 */
    /* 这里只放语句：变量已经在上面 USER CODE PV 区声明好了 */
    /* USER CODE END 2 */

    while (1)
    {
        /* USER CODE BEGIN 3 */
        GPIO_PinState raw = HAL_GPIO_ReadPin(KEY_GPIO_Port, KEY_Pin);
        uint32_t now = HAL_GetTick();

        if (raw != last_raw) { last_raw = raw; key_tick = now; }

        if ((now - key_tick) >= 20) {
            if (key_state == 0 && raw == GPIO_PIN_SET) {
                key_state = 1; press_cnt++; toggle_cnt++;
                HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin);
                printf("PRESS #%lu t=%lu reg=%d\r\n",
                       (unsigned long)press_cnt, (unsigned long)now,
                       GPIO_READ_BIT(GPIOA, 0));
            } else if (key_state == 1 && raw == GPIO_PIN_RESET) {
                key_state = 0; seq++;
                printf("RELEASE #%lu t=%lu\r\n", (unsigned long)seq, (unsigned long)now);
            }
        }
        /* USER CODE END 3 */
    }
}
```

**这份骨架覆盖**：A1（读按键）、A2（状态跟随可在其基础上改）、A3/B1（非阻塞去抖）、A4（事件日志）、B4（寄存器直读）。A5 是查资料任务，B2 靠计数器统计，B3 靠示波器或采样日志。

**代码该放哪个 `USER CODE` 区**（放错会被 CubeMX 覆盖，或编译报错）：

| 内容 | 放哪个区 | 为什么 |
|---|---|---|
| 变量声明 | `USER CODE BEGIN/END PV` | 在 `main()` 之前、文件作用域；CubeMX 重新生成时不会清掉，也不会触发 C90 的"语句后声明"报错 |
| `main()` 里的语句 | `USER CODE BEGIN 2` / `BEGIN 3` | 保护区，重新生成不会覆盖 |
| 自己写的函数 | `USER CODE BEGIN 4`（`main()` 之后） | 同上；如果 `main()` 要调用它，记得在 `USER CODE BEGIN 0` 里放函数原型 |

---

## 下一步

- 回到任务卡 `tasks/P03-GPIO与按键.md` 逐条勾选
- 把代码、串口日志、波形图、原理图截图、统计表整理进 `projects/P03-GPIO与按键/`
- 照 `REVIEW_TEMPLATE.md` 准备 10 分钟演示
