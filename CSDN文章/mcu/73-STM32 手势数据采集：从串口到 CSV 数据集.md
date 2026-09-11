---
number: 73
category:
  - MCU
  - TinyML
  - Embedded AI
series: STM32
level: Advanced
type: Project
status: Completed
---

# STM32 手势数据采集：从串口到 CSV 数据集

> STM32 + MPU6050 + Python + CSV + TinyML 项目实战系列 · 第 2 篇
>
> 本篇继续上一篇的项目启动工作，重点解决一个真正的工程问题：**如何把 STM32 采集到的传感器数据，稳定、自动地变成可以直接用于 TinyML 训练的数据集。**

上一篇已经完成了 MPU6050 驱动、KEY0 触发采样以及 STM32 端的数据采集链路。本篇不再重复驱动部分，而是把重点放到“数据离开 MCU 以后发生了什么”。

---

## 一、为什么不能只在串口助手里看数据？

最开始做传感器实验时，串口打印几行数据就足够判断 MPU6050 是否正常：

```text
-0.006,0.028,0.953
-0.011,0.011,0.970
-0.002,0.028,0.953
...
```

但是，如果目标是训练一个手势识别模型，仅仅“看到数据”远远不够。

我们需要完成：

```text
STM32
  │
  │ MPU6050
  ▼
三轴加速度数据
  │
  │ USART1 / 115200
  ▼
PC 串口
  │
  │ Python 自动接收
  ▼
CSV 文件
  │
  ├── idle
  ├── wave_lr
  ├── lift_ud
  └── shake_fast
  │
  ▼
TinyML 数据集
```

这样，传感器实验才真正从“嵌入式调试”进入了“数据工程”。

---

## 二、当前项目的数据格式

项目中的实际数据文件使用 CSV 保存，字段为：

```csv
timestamp,accX,accY,accZ
0,-0.006,0.028,0.953
20,-0.011,0.011,0.970
40,-0.002,0.028,0.953
60,-0.007,0.031,0.953
```

这里有四列：

| 字段 | 含义 |
|---|---|
| `timestamp` | 当前采样时间，单位 ms |
| `accX` | X 轴加速度 |
| `accY` | Y 轴加速度 |
| `accZ` | Z 轴加速度 |

例如项目中的 `idle.01.csv` 从 `0 ms` 开始，以 `20 ms` 为间隔记录数据，直到约 `1980 ms`，对应约 50 Hz 的采样节奏。实际文件中保存的是完整采样序列，而不是人工复制串口输出。fileciteturn36file0

> **为什么保留 timestamp？**
>
> 因为时间轴本身就是时序数据的重要组成部分。即使后续模型训练阶段主要使用加速度序列，保留时间戳也方便检查采样频率、数据丢失以及采集过程是否稳定。

---

## 三、Python 自动采集程序

本项目没有采用“串口助手 → 手动复制 → 新建 CSV”的低效方式，而是自己写了一个 Python 自动化工具。

核心脚本：

```text
01_Data_Acquisition/
├── collect_gestures.py
├── collect_gestures.md
├── test_read.py
└── dataset/
```

其中 `collect_gestures.py` 使用 `pyserial` 与 STM32 通信，并自动创建数据目录。程序当前配置为 `COM9 / 115200`。fileciteturn31file0

核心配置：

```python
import os
import glob
import serial
import time

PORT = "COM9"
BAUD_RATE = 115200
SAVE_DIR = "dataset"

os.makedirs(SAVE_DIR, exist_ok=True)
```

这里有三个很典型的工程细节：

1. 串口号集中配置，不写死在多个函数里。
2. 波特率固定为 `115200`，与 STM32 USART1 配置保持一致。
3. 启动时自动创建 `dataset` 目录，避免第一次运行时报目录不存在。

---

## 四、为什么要自动编号文件？

如果手工保存数据，很容易出现：

```text
idle.csv
idle_new.csv
idle_final.csv
idle_final2.csv
idle_final_really.csv
```

对于机器学习项目，这种命名方式很快就会失控。

所以脚本采用：

```text
idle.01.csv
idle.02.csv
...
idle.15.csv
```

这样的规则。

脚本会扫描当前目录中已有的同标签文件，找到最大的编号，再自动加 1：

