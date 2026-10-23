# 修订记录

## v1.2 · 2026-10-23（深度审读修复）

修订依据：对全仓库 46 个文件做第二轮**逐行**审读（4 路并行分别覆盖 P00-P02 / P03-P05 / P06-P07 / P08-P10，加上方案层人工核对与脚本校验），并抽验复现了全部阻断级问题。基线版本 `v1.1-baseline`（git tag）。

原则：**不改项目编号、周期、日历骨架、任务点数量（117 个不变）、锚点 id、A/B 档归属**；只修错误、漏洞、自相矛盾与口径不一致。

### 一、阻断级修复（3 条，不修会做不出来/验收必失败）

| # | 位置 | 修复内容 |
|---|---|---|
| 1 | `tutorials/P05` B2 | 电机三档 CCR 计算写成 `×(99+1)/100`，而 TIM3 的 ARR=999 → 三档实际变成 3%/6%/10%。改为按 `__HAL_TIM_GET_AUTORELOAD(&htim3)+1` 计算，并在验证里写明"三档应约 30%/60%/100%（CCR ≈ 300/600/1000）" |
| 2 | `tutorials/P04` B1 | 双击报出 `DOUBLE` 后回到 PRESSED，第二击松开又被当短按 → **每次双击多报一条 SHORT**。新增 `SWALLOW` 状态吞掉第二击；`SHORT` 的时长改用真实按压时长（原为 0）；验证与常见问题补对应条目 |
| 3 | `tutorials/P09`「把中断搬进 RTOS」 | 中断里用的 `osMessageQueuePutFromISR()` **在 CMSIS-RTOS2 中不存在**，照抄编译不过。改为 `xQueueSendFromISR + portYIELD_FROM_ISR`，并说明"`osMessageQueuePut` 只能在任务里用" |

### 二、严重级修复（会做不出来 / 结论不可信 / 验收口径对不上）

| 位置 | 修复内容 |
|---|---|
| `tutorials/P08` A1 | **补 ADC 时钟分频**（`PCLK2/6` → 12 MHz）：原文全篇缺这一步，F103 默认 PCLK2/2 = 36 MHz 超 14 MHz 上限，读数不可信；删除 F1 没有的 `Resolution` 选项 |
| `tutorials/P09` B3 | 栈口径统一到 **`osThreadGetStackSpace()`（字节）、最紧张任务剩余 ≥256 B**，删除"使用率 40%~80%""剩余 >85% 浪费"；`tasks/P09` 的验收标准同步改写，三处口径一致 |
| `tutorials/P07` 附录 A | `#define OLED_DRIVER SSD1306` + `#if (OLED_DRIVER == SH1106)` 两个标识符均未定义 → 预处理恒真、整屏右移 2 列。改为数值宏（`OLED_DRIVER_SSD1306/V1/2`） |
| `tutorials/P06` A1 | 单字节阻塞回显在突发发送下会 RX 溢出丢字 → 改为整段缓冲回发（或降波特率），并补 ORE 排查与 `HAL_UART_ErrorCallback` 重挂 |
| `tutorials/P07` B2 | 示波器 10 µs/格 看不到 start+stop → 改 30~50 µs/格、只发单字节命令、SDA 下降沿触发 |
| `tutorials/P03` 0.3 | 新建工程补 **C99 Mode + Use MicroLIB** 两步（附骨架在语句后声明变量，漏勾 C99 必编译报错） |
| `tutorials/P02` 示例 | 变量/对象声明从 CubeMX 生成区移入 `USER CODE` 区；A5 补"printf 前置检查"（USART1/MicroLIB/交叉接线），说明 P01 B 档后延会让 P02 全部串口任务没有输出 |
| `tutorials/P01` A6 | Git 落地流程统一为"提交到 P00 仓库的 `projects/P01-…/`"，删除 A6 里的 `git init` + `git remote add`（复用仓库时会报错） |
| `tutorials/P10` 周计划 / 选题池 | 第 1 周改为"到货清点 + 搭骨架"（下单在 P09 第 4 周）；文档与异常测试前移到交付日 06-11 之前；选题池首项改为"温湿度**记录仪**（+Flash 存历史/查询）"避免与 P07 B5 重复 |
| `tutorials/P04` A3/A4 | 删除任务卡之外的 `MID` 事件（500 ms~1 s 归短按）；补 30 ms 最小间隔过滤 + 结束时清标志，消除 `dur≈0` 幽灵事件 |
| `tutorials/P05` 资源表 | 补 **PB6/PB7 = I2C1** 的跨项目冲突提醒；输入捕获口径统一为 **PWM Input（PB6）**；A4/B1 的无示波器自测同步改口径 |
| `tutorials/P05` A3 | 删掉"LED 直接跨在 PA6 与 GND 之间"（与同段"串 1kΩ"矛盾，照做会烧 LED/IO） |
| `tutorials/P05` B2/B4 | 补 CubeMX 通道配置步骤（CH2 不配就没有波形）；B4 的恒真校验改为 `HAL_TIM_PWM_GetState()` 真实性检查；"加一行不改代码"补"仍需先在 CubeMX 配该通道" |
| `tutorials/P05` B5 | 去掉与 A1 的重复、去掉 `tick_1ms` 重复定义；改写为"软件定时器调度 + 与 A2 阻塞写法对比实验" |
| `tutorials/P03` A5 / A4 | README 模板不再把结论填死；A4 与 A3 去抖口径对齐；心跳改成 `now - last_hb >= 500` |
| `tutorials/P02` B4 / A4 | 测电阻前必须断电；同名 `led_is_on()` 拆分改名 |
| `tutorials/P09` | 修正"`osThreadId_t` 与 `TaskHandle_t` 不能混用"的错误结论；补 heap 预算与 `configCHECK_FOR_STACK_OVERFLOW` 的准确路径；区分 HAL assert 与 `configASSERT` |
| `GITHUB_新手上路` | "可以只用 Git 不用 GitHub"改为"Git 可离线用，但本方案交作业必须用 GitHub" |

