---
number: 72
category:
  - MCU
  - TinyML
  - Embedded AI
series: STM32
level: Advanced
type: Project
status: Completed
---

@[TOC](文章目录)

# 基于 STM32 与 TinyML 的端侧手势识别：项目启动与 MPU6050 数据采集

> **STM32 + TinyML 端侧手势识别系列 · 第 72 篇**
>
> 🎯 **本文重点**：项目架构、MPU6050 驱动、STM32 数据采集、KEY0 触发、Python 数据链路，以及第一阶段数据集。

---

## 一、为什么开始做这个项目？

前面的学习更多是围绕知识点展开：C 语言、数据结构、51 单片机、STM32，以及各种开发工具。

但真正开始做嵌入式 AI 项目之后，我希望把这些知识串成一条完整的工程链：

```text
传感器 → STM32 驱动 → 数据采集 → Python → CSV
                                  ↓
                              TinyML 训练
                                  ↓
                           STM32 端侧推理
                                  ↓
                            OLED / 串口输出
```

因此，这次选择 **MPU6050 + STM32F103 + TinyML**，尝试完成一个真正运行在 MCU 上的手势识别系统。

> 💡 **项目思路变化**：从“学会一个知识点”，逐渐转向“让多个知识点共同完成一个系统”。

---

## 二、项目总体架构

当前核心硬件平台为 **STM32F103ZET6**，通过 MPU6050 获取三轴加速度数据，后续模型最终在 STM32 本地完成推理。

```text
┌─────────────┐
│   MPU6050   │ 三轴加速度
└──────┬──────┘
       │ I²C
       ▼
┌─────────────┐
│ STM32F103   │ 驱动 / 采样 / 控制
└──────┬──────┘
       │ USART1
       ▼
┌─────────────┐
│   Python    │ 自动化采集
└──────┬──────┘
       │ CSV
       ▼
┌─────────────┐
│ TinyML模型  │ Edge Impulse
└──────┬──────┘
       │ C++部署包
       ▼
┌─────────────┐
│ STM32推理   │
└──────┬──────┘
       ▼
 OLED / UART
```

最终目标不是“电脑上能调用模型”，而是让模型真正进入资源有限的 MCU。

---

## 三、硬件平台

| 模块 | 当前方案 |
|:---:|:---|
| MCU | STM32F103ZET6 |
| CPU | ARM Cortex-M3 @ 72 MHz |
| 传感器 | MPU6050 |
| 通信 | I²C + USART1 |
| 显示 | 0.96 寸 I²C OLED |
| 调试 | 串口 / VOFA+ |
| ML | Edge Impulse + TinyML |
| 类别 | 4 类 |

当前定义 4 类手势：

| Label | 含义 | 第一阶段样本 |
|:---:|:---|---:|
| `idle` | 静止 / 待机 | 15 |
| `wave_lr` | 左右水平挥动 | 15 |
| `lift_ud` | 上下抬动 | 15 |
| `shake_fast` | 快速晃动 | 15 |
| **合计** | **4 类** | **60** |

> ⚠️ 60 个样本的目标是验证完整技术链，而不是证明模型已经具有很强的泛化能力。

---

## 四、先把 MPU6050 驱动真正跑起来

项目首先解决的是“数据从哪里来”。`mpu6050.h` 定义了器件地址和核心寄存器：

```c
#define MPU6050_ADDR         (0x68 << 1)
#define MPU6050_SMPLRT_DIV   0x19
#define MPU6050_CONFIG       0x1A
#define MPU6050_GYRO_CONFIG  0x1B
#define MPU6050_ACCEL_CONFIG 0x1C
#define MPU6050_ACCEL_XOUT_H 0x3B
#define MPU6050_PWR_MGMT_1   0x6B
#define MPU6050_WHO_AM_I     0x75
```

并将原始数据与工程中使用的浮点数据统一封装：

```c
typedef struct {
    int16_t Accel_X_RAW;
    int16_t Accel_Y_RAW;
    int16_t Accel_Z_RAW;
    float Ax, Ay, Az;
    float Gx, Gy, Gz;
    float Temperature;
} MPU6050_Data_t;
```

这样可以把**底层寄存器读取**与**上层 TinyML 数据处理**分开。

> 🔧 项目源码：`02_Source_Code/Elite_F103_Demo/Core/Inc/mpu6050.h`

---

## 五、STM32 先完成“数据能看到”

主程序启动时初始化 GPIO、USART1、I2C1 和 I2C2，并检查 MPU6050：

```c
MX_GPIO_Init();
MX_USART1_UART_Init();
MX_I2C1_Init();
MX_I2C2_Init();

if (MPU6050_Init(&hi2c1) == 0) {
    printf("[OK] MPU6050 detected on 0xD0\\r\\n");
    mpu_online = 1;
}
```

实际调试时遵循：

```text
硬件 / I²C
    ↓
驱动
    ↓
数据格式
    ↓
数据采集
    ↓
最后才看模型
```

