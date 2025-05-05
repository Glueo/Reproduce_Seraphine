# Seraphine 项目技术栈和架构设计分析

## 技术栈分析

* **编程语言**：项目使用 Python（推荐 3.8）开发。
* **图形界面框架**：采用 PyQt5 构建桌面 UI，同时使用 PyQt-Fluent-Widgets 库实现 Fluent 风格控件。
* **依赖库**：通过 `requirements.txt` 指定了主要 Python 库，包括 `PyQt5`、`PyQt5-sip`、`PyQt-Fluent-Widgets`、`Requests`（HTTP 请求）、`aiohttp`（异步 HTTP）、`qasync`（异步与 Qt 事件循环结合）、`psutil`（系统进程监控）、`pygetwindow`（窗口操作）、`pyperclip`（剪贴板）等。这些库分别用于 GUI、与英雄联盟客户端交互、异步任务处理、窗口/剪贴板控制等功能。
* **辅助工具链**：项目使用 PyInstaller（在 `make.ps1` 脚本中调用）将 Python 程序打包为 Windows 可执行文件，并使用 7-Zip 压缩产物。项目在 GitHub Actions 中配置了 CI/CD 流程（`.github/workflows/build_seraphine.yaml`），自动进行环境部署、依赖安装、打包和发布到 GitHub/Gitee Release。版本管理采用 Git（支持自动同步到码云），无使用 Docker 等容器技术。

## 架构设计分析

Glueo/Seraphine 是一个单体桌面应用，其总体结构划分为**界面层**和**逻辑层**。代码组织形式如下：

* `app/view/` 目录下存放各个界面（如主窗口、查询界面、设置界面等）的实现；`app/components/` 目录存放自定义控件（如英雄图标、信息卡片等）。这些模块负责用户交互和数据显示。
* `app/lol/` 目录下存放与英雄联盟客户端交互的核心逻辑：`connector.py` 负责通过本地 LCU API（League Client Update 本地接口）建立 HTTP 和 WebSocket 连接；`tools.py` 实现具体功能（如自动接受匹配、自动选人/禁用英雄、一键卸下勋章等），调用 `connector` 提供的接口执行操作。
* `app/common/` 包含配置管理（`config.py` 定义全局配置和持久化方案）、日志记录、信号总线等公共模块，供全局使用。`app/resource/` 中存放 UI 资源（图标、样式表、国际化文件），`document/` 目录用于项目说明文档。根目录下的 `main.py` 是程序入口，加载配置和翻译器后创建主窗口；`Seraphine.pro` 文件用于 Qt Linguist 翻译；`make.ps1` 和 `.github` 下的工作流脚本用于自动打包和发布。

**前后端交互**：由于这是单体应用，所谓“前端”和“后端”都在同一个进程中运行。界面层通过信号槽和直接调用触发后端逻辑（如响应按钮点击启动异步任务）。与英雄联盟客户端的交互采用 Riot 提供的本地接口协议——RESTful HTTP 和 WebSocket（即 LCU 本地API）。程序通过 `LolClientConnector` 使用 HTTPS 请求（访问 `https://127.0.0.1:端口/`）以及 WebSocket（`wss://127.0.0.1:端口/`）与游戏客户端通信，获取召唤师信息、对局数据、比赛结果等，并在客户端发生事件（如进入选人阶段、游戏开始等）时接收实时推送。UI 层通过 `qasync` 将 Python 的 `asyncio` 异步调用与 Qt 主事件循环结合，将获取到的数据和事件通过信号传递给界面控件，实现界面刷新或自动操作（如自动接受匹配、自动选择英雄等）。项目**未暴露外部网络 API**，所有通信都是在本地完成的。

**主要模块职责**：

