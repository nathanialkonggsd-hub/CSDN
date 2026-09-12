---
number: 75
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

# TinyML 模型部署：从 Edge Impulse 到 STM32F103

> **STM32 + TinyML 端侧手势识别系列 · 第 75 篇**
>
> 🎯 **本文重点**：把训练完成的 TinyML 模型真正带进 STM32F103，并解决部署阶段的典型资源与运行时问题。

---

## 一、模型训练完成，只是完成了一半

前面已经完成：

```text
MPU6050
  ↓
STM32
  ↓
Python / CSV
  ↓
Feature / DSP
  ↓
Edge Impulse
  ↓
V2 模型
```

现在的问题变成：

> **这个模型能不能在一颗只有有限 Flash / SRAM 的 STM32F103 上稳定运行？**

这一步才真正进入 Embedded AI。

---

## 二、为什么选择静态部署思路？

MCU 端最怕“运行时再动态加载一堆东西”。

因此本项目优先使用 Edge Impulse 的 MCU / EON Compiler 方案，将模型和 DSP 代码生成适合嵌入式工程集成的 C++ 文件。

整体结构可以理解成：

```text
Edge Impulse
    ↓
生成 C++ 模型包
    ↓
Keil / MDK 工程
    ↓
STM32F103
    ↓
静态链接 / 固定内存
    ↓
端侧推理
```

> 💡 **核心原则**：在 MCU 上，能在编译期确定的东西，尽量不要留到运行时。

---

## 三、第一次真正的坑：Semihosting

模型刚集成进 Keil 后，一个非常典型的问题出现了：程序可能在启动阶段直接卡住。

排查后发现是 **Semihosting**。

Semihosting 会让某些标准输入输出操作依赖调试器。对于最终脱离调试器独立运行的 MCU 程序，这种依赖很危险。

因此工程中需要关闭不必要的 Semihosting，并确保日志走 USART 等真实硬件接口。

> ⚠️ **经验**：看到“程序莫名其妙卡死”，不要只盯着模型。先检查 C 库、调试选项和运行时依赖。

---

## 四、第二个坑：DSP Heap `-1002`

进入 DSP / 特征处理后，又遇到内存不足问题。

可以把 MCU 内存理解成：

```text
Flash → 程序 / 模型 / 常量
SRAM  → 栈 / 堆 / 全局变量 / 中间缓冲区
```

模型能编译通过，不代表运行时一定有足够 SRAM。

本项目出现过 DSP heap `-1002`，最后通过检查内存布局、调整堆空间以及减少不必要的运行时开销解决。

> 🔧 **排查顺序**：
>
> 1. 看错误码；
> 2. 看 map 文件；
> 3. 看 heap / stack；
> 4. 看全局数组；
> 5. 再决定是否改模型。

---

## 五、第三个坑：TFLite Micro 依赖过重

如果直接把完整动态解释器相关依赖全部带入 STM32F103，资源压力会明显增加。

项目中实际遇到过：

```text
模型本身没问题
        ↓
运行时依赖过多
        ↓
RAM / Flash 压力增加
        ↓
甚至触发 HardFault
```

因此这里采用了更偏工程化的方式：**只保留当前模型真正需要的接口与依赖**，并通过 Safe Stub 等方式避免无关动态运行时依赖进入最终固件。

> 💡 对资源受限 MCU 来说，“把所有库都塞进去”不是工程化。

---

## 六、模型接口：让上层只关心输入和输出

项目最终将模型调用封装为：

```cpp
int Model_Run_Inference(
    float *raw_features,
    int feature_count,
    float *out_confidence
);
```

上层不需要知道 Edge Impulse 内部到底有多少层，只需要：

```text
输入：300 个 float
       ↓
 Model_Run_Inference()
       ↓
输出：类别 ID + confidence
```

这一步很重要，因为它把：

- 传感器采集
- 数据缓存
- DSP
- 分类器

隔离开来。

后续替换模型时，上层逻辑也更容易保持稳定。

---

## 七、为什么“模型部署”本身就是一项工程？

电脑上的模型运行环境通常拥有：

- 大量 RAM
- 完整 C/C++ 运行时
- 文件系统
- 动态内存
- 丰富的库

而 STM32F103 只有有限的 Flash 和 SRAM。

所以部署实际上是在做一次：

> **从“桌面软件思维”到“嵌入式资源思维”的转换。**

你不仅要问：

> 模型准不准？

还要问：

> 模型多大？
> 需要多少 RAM？
> DSP 多久？
> 能不能静态运行？
> 会不会触发 HardFault？

---

## 八、本篇小结

本文完成了模型从训练环境进入 MCU 工程的关键步骤，并解决了部署阶段的典型问题：

| 问题 | 本质 | 处理方向 |
|:---|:---|:---|
| Semihosting 卡死 | 调试运行时依赖 | 关闭 / 改用串口 |
| DSP `-1002` | SRAM / heap 压力 | 调整内存布局 |
| TFLite 依赖过重 | Runtime 组件过多 | 精简依赖 / Safe Stub |
| HardFault 风险 | 运行时资源不足 | 检查 RAM / Stack / Heap |

> 🎯 **下一步**：让 STM32 不只是“拥有模型”，而是真正执行一次完整手势推理。

---

## 📚 系列导航

| # | 主题 | 状态 |
|---:|:---|:---:|
| 72 | 项目启动：系统架构与 MPU6050 数据采集 | ✅ |
| 73 | STM32 手势数据采集：从串口到 CSV 数据集 | ✅ |
| 74 | TinyML 特征提取与模型训练：V1 → V2 | ✅ |
| **75** | **TinyML 模型部署：Edge Impulse → STM32F103** | 🟢 本文 |
| 76 | STM32 端侧实时手势推理 | ⏳ |
| 77 | TinyML 工程优化：从 HardFault 到 32ms 推理 | ⏳ |
| 78 | 第一阶段总结：STM32 TinyML 手势识别系统完整跑通 | ⏳ |
| 79+ | 第二阶段：数据集扩充与泛化优化 | 🔮 |

## 🔗 项目地址

- **STM32 TinyML 项目仓库**：https://github.com/nathanialkonggsd-hub/STM32-Gesture-TinyML

## 👋 结语

感谢阅读！如果这篇文章对你有帮助，欢迎收藏、点赞，也欢迎在评论区交流。

项目仍在持续迭代，后续会继续记录从“能跑”到“可靠、可复现、可泛化”的过程。