> 💡 **经验**：模型识别错误不一定是模型的问题。嵌入式 AI 中，传感器、单位、串口和内存都可能是根因。

---

## 六、KEY0：把一次按键变成一次稳定采集事件

项目使用开发板 `KEY0` 触发手势采集。为了避免机械抖动，加入了边沿检测和 50 ms 防抖：

```c
if (current_pin != key_last_state &&
    (now - last_debounce_tick > 50)) {
    last_debounce_tick = now;
    key_last_state = current_pin;

    if (current_pin == GPIO_PIN_RESET) {
        return 1;
    }
}
```

这一步看似简单，却是后续状态机稳定运行的基础。

---

## 七、一次手势到底采多少数据？

当前工程定义：

```c
#define RECORD_SAMPLES 100
static float raw_features[RECORD_SAMPLES * 3];
```

也就是：

> **100 个采样点 × 3 个加速度轴 = 300 个原始输入值**

当前采样节拍约为 **50 Hz**，因此一次窗口约覆盖 2 秒。

这个窗口需要同时满足：

- 能覆盖一次完整动作；
- 不让 MCU 处理过长的数据；
- 与后续模型输入维度保持一致。

---

## 八、STM32 → Python → CSV

如果每次都手动复制串口输出，后续扩充数据集会非常痛苦。因此项目加入了 `collect_gestures.py`，使用 `pyserial` 自动接收数据并保存 CSV。

文件自动编号：

```text
idle.01.csv
idle.02.csv
...
idle.15.csv
```

脚本通过两个标志判断一次采集的边界：

```text
--- START_OF_GESTURE ---
        ↓
     缓冲数据
        ↓
--- END_OF_GESTURE ---
        ↓
      写入 CSV
```

> 🔧 项目源码：`01_Data_Acquisition/collect_gestures.py`

---

## 九、一次串口问题：代码和文档必须同步

开发过程中曾出现：文档写 `COM7`，Python 实际使用 `COM9`。

```python
PORT = "COM9"
BAUD_RATE = 115200
```

后来已经修复并统一配置。

> ⚠️ **工程提醒**：代码能运行 ≠ 项目文档正确。

当一个项目同时拥有 STM32 工程、Python 工具、数据集和说明文档时，配置一致性本身就是工程质量的一部分。

---

## 十、第一阶段成果

截至本篇，已经跑通：

```text
MPU6050
   ↓
STM32 读取
   ↓
KEY0 触发
   ↓
100 × 3 采样窗口
   ↓
USART1
   ↓
Python 自动保存
   ↓
4 类 × 15 个 CSV
```

项目实物图：

> 🖼️ **图片 01｜STM32F103ZET6 开发板实物图**
>
> CSDN 编辑器当前无法稳定转存外部图片地址，请将原图下载到本地，再通过 CSDN 工具栏「图片」重新上传。
>
> 原图地址：
> https://github.com/nathanialkonggsd-hub/STM32-Gesture-TinyML/blob/main/STM32-Gesture-TinyML/03_Images_and_Results/STMF103ZET6.jpg

---

## 十一、这一阶段真正学到了什么？

### 1. 驱动只是起点

能读出 MPU6050，不代表系统完成；真正有价值的是把数据继续送入数据集和模型。

### 2. 数据采集本身就是工程

串口、时间戳、文件命名、标签和采样频率都会直接影响后面的模型。

### 3. 从这里开始，项目进入“数据驱动”阶段

下一篇不再讨论传感器驱动，而是处理：

```text
STM32 串口
    ↓
Python
    ↓
CSV
    ↓
数据集质量
```

---

## 📚 系列导航

| # | 主题 | 状态 |
|---:|:---|:---:|
| **72** | **项目启动：系统架构与 MPU6050 数据采集** | 🟢 本文 |
| 73 | STM32 手势数据采集：从串口到 CSV 数据集 | ⏳ |
| 74 | TinyML 特征提取与模型训练：V1 → V2 | ⏳ |
| 75 | TinyML 模型部署：Edge Impulse → STM32F103 | ⏳ |
| 76 | STM32 端侧实时手势推理 | ⏳ |
| 77 | TinyML 工程优化：从 HardFault 到 32ms 推理 | ⏳ |
| 78 | 第一阶段总结：STM32 TinyML 手势识别系统完整跑通 | ⏳ |
| 79+ | 第二阶段：数据集扩充与泛化优化 | 🔮 |

## 🔗 项目地址

- **STM32 TinyML 项目仓库**：https://github.com/nathanialkonggsd-hub/STM32-Gesture-TinyML
- **CSDN 文章仓库**：https://github.com/nathanialkonggsd-hub/CSDN

## 👋 结语

感谢阅读！如果这篇文章对你有帮助，欢迎收藏、点赞，也欢迎在评论区交流。

项目仍在持续开发，下一篇继续进入数据采集与数据集工程。
