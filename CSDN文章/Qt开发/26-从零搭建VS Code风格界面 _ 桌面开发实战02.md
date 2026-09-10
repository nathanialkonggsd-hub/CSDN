@[TOC](开启跨平台 UI 之旅)

# 前言

上一篇我们搞定了 Qt 的极简安装和 CMake 环境配置，电脑上已经有了能跑 Qt 项目的"厨房"了。光有厨房还不够，得开始真正做饭了。

我们最终的目标是打造一个 VS Code 级别的代码编辑器。饭得一口一口吃，这篇先搞定地基——**把窗口的"骨架"搭起来**。

想象一下，你打开 VS Code，看到的是深邃的暗色主题、干净利落的无边框窗口、标题栏上最小化/最大化/关闭按钮悬停时会变色、底部状态栏显示着行列号和编码信息。这些看似理所当然的界面元素，底层到底是怎么实现的？

今天就来揭秘。跟着写完这篇，你将收获：

- 一个 **无边框暗色窗口**，告别系统默认的白色标题栏
- **自定义标题栏**，包含软件名称、菜单栏、窗口控制按钮
- 窗口四周 **8 像素热区拖拽缩放**
- **状态栏信息面板**（字数统计、行列号、缩放百分比、编码信息）
- 初步了解 Qt 的 **eventFilter 事件过滤器** 机制

废话不多说，开干。

---

## 1 项目创建与 CMake 配置

打开 Qt Creator，点击 **文件 -> 新建项目**，选择 **Qt Widgets Application**。项目名称随意取，我这里叫 `MyNotepad_pro`。

<!-- 截图：Qt Creator 新建项目向导截图 -->

> 建议勾选 **将解决方案和项目放在同一目录中**，避免嵌套太深导致路径问题。

创建完成后，打开 `CMakeLists.txt`，默认内容比较干净。因为后续我们会用到多线程（异步语法高亮）和 SVG 图标渲染，需要额外引入两个模块：

```cmake
find_package(Qt6 6.5 REQUIRED COMPONENTS Core Widgets Concurrent Svg)
```

然后在 `target_link_libraries` 中把它们链上：

```cmake
target_link_libraries(MyNotepad_pro
    PRIVATE
        Qt::Core
        Qt::Widgets
        Qt::Concurrent
        Qt::Svg
)
```

> 💡 **为什么现在就加 Svg？** 因为后续实现侧边栏图标时，我们会用到 `QSvgRenderer` 来渲染内联 SVG 字符串[^1]。提前加好，免得后面报链接错误。

点一下左下角的绿色运行按钮，弹出一个空白窗口就算成功了。

---

## 2 无边框窗口：告别系统原生标题栏

默认的 Qt 窗口长这样：白色的标题栏、系统自带的最小化/最大化/关闭按钮。说实话，挺丑的，而且完全没法自定义样式。

VS Code、Discord、Slack 这些现代应用清一色用的是 **无边框窗口**——去掉系统标题栏，自己画一个。这样做的好处是：

1. **外观完全可控**：颜色、字体、按钮图标想怎么设计就怎么设计
2. **沉浸式体验**：标题栏和内容区域可以融为一体
3. **跨平台一致**：Windows/Mac/Linux 上长得一模一样

实现只需要一行代码：

```cpp
// 1. 斩断系统原生边框
this->setWindowFlags(Qt::FramelessWindowHint | Qt::WindowMinimizeButtonHint);
```

<!-- 截图：有系统边框 vs 无边框窗口对比 -->

`Qt::FramelessWindowHint` 是核心标志，它的作用是告诉操作系统："别给我画系统标题栏了，我自己来。" 但光加这个的话，窗口连最小化按钮都没了，所以加上 `Qt::WindowMinimizeButtonHint` 保留最小化功能[^2]。

运行一下，你会发现窗口变成了一块纯黑色的矩形——没有标题栏，没有按钮，甚至拖都拖不动。别慌，这些都是我们接下来要自己实现的。

