@[TOC](开启跨平台 UI 之旅)
# 前言

学完 C/C++ 语法，大家是不是早就受够了那个黑乎乎的控制台窗口？敲了半天代码，连个像样的按钮都做不出来，谁不想搞出一个真正带界面、能给别人用的桌面软件呢。

听说 Qt 跨平台很牛，结果刚打开官方安装器就傻眼了——全家桶动辄大几十个 G，勾选项看的人眼花缭乱，一不小心 C 盘直接红了。好不容易装完，顺着教程建个项目，有时只是挪了一下文件夹，就跳出一堆玄学的 CMake 编译报错，极其搞心态，分分钟劝退。

这篇专栏不废话，全是我亲自踩过的坑。直接手把手带你避开 50G 的安装器陷阱，极简配置出纯净的环境。

---


## 1️⃣Qt & QC 极简安装

> 官方地址：https://www.qt.io/development/download

Qt 那个在线安装器里面 90% 的企业级组件和移动端编译包，初阶段做个简单的桌面软件根本碰不到。

在安装列表里，**只勾选绿色项**就行了：
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/fa1fadf85f1d40cf801ed748c2e08745.png#pic_center)

* 在 **Developer Tools** 目录里，展开 **`Qt Creator`**。展开最新的 **Qt 版本号**（比如 Qt 6.x.x），在这一列里，勾选 **`sources`**
 **`MinGW 64-bit`**/**`MSVC 64-bit`** 。
 展开 **Additional Libraries** 勾选 **`Qt HTTP Server`** **`Qt Charts`** **`Qt Multimedia`** **`Qt Serial Port`**

> 如果你的电脑里装了 VS ，并打算用 VSIDE ，则三个都勾选，否则，使用 QCIDE 就勾选 `MinGW 64-bit`。
## 2️⃣Qt 简介
Qt 是一个强大的跨平台 C++ 图形用户界面应用程序开发框架。将 Qt 与 Visual Studio 强强联合，不仅能够充分发挥 VS 在代码编辑、强大调试和大型工程管理上的优势，还能无缝体验 Qt 优雅的界面构建能力。
### Qt 核心独立工具一览
> 在成功安装 Qt之后，在系统的开始菜单中找到一系列按功能划分的独立工具。它们各司其职，涵盖了从文档查阅、界面绘制到程序国际化的完整工作流：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/15df685f0ddc4f6f9677cc3842c9c6cc.png#pic_center)

> * **Qt Assistant**   官方的帮助文档阅读器。它提供了极其详尽的 Qt 类和函数介绍，支持全文检索，是你开发过程中的得力助手。
> * **Qt Designer**   强大的可视化界面设计师。支持所见即所得的拖拽式开发，**完全无需编写代码**即可快速制作出精美的 UI 界面。
> * **Qt Linguist**   专为应用国际化设计的“语言家”工具，能够高效管理和提取文本，轻松完成软件的多语言翻译工作。
> * **Qt 命令行工具**   为开发者提供纯粹的命令行环境，带有相关的编译套件（支持 MinGW 13.1.0 64-bit 与 MSVC 2022 64-bit 等）。
> * **Qt Creator**   Qt 官方出品的综合性 IDE。如果你不使用 Visual Studio，这款工具将是你的主力开发环境，其体验类似于经典的 VS2026。

## 3️⃣新建项目IDE选择与介绍
### VS
如果你已经有了项目，可以直接在项目文件夹下右键用VS打开
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/81c7b18d839345cf92afd0233c638d18.png#pic_center)
####  Qt 集成插件
打开 Visual Studio，呼出扩展窗口。搜索“Qt”。安装官方扩展插件 **`Qt Visual Studio Tools`**。
装完后，上方会提示关闭VS，跟着操作就行了。
> 注：下面的CMake tools是假的，不要装错

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/7908f2db94b34d6ea76708b9e2c2fc65.png#pic_center)
#### 选择项目模板

> 安装完插件后，再次打开VS 
> 在 Visual Studio 的起始页面点击“创建新项目”。对于绝大多数传统桌面客户端开发，我们选择 **`Qt Widgets Application`** 模板即可。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/a1a89f0365544a0ab618cfdd0d80c6c5.png#pic_center)

接下来进入“配置新项目”的常规窗口。在这里，我们输入基本的工程信息
建议勾选 **将解决方案和项目放在同一目录中** 复选框。
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/dc0175ffdb7a424e9c43c025fa534d64.png#pic_center)

配置好路径后，会正式进入 Qt 插件专属的“Qt Widgets Application Wizard”向导首页。

向导会自动检测环境，我们需要确认以下核心选项的配置：

* **Build System**：默认选择 `Qt Visual Studio Project (Qt/MSBuild)` 即可。
* **Build Configurations**：构建环境

> 这里虽然给了选项，但我们只能选一个，通常就选Debug，如果这里没有选项，就请先看[点击这里跳转](#问题解决)，配置好后重新操作。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/160bb442534a49d89e4453ef72722402.png#pic_center)

> **💡 避坑指南**
> 
> 请将视线移到向导窗口的左下角 “Project Settings” 区域。你会看到一个名为 **Enable PCH Support** 的复选框。正如图中所示的那样——**在新项目中，强烈建议不要勾选它！**
> 
> 尽管 PCH（预编译头文件）的初衷是为了加速编译，但在如今的 Qt 开发生态中，尤其是当你计划混合使用 CMake 构建系统时，IDE 自动生成的 PCH 机制往往会导致严重的依赖混乱。它会“自作主张”地缓存一些头文件，从而在跨平台编译或迁移 CMake 时引发令人头疼的配置报错。[^1] 保持代码结构的标准和干净，才是长远之计。

[^1]: 预编译头文件（Precompiled Header）虽然能将庞大且不常变动的头文件（如 `<QtWidgets>`）提前编译为二进制缓存，但在 CMake 的跨平台语法体系中，处理特定于 MSBuild 的 PCH 设置非常繁琐。为了保证项目在 Windows、Linux 和 Mac 上的一致性，放弃 IDE 专属的 PCH 优化是明智的选择。

最后，就是一些基本信息，大家看的懂的，不多介绍，唯一要注意的是图中黄框内选项![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/f55c0beb0f4748959ac82aa4ad541307.png#pic_center)

#### 初次加载 CMake 配置错误的解决办法

在 VS 中打开新创建的 Qt 项目时，可能并不会看到成功编译的提示，而是遭遇 CMake 生成失败。

在输出窗口中，你通常会看到类似下面这样的致命错误信息：

> Could not find a package configuration file provided by “Qt6” (requested version 6.5) with any of the following names:
> 
> Qt6.cps
> qt6.cps
> Qt6Config.cmake
> qt6-config.cmake

这通常是因为我们在 `CMakeLists.txt` 中要求查找特定版本的 Qt，但编译器却不知道去哪里寻找这些库文件。

注意 Visual Studio 的顶部边缘，你会发现一条醒目的黄色通知栏提示——“Qt Visual Studio Tools — You must select a Qt version to use for development. Select Qt version...”

<span id="问题解决"></span>
> 在顶部菜单栏中，依次点击 **扩展(X)** -> **Qt VS Tools** -> **Qt Versions**。
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/b4977f4723d34613a19e7229660bd362.png#pic_center)
> 点击input导入，选择你安装Qt的文件夹找到msvc2022_64文件打开
> 例：D:\Dev\Qt\6.11.1\msvc2022_64

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/4aae33b9e3124447be3998cfc83af501.png#pic_center)

等待添加后，再次生成 CMake 缓存，之前的报错就会无了。当然如果你用的是博主同款6.11版本，可能会有新的报错

> 报错原因：项目配置中开启了“将警告视为错误”（即 /WX 编译选项）。Qt 6.11 提示 compressEvent 将在 Qt 7 中移除，在头文件里触发了这个警告，编译器把它当成了严重错误，直接中断了生成。

在“解决方案资源管理器”中，右键点击你的项目 VS_Qt_CMake，选择最下方的 “属性” 。

在左侧菜单展开 C/C++ -> “预处理器” 。

在 “预处理器定义”中，点击编辑，添加一行宏定义：QT_NO_DEPRECATED_WARNINGS。

点击“应用”并“确定”，然后重新生成解决方案。
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/b272090f924241fc9be452514306b295.png#pic_center)
再次运行就成功出现这个弹窗，这就是你未来大展身手的应用UI界面
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/2ddc1bbab1954c74b1580f37febc1db3.png)


> 如果你在配置路径时遇到更复杂的环境变量问题，可以随时参考 [Qt 官方 CMake
> 配置文档](https://doc.qt.io/qt-6/cmake-get-started.html) 获取权威解答。

### QC
QC的项目创建非常简单，<kbd>win</kbd> + <kbd>s</kbd>搜索Qt，点击绿色QC图标打开，点击创建项目![点击创建项目](https://i-blog.csdnimg.cn/direct/505fc7751e264167b26452fa22e5e2d4.png#pic_center)

> 模板选择

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/cb41e4898e56453b9e6cf3b158a56cb4.png#pic_center)
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/442c6de5367844f280e6aa9a847d8679.png#pic_center)

> 选CMake或CMake with Qt 5....
> 千万不要Qbs，版本弃子

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/914be10629dc417a9021d903f898f633.png#pic_center)

> 基本信息设置，按需选择

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/df537f18b96b46ca8cc22384f1ce4f46.png#pic_center)

> 在QC中，我们用MinGW更好，只需勾选前两个

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/5e18071b75b34c58b99d6e53bea504dc.png#pic_center)

---

## 4️⃣ 项目界面简介
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/69231df623a24d84a050732fdccb4ce1.png#pic_center)
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/8ad280a25d074951ab4f21feadbd0915.png#pic_center)

------

希望这篇文章能帮你快速扫平 Qt 的配置障碍！如果觉得对你有帮助，欢迎点赞收藏。我们下一篇博客将正式进入 Qt 界面的核心开发，敬请期待！
