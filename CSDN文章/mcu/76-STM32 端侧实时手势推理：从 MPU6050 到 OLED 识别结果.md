---
number: 76
category:
  - MCU
  - TinyML
  - Embedded AI
series: STM32
author: 小苏
level: Advanced
type: Project
status: Completed
---

# STM32 端侧实时手势推理：从 MPU6050 到 OLED 识别结果

上一篇完成了 **Edge Impulse → C++ → Keil → STM32F103** 的模型部署，本篇继续向前一步：

> **让模型真正接管 STM32 上的一次完整手势识别流程。**

这一篇不再停留在“模型已经导入工程”，而是把整个端侧链路串起来：

```text
MPU6050
   ↓
三轴加速度实时采样
   ↓
100 × (X,Y,Z)
   ↓
300 个 float 特征
   ↓
Edge Impulse run_classifier()
   ↓
4 类手势概率
   ↓
选择最高置信度类别
   ↓
串口 + OLED 输出
```

这也是整个 TinyML 项目从“模型部署”进入“系统运行”的关键一步。

---

## 一、先明确：STM32 上到底跑了什么？

本项目当前模型接口定义非常直接：一次传入 **100 组三轴采样数据**，也就是 300 个 `float`：

```text
x0, y0, z0,
x1, y1, z1,
...
x99, y99, z99
```

项目中的 `model_runner.h` 已经明确规定了这个接口：

```c
int Model_Run_Inference(
    float *raw_features,
    int feature_count,
    float *out_confidence
);
```

返回值是最高得分的类别编号，同时通过 `out_confidence` 返回对应置信度。fileciteturn49file0

因此 STM32 端真正需要解决的并不是“怎么调用一个神秘的 AI API”，而是三个非常具体的问题：

1. 如何准备正确的数据格式？
2. 如何把数据交给 Edge Impulse？
3. 如何把模型输出转换成用户能看懂的结果？

---

## 二、100 × 3 的数据窗口

STM32 主程序中定义了：

```c
#define RECORD_SAMPLES 100
static float raw_features[RECORD_SAMPLES * 3];
```

也就是说：

```text
100 samples × 3 axes
= 300 float
```

在采集模式下，每读取一次 MPU6050，就依次写入：

```c
raw_features[record_count * 3 + 0] = current_ax;
raw_features[record_count * 3 + 1] = current_ay;
raw_features[record_count * 3 + 2] = current_az;
```

最终内存中的排列就是：

```text
[x0,y0,z0, x1,y1,z1, x2,y2,z2, ...]
```

这个排列非常重要。

因为模型训练阶段的数据排列方式和 MCU 推理阶段必须保持一致，否则即使模型本身训练得很好，输入数据一旦错位，最终结果也会完全失真。

---

## 三、为什么使用三轴，而不是只采 X 轴？

上一篇已经提到过单轴显示带来的问题，本篇真正进入推理后，这个问题更加明显。

手势识别不是简单的：

```text
X 变化 → 某手势
```

而是：

```text
X(t)
Y(t)
Z(t)
 ↓
一个完整的三维运动轨迹
```

例如当前项目的四个类别：

| 类别 | 含义 |
|---|---|
| `idle` | 静止/待机 |
| `lift_ud` | 上下运动 |
| `shake_fast` | 快速晃动 |
| `wave_lr` | 左右挥动 |

不同动作的主要变化轴可能不同，因此采集阶段保留完整三轴信息，能够避免过早丢失信息。

---

## 四、从数组到 Edge Impulse：`signal_t`

真正连接“自己的数组”和 Edge Impulse 模型的关键，在 `model_runner.cpp` 中。

项目没有直接把 `raw_features` 强行转换成某种内部模型结构，而是实现了 Edge Impulse 所需要的数据读取回调：

```c
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

这里实际上做了一层“适配”：

```text
STM32 raw_features[]
        ↓
raw_feature_get_data()
        ↓
signal_t
        ↓