### 三、方案层口径统一（跨文件矛盾）

| 位置 | 修复内容 |
|---|---|
| 11 张任务卡（22 处标题） | 「保底 / 进阶」→「**通关底线 / 能力上限**」，与 `MASTER_PLAN` §1 的用词理由一致 |
| `GANTT.csv` | P10 备注"交付日提前到 06-13"→ **06-11**（与本行交付日列一致）；P09 交付日 05-20 → **05-21**（slot 末 −2 天）；P01 交付日 09-25（中秋）→ **09-24** |
| `MASTER_PLAN` §2.2 | 补"交付日遇法定假日再提前 1 天"规则（对应 P01 的改动） |
| `MASTER_PLAN` §10 | 预算从旧口径 2,700~6,500 改为 **3,830~7,960（§1）+ §2 另计**；**新增 §10.1 全项目引脚总表**（含 PB6/PB7 复用冲突） |
| `BOM` §6 | 班级规模伸缩表按 §2 单价与 `ceil(N/4)` **重算**（30 人 §2 由 6,770~17,200 改为 7,110~18,600；60 人同步修正），并补情景 A 的 §2 参考值 |
| `tasks/P05` | 难度 ★★★ → **★★★★**（与 GANTT/CHANGELOG 一致） |
| `tasks/P01` A6 | 删除"或 Gitee"，改为提交到 P00 仓库 |
| `tasks/P04` A3 | 补"500 ms~1 s 归短按"的边界说明 |
| `tasks/P10` | 前置项目数由"至少 7 个"改为"P01~P09 全部通过" |
| `README` | 示例提交信息 "B档4条" → **B档5条** |
| `CHANGELOG` v1.1 条目 | 文件计数 32 → **31**（9 + 11 + 11） |

### 四、一般级清理

按审读清单逐条修（P00~P10 共 50 余条），包括：CubeMX 菜单名（`Reset_State`、`I2C1` 重映射、Timebase Source）、`.gitignore` 的 `MDK-ARM/*/` 与 `RTE`、MDK-Lite 32 KB 限制、ODR/BSRR 对比、`printf` 格式符与 MicroLIB 浮点提醒、P03 的 LED 极性判断方法、P06 帧格式 LEN 上限与 5 种异常输入、P07 字模子集与换屏说明、P08 万用表档位与 VDDA 参考、P09 记录表行数、P10 自查清单第 15 项等。逐条明细见 `tasks/` 与 `tutorials/` 的对应位置。

### 五、本次仍未改动 / 需上机实测确认

