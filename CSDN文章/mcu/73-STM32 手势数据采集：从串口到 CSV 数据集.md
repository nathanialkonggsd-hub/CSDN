> 🧭 **文章档案｜STM32 TinyML 手势识别系列**
>
> **📌 编号**：No.73　｜　**🚀 阶段**：第一阶段 · 数据工程　｜　**🛠️ 类型**：实战记录
>
> **🔩 技术栈**：STM32F103ZET6 · MPU6050 · USART1 · Python · PySerial · CSV
>
> **🎯 本篇关键词**：串口采集 · 数据边界 · 自动编号 · 标签体系 · 数据集管理
>
> **✨ 本篇特色**：不再只是“把数据打印出来”，而是把 STM32 产生的数据真正变成可以训练、可以复用、可以继续扩充的数据资产。
>
> **🧪 实战状态**：已跑通　｜　**📈 项目主线**：硬件 → 数据 → 数据集 → 模型

@[TOC](文章目录)

# STM32 手势数据采集：从串口到 CSV 数据集

> **STM32 + MPU6050 + Python + CSV + TinyML 项目实战系列 · 第 73 篇**
>
> 🎯 **本文重点**：把 STM32 产生的三轴传感器数据，稳定、自动地转换成可训练的数据集。

---

## 一、为什么不能只在串口助手里看数据？

串口打印可以证明 MPU6050 正常，但 TinyML 真正需要的是**结构化、可复用、可扩充的数据集**。

```text
STM32
  ↓ USART1 / 115200
PC 串口
  ↓
Python
  ↓
CSV
  ↓
TinyML Dataset
```

> 💡 **关键变化**：从“看数据”升级到“管理数据”。

---

## 二、当前 CSV 数据格式

项目中的数据按时间序列保存：

```csv
timestamp,accX,accY,accZ
0,-0.006,0.028,0.953
20,-0.011,0.011,0.970
40,-0.002,0.028,0.953
60,-0.007,0.031,0.953
```

| 字段 | 含义 |
|:---:|:---|
| `timestamp` | 采样时间，ms |
| `accX` | X 轴加速度 |
| `accY` | Y 轴加速度 |
| `accZ` | Z 轴加速度 |

例如 `idle.01.csv` 从 `0 ms` 开始，以约 `20 ms` 的间隔记录到约 `1980 ms`，对应约 50 Hz。

> ⚠️ **不要忽略 timestamp。** 它可以帮助检查采样频率、丢样和数据完整性。

---

## 三、Python 自动采集程序

核心脚本：

```text
01_Data_Acquisition/
├── collect_gestures.py
├── collect_gestures.md
├── test_read.py
└── dataset/
```

当前配置：

```python
PORT = "COM9"
BAUD_RATE = 115200
SAVE_DIR = "dataset"
```

启动时自动创建数据目录：

```python
os.makedirs(SAVE_DIR, exist_ok=True)
```

这几个细节虽然简单，却能明显降低实验过程中的重复劳动。

---

## 四、为什么要自动编号文件？

如果人工保存，很容易变成：

```text
idle.csv
idle_new.csv
idle_final.csv
idle_final2.csv
```

因此项目统一采用：

```text
idle.01.csv
idle.02.csv
...
idle.15.csv
```

程序会扫描已有文件，找到最大编号后自动 +1：

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

> 🎯 **原则**：让程序负责重复劳动，把注意力留给实验本身。

---

## 五、串口打开方式：先解决“能不能稳定读”

项目使用：

```python
ser = serial.Serial()
ser.port = PORT
ser.baudrate = BAUD_RATE
ser.timeout = 1.0
ser.dtr = False
ser.rts = False
ser.open()
```

开发阶段还遇到过：**VOFA+ 和 Python 不能同时独占 COM9**。

排查串口问题时，我没有直接修改完整程序，而是保留了 `test_read.py` 这个最小测试程序：

```text
完整系统
   ↓
发现串口异常
   ↓
最小化为 test_read.py
   ↓
确认串口本身是否有数据
   ↓
回到完整采集程序
```

> 💡 这也是嵌入式调试里非常实用的思路：**先缩小问题边界，再修改主程序。**

---

## 六、一次采集什么时候开始、什么时候结束？

Python 不会收到一行就保存，而是依靠 STM32 的边界标志：