run_classifier()
```

然后构造模型输入：

```c
signal_t signal;
signal.total_length = EI_CLASSIFIER_DSP_INPUT_FRAME_SIZE;
signal.get_data = &raw_feature_get_data;
```

最后真正启动推理：

```c
ei_impulse_result_t result = { 0 };
EI_IMPULSE_ERROR err = run_classifier(
    &signal,
    &result,
    false
);
```

这几行代码，就是整个 STM32 TinyML 推理链路的核心入口。fileciteturn48file0

---

## 五、`signal_t` 到底解决了什么问题？

可以把 `signal_t` 理解成模型和底层数据之间的一个“数据接口”。

模型并不需要知道：

> “你的数据究竟存在哪个数组里？”

它只需要知道：

> “当我要第 `offset` 个开始的 `length` 个数据时，你能不能给我？”

于是 MCU 端就可以保持自己的内存结构，而 Edge Impulse 通过回调函数读取数据。

这种设计对嵌入式系统非常重要，因为 MCU 的 RAM 很宝贵。

如果每个模块都要求复制一份完整数据，很容易出现：

```text
原始数据
   ↓ copy
DSP 缓冲区
   ↓ copy
模型输入缓冲区
   ↓ copy
临时 Tensor
```

最终 RAM 消耗可能迅速增加。

而当前实现至少在最外层数据输入这里采用了指针 + 回调方式，减少了不必要的数据复制。fileciteturn48file0

---

## 六、模型输出不是一个“答案”，而是一组概率

`run_classifier()` 返回之后，项目代码并没有直接认为结果就是某个手势。

首先遍历所有类别：

```c
for (size_t ix = 0;
     ix < EI_CLASSIFIER_LABEL_COUNT;
     ix++) {
    printf("[%-10s] : %.3f\r\n",
           result.classification[ix].label,
           result.classification[ix].value);
}
```

因此一次推理实际上得到的是类似这样的结果：

```text
[idle      ] : 0.012
[lift_ud   ] : 0.041
[shake_fast] : 0.921
[wave_lr   ] : 0.026
```

这里真正代表模型判断强度的是 `value`，而不是类别编号本身。

所以项目又做了一次遍历，寻找最大概率：

```c
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

最后：

```c
*out_confidence = max_score;
return max_idx;
```

也就是说：

```text
模型输出概率向量
        ↓
寻找最大值
        ↓
得到类别 ID
        ↓
得到 confidence
```

这个过程本质上就是一次简单的 `argmax`。fileciteturn48file0

---

## 七、STM32 主程序如何组织整个识别流程？

当前工程并不是按“按键 → 推理 → 结束”的简单结构运行，而是使用了一个小型状态机：

```c
typedef enum {
    MODE_MONITOR = 0,
    MODE_RECORDING,
    MODE_SHOW_RESULT
} SystemMode_t;
```

三个状态分别对应：

### 1. `MODE_MONITOR`

待机状态。

STM32 持续读取 MPU6050，同时进行波形监控，并等待 KEY0。

### 2. `MODE_RECORDING`

按下 KEY0 后进入采集状态。

开始连续记录 100 组数据：

```text
第 1 组 → X Y Z
第 2 组 → X Y Z
...
第 100 组 → X Y Z
```

### 3. `MODE_SHOW_RESULT`

100 组数据采集完成以后立即进行本地推理，然后把结果锁定显示一段时间，再自动返回待机。

因此整个控制逻辑可以画成：

```text
       ┌─────────────┐
       │   MONITOR   │
       └──────┬──────┘
              │ KEY0
              ↓
       ┌─────────────┐
       │  RECORDING  │
       │ 100×3 sample│
       └──────┬──────┘
              │ 100 samples
              ↓
       ┌─────────────┐
       │   INFER     │
       └──────┬──────┘
              ↓
       ┌─────────────┐
       │ SHOW_RESULT │
       └──────┬──────┘
              │ 2 s
              ↓
       ┌─────────────┐
       │   MONITOR   │
       └─────────────┘
```

这种设计虽然还比较简单，但已经具备了一个完整嵌入式应用的基本形态：**输入、处理、输出、状态切换。** fileciteturn50file0

---

## 八、为什么要把推理耗时单独测出来？

采集完成后，代码使用 `HAL_GetTick()` 记录推理前后的时间：