* **界面模块（`app/view`, `app/components`）**：负责窗口和控件的布局、事件处理、数据显示。主窗口（MainWindow）管理导航和各子页面，用于显示查询结果、战绩详情、设置选项等；自定义组件（按钮、图标、信息卡等）封装了通用 UI 元素，增强可重用性。
* **逻辑模块（`app/lol`）**：核心为 `LolClientConnector`，它读取配置后启动后台任务，与本地客户端建立连接、订阅事件、执行 REST API 调用。比如获取当前召唤师数据、对局记录、英雄静态资源路径等；以及操作接口，如接受匹配(`acceptMatchMaking`)、选人(`selectChampion`)、接受交易(`acceptTrade`)等。`tools.py` 则将具体功能组合为异步函数（如自动选人、自动换线、创建训练模式房间等），根据用户配置自动调用 `connector` 接口。
* **配置与公共模块（`app/common`）**：`config.py` 定义了全部可配置项（界面缩放、功能开关等）并实现 JSON 文件读写；`logger.py` 负责日志记录；`signals.py` 定义了跨模块的信号总线，用于模块间解耦通信；其他工具模块提供常用功能（字符串转换、样式等）。
* **部署与发布**：项目通过 GitHub Actions 自动化构建。`.github/workflows/build_seraphine.yaml` 中定义了 Windows 环境（Python 3.8），安装依赖后使用 PyInstaller 打包；生成的 `Seraphine.zip` 附加到 GitHub Release 并同步到 Gitee Release（使用 `sync.py` 脚本）。没有使用 Dockerfile 或 docker-compose，运行时只需按 README 指导安装依赖或使用打包好的可执行程序运行。

以上分析涵盖了 Glueo/Seraphine 项目的所有主要技术组件与架构设计。通过 Python/PyQt 技术栈实现了对英雄联盟客户端 LCU 接口的全面封装，前后端模块分工明确，CI/CD 自动化支持 Windows 平台的持续发布。

**参考资料：** 项目 README、配置文件、依赖声明等。

---

## 🧭 阶段一：夯实基础（预计 2 周）

**目标**：掌握 Python 异步编程、Git 版本控制、虚拟环境管理等基础技能。

### 学习内容：

* **Python 异步编程（asyncio）**：了解 `async`/`await` 语法，掌握异步任务的创建与管理。
* **Git 版本控制**：学习 Git 的基本操作，如克隆、提交、分支管理等。
* **虚拟环境管理**：使用 `venv` 或 `conda` 创建和管理项目的虚拟环境。

### 推荐资源：

* [Python 异步编程官方文档](https://docs.python.org/3/library/asyncio.html)
* [Pro Git 书籍](https://git-scm.com/book/zh/v2)

---

## 🎨 阶段二：掌握 GUI 开发（预计 3 周）

**目标**：学习使用 PyQt5 和 PyQt-Fluent-Widgets 开发现代化桌面应用界面。

### 学习内容：

* **PyQt5 基础**：窗口创建、布局管理、信号与槽机制等。
* **Qt Designer 使用**：通过可视化方式设计界面，并将 `.ui` 文件转换为 Python 代码。
* **PyQt-Fluent-Widgets**：应用 Fluent Design 风格的控件，提升界面美观性。([腾讯云 - 产业智变 云启未来][1])

### 推荐资源：

* [PyQt5 教程（W3Schools 中文网）](https://www.w3schools.cn/pyqt5/)
* [PyQt-Fluent-Widgets 官方文档](https://pyqt-fluent-widgets.readthedocs.io/zh-cn/latest/)

---

## 🔌 阶段三：实现核心功能（预计 3 周）

**目标**：实现与英雄联盟客户端的交互功能，包括自动接受匹配、自动选人等。

### 学习内容：

* **LCU API**：了解英雄联盟客户端的本地 API，掌握其使用方法。
* **异步 HTTP 请求**：使用 `aiohttp` 库发送异步请求，与客户端进行通信。
* **qasync 集成**：将 `asyncio` 协程与 PyQt5 的事件循环集成，实现异步任务与 GUI 的协同工作。

### 推荐资源：

* [qasync 使用教程（CSDN 博客）](https://blog.csdn.net/gitblog_00955/article/details/141741510)

---

## 📦 阶段四：打包与部署（预计 1 周）

**目标**：将开发完成的应用打包成可执行文件，并配置自动化部署流程。([CSDN 博客][2])

### 学习内容：

* **PyInstaller**：使用 PyInstaller 将 Python 应用打包为独立的可执行文件。
* **GitHub Actions**：配置持续集成工作流，实现自动化构建和发布。([CSDN 博客][2], [CSDN 博客][3])

### 推荐资源：

* [PyInstaller 打包教程（CSDN 博客）](https://blog.csdn.net/qq_48979387/article/details/132359366)
* [GitHub Actions 快速入门](https://docs.github.com/zh/actions/writing-workflows/quickstart)
