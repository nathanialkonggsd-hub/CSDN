# 🚀 Claude Code 安装与使用完全指南（26.6.26最新版）| 国内用户专属教程

> 📝 **前言**：Claude Code 是我目前用过最强大的 AI 编程工具，没有之一。它可能不是最简单的，但绝对是上限最高的。一旦跑通安装、接上模型、定好规范，你会发现很多原本需要几小时的工作，现在几分钟就能搞定。

 ![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/c692867437fa4642b7e3930f1b7b666e.png#pic_center) 

---

@[TOC]( 目录)

---

## 一、Claude Code 是什么？

### 1.1 一句话介绍

**Claude Code = 能动手干活的 AI 编程助手**

你平时用的豆包、DeepSeek、Kimi 这些网页聊天，都只能「说」不能「做」。而 Claude Code 是一个运行在你电脑终端里的 AI 编程助手，它能直接操作你的代码文件。

### 1.2 对比普通 AI 聊天

| 特性 | 普通AI聊天（豆包/DeepSeek/Kimi） | Claude Code |
|:---:|:---:|:---:|
| 🎯 交互方式 | 只能聊天讨论 | **直接操作文件** |
| ✏️ 代码修改 | 需要复制粘贴 | **自动修改代码** |
| ⚡ 任务执行 | 一问一答 | **自动执行多步骤任务** |
| 📁 项目支持 | 需要手动上传 | **直接读取项目文件夹** |
| 🔧 工具调用 | 不支持 | **支持终端命令、Git等** |

> 💡 **一句话总结**：网页聊天是顾问给建议，Claude Code 是打工人直接干活。

### 1.3 核心优势

```
✅ 框架本身足够强 - 上下文理解、文件操作、工具调用这些底层能力，Claude Code是目前做得最扎实的
✅ 模型可替换 - Claude官方模型太贵？可以接DPV4、GLM-5.2、MiniMax M3等国产模型
✅ 没有封号风险 - 不用国外手机号、不用Visa卡，完全本地化的部署路径
```

---

## 二、安装前准备

### 2.1 系统要求

| 操作系统 | 最低版本 | 推荐版本 |
|:---:|:---:|:---:|
| 🪟 Windows | Windows 10 | Windows 11 |
| 🍎 macOS | 10.15 (Catalina) | macOS 14+ |
| 🐧 Linux | Ubuntu 22.04+ / Debian 11+ | 最新稳定版 |

### 2.2 必装软件

#### Node.js 18+（必须）

下载地址：[https://nodejs.org/](https://nodejs.org/)

安装时一路 Next 即可，安装完成后打开终端验证：

```bash
node --version
# 应该显示 v18.x.x 或更高版本
```

#### Git for Windows（Windows 用户必须）

下载地址：[https://git-scm.com/download/win](https://git-scm.com/download/win)

安装时保持默认选项，一路 Next 即可。

### 2.3 设置国内 npm 镜像

国内直接用 npm 下载较慢，建议先配置国内镜像：

```bash
npm config set registry https://registry.npmmirror.com
```

验证配置是否成功：

```bash
npm config get registry
# 应该显示 https://registry.npmmirror.com
```

---

## 三、Windows 安装教程

### 3.1 方法一：官方安装脚本（推荐）✨

以 **管理员身份** 打开 PowerShell（按 `Win + X`，选择"Windows PowerShell (管理员)"），运行：

```powershell
irm https://claude.ai/install.ps1 | iex
```

### 3.2 方法二：使用 npm 安装

```bash
npm install -g @anthropic-ai/claude-code
```

### 3.3 方法三：使用 winget 安装

```bash
winget install Anthropic.ClaudeCode
```

### 3.4 验证安装

安装完成后，在终端输入以下命令检查版本号：

```bash
claude --version
```

如果正常显示版本号（如 `v2.1.191`），则说明安装成功！🎉

### 3.5 安装完成界面

输入claude
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/150e722b6d5742c39157e5c88726615e.png#pic_center)
> 📌 **注意**：安装完成后，Claude Code 的可执行文件默认在 `C:\Users\你的用户名\.local\bin`。如果命令无法识别，需要将此路径添加到系统环境变量。

---

## 四、macOS 安装教程

### 4.1 方法一：官方安装脚本（推荐）

打开终端（按 `Command + 空格`，搜索"终端"），运行：

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

如果提示权限问题，改用：

```bash
sudo curl -fsSL https://claude.ai/install.sh | bash
```

### 4.2 方法二：使用 Homebrew

先安装 Homebrew（如果已有可跳过）：

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

然后安装 Claude Code：

```bash
brew install --cask claude-code@latest
```

### 4.3 验证安装

```bash
claude --version
```

---

## 五、Linux 安装教程

### 5.1 安装 Node.js

```bash
# 安装 Node.js LTS 版本
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo bash -
sudo apt-get install -y nodejs

# 验证安装
node --version
```

### 5.2 安装 Claude Code

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

### 5.3 验证安装

```bash
claude --version
```

---

## 六、国内用户配置（重点！）

> ⚠️ **重要提示**：由于网络原因，国内用户需要配置中转站才能正常使用 Claude Code。这是最关键的一步！

### 6.1 配置 onboarding 完成状态

> 如果你使用的cc switch，请跳过。建议直接使用cc switch

首次启动前，需要在配置文件中添加已完成引导的状态，**否则会卡在初始化界面**。
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/cdde7acdb8894673899ad3a33e8836c5.png#pic_center)
#### Windows 系统

配置文件位置：`C:\Users\你的用户名\.claude.json`

如果文件不存在，请新建；如果已存在，请在花括号内添加：

```json
{
  "hasCompletedOnboarding": true,
  "autoUpdaterStatus": "disabled"
}
```

> 💡 添加 `autoUpdaterStatus` 为 `disabled` 可防止软件尝试自动更新导致连接超时。

#### macOS / Linux 系统

配置文件位置：`~/.claude.json`

```bash
# 编辑配置文件
vi ~/.claude.json
```

添加以下内容：

```json
{
  "hasCompletedOnboarding": true
}
```

### 6.2 配置 API 端点（方法一：推荐使用 CC-Switch）

详见 [第七节：CC-Switch 多模型管理工具](#jump_step2)

### 6.3 配置 API 端点（方法二：手动配置 settings.json）

#### Windows 系统

配置文件位置：`C:\Users\你的用户名\.claude\settings.json`

```json
{
  "env": {
    "ANTHROPIC_API_KEY": "sk-你的API密钥",
    "ANTHROPIC_BASE_URL": "https://你的中转站地址",
    "API_TIMEOUT_MS": "3000000",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1"
  },
  "model": "claude-sonnet-4-6",
  "permissions": {
    "allow": [
      "Bash(*)",
      "Edit",
      "Write",
      "Read",
      "Glob",
      "Grep",
      "WebFetch(*)",
      "WebSearch(*)",
      "NotebookEdit",
      "Agent"
    ]
  }
}
```

#### macOS / Linux 系统

配置文件位置：`~/.claude/settings.json`

```json
{
  "env": {
    "ANTHROPIC_API_KEY": "sk-你的API密钥",
    "ANTHROPIC_BASE_URL": "https://你的中转站地址",
    "API_TIMEOUT_MS": "3000000",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1"
  },
  "model": "claude-sonnet-4-6"
}
```

### 6.4 使用 PowerShell 配置环境变量（Windows）

打开 PowerShell（管理员），运行以下命令：

```powershell
# 请先替换下面的 你的 API 密钥
[System.Environment]::SetEnvironmentVariable('ANTHROPIC_BASE_URL', 'https://你的中转站地址', 'User')
[System.Environment]::SetEnvironmentVariable('ANTHROPIC_API_KEY', 'sk-你的API密钥', 'User')
[System.Environment]::SetEnvironmentVariable('CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC', '1', 'User')
[System.Environment]::SetEnvironmentVariable('API_TIMEOUT_MS', '3000000', 'User')
```

**设置完成后，请重启 PowerShell 窗口以生效。**

---

<span id="jump_step2"></span>
## 七、CC-Switch 多模型管理工具

### 7.1 什么是 CC-Switch？

CC-Switch 是一个图形化工具，可以让你在 Claude Code 中一键切换不同模型（DeepSeek、GLM、MiniMax 等），无需手动修改配置文件。
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/fc8ad638872b4e8f9ff24974f8ab5ed2.png)


### 7.2 安装 CC-Switch

#### Windows 安装

1. 下载安装包：[GitHub Releases](https://github.com/farion1231/cc-switch/releases)
2. 文件名：`CC-Switch-v3.16.3-Windows.msi`
3. 双击安装，一路 Next

> 📦 本文章已提供安装包：[`CC-Switch-v3.16.3-Windows.msi`](https://download.csdn.net/download/ysu_0314/93035091)

#### macOS 安装

```bash
brew tap farion1231/ccswitch
brew install --cask cc-switch
```

### 7.3 使用 CC-Switch 配置
1. 打开 CC-Switch
2. 点击右上角 "+" 号
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/13bafe53212747c1a623e391a15db85e.png)
#### 官方快捷配置
如上图，已经提供了许多知名模型厂商，点击，下方即自动配置好
填入 API Key 和 URL
点击"保存"后 "启用"
#### 自定义配置
1. 选择"自定义配置"
2. 填入 API Key 和 URL
3. 点击"保存"后 "启用"
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/a2247309cfd14fd182f98cdc02adb3e0.png#pic_center)

---

## 八、首次启动与登录

### 8.1 启动 Claude Code

1. 打开终端
2. 进入项目目录：
```bash
   cd D:\MyProject
   ```
3. 启动 Claude Code：
```bash
   claude
   ```

### 8.2 信任文件夹

首次在某个文件夹中启动时，Claude Code 会请求权限：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/1a67511bfac9457aaa233275775b96a6.png#pic_center)

请选择 **"1. Yes, I trust this folder"**，然后按 Enter 确认。

### 8.3 登录方式选择

> 使用cc switch则跳过到8.5

Claude Code 支持多种登录方式：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/150e722b6d5742c39157e5c88726615e.png#pic_center)

| 登录方式 | 说明 | 推荐指数 |
|:---:|:---:|:---:|
| 1. Claude account with subscription | 需要 Pro/Max/Team/Enterprise 订阅 | ⭐⭐⭐ |
| 2. Anthropic Console account | API 使用量计费 | ⭐⭐⭐⭐ |
| 3. 3rd-party platform | Amazon Bedrock / Microsoft Foundry / Vertex AI | ⭐⭐⭐⭐⭐ |

> 💡 **国内用户推荐**：使用第三方平台（选项3），或者直接配置中转站 API。

### 8.4 安全提示

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/1b1a40c90cbb47f8b974ebb2d1e410ca.png#pic_center)

Claude Code 会提示你：
1. Claude 可能会犯错，你需要对其操作负责
2. 由于 prompt injection 风险，只在你信任的代码中使用

按 Enter 继续。

### 8.5 主界面

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/c692867437fa4642b7e3930f1b7b666e.png#pic_center)

启动成功后，你会看到：
- **版本信息**：Claude Code v2.1.191
- **当前模型**：Opus 4.8 (1M context)
- **工作目录**：D:\code\MS_GUI
- **提示信息**：Run /init to create a CLAUDE.md file

### 8.6 常见登录问题

#### 问题：Unable to connect to Anthropic services

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/f0c7be2b15c04c2f871df7732a5e7df9.png#pic_center)

**原因**：国内网络无法直接连接 Anthropic 官方服务器

**解决方案**：
1. 配置中转站 API（参考第六节）
2. 或者使用 CC-Switch 工具配置

#### 问题：认证码输入

> 需要谷歌邮箱

![外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传](https://i-blog.csdnimg.cn/direct/afabfb1bb3ad4db7861b653f2672cfa8.png)
如果选择 OAuth 登录，浏览器会打开认证页面，显示认证码。将认证码粘贴到终端中即可完成登录。

---

## 九、常用命令速查表

### 9.1 命令行命令（在终端中使用）

| 命令 | 功能 | 示例 |
|:---|:---|:---|
| `claude` | 启动交互模式（最常用） | `claude` |
| `claude "任务描述"` | 执行一次性任务后退出 | `claude "fix the build error"` |
| `claude -p "查询内容"` | 执行单次查询后立即退出 | `claude -p "explain this function"` |
| `claude -c` | 继续当前目录的最近一次对话 | `claude -c` |
| `claude -r` | 从历史中选择并恢复一次对话 | `claude -r` |
| `claude commit` | 让 Claude 自动生成 Git 提交信息 | `claude commit` |
| `claude update` | 手动更新 Claude Code 到最新版本 | `claude update` |
| `claude --version` | 查看当前安装的版本号 | `claude --version` |

### 9.2 交互模式内命令（在 Claude Code 界面中使用）

| 命令 | 功能 | 示例 |
|:---|:---|:---|
| `/help` | 显示所有可用命令和功能说明 | `/help` |
| `/login` | 登录账号或切换到其他账号 | `/login` |
| `/resume` | 从历史中恢复之前中断的对话 | `/resume` |
| `/clear` | 清除当前对话历史，开始全新会话 | `/clear` |
| `/config` | 查看和修改 Claude Code 配置 | `/config` |
| `/init` | 创建 CLAUDE.md 项目说明文件 | `/init` |
| `exit` 或 `Ctrl+C` | 退出 Claude Code，返回终端 | `exit` |

### 9.3 快捷键

| 快捷键 | 功能 |
|:---:|:---|
| `Ctrl+C` | 中断当前操作 / 退出 |
| `Ctrl+L` | 清屏 |
| `Tab` | 自动补全 |
| `↑` / `↓` | 浏览历史命令 |

---

## 十、实战演示：让 Claude 帮你写代码

### 10.1 使用 Git 功能

Claude Code 让 Git 操作变得像日常对话一样简单：

#### 查看已修改的文件

```
what files have I changed?
```

#### 提交更改（Claude 会自动生成提交信息）

```
commit my changes with a descriptive message
```

#### 创建新分支

```
create a new branch called feature/quickstart
```

#### 查看最近的提交历史

```
show me the last 5 commits
```

#### 协助解决合并冲突

```
help me resolve merge conflicts
```

### 10.2 修复 Bug 或添加功能

#### 添加新功能

用自然语言描述你想要添加的功能，Claude Code 会找到合适的位置并实现它：

```
add input validation to the user registration form
```

#### 修复现有问题

描述 Bug 现象，让 Claude 定位并修复：

```
there's a bug where users can submit empty forms - fix it
```

### 10.3 Claude Code 的问题处理流程

1. **定位相关代码**：在代码库中找到与问题相关的文件和函数
2. **理解上下文**：分析代码逻辑，理解问题的根本原因
3. **实现解决方案**：给出修改建议并展示 diff
4. **运行测试**：如果项目中存在测试，Claude 会尝试运行以验证修复效果

### 10.4 其他常见工作流

#### 代码重构

```
refactor the authentication module to use async/await instead of callbacks
```

#### 编写测试

```
write unit tests for the calculator functions
```

#### 更新文档

```
update the README with installation instructions
```

#### 代码审查

```
review my changes and suggest improvements
```

> 💡 **Claude Code 是你的 AI 结对编程伙伴**。像跟一个有经验的同事交流一样跟它对话——描述你想实现什么，不用拘泥于特定命令格式，用自然语言表达即可，它会帮你找到最佳实现方式。

### 10.5 实际运行效果

![](https://i-blog.csdnimg.cn/direct/d7fe7d4ced0b47718c44a31943c1366b.png)


如图所示，Claude Code 可以：
- 自动分析代码问题
- 展示代码 diff（红色删除，绿色新增）
- 执行构建和测试
- 提供修复建议

---

## 十一、IDE 集成

### 11.1 VS Code 集成（推荐）

VS Code 的集成是目前最成熟的。

#### 安装方式

1. 在项目根目录打开 VS Code 终端
2. 运行 `claude` 命令
3. 工具会自动检测 VS Code 环境并提示安装扩展

#### 快捷键

| 快捷键（Windows/Linux） | 功能 |
|:---:|:---|
| `Ctrl+Esc` | 快速启动 Claude 界面 |
| `Alt+Ctrl+K` | 将文件引用插入到提示中 |

#### 核心功能

- **集成差异查看器**：可以直接在 VS Code 的 diff 视图中审查代码变更
- **自动上下文感知**：自动将选中的代码、打开的文件作为上下文
- **诊断信息共享**：编辑器中的错误和警告会实时共享给 Claude

### 11.2 JetBrains IDEs（Beta）

对于 IntelliJ IDEA、PyCharm、WebStorm 等：

1. 打开 `Settings/Preferences` -> `Plugins`
2. 搜索 "Claude Code"
3. 从 Marketplace 安装

> ⚠️ 目前处于 Beta 测试阶段，可能会有不稳定情况。

---

## 十二、常见问题排查

### Q1：Windows 命令行权限问题

**现象**：运行 claude 命令时提示权限错误

**解决方案**：

```powershell
# 以管理员身份运行 PowerShell
Set-ExecutionPolicy RemoteSigned
```

然后重启 PowerShell 即可。

### Q2：找不到 claude 命令

**原因**：npm 全局安装路径未添加到系统环境变量

**解决方案**：

1. 查找 npm 全局安装路径：
   ```bash
   npm config get prefix
   ```

2. 将获取的路径（如 `C:\Users\你的用户名\AppData\Roaming\npm`）添加到系统环境变量的 Path 中

3. 重启终端验证：
   ```bash
   claude --version
   ```

### Q3：Node.js 版本过低

**解决方案**：

```bash
# Windows：重新下载新版 Node.js 安装包
# macOS：
brew update
brew upgrade node

# Linux：
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
nvm install 20
nvm use 20
```

### Q4：npm 安装速度慢

**解决方案**：

```bash
# 使用淘宝镜像
npm config set registry https://registry.npmmirror.com/

# 或安装时指定
npm install -g @anthropic-ai/claude-code --registry=https://registry.npmmirror.com/
```

### Q5：显示 "offline" 状态

Claude Code 通过连接 Google 来判断网络状态。显示 "offline" 不影响正常使用，只是表明无法连接到 Google。

### Q6：API 报错 "fetch failed"

可能是中转站不稳定导致，建议：

1. 退出 Claude Code（`Ctrl+C`）
2. 重新运行 `claude` 命令
3. 如果问题持续，请稍后再试

### Q7：401 Invalid API key 错误

检查 `ANTHROPIC_API_KEY` 是否正确，确保：
- 没有多余的空格或引号
- API Key 没有过期
- 在平台重新生成 API KEY

---

## 十三、总结与资源

### 13.1 总结

通过以上步骤，你可以在 Windows、macOS 或 Linux 系统上成功安装配置 Claude Code，并使用国内 API 代理服务。整个安装过程大约需要 10-15 分钟，配置完成后即可在终端中享受 AI 编程助手的便利。

**Claude Code 的核心优势**：

```
🎯 可控性 - 所有组件都在自己手里，模型效果不好？换一个
🚀 高效率 - 很多原本需要几小时的工作，现在几分钟就能搞定
🔧 灵活性 - 支持多种模型切换，包括国产模型
🛡️ 安全性 - 完全本地化部署，没有封号风险
```

### 13.2 相关资源

| 资源 | 链接 |
|:---|:---|
| 🌐 Node.js 官网 | https://nodejs.org/ |
| 📥 Git 下载 | https://git-scm.com/download/win |
| 📚 Claude Code 官方文档 | https://docs.anthropic.com/en/docs/claude-code |
| 🔧 CC-Switch 项目 | https://github.com/farion1231/cc-switch |
| 📖 Claude Code GitHub | https://github.com/anthropics/claude-code |
| 🇨🇳 国内中转站示例 | https://xiaomuai.cn/ |

### 13.3 推荐阅读

- [Claude Code 官方文档](https://docs.anthropic.com/en/docs/claude-code) - 完整的功能说明和 API 参考
- [CC-Switch 安装与配置](https://www.yuque.com/xiaomuai/acaeth/zgxaf9r1gs4t7m1u) - 详细的 CC-Switch 使用教程
- [配置 VS Code 访问 Claude Code](https://www.yuque.com/xiaomuai/acaeth/vbbsu3g3dh2dkgpi) - VS Code 集成配置

---

>  🫡 **AI优先**：遇到问题无法解决？先问AI！
> 💬 **交流互动**：如果在安装过程中遇到问题，欢迎在评论区留言讨论！
>
> 👍 **觉得有用？** 点赞 + 收藏 + 关注，获取更多 AI 编程工具教程！