```c
uint32_t t_start = HAL_GetTick();

last_gesture_id = Model_Run_Inference(
    raw_features,
    RECORD_SAMPLES * 3,
    &last_confidence
);

last_infer_time = HAL_GetTick() - t_start;
```

然后把结果打印出来：

```c
printf(
    ">> [识别成功] 手势: %s | 置信度: %.2f | 推理耗时: %lums\r\n",
    gesture_names[last_gesture_id],
    last_confidence,
    (unsigned long)last_infer_time
);
```

这一步很重要，因为 TinyML 的评价标准不能只有准确率。

真正部署到 MCU 后，需要同时关注：

```text
Accuracy
   +
Latency
   +
RAM
   +
Flash
   +
功耗
```

一个在 PC 上准确率很高，但 MCU 上要跑几百毫秒甚至几秒的模型，并不一定适合实时应用。

项目 README 中记录的优化结果是：端侧推理耗时曾从约 **72ms 降到约 32ms**。这也是为什么前面 V1 → V2 的特征提取优化非常有意义——前级 DSP 和模型推理共同决定最终体验。

---

## 九、OLED：把 AI 从“串口输出”变成真正的嵌入式应用

如果没有 OLED，整个系统也可以运行：

```text
STM32
 ↓
UART
 ↓
PC
 ↓
串口助手 / VOFA+
```

但这种方式更像调试工具。

加入 OLED 后，系统可以直接脱离 PC 展示结果：

```text
STM32
 ├── MPU6050
 ├── TinyML
 └── OLED
```

项目中 OLED 刷新周期设置为约 50ms，也就是约 20 FPS：

```c
if (oled_online &&
    (HAL_GetTick() - last_oled_tick >= 50)) {
    last_oled_tick = HAL_GetTick();
    OLED_Clear();
    ...
}
```

因此 OLED 并不是每次循环都疯狂刷新，而是由独立的时间条件控制。

这也是嵌入式程序中非常常见的思想：

> **不要让低优先级显示刷新拖慢核心数据处理。**

fileciteturn50file0

---

## 十、采集阶段：OLED 显示进度

进入 `MODE_RECORDING` 后，OLED 显示：

```c
OLED_ShowString(
    0, 0,
    ">> SAMPLING <<",
    OLED_COLOR_WHITE
);

OLED_DrawBar(
    10, 24, 108, 12,
    (float)record_count,
    0.0f,
    100.0f
);
```

所以用户能够看到：

```text
>> SAMPLING <<

██████████░░░░░
```

随着 `record_count` 从 0 增加到 100，进度条逐渐完成。

这个设计看似只是 UI，但实际上解决了一个非常实际的问题：

> 用户需要知道系统到底有没有收到按键，以及当前采样是否正在进行。

否则整个采集过程黑屏等待，使用体验会非常差。

---

## 十一、结果阶段：OLED 显示手势与置信度

推理完成后，进入 `MODE_SHOW_RESULT`。

OLED 显示：

```text
RESULT:
shake_fast
Conf: 0.92
Time: 32ms
```

对应代码结构是：

```c
OLED_ShowString(0, 0, "RESULT:", OLED_COLOR_WHITE);

OLED_ShowString(
    0, 16,
    (char*)gesture_names[last_gesture_id],
    OLED_COLOR_WHITE
);

snprintf(
    disp_buf,
    sizeof(disp_buf),
    "Conf: %.2f",
    last_confidence
);

OLED_ShowString(
    0, 32,
    disp_buf,
    OLED_COLOR_WHITE
);
```

同时还显示推理耗时：

```c
snprintf(
    disp_buf,
    sizeof(disp_buf),
    "Time: %lums",
    (unsigned long)last_infer_time
);
```

因此最终系统展示的已经不是一个单纯的“类别”，而是：

```text
识别类别
+
置信度
+
推理耗时
```

这三个指标组合起来，才更接近一个真正的 TinyML Demo。fileciteturn50file0

---

## 十二、为什么还要保留串口输出？

既然 OLED 已经能够显示结果，为什么不直接删掉 `printf()`？

因为串口在开发阶段依然是非常重要的调试接口。

模型会输出完整的类别分布，例如：