1. **未编译、未上板**：所有 C 代码改动仍是静态审读结果。建议开班前用 CubeMX + Keil 实际编译并上板验证：`P02 A4`、`P04 A1/A3/B1`、`P05 A3/B2/B3/B4`、`P07 附录 A`、`P08 A1`、`P09 A2/B3`。
2. **CubeMX 版本相关**（以本机 6.18.1 为准）：`Reset_State` 菜单名、`Toolchain/IDE` 无版本子项、I2C1 重映射的实际操作路径、FreeRTOS 的 Timebase Source 默认选哪个定时器。
3. **Keil 工程树**：`.gitignore` 的 `MDK-ARM/*/` 是否会连 `RTE_Components.h` 一起忽略；构建产物实际落在 `Objects/` 还是 `MDK-ARM/<工程名>/`。
4. **MicroLIB 浮点**：`printf("%.2f")` 在不同 MDK 版本下的表现（影响 P02 B2）。
5. **P07 字模子集**：B1 菜单用词是否全部落在随附的 42 字形内（已改为纯大写短词，建议上板确认一遍）。
6. **AHT20 换算系数**：仍按数据手册 `T = −50 + 200×raw/2^20`、`RH = 100×raw/2^20`。
7. **校历**：`MASTER_PLAN` §2.1 的节假日仍按中国大陆法定节假日估算，**开学前必须按本校校历核对**（尤其军训时间）。

## v1.1 · 2026-10-18（审读修订）

修订依据：对全仓库 31 个文件做静态审读（方案层 9 份 + 11 张任务卡 + 11 篇教程），对日历、slot 算术、117 个任务点、锚点做逐项复算，并复核了 STM32 技术细节与数值计算。
原则：**不改动项目编号、周期、日历骨架、任务点数量（117 个不变）、锚点 id**，只修错误、漏洞与自相矛盾。

### 一、阻断级修复（不修会做不出来/学错知识）

| # | 位置 | 修复内容 |
|---|---|---|
| 1 | `tasks/P03` A1、`tutorials/P03`、`hints/P03`、`FAQ` Q10、`tutorials/README.md` | 按键上下拉方向**四处统一为"PA0 + 内部下拉 + 按下为高"**（原文标题/任务卡/FAQ 写"上拉"、教程正文写 `Pull-down`，且 README 示例引脚误写 `GPIOC`） |
| 2 | `tasks/P04` A1、`tutorials/P04` | 外部中断边沿由"下降沿"统一改为**上升沿**（原写法只在松手时触发，长按/双击永远判不出） |
| 3 | `tutorials/P04` A2、B3 | 共享变量从 `USER CODE BEGIN 4`（在 `main()` 之后）移到 **`USER CODE BEGIN 0`**，并说明"变量放 0 区、函数放 4 区" |
| 4 | `tutorials/P08` B4 | Flash 地址 `0x0801FC00` → **`0x0800FC00`**（F103C8 只有 64KB），并补地址推导与 `FLASH_SIZE` 运行时断言 |
| 5 | `tutorials/P07`（新增附录 A）、`tasks/P07` A2 | 补一份**可直接编译的最小 SSD1306 I2C 驱动**（初始化序列 / 显存写入 / 6×8 字模 / PCtoLCD2002 设置 / 统一 `oled_*` 接口），不再让零基础学员"自己上网找" |
| 6 | `tutorials/P02` B2 | `printf` 格式符 3 个、实参 2 个 → 修正为 2/2，并补"格式符个数必须等于实参个数"的说明 |
| 7 | `tutorials/P05` A3、`BOM` | 明确 PC13 无定时器通道、须外接 LED 到 PA6 + 1kΩ；BOM 第 1 批补"5mm LED + 1kΩ × 15 套" |
| 8 | `tutorials/P09` A2 | 补 `#include "FreeRTOS.h"` / `"task.h"`；改用显式 `osThreadAttr_t` 并说明 **CMSIS-RTOS2 的 `stack_size` 单位是字节**（CubeMX 界面是 words） |
| 9 | `BOM` 第 1 批 | 补 **ST-Link V2 × 15**（蓝板不含烧录器）、**USB-TTL × 15 提前到第 1 批**（P01 B 档就要串口）、外接 LED/电阻、5V 独立电源、MOS+续流二极管 |
| 10 | `REVIEW_TEMPLATE`、`MASTER_PLAN` §6 | 学期评审表补第 6 维凑满 **30 分**（原文 5 维=25 分却用 30 分门槛）；项目/学期门槛统一为 18/30、24/30；"通过"判据补 B 档规则 |
| 11 | `README`、`MASTER_PLAN` §1/§4、`REVIEW_TEMPLATE` | A/B 档表述由"保底/进阶 + 全员必做"改为**"通关底线 / 能力上限，两档都是全员要求"**，并明确"B 档当次可欠账、学期末必须补齐" |
| 12 | `BOM` | 传感器指定 **I2C 型号（AHT20 0x38 / SHT30 0x44）**，明确禁止 DHT11/光敏电阻作 P07 主器件；舵机/电机 8 → **15 套**；示波器台数与预算计入合计 |

