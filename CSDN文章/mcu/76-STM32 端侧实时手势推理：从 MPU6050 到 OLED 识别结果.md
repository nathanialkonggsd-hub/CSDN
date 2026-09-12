> 🧭 **文章档案｜STM32 TinyML 手势识别系列**
>
> **📌 编号**：No.76　｜　**🚀 阶段**：第一阶段 · 端侧推理　｜　**🛠️ 类型**：系统实战
>
> **🔩 技术栈**：STM32F103ZET6 · MPU6050 · Edge Impulse · C++ · OLED · USART1
>
> **🎯 本篇关键词**：300 点窗口 · `signal_t` · `run_classifier()` · 状态机 · KEY0 防抖 · OLED 结果
>
> **✨ 本篇特色**：这是模型第一次真正站在 STM32 上完成“采样 → 推理 → 输出”的闭环——从训练电脑里的模型，变成手里这块板子上的实时能力。
>
> **🧪 实战状态**：端到端已跑通　｜　**📈 项目主线**：传感器 → 推理 → 结果

@[TOC](文章目录)

# STM32 端侧实时手势推理：从 MPU6050 到 OLED 识别结果

> **STM32 + TinyML 端侧手势识别系列 · 第 76 篇**
>
> 🎯 **本文重点**：把 MPU6050 的实时数据送入模型，在 STM32F103 上完成一次完整的“采样 → DSP → 分类 → OLED / UART 输出”。

---

## 一、模型终于真正“跑起来”了吗？

上一篇只是把模型集成进 STM32 工程。

这一篇才开始真正执行：

```text
MPU6050
   ↓
100 × 3 采样窗口
   ↓
300 个 float
   ↓
Edge Impulse DSP
   ↓
分类器
   ↓
4 类概率
   ↓
argmax
   ↓
手势 + confidence
   ↓
OLED / UART
```

> 💡 这条链路跑通，才意味着 TinyML 真正进入了 MCU 应用层。

---

## 二、一次推理的数据窗口

当前工程使用：

```c
#define RECORD_SAMPLES 100
static float raw_features[RECORD_SAMPLES * 3];
```

实际内存顺序为：

```text
[x0, y0, z0,
 x1, y1, z1,
 ...,
 x99, y99, z99]
```

所以：

> **100 个采样点 × 3 个轴 = 300 个 float 输入值。**

数据顺序必须与训练阶段保持一致，否则模型即使“能运行”，结果也可能完全错误。

---

## 三、模型接口如何接入？

上层调用统一封装接口：

```c
int Model_Run_Inference(
    float *raw_features,
    int feature_count,
    float *out_confidence
);
```

内部通过 `signal_t` 将原始数据提供给 Edge Impulse：

```cpp
signal_t signal;
signal.total_length = EI_CLASSIFIER_DSP_INPUT_FRAME_SIZE;
signal.get_data = &raw_feature_get_data;

run_classifier(&signal, &result, false);
```

模型输出的是多个类别概率，因此需要寻找最大概率对应的类别：

```text
result.classification[]
        ↓
遍历所有类别
        ↓
找到最大 value
        ↓
得到 class_id
        ↓
保存 max confidence
```

当前类别：

| ID / Label | 含义 |
|:---:|:---|
| `idle` | 静止 |
| `wave_lr` | 左右挥动 |
| `lift_ud` | 上下抬动 |
| `shake_fast` | 快速晃动 |

---

## 四、状态机：让 MCU 知道“现在该做什么”

项目没有把所有逻辑塞进一个 `while(1)`，而是划分为三个核心状态：

```text
MODE_MONITOR
      ↓ KEY0
MODE_RECORDING
      ↓ 100 samples
MODE_SHOW_RESULT
      ↓
回到 MODE_MONITOR
```

### `MODE_MONITOR`

实时显示 / 等待下一次触发。

### `MODE_RECORDING`

连续采集 100 个三轴样本，并更新 OLED 进度。

> 🖼️ **图片 01｜OLED 采样进度**
>
> 请将原图下载到本地后，通过 CSDN 工具栏「图片」上传。
>
> 原图地址：
> https://github.com/nathanialkonggsd-hub/STM32-Gesture-TinyML/blob/main/STM32-Gesture-TinyML/03_Images_and_Results/oled_sampling_progress.jpg