```python
def get_next_filename(label: str) -> str:
    pattern = os.path.join(SAVE_DIR, f"{label}.*.csv")
    existing_files = glob.glob(pattern)
    indices = []

    for f in existing_files:
        try:
            name_part = os.path.basename(f).split(".")
            if len(name_part) >= 3 and name_part[-1] == "csv":
                indices.append(int(name_part[1]))
        except ValueError:
            continue

    next_idx = max(indices, default=0) + 1
    return os.path.join(SAVE_DIR, f"{label}.{next_idx:02d}.csv")
```

这个函数虽然不复杂，但它体现了一个很重要的思路：

> **让工具负责重复劳动，把人的注意力留给实验本身。**

---

## 五、串口打开方式也有讲究

项目中专门设置了：

```python
ser = serial.Serial()
ser.port = PORT
ser.baudrate = BAUD_RATE
ser.timeout = 1.0
ser.dtr = False
ser.rts = False
ser.open()
```

其中 `dtr=False`、`rts=False` 是针对实际开发板串口使用场景做的兼容设置。

另外，开发过程中还遇到过一个很典型的问题：

> **VOFA+ 和 Python 不能同时独占同一个串口。**

也就是说，如果 VOFA+ 正连接着 `COM9`，Python 再尝试打开它，就可能直接失败。

这个问题后来已经修复，当前数据采集脚本可以正常使用 `COM9`。

项目中还保留了一个更简单的 `test_read.py`，它直接打开 `COM9`、115200 波特率并持续读取原始串口数据，用于排查“到底是采集逻辑有问题，还是串口本身没有数据”的问题。fileciteturn34file0

这也是我觉得比较值得保留的一类工程代码：

```text
完整程序
   ↓
发现问题
   ↓
拆出最小测试程序
   ↓
确认问题边界
   ↓
回到完整系统
```

比直接在几百行代码里盲目加 `printf` 高效得多。

---

## 六、如何判断一次采集真正开始？

Python 并不是收到一行数据就立即保存，而是通过 STM32 输出的标志判断一次完整动作的边界。

脚本监听：

```text
--- START_OF_GESTURE ---
```

收到后：

```python
recording = True
buffer = []
```

之后把包含逗号的数据行放进缓冲区：

```python
if recording:
    if "," in raw_line:
        buffer.append(raw_line)
```

直到收到：

```text
--- END_OF_GESTURE ---
```

才一次性写入 CSV。

这样设计比“固定等待 2 秒然后强行保存”更可靠，因为**数据是否完整由设备端的开始/结束标志决定**。

完整流程可以理解成：

```text
等待串口
   │
   ▼
START_OF_GESTURE
   │
   ▼
开始缓存三轴数据
   │
   ▼
持续接收
   │
   ▼
END_OF_GESTURE
   │
   ▼
生成唯一文件名
   │
   ▼
写入 CSV
```

---

## 七、四种手势标签

当前第一阶段数据集采用 4 个类别：

| 标签 | 含义 |
|---|---|
| `idle` | 静止 / 待机 |
| `wave_lr` | 左右摇动 |
| `lift_ud` | 上下运动 |
| `shake_fast` | 快速前后/抖动动作 |

这里有一个值得注意的地方：

**标签不是随便起的。**

后续 TinyML 模型输出的分类结果，最终就会对应这些 label。

所以从一开始就应该保持：

```text
采集标签
    =
训练标签
    =
模型输出类别
```

如果后面把 `wave_lr` 改成 `left_right`，却只修改了数据集而没有修改模型配置，就可能出现数据集、训练工程和 MCU 推理结果之间的不一致。

---

## 八、实际数据集规模

第一阶段采用每类 15 次采集：

```text
idle       15
wave_lr    15
lift_ud    15
shake_fast 15
----------------
总计       60 次采集
```

项目中的采集说明也采用了连续按 15 次 KEY0、自动生成 `01` 到 `15` 文件的流程，然后依次切换到其他标签。fileciteturn37file0

这里需要特别说明：

> **60 次采集不是为了声称模型已经具备很强的泛化能力。**

它的主要目标是验证完整技术链：

```text
传感器
 → STM32
 → 串口
 → Python
 → CSV
 → 数据集
 → TinyML
 → MCU 推理
```

对于项目第一阶段来说，先把这条链跑通，比一开始追求庞大的数据集更重要。

---

## 九、一份 CSV 到底长什么样？

