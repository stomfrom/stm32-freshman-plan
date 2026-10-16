# P06 手把手教程：UART 与自定义协议

> 配套任务卡：`tasks/P06-UART与自定义协议.md`
> 芯片：**STM32F103C8T6**｜LED = **PC13**（低电平点亮）｜按键 = **PA0**（按下为高电平）
> 环境：STM32CubeMX + Keil MDK（HAL 库）
>
> ⚠️ 本教程的串口使用 **USART1（PA9=TX, PA10=RX）**，波特率 **115200**。

---

## 目录

| 任务点 | 教程 |
|---|---|
| A1 串口回显 | [跳转](#p06-a1) |
| A2 printf 重定向 | [跳转](#p06-a2) |
| A3 定义帧格式 | [跳转](#p06-a3) |
| A4 状态机解析与应答 | [跳转](#p06-a4) |
| A5 5 种异常输入验证 | [跳转](#p06-a5) |
| B1 升级 CRC 校验 | [跳转](#p06-b1) |
| B2 中断+环形缓冲，100 帧 | [跳转](#p06-b2) |
| B3 协议分层 | [跳转](#p06-b3) |
| B4 协议说明文档 | [跳转](#p06-b4) |
| B5 主动周期上报 | [跳转](#p06-b5) |

---

<a id="p06-a0"></a>
## 0 · 开工前的准备

### 0.1 串口通信的基本参数

两端必须**完全一致**，否则收到乱码：

| 参数 | 值 |
|---|---|
| 波特率 | 115200 |
| 数据位 | 8 |
| 停止位 | 1 |
| 校验位 | 无 |
| 流控 | 无 |

### 0.2 为什么需要"协议"

串口本身只负责**一个字节一个字节地传**，它不知道"这一串字节是一句话"。所以要在数据外面套一层约定，叫**帧（Frame）**：

```
裸串口：  A 3 5 F 2 9 4 C ...     ← 收到一堆字节，不知道从哪切分
加帧后：  [帧头][长度][命令][数据][校验]  ← 告诉你"从这里开始，到这里结束"
```

**没有帧同步，接收方永远不知道数据从哪个字节开始**——这是所有串口协议要解决的第一个问题。

### 0.3 本项目使用的帧格式（推荐）

| 字段 | 字节数 | 说明 |
|---|---|---|
| 帧头 | 2 | **固定 `0xAA 0x55`**（双字节，降低误同步概率） |
| 长度 | 1 | 后续 DATA 的字节数（0~255） |
| 命令码 | 1 | 表示要做什么（0x01=点亮, 0x02=熄灭, 0x03=查询…) |
| 数据 | N | 载荷，长度由上面决定 |
| 校验 | 2 | CRC16（A 档先用单字节累加和，B1 升级） |

**完整一帧**：`AA 55 | LEN | CMD | DATA[N] | CRC`

---

<a id="p06-a1"></a>
## A1 · 串口回显

**任务**：实现串口收发字符串：串口助手发什么，单片机就原样回什么（回显），连续收发 30 次正确。

### 知识点

**轮询收发 vs 中断收发**：

| 方式 | 特点 | 适用 |
|---|---|---|
| 轮询 `HAL_UART_Receive` | 阻塞等待，简单 | 简单调试、数据量小 |
| 中断 `HAL_UART_Receive_IT` | 不阻塞，配合缓冲区 | 正式项目 |

A1 先用轮询，理解基本收发；B2 换成中断 + 环形缓冲。

### 操作步骤

CubeMX 确认 `USART1`：`Mode=Asynchronous`，`Baud Rate=115200`。

```c
/* USER CODE BEGIN 2 */
uint8_t rx_byte;
/* USER CODE END 2 */

while (1)
{
    /* USER CODE BEGIN 3 */
    /* 阻塞方式收 1 个字节，超时 100ms */
    if (HAL_UART_Receive(&huart1, &rx_byte, 1, 100) == HAL_OK)
    {
        HAL_UART_Transmit(&huart1, &rx_byte, 1, 100);   /* 原样发回 */
    }
    /* USER CODE END 3 */
}
```

**测试**：打开串口助手，随意输入字符，应看到原样返回。连续测 30 次。

### 验证方法

- 串口助手输入 `Hello STM32` → 返回 `Hello STM32`
- 输入 30 次不同类型内容（含中文、特殊符号）都能原样回
- 不乱码、不丢字符

### 常见问题

| 现象 | 原因 | 怎么办 |
|---|---|---|
| 全是乱码 | 波特率不一致 | 两端都设 115200 |
| 收不到任何数据 | TX/RX 接反（应交叉：TX→RX、RX→TX） | 交换两根线 |
| 丢字符 | 主循环里有耗时操作，轮询来不及 | 换中断方式（B2） |
| 只收到部分 | 超时太短 | 提高超时时间 |

---

<a id="p06-a2"></a>
## A2 · printf 重定向

**任务**：重定向 `printf` 到串口，用它打印程序运行中的关键变量，方便后续调试。

### 知识点

`printf` 默认输出到"标准输出"，单片机没有屏幕，所以要把 `fputc`（printf 底层逐字符输出函数）**改成往串口发**。

**必须勾 Use MicroLIB**：Keil 默认的标准库对单片机太重（占几十 KB Flash）。MicroLIB 是精简版。不勾会导致：
- 编译报错 / 卡死
- 或者 Flash 装不下

### 操作步骤

1. Keil → `Options for Target`（魔术棒图标）→ `Target` 标签 → 勾选 **Use MicroLIB**
2. 代码：

```c
/* USER CODE BEGIN Includes */
#include <stdio.h>
/* USER CODE END Includes */

/* USER CODE BEGIN 0 */
/* printf 重定向：printf 最终会逐字符调用 fputc */
int fputc(int ch, FILE *f)
{
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, 0xFFFF);
    return ch;
}
/* USER CODE END 0 */
```

3. 使用：

```c
int voltage = 1234;
printf("voltage=%d mV\r\n", voltage);    /* 注意 \r\n，否则串口助手不换行 */
```

### 验证方法

- 串口助手能看到格式化输出（不只是单个字符）
- 能打印整数、字符串、多个参数
- 换行正常（`\r\n` 要写全，只写 `\n` 在很多串口助手里不换行）

### 常见问题

| 现象 | 原因 | 怎么办 |
|---|---|---|
| 编译报 `__use_no_semihosting` 类错误 | 没勾 Use MicroLIB | 勾上 |
| 程序卡死 | `fputc` 里用了超时太长，或时钟/串口没初始化 | 确认初始化顺序 |
| 输出不换行 | 只写了 `\n` | 改成 `\r\n` |
| 中文乱码 | 串口助手编码不对 | 设为 UTF-8 |

---

<a id="p06-a3"></a>
## A3 · 定义帧格式

**任务**：定义一帧数据格式（帧头 + 长度 + 命令码 + 数据 + 校验），把格式写成一张表放进 README。

### 知识点

**设计帧格式要回答的四个问题**：

1. **怎么找到帧的开头？** → 帧头（固定字节序列）
2. **怎么知道帧有多长？** → 长度字段
3. **怎么知道要做什么？** → 命令码
4. **怎么知道收到的是对的？** → 校验

**双字节帧头的意义**：单字节帧头（如 `0xAA`）在数据里出现概率高，容易误同步。双字节 `AA 55` 组合出现的概率是 1/65536，大幅降低。

### 操作步骤

定义命令码（写进 README）：

| 命令码 | 名称 | 数据长度 | 说明 |
|---|---|---|---|
| `0x01` | CMD_LED_ON | 0 | 点亮 LED |
| `0x02` | CMD_LED_OFF | 0 | 熄灭 LED |
| `0x03` | CMD_GET_STATUS | 0 | 查询状态 |
| `0x04` | CMD_SET_DUTY | 1 | 设置 PWM 占空比（0~100） |

应答帧：命令码用 `原命令码 | 0x80`（如 0x81 表示对 0x01 的应答），数据里放结果。

写一个组装帧的函数：

```c
#define FRAME_HEAD1   0xAA
#define FRAME_HEAD2   0x55
#define FRAME_MAX_DATA 32

/* 组装一帧到 buf，返回整帧长度 */
uint16_t frame_build(uint8_t *buf, uint8_t cmd, const uint8_t *data, uint8_t len)
{
    uint16_t i = 0;
    buf[i++] = FRAME_HEAD1;
    buf[i++] = FRAME_HEAD2;
    buf[i++] = len;                      /* 长度（DATA 字节数） */
    buf[i++] = cmd;
    for (uint8_t k = 0; k < len; k++) buf[i++] = data[k];
    uint8_t sum = 0;                     /* A 档：单字节累加和 */
    for (uint16_t k = 2; k < i; k++) sum += buf[k];   /* 从长度字段开始累加 */
    buf[i++] = sum;
    return i;
}
```

### 验证方法

- README 里有完整的帧格式表（字段 / 字节数 / 说明 / 示例值）
- 能手算出一帧 `LED_ON` 的完整字节序列
- 字段顺序与实际 `frame_build` 代码一致

---

<a id="p06-a4"></a>
## A4 · 状态机解析与应答

**任务**：用状态机逐字节解析收到的数据：按帧头同步、按长度收齐、校验通过才算一帧有效，收到有效帧后回一帧应答。

### 知识点

**为什么用状态机**：数据是**一个字节一个字节**来的，你无法预知哪次调用会收到帧的哪一部分。状态机记住了"我收到哪儿了"：

```
       ┌──── 收到 0xAA ────┐
       v                    │
   [等帧头1] ──0xAA──> [等帧头2] ──0x55──> [收长度] ──> [收数据×N] ──> [校验]
       ^                                                                    │
       └──────────────── 校验通过→处理 / 失败→丢弃 ─────────────────────────┘
```

**关键**：任何一步出错（帧头不对、超时），**都要回到"等帧头1"**，否则会卡死。

**超时保护**：如果一帧收了一半就再没数据了，必须超时复位状态机——否则永远卡在中间状态（这是最隐蔽的坑）。

### 操作步骤

```c
/* 接收状态机状态 */
typedef enum {
    ST_HEAD1 = 0, ST_HEAD2, ST_LEN, ST_CMD, ST_DATA, ST_SUM
} rx_state_t;

static rx_state_t s_state = ST_HEAD1;
static uint8_t    s_len, s_cmd, s_data[FRAME_MAX_DATA], s_idx, s_sum;
static uint32_t   s_last_byte_tick = 0;

/* 每收到一个字节调用一次 */
void frame_feed(uint8_t b)
{
    switch (s_state)
    {
    case ST_HEAD1:
        if (b == FRAME_HEAD1) s_state = ST_HEAD2;
        break;

    case ST_HEAD2:
        s_state = (b == FRAME_HEAD2) ? ST_LEN : ST_HEAD1;
        break;

    case ST_LEN:
        if (b > FRAME_MAX_DATA) { s_state = ST_HEAD1; break; }  /* 长度非法，丢弃 */
        s_len = b; s_idx = 0; s_sum = b; s_state = ST_CMD;
        break;

    case ST_CMD:
        s_cmd = b; s_sum += b;
        s_state = (s_len > 0) ? ST_DATA : ST_SUM;
        break;

    case ST_DATA:
        s_data[s_idx++] = b; s_sum += b;
        if (s_idx >= s_len) s_state = ST_SUM;
        break;

    case ST_SUM:
        if (b == s_sum) { frame_handle(s_cmd, s_data, s_len); }   /* 校验通过 */
        else            { printf("ERR checksum\r\n"); }           /* 校验失败 */
        s_state = ST_HEAD1;                                       /* 无论如何回起点 */
        break;
    }
    s_last_byte_tick = HAL_GetTick();
}

/* 处理有效帧 */
void frame_handle(uint8_t cmd, const uint8_t *data, uint8_t len)
{
    uint8_t resp[8]; uint16_t n = 0;

    switch (cmd)
    {
    case 0x01: HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_RESET); printf("LED ON\r\n");  break;
    case 0x02: HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_SET);   printf("LED OFF\r\n"); break;
    case 0x03: printf("STATUS: led=%d\r\n", HAL_GPIO_ReadPin(LED_GPIO_Port, LED_Pin)); break;
    default:   printf("UNKNOWN cmd 0x%02X\r\n", cmd); break;
    }

    /* 回一帧应答 */
    n = frame_build(resp, cmd | 0x80, NULL, 0);
    HAL_UART_Transmit(&huart1, resp, n, 100);
}
```

主循环里接收 + 超时检查：

```c
uint8_t rx;
if (HAL_UART_Receive(&huart1, &rx, 1, 0) == HAL_OK) frame_feed(rx);

/* 超时保护：超过 100ms 没收齐一帧，复位状态机 */
if (s_state != ST_HEAD1 && (HAL_GetTick() - s_last_byte_tick) > 100)
{
    printf("TIMEOUT reset\r\n");
    s_state = ST_HEAD1;
}
```

### 验证方法

- 用串口助手发送正确帧（十六进制模式）：`AA 55 00 01 01`（LEN=0, CMD=01, 校验=0+1=01）→ LED 点亮 + 收到应答
- 连续发 30 次都正确
- 发校验错的帧 → 打印 `ERR checksum`，程序继续工作

### 常见问题

| 现象 | 原因 | 怎么办 |
|---|---|---|
| 完全没反应 | 校验和计算范围两端不一致 | 约定好"从长度字段开始累加"并写进文档 |
| 收到一半就卡住 | 没有超时复位 | 加上超时检查 |
| 偶尔误触发 | 数据里出现了帧头序列 | 用双字节帧头；必要时转义 |
| 应答变成乱码 | 应答组装时长度/校验算错 | 用 `frame_build` 统一组装 |

---

<a id="p06-a5"></a>
## A5 · 5 种异常输入验证

**任务**：构造 5 种异常输入（帧头错、长度错、校验错、半帧、超长帧）并逐一验证：程序不崩溃、能丢弃错误帧并继续接收。

### 知识点

**为什么要专门测异常**：正常帧谁都能收。**能优雅地处理错误帧**才说明协议实现是健壮的——否则现场一个干扰就让设备死机。

**核心原则**：**任何异常都必须能让状态机回到"等帧头1"**，并且**后续正常帧仍能处理**（不能"错一次就永久瘫痪"）。

### 操作步骤

用串口助手的**十六进制发送**模式，逐个构造并验证：

| # | 异常类型 | 构造的字节 | 期望现象 |
|---|---|---|---|
| 1 | 帧头错 | `BB 55 00 01 01` | 不处理，静默丢弃 |
| 2 | 长度错 | `AA 55 FF 01 01` | 长度超限 → 丢弃并复位 |
| 3 | 校验错 | `AA 55 00 01 FF` | 打印 `ERR checksum` |
| 4 | 半帧 | `AA 55 00 01`（缺校验） | 等 100ms 后打印 `TIMEOUT reset` |
| 5 | 超长帧 | `AA 55 32 ...`（LEN=50 > 32） | 长度非法 → 丢弃 |

**每种异常之后，立刻再发一帧正常帧**（`AA 55 00 01 01`），确认它**仍然能正常工作**——这一步最关键。

填表放进 README：

| 异常类型 | 发送内容 | 程序现象 | 之后的正常帧是否仍工作 |
|---|---|---|---|
| 帧头错 | | | ✅ / ❌ |
| 长度错 | | | |
| 校验错 | | | |
| 半帧 | | | |
| 超长帧 | | | |

### 验证方法

- 5 种异常全部被正确丢弃，程序不重启、不卡死
- **每种异常之后的正常帧都能正常处理**（这一条是验收重点）
- 能讲清每种异常在状态机里是怎么被"挡住"的

---

<a id="p06-b1"></a>
## B1 · 升级 CRC 校验

**任务**：把简单校验和升级成 CRC 校验（CRC8 或 CRC16），校验失败时丢弃该帧并把错误计数加一。

### 知识点

**累加和为什么不够**：累加和检测不出"字节顺序颠倒"（`1+2` 和 `2+1` 相同）和某些多字节同时出错的情况。CRC 能检测绝大多数突发错误。

**CRC16-MODBUS 参数**（必须双方一致，否则永远对不上）：

| 参数 | 值 |
|---|---|
| 初值 | 0xFFFF |
| 多项式 | 0x8005（反射形式 0xA001） |
| 输入反射 | 是 |
| 输出异或 | 0x0000 |
| 字节序 | 小端（低字节先发） |

**⚠️ 四个参数任意一个不同，校验就永远失败**——所以必须写进协议文档。

### 操作步骤

```c
/* CRC16-MODBUS（表驱动或逐位计算） */
static uint16_t crc16_modbus(const uint8_t *data, uint16_t len)
{
    uint16_t crc = 0xFFFF;
    for (uint16_t i = 0; i < len; i++)
    {
        crc ^= data[i];
        for (uint8_t b = 0; b < 8; b++)
        {
            if (crc & 0x0001) crc = (crc >> 1) ^ 0xA001;
            else              crc >>= 1;
        }
    }
    return crc;
}
```

改造接收状态机：`ST_SUM` 拆成 `ST_CRC_LO` / `ST_CRC_HI` 两态（收 2 字节）：

```c
case ST_CRC_LO:
    s_crc_rx = b;
    s_state = ST_CRC_HI;
    break;

case ST_CRC_HI:
{
    s_crc_rx |= ((uint16_t)b << 8);           /* 小端：低字节先来 */
    uint16_t calc = crc16_modbus(&s_frame[2], 2 + s_len);  /* 从长度字段开始算 */
    if (calc == s_crc_rx) { frame_handle(s_cmd, s_data, s_len); }
    else { s_crc_err++; printf("CRC ERR total=%lu\r\n", (unsigned long)s_crc_err); }
    s_state = ST_HEAD1;
    break;
}
```

（需要把"长度+命令+数据"拼进一个连续数组再算 CRC，或用支持分段的 CRC 实现。）

### 验证方法

- 正常帧通过校验
- 故意改 CRC 一个字节 → 打印 CRC 错误且**错误计数递增**
- 与在线 CRC 计算器比对，确认你的实现对得上（这一步能快速验证参数是否一致）

### 常见问题

| 现象 | 原因 | 怎么办 |
|---|---|---|
| 所有帧都校验失败 | CRC 参数与对端不一致 | 用在线计算器比对同一串数据 |
| 只对短帧有效 | CRC 计算范围（从哪字节到哪字节）不一致 | 明确写进文档 |
| 字节序反了 | 大小端不一致 | 小端=低字节先发，核对 |

---

<a id="p06-b2"></a>
## B2 · 中断 + 环形缓冲，100 帧

**任务**：用串口接收中断 + 环形缓冲区处理不定长数据，连续发送 100 帧，记录丢包数与错帧数。

### 知识点

**为什么需要环形缓冲**：串口数据来的速度可能快过处理速度。中断里必须**立刻把字节存起来**，主循环慢慢处理。环形缓冲的读写指针循环移动，天然适合这个场景。

```
      ┌─────────────────────────────┐
      │ 0  1  2  3  ...          N-1 │
      └───┬───────────────┬──────────┘
          │               │
     读指针 ┘          写指针 ┘     （都循环前进，相遇=空，追尾=满）
```

**竞态问题**：中断里动写指针，主循环动读指针。如果主循环读指针更新时被打断，可能出错。单生产者单消费者场景下，**只要保证指针操作是原子的**（16 位对齐读写），一般安全。

### 操作步骤

```c
#define RB_SIZE 128            /* 必须是 2 的幂，便于用 & 取模 */

static volatile uint8_t  rb_buf[RB_SIZE];
static volatile uint16_t rb_head = 0;   /* 写指针（仅中断改） */
static volatile uint16_t rb_tail = 0;   /* 读指针（仅主循环改） */

/* 入队（中断里调用） */
static inline void rb_put(uint8_t b)
{
    uint16_t next = (rb_head + 1) & (RB_SIZE - 1);
    if (next != rb_tail) { rb_buf[rb_head] = b; rb_head = next; }   /* 满了就丢 */
    else s_rb_overflow++;
}

/* 出队（主循环调用） */
static inline int rb_get(uint8_t *b)
{
    if (rb_tail == rb_head) return 0;      /* 空 */
    *b = rb_buf[rb_tail];
    rb_tail = (rb_tail + 1) & (RB_SIZE - 1);
    return 1;
}

/* 中断回调：每次收到 1 字节进队 */
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1) {
        rb_put(g_rx_tmp);
        HAL_UART_Receive_IT(&huart1, &g_rx_tmp, 1);   /* 重新挂上接收 */
    }
}
```

主循环：

```c
uint8_t b;
while (rb_get(&b)) frame_feed(b);    /* 把缓冲区里的字节喂给状态机 */
```

**关键**：`HAL_UART_RxCpltCallback` 里**必须重新调用 `HAL_UART_Receive_IT`**，否则只会收到一个字节就停了。

### 验证方法

- 连续发 100 帧（可写个上位机脚本或串口助手循环发送），记录：

| 指标 | 数值 |
|---|---|
| 发送帧数 | 100 |
| 成功处理帧数 | ____ |
| 丢包数 | ____ |
| 错帧数 | ____ |
| 验收（丢包≤1） | ✅/❌ |

- 高速发送时也不丢（这是环形缓冲的价值）

---

<a id="p06-b3"></a>
## B3 · 协议分层

**任务**：把协议按层拆分（物理层收发 / 帧解析层 / 命令处理层），在 README 里画出分层图并说明每层职责。

### 知识点

**为什么要分层**：现在"收字节、拆帧、执行命令"混在一起，换串口（比如改用 UART2 或 RS485）要改一大片。分层后每层各管一件事：

```
┌────────────────────────────────┐
│ 命令处理层  cmd_handler.c      │  收到合法帧后"做什么"（点灯、查状态）
│   ── 不关心字节怎么来的        │
├────────────────────────────────┤
│ 帧解析层    frame_parse.c      │  字节流 → 完整帧（同步、校验）
│   ── 不关心是串口还是别的方式  │
├────────────────────────────────┤
│ 物理层      uart_port.c        │  收发字节（中断/环形缓冲）
│   ── 换成 RS485 只改这一层     │
└────────────────────────────────┘
```

**判断分层是否到位**：命令处理层的代码里**不出现任何寄存器或串口 API**。它只收到"一个命令 + 一段数据"，然后返回"结果"。

### 操作步骤

拆成 3 组文件：

| 文件 | 职责 | 对外接口 |
|---|---|---|
| `uart_port.c/.h` | 收发字节、环形缓冲 | `uart_port_init()`、`uart_port_send(buf,len)` |
| `frame_parse.c/.h` | 状态机、CRC | `frame_feed(byte)`、注册回调 |
| `cmd_handler.c/.h` | 执行命令、组装应答 | `cmd_exec(cmd, data, len)` |

在 README 里画出上面那张分层图，并写清每层职责与"换传输方式要动哪一层"。

### 验证方法

- `cmd_handler.c` 里搜不到 `HAL_UART_` 和 `GPIOA->`（不碰底层）
- 能回答"如果把串口换成 RS485，要改哪几个文件"（答：只改 `uart_port.c`）
- 分层图与文件结构一致

---

<a id="p06-b4"></a>
## B4 · 协议说明文档

**任务**：写一份《协议说明文档》，包含帧格式表、命令码含义、校验算法说明，让另一位同学照着能实现出双方的通信。

### 知识点

**"能让别人照着实现"** 是判断文档合格的标准。对方需要知道：

1. 字节怎么排（帧格式表 + 一个真实示例）
2. 每个命令码代表什么、数据段怎么解释
3. CRC 怎么算（四个参数 + 参考代码或在线工具链接）
4. 异常怎么处理（比如校验失败对方要不要重发）

### 操作步骤

写一份 `PROTOCOL.md`（放进交付物），结构照抄：

```markdown
# 通信协议说明 v1.0

## 1. 物理层
- 接口：UART，115200-8-N-1

## 2. 帧格式
| 字段 | 长度 | 说明 |
|---|---|---|
| 帧头 | 2 | 0xAA 0x55 |
| 长度 | 1 | DATA 字节数 |
| 命令码 | 1 | 见下表 |
| 数据 | N | 载荷 |
| CRC | 2 | CRC16-MODBUS，小端 |

## 3. 命令码
| 码 | 名称 | 数据 | 应答码 |
|---|---|---|---|
| 0x01 | LED_ON | 无 | 0x81 |
| ... | | | |

## 4. 完整示例
发送点亮 LED：AA 55 00 01 <CRC低> <CRC高>
（给一个用真实 CRC 值算好的例子）

## 5. CRC 算法
参数：初值 0xFFFF / 多项式 0xA001 / 反射 / 无输出异或
参考实现：（贴 crc16_modbus 代码）

## 6. 异常处理约定
- 校验失败：接收方丢弃该帧并计数，不要求重发
- 超时：100ms 未收齐一帧则复位状态机
```

### 验证方法

- **请另一位同学照文档实现**，双方能正常通信（评审时演示这一步）
- 文档里的示例帧用真实 CRC 值，对方照着算能对上
- 文档里没出现"你懂的""同上"这类含糊表述

---

<a id="p06-b5"></a>
## B5 · 主动周期上报

**任务**：实现"命令回显应答"以外的第二种交互：单片机主动周期性上报一帧状态数据（含运行秒数与 LED 状态）。

### 知识点

**两种通信模式**：

| 模式 | 谁发起 | 场景 |
|---|---|---|
| 请求-应答 | 上位机问，单片机答 | 查询状态、下发命令 |
| 主动上报 | 单片机自己定时发 | 数据采集、心跳、报警 |

**主动上报的设计要点**：
- **周期固定**（比如每秒一帧），方便上位机做超时判断
- **不能阻塞**：上报要用非阻塞方式发（或用中断/DMA 发送），否则会拖住主循环

### 操作步骤

用 P05 的 1ms tick 做周期：

```c
uint32_t t_report = 0;

while (1)
{
    uint32_t now = tick_1ms;

    /* 每秒主动上报一帧状态 */
    if (now - t_report >= 1000)
    {
        t_report = now;

        uint8_t payload[5];
        uint32_t sec = now / 1000;
        payload[0] = (sec >> 24) & 0xFF;
        payload[1] = (sec >> 16) & 0xFF;
        payload[2] = (sec >> 8)  & 0xFF;
        payload[3] =  sec        & 0xFF;
        payload[4] = (HAL_GPIO_ReadPin(LED_GPIO_Port, LED_Pin) == GPIO_PIN_RESET) ? 1 : 0;

        uint8_t frame[16];
        uint16_t n = frame_build(frame, 0x10, payload, 5);   /* 0x10 = 状态上报 */
        uart_port_send(frame, n);
    }

    /* 同时照常处理接收 */
    uint8_t b;
    while (rb_get(&b)) frame_feed(b);
}
```

**上位机侧**：记录每帧到达的时间戳，验证间隔是否稳定。

### 验证方法

- 每秒收到一帧，间隔偏差 ≤10%（连续观察 30 帧）
- 帧里的运行秒数随时间递增、与实际情况吻合
- LED 状态变化后，下一帧上报的值跟着变
- **主动上报不影响命令应答**（发命令照样立刻回）

---

## 附：调试串口协议的通用套路

| 手段 | 用途 |
|---|---|
| **十六进制打印每个收到的字节**（`%02X`） | 一眼看出帧头在哪、长度对不对、校验差多少 |
| **校验值单独打印** | 对比"计算值 vs 收到值"，判断是算法不一致还是数据错 |
| **降波特率到 9600 再测** | 若变正常，说明是处理速度问题不是协议逻辑问题 |
| **给每类异常配计数器** | 跑一段时间看哪类错误在涨，比单步调试快得多 |
| **先自发自收（回环）** | TX 与 RX 短接，先验证收发链路再联调 |

---

## 下一步

- 回到任务卡 `tasks/P06-UART与自定义协议.md` 逐条勾选
- 交付物：源码（分层）、协议文档、100 帧压力测试记录、5 种异常验证表、分层图
- 照 `REVIEW_TEMPLATE.md` 准备 10 分钟演示，重点能讲：帧格式设计取舍、异常处理策略