> ⚠️ **注意**：去掉系统标题栏后，窗口的拖拽移动、缩放、关闭等操作全部失效。这些功能需要我们手动实现，这也是本文后面几节的核心内容。

---

## 3 窗口拖拽移动：让无边框窗口听话

无边框窗口最大的问题：**拖不动了**。系统标题栏帮你处理的拖拽逻辑，现在得自己写。

### 原理

鼠标拖拽窗口的本质是：

1. **鼠标按下时**：记录鼠标位置与窗口左上角的 **坐标差值**
2. **鼠标移动时**：用当前鼠标全局坐标 **减去** 这个差值，得到窗口应该在的位置
3. **调用 `move()`**：把窗口移到新位置

这个差值在整个拖拽过程中保持不变，所以窗口才会"粘"在鼠标下面跟着走。

### 代码实现

首先在 `mainwindow.h` 中添加成员变量和事件声明：

```cpp
// mainwindow.h
class MainWindow : public QMainWindow {
protected:
    void mousePressEvent(QMouseEvent *event) override;
    void mouseMoveEvent(QMouseEvent *event) override;

private:
    QPoint dragPosition; // 记录鼠标与窗口左上角的坐标差值
};
```

然后在 `mainwindow.cpp` 中实现：

```cpp
// 鼠标按下时：记录坐标差值
void MainWindow::mousePressEvent(QMouseEvent *event)
{
    if (event->button() != Qt::LeftButton) return;
    if (isMaximized()) return; // 最大化时不允许拖拽

    // 只在标题栏区域（y < 35）允许拖拽
    if (event->pos().y() < 35) {
        dragPosition = event->globalPosition().toPoint() - frameGeometry().topLeft();
        event->accept();
    }
}

// 鼠标移动时：实时移动窗口
void MainWindow::mouseMoveEvent(QMouseEvent *event)
{
    if ((event->buttons() & Qt::LeftButton) && event->pos().y() < 35) {
        move(event->globalPosition().toPoint() - dragPosition);
        event->accept();
    }
}
```

<!-- 截图：窗口拖拽移动效果 -->

几个关键点：

- **`globalPosition()`** 返回的是鼠标在屏幕上的全局坐标，`frameGeometry().topLeft()` 是窗口在屏幕上的位置，两者相减就是坐标差值
- **`event->pos().y() < 35`** 限制了只有在标题栏高度（35 像素）以内才能拖拽，编辑区域的拖拽应该交给 QTextEdit 处理
- **`event->accept()`** 告诉 Qt 这个事件已经被处理了，不要再往下传递

运行一下，现在窗口可以拖动了。但还不能缩放，接着搞。

---

## 4 窗口边缘缩放：8像素热区的魔法

VS Code 窗口四周拖拽可以缩放，鼠标靠近边缘时光标还会变成双向箭头。这个效果怎么实现的？

### 热区检测原理

思路很简单：把窗口想象成一个矩形，在四条边和四个角各画一个 **8 像素宽的热区**。鼠标进入热区时切换光标，按下左键时进入缩放模式。

首先定义缩放方向的枚举：

```cpp
// mainwindow.h
enum ResizeEdge {
    NoEdge = 0,
    Left = 1, Right = 2, Top = 4, Bottom = 8,                  // 四条边
    TopLeft = 5, TopRight = 6, BottomLeft = 9, BottomRight = 10 // 四个角
};
int resizeEdge = NoEdge;
int lastHoverEdge = NoEdge; // 缓存上次 hover 边缘，避免频繁 setCursor 导致卡顿
QPoint resizeStartPos;      // 缩放开始时的鼠标位置
QRect resizeStartGeo;       // 缩放开始时的窗口几何
static const int RESIZE_MARGIN = 8; // 热区宽度（8像素）[^3]
```

> 💡 **为什么是 8 像素？** 这是 Windows 系统的标准热区大小，太小了鼠标难以精准定位，太大了容易误触。

然后是碰撞检测函数——判断鼠标点在哪个热区：