以仓库中的 `idle.01.csv` 为例，开头数据是：

```csv
timestamp,accX,accY,accZ
0,-0.006,0.028,0.953
20,-0.011,0.011,0.970
40,-0.002,0.028,0.953
60,-0.007,0.031,0.953
80,-0.004,0.029,0.957
100,-0.007,0.030,0.959
```

可以明显看到，在 `idle` 状态下，三轴数据整体变化幅度比较小，而 Z 轴保持在接近重力加速度的量级。

但这里**不要急着直接把“某个轴接近 1g”当成分类规则**。

TinyML 的意义就在于：让模型从完整的时序变化中学习类别特征，而不是人工写死：

```c
if (accX > 某个值)
    return LEFT;
```

这也是从传统嵌入式规则判断走向机器学习分类的关键一步。

---

## 十、为什么数据采集本身就是一个工程问题？

做到这里，我发现 TinyML 项目最容易被低估的部分其实不是模型，而是**数据**。

一个模型最终表现不好，可能根本不是模型结构的问题，而是：

- 采样频率不稳定
- 数据长度不一致
- 标签混乱
- 串口丢数据
- 动作幅度差异太大
- 采集姿态过于单一
- 不同类别之间存在明显重叠
- 数据量太少

所以整个项目实际上应该拆成：

```text
数据质量
   ↓
特征质量
   ↓
模型训练
   ↓
模型部署
   ↓
端侧推理
```

而不是简单理解成：

```text
“找个模型 → 训练 → 下载到 STM32”
```

---

## 十一、下一步：进入 TinyML 特征提取与训练

现在数据已经具备了进入机器学习流程的基本条件。

下一篇将进入真正的 TinyML 环节：

```text
CSV 数据集
   ↓
数据检查
   ↓
Feature / DSP
   ↓
特征空间观察
   ↓
模型训练
   ↓
Accuracy / Confusion Matrix
   ↓
选择适合 MCU 的模型
```

这里会重点讨论一个问题：

> **为什么同样是三轴加速度数据，直接把原始数据塞进模型，和先做特征提取，最终得到的模型效果可能完全不同？**

这才是下一阶段真正值得学习的部分。

---

## 十二、本篇小结

本篇没有写复杂算法，但完成了 TinyML 项目中非常关键的一层：**数据工程链路**。

现在已经可以做到：

- STM32 负责采集 MPU6050 三轴加速度
- USART1 以 115200 与 PC 通信
- Python 自动监听串口
- KEY0 触发一次完整采集
- `START/END` 标志划分动作边界
- 自动生成递增 CSV 文件名
- 按四种手势分类保存
- 第一阶段形成 4 × 15 = 60 次采集的数据集

到这里，项目已经真正从：

> **“STM32 读取传感器”**

进入了：

> **“STM32 为机器学习提供数据”**。

这两个阶段看起来只差一步，实际上思维方式已经发生了变化。

---

## 📚 STM32 + TinyML 系列进度

| # | 主题 | 状态 |
|---:|---|:---:|
| 72 | 项目启动：系统架构与 MPU6050 数据采集 | ✅ 已完成 |
| 73 | 数据采集：STM32 → Python → CSV 数据集 | 🟢 本文 |
| 74 | TinyML：特征提取与模型训练 | ⏳ 待更新 |
| 75 | 模型部署：TinyML → STM32 | ⏳ 待更新 |
| 76 | STM32 端侧实时推理 | ⏳ 待更新 |
| 77 | 低置信度、Bug 定位与性能优化 | ⏳ 待更新 |
| 78 | 第一阶段项目总结：完整系统跑通 | ⏳ 待更新 |
| 79+ | 第二阶段：数据集扩充与模型泛化优化 | 🔮 规划中 |

---

## 🔗 项目源码

完整项目持续更新：

`nathanialkonggsd-hub/STM32-Gesture-TinyML`

数据采集相关代码位于：

```text
STM32-Gesture-TinyML/
└── 01_Data_Acquisition/
    ├── collect_gestures.py
    ├── collect_gestures.md
    ├── test_read.py
    └── dataset/
```

本篇中的代码、数据格式和采集流程均以项目仓库当前实际文件为准。fileciteturn31file0turn34file0turn36file0

---

**下一篇：74｜TinyML：特征提取与模型训练**
