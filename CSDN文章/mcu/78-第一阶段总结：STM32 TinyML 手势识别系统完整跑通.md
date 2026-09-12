---
number: 78
category:
  - MCU
  - TinyML
  - Embedded AI
series: STM32
level: Advanced
type: Project Summary
status: Completed
---

@[TOC](文章目录)

# 第一阶段总结：STM32 TinyML 手势识别系统完整跑通

> **STM32 + TinyML 端侧手势识别系列 · 第 78 篇**
>
> 🏁 **第一阶段总结篇**：从传感器、数据采集，到 TinyML 训练、模型部署、STM32 端侧推理和工程优化，完整复盘这一阶段真正跑通的内容。

---

## 一、从“学知识”到“跑系统”

做到第 78 篇，我越来越明显地感觉到：一个真正有价值的嵌入式项目，并不是把某个传感器驱动出来就结束了。

真正完整的工程链路应该是：

```text
硬件
 ↓
传感器采集
 ↓
数据集构建
 ↓
Feature / DSP
 ↓
模型训练
 ↓
模型部署
 ↓
MCU 端侧推理
 ↓
显示 / 通信
 ↓
问题定位与优化
```

这也是这一阶段最重要的变化：

> **从“知识点驱动”转向“项目驱动”。**

---

## 二、第一阶段硬件系统

| 模块 | 方案 |
|:---:|:---|
| MCU | STM32F103ZET6 |
| CPU | Cortex-M3 @ 72 MHz |
| SRAM | 64 KB |
| Flash | 512 KB |
| 传感器 | MPU6050 |
| 显示 | SSD1306 OLED |
| 通信 | USART1 / I²C |
| TinyML | Edge Impulse |

核心数据链路：

```text
MPU6050
   ↓ I²C
STM32F103ZET6
   ↓ USART1
Python
   ↓ CSV
Edge Impulse
   ↓ C++
STM32F103
   ↓
OLED / UART
```

---

## 三、数据集：4 类 × 15 个样本

当前第一阶段数据集：

| Label | 含义 | 样本 |
|:---:|:---|---:|
| `idle` | 静止 | 15 |
| `wave_lr` | 左右挥动 | 15 |
| `lift_ud` | 上下抬动 | 15 |
| `shake_fast` | 快速晃动 | 15 |
| **合计** | **4 类** | **60** |

一次手势使用：

```text
100 samples × 3 axes = 300 raw values
```

> ⚠️ 当前 60 个样本的主要作用是验证工程链路，而不是证明真实场景中的泛化能力。

---

## 四、模型：V1 → V2

第一版模型首先建立 baseline。

V1 暴露的问题包括：

- 特征处理约 11 ms；
- 特征空间存在部分混杂；
- 个别样本存在误分类点。

之后通过 Feature / DSP 参数调整得到 V2：

```text
V1 ≈ 11 ms
     ↓
Feature / DSP 优化
     ↓
V2 ≈ 2 ms
```