```cpp
int MainWindow::hitTestResizeEdge(const QPoint &pos)
{
    if (isMaximized()) return NoEdge; // 最大化时禁止缩放

    int x = pos.x(), y = pos.y();
    int w = width(), h = height();
    int m = RESIZE_MARGIN;

    bool left   = x < m;
    bool right  = x > w - m;
    bool top    = y < m;
    bool bottom = y > h - m;

    if (top && left)     return TopLeft;
    if (top && right)    return TopRight;
    if (bottom && left)  return BottomLeft;
    if (bottom && right) return BottomRight;
    if (top)    return Top;
    if (bottom) return Bottom;
    if (left)   return Left;
    if (right)  return Right;

    return NoEdge;
}
```

<!-- 截图：边缘缩放光标变化效果（8种方向） -->

### 光标切换

鼠标移动时，根据热区检测结果切换光标样式。这段逻辑放在 `event()` 中处理（不用 `eventFilter`，原因下面会解释）：

```cpp
bool MainWindow::event(QEvent *event)
{
    if (event->type() == QEvent::HoverMove && resizeEdge == NoEdge) {
        QHoverEvent *he = static_cast<QHoverEvent*>(event);
        QPoint pos = he->position().toPoint();
        int edge = hitTestResizeEdge(pos);

        if (edge != lastHoverEdge) {
            lastHoverEdge = edge;
            switch (edge) {
                case Left: case Right:     setCursor(Qt::SizeHorCursor);   break;
                case Top: case Bottom:     setCursor(Qt::SizeVerCursor);   break;
                case TopLeft: case BottomRight:   setCursor(Qt::SizeFDiagCursor); break;
                case TopRight: case BottomLeft:   setCursor(Qt::SizeBDiagCursor); break;
                default: unsetCursor(); break;
            }
        }
        return false; // 不吞噬事件，继续走 mousePressEvent/mouseMoveEvent
    }

    // 鼠标离开窗口时强制重置光标，防止卡在缩放样式
    if (event->type() == QEvent::Leave) {
        if (lastHoverEdge != NoEdge) {
            lastHoverEdge = NoEdge;
            unsetCursor();
        }
        return false;
    }

    return QMainWindow::event(event);
}
```

### 缩放实现

鼠标按下时记录起始状态，移动时实时计算新窗口矩形：

```cpp
// mousePressEvent 中记录缩放起始状态
if (event->button() == Qt::LeftButton && !isMaximized()) {
    int edge = hitTestResizeEdge(event->pos());
    if (edge != NoEdge) {
        resizeEdge = edge;
        resizeStartPos = event->globalPosition().toPoint();
        resizeStartGeo = geometry();
    }
}

// mouseMoveEvent 中的缩放逻辑
if (resizeEdge != NoEdge && (event->buttons() & Qt::LeftButton)) {
    QPoint delta = event->globalPosition().toPoint() - resizeStartPos;
    QRect newGeo = resizeStartGeo;

    if (resizeEdge & Left)   newGeo.setLeft(newGeo.left() + delta.x());
    if (resizeEdge & Right)  newGeo.setRight(newGeo.right() + delta.x());
    if (resizeEdge & Top)    newGeo.setTop(newGeo.top() + delta.y());
    if (resizeEdge & Bottom) newGeo.setBottom(newGeo.bottom() + delta.y());

    // 最小尺寸防护，防止拖成 0 或负数
    if (newGeo.width() < 400) {
        if (resizeEdge & Left) newGeo.setLeft(newGeo.right() - 400);
        else newGeo.setRight(newGeo.left() + 400);
    }
    if (newGeo.height() < 300) {
        if (resizeEdge & Top) newGeo.setTop(newGeo.bottom() - 300);
        else newGeo.setBottom(newGeo.top() + 300);
    }

    setGeometry(newGeo);
}
```

