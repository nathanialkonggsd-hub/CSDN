@[TOC](本文目录)

---

> 🎯 **系列第 2 篇** · 前面我们讲了扫雷的核心算法，现在来看这些内容是怎么==渲染到屏幕==上的。  
> 📖 **上一篇**：[项目概览与扫雷核心算法](https://blog.csdn.net/ysu_0314/article/details/162350482?sharetype=blogdetail&sharerId=162350482&sharerefer=PC&sharesource=ysu_0314&spm=1011.2480.3001.8118)

---

# 🎨  2 | Raylib 渲染架构 —— 从 Camera2D 踩坑到 RenderTexture2D

---

## 📌 1. 问题：窗口该多大？

扫雷的棋盘尺寸是**动态**的：

| 难度 | 棋盘像素（40px 格子） | 需要窗口 |
|------|:--------------------:|:--------:|
| Beginner 9×9 | 360 × 360 | 很小就够了 |
| Expert 16×30 | **1200 × 640** | 需要大窗口 |

如果每次切难度都 `SetWindowSize()` —— Windows DPI 缩放会干预，窗口忽大忽小，全屏时布局全乱。

**解决方案迭代了 4 个阶段。前两个是坑，后两个是答案。**

---

## ❌ 2. Phase 1：固定窗口 + SetWindowSize

```c
int winW = game.cols * CELL_SIZE + 2 * PADDING;
int winH = game.rows * CELL_SIZE + UI_PANEL_HEIGHT + PADDING;
SetWindowSize(winW, winH);
```

> 📋 **Phase 1 的问题**：`SetWindowSize()` 每次切难度窗口会跳变。下面是当时记录的典型问题：
> - ❌ Beginner → Expert：窗口从 400×500 跳到 1300×680，肉眼可见闪烁
> - ❌ Windows DPI 缩放抢夺控制权，窗口有时仅 200×200
> - ❌ 全屏/窗口切换时布局完全错位

**问题清单**：
| 问题 | 严重性 |
|------|:------:|
| Windows DPI 缩放使窗口只有 200×200 | ❌ 致命 |
| 切难度窗口跳变 | ⚠️ 体验差 |
| 全屏/窗口切换布局重算 | ⚠️ 不可靠 |
| 用户不能自由拖拽缩放 | ❌ 不友好 |

> 💡 **教训**：永远不要让窗口尺寸跟着游戏内容变。用==固定画布 + 自动缩放==代替。

---

## ❌ 3. Phase 2：Camera2D —— 理想很丰满

改用固定内部画布（960×640），用 Raylib 的 `Camera2D` 把画布坐标自动映射到窗口：

```c
Camera2D camera = {0};
camera.zoom = fminf((float)screenW / CANVAS_WIDTH,
                    (float)screenH / CANVAS_HEIGHT);
BeginMode2D(camera);
// 所有绘制用 960×640 坐标
EndMode2D();
```

鼠标坐标用 `GetScreenToWorld2D()` 转换回画布空间。

### 🕳️ 坑 1：黑边

Camera2D 只渲染画布区域，宽高比不一致时左右/上下全是黑边。背景图填不满。

### 🕳️ 坑 2：Camera2D + Texture 管线冲突

```c
BeginTextureMode(canvasTarget);
BeginMode2D(camera);  // 嵌套使用！渲染管线冲突
```

Raylib 的这两个模式嵌套时缩放行为不可预测。高 DPI 窗口下文字画错位置、点击区域完全偏移。

### 🕳️ 坑 3：缩放锯齿

Camera2D 用 GL_NEAREST，对棋盘像素风格是好事，但文字全是锯齿。

> 🐛 **代价**：**6 个 Fix（#77-#82）实现 Camera2D 方案，又用 6 个 Fix（#108-#112）全部推翻。**

---

## ✅ 4. Phase 3：RenderTexture2D —— 最终方案

### 4.1 核心架构

```c
// 1️⃣ 创建固定画布纹理
canvasTarget = LoadRenderTexture(CANVAS_WIDTH, CANVAS_HEIGHT);

// 2️⃣ 每帧先画到纹理
BeginTextureMode(canvasTarget);
ClearBackground(BLANK);
// ... 所有游戏内容（用 1280×720 坐标）
DrawToast();
EndTextureMode();

// 3️⃣ 等比缩放 + 居中，blit 到窗口
float canvasScale = fminf((float)screenW / CANVAS_WIDTH,
                          (float)screenH / CANVAS_HEIGHT);
float offsetX = (screenW - CANVAS_WIDTH * canvasScale) / 2.0f;
float offsetY = (screenH - CANVAS_HEIGHT * canvasScale) / 2.0f;

// 🖼️ blit 时用 bilinear 过滤，避免马赛克
SetTextureFilter(canvasTarget.texture, TEXTURE_FILTER_BILINEAR);

Rectangle src = {0, 0, CANVAS_WIDTH, -CANVAS_HEIGHT};
Rectangle dst = {offsetX, offsetY,
                 CANVAS_WIDTH * canvasScale,
                 CANVAS_HEIGHT * canvasScale};
DrawTexturePro(canvasTarget.texture, src, dst, (Vector2){0,0}, 0.0f, WHITE);
```

> 🖼️ **RenderTexture2D 渲染管线**：
> ```mermaid
> graph LR
>     subgraph "每帧渲染循环"
>     A["🎮 游戏逻辑<br/>UpdateGame()"] --> B["🖌️ 画到画布<br/>BeginTextureMode()<br/>1280×720"]
>     B --> C["📐 缩放计算<br/>canvasScale = min(sW/1280, sH/720)"]
>     C --> D["🖼️ Blit 到窗口<br/>DrawTexturePro()"]
>     D --> E["🖥️ 屏幕显示"]
>     end
>     F["🎯 鼠标输入<br/>ScreenToCanvas()"] -.-> A
>     style B fill:#4a6d,color:#fff
>     style D fill:#4a6d,color:#fff
> ```

### 4.2 鼠标坐标转换

```c
static Vector2 ScreenToCanvas(Vector2 screenPos) {
    return (Vector2){
        (screenPos.x - canvasOffsetX) / canvasScale,
        (screenPos.y - canvasOffsetY) / canvasScale
    };
}
```

> ⚠️ **对照 Fix #88**：如果不转换，按钮画在画布 (640, 200) 但碰撞检测以为在窗口 (640, 200)。窗口一缩放，**点击完全错位**。

### 4.3 背景图的处理

背景图（泼墨风格 `background.png`）直接画到**全屏窗口**，不在画布内：

```c
BeginDrawing();
ClearBackground(BLACK);
// 画背景到全屏（窗口坐标系）
DrawTexturePro(bgTexture, src, (Rectangle){0,0, screenW, screenH}, ...);
// 画遮罩到全屏
DrawRectangle(0, 0, screenW, screenH, Fade(BLACK, 0.7f));
// 画游戏到画布（画布坐标系）
BeginTextureMode(canvasTarget);
// ...
```

这样窗口宽高比任意变化，背景都填满，画布永远居中。

---

## 🚀 5. Phase 4：全屏高清（ canvasMult ）

### 5.1 问题

1280×720 画布在全屏（2560×1600）被放大 **2 倍以上**。即使 bilinear 过滤，文字边缘仍有"柔化感"。**中文笔画密集，放大后笔画之间模糊粘连（Fix #221）。**

> 📋 **canvasMult 要解决的问题**：全屏时 1280×720 画布被放大到 2~3 倍。
> - 修复前：中文笔画密集处在放大后互相粘连模糊（英文几乎无影响）
> - 修复后：canvasMult 将渲染目标重建为 2560×1440+，Camera2D 保持 1280×720 布局，下采样后中文清晰锐利

### 5.2 解决方案

```c
static void UpdateCanvasScale(void) {
    // ... 计算 canvasScale ...

    if (canvasScale > 1.0f) {
        int mult = (int)ceilf(canvasScale);
        if (mult < 1) mult = 1;
        if (mult > 4) mult = 4;
        targetW = CANVAS_WIDTH * mult;   // 1280 → 2560
        targetH = CANVAS_HEIGHT * mult;  // 720 → 1440
        canvasMult = mult;
    }

    // 重建高分辨率画布
    if (canvasTarget.texture.width != targetW) {
        UnloadRenderTexture(canvasTarget);
        canvasTarget = LoadRenderTexture(targetW, targetH);
        SetTextureFilter(..., TEXTURE_FILTER_BILINEAR);
    }
}
```

绘制时用 `Camera2D.zoom = canvasMult` 映射：

```c
BeginTextureMode(canvasTarget);
Camera2D cam = {0};
cam.zoom = (float)canvasMult;
BeginMode2D(cam);
// ... 所有 1280×720 布局代码零改动 ...
EndMode2D();
EndTextureMode();
```

| 显示器 | canvasScale | mult | 实际 RT 分辨率 |
|-------|:-----------:|:----:|:-------------:|
| 1280×720 | 1.0 | 1 | 1280×720 |
| 1920×1080 | 1.5 | **2** | **2560×1440** |
| 2560×1600 | 2.0 | **2** | **2560×1440** |
| 3840×2160 (4K) | 3.0 | **3** | **3840×2160** |

> 🔑 **关键**：所有现有布局代码**零改动**。Camera2D 自动把 1280×720 的绘制指令映射到高分辨率画布上，下采样回窗口时清晰度远超纯 1280 放大。

---

## 📊 6. 架构演进总结

> 🖼️ **架构演进路线**：
> ```mermaid
> graph LR
>     P1["❌ Phase 1<br/>固定窗口<br/>SetWindowSize"] --> P2["❌ Phase 2<br/>Camera2D<br/>黑边+锯齿"]
>     P2 --> P3["✅ Phase 3<br/>RT + BILINEAR<br/>1280×720"]
>     P3 --> P4["🚀 Phase 4<br/>canvasMult<br/>高清全屏"]
>     style P3 fill:#2d5a,color:#fff
>     style P4 fill:#2d5a,color:#fff
> ```

| 阶段 | 画布 | 自适应 | 文字清晰度 | 代码复杂度 |
|------|:----:|:------:|:----------:|:----------:|
| ❌ Phase 1 固定窗口 | 随棋盘变 | ❌ | 一般 | 简单 |
| ❌ Phase 2 Camera2D | 960×640 | ✅ | ❌ 锯齿多 | 复杂（坑多） |
| ✅ Phase 3 RT+BILINEAR | 1280×720 | ✅ | ✅ 英文清晰 | 适中 |
| 🚀 **Phase 4 canvasMult** | **自适应高分辨率** | ✅ | **✅ 中文也清晰** | 稍复杂 |

> 💡 **最终选型**：`RenderTexture2D` + 固定 1280×720 逻辑画布 + `TEXTURE_FILTER_BILINEAR` + 全屏 `canvasMult` 高分辨率重建 + `Camera2D` zoom 自动映射。

---

## 🔜 下篇预告

> 第 3 篇：[**222 个 Bug 修复教会我的事**](https://blog.csdn.net/ysu_0314/article/details/162384510?sharetype=blogdetail&sharerId=162384510&sharerefer=PC&sharesource=ysu_0314&spm=1011.2480.3001.8118)  
> 栈溢出、浮点 UB、字体缺失、异步保存、初始化顺序错误…… 精选 15 个让你看完直呼"我也踩过"的经典修复故事。**（可能是全系列最有趣的一篇）**

---

## 📚 系列目录

| # | 标题 | 状态 |
|---|------|:----:|
| 1 | [项目概览与扫雷核心算法](https://blog.csdn.net/ysu_0314/article/details/162350482?sharetype=blogdetail&sharerId=162350482&sharerefer=PC&sharesource=ysu_0314&spm=1011.2480.3001.8118)| ✅ 已发布 |
| 2 | **Raylib 渲染架构** ← 本文 | ✅ 已发布 |
| 3 | [222 个 Bug 修复教会我的事](https://blog.csdn.net/ysu_0314/article/details/162384510?sharetype=blogdetail&sharerId=162384510&sharerefer=PC&sharesource=ysu_0314&spm=1011.2480.3001.8118) | ✅ 已发布 |
| 4 | [中英文双语的工程实现](https://blog.csdn.net/ysu_0314/category_13180582.html?fromshare=blogcolumn&sharetype=blogcolumn&sharerId=13180582&sharerefer=PC&sharesource=ysu_0314&sharefrom=from_link) | 📝 待发布 |
| 5 | [持久化、撤销、提示：非核心功能](https://blog.csdn.net/ysu_0314/category_13180582.html?fromshare=blogcolumn&sharetype=blogcolumn&sharerId=13180582&sharerefer=PC&sharesource=ysu_0314&sharefrom=from_link) | 📝 待发布 |
| 6 | [200 个单元测试：C 项目也能 TDD](https://blog.csdn.net/ysu_0314/category_13180582.html?fromshare=blogcolumn&sharetype=blogcolumn&sharerId=13180582&sharerefer=PC&sharesource=ysu_0314&sharefrom=from_link) | 📝 待发布 |
| 7 | [960×640 → 1280×720：全局缩放重构实录](https://blog.csdn.net/ysu_0314/category_13180582.html?fromshare=blogcolumn&sharetype=blogcolumn&sharerId=13180582&sharerefer=PC&sharesource=ysu_0314&sharefrom=from_link) | 📝 待发布 |

---

> 👍 **如果你觉得有帮助，点赞 + 收藏 + 关注，三连支持作者继续写下去 🚀**