```text
--- [Edge Impulse 分布详情] ---
[idle      ] : 0.xxx
[lift_ud   ] : 0.xxx
[shake_fast] : 0.xxx
[wave_lr   ] : 0.xxx
-------------------------------
```

OLED 只显示最大类别，而串口能够保留完整概率向量。

这意味着当模型误判时，可以进一步分析：

```text
真实：wave_lr

idle       0.03
lift_ud    0.12
shake_fast 0.36
wave_lr    0.49
```

这和：

```text
wave_lr
```

完全不是同一种调试信息。

前者可以帮助我们判断模型到底是“非常自信地错了”，还是“两个类别本来就很接近”。

因此当前工程实际上形成了：

```text
OLED → 用户交互
UART → 工程调试
```

两套输出通道各司其职。

---

## 十三、开发过程中一个容易忽略的问题：按键不能直接读就算了

当前项目 KEY0 使用了边沿检测 + 50ms 去抖：

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

这里解决了两个问题：

### 1. 机械抖动

真实按键按下时并不是理想的：

```text
1 ───────────┐
             └──────── 0
```

而可能是：

```text
1 ───┐ ┌─┐ ┌──┐
     └─┘ └─┘  └──── 0
```

如果程序直接检测低电平，很容易一次按键触发多次动作。

### 2. 防止重复触发

项目采用状态变化检测，而不是持续低电平触发，因此一次按下只会启动一次采集。

这再次说明：

> TinyML 项目并不只有 AI。

前后的 GPIO、I2C、定时、状态机、显示和通信同样决定最终系统能不能稳定运行。fileciteturn50file0

---

## 十四、当前系统已经形成完整闭环

到这里，可以把整个第一阶段系统重新画出来：

```text
┌──────────────────────┐
│       MPU6050        │
│      三轴加速度       │
└──────────┬───────────┘
           │ I2C1
           ↓
┌──────────────────────┐
│       STM32F103      │
│                      │
│  100 × (X,Y,Z)       │
│        ↓             │
│   300 float buffer   │
│        ↓             │
│  DSP / Feature       │
│        ↓             │
│  TinyML Inference    │
└───────┬───────┬──────┘
        │       │
       UART    I2C2
        │       │
        ↓       ↓
     VOFA+     OLED
                │
                ↓
        手势 + 置信度
        + 推理耗时
```

这已经不再是一个孤立的“机器学习实验”。

它是一个完整的嵌入式 AI 小系统：

> **传感器负责感知，STM32 负责计算，TinyML 负责判断，OLED 负责反馈。**

---

## 十五、目前四类手势

当前模型定义了四个输出类别：

```text
0 → idle
1 → lift_ud
2 → shake_fast
3 → wave_lr
```

对应的实际含义是：

| ID | Label | 含义 |
|---:|---|---|
| 0 | `idle` | 静止/待机 |
| 1 | `lift_ud` | 上下运动 |
| 2 | `shake_fast` | 快速晃动 |
| 3 | `wave_lr` | 左右挥动 |

需要注意的是，这个类别映射是当前模型接口中的实际定义，后续如果重新训练并调整标签顺序，MCU 端的显示映射也必须同步修改。fileciteturn49file0

---

## 十六、这一阶段真正学到的东西

做到这里，我觉得这个项目最有价值的地方已经不是“成功识别了一个手势”。

而是第一次完整经历了：

```text
传感器
 ↓
驱动
 ↓
数据采集
 ↓
Python 数据集
 ↓
特征工程
 ↓
模型训练
 ↓
模型导出
 ↓
Keil 集成
 ↓
STM32 推理
 ↓
OLED 输出
```

其中每一个箭头都可能出问题。

例如：

- 数据格式不一致 → 模型输入错误
- 采样窗口不一致 → 推理异常
- 特征参数不合适 → 类别混淆
- DSP 太慢 → 实时性下降
- RAM 不够 → 程序崩溃
- TFLite Micro 依赖复杂 → 链接/运行异常
- UI 刷新过于频繁 → 影响主循环
- 按键没有去抖 → 一次操作触发多次

因此 TinyML 真正锻炼的是一种系统工程能力：