这里有一个巧妙的设计：`ResizeEdge` 的枚举值是用 **位掩码** 设计的——`Left=1, Right=2, Top=4, Bottom=8`，所以 `TopLeft = Top | Left = 5`。检测时用 `resizeEdge & Left` 就能判断是否包含"左边"方向，无论它是单独的左边还是左上角/左下角。这让四个角的缩放逻辑变得极其简洁。

> 💡 **为什么要缓存 `lastHoverEdge`？** 因为 `HoverMove` 事件频率极高（鼠标每移动几个像素就触发一次），如果不做缓存，每帧都要调用 `setCursor()`，会导致光标频繁闪烁和性能卡顿。

---

## 5 自定义标题栏：VS Code 风格顶栏

现在窗口可以拖拽和缩放了，但还没有标题栏——没有软件名字，没有菜单，没有关闭按钮。接下来我们要自己造一个。

标题栏的布局结构如下：

```
+--------+------+-----------------------------------+--------+--------+--------+
| 软件名 | 菜单栏(文件 编辑 格式...)               |  弹簧  | 最小化 |最大化  | 关闭  |
+--------+------+-----------------------------------+--------+--------+--------+
|<-- 15px -->|                                         |<-- addStretch -->|<-- btn -->|
```

> 这里用了 `QHBoxLayout` 水平布局，中间一个 `addStretch()` 强力弹簧，把左侧的标题和菜单栏推到左边，右侧的按钮推到最右边，自动填满整个宽度。

### 5.1 暗色主题 CSS 样式表

先给标题栏穿上 VS Code 的暗色外衣：

```cpp
QWidget *titleBar = new QWidget(this);
titleBar->setFixedHeight(35); // 顶栏高度
titleBar->setSizePolicy(QSizePolicy::Expanding, QSizePolicy::Fixed);

// 高级暗黑系 CSS 样式表
titleBar->setStyleSheet(
    "QWidget { background-color: #1e1e1e; }"     // 底色与编辑器融为一体
    "QLabel { color: #cccccc; font-weight: bold; font-family: 'Segoe UI', 'Microsoft YaHei'; }"
    "QPushButton { border: none; background: transparent; color: #cccccc;"
    "              font-size: 14px; width: 45px; height: 35px; }"
    "QPushButton:hover { background-color: #333333; }"              // 普通按钮悬停变灰
    "QPushButton#closeBtn:hover { background-color: #e81123; color: white; }" // 关闭按钮悬停变红
);
```

<!-- 截图：CSS 暗色主题效果 -->

逐行解释：

| 选择器 | 作用 |
|--------|------|
| `QWidget { background-color: #1e1e1e; }` | VS Code 标志性的深色底色 |
| `QPushButton:hover { background-color: #333333; }` | 悬停时背景变深灰，提供视觉反馈 |
| `QPushButton#closeBtn:hover` | `#closeBtn` 是 **objectName 选择器**，只有关闭按钮会悬停变红 |

> 💡 **`#closeBtn` 是什么？** 这是 Qt CSS 中的 objectName 选择器。通过 `closeBtn->setObjectName("closeBtn")` 给按钮起个名字，然后在样式表中用 `#名字` 精准选中它。这样就能让关闭按钮有独立的悬停样式，而不影响其他按钮。

菜单栏也需要透明化，融入标题栏：

```cpp
ui->menubar->setStyleSheet(
    "QMenuBar { background-color: transparent; color: #cccccc; }"
    "QMenuBar::item { background: transparent; padding: 5px 10px; margin-top: 2px; }"
    "QMenuBar::item:selected { background: #333333; border-radius: 4px; }"
);
```

### 5.2 QPainter 手绘图标（零外部资源）

这是整个项目最酷的设计之一——**所有图标全部用代码手绘，不依赖任何图片文件**[^4]。

为什么这么做？想想看，如果你的项目引用了一堆 `.png` / `.svg` 图标文件，换个主题颜色就得切几十张图。而纯代码绘制的图标，改一行颜色代码就完事了，还能保证矢量无损缩放。

标题栏三个按钮（最小化、最大化、关闭）的图标长这样：

