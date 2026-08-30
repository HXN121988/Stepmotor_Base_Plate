# STM32F103C8T6 + TMC2209 步进电机驱动底板
> 基于STM32F103C8T6主控 + TMC2209步进驱动的硬件底板工程，包含原理图、PCB、BOM清单、Keil MDK测试代码。支持梯形加减速，适合小型步进电机运动控制开发。

[![Hardware Version](https://img.shields.io/badge/HW-V1.0-blue)](.)
[![MCU](https://img.shields.io/badge/MCU-STM32F103C8T6-orange)](.)
[![Driver](https://img.shields.io/badge/Driver-TMC2209-green)](.)
[![License](https://img.shields.io/badge/license-MIT-yellow)](.)

## 📋 项目简介
本项目是一块步进电机控制底板，主控采用STM32F103C8T6，外接TMC2209步进电机驱动模块。
- 输入电源：**DC 28V**，板载TPS54302实现28V转5V电源；
- 板载ESD/TVS保护电路，电源防浪涌；
- 引出OLED I2C接口、按键输入接口；
- Keil C代码库：支持定步数运行、转速控制、**梯形加减速算法**，解决高速丢步问题。

适合：小型运动机构、DIY、学习TMC2209步进电机控制。

## 🧩 硬件说明
### 硬件框图
- 主控：STM32F103C8T6（排母外接）
- 电机驱动：TMC2209模块（H3、H4接插件）
- 电源输入：DC‑005‑A200 DC座，输入28V
- 电源芯片：TPS54302DDCR（28V →5V）
- TVS保护：
  - D2: SMBJ28CA（电源输入端浪涌保护）
  - D5: SMBJ6.8A（5V输出端保护）
- 外设接口
  - OLED I2C接口：GND / 3V3 / SCL / SDA
  - 4路按键输入
  - 电机端子：ZX‑HY2.0‑4P（M1A M1B M2A M2B）

### 文件目录（硬件）
```
hardware/
├── SCH_Schematic.pdf      # 原理图PDF
├── PCB_PCB.pdf            # PCB图+BOM清单
└── BOM.csv                # 物料清单
```

### 主要器件BOM摘要
|器件|型号|封装|说明|
|---|---|---|---|
|U4|TPS54302DDCR|TSOT‑23‑6|28V转5V降压芯片|
|D2|SMBJ28CA|SMB|输入28V TVS保护管|
|D5|SMBJ6.8A|SMB|5V输出TVS保护管|
|L2|10uH|L0603|DC‑DC功率电感|
|DC1|DC‑005‑A200|直插|DC电源插座|

> 完整BOM请看`PCB_PCB.pdf`文档。

## ⚙️ 软件说明
开发环境：**Keil MDK‑ARM5**，标准库STM32F10x_StdPeriph_Driver。

### 代码目录结构
```
software/
├── main.c              # 主函数，测试示例
├── step_motor.c/h      # 步进电机驱动库
├── systick.c/h         # SysTick微秒/毫秒延时
└── README.md           # 软件说明
```

### 📌 硬件引脚映射（step_motor.h）
```c
#define STEP_PORT       GPIOA
#define STEP_PIN        GPIO_Pin_0     // TMC2209 STEP脉冲
#define DIR_PORT        GPIOA
#define DIR_PIN         GPIO_Pin_1     // TMC2209 DIR方向
#define EN_PORT         GPIOA
#define EN_PIN          GPIO_Pin_2     // TMC2209 EN使能
```

### 🚀 API接口说明
```c
//初始化步进电机GPIO
void StepMotor_Init(void);

//使能驱动器输出
void StepMotor_Enable(void);

//关闭驱动器输出
void StepMotor_Disable(void);

//设置电机方向 DIR_CW正转 / DIR_CCW反转
void StepMotor_SetDir(uint8_t dir);

//定步数恒速运行
//steps:脉冲数，dir方向，pulse_interval_us脉冲间隔(us，越小速度越快)
void StepMotor_Steps(uint32_t steps, uint8_t dir, uint32_t pulse_interval_us);

//按转速运行指定圈数
//rpm转速(转每分钟), dir方向, rounds圈数
void StepMotor_RunRPM(uint16_t rpm, uint8_t dir, uint32_t rounds);

//梯形加减速运动，长距离防丢步
void StepMotor_MoveWithAccel(uint32_t total_steps, uint8_t dir,
                             uint16_t start_rpm, uint16_t max_rpm, uint16_t acc_steps);
```

### 使用示例（main.c）
```c
//示例1：正转1圈60RPM，延时0.5s，反转1圈
//StepMotor_RunRPM(60, DIR_CW, 1);
//Delay_ms(500);
//StepMotor_RunRPM(60, DIR_CCW, 1);
//Delay_ms(500);

//示例2：梯形加减速运动（推荐长距离运动使用）
//总步数10000步，启动转速20rpm，最高转速120rpm，加减速段各2000步
StepMotor_MoveWithAccel(10000, DIR_CW, 20, 120, 2000);
Delay_ms(1000);
StepMotor_MoveWithAccel(10000, DIR_CCW, 20, 120, 2000);
Delay_ms(1000);
```

> 默认参数：`PULSE_PER_ROUND 3200`，对应200步电机，16细分。修改这个宏适配不同细分设置。

## ⚠️ 注意事项
1. 输入电压严格为**28V DC**，不要超压，否则会烧毁TVS与电源芯片。
2. TMC2209模块电流电位器需要预先调好，不要超过电机额定电流，防止过热。
3. `SysTick`时钟源配置为HCLK/8，请确认系统时钟配置正确（72MHz）。
4. EN引脚高电平为驱动器失能，低电平为使能，符合TMC2209硬件逻辑。
5. PCB工程使用**嘉立创EDA**绘制，原理图版本V1.0。

## 📂 仓库文件清单
```
├── hardware/
│   ├── SCH_Schematic.pdf
│   └── PCB_PCB.pdf
├── software/
│   ├── main.c
│   ├── step_motor.c
│   ├── step_motor.h
│   ├── systick.c
│   └── systick.h
└── README.md
```

## 📄 License
MIT License
> 你可以自由使用、修改、分发，用于个人学习、DIY项目。商业使用请自行评估硬件风险。

## 🤝 贡献
欢迎Issue反馈硬件bug、代码bug，也欢迎提交PR优化代码。

---

### 复制提示
直接全选复制上面全部文本，粘贴到Github仓库的`README.md`即可，图片链接你后续可以自行补充。
如果你需要，我还可以帮你生成github仓库的`.gitignore`（Keil工程专用）。
