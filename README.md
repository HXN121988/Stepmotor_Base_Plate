# STM32F103 + TMC2209 步进电机驱动底板开源项目
基于 **STM32F103C8T6** 主控与 **TMC2209** 步进电机驱动模块的一体式硬件底板工程，附带完整 Keil 嵌入式 C 语言驱动代码。支持恒速运行、定转速圈数运行以及梯形加减速长距离运动，有效避免高速启动丢步，适合小型步进电机运动控制开发与 DIY 项目。
---
## 📋 目录
- [主要特性](#-主要特性)
- [硬件说明](#-硬件说明)
- [软件架构](#-软件架构)
- [关键参数配置](#-关键参数配置)
- [API 函数说明](#-api-函数说明)
- [快速上手](#-快速上手)
- [注意事项](#-注意事项)
- [仓库文件清单](#-仓库文件清单)
- [许可证与贡献](#-许可证与贡献)
---
## 🚀 主要特性
- **完整软硬件方案**：包含原理图、PCB、BOM 物料清单与完整 Keil 工程代码，开箱即用。
- **三种运动控制模式**：定步数恒速、按 RPM 转速定圈、线性梯形加减速，适配不同运动场景。
- **硬件保护设计**：板载输入输出 TVS 浪涌保护、28V 转 5V DC-DC 电源，工作稳定可靠。
- **高精度脉冲输出**：基于 SysTick 的微秒级阻塞延时，脉冲宽度与间隔精确可控。
- **引脚可配置化**：STEP、DIR、EN 控制引脚通过宏定义封装，可快速适配不同硬件布局。
- **加减速防丢步**：内置线性加速→匀速→减速算法，适配大惯量负载，降低高速失步风险。
---
## 🔌 硬件说明
本项目为一体式步进电机控制底板，主控采用 STM32F103C8T6 最小系统排母接入，驱动采用 TMC2209 模块，板载电源与保护电路。

### 核心硬件参数
- 输入电源：DC 28V（DC-005 电源座）
- 板载电源：28V 转 5V DC-DC（TPS54302），最大输出 3A
- 保护电路：输入端 SMBJ28CA TVS、输出端 SMBJ6.8A TVS
- 扩展接口：I2C OLED 接口、4 路独立按键、4P 电机接线端子
- 绘制工具：嘉立创 EDA，硬件版本 V1.0

### 默认引脚映射
控制引脚可在 `step_motor.h` 中自定义修改：
| 功能 | STM32F103 引脚 | TMC2209 对应 | 说明 |
| :--- | :--- | :--- | :--- |
| **STEP** | PA0 | STEP | 步进脉冲输入（上升沿触发） |
| **DIR** | PA1 | DIR | 旋转方向控制 |
| **EN** | PA2 | EN | 驱动器使能（低电平有效） |
| GND | GND | GND | 主控与驱动共地 |

> ⚠️ 注意：上电后 EN 引脚默认高电平，驱动器处于失能状态；调用 `StepMotor_Enable()` 后驱动器使能，电机进入锁轴状态。
---
## 🗂️ 软件架构
代码基于 STM32 标准外设库开发，采用分层设计，逻辑清晰易移植：
```
├── main.c              # 主程序，包含多种运动测试示例
├── step_motor.c / .h   # 步进电机核心驱动库
└── systick.c / .h      # SysTick 延时驱动，提供 us/ms 级延时
```
- 驱动层：step_motor 模块封装所有电机控制逻辑，与业务代码解耦
- 工具层：systick 模块提供精确延时，保证脉冲时序精度
- 应用层：main 函数中提供多组测试用例，可直接调用验证
---
## ⚙️ 关键参数配置
所有配置集中在 `step_motor.h` 头文件中，按需修改即可：
```c
/************************ 硬件配置宏 ************************/
#define STEP_PORT       GPIOA
#define STEP_PIN        GPIO_Pin_0
#define DIR_PORT        GPIOA
#define DIR_PIN         GPIO_Pin_1
#define EN_PORT         GPIOA
#define EN_PIN          GPIO_Pin_2

/************************ 参数配置宏 ************************/
#define DIR_CW          1       // 正转
#define DIR_CCW         0       // 反转
#define PULSE_PER_ROUND 3200    // 每圈脉冲数（默认200步电机+16细分）
#define PULSE_HIGH_US   5       // 脉冲高电平最小宽度（单位：微秒）
```
> 提示：修改 `PULSE_PER_ROUND` 即可适配不同细分设置，如 8 细分对应 1600 脉冲/圈。
---
## 📖 API 函数说明
### 基础控制函数
```c
// 步进电机GPIO初始化，上电默认驱动器失能
void StepMotor_Init(void);

// 使能驱动器输出，电机进入锁轴状态
void StepMotor_Enable(void);

// 关闭驱动器输出，电机自由状态
void StepMotor_Disable(void);

// 设置旋转方向：DIR_CW 正转 / DIR_CCW 反转
void StepMotor_SetDir(uint8_t dir);
```

### 运动控制函数
```c
// 定步数恒速运行
// steps：总脉冲数；dir：旋转方向；pulse_interval_us：脉冲间隔（微秒，值越小速度越快）
void StepMotor_Steps(uint32_t steps, uint8_t dir, uint32_t pulse_interval_us);

// 按指定转速运行指定圈数
// rpm：转速（转/分钟）；dir：旋转方向；rounds：运行圈数
void StepMotor_RunRPM(uint16_t rpm, uint8_t dir, uint32_t rounds);

// 梯形加减速运行（长距离推荐，防丢步）
// total_steps：总步数；start_rpm：启动转速；max_rpm：最高转速；acc_steps：加减速段步数
void StepMotor_MoveWithAccel(uint32_t total_steps, uint8_t dir,
                             uint16_t start_rpm, uint16_t max_rpm, uint16_t acc_steps);
```
---
## 🚀 快速上手
1. 硬件连接：接入 28V 直流电源，将步进电机连接到端子座
2. 打开 Keil 工程，引入对应文件，配置系统时钟为 72MHz
3. 在 main 函数中调用初始化与控制函数：
```c
int main(void)
{
    // 外设初始化
    SysTick_Init();
    StepMotor_Init();
    // 使能驱动器
    StepMotor_Enable();
    
    while (1)
    {
        // 示例：梯形加减速往返运动
        StepMotor_MoveWithAccel(10000, DIR_CW, 20, 120, 2000);
        Delay_ms(1000);
        StepMotor_MoveWithAccel(10000, DIR_CCW, 20, 120, 2000);
        Delay_ms(1000);
    }
}
```
4. 编译下载程序到 STM32，电机即可按设定参数运行。
---
## ⚠️ 注意事项
1. 电源输入为 DC 28V，请勿超压使用，避免损坏电源芯片与保护器件。
2. TMC2209 模块的驱动电流需提前通过电位器调节，不可超过电机额定电流。
3. SysTick 时钟源配置为 HCLK/8，请确保系统主频配置为 72MHz，保证延时精度。
4. 大负载高速运动时，建议使用梯形加减速模式，降低启动电流与失步风险。
5. 驱动器失能状态下电机无锁止力，设备垂直安装时请注意防止负载坠落。
---
## 📁 仓库文件清单
```
├── hardware/
│   ├── SCH_Schematic.pdf      # 电路原理图
│   └── PCB_PCB.pdf            # PCB版图与BOM物料清单
└── software/
    ├── main.c                 # 主程序与测试示例
    ├── step_motor.c           # 步进电机驱动实现
    ├── step_motor.h           # 驱动头文件与配置
    ├── systick.c              # SysTick延时实现
    └── systick.h              # 延时函数声明
```
---
## 📄 许可证与贡献
本项目采用 **MIT License** 开源协议，可自由用于个人学习、DIY 改造与商业项目。

欢迎提交 Issue 反馈硬件或代码问题，也可以通过 Pull Request 优化代码与文档。