```cpp
// 矢量图标绘制：标题栏三个控制按钮（纯 QPainter，无外部图片）
auto makeTitleBtnIcon = [](int type) -> QIcon {
    QPixmap pix(16, 16);          // 创建 16x16 的画布
    pix.fill(Qt::transparent);    // 透明背景
    QPainter p(&pix);             // 拿起画笔
    p.setRenderHint(QPainter::Antialiasing); // 开启抗锯齿，线条更丝滑
    p.setPen(QPen(QColor(0xcc, 0xcc, 0xcc), 1.5)); // 浅灰色画笔，1.5像素粗

    if (type == 0) { // 最小化：一条短横线
        p.drawLine(3, 8, 13, 8);
    } else if (type == 1) { // 最大化：一个小方框
        p.drawRect(3, 3, 10, 10);
    } else if (type == 2) { // 关闭：一个叉
        p.drawLine(4, 4, 12, 12);  // 左上到右下
        p.drawLine(12, 4, 4, 12);  // 右上到左下
    }
    return QIcon(pix); // 把画好的图封装成 QIcon 返回
};
```

<!-- 截图：QPainter 绘制的三个按钮图标 -->

整个流程就三步：**创建透明画布 -> 拿画笔画图形 -> 封装成 QIcon**。没有外部依赖，没有文件路径问题，图标颜色随样式表动态变化。

创建三个按钮并绑定图标：

```cpp
QPushButton *minBtn = new QPushButton(makeTitleBtnIcon(0), "", titleBar);
QPushButton *maxBtn = new QPushButton(makeTitleBtnIcon(1), "", titleBar);
QPushButton *closeBtn = new QPushButton(makeTitleBtnIcon(2), "", titleBar);
closeBtn->setObjectName("closeBtn"); // 给关闭按钮起名，用于 CSS 精准定位

minBtn->setToolTip("最小化");
maxBtn->setToolTip("最大化");
closeBtn->setToolTip("关闭");
```

### 5.3 信号槽绑定窗口控制

按钮有了，图标有了，接下来把它们和窗口操作连起来：

```cpp
// 用水平布局组装标题栏
QHBoxLayout *titleLayout = new QHBoxLayout(titleBar);
titleLayout->setContentsMargins(15, 0, 0, 0); // 左侧留 15 像素边距
titleLayout->setSpacing(0);

QLabel *titleLabel = new QLabel("CodeEditor V1", titleBar); // 软件名

titleLayout->addWidget(titleLabel);   // 1. 最左边放软件名字
titleLayout->addSpacing(20);          // 2. 留出一点空隙

ui->menubar->setParent(titleBar);     // 转移菜单栏的父对象到 titleBar
titleLayout->addWidget(ui->menubar);  // 3. 把菜单栏塞在标题旁边

titleLayout->addStretch();            // 4. 强力弹簧：把后面的按钮全部推到最右边

titleLayout->addWidget(minBtn);       // 5. 依次放入右侧按钮
titleLayout->addWidget(maxBtn);
titleLayout->addWidget(closeBtn);

// 用组装好的顶栏替换整个窗口的顶部区域
this->setMenuWidget(titleBar);
```

然后用 `connect()` 把按钮信号绑定到窗口控制槽函数：

```cpp
connect(minBtn, &QPushButton::clicked, this, &QWidget::showMinimized);
connect(maxBtn, &QPushButton::clicked, this, [=]() {
    this->isMaximized() ? this->showNormal() : this->showMaximized();
});
connect(closeBtn, &QPushButton::clicked, this, &QWidget::close);
```

<!-- 截图：按钮 hover 变色效果 -->

`connect()` 是 Qt 最核心的函数之一，四个参数分别是：

| 参数 | 含义 |
|------|------|
| `minBtn` | 信号发送者（谁发出了信号） |
| `&QPushButton::clicked` | 信号（发生了什么事件） |
| `this` | 信号接收者（谁来处理） |
| `&QWidget::showMinimized` | 槽函数（怎么处理） |

