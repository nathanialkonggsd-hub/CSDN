# 基于 STM32 与 TinyML 的端侧手势识别系统

本项目在无硬件 FPU 的 STM32F103 (Cortex-M3 @ 72MHz) 平台上部署了 Edge Impulse 导出的 TinyML 模型。系统通过 MPU6050 采集加速度数据，在端侧完成特征提取与静态神经网络分类，支持 4 类手势动作识别，并在 0.96 寸 I2C OLED 上实时显示当前姿态示波器、主导轴状态与分类结果。

> **摘要**：本文完整记录了在无硬件 FPU 的 STM32F103 (Cortex-M3 @ 72MHz) 平台上，从零部署 Edge Impulse 导出 TinyML 手势识别系统的全过程。系统通过 MPU6050 采集三轴加速度数据，在端侧完成频域特征提取与静态神经网络推理，实现 4 类手势的实时识别，并在 0.96 寸 OLED 上可视化展示。文章重点剖析了开发中遇到的六大关键技术难题——TFLite 解释器依赖爆炸与 HardFault 陷阱、传感器量纲失配导致的置信度塌陷、DSP 堆内存耗尽、CMSIS-DSP 硬件 RFFT 约束、示波器单轴盲区以及 UI 视觉遮挡，并给出了对应的工程化解决方案。经过两轮模型迭代与系统优化，端侧推理耗时由初始的 72ms 压缩至平均 32ms，分类置信度稳定在 0.85~0.95 区间，为嵌入式端侧 AI 应用提供了完整的实战参考。

 ***项目地址***：
 GitHub：[https://github.com/nathanialkonggsd-hub/STM32-Gesture-TinyML](https://github.com/nathanialkonggsd-hub/STM32-Gesture-TinyML)
Gitee：[https://gitee.com/suuus2749/project/tree/master/STM32-Gesture-TinyML](https://gitee.com/suuus2749/project/tree/master/STM32-Gesture-TinyML)
## 目录

- [一、 开发时间历程 (Development Timeline)](#一-开发时间历程-development-timeline)
- [二、 系统架构与工作流](#二-系统架构与工作流)
  - [1. 硬件连接与通信拓扑](#1-硬件连接与通信拓扑)
  - [2. 状态机设计](#2-状态机设计)
- [三、 手势分类模型与迭代对比](#三-手势分类模型与迭代对比)
- [四、 关键排障与技术突破](#四-关键排障与技术突破)
  - [1. 规避 TFLite 解释器依赖爆炸与 HardFault 陷阱](#1-规避-tflite-解释器依赖爆炸与-hardfault-陷阱)
  - [2. 传感器量纲失配引发的模型置信度塌陷](#2-传感器量纲失配引发的模型置信度塌陷)
  - [3. 堆栈空间规划与 DSP 处理报错消除 (-1002)](#3-堆栈空间规划与-dsp-处理报错消除--1002)
  - [4. CMSIS-DSP 硬件 RFFT 约束与动态软解降级](#4-cmsis-dsp-硬件-rfft-约束与动态软解降级)
  - [5. 示波器单轴盲区排查与主导轴自适应追踪](#5-示波器单轴盲区排查与主导轴自适应追踪)
  - [6. 嵌入式 6x8 紧凑字库集成（解决 UI 视觉遮挡）](#6-嵌入式-6x8-紧凑字库集成解决-ui-视觉遮挡)
- [五、 核心运行指标](#五-核心运行指标)
- [六、 成果展示与问题复现对比](#六-成果展示与问题复现对比)
  - [1. 调试演进与问题复现留样](#1-调试演进与问题复现留样)
  - [2. 上位机实时波形监控 (VOFA+)](#2-上位机实时波形监控-vofa)
  - [3. 最终系统完整交互流程](#3-最终系统完整交互流程)

---

## 一、 开发时间历程 (Development Timeline)

| 时间节点 | 阶段目标 | 核心工作与攻关记录 | 阶段成果 / 交付状态 |
| :--- | :--- | :--- | :--- |
| **2026-09-04** | **需求定义与环境预研** | 确定 STM32F103 部署端侧神经网络的技术路线；配置 Keil MDK (Arm Compiler 6) 与 STM32CubeMX 工程脚手架；梳理 MPU6050 采集节拍与 0.96 寸 OLED 刷新时序需求。 | 完成最小系统软硬件环境规划与技术栈选型。 |
| **2026-09-05** | **传感器与显示驱动链路打通** | 完成 I2C1 (MPU6050) 与 I2C2 (SSD1306) 驱动调试；实现脱机正弦波数据仿真机制以保障无传感器调试；配置 USART1 重定向与 VOFA+ 上位机实时波形输出。 | 硬件外设基础链路就绪，实现传感器离线安全兜底与波形联调。 |
| **2026-09-06** | **手势数据集构建与初版模型训练** | 在 Edge Impulse 平台采集 4 类典型姿态（idle、wave_lr、shake_fast、lift_ud）样本集；完成前级 Spectral Features 提取配置；训练并导出 V1 版本的 C++ 部署包。 | 完成云端数据集闭环与初版神经网络权重生成。 |
| **2026-09-07** | **工程初步移植与关键缺陷暴露** | 将 Edge Impulse SDK 导入 Keil 工程；遇到标准库 Semihosting 调试死锁，配置桩函数剥离半主机；遭遇 DSP 堆内存耗尽报错 (`-1002`)，扩展启动文件堆栈；排查出重力加速度量纲失配问题[cite: 1]。 | 解决死锁与 OOM 故障；定位并修正导致动作识别置信度仅 0.23 的量纲缺陷[cite: 1]。 |
| **2026-09-08** | **静态算子链接与模型迭代** | 启用 EON Compiler 静态展开编译；重构 `model_runner.cpp` 的 Safe Stub 存根，杜绝动态解释器依赖膨胀与空指针引发的 HardFault 崩溃；在云端调整特征提取窗口生成 V2 模型。 | 实现纯静态神经网络前向推理闭环，准确率与推理稳定性达标。 |
| **2026-09-09** | **UI 交互重构与推理深度优化** | 排查示波器固定 X 轴导致的单轴盲区，实现三轴动态幅值比对的主导轴追踪；内嵌 6x8 紧凑字库彻底消除屏幕视觉遮挡；对齐 CMSIS-DSP 硬件约束。 | 端侧推理耗时由初始的 72ms 进一步压缩至平均 32ms；完成全流程实机验证与文档归档。 |

---

## 二、 系统架构与工作流

### 1. 硬件连接与通信拓扑
* **主控芯片**：STM32F103ZET6 (Cortex-M3 @ 72MHz，64KB SRAM，512KB Flash)。
* **传感器总线 (I2C1)**：MPU6050 挂载于硬件 I2C1（写地址 `0xD0`），配置为 50Hz 采样率。
* **屏幕总线 (I2C2)**：0.96 寸 SSD1306 OLED 挂载于硬件 I2C2（写地址 `0x78`），采用独立双总线物理隔离，避免屏幕刷屏长周期阻塞传感器读取。
* **人机交互与调试**：
  * 板载按键 `KEY0` (PE4) 采用时间戳防抖边沿触发采样。
  * `USART1` (115200) 输出 4 通道平滑波形（适配 VOFA+）及推理耗时详情。

### 2. 状态机设计
* **待机监控态 (`MODE_MONITOR`)**：
  * 实时采集 MPU6050 三轴动态加速度，进行 10 点滑动均值滤波并发送串口波形。
  * 左侧绘制 X/Y/Z 三轴双向光柱，右侧微型示波器绘制动态波形。
* **特征采集态 (`MODE_RECORDING`)**：
  * 按下 KEY0 触发，以 50Hz 节拍连续采集 100 组三轴加速度点（共 300 维浮点特征）存入采样缓冲区。
  * OLED 显示采样进度条。
* **结果展示态 (`MODE_SHOW_RESULT`)**：
  * 调用本地分类模型完成计算，在 OLED 界面锁存显示识别标签、置信度与推理耗时，保持 2 秒后自动切回待机监控。

---

## 三、 手势分类模型与迭代对比

基于 Edge Impulse 平台构建，支持以下 4 类手势：
* `idle`：静止平放 / 待机
* `wave_lr`：水平左右挥动
* `shake_fast`：快速晃动
* `lift_ud`：垂直上下抬起

通过调整频域分析窗口与特征提取参数，模型经历了两次迭代：
* **V1 模型**：初始参数下特征生成耗时较长（估算 11ms），且验证集存在孤立误判点。
* **V2 模型**：调整特征提取参数后，端侧特征处理估算耗时从 11ms 降低至 2ms，4 类动作在特征空间中的聚类间距更清晰，混淆矩阵验证达到 100% 准确率。

<table style="width: 100%; border-collapse: collapse; text-align: center; border: 1px solid #444; background-color: #1e1e1e; color: #fff; font-family: sans-serif;">
    <thead>
        <tr style="background-color: #2d2d2d;">
            <th style="width: 15%; padding: 15px; border: 1px solid #444; font-size: 16px; font-weight: bold;">模型<br>版本</th>
            <th style="width: 42.5%; padding: 15px; border: 1px solid #444; font-size: 16px; font-weight: bold;">特征空间聚类分布 (Spectral Features)</th>
            <th style="width: 42.5%; padding: 15px; border: 1px solid #444; font-size: 16px; font-weight: bold;">训练混淆矩阵与验证表现</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td style="padding: 20px; border: 1px solid #444; font-weight: bold; font-size: 18px; vertical-align: middle;">V1<br>(初始<br>版本)</td>
            <td style="padding: 10px; border: 1px solid #444; vertical-align: middle;">
                <img src="https://i-blog.csdnimg.cn/direct/a877a9a6b2dd484fb0e2615548166f9f.jpeg" style="width: 100%; max-width: 450px; display: block; margin: 0 auto; border-radius: 4px;" alt="V1特征空间">
            </td>
            <td style="padding: 10px; border: 1px solid #444; vertical-align: middle;">
                <img src="https://i-blog.csdnimg.cn/direct/18f8db04721743e7a5d4b6d7977628e5.png" style="width: 100%; max-width: 450px; display: block; margin: 0 auto; border-radius: 4px;" alt="V1混淆矩阵">
            </td>
        </tr>
        <tr>
            <td style="padding: 20px; border: 1px solid #444; font-weight: bold; font-size: 18px; vertical-align: middle;">V2<br>(优化<br>版本)</td>
            <td style="padding: 10px; border: 1px solid #444; vertical-align: middle;">
                <img src="https://i-blog.csdnimg.cn/direct/f1ffaabefa1b413caa975ab9c2a01711.png" style="width: 100%; max-width: 450px; display: block; margin: 0 auto; border-radius: 4px;" alt="V2特征空间">
            </td>
            <td style="padding: 10px; border: 1px solid #444; vertical-align: middle;">
                <img src="https://i-blog.csdnimg.cn/direct/f200e448fc0c4139bc659cc10a662d53.png" style="width: 100%; max-width: 450px; display: block; margin: 0 auto; border-radius: 4px;" alt="V2混淆矩阵">
            </td>
        </tr>
    </tbody>
</table>

---

## 四、 关键排障与技术突破

### 1. 规避 TFLite 解释器依赖爆炸与 HardFault 陷阱
* **阻碍**：
  模型启用了 EON Compiler（`EI_CLASSIFIER_COMPILED = 1`），网络权重与拓扑已被展开为静态 C++ 代码。但在链接阶段，链接器仍会拉取算子注册（`Register_FULLY_CONNECTED` 等）与临时张量管理符号。
  如果直接将官方的 `micro_allocator.cc` 和 `micro_interpreter.cc` 加入工程，会引发依赖连锁反应，拉扯出 FlatBuffers、内存规划器等几十个未定义符号；而若在存根函数中直接返回 `nullptr`，模型初始化向该指针写入元数据时会访问非法内存地址，直接触发 Cortex-M3 **HardFault** 卡死。
* **解决方法**：
  移除工程中多余的 TFLite 动态分配器源码，在 `model_runner.cpp` 中重构 **Safe Stub**：使用预先分配好的全局静态结构体 `TfLiteTensor` 和静态缓冲数组作为兜底实体。当解释器接口申请临时张量时返回该合法静态地址，既斩断了庞大的动态解释器依赖树，又彻底杜绝了因空指针解引用导致的内核崩溃。

### 2. 传感器量纲失配引发的模型置信度塌陷
* **阻碍**：
  在解决内存报错后，系统能够完成推理流程，但判别置信度反常：仅静止状态下能识别 `idle`，做动作时置信度在 0.23 附近徘徊，输出概率扁平化[cite: 1]。
* **解决方法**：
  MPU6050 底层解析出的数据单位为重力加速度倍数（g），而模型训练绑定的单位为国际单位制（m/s²）[cite: 1]。在填充 `raw_features` 数组时统一乘上重力常数 9.80665f 对齐量纲，动作能量分布恢复正常[cite: 1]。

### 3. 堆栈空间规划与 DSP 处理报错消除 (`-1002`)
* **阻碍**：
  初次调用 `run_classifier()` 时，系统返回错误码 `-1002`（`EI_IMPULSE_DSP_ERROR`）[cite: 1]。排查发现标准启动文件默认堆空间过小（512 字节），导致 DSP 阶段分配 FFT 矩阵与滤波器临时缓存时内存耗尽（OOM）[cite: 1]。
* **解决方法**：
  在 `startup_stm32f103xe.s` 中调整堆栈分配：栈空间配置为 **2KB**（`0x00000800`），堆空间扩展至 **10KB**（`0x00002800`），为 Edge Impulse 的矩阵提取提供足够的安全裕度，同时全局静态变量与堆栈总开销完全控制在芯片 64KB SRAM 范围以内。

### 4. CMSIS-DSP 硬件 RFFT 约束与动态软解降级
* **现象**：
  推理初始化时串口输出：`INFO: HW RFFT failed, FFT size not supported. Must be a power of 2 between 32 and 4096, (size was 16)`。
* **原因与处理**：
  ARM 官方 CMSIS-DSP 汇编优化的实数快速傅里叶变换（`arm_rfft_fast_f32`）硬性要求输入点数必须为 32 到 4096 之间的 2 的整次幂[cite: 1]。当前切片窗口点数为 16，导致硬件算子未能命中。
  Edge Impulse SDK 检测到后自动降级调用内置的纯 C 软件算法（`kiss_fft.cpp`）完成频谱计算，计算逻辑无损且推理闭环正常[cite: 1]。

### 5. 示波器单轴盲区排查与主导轴自适应追踪
* **阻碍**：
  原示波器逻辑写死为仅压入 X 轴数据（`current_ax`）。当测试上下移动（Z 轴主导）或前后移动（Y 轴主导）时，屏幕上仅能捕捉到微弱的共振杂波，波形形态雷同且无法直观判断运动方向。
* **解决方法**：
  在入队前引入主导轴判定逻辑：对 Z 轴去除 1.0g 重力加速度基准，实时比较三轴动态绝对值幅值，选取变化最剧烈的一路作为主导分量推入环形队列；同时在示波器角落动态渲染当前轴标签与带符号数值（如 `Y:+0.6`），使动作的方向与振幅一目了然。

### 6. 嵌入式 6x8 紧凑字库集成（解决 UI 视觉遮挡）
* **阻碍**：
  原有底层显示函数写死了 8x16 点阵字库，字符串 `Y:+0.6` 占用宽度达到 48 像素，直接填满了示波器（宽 63 像素）的大半显示区域，严重遮挡曲线轨迹。
* **解决方法**：
  在 `oled.c` 中内嵌独立的 ASCII 6x8 列优先微型点阵表，封装出 `OLED_ShowString_6x8` 专用接口。将标签占用的字符宽度从 48 像素压缩至 36 像素，高度减半为 8 像素，使标签能够紧凑嵌入示波器边角，不再干扰核心波形。

---

## 五、 核心运行指标

* **输入维度**：300 维原始特征输入（100 采样点 × 3 轴加速度）。
* **计算耗时**：端侧 DSP 频域特征提取 + 神经网络静态推理总耗时由初始版的 72ms 降至平均 **32ms**（Cortex-M3 @ 72MHz 模型迭代优化后）。
* **分类表现**：稳定识别 4 类典型姿态，置信度集中在 **0.85 ~ 0.95** 区间[cite: 1]。
* **资源开销**：整机静态变量与运行时动态内存安全运行在 STM32F103 内部 SRAM 范围内，无死锁与内存越界问题。

---

## 六、 成果展示与问题复现对比

### 1. 调试演进与问题复现留样
记录开发过程中遭遇的缺陷状态与功能验证节点：


<table style="width: 100%; border-collapse: collapse; text-align: center; border: 1px solid #444; background-color: #1e1e1e; color: #fff; font-family: sans-serif;">
    <thead>
        <tr style="background-color: #2d2d2d;">
            <th style="width: 33.33%; padding: 15px; border: 1px solid #444; font-size: 16px; font-weight: bold;">传感器离线正弦仿真模式</th>
            <th style="width: 33.33%; padding: 15px; border: 1px solid #444; font-size: 16px; font-weight: bold;">量纲失配缺陷 (置信度仅 0.23)</th>
            <th style="width: 33.33%; padding: 15px; border: 1px solid #444; font-size: 16px; font-weight: bold;">单轴波形 (无主导轴与方向标签)</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td style="padding: 15px; border: 1px solid #444; vertical-align: middle;">
                <img src="https://i-blog.csdnimg.cn/direct/a31975df9e2d4a729a01ae59d2f3513a.jpeg" style="width: 100%; max-width: 300px; display: block; margin: 0 auto; border-radius: 4px;" alt="传感器离线正弦仿真模式">
            </td>
            <td style="padding: 15px; border: 1px solid #444; vertical-align: middle;">
                <img src="https://i-blog.csdnimg.cn/direct/e90c87efac5d4fa895bb848183ed43e2.jpeg" style="width: 100%; max-width: 300px; display: block; margin: 0 auto; border-radius: 4px;" alt="量纲失配缺陷">
            </td>
            <td style="padding: 15px; border: 1px solid #444; vertical-align: middle;">
                <img src="https://i-blog.csdnimg.cn/direct/79746792fd2f455597893052bc69b50c.jpeg" style="width: 100%; max-width: 300px; display: block; margin: 0 auto; border-radius: 4px;" alt="单轴波形">
            </td>
        </tr>
    </tbody>
</table>

### 2. 上位机实时波形监控 (VOFA+)
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/337275de6adb4ef8b9448a60e233b80c.png#pic_center)


### 3. 最终系统完整交互流程
<table style="width: 100%; border-collapse: collapse; text-align: center; border: 1px solid #444; background-color: #1e1e1e; color: #fff; font-family: sans-serif;">
    <thead>
        <tr style="background-color: #2d2d2d;">
            <th style="width: 33.33%; padding: 15px; border: 1px solid #444; font-size: 16px; font-weight: bold;">1. 待机监控与主导轴示波器 (`MODE_MONITOR`)</th>
            <th style="width: 33.33%; padding: 15px; border: 1px solid #444; font-size: 16px; font-weight: bold;">2. 实时动作采样 (`MODE_RECORDING`)</th>
            <th style="width: 33.33%; padding: 15px; border: 1px solid #444; font-size: 16px; font-weight: bold;">3. 推理结果与 32ms 耗时锁存 (`MODE_SHOW_RESULT`)</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td style="padding: 15px; border: 1px solid #444; vertical-align: middle;">
                <img src="https://i-blog.csdnimg.cn/direct/06ea800f5a7d4dd882a1ddce6ca8726e.jpeg" style="width: 100%; max-width: 300px; display: block; margin: 0 auto; border-radius: 4px;" alt="待机监控与主导轴示波器">
            </td>
            <td style="padding: 15px; border: 1px solid #444; vertical-align: middle;">
                <img src="https://i-blog.csdnimg.cn/direct/76ec1c9053484da6a1aba6c1eb7e3267.jpeg" style="width: 100%; max-width: 300px; display: block; margin: 0 auto; border-radius: 4px;" alt="实时动作采样">
            </td>
            <td style="padding: 15px; border: 1px solid #444; vertical-align: middle;">
                <img src="https://i-blog.csdnimg.cn/direct/293afd0247824fd688e38bd3a1d8df1f.jpeg" style="width: 100%; max-width: 300px; display: block; margin: 0 auto; border-radius: 4px;" alt="推理结果与32ms耗时锁存">
            </td>
        </tr>
    </tbody>
</table>