### `MODE_SHOW_RESULT`

调用模型，显示最终类别、置信度和推理耗时。

---

## 五、KEY0 触发 + 50 ms 防抖

按键事件通过边沿检测和时间戳实现：

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

这样一次真实按键动作才会对应一次采集事件。

---

## 六、从推理到 OLED

模型完成后，程序记录：

```c
uint32_t t_start = HAL_GetTick();
last_gesture_id = Model_Run_Inference(
    raw_features,
    RECORD_SAMPLES * 3,
    &last_confidence
);
last_infer_time = HAL_GetTick() - t_start;
```

OLED 最终显示：

```text
RESULT:
Gesture: wave_lr
Conf: 0.92
Time: 32ms
```

实际识别结果也会通过 USART1 输出，方便使用串口 / VOFA+ 进一步观察。

> 🖼️ **图片 02｜STM32 OLED 手势识别结果**
>
> 请将原图下载到本地后，通过 CSDN 工具栏「图片」上传。
>
> 原图地址：
> https://github.com/nathanialkonggsd-hub/STM32-Gesture-TinyML/blob/main/STM32-Gesture-TinyML/03_Images_and_Results/oled_result_display.jpg

---

## 七、为什么还不能说“系统已经可靠”？

因为当前数据集只有：

```text
4 类 × 15 个样本 = 60
```

模型在开发数据上运行正常，只能证明当前工程链路基本正确。

真正的可靠性还需要回答：

- 新动作速度能不能识别？
- 动作幅度变化能不能识别？
- 握持方向变化能不能识别？
- 不同用户能不能识别？
- 未参与训练的数据能不能识别？

因此：

> **能跑 ≠ 好用。**

---

## 八、当前端到端链路

```text
┌──────────┐
│ MPU6050  │
└────┬─────┘
     ↓
┌──────────┐
│ 100×3    │
└────┬─────┘
     ↓
┌───────────┐
│ DSP/Feature│
└────┬──────┘
     ↓
┌──────────┐
│ TinyML   │
└────┬─────┘
     ↓
┌──────────┐
│ 概率输出 │
└────┬─────┘
     ↓
┌──────────────┐
│ OLED / UART  │
└──────────────┘
```

到这里，项目已经从“模型部署实验”进入了真正的端侧应用阶段。

---

## 九、本篇小结

本篇完成：

- 100 × 3 原始采样窗口
- `signal_t` + `run_classifier()`
- 四类概率输出
- argmax 分类
- KEY0 触发与 50 ms 防抖
- OLED 采样进度
- OLED 识别结果
- UART 调试输出
- 推理时间统计

下一篇开始进入真正的工程优化：

> **低置信度、传感器量纲、DSP / RFFT、HardFault、RAM / Heap / Stack，以及推理性能。**

---

## 📚 系列导航

| # | 主题 | 状态 |
|---:|:---|:---:|
| 72 | 项目启动：系统架构与 MPU6050 数据采集 | ✅ |
| 73 | STM32 手势数据采集：从串口到 CSV 数据集 | ✅ |
| 74 | TinyML 特征提取与模型训练：V1 → V2 | ✅ |
| 75 | TinyML 模型部署：Edge Impulse → STM32F103 | ✅ |
| **76** | **STM32 端侧实时手势推理** | 🟢 本文 |
| 77 | TinyML 工程优化：从 HardFault 到 32ms 推理 | ⏳ |
| 78 | 第一阶段总结：STM32 TinyML 手势识别系统完整跑通 | ⏳ |
| 79+ | 第二阶段：数据集扩充与泛化优化 | 🔮 |

## 🔗 项目地址

- [**STM32 TinyML 项目仓库**：https://github.com/nathanialkonggsd-hub/STM32-Gesture-TinyML](https://github.com/nathanialkonggsd-hub/STM32-Gesture-TinyML)

## 👋 结语

感谢阅读！如果这篇文章对你有帮助，欢迎收藏、点赞，也欢迎在评论区交流。

项目仍在持续迭代，后续会继续记录从“能跑”到“可靠、可复现、可泛化”的过程。
