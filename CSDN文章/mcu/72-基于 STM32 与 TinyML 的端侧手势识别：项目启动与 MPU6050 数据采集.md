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

# 基于 STM32 与 TinyML 的端侧手势识别：项目启动与 MPU6050 数据采集

> **STM32 + TinyML 端侧手势识别系列 · 第 72 篇**
>
> 本篇不是一篇“把所有成果一次讲完”的总结，而是记录这个项目从想法落地到第一阶段数据链路跑通的过程。后续的模型训练、模型部署、端侧推理和工程优化，将继续拆成独立文章。

---

## 一、为什么开始做这个项目？

前面的学习更多是围绕知识点展开：C 语言、数据结构、51 单片机、STM32，以及各种开发工具。

但真正开始做嵌入式 AI 项目之后，我希望把这些知识串成一条完整的工程链：

```text
传感器
  ↓
STM32 驱动
  ↓
实时数据采集
  ↓
Python 数据处理
  ↓
数据集
  ↓
TinyML 模型
  ↓
STM32 端侧推理
  ↓
手势识别结果
```

因此，这次选择 **MPU6050 + STM32F103 + TinyML**，尝试完成一个真正运行在 MCU 上的手势识别系统。

这也是我第一次把“嵌入式驱动、数据采集、机器学习模型和 MCU 部署”放到同一个项目中。

---

## 二、项目总体架构

目前项目的核心硬件平台为 **STM32F103ZET6**，主控使用 Cortex-M3，系统通过 MPU6050 获取运动数据，并最终在 STM32 本地完成模型推理。

```text
             ┌──────────────┐
             │   MPU6050    │
             │ 三轴加速度数据 │
             └──────┬───────┘
                    │ I²C
                    ↓
             ┌──────────────┐
             │  STM32F103   │
             │ 驱动/采样/控制 │
             └──────┬───────┘
                    │ USART1
                    ↓
             ┌──────────────┐
             │    Python    │
             │ 自动化数据采集 │
             └──────┬───────┘
                    │ CSV
                    ↓
             ┌──────────────┐
             │ TinyML模型训练 │
             │ Edge Impulse │
             └──────┬───────┘
                    │ C++部署包
                    ↓
             ┌──────────────┐
             │  STM32端侧推理 │
             └──────┬───────┘
                    ↓
             ┌──────────────┐
             │ OLED / 串口输出 │
             │ 识别结果+概率  │
             └──────────────┘
```

项目的最终目标并不是“在电脑上调用一个模型”，而是让模型真正进入 MCU，在资源受限的嵌入式环境中完成推理。

---

## 三、硬件平台

| 模块 | 当前方案 |
|---|---|
| MCU | STM32F103ZET6 |
| CPU | ARM Cortex-M3 @ 72 MHz |
| 传感器 | MPU6050 |
| 通信 | I²C + USART1 |
| 显示 | 0.96 寸 I²C OLED |
| 调试 | 串口 / VOFA+ |
| ML方案 | Edge Impulse + TinyML |
| 手势类别 | 4 类 |

项目目前定义了 4 种类别：

- `idle`：静止 / 待机
- `wave_lr`：左右水平挥动
- `lift_ud`：上下抬动
- `shake_fast`：快速晃动

第一阶段每类采集 **15 个样本**，共 **60 个样本文件**。

这里需要特别说明：60 个样本并不是为了声称模型已经具有很强的泛化能力，而是首先验证“**数据采集 → 训练 → 部署 → MCU 推理**”这条完整技术链能够跑通。

---

## 四、第一步：把 MPU6050 驱动真正跑起来

项目首先要解决的问题不是模型，而是最基础的数据来源：**STM32 能不能稳定、正确地读取 MPU6050？**

项目中的 `mpu6050.h` 对器件地址和核心寄存器进行了定义，例如：

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

同时把读取结果整理成 `MPU6050_Data_t`：

```c
typedef struct {
    int16_t Accel_X_RAW;
    int16_t Accel_Y_RAW;
    int16_t Accel_Z_RAW;

    float Ax;
    float Ay;
    float Az;

    float Gx;
    float Gy;
    float Gz;

    float Temperature;
} MPU6050_Data_t;
```

这样做的好处是：底层寄存器读取和上层手势识别逻辑被分离开来。

后面模型需要的主要是三轴加速度，因此上层可以直接使用 `Ax / Ay / Az`，而不需要反复处理 16 位原始 ADC 数据。