> 最大化按钮用了 lambda 表达式 `[=]()` 作为槽函数，因为需要一个简单的三元判断逻辑——如果当前已最大化就还原，否则最大化。

---

## 6 菜单栏融入标题栏：无缝一体

上一节的代码里其实已经用到了这个技巧，这里单独拎出来讲一下。

默认情况下，Qt 的 `QMenuBar` 是独立占据一行的。但我们要做的是 VS Code 那种效果——菜单栏直接跟在标题栏的软件名后面，融为一体。

核心技巧就两步：

**第一步：转移父对象**

```cpp
ui->menubar->setParent(titleBar);
```

`ui->menubar` 默认是 `QMainWindow` 的子对象。通过 `setParent()` 把它的父对象改成我们自定义的 `titleBar`，这样菜单栏就跟着标题栏走了，不再占用 QMainWindow 的独立空间。

**第二步：塞进水平布局**

```cpp
titleLayout->addWidget(ui->menubar);
```

把菜单栏当成一个普通 widget 塞进标题栏的水平布局里。配合前面的 `addSpacing(20)` 和后面的 `addStretch()`，最终效果是：

```
[CodeEditor V1]  [文件  编辑  格式  视图  调试  窗口  帮助]  <----弹簧---->  [_] [□] [X]
```

<!-- 截图：菜单栏融入标题栏的最终效果 -->

菜单栏下拉菜单的效果：

<!-- 截图：菜单下拉效果（hover 高亮） -->

菜单栏的透明样式在 5.1 节已经讲过了，关键的几行：

```cpp
"QMenuBar { background-color: transparent; color: #cccccc; }"         // 透明底色
"QMenuBar::item { background: transparent; padding: 5px 10px; }"      // 菜单项间距
"QMenuBar::item:selected { background: #333333; border-radius: 4px; }" // 选中时圆角高亮
```

> 💡 **为什么要用 `setMenuWidget()`？** `setMenuWidget()` 会用你传入的 widget 替换 QMainWindow 默认的菜单栏区域。这样标题栏就完全占据了窗口顶部，而不是被系统菜单栏挤到下面去。

---

## 7 状态栏：极客专属信息面板

打开 VS Code，看最底部——左边显示行列号和编码，右边显示缩放百分比和行结束符。这就是状态栏。

### 7.1 左侧信息区：字数统计 + 行列号

```cpp
// 左侧：字数统计（用 addWidget，靠左排列）
QLabel *wordCountLabel = new QLabel("字数: 0", this);
wordCountLabel->setObjectName("wordCountLabel");
ui->statusbar->addWidget(wordCountLabel);

// 左侧：行列号（点击可跳转到指定行）
QLabel *cursorLabel = new QLabel("第 1 行, 第 1 列", this);
cursorLabel->setObjectName("cursorLabel");
cursorLabel->installEventFilter(this); // 安装事件过滤器，后续实现点击跳转
ui->statusbar->addWidget(cursorLabel);
```

### 7.2 右侧信息区：缩放 + 编码

```cpp
// 右侧：缩放百分比（点击可重置为 100%）
QLabel *zoomLabel = new QLabel("100%", this);
zoomLabel->setObjectName("zoomLabel");
zoomLabel->installEventFilter(this); // 安装事件过滤器

QLabel *lineEndLabel = new QLabel("Windows (CRLF)", this);
QLabel *encodingLabel = new QLabel("UTF-8", this);

// 调整间距，让它们看起来像现代编辑器
zoomLabel->setContentsMargins(10, 0, 10, 0);
lineEndLabel->setContentsMargins(10, 0, 10, 0);
encodingLabel->setContentsMargins(10, 0, 10, 0);

// 核心黑科技：addPermanentWidget 会把这些标签钉在窗口最右下角！
ui->statusbar->addPermanentWidget(zoomLabel);
ui->statusbar->addPermanentWidget(lineEndLabel);
ui->statusbar->addPermanentWidget(encodingLabel);
```