### 二、技术准确性修正（"结论对、过程错"）

| 位置 | 修复内容 |
|---|---|
| `tutorials/P01` A2 | "没焊晶振 → 改选 Bypass" 错误 → 正确做法是**只选 HSI**（Bypass 是外部有源时钟）。 |
| `tutorials/P05` A5 | `HAL_RCC_GetPCLK1Freq()` 是 **36MHz**，定时器时钟是 ×2；改为打印 `SystemCoreClock` 与 `GetPCLK1Freq()*2` |
| `tutorials/P05` B2/B4 | 同一个 TIM3 两个通道跑 1kHz + 10kHz **物理上不可能**（ARR 是每定时器一个）；默认改为同频 1kHz，要 10kHz 明确要求换定时器 |
| `tutorials/P05` B3 | `Input Capture direct mode` 却把 CCR 当周期 → 改为 **PWM Input 模式**（CH1 周期 / CH2 脉宽），并给出 direct capture 的"两次捕获求差"备选写法 |
| `tutorials/P05`（新增资源分配表） | 新增 TIM2/TIM3/TIM4 通道分配表，每个 B 档任务开头标注"本条占用哪路、和谁冲突" |
| `tutorials/P08` A2 | "`raw*3300` 在 16 位会溢出" 是错的 → Cortex-M3 上 `int` 32 位会整型提升，不会溢出；强转理由改为"显式表达意图" |
| `tutorials/P08` B3 | 标定系数 `b_x1000 = −2` → **`−1500`**（b 也放大了 1000 倍），并补整数推导与真值校验 |
| `tutorials/P02` A4 | 位宏 `SET_BIT/CLEAR_BIT/TOGGLE_BIT/READ_BIT` **与 CMSIS 撞名** → 改为 `BIT_SET/BIT_CLEAR/BIT_TOGGLE/BIT_READ`，并说明撞名后果 |
| `tutorials/P09` A3 | "`HAL_Delay` 让其他任务全停" → 改为准确机制（忙等、不关调度器、同级时间片抖动、低优先级被饿），并给可复现实验条件 |
| `tutorials/P09`（新增小节） | 新增"把 P04/P06 的中断搬进 FreeRTOS"：`configMAX_SYSCALL_INTERRUPT_PRIORITY=5`、ISR 优先级必须 ≥5、只能用 `...FromISR` |
| `tutorials/P04` A1 | EXTI 抢占优先级 1 → **6**（为 P09 的 RTOS 留出，<5 的中断不能调用 RTOS API） |
| `tutorials/P06` A3/B1 | 帧格式表"校验 2 字节 CRC16"与 A 档实现（1 字节累加和）矛盾 → A 档表格改 1 字节，B1 给 v2（CRC16-MODBUS）与版本约定 |
| `tutorials/P06` A4/B2 | 补前置声明、缺失变量声明、**首次 `HAL_UART_Receive_IT` 挂载**（不挂就一个字节都收不到）、上位机循环发送脚本 |

### 三、结构与流程修正