> 项目源码：[`mpu6050.h`](https://github.com/nathanialkonggsd-hub/STM32-Gesture-TinyML/blob/main/STM32-Gesture-TinyML/02_Source_Code/Elite_F103_Demo/Core/Inc/mpu6050.h)

---

## 五、STM32端先完成“数据能看到”

在真正采集数据之前，我先让 STM32 把传感器状态和实时数据通过 USART1 输出。

主程序启动时会初始化 GPIO、USART1、I2C1 和 I2C2，然后分别检查 MPU6050 和 OLED 是否在线：

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

这种设计看起来很简单，但对后面的调试非常重要：

```text
传感器没数据
    ↓
先判断硬件/I²C
    ↓
再判断驱动
    ↓
再判断数据格式
    ↓
最后才进入模型问题
```

不要一看到模型识别错误，就直接怀疑神经网络。

嵌入式 AI 的问题往往发生在模型之外。

> 项目主程序：[`main.c`](https://github.com/nathanialkonggsd-hub/STM32-Gesture-TinyML/blob/main/STM32-Gesture-TinyML/02_Source_Code/Elite_F103_Demo/Core/Src/main.c)

---

## 六、按键触发采集：让 STM32 负责“采什么”

项目使用开发板上的 `KEY0` 作为一次手势采集的触发信号。

为了避免机械按键抖动和连续触发，代码没有简单地判断 GPIO 电平，而是加入了时间戳防抖和边沿检测：

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

这一步的意义是把一次物理按键动作转换成一次稳定的软件事件。

后续状态机再根据这个事件进入数据采集状态。

---

## 七、一次手势到底采多少数据？

当前工程把单次手势采集定义为：

```c
#define RECORD_SAMPLES 100
static float raw_features[RECORD_SAMPLES * 3];
```

也就是说，一次手势包含：

```text
100 个采样点
×
3 个加速度轴
=
300 个原始特征值
```

这就是后面模型输入的基础数据。

目前系统按照约 **50 Hz** 的节拍采集，因此单次窗口约覆盖 2 秒左右的运动过程。

这一设计同时兼顾了两个目标：

1. 能够覆盖一次完整的手势动作；
2. 不让 MCU 需要处理过长的数据窗口。

---

## 八、从 STM32 到 Python：把数据真正保存下来

STM32 负责产生数据，但如果每次都手动复制串口输出，效率会非常低。

因此我在项目中写了一个 Python 自动化采集工具 `collect_gestures.py`。

工具基于 `pyserial`，会自动创建 `dataset` 目录，并按照标签自动生成文件名：

```text
idle.01.csv
idle.02.csv
...
idle.15.csv

wave_lr.01.csv
...

lift_ud.01.csv
...

shake_fast.01.csv
...
```

脚本会监听：

```text
--- START_OF_GESTURE ---
```

检测到开始标志后，将串口中的采样数据暂存到缓冲区；检测到：

```text
--- END_OF_GESTURE ---
```

后，再统一写入 CSV 文件。

核心逻辑大致是：

```python
if "--- START_OF_GESTURE ---" in raw_line:
    recording = True
    buffer = []

if "--- END_OF_GESTURE ---" in raw_line:
    if recording and buffer:
        recording = False
        filepath = get_next_filename(label)
        with open(filepath, "w", encoding="utf-8") as f:
            for item in buffer:
                f.write(f"{item}\n")
```

相比人工复制数据，这种方式更适合后续不断扩充数据集。

> 项目源码：[`collect_gestures.py`](https://github.com/nathanialkonggsd-hub/STM32-Gesture-TinyML/blob/main/STM32-Gesture-TinyML/01_Data_Acquisition/collect_gestures.py)

---

## 九、一次串口问题：文档和代码必须同步

数据采集阶段还发现了一个很典型的工程问题。

最初的采集说明文档写的是 `COM7`，而 Python 程序实际配置为：

```python
PORT = "COM9"
BAUD_RATE = 115200
```

这并不是 TinyML 本身的问题，而是硬件开发过程中非常常见的**配置漂移**：设备更换了串口号，但说明文档没有同步修改。

后来已经完成修复并统一配置。

这个问题也让我意识到：

> **代码能运行 ≠ 项目文档正确。**

当项目开始出现 Python 工具、STM32 工程、数据集和多份说明文档以后，配置一致性本身就成为工程质量的一部分。

> 数据采集说明：[`collect_gestures.md`](https://github.com/nathanialkonggsd-hub/STM32-Gesture-TinyML/blob/main/STM32-Gesture-TinyML/01_Data_Acquisition/collect_gestures.md)

---

## 十、第一阶段数据集

当前第一阶段最终形成了 4 类手势：

| Label | 含义 | 样本数 |
|---|---|---:|
| `idle` | 静止 | 15 |
| `wave_lr` | 左右挥动 | 15 |
| `lift_ud` | 上下抬动 | 15 |
| `shake_fast` | 快速晃动 | 15 |
| **合计** | **4 类** | **60** |

这批数据的主要目的，是先把完整流程验证起来。

所以当前阶段不应该把 60 个样本包装成“大规模数据集”。真正需要在后续阶段解决的是：

- 同一种手势不同速度是否仍然能够识别？
- 动作幅度变化后是否还能识别？
- 不同握持姿态是否影响结果？
- 不同使用者的数据是否仍然有效？
- `wave_lr`、`shake_fast` 等相似动作之间是否会混淆？

这些问题才决定模型最终能不能从“能跑”走向“好用”。

---

## 十一、项目图片与实际运行资料

项目运行过程中的照片、模型特征图、训练结果以及 OLED 输出截图均已经整理在项目仓库的 `03_Images_and_Results` 目录中。

其中包括：

- STM32F103 开发板实物图
- Edge Impulse V1 / V2 特征空间图
- 模型训练指标图
- OLED 采样进度显示
- OLED 待机状态显示
- OLED 最终识别结果
- 低置信度问题复现截图

项目图片目录：

[`03_Images_and_Results`](https://github.com/nathanialkonggsd-hub/STM32-Gesture-TinyML/tree/main/STM32-Gesture-TinyML/03_Images_and_Results)

例如项目中的开发板照片：

![STM32F103ZET6 开发板](https://raw.githubusercontent.com/nathanialkonggsd-hub/STM32-Gesture-TinyML/main/STM32-Gesture-TinyML/03_Images_and_Results/STMF103ZET6.jpg)

采样与识别结果相关图片也会在后续文章中继续引用，而不是把所有图片一次性堆在本篇文章中。

---

## 十二、目前已经完成到什么程度？

截至本篇，项目已经完成了第一条完整的数据链路：

```text
MPU6050
  ↓
STM32驱动
  ↓
实时读取三轴加速度
  ↓
KEY0触发采样
  ↓
USART1输出
  ↓
Python自动采集
  ↓
CSV数据集
  ↓
4类 × 15样本
```

更重要的是，项目后续的 TinyML 模型已经生成，并已经能够在 STM32 上运行。因此本篇的数据采集并不是孤立实验，而是整个端侧 AI 系统的输入端。

后续文章会把已经完成的模型训练、模型部署和端侧推理拆开详细记录。

---

## 十三、这一阶段最大的收获

这次项目让我第一次比较明确地感受到：

**嵌入式 AI 不是“把一个模型扔进单片机”这么简单。**

真正的工程链条至少包括：

```text
硬件
 ↓
驱动
 ↓
采样
 ↓
数据格式
 ↓
数据集
 ↓
特征工程
 ↓
模型
 ↓
部署
 ↓
内存
 ↓
推理性能
 ↓
用户交互
```

其中任何一个环节出现问题，最后都可能表现成“模型识别不准”。

所以这次项目的重点并不是单纯追求一个漂亮的识别率，而是先把整个系统真正跑起来。

---

# 十四、系列进度

### 📚 STM32 + TinyML 端侧手势识别系列

| # | 文章主题 | 状态 |
|---:|---|:---:|
| **72** | 项目启动：系统架构与 MPU6050 数据采集 | 🟢 **本文** |
| **73** | 数据采集：STM32 → Python → CSV 数据集 | ⏳ 待更新 |
| **74** | TinyML：特征提取与模型训练 | ⏳ 待更新 |
| **75** | 模型部署：TinyML 模型移植到 STM32 | ⏳ 待更新 |
| **76** | 端侧推理：STM32 实时手势识别 | ⏳ 待更新 |
| **77** | 工程优化：低置信度、Bug 定位与性能优化 | ⏳ 待更新 |
| **78** | 第一阶段总结：完整系统跑通 | ⏳ 待更新 |
| **79+** | 第二阶段：数据集扩充与模型泛化优化 | 🔮 规划中 |

> **系列持续更新中。下一篇将重点记录数据采集工具、CSV 数据组织以及数据集构建过程。**

---

## 十五、项目地址

**GitHub：**

https://github.com/nathanialkonggsd-hub/STM32-Gesture-TinyML

项目目录中保留了 STM32 工程、数据采集工具、数据集、模型部署文件以及运行结果图片，本文中的代码和图片均尽量对应实际项目资料，而不是凭空编写示例。

---

## 结语

72 篇文章之后，我开始尝试改变一种记录方式：

以前是：

> 学到一个知识点 → 写一篇文章。

现在更希望变成：

> **确定一个问题 → 做项目 → 写代码 → 实验 → 遇到 Bug → 解决 → 总结 → 形成系列。**

这次 STM32 + TinyML 手势识别项目，就是这个转变的开始。

第一阶段先把数据链路跑通，下一阶段继续向真正的 **Embedded AI / TinyML 工程实践**推进。