<!-- 截图：完整状态栏效果（左侧+右侧） -->

### 知识点：addWidget vs addPermanentWidget

| 方法 | 位置 | 特点 |
|------|------|------|
| `addWidget()` | 左侧 | 按添加顺序从左到右排列，内容会被挤压 |
| `addPermanentWidget()` | 右侧 | 固定在状态栏最右边，不会被左侧内容挤压 |

> 💡 **为什么要给 label 起 `objectName`？** 因为后面在 `eventFilter` 中，我们需要通过 `objectName` 来精准识别是哪个 label 被点击了。比如点击 `cursorLabel` 弹出"跳转到行"对话框，点击 `zoomLabel` 重置缩放到 100%。不给名字的话，就只能挨个判断指针地址，非常脆弱。

---

## 8 eventFilter 事件过滤器入门

前面的状态栏代码里出现了两行看起来很神秘的东西：

```cpp
cursorLabel->installEventFilter(this);
zoomLabel->installEventFilter(this);
```

这就是 **事件过滤器**——Qt 中一个非常强大的事件处理机制。后续实现终端键盘交互、标签页中键关闭、代码补全等功能时会大量使用。

### 为什么需要事件过滤器？

Qt 中处理事件有两种方式：

| 方式 | 适用场景 |
|------|----------|
| 重写 `event()` | 只需要处理 **自己类** 的事件（如 `MainWindow` 处理窗口事件） |
| `eventFilter()` | 需要拦截 **其他对象** 的事件（如 `MainWindow` 拦截 `QLabel` 的点击事件） |

在我们的项目中，状态栏的 label 属于 `QLabel` 类，它们自己不会处理点击事件。但我们想让 `MainWindow` 知道"用户点了 cursorLabel"，就需要用事件过滤器来 **截获** label 的鼠标点击事件。

### 安装与实现

**第一步：在目标对象上安装事件过滤器**

```cpp
cursorLabel->installEventFilter(this); // 告诉 cursorLabel：你收到的事件先给我看一眼
```

这里的 `this` 是 `MainWindow`，意思是：当 `cursorLabel` 收到任何事件时，先通知我（MainWindow），让我决定要不要处理。

**第二步：在 MainWindow 中重写 eventFilter()**

```cpp
bool MainWindow::eventFilter(QObject *obj, QEvent *event)
{
    // 拦截鼠标点击事件
    if (event->type() == QEvent::MouseButtonPress) {
        QMouseEvent *me = static_cast<QMouseEvent*>(event);

        // 判断是不是 cursorLabel 被点了
        QLabel *label = qobject_cast<QLabel*>(obj);
        if (label && label->objectName() == "cursorLabel" && me->button() == Qt::LeftButton) {
            // 弹出"跳转到行"对话框（后续文章会实现）
            QTextEdit *edit = qobject_cast<QTextEdit*>(ui->tabWidget->currentWidget());
            if (edit) {
                bool ok = false;
                int line = QInputDialog::getInt(this, "跳转到行", "行号:", 1, 1,
                                                 edit->document()->blockCount(), 1, &ok);
                if (ok) {
                    QTextCursor cursor = edit->textCursor();
                    cursor.movePosition(QTextCursor::Start);
                    cursor.movePosition(QTextCursor::Down, QTextCursor::MoveAnchor, line - 1);
                    edit->setTextCursor(cursor);
                    edit->ensureCursorVisible();
                }
            }
            return true; // 返回 true = 事件已处理，不要再传递
        }

        // 判断是不是 zoomLabel 被点了
        if (label && label->objectName() == "zoomLabel" && me->button() == Qt::LeftButton) {
            // 重置缩放到 100%（后续文章会实现）
            return true;
        }
    }

    // 没有处理的事件，交给父类继续处理
    return QMainWindow::eventFilter(obj, event);
}
```

### 核心流程