![V2 特征空间](https://cdn.jsdelivr.net/gh/nathanialkonggsd-hub/STM32-Gesture-TinyML@main/STM32-Gesture-TinyML/03_Images_and_Results/ei_feature_explorer_V2.png#pic_center)

当前数据集 / 当前验证划分下，V2 验证结果达到 **100%**。

> ⚠️ **必须强调**：100% 验证结果 ≠ 真实环境 100% 准确率。第二阶段必须使用更大、更丰富且独立的测试数据验证泛化能力。

---

## 五、模型部署：真正进入 STM32

模型部署过程中，真正花时间的并不是“把文件复制进 Keil”，而是解决资源问题。

### 1. Semihosting

关闭不必要的调试运行时依赖，避免 MCU 脱离调试器后卡死。

### 2. DSP Heap `-1002`

通过检查 SRAM、Heap、Stack、全局缓冲区和模型中间数据，调整内存布局。

### 3. TFLite Micro 依赖

避免把不必要的完整 Runtime 依赖全部带入 STM32F103，精简模型运行时接口并采用 Safe Stub 等工程方式。

### 4. HardFault

重点检查：

```text
Stack
Heap
指针
数组边界
模型依赖
DSP 缓冲区
```

---

## 六、端侧推理：从输入到结果

当前模型调用链：

```text
300 float
   ↓
signal_t
   ↓
run_classifier()
   ↓
4 类 probability
   ↓
argmax
   ↓
class_id + confidence
```

程序同时使用：

```text
MODE_MONITOR
      ↓
MODE_RECORDING
      ↓
MODE_SHOW_RESULT
```

完成一次完整识别。

![OLED 手势识别结果](https://cdn.jsdelivr.net/gh/nathanialkonggsd-hub/STM32-Gesture-TinyML@main/STM32-Gesture-TinyML/03_Images_and_Results/oled_result_display.jpg#pic_center)

---

## 七、工程优化：72 ms → 32 ms

项目没有只凭感觉判断“速度变快了”，而是通过 `HAL_GetTick()` 实际测量：

```c
uint32_t t_start = HAL_GetTick();
Model_Run_Inference(...);
last_infer_time = HAL_GetTick() - t_start;
```

结果从：

```text
≈ 72 ms
```

优化到：

```text
≈ 32 ms
```

同时处理了：

- 传感器单位不一致；
- CMSIS-DSP / RFFT 约束；
- DSP Heap 问题；
- HardFault；
- 单轴调试盲区；
- OLED 显示空间有限。

> 🎯 **这一阶段真正验证的不是一个数字，而是“测量 → 定位 → 修改 → 再测量”的工程闭环。**

---

## 八、目前系统已经具备什么能力？

### 已经完成

- [x] MPU6050 驱动
- [x] STM32 三轴数据采集
- [x] Python 自动化数据集采集
- [x] CSV 数据管理
- [x] Edge Impulse Feature / DSP
- [x] V1 → V2 模型迭代
- [x] TinyML 模型部署
- [x] STM32 端侧推理
- [x] OLED / UART 输出
- [x] Bug 定位与性能优化

### 还没有解决

- [ ] 大规模数据集
- [ ] 不同用户泛化
- [ ] 不同动作幅度 / 速度泛化
- [ ] 独立测试集
- [ ] 长时间稳定性验证
- [ ] 第二阶段模型迭代

---

## 九、项目仓库结构

当前项目核心结构：

```text
STM32-Gesture-TinyML/
├── 01_Data_Acquisition/
│   ├── collect_gestures.py
│   ├── collect_gestures.md
│   └── dataset/
│
├── 02_Source_Code/
│   └── Elite_F103_Demo/
│
├── 03_Images_and_Results/
│   ├── feature images
│   ├── training metrics
│   └── OLED results
│
└── Edge Impulse MCU project ZIP
```

这使得项目已经不只是“一个 Keil 工程”，而是包含：

> **代码 + 数据 + 模型 + 结果 + 问题复盘** 的完整项目档案。

---

## 十、第一阶段最大的收获

### ① 学会了把不同技术串起来

C / C++、STM32、Python、TinyML、OLED、串口，不再是孤立知识点。

### ② 开始关注工程约束

模型大小、RAM、Heap、Stack、推理时间和数据格式都成为必须考虑的对象。

### ③ 开始形成项目化思维

```text
学习知识
   ↓
解决问题
   ↓
形成代码
   ↓
留下数据
   ↓
形成文章
   ↓
沉淀成项目
```

这比单纯“写完一篇教程”更有价值。

---

## 十一、第二阶段：从“能跑”走向“好用”

第二阶段不再重复部署流程，而是集中解决模型泛化问题。

计划方向：

```text
扩大数据集
   ↓
增加动作速度 / 幅度变化
   ↓
增加握持姿态变化
   ↓
尽可能引入更多使用者数据
   ↓
训练 / 验证 / 测试严格分离
   ↓
分析混淆矩阵
   ↓
优化模型
   ↓
重新部署 STM32
```

下一篇将进入：

> **STM32 TinyML 第二阶段：扩充数据集与提升模型泛化能力。**

---

## 系列进度

| # | 主题 | 状态 |
|---:|:---|:---:|
| 72 | 项目启动：系统架构与 MPU6050 数据采集 | ✅ |
| 73 | STM32 手势数据采集：从串口到 CSV 数据集 | ✅ |
| 74 | TinyML 特征提取与模型训练：V1 → V2 | ✅ |
| 75 | TinyML 模型部署：Edge Impulse → STM32 | ✅ |
| 76 | STM32 端侧实时手势推理 | ✅ |
| 77 | TinyML 工程优化：从 HardFault 到 32ms 推理 | ✅ |
| **78** | **第一阶段总结：STM32 TinyML 手势识别系统完整跑通** | 🟢 本文 |
| 79+ | 第二阶段：数据集扩充与泛化优化 | 🔮 |

---

## 结语

第一阶段真正完成的，不只是一个“能识别手势”的 Demo。

更重要的是，我第一次把：

> **传感器 → 数据 → AI → MCU → 工程优化**

串成了一条完整链路。

下一阶段，目标也会发生变化：

> **不再只追求“能跑”，而是开始追求“可靠、可复现、可泛化”。**

**项目状态：** 🚧 持续开发中