| 位置 | 修复内容 |
|---|---|
| `MASTER_PLAN` §2.1/§2.2 | 新增**不可用时间扣除表**（法定节假日 12 个工作日、军训、期末复习周）与**缓冲策略**（交付日提前 2 天、B 档欠账后延、学期末 2 周补交窗口） |
| `MASTER_PLAN` §6.1 | 新增**评审人力模型**（15/30/60 人三档工时）、学长初筛+指导人复核分工、**首年无学长的替代方案** |
| `MASTER_PLAN` §3 | 能力模型加"由哪些任务支撑"列，让承诺可追溯（原"会用位运算操作寄存器""照手册驱动新外设"几乎没有支撑任务） |
| `MASTER_PLAN` §10 | 统一开发板型号为 F103C8T6 蓝板（原文允许 F4，与教程引脚冲突）；示波器口径统一"每 4 人 1 台"；删掉"已有开发板"与 BOM 重复采购的矛盾 |
| `MASTER_PLAN` §8、`ONBOARDING` | 跳级/插班补交要求由"交自测清单"改为**补交 A 档四件套**（清单不能替代可运行工程） |
| `README` | 结构树补 `tutorials/README.md` 与 `LICENSE`；新增"同一知识点只在一处写全"的维护约定；P00 窗口改为整两周 08-31~09-13；补"把带教人加为协作者"（私有仓库否则打不开） |
| `GANTT.csv` | P00 日期改为 08-31~09-13；P05 难度 ★★★ → ★★★★（实际负荷高于 P07）；新增"交付日"实际日期与"备注"列（节假日/缓冲/欠账窗口）；补"学期末补交与归档"行 |
| `REVIEW_TEMPLATE` | 补"评审人""评审方式""总分""判定勾选""B 档欠账登记"字段；第 6 维改为按 B 档完成质量给分；PITFALLS 贡献的加分路径落到"文档质量"维度 |
| `FAQ` | 编号重排为 Q1~Q16（原文 Q6 出现两次、编号只到 Q15）；Q10 的上下拉结论同步；补"Keil 必须勾 C99""USER CODE 4 变量看不见""无示波器如何验收" |
| `BOM` | 重写为"分批 + 共用 + 情景合计 + 班级规模伸缩公式 + P10 备料时间表"；补全口径预算（原 2,700~6,500 未含 ST-Link、串口模块与共用设备）；明确"谁付费"必须开班前定 |
| `PITFALLS` | 新增 13 条真实坑（上下拉配反、边沿选反、USER CODE 4、C90 声明、每定时器一个 ARR、PCLK1=36MHz、printf 格式符、首次挂接收、OLED 白屏、Flash 地址、CMSIS 栈单位、HAL_Delay、ISR FromISR） |
| `tutorials/README.md` | 锚点示例改为 PA0/下拉（原示例误写 GPIOC/上拉）；硬件前提补"上板先实测按键与 LED 极性"三步法；补锚点维护规则 |
| `LICENSE`（新增） | 文档 CC BY 4.0 / 代码 MIT 双授权（原仓库无 LICENSE，他人无法合法复用） |
| `tasks/P00` | 新增"两周怎么排"（必修 9 条 + 选做 6 条两轮），降低零基础两周自学 15 点的放弃率；补"加带教人为协作者" |
| `tasks/P10`、`tutorials/P10` | 重画接线图（原图把 PB7 接到传感器 GND，且 I2C 用了需 Remap 的 PB8/PB9）；选题池传感器改为 I2C 型号，单总线器件标注"计划外自学 10~20 小时"；A2 器件清单提前到 P09 中期；A5 要求说明与模板的差异点；A3 补"接线三条铁律" |
| `tutorials/P07`、`tasks/P07` | 原 10 条任务**没有一条读真实传感器**（名不副实）→ B5 改为"AHT20 温湿度上屏 + 超限告警"，A 档/任务点数量不变（仍 A5+B5） |

### 四、未改动（保留原样）

`tasks/P08-ADC与数据采集.md`、`tutorials/P00-Git与GitHub入门.md`、`hints/P00`、`hints/P01`、`hints/P02`、`hints/P06`、`hints/P07`、`hints/P08`、`hints/P10`、`GITHUB_新手上路.md`、`.gitignore`、`.gitattributes`。

其中 `tasks/P08` 未改：Flash 地址问题只出现在教程 B4，任务卡未给具体地址；如需任务卡也写地址推导，可后续补。

### 五、仍需人工确认/实测的项

1. **未编译、未上板**：所有代码修改均为静态审读结果，建议开课前用 Keil + CubeMX 实际编译一遍 P02 A4、P05 B3/B4、P07 附录 A、P09 A2 这几段代码。
2. **AHT20 换算系数**：已按数据手册改为 `T = −50 + 200×raw/2^20`、`RH = 100×raw/2^20`；若手上模块手册不同，改代码里的宏即可。
3. **P06 帧格式**：A 档仍为"1 字节累加和无版本字段"，v2 才引入 CRC16 与版本号；若要 A 档就带版本号，需同步改 A3 帧表、A4 状态机、A5 异常帧。
4. **校历**：`MASTER_PLAN` §2.1 的节假日按中国大陆法定节假日估算，**开学前必须按本校实际校历（尤其军训时间）核对并改回表中**。
5. **板子极性**：教程按"常见蓝板 PA0 按下为高"编写；不同批次差异较大，已在 `tutorials/README.md` 要求"上板第一步先实测极性"。