```
QLabel 收到鼠标点击事件
        ↓
eventFilter() 被触发（参数 obj = 被点击的 QLabel）
        ↓
通过 qobject_cast + objectName 判断是哪个 label
        ↓
执行对应的逻辑（跳转行 / 重置缩放）
        ↓
return true  →  事件到此结束
return false →  继续传递给 QLabel 自己处理
```

> ⚠️ **务必在函数末尾调用 `return QMainWindow::eventFilter(obj, event)`**，否则那些你没有处理的事件（比如窗口重绘、键盘输入等）会被直接丢弃，导致界面异常。

> 💡 **后续展望**：在后面的系列文章中，`eventFilter` 将承担更重要的角色——终端键盘输入拦截（Ctrl+C 中断、Tab 补全、历史命令回溯）、标签页中键关闭、智能代码补全等，全靠它。

---

# 本篇总结

到这里，我们的编辑器已经有了一个像样的"骨架"。回顾一下本篇实现了什么：

| 功能 | 核心技术 | 涉及的 Qt 类 |
|------|----------|-------------|
| 无边框窗口 | `setWindowFlags` | `QMainWindow` |
| 窗口拖拽移动 | `mousePressEvent` + `mouseMoveEvent` | `QMouseEvent` |
| 边缘 8px 热区缩放 | `hitTestResizeEdge` + `event()` | `QHoverEvent` |
| 自定义标题栏 | `QHBoxLayout` + `addStretch` | `QWidget`, `QPushButton`, `QLabel` |
| 暗色主题 CSS | `setStyleSheet()` | Qt Style Sheets |
| QPainter 手绘图标 | 透明画布 + 画笔 | `QPixmap`, `QPainter`, `QIcon` |
| 菜单栏融入标题栏 | `setParent()` + `setMenuWidget()` | `QMenuBar` |
| 状态栏信息面板 | `addWidget` / `addPermanentWidget` | `QStatusBar`, `QLabel` |
| 事件过滤器 | `installEventFilter` + `eventFilter()` | `QEvent`, `QObject` |

本篇核心代码量约 **200 行**，但这 200 行撑起了整个应用的界面骨架。

---

# 下一篇预告

窗口搭好了，接下来要往里面填内容了。**第 3 篇**将实现：

- **多标签页编辑器**：`QTabWidget` 实现 VS Code 风格的多文件标签页
- **文件操作**：新建、打开、保存、另存为
- **未保存提示**：关闭标签页时检测修改状态，弹出确认对话框
- **拖拽文件打开**：直接把文件从资源管理器拖进编辑器
- **最近文件列表**：用 `QSettings` 持久化最近打开的文件

敬请期待！

---

## 脚注

[^1]: `QSvgRenderer` 是 Qt 提供的 SVG 矢量图渲染器，可以将 SVG 字符串或文件渲染成 `QPixmap`。我们后续会在侧边栏图标中使用内联 SVG 字符串，实现零外部图片文件的纯代码图标方案。

[^2]: `Qt::WindowFlags` 是一个位掩码枚举，常用的标志包括：`Qt::FramelessWindowHint`（无边框）、`Qt::WindowMinimizeButtonHint`（保留最小化）、`Qt::WindowMaximizeButtonHint`（保留最大化）、`Qt::WindowStaysOnTopHint`（窗口置顶）、`Qt::Tool`（工具窗口，无任务栏图标）。多个标志可以用 `|` 组合使用。

[^3]: `RESIZE_MARGIN` 取 8 像素是基于 Windows 系统的人机交互规范。Windows 的窗口边框热区标准为 4-8 像素，VS Code 和 Electron 应用通常使用 4-6 像素。8 像素是相对宽松的选择，对触控板用户更友好。

[^4]: 项目中所有图标（活动栏、侧边栏按钮、标题栏控制按钮、断点红点、GDB 执行箭头）全部使用 `QPainter` 或 `QSvgRenderer` 纯代码绘制，不依赖任何外部图片资源。这种设计的优势在于：零文件依赖、矢量无损缩放、颜色可动态调整、打包体积更小。