```text
--- START_OF_GESTURE ---
        ↓
      开始缓存
        ↓
    三轴数据持续进入
        ↓
--- END_OF_GESTURE ---
        ↓
     生成唯一文件名
        ↓
       写入 CSV
```

核心逻辑：

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

相比“固定等待 2 秒然后保存”，这种设计更容易保证数据边界完整。

---

## 七、四种手势标签

| 标签 | 含义 |
|:---:|:---|
| `idle` | 静止 / 待机 |
| `wave_lr` | 左右摇动 |
| `lift_ud` | 上下运动 |
| `shake_fast` | 快速前后 / 抖动 |

必须保证：

```text
采集标签
   =
训练标签
   =
模型输出类别
```

标签一旦进入数据集和模型配置，后续不要随意改名。

---

## 八、第一阶段数据规模

```text
idle        15
wave_lr     15
lift_ud     15
shake_fast  15
----------------
总计        60
```

第一阶段的目标不是追求“大数据集”，而是先验证完整链路：

```text
MPU6050 → STM32 → USART1 → Python → CSV → TinyML → STM32
```

> ⚠️ 60 个样本不能代表真实场景中的泛化能力。第二阶段将重点扩充数据量、动作变化和测试集。

---

## 九、一份 CSV 到底长什么样？

```csv
timestamp,accX,accY,accZ
0,-0.006,0.028,0.953
20,-0.011,0.011,0.970
40,-0.002,0.028,0.953
60,-0.007,0.031,0.953
80,-0.004,0.029,0.957
100,-0.007,0.030,0.959
```

`idle` 状态下三轴变化通常较小，Z 轴接近重力加速度量级。

但不能简单写成：

```c
if (accX > 某个值)
    return LEFT;
```

TinyML 的目标是从**完整的时序变化**中学习类别特征。

---

## 十、数据采集为什么本身就是工程问题？

模型效果不好，不一定是模型结构的问题。

常见根因包括：

- 采样频率不稳定
- 数据长度不一致
- 标签混乱
- 串口丢数据
- 动作幅度差异过大
- 采集姿态过于单一
- 类别之间存在重叠
- 数据量太少

因此项目的数据链路应该理解为：

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

---

## 十一、采集链路实物记录

当前项目的采样过程已经在 OLED 上加入进度显示，后续可直接对应每一次 100 点采样窗口。

> 🖼️ **图片 01｜OLED 采样进度**
>
> CSDN 编辑器无法稳定转存当前外部图片地址时，请将原图下载到本地，再通过 CSDN 工具栏「图片」重新上传。
>
> 原图地址：
> https://github.com/nathanialkonggsd-hub/STM32-Gesture-TinyML/blob/main/STM32-Gesture-TinyML/03_Images_and_Results/oled_sampling_progress.jpg

---

## 十二、本篇小结

本篇完成了 TinyML 项目最容易被低估的一环：**数据工程**。

现在已经具备：

- STM32 自动输出数据
- Python 自动接收
- 自动编号保存
- 明确的标签体系
- 4 类 × 15 个样本
- CSV 数据集

下一篇正式进入：

> **CSV → 特征提取 → 特征空间 → 模型训练**

---

## 📚 系列导航

| # | 主题 | 状态 |
|---:|:---|:---:|
| 72 | 项目启动：系统架构与 MPU6050 数据采集 | ✅ |
| **73** | **STM32 手势数据采集：从串口到 CSV 数据集** | 🟢 本文 |
| 74 | TinyML 特征提取与模型训练：V1 → V2 | ⏳ |
| 75 | TinyML 模型部署：Edge Impulse → STM32F103 | ⏳ |
| 76 | STM32 端侧实时手势推理 | ⏳ |
| 77 | TinyML 工程优化：从 HardFault 到 32ms 推理 | ⏳ |
| 78 | 第一阶段总结：STM32 TinyML 手势识别系统完整跑通 | ⏳ |
| 79+ | 第二阶段：数据集扩充与泛化优化 | 🔮 |

## 🔗 项目地址

- [**STM32 TinyML 项目仓库**：https://github.com/nathanialkonggsd-hub/STM32-Gesture-TinyML](https://github.com/nathanialkonggsd-hub/STM32-Gesture-TinyML)

## 👋 结语

感谢阅读！如果这篇文章对你有帮助，欢迎收藏、点赞，也欢迎在评论区交流。

项目仍在持续迭代，后续会继续记录从“能跑”到“可靠、可复现、可泛化”的过程。