> **把一个算法从 PC 环境一路搬到资源受限的 MCU，并让它最终成为一个可交互的系统。**

---

## 十七、当前阶段的边界：能跑 ≠ 已经足够可靠

虽然当前模型已经可以在 STM32 上完成端侧推理，但这个项目仍然有明显的下一步优化空间。

当前数据集只有：

```text
4 类 × 15 个样本 = 60 个样本
```

而且实际使用时还会遇到：

- 动作速度不同；
- 动作幅度不同；
- 手持姿态不同；
- 传感器安装方向变化；
- 不同使用者动作习惯不同；
- 静止状态和轻微抖动之间的边界。

因此下一阶段不能只看训练集/验证集准确率，而应该开始测试：

```text
模型泛化能力
      ↓
不同动作速度
      ↓
不同动作幅度
      ↓
不同采样方式
      ↓
更多真实样本
      ↓
误判案例分析
```

这也是为什么当前项目 README 仍然保持：

```text
🚧 持续开发中
```

当前文章完成的是“端侧实时推理闭环”，而不是宣称整个项目已经永久完成。

---

## 十八、下一步：从“能跑”进入“优化”

目前最值得继续研究的问题已经从：

> “模型能不能在 STM32 上跑？”

变成：

> “为什么有些时候模型会低置信度甚至误判？如何定位？如何优化？”

这正好对应下一篇：

**77｜工程优化：低置信度、Bug 定位与性能优化**

届时会重点分析项目实际开发中遇到的几个问题：

```text
MPU6050 单位问题
       ↓
DSP / RFFT 参数问题
       ↓
低置信度与误判
       ↓
单轴显示盲区
       ↓
RAM / Heap / Stack
       ↓
推理耗时优化
```

这部分其实是整个系列里最接近“工程实践”的内容。

---

## 十九、系列进度

| # | 主题 | 状态 |
|---:|---|:---:|
| 72 | 项目启动：系统架构与 MPU6050 数据采集 | ✅ 已完成 |
| 73 | STM32 手势数据采集：从串口到 CSV 数据集 | ✅ 已完成 |
| 74 | TinyML 特征提取与模型训练：Edge Impulse V1 → V2 | ✅ 已完成 |
| 75 | TinyML 模型部署：从 Edge Impulse 到 STM32F103 | ✅ 已完成 |
| 76 | STM32 端侧实时手势推理：从 MPU6050 到 OLED | 🟢 本文 |
| 77 | 低置信度、Bug 定位与性能优化 | ⏳ 待更新 |
| 78 | 第一阶段项目总结：完整系统跑通 | ⏳ 待更新 |
| 79+ | 第二阶段：数据集扩充与模型泛化优化 | 🔮 规划中 |

---

## 二十、项目地址

完整源码、数据集、模型导出文件以及开发过程中的结果截图均保存在 GitHub 项目：

**STM32-Gesture-TinyML**

```text
STM32-Gesture-TinyML/
├── 01_Data_Acquisition/
├── 02_Source_Code/
│   └── Elite_F103_Demo/
└── 03_Images_and_Results/
```

其中本篇主要对应：

```text
02_Source_Code/Elite_F103_Demo/
├── Core/Src/main.c
├── Core/Src/model_runner.cpp
└── Core/Inc/model_runner.h
```

这些文件共同组成当前 STM32 端的实时识别闭环。

---

# 总结

第 75 篇解决的是：

> **“模型怎么放进 STM32？”**

而第 76 篇解决的是：

> **“放进去之后，怎么让传感器数据真正经过模型，最终变成一个可交互的识别结果？”**

现在整个链路已经真正闭环：

```text
MPU6050
  ↓
100×三轴采样
  ↓
300 float
  ↓
DSP / 特征提取
  ↓
TinyML 模型
  ↓
4 类概率
  ↓
argmax
  ↓
手势 + confidence
  ↓
OLED / UART
```

对于一个 STM32 TinyML 项目来说，这一步非常关键。

因为从这里开始，项目已经从“**训练了一个模型**”正式进入“**做出了一个端侧 AI 系统**”。

而下一步，就该开始认真处理它的可靠性、性能和泛化能力了。
