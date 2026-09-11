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

# TinyML 模型部署：从 Edge Impulse 到 STM32F103

> STM32 与 TinyML 端侧手势识别系列 · 03
>
> 上一篇完成了数据集、特征工程与模型训练。真正困难的部分从这里开始：**训练好的模型如何进入一颗资源有限的 STM32F103，并稳定运行起来？**

---

## 一、从“模型训练完成”到“MCU 能运行”

在 PC 上训练模型，只需要关心数据、特征和准确率；到了 STM32，问题马上变成了工程问题：

```text
Edge Impulse 模型
        ↓
生成 C/C++ 推理代码
        ↓
加入 STM32 / Keil 工程
        ↓
处理 DSP 与 TensorFlow Lite Micro 依赖
        ↓
处理 RAM / Flash / Heap
        ↓
解决链接、运行时和 HardFault
        ↓
STM32F103 实时推理
```

因此，“模型部署”并不是简单复制几个 `.h` 文件，而是一次 **AI 模型 → 嵌入式软件系统** 的迁移。

本项目使用 STM32F103 作为端侧推理平台，模型通过 Edge Impulse 生成的 C++ SDK 接入 STM32 工程。

---

## 二、STM32 工程里的模型接口

这次项目没有让 `main.c` 直接处理 Edge Impulse 的底层细节，而是单独建立了：

```text
Core/Inc/model_runner.h
Core/Src/model_runner.cpp
```

这种拆分非常重要：主程序负责“什么时候采集、什么时候识别”，模型模块负责“怎么推理”。

项目中的 `model_runner.h` 对外只暴露两个接口：

```cpp
void Model_Init(void);

int Model_Run_Inference(
    float *raw_features,
    int feature_count,
    float *out_confidence
);
```

其中模型接口明确规定：100 组三轴采样数据按照

```text
x0, y0, z0,
 x1, y1, z1,
 ...
```

排列，共 300 个 `float`，然后送入分类器。fileciteturn46file0

这意味着前面第 72、73 篇建立的数据采集链路，在这里终于和 AI 推理接口接上了。

---

## 三、为什么需要 signal 回调？

Edge Impulse 的分类器并不是简单接收一个 `float[300]` 后直接返回结果，它通过 `signal_t` 获取输入数据。

项目中的实现把 STM32 内部的特征缓存包装成了 Edge Impulse 所需要的 signal：

```cpp
static float* g_features_ptr = NULL;
static size_t g_features_len = 0;

static int raw_feature_get_data(
    size_t offset,
    size_t length,
    float* out_ptr
) {
    for (size_t i = 0; i < length; i++) {
        if ((offset + i) < g_features_len) {
            out_ptr[i] = g_features_ptr[offset + i];
        } else {
            out_ptr[i] = 0.0f;
        }
    }
    return 0;
}
```

然后构造：

```cpp
signal_t signal;
signal.total_length = EI_CLASSIFIER_DSP_INPUT_FRAME_SIZE;
signal.get_data = &raw_feature_get_data;
```

最终调用：

```cpp
ei_impulse_result_t result = { 0 };
EI_IMPULSE_ERROR err = run_classifier(
    &signal,
    &result,
    false
);
```

这些代码全部来自项目当前的 `model_runner.cpp`。fileciteturn45file0

这里有一个值得注意的工程细节：代码对超出实际特征缓存的读取进行了 **0 填充**。这相当于给模型输入接口增加了一层边界保护，避免直接访问缓存之外的内存。fileciteturn45file0

---

## 四、模型输出不是一个“答案”，而是一组分类分数

推理完成后，项目没有直接写死某一个类别，而是遍历所有分类结果：

```cpp
int max_idx = 0;
float max_score = 0.0f;

for (size_t ix = 0;
     ix < EI_CLASSIFIER_LABEL_COUNT;
     ix++) {
    if (result.classification[ix].value > max_score) {
        max_score = result.classification[ix].value;
        max_idx = (int)ix;
    }
}
```

最终：

```cpp
*out_confidence = max_score;
return max_idx;
```

也就是说，模型模块对上层程序提供两个核心信息：

- `max_idx`：预测类别
- `max_score`：最高类别的置信度

项目当前定义的 4 个类别是：

| ID | 标签 |
|---:|---|
| 0 | `idle` |
| 1 | `lift_ud` |
| 2 | `shake_fast` |
| 3 | `wave_lr` |

