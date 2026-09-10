@[TOC](🎯目录)
## 0. 目录前置（太长不看 TL;DR & 快速导航）

> **💡 博主贴心提示：**
> 知道大家时间宝贵，写代码最烦的就是看那种婆婆妈妈的长篇大论。
> 这里特意给心急的同学准备了**“10分钟通关秘籍”**。只要严格跟着下面这三步走，喝杯水的功夫就能全部搞定！

**⚡ 快速通关路线图：**

* **[Step 1: 下载并安装编译器 (MinGW-w64)](https://release-assets.githubusercontent.com/github-production-release-asset/446033510/7e78df21-05e1-4dd7-91f6-6f82abe936c5?sp=r&sv=2018-11-09&sr=b&spr=https&se=2026-06-10T07%3A24%3A51Z&rscd=attachment%3B+filename%3Dx86_64-16.1.0-release-posix-seh-ucrt-rt_v14-rev1.7z&rsct=application%2Foctet-stream&skoid=96c2d410-5711-43a1-aedd-ab1947aa7ab0&sktid=398a6654-997b-47e9-b12b-9515b896b4de&skt=2026-06-10T06%3A24%3A34Z&ske=2026-06-10T07%3A24%3A51Z&sks=b&skv=2018-11-09&sig=Pg5GNcU3ypcbttvEcnFbXQ1tp4lAm9OlBbtS30sD9lU%3D&jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmVsZWFzZS1hc3NldHMuZ2l0aHVidXNlcmNvbnRlbnQuY29tIiwia2V5Ijoia2V5MSIsImV4cCI6MTc4MTA3NjU3NiwibmJmIjoxNzgxMDcyOTc2LCJwYXRoIjoicmVsZWFzZWFzc2V0cHJvZHVjdGlvbi5ibG9iLmNvcmUud2luZG93cy5uZXQifQ.sERPseBQEFhBl_jExl2snu_3lVGwe_FT5MlBMG-XGl8&response-content-disposition=attachment%3B%20filename%3Dx86_64-16.1.0-release-posix-seh-ucrt-rt_v14-rev1.7z&response-content-type=application%2Foctet-stream)**(官网版GitHub) 
[下载并安装编译器 (MinGW-w64)](https://download.csdn.net/download/ysu_0314/92963324)(CSDN资源版) —— ⏳ 耗时约 3 分钟
* **[Step 2: 配置系统环境变量](#jump_step2)** —— ⏳ 耗时约 2 分钟
* **[Step 3: 安装 VSCode 核心插件](#jump_step3)** —— ⏳ 耗时约 3 分钟
* **[Step 4: 一键获取配置文件包](https://download.csdn.net/download/ysu_0314/92968471)** —— ⏳ 耗时约 2 分钟

## 1. 痛点Hook：你是不是也快被环境配置逼疯了？
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/176d63ed938f4b7a97295f77222d6623.png#pic_center)

家人们，咱就是说，你是不是也经历了这样一个令人绝望的下午？

本来这是一个惬意的周末，你正准备开着超跑在地平线4乡间快乐兜风，或者单纯想打两把游戏放松一下。但在那之前，你想着先用先用刚下载的VSCode，速战速决搞定 C 语言的课后代码。

结果，噩梦开始了。

### 盲目照搬，越搞越乱
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/9f9ee30381684e22a9e3aac41feeba35.png#pic_center)

故事的开头总是相似的：你满怀学霸之魂，准备大干一场。你打开搜索引擎，翻出了一篇不知道是哪年哪月的“老古董”教程。接下来，你开启了**盲目复刻模式**：

* 在扩展商店里装了一堆不知名、连读都读不顺溜的奇怪插件；
* 跟着毫无逻辑的步骤，把系统的环境变量改得面目全非；
* 看着毫无反应的黑框框，你甚至开始怀疑是不是输入法设置有冲突，或者是系统编码导致了什么玄学报错，一通瞎关，结果毫无卵用。

### 压死骆驼的最后一个 F5

折腾了两个小时，你以为终于大功告成了？好不容易敲完一段简简单单的 `Hello World` 测试代码，你深吸一口气，双手合十，无比虔诚地按下了键盘上的 **F5** 键……
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/f3f9a2d41db94bf7ac66e874c46839ad.png#pic_center)

结果呢？终端里弹出的根本不是象征胜利的输出，而是一串能让人瞬间血压飙升的刺眼红字：
==`'gcc' 不是内部或外部命令，也不是可运行的程序或批处理文件。`==

还没等你从心肺骤停中缓过神来，紧接着系统又给你补了一记重拳——屏幕中央弹出了一个让人彻底崩溃的对话框：
==`preLaunchTask "C/C++: gcc.exe build active file" terminated with exit code -1`==

### 别慌，这篇教程就是你的“速效救心丸”
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/a2a4612ac144470abbe99c02ba5117a0.png#pic_center)


看着乱码一样的 `.json` 配置文件，屏幕前的你是不是已经在怀疑人生，甚至想直接卸载保平安了？
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/ccf8cb6e00484e839d409bc686bcd8cb.png#pic_center)

**别急着关电脑！** 作为最喜欢踩坑的技输博主，我太懂这种在入门第一步就被门槛绊倒的挫败感了。配置环境最怕的就是**知其然而不知其所以然**的瞎折腾。

今天这篇**纯纯的保姆级、无死角避坑指南**，不扯高深概念，不整花里胡哨的废话。我将手把手带你从零开始，把乱七八糟的旧配置全部清空，只用最干净、最直接，最简单，最`豆包`的方式，带你一站式打通 C 语言的编译与调试任督二脉。

准备好了吗？跟着我往下看，保证你今天按下的 F5，绝对顺滑如丝！
> ps: 应该没人不会下vscode吧?以防万一官方下载链接放这：
下载地址：https://code.visualstudio.com/
## 2.VSCode 核心界面认门

> 我们先来认识一下VSCode界面

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/ca8bac70c0fc4a5f88697ad6961448f5.png#pic_center)
其实拆解开来，VSCode 的整个大界面，我们日常高频用到的只有 **四大核心板块**：
### 1️⃣ 活动栏
位于界面最左侧的一竖排图标。这里是切换各个视图的地方，最上面的“文件”图标就是我们接下来要说的资源管理器，下面的放大镜、插件积木图标等，点一下就能让旁边变出不同的面板。
### 2️⃣ 资源管理器边栏
活动栏右侧展开的这个大列表，就是你当前写代码的“工作区”目录。
* **博主敲黑板**：写代码最忌讳的就是文件乱丢、桌面一团糟！强烈建议大家养成好习惯，像优秀工程师一样在里面建好清晰的文件夹（比如按照日期、课程进度或者项目名称分门别类搞好层级）。把 `.vscode` 配置文件夹、你的 `.c`/`.cpp` 源代码文件规规矩矩地收纳在这里，以后复习或者交作业再也不用全盘乱翻。

### 3️⃣ 中央代码编辑区（Editor）
屏幕正中间、带有数字行号的黑色大区域，就是你运筹帷幄、挥洒才华的地方。你所有惊艳的 C 语言代码、复杂的算法公式，全是在这里一行行敲出来的。这里带有智能代码高亮，写错了字母会贴心地给你打上波浪线。

### 4️⃣ 底部面板与终端（Panel & Terminal）
写完代码、按下编译运行之后，屏幕下半部分就会顺滑地吐出这个面板。
* 我们打交道最多的就是 **“终端（Terminal）”** 这一栏。它就像是一个“双向沟通”的窗口：你的代码里用 `printf` 打印出来的中文成果会在这里展示；而代码里需要你用键盘输入的数字、字符，也是在这里敲击回车送进电脑的。

---
## 3. 核心实操步骤（手把手极简流程）

### 步骤一：搞定编译器（MinGW-w64）

> VS与Vscode的区别：VS中集成了MSVC编译器

很多新手容易产生一个误区：以为装了 VSCode 就能直接写代码运行了。**错！大错特错！**
敲黑板了同学们，**VSCode 本质上只是一个高级点的“记事本”**，它只有让你把代码写得更好看的功能，但它自己**不会**把代码翻译成电脑能懂的语言（也就是[编译](https://blog.csdn.net/ysu_0314/article/details/161634695?fromshare=blogdetail&sharetype=blogdetail&sharerId=161634695&sharerefer=PC&sharesource=ysu_0314&sharefrom=from_link)）。所以，我们需要给它配一个“翻译官”——编译器。

目前主流的 C/C++ 编译方案有三种，我给大家列了个极简对比表格：

| 编译器方案 | 适用人群 / 特点 | 推荐指数 |
| :--- | :--- | :--- |
| **MinGW-w64** | 轻量级，安装极简，**最适合新手/轻量开发** | ✅ **本文首选，五星爆推！给到夯** |
| **MSVC (Visual Studio)** | 微软亲儿子，适合重型工程、Windows原生开发，但太臃肿，且太集成，不适合了解底层逻辑 | ⭐⭐⭐（杀鸡焉用牛刀）只能给到NPC |
| **WSL (Linux子系统)** | 极客专属，适合后期进阶、服务器开发或搞大模型部署 | ⭐⭐⭐⭐（新手暂勿轻易碰瓷）给到顶级 |

**👉 极速操作指南：**
1. 去官网下载 MinGW-w64 的压缩包。
> 直接edge搜索minGW，认准www.mingw-w64.org网址

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/5ec1b03871ac4a3dbd77d77e1feb77d2.png#pic_center)

> 进入后先点击Downloads，再点击Pre-built

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/302ab24537fe46e28bf30a2c9daf5e47.png#pic_center)

> 下滑找到MinGW-W64-builds，点击进入

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/487fa4a6f9884dfaa0a239411f2e8782.png#pic_center)

> 跳转后最上方会有GitHub链接，稍加魔法后点击进入

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/08811008f63946d1b8f760488b1443d8.png#pic_center)

> 下滑找到这个，点击下载

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/9b307c7ce1a04ec4b4d7a67d0fd3ccfd.png#pic_center)

2. 解压到电脑的 D 盘你能找到的路径，路径参考：`D:\mingw64`。
3. **⚠️ 致命强调**：你的解压路径里，**绝对、绝对、绝对不能出现中文或者空格！** 哪怕叫 `D:\我的代码\mingw64` 都会让你后期的报错千奇百怪！

> 当然，由于GitHub下载太慢，这里也为大家准备了下载包：[下载地址](https://download.csdn.net/download/ysu_0314/92963324)

---
<span id="jump_step2"></span>
### 步骤二：配置环境变量

这一步是让你的 Windows 电脑能够认识刚刚下载的“翻译官”。

> 按下 `Win` 键，直接搜索 **“环境变量”**，点击“编辑系统环境变量”。
> 
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/669afe7ca25541d696834b1b48f276ba.png#pic_center)
> 点击右下角的“环境变量“按钮。在系统变量中找到Path双击

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/02cb562495874e0e80d611033a0a4730.png#pic_center)

> 点击新建，将刚才解压的mingw64\bin地址填入

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/9c001949e1844e898271eda3b6318faa.png#pic_center)


> 🚨 **【避坑警告】**
> 配置完环境变量后，**必须、必须、必须彻底重启你的 VSCode！**。如果不重启，VSCode 会一直读旧的系统状态，终端依然会无情地提示你：`找不到 gcc`！

---
<span id="jump_step3"></span>
### 步骤三：VSCode 插件

> vscode自带的插件商店在最左侧第六个模块，或者快捷键`ctrl+shift+x`

VSCode 本身只是个毛坯房，想把它变成能流畅跑 C /C++语言的豪华别墅，全靠“软装”——也就是装插件。
但新手千万别去商店里一通乱搜、看哪个顺眼就装哪个！装多了不仅拖慢电脑速度，还会引起各种玄学冲突。

#### 🧱 第一类：必备

1. **`Chinese (Simplified) Language Pack`** 
   * **介绍**：母语看着就是最亲切的！装完重启，满屏的英文界面瞬间变中文，极大降低学习难度。**认准 Microsoft 官方出品**。
   ![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/a1fe202b0af347c48c1e4d96c16a2504.png#pic_center)

2. **`C/C++` 与 `C/C++ Extension Pack`** 
   * **介绍**：这是 VSCode 写 C/C++ 的灵魂！它俩打包为你提供了代码高亮、智能提示（输一个字母自动补全一长串）以及最核心的调试功能。
   ![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/8d1fb834750b490f9ef43b4fbfea1f41.png#pic_center)

3. **`C/C++ Runner`** 
   * **介绍**：很多老教程会让你装 Code Runner，但其实 `C/C++ Runner` 对咱们这门语言的适配更友好。装上它之后，你会发现状态栏多了一键编译运行的小火箭 🚀，写完单文件代码直接点一下就能跑，省去了手敲 `gcc` 命令的折磨。
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/5a87187f01554e81a06279b963de7fdf.png#pic_center)

#### 🚀 第二类：扩展提升

当你把基本的 `Hello World` 跑通后，下面这几个插件能让你爱不释手：

1. **`GBKtoUTF8`** 
   * **介绍**：Windows 系统的中文编码经常和 VSCode 的 UTF-8 冲突，导致终端输出满屏的“烫烫烫”。这个轻量级小插件能帮你自动转换编码，完美拯救乱码强迫症。
   ![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/2ad3cb93f19947c4adf1429f1ebb7257.png#pic_center)

2. **`koroFileHeader`** 
   * **介绍**：大学生写上机作业或实验报告必备！配置好之后，按个快捷键，代码文件开头就会自动生成非常专业的文件头注释（还有图案）。
   ![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/492e25a016cd4e03887a5bc4875c9e23.png#pic_center)![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/e9cab8e2e1c64b6ba701124ec7432af6.png#pic_center)

3. **`vscode-icons`** 
   * **介绍**：默认的文件夹图标太单调了。装上它，你的 C 文件、头文件、系统文件夹都会换上各种精美好看的专属 Icon，不仅找文件更直观，敲代码的心情都能变好。
   ![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/7ea9a58bfb12462dbf9fce2498dbc0db.png#pic_center)

4. **`Include Autocomplete`** 
   * **介绍**：记不住那么多的 `#include` 库名字？这个插件能在你敲代码引入头文件时，自动给你下拉提示，省去翻书查单词的时间。

> **💡 进阶折腾党专属提示：**
> 如果你在插件里还看到了 `CMake Tools`、`CodeLLDB` 或者 `Remote - SSH`，新手小白请**直接无视它们**！那是给以后做大型工程项目，或者远程连接服务器部署跑大模型用的。咱们初学阶段，老老实实装前面那几个就完全足够了，别给自己增加不必要的配置负担！

### 步骤四：C / C++ 配置

> 当然光有了插件代码依旧跑不畅快，因此我们需要做一些必要的配置

#### Ⅰ. 创建工作区与配置文件夹
跟着博主一步步来，不要自己乱建：
1. 在 D 盘建一个专门写代码的文件夹，**名字必须全英文**，比如 `D:\C_code`，然后在 VSCode 里“打开文件夹”把它打开。
2. 在这个文件夹里，新建一个名字叫 `.vscode` 的文件夹（注意前面有个点！）。


#### Ⅱ. C / C++ 配置

> **💡 博主敲黑板：**
> 很多网上的教程最不负责任的地方就在于：把 C 和 C++ 的配置混为一谈！结果就是你写 `.c` 文件时报 C++ 的错，写 `.cpp` 时编译器又找不到库。
> 记住了：**C 语言的专属编译器叫 `gcc`，C++ 的专属编译器叫 `g++`！** 下面博主将采用类似官方 UI 引导的方式，手把手带你分别解开 C 和 C++ 的封印。不管你学哪个，都能精准对号入座！

---

##### 🛠️ 第一部分：C 语言配置指南（面向 `.c` 文件）

> 注：以下mingw64路径皆以`D:\mingw64\bin`为例，如果你下载在其他路径，请注意替换

######  第1️⃣步：召唤 `c_cpp_properties.json`（智能提示与头文件配置）
> 在 VSCode 中按下快捷键 `Ctrl + Shift + P` 唤醒万能命令面板，输入 `Edit
> Configurations`，在下拉菜单中点击 **`C/C++: 编辑配置 (UI)`**。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/0ade5397451049eca23612b18abe9a0d.png#pic_center)

> 在弹出的图形界面中进行以下三项关键修改：
>      **编译器路径**：手动填入或选择 `D:\mingw64\bin\gcc.exe`（注意是 **gcc** ）。具体路径为你的mingw64路径
>      **IntelliSense 模式**：选择 `windows-gcc-x64`。
>      **C 标准**：选择 `c11` 或 `c17`。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/dc3ec20e32f84ce3b4b87eeba7fc028c.png#pic_center)

>  **一键抄作业**：此时 `.vscode` 文件夹下会自动生成一个
> `c_cpp_properties.json`。如果你嫌点UI麻烦，直接新建该文件并copy以下代码

```json
{
    "configurations": [
        {
            "name": "windows-gcc-x64",
            "includePath": [
                "${workspaceFolder}/**"
            ],
            "compilerPath": "D:/mingw64/bin/gcc.exe",
            "cStandard": "c17",
            "cppStandard": "${default}",
            "intelliSenseMode": "windows-gcc-x64",
            "compilerArgs": [
                ""
            ]
        }
    ],
    "version": 4
}
```

######  第2️⃣步：定制 `tasks.json`（C 语言的编译任务）
 

> 点击顶部菜单栏的 `终端` -> `配置（默认生成）任务`，在弹出的列表中选择 **`C/C++: gcc.exe 生成活动文件`**。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/ca15bbab0b39405b94b6441abf1af0e1.png#pic_center)
> **一键抄作业**：VSCode 会自动在 `.vscode` 目录下生成一个 `tasks.json`。下面这段博主精心调教的 C 语言专属编译配置，即适配中文编码，避免乱码，又能同时编译多.c文件
```json
{
	"version": "2.0.0",
	"tasks": [
		{
			"type": "cppbuild",
			"label": "C: gcc.exe 生成活动文件",
			"command": "D:/mingw64/bin/gcc.exe",
			"args": [
				"-fdiagnostics-color=always",
				"-g",
				"${workspaceFolder}\\*.c",
				"-o",
				"${workspaceFolder}\\${workspaceRootFolderName}.exe",
				"-fexec-charset=GBK" 
			],
			"options": {
				"cwd": "${workspaceFolder}"
			},
			"problemMatcher": [
				"$gcc"
			],
			"group": {
                "kind": "build",
                "isDefault": true
            }
		}
	]
}
```

######  第3️⃣步：激活 `launch.json`（C 语言断点调试）

> 点击 VSCode 左侧侧边栏的“大虫子加播放键”图标（运行和调试），点击“创建 launch.json 文件”，选择配置 `C++ (GDB/LLDB)` 

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/6a0d33f7041342a6b7ef52e93a090708.png#pic_center)


>  **一键抄作业**：在 `.vscode` 文件夹下手动新建一个 `launch.json`文件，直接把下面这段代码copy进去

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "C Runner: Debug Session",
      "type": "cppdbg",
      "request": "launch",
      "program": "${workspaceFolder}\\${workspaceRootFolderName}.exe",
      "args": [],
      "stopAtEntry": false,
      "cwd": "${fileDirname}",
      "environment": [],
      "externalConsole": false,
      "MIMode": "gdb",
      "miDebuggerPath": "D:/mingw64/bin/gdb.exe",
      "setupCommands": [
        {
          "description": "为 gdb 启用整齐打印",
          "text": "-enable-pretty-printing",
          "ignoreFailures": true
        }
      ],
      "preLaunchTask": "C: gcc.exe 生成活动文件"
    },
  ]
}
```

> 注：launch.json中的preLaunchTask必须与tasks.json中的label内容一致

---

##### 🧪 第二部分：C++ 配置指南（面向 `.cpp` 文件）

> 如果你学习的是面向对象程序设计、数据结构或者更高级的开发，主要写 `.cpp` 结尾的文件，请使用下面这套 **G++ 编译器**！

######  第1️⃣步：召唤 `c_cpp_properties.json`（指向 G++）

> **一键抄作业**：同样是通过 `Ctrl + Shift + P` 进入，但我们要把编译器换成 `g++.exe`，C++标准拉到 `c++17`。直接copy以下内容

```json
{
    "configurations": [
        {
            "name": "windows-gcc-x64",
            "includePath": [
                "${workspaceFolder}/**"
            ],
            "compilerPath": "D:/mingw64/bin/g++.exe",
            "cStandard": "c17",
            "cppStandard": "c++17",
            "intelliSenseMode": "windows-gcc-x64"
        }
    ],
    "version": 4
}
```

######  第2️⃣步：定制 `tasks.json`（ C++ 编译内核）

> **一键抄作业**：C++ 编译必须调用 `g++.exe`。 `tasks.json`

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "type": "cppbuild",
            "label": "C++: g++.exe 编译当前文件夹下所有 C++ 文件",
            "command": "D:/mingw64/bin/g++.exe",
            "args": [
                "-fdiagnostics-color=always",
                "-g",
                "${fileDirname}\\*.cpp", 
                "-o",
                "${fileDirname}\\program.exe", 
                "-fexec-charset=GBK" 
            ],
            "options": {
                "cwd": "${workspaceFolder}"
            },
            "problemMatcher": [
                "$gcc"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            }
        }
    ]
}
```

######  第3️⃣步：激活 `launch.json`（C++ 专属调试骨架）

> **一键抄作业**：`launch.json`中gdb.exe调试器同时支持C/C++，所有只需要将预启动任务的名字与 tasks.json 对齐

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "C++ Runner: 多文件调试会话",
            "type": "cppdbg",
            "request": "launch",
            "program": "${fileDirname}\\program.exe",
            "args": [],
            "stopAtEntry": false,
            "cwd": "${fileDirname}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "miDebuggerPath": "D:/mingw64/bin/gdb.exe", 
            "setupCommands": [
                {
                    "description": "为 gdb 启用整齐打印",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ],
            "preLaunchTask": "C++: g++.exe 编译当前文件夹下所有 C++ 文件"
        }
    ]
}
```

---

#### Ⅲ. 测试运行🏁 
> 配置好后，以后再写C/C++代码时，可以直接新建一个项目，将C的.vscode/C++的.vscode文件直接复制进去

在项目下你可以建一个 `test.c` 或 `demo.cpp` 塞入标准测试代码。
```cpp
#include <iostream>
#include <iomanip>

using namespace std;

int main() {
    double celsius, fahrenheit;
    
    cout << "===========================" << endl;
    cout << "   气象数据极速转换终端" << endl;
    cout << "===========================" << endl;
    cout << "请输入当前摄氏度 (℃): ";
    
    // 如果运行时这行代码卡住，记得在终端里用鼠标点一下输入数字！
    cin >> celsius;

    // 气象学核心转换公式：F = C * 1.8 + 32
    fahrenheit = (celsius * 9.0 / 5.0) + 32.0;

    cout << fixed << setprecision(2);
    cout << "   转换结果: " << fahrenheit << " ℉" << endl;
    cout << "\n  Hello World! 环境配置大成功！可以去吃顿好的了！" << endl;

    return 0;
}
```
只要你停留在 `.c` 或 `.cpp` 页面上，**按下 F5 键**，VSCode 就会自动识别你当前的文件类型，调用你刚刚配置好的编译任务，丝滑流畅地弹出终端并打印出正确答案！
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/9ac370aa7950487aa164a33a1bd08c06.png#pic_center)

再也没有报错弹窗，再也没有玄学不兼容，属于你的丝滑编程时代，正式开启！
## 4. 常见报错排雷指南（新手必看！专治各种不服）

> **💡 博主贴心提示：**
> 很多新手在跑通代码前，都会被各种奇奇怪怪的报错搞得怀疑人生。别怕，你遇到的坑，前辈们早就踩过无数遍了。这里整理了全网搜索量最高的两个“爆款”报错，帮你精准排雷！

### 💥 Q1：终端输出中文乱码怎么办？（满屏“烫烫烫”和“锟斤拷”）
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/5925a6b088eb4c1eb3af9170cf2762b0.png#pic_center)
好不容易代码能跑了，满心欢喜地输入一句中文，结果按完 F5 弹出来的全是不认识的火星文。这其实是 Windows 系统默认编码（GBK）和 VSCode 默认编码（UTF-8）冲突导致的。

**【✅ 保姆级解决方案】**

这里给大家提供两种解法，任选其一即可：

* **解法 A（治标法：临时修改文件编码）：**
  看 VSCode 的右下角，找到写着 **`UTF-8`** 的地方，点它一下。在顶部弹出的菜单中选择 `通过编码保存 (Save with Encoding)`，然后搜索并选择 **`GBK`**（或者 `Simplified Chinese (GBK)`）。重新编译运行即可。
* **解法 B（治本法：修改编译参数 —— 博主强烈推荐⭐）：**
  一劳永逸的方法是直接告诉编译器“请用兼容 Windows CMD 的格式输出中文”。
  打开我们刚刚配置的 `.vscode` 文件夹里的 
`tasks.json`，在 `"args"` 那个中括号里，加上一行神奇的咒语：
  `"-fexec-charset=GBK",`
  * 如果你采用的就是本文配置，则无需重复添加



---

### 💥 Q2：代码第一行出现红色波浪线，提示 `include errors detected`？

代码明明是照着书上一个字母一个字母敲的，结果第一行就被打上了醒目的红叉叉，鼠标放上去一看，提示“找不到头文件”。新手看到这个往往心里一紧：完了，我的 C/C++ 库是不是没装好？

**【✅ 保姆级解决方案】**

别慌！这不是你的代码有问题，也不是你的电脑坏了，而是 **VSCode 的“代码提示功能（IntelliSense）”迷路了**。它不知道你的编译器（也就是那些核心的头文件库）藏在电脑的哪个角落。

**解决步骤：**
1. 打开我们刚才建好的 `.vscode` 文件夹，找到 `c_cpp_properties.json` 文件。
2. 找到里面叫 `"compilerPath"` 的这一行。
3. 核对路径！ 看看引号里的路径是不是和你真实安装 MinGW 的路径**完全一致**。如果你没有听从博主的建议把 MinGW 解压到 `D:\mingw64`，而是放到了其他路径，那你必须把这行改成你自己的真实路径，例如：
   `"compilerPath": "D:/你的安装目录/mingw64/bin/g++.exe"`

改完保存，稍微等两三秒，你会发现那条惹人烦的红色波浪线奇迹般地消失了！
## 5. 结尾：别让报错浇灭了你写代码的热情！

> **🎉 恭喜通关！**
> **走到这里，你已经成功跨过了新手最难的一道坎，完美配置好了 C/C++ 开发环境！** 现在，你可以正式开始你的编程之旅，或者安心去写你的课后大作业了。
> **强烈建议大家一键三连并收藏本文！** 下次重装系统、换新电脑，或者是帮班里的同学配环境时，不用再去网上大海捞针，直接翻出这篇教程无脑“抄作业”，绝对能帮你省下大把时间！

👇 **【评论区等你的故事】**

老实交代，你在配置 C/C++ 环境的这条路上，**踩过最坑、最让人破防的报错是什么？**
是死活找不到的编译器路径？满屏幕让人头皮发麻的乱码？还是遇到过什么玄学问题？

**欢迎在评论区大倒苦水！** 如果你在实操中碰到了文中没有提到的“奇葩”报错，千万别一个人默默生闷气，直接把错误代码或者提示贴在评论区。博主常驻后台，**在线帮你无情抓虫（Debug）！** 🐛🔨