接口定义可以直接在项目的 `model_runner.h` 中看到。fileciteturn46file0

---

## 五、模型为什么会成为 STM32 上最麻烦的部分？

STM32F103 并不是为运行现代机器学习框架设计的 MCU。

所以在实际部署过程中，问题往往不是“模型数学上能不能跑”，而是：

> **编译器、链接器、DSP、TensorFlow Lite Micro、动态内存以及 MCU 本身的资源限制，能不能同时接受这个模型。**

项目工程中可以看到，除了业务代码，还加入了完整的 Edge Impulse SDK 与 CMSIS 相关目录。工程同时存在 `model_runner.cpp`、`model_runner.h` 以及 CMSIS 目录。fileciteturn44file0

这也是为什么 TinyML 项目到了部署阶段，代码量会突然膨胀。

---

## 六、一个很典型的嵌入式问题：半主机模式

在 PC 上使用 `printf()` 非常自然，但在 Keil/ARM 环境中，如果标准库 I/O 没有正确重定向，就可能触发半主机机制。

本项目的 `main.c` 一开始就主动禁用了 ARM 半主机模式，并把 `printf` 重定向到了 USART1：

```c
int fputc(int ch, FILE *f) {
    (void)f;
    HAL_UART_Transmit(
        &huart1,
        (uint8_t *)&ch,
        1,
        0xFFFF
    );
    return ch;
}
```

同时实现了 `_sys_open`、`_sys_close`、`_sys_write`、`_sys_read`、`_sys_exit` 等桩函数。fileciteturn47file0

这个改动看起来和 AI 没什么关系，但实际上它是 **嵌入式 AI 能否正常调试的重要基础设施**。

因为模型运行过程中需要打印：

```text
分类结果
置信度
推理耗时
错误信息
```

如果 `printf` 本身就把程序卡死，那么后面的模型调试根本无从谈起。

---

## 七、真正棘手的问题：TensorFlow Lite Micro 运行时依赖

项目在模型部署过程中遇到了一个非常典型的问题：Edge Impulse 生成的推理代码依赖 TensorFlow Lite Micro 的运行时组件，而 STM32F103 的资源和当前工程配置并不一定能直接满足这些依赖。

最终项目里的 `model_runner.cpp` 增加了一组 **EON 静态安全存根（Safe Stub）**。

例如：

```cpp
static TfLiteTensor g_safe_dummy_tensor;
static uint8_t g_safe_dummy_buffer[256];
```

并提供安全的 Tensor 返回：

```cpp
static TfLiteTensor* GetSafeTensor(void) {
    memset(
        &g_safe_dummy_tensor,
        0,
        sizeof(TfLiteTensor)
    );

    g_safe_dummy_tensor.data.data =
        g_safe_dummy_buffer;
    g_safe_dummy_tensor.bytes =
        sizeof(g_safe_dummy_buffer);
    g_safe_dummy_tensor.type = kTfLiteInt8;

    return &g_safe_dummy_tensor;
}
```

同时，`MicroContext` 的临时输入、输出和中间 Tensor 分配接口都返回这个静态实体，而不是继续返回 `nullptr`。fileciteturn45file0

### ⚠️ 这里必须强调

这种 Safe Stub **不是通用的 TinyML 部署方案**，也不应该简单理解为“把 TensorFlow Lite Micro 替换掉模型就能正常推理”。

它是本项目针对当前 Edge Impulse SDK、编译环境和运行时问题进行的工程性处理。

真正的生产级方案仍然应该理解并正确配置：

- TensorFlow Lite Micro runtime
- Operator registration
- Tensor arena
- 内存分配
- CMSIS-DSP
- 编译器 ABI / C++ runtime
- 模型所需算子

这一点非常重要，否则很容易把“程序不崩”误认为“模型正确运行”。

---

## 八、模型最终是怎么进入主程序的？

模型模块和主程序之间的关系可以简化成：

```text
MPU6050
   ↓
current_ax / current_ay / current_az
   ↓
raw_features[300]
   ↓
Model_Run_Inference()
   ↓
run_classifier()
   ↓
classification[]
   ↓
max_idx + confidence
   ↓
OLED / USART
```

项目的 `main.c` 中，系统在采满 100 个采样点之后，立即调用：

```c
last_gesture_id = Model_Run_Inference(
    raw_features,
    RECORD_SAMPLES * 3,
    &last_confidence
);
```

随后计算推理耗时：

```c
uint32_t t_start = HAL_GetTick();
last_gesture_id = Model_Run_Inference(
    raw_features,
    RECORD_SAMPLES * 3,
    &last_confidence
);
last_infer_time = HAL_GetTick() - t_start;
```

并把结果打印出来。fileciteturn47file0

因此现在整个链路已经形成闭环：

**采集 → 特征缓存 → 模型推理 → 类别 → 置信度 → 推理耗时。**

---

## 九、为什么这里还需要状态机？

如果把采集、推理、显示全部塞进一个 `while(1)` 里，很快就会变得难以维护。

当前项目已经明确划分了三个模式：

```c
typedef enum {
    MODE_MONITOR = 0,
    MODE_RECORDING,
    MODE_SHOW_RESULT
} SystemMode_t;
```

对应：

| 状态 | 功能 |
|---|---|
| `MODE_MONITOR` | 待机、监控三轴数据、显示波形 |
| `MODE_RECORDING` | 连续采集 100 组数据 |
| `MODE_SHOW_RESULT` | 锁定显示识别结果 |

采集满 100 个点后进入推理，然后进入结果显示，2 秒后自动返回监控状态。fileciteturn47file0

这实际上已经不是一个“跑模型的 Demo”，而开始具备一个嵌入式应用的基本软件架构。

---

## 十、这次部署真正学到的东西

### 1. TinyML 的难点不只在模型

模型训练可能只需要几分钟，但 MCU 部署可能需要大量时间解决：

```text
编译
→ 链接
→ 内存
→ 运行时
→ 崩溃
→ 调试
→ 再编译
```

### 2. `nullptr` 和内存问题在 MCU 上尤其危险

PC 程序中一个运行时问题可能只是异常退出；在 MCU 上则可能直接变成：

```text
HardFault
```

而且没有操作系统帮你兜底。

### 3. 模型接口必须和数据采集严格一致

训练阶段是什么数据格式，部署阶段就必须尽量保持一致：

```text
采样频率
轴顺序
单位
窗口长度
数据排列
特征处理方式
```

任何一个环节不一致，都可能出现“程序正常运行，但识别效果异常”。

### 4. 工程抽象开始产生价值

把模型封装成：

```c
Model_Run_Inference(...)
```

比让 `main.c` 直接操作 Edge Impulse 底层 API 更容易继续扩展。

---

## 十一、当前阶段成果

到这里，本项目已经完成了非常关键的一次跨越：

```text
PC 数据集
    ↓
Edge Impulse
    ↓
特征提取
    ↓
模型训练
    ↓
C++ 模型 SDK
    ↓
Keil / STM32 工程
    ↓
STM32F103
```

也就是说，模型已经不再停留在 PC/云端，而是进入了 MCU 工程。

下一篇将继续解决最关键的问题：

> **模型进入 MCU 以后，如何真正拿着 MPU6050 的实时数据进行分类？**

这就是端侧实时推理。

---

## 十二、系列进度

| # | 主题 | 状态 |
|---:|---|:---:|
| 72 | 项目启动：系统架构与 MPU6050 数据采集 | ✅ 已完成 |
| 73 | STM32 → Python → CSV 数据集 | ✅ 已完成 |
| 74 | TinyML：特征提取与模型训练 | ✅ 已完成 |
| 75 | 模型部署：TinyML → STM32 | 🟢 本文 |
| 76 | STM32 端侧实时推理 | ⏳ 待更新 |
| 77 | 低置信度、Bug 定位与性能优化 | ⏳ 待更新 |
| 78 | 第一阶段项目总结 | ⏳ 待更新 |
| 79+ | 第二阶段：数据集扩充与模型泛化优化 | 🔮 规划中 |

---

## 项目地址

[GitHub：STM32-Gesture-TinyML](https://github.com/nathanialkonggsd-hub/STM32-Gesture-TinyML)

---

## 结语

第 72 篇解决的是“**传感器能不能采到数据**”，第 73 篇解决的是“**数据能不能形成数据集**”，第 74 篇解决的是“**数据能不能训练出模型**”。

而第 75 篇解决的是：

> **模型能不能真正进入 MCU。**

这也是从“会使用机器学习工具”走向“会做嵌入式 AI 工程”的分水岭。

真正的 TinyML，不是把一个模型文件复制进单片机，而是让 **数据、算法、软件、硬件和资源限制**在同一个系统里共同工作。
