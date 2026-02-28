# CefGlue 学习指南

> **目标**：深入理解 CefGlue 的架构与实现原理，掌握 Avalonia 集成机制，达到能自主改造的程度。

---

## 一、项目概览

CefGlue 是一个 .NET 绑定库，用于将 Chromium Embedded Framework (CEF) 嵌入到 .NET 应用中。它通过 P/Invoke 调用 CEF 的 C API (`libcef`)，提供了面向对象的 C# 封装，并支持 **Avalonia** 和 **WPF** 两种 UI 框架的集成。

### 核心特点

| 特性 | 说明 |
|------|------|
| **P/Invoke 绑定** | 通过 `DllImport` 直接调用 `libcef.dll` 的 C API |
| **多进程架构** | Browser Process + Renderer Process 分离 |
| **双渲染模式** | Windowed (原生窗口) + OSR (离屏渲染) |
| **跨平台** | Windows / macOS / Linux |
| **UI 框架集成** | Avalonia (跨平台) + WPF (仅 Windows) |

---

## 二、仓库目录结构

```
CefGlue/                          # 仓库根目录
├── CefGlue/                      # 🔵 核心层：CEF C API 的 .NET P/Invoke 绑定
│   ├── CefRuntime.cs             #    CEF 运行时入口（加载、初始化、关闭）
│   ├── Interop/                  #    P/Invoke 声明层（libcef.g.cs 等自动生成）
│   │   ├── libcef.g.cs           #    自动生成的 DllImport 声明
│   │   ├── Base/                 #    基础互操作类型
│   │   ├── Classes.g/            #    自动生成的 CEF 类型结构
│   │   └── Structs/              #    CEF 结构体映射
│   ├── Classes.Handlers/         #    CEF Handler 抽象基类（CefClient, CefApp...）
│   ├── Classes.Proxies/          #    CEF 托管代理类（CefBrowser, CefFrame...）
│   ├── Classes.g/                #    自动生成的类
│   ├── Enums/                    #    CEF 枚举映射
│   ├── Structs/                  #    CEF 结构体映射
│   ├── Platform/                 #    平台特定代码
│   └── Wrapper/                  #    MessageRouter 等高级封装
│
├── CefGlue.Common/               # 🟢 公共适配层：框架无关的浏览器逻辑
│   ├── BaseCefBrowser.cs         #    浏览器控件基类（抽象 UI 框架差异）
│   ├── CommonBrowserAdapter.cs   #    浏览器适配器（Windowed 模式核心）
│   ├── CommonOffscreenBrowserAdapter.cs  # 离屏渲染适配器（OSR 模式核心）
│   ├── CommonCefClient.cs        #    CefClient 实现（事件分发中心）
│   ├── CefRuntimeLoader.cs       #    CEF 运行时加载器
│   ├── BrowserCefApp.cs          #    Browser 进程的 CefApp 实现
│   ├── Platform/                 #    平台抽象接口（IControl, IOffScreenControlHost）
│   ├── InternalHandlers/         #    内部 Handler 实现
│   ├── Handlers/                 #    用户可扩展的 Handler
│   ├── Events/                   #    事件参数定义
│   ├── JavascriptExecution/      #    JS 执行引擎
│   ├── ObjectBinding/            #    .NET <-> JS 对象绑定
│   └── Helpers/                  #    辅助工具类
│
├── CefGlue.Common.Shared/        # 🟡 共享层：Browser/Renderer 进程间共享代码
│   ├── RendererProcessCommunication/  # IPC 消息定义
│   ├── Serialization/            #    序列化工具
│   └── CustomScheme.cs           #    自定义 Scheme 定义
│
├── CefGlue.Avalonia/             # 🟣 Avalonia 集成层
│   ├── AvaloniaCefBrowser.cs     #    Avalonia 浏览器控件入口
│   ├── AvaloniaRenderSurface.cs  #    Avalonia 离屏渲染表面
│   ├── InputExtensions.cs        #    输入事件转换
│   ├── KeyInterop.cs             #    键盘映射
│   ├── CursorsProvider.cs        #    光标处理
│   └── Platform/                 #    平台实现
│       ├── AvaloniaControl.cs    #       Windowed 模式控件
│       ├── AvaloniaOffScreenControlHost.cs  # OSR 模式控件
│       ├── AvaloniaPopup.cs      #       弹出窗口
│       ├── Windows/              #       Windows 特定实现
│       ├── Linux/                #       Linux 特定实现
│       └── MacOS/                #       macOS 特定实现
│
├── CefGlue.WPF/                  # WPF 集成层（结构类似 Avalonia）
├── CefGlue.BrowserProcess/       # 🔴 Renderer 子进程可执行文件
│   ├── Program.cs                #    子进程入口点
│   ├── RendererCefApp.cs         #    Renderer 进程的 CefApp
│   └── ObjectBinding/            #    Renderer 侧对象绑定
│
├── CefGlue.Demo.Avalonia/        # Avalonia 示例应用
├── CefGlue.Demo.WPF/             # WPF 示例应用
├── CefGlue.Interop.Gen/          # Interop 代码生成器
└── CefGlue.Tests/                # 测试项目
```

---

## 三、整体架构图

### 3.1 分层架构

```mermaid
graph TB
    subgraph "Application Layer 应用层"
        APP["Demo.Avalonia / 你的应用"]
    end

    subgraph "UI Framework Layer UI框架层"
        AVA["CefGlue.Avalonia"]
        WPF["CefGlue.WPF"]
    end

    subgraph "Common Adapter Layer 公共适配层"
        COMMON["CefGlue.Common"]
    end

    subgraph "Shared Layer 共享层"
        SHARED["CefGlue.Common.Shared"]
    end

    subgraph "Core Binding Layer 核心绑定层"
        CORE["CefGlue (P/Invoke)"]
    end

    subgraph "Native Layer 原生层"
        CEF["libcef.dll (CEF C API)"]
        CHROMIUM["Chromium"]
    end

    subgraph "Subprocess 子进程"
        BP["CefGlue.BrowserProcess"]
    end

    APP --> AVA
    APP --> WPF
    AVA --> COMMON
    WPF --> COMMON
    COMMON --> CORE
    COMMON --> SHARED
    BP --> SHARED
    BP --> CORE
    CORE --> CEF
    CEF --> CHROMIUM

    style APP fill:#e1f5fe
    style AVA fill:#e8eaf6
    style WPF fill:#e8eaf6
    style COMMON fill:#e8f5e9
    style SHARED fill:#fff9c4
    style CORE fill:#bbdefb
    style CEF fill:#ffcdd2
    style CHROMIUM fill:#ffcdd2
    style BP fill:#ffccbc
```

### 3.2 CEF 多进程架构

```mermaid
graph LR
    subgraph "Browser Process 浏览器进程（你的应用）"
        direction TB
        MAIN["Main Thread<br/>UI + CEF初始化"]
        IO["IO Thread"]
        RENDER_HOST["Render Process Host"]
    end

    subgraph "Renderer Process 渲染进程（子进程）"
        direction TB
        RENDERER["Renderer Thread"]
        V8["V8 JS Engine"]
        DOM["DOM / Blink"]
    end

    subgraph "GPU Process GPU进程"
        GPU["GPU 合成"]
    end

    MAIN <-->|"IPC (CefProcessMessage)"| RENDERER
    RENDER_HOST --> RENDERER
    RENDERER --> V8
    RENDERER --> DOM
    MAIN --> GPU

    style MAIN fill:#bbdefb
    style RENDERER fill:#c8e6c9
    style GPU fill:#fff9c4
```

> **关键点**：CefGlue 的 `CefGlue.BrowserProcess` 项目就是这个 Renderer Process 的实现。主应用（Browser Process）通过设置 `BrowserSubprocessPath` 指定子进程路径。

### 3.3 类继承与组合关系

```mermaid
classDiagram
    class BaseCefBrowser {
        <<abstract>>
        #CommonBrowserAdapter _adapter
        +Address
        +ExecuteJavaScript()
        +EvaluateJavaScript()
        +RegisterJavascriptObject()
        #CreateControl()* IControl
        #CreateOffScreenControlHost()* IOffScreenControlHost
        #CreatePopupHost()* IOffScreenPopupHost
    }

    class AvaloniaCefBrowser {
        +CreateControl() AvaloniaControl
        +CreateOffScreenControlHost() AvaloniaOffScreenControlHost
        +CreatePopupHost() AvaloniaPopup
    }

    class CommonBrowserAdapter {
        <<ICefBrowserHost>>
        -CefBrowser _browser
        -CommonCefClient _cefClient
        -JavascriptExecutionEngine
        -NativeObjectRegistry
        +CreateBrowser()
        +NavigateTo()
        +HandleBrowserCreated()
    }

    class CommonOffscreenBrowserAdapter {
        -IOffScreenControlHost Control
        -IOffScreenPopupHost Popup
        +HandleViewPaint()
        +GetViewRect()
        +ResizeBrowser()
    }

    class CommonCefClient {
        <<CefClient>>
        -CefLifeSpanHandler
        -CefDisplayHandler
        -CefRenderHandler
        -CefLoadHandler
        -MessageDispatcher
        +OnProcessMessageReceived()
    }

    class IControl {
        <<interface>>
        +GetHostViewHandle()
        +InitializeRender()
        +DestroyRender()
        +GotFocus event
        +SizeChanged event
    }

    class IOffScreenControlHost {
        <<interface>>
        +RenderSurface
        +MouseMoved event
        +KeyDown event
        +Focus()
        +StartDrag()
    }

    BaseCefBrowser <|-- AvaloniaCefBrowser
    BaseCefBrowser *-- CommonBrowserAdapter
    CommonBrowserAdapter <|-- CommonOffscreenBrowserAdapter
    CommonBrowserAdapter *-- CommonCefClient
    CommonBrowserAdapter ..> IControl
    CommonOffscreenBrowserAdapter ..> IOffScreenControlHost
    AvaloniaCefBrowser ..> AvaloniaControl : creates
    AvaloniaCefBrowser ..> AvaloniaOffScreenControlHost : creates

    class AvaloniaControl {
        <<IControl>>
    }
    class AvaloniaOffScreenControlHost {
        <<IOffScreenControlHost>>
    }

    IControl <|.. AvaloniaControl
    IOffScreenControlHost <|.. AvaloniaOffScreenControlHost
    AvaloniaControl <|-- AvaloniaOffScreenControlHost
```

---

## 四、核心原理详解

### 4.1 P/Invoke 绑定机制

CefGlue 的核心是通过 P/Invoke 调用 CEF 的 C API。整个调用链：

```mermaid
sequenceDiagram
    participant App as 应用代码
    participant CefRuntime as CefRuntime.cs
    participant Interop as libcef.g.cs (P/Invoke)
    participant Native as libcef.dll (Native)

    App->>CefRuntime: CefRuntime.Initialize(...)
    CefRuntime->>CefRuntime: LoadIfNeed() / CheckVersion()
    CefRuntime->>Interop: libcef.initialize(n_args, n_settings, n_app, ...)
    Note over Interop: [DllImport("libcef")]<br/>static extern int cef_initialize(...)
    Interop->>Native: cef_initialize(...)
    Native-->>Interop: return 0/1
    Interop-->>CefRuntime: success/failure
    CefRuntime-->>App: initialized
```

**关键文件**：
- `CefGlue/Interop/libcef.g.cs` — 自动生成的 `[DllImport]` 声明
- `CefGlue/CefRuntime.cs` — 封装了 CEF 全局函数调用
- `CefGlue/Interop/version.g.cs` — CEF 版本和 API Hash 检查

### 4.2 Handler 模式（回调机制）

CEF 通过 Handler 模式实现回调。CefGlue 将 C 函数指针回调转换为 C# 虚方法重写：

```mermaid
graph TB
    subgraph "CEF Native (C)"
        CEF_CLIENT["cef_client_t (C struct)<br/>内含函数指针"]
    end

    subgraph "CefGlue Core (Interop)"
        CEF_CLIENT_G["cef_client_t (C# struct)<br/>Marshal 映射"]
    end

    subgraph "CefGlue Handler (抽象层)"
        CEF_CLIENT_CS["CefClient (C# abstract class)<br/>virtual GetLifeSpanHandler()<br/>virtual GetDisplayHandler()<br/>..."]
    end

    subgraph "CefGlue.Common (实现层)"
        COMMON_CLIENT["CommonCefClient : CefClient<br/>override GetLifeSpanHandler()<br/>override GetDisplayHandler()<br/>..."]
    end

    CEF_CLIENT <-->|P/Invoke Marshal| CEF_CLIENT_G
    CEF_CLIENT_G <-->|Wrapper 转换| CEF_CLIENT_CS
    CEF_CLIENT_CS <|-- COMMON_CLIENT

    style CEF_CLIENT fill:#ffcdd2
    style CEF_CLIENT_G fill:#ffe0b2
    style CEF_CLIENT_CS fill:#bbdefb
    style COMMON_CLIENT fill:#c8e6c9
```

### 4.3 两种渲染模式

#### Windowed 模式（默认，Windows/Linux）

```mermaid
sequenceDiagram
    participant Avalonia as Avalonia UI
    participant Control as AvaloniaControl
    participant Adapter as CommonBrowserAdapter
    participant CEF as CefBrowserHost

    Avalonia->>Control: SizeChanged event
    Control->>Adapter: HandleControlSizeChanged(size)
    Adapter->>Adapter: CreateBrowser(width, height)
    Adapter->>Control: GetHostViewHandle()
    Control-->>Adapter: 窗口句柄 (IntPtr)
    Adapter->>CEF: CefBrowserHost.CreateBrowser(windowInfo, ...)
    Note over CEF: CEF 创建原生子窗口<br/>嵌入到宿主窗口中
    CEF-->>Adapter: OnBrowserCreated(browser)
    Adapter->>Control: InitializeRender(browserHandle)
    Note over Control: NativeControlHost 嵌入<br/>CEF 原生窗口
```

#### OSR 离屏渲染模式（macOS 必须，跨平台可选）

```mermaid
sequenceDiagram
    participant Avalonia as Avalonia UI
    participant Control as AvaloniaOffScreenControlHost
    participant Adapter as CommonOffscreenBrowserAdapter
    participant CEF as CefBrowserHost
    participant Surface as AvaloniaRenderSurface

    Avalonia->>Control: SizeChanged
    Control->>Adapter: HandleControlSizeChanged(size)
    Adapter->>Surface: Resize(width, height)
    Adapter->>CEF: BrowserHost.WasResized()

    Note over CEF: CEF 渲染到内存 buffer

    CEF->>Adapter: HandleViewPaint(buffer, w, h, dirtyRects)
    Adapter->>Surface: Render(buffer, w, h, dirtyRects)
    Surface->>Surface: Lock WriteableBitmap
    Surface->>Surface: Buffer.MemoryCopy(...)
    Surface->>Avalonia: Image.InvalidateVisual()

    Note over Avalonia: 用户鼠标事件
    Avalonia->>Control: PointerMoved
    Control->>Adapter: HandleMouseMove(cefMouseEvent)
    Adapter->>CEF: SendMouseMoveEvent(...)
```

### 4.4 启动流程

```mermaid
sequenceDiagram
    participant App as Program.cs
    participant Loader as CefRuntimeLoader
    participant RT as CefRuntime
    participant CEF as libcef.dll

    App->>App: AppBuilder.Configure<App>()
    App->>Loader: CefRuntimeLoader.Initialize(settings)
    Note over Loader: 延迟初始化<br/>保存回调

    App->>App: StartWithClassicDesktopLifetime(args)

    Note over App: Avalonia 创建窗口<br/>第一次创建 AvaloniaCefBrowser

    App->>Loader: BaseCefBrowser() → Load()
    Loader->>RT: CefRuntime.Load()
    RT->>CEF: LoadLibrary("libcef.dll")
    RT->>CEF: cef_api_hash(0) → 版本检查

    Loader->>Loader: 查找 BrowserSubprocessPath
    Loader->>RT: CefRuntime.Initialize(args, settings, app)
    RT->>CEF: cef_initialize(...)

    Note over CEF: CEF 启动<br/>创建 UI/IO 线程<br/>注册消息循环

    Loader->>Loader: 注册 SchemeHandler
    Loader->>Loader: 注册 ProcessExit → Shutdown
```

### 4.5 Browser 与 Renderer 进程通信

```mermaid
sequenceDiagram
    participant Browser as Browser Process<br/>(主应用)
    participant IPC as CefProcessMessage<br/>(IPC 通道)
    participant Renderer as Renderer Process<br/>(BrowserProcess.exe)

    Note over Browser: 用户调用<br/>EvaluateJavaScript("1+1")

    Browser->>IPC: SendProcessMessage(PID_RENDERER, msg)
    IPC->>Renderer: OnProcessMessageReceived()
    Renderer->>Renderer: V8 执行 JavaScript
    Renderer->>IPC: SendProcessMessage(PID_BROWSER, result)
    IPC->>Browser: OnProcessMessageReceived()

    Note over Browser: 通过 MessageDispatcher<br/>路由到 JavascriptExecutionEngine

    Browser->>Browser: Task<T> 完成, 返回结果
```

---

## 五、Avalonia 在 CefGlue 中的作用

Avalonia 在 CefGlue 中扮演 **UI 宿主** 的角色。具体职责：

### 5.1 职责划分

```mermaid
graph LR
    subgraph "Avalonia 负责"
        A1["提供窗口/控件容器"]
        A2["捕获输入事件<br/>(鼠标/键盘/拖拽)"]
        A3["显示渲染结果<br/>(WriteableBitmap / NativeControlHost)"]
        A4["管理控件生命周期"]
        A5["提供 UI 线程调度<br/>(Dispatcher)"]
    end

    subgraph "CEF 负责"
        C1["网页加载与渲染"]
        C2["JavaScript 执行"]
        C3["多进程管理"]
        C4["网络请求"]
        C5["生成渲染缓冲区"]
    end

    subgraph "CefGlue.Common 负责"
        G1["事件转换桥接"]
        G2["生命周期管理"]
        G3["JS ↔ .NET 互操作"]
    end

    A2 -->|转换为 CefMouseEvent| G1
    G1 -->|SendMouseMoveEvent| C1
    C5 -->|Paint 回调| G1
    G1 -->|WriteableBitmap| A3
```

### 5.2 关键集成点

| 集成点 | Avalonia 实现类 | 作用 |
|--------|----------------|------|
| 控件容器 | `AvaloniaCefBrowser : BaseCefBrowser` | 浏览器控件入口 |
| 窗口嵌入 (Windowed) | `AvaloniaControl` → `NativeControlHost` | 将 CEF 原生窗口嵌入 Avalonia |
| 离屏渲染 (OSR) | `AvaloniaOffScreenControlHost` + `AvaloniaRenderSurface` | 通过 WriteableBitmap 显示 |
| 输入转换 | `InputExtensions.cs` + `KeyInterop.cs` | Avalonia 事件 → CEF 事件 |
| 光标处理 | `CursorsProvider.cs` | CEF 光标类型 → Avalonia Cursor |
| 弹出窗口 | `AvaloniaPopup` / `ExtendedAvaloniaPopup` | CEF 弹出菜单等 |

---

## 六、学习路线图

### 阶段一：基础理解（1-2天）

```mermaid
graph TD
    S1["1. 运行 Demo.Avalonia<br/>体验完整功能"] --> S2["2. 阅读 Program.cs<br/>理解启动流程"]
    S2 --> S3["3. 阅读 CefRuntimeLoader.cs<br/>理解 CEF 加载配置"]
    S3 --> S4["4. 阅读 AvaloniaCefBrowser.cs<br/>理解继承结构"]
    S4 --> S5["5. 阅读 BaseCefBrowser.cs<br/>理解适配器模式"]

    style S1 fill:#e1f5fe
    style S2 fill:#e1f5fe
    style S3 fill:#e1f5fe
    style S4 fill:#e1f5fe
    style S5 fill:#e1f5fe
```

**阅读顺序**：

1. `CefGlue.Demo.Avalonia/Program.cs` — 应用启动、CEF 初始化
2. `CefGlue.Common/CefRuntimeLoader.cs` — CEF 运行时加载逻辑
3. `CefGlue.Avalonia/AvaloniaCefBrowser.cs` — Avalonia 入口控件
4. `CefGlue.Common/BaseCefBrowser.cs` — 公共浏览器基类
5. `CefGlue.Demo.Avalonia/BrowserView.axaml.cs` — XAML 中使用的方式

### 阶段二：适配层深入（2-3天）

```mermaid
graph TD
    S6["6. CommonBrowserAdapter.cs<br/>Windowed 模式核心"] --> S7["7. CommonOffscreenBrowserAdapter.cs<br/>OSR 模式核心"]
    S7 --> S8["8. CommonCefClient.cs<br/>Handler 分发中心"]
    S8 --> S9["9. ICefBrowserHost.cs<br/>浏览器宿主接口"]
    S9 --> S10["10. Platform/ 目录<br/>IControl / IOffScreenControlHost"]

    style S6 fill:#c8e6c9
    style S7 fill:#c8e6c9
    style S8 fill:#c8e6c9
    style S9 fill:#c8e6c9
    style S10 fill:#c8e6c9
```

**重点理解**：
- `CommonBrowserAdapter.CreateBrowser()` — 浏览器创建流程
- `CommonBrowserAdapter.OnBrowserCreated()` — 浏览器就绪后的初始化
- `CommonOffscreenBrowserAdapter.HandleViewPaint()` — OSR 帧渲染
- `CommonCefClient` — 如何将 CEF 回调路由到 `ICefBrowserHost`

### 阶段三：Avalonia 集成层（2-3天）

```mermaid
graph TD
    S11["11. AvaloniaControl.cs<br/>Windowed 模式实现"] --> S12["12. AvaloniaOffScreenControlHost.cs<br/>OSR 模式 + 输入处理"]
    S12 --> S13["13. AvaloniaRenderSurface.cs<br/>WriteableBitmap 渲染"]
    S13 --> S14["14. InputExtensions.cs<br/>输入事件转换"]
    S14 --> S15["15. KeyInterop.cs<br/>键盘映射表"]

    style S11 fill:#e8eaf6
    style S12 fill:#e8eaf6
    style S13 fill:#e8eaf6
    style S14 fill:#e8eaf6
    style S15 fill:#e8eaf6
```

**重点理解**：
- `AvaloniaControl.GetHostViewHandle()` — 如何获取原生窗口句柄
- `AvaloniaControl.InitializeRender()` — NativeControlHost 嵌入
- `AvaloniaOffScreenControlHost` — 鼠标/键盘/拖拽事件 → CEF 事件
- `AvaloniaRenderSurface` — WriteableBitmap 的 Lock/Copy/Unlock 流程

### 阶段四：核心绑定层（3-5天）

```mermaid
graph TD
    S16["16. CefRuntime.cs<br/>全局 API 入口"] --> S17["17. Interop/libcef.g.cs<br/>P/Invoke 声明"]
    S17 --> S18["18. Classes.Handlers/<br/>CefClient, CefApp 等"]
    S18 --> S19["19. Classes.Proxies/<br/>CefBrowser, CefFrame 等"]
    S19 --> S20["20. CefGlue.Interop.Gen/<br/>代码自动生成器"]

    style S16 fill:#bbdefb
    style S17 fill:#bbdefb
    style S18 fill:#bbdefb
    style S19 fill:#bbdefb
    style S20 fill:#bbdefb
```

**重点理解**：
- `CefRuntime.Initialize()` — 参数转换 (managed → native) 的模式
- `unsafe` / `fixed` / 指针操作 — 与 C API 交互的核心技术
- Handler 的 `ToNative()` 方法 — 如何从 C# 对象创建 C 结构体
- Proxy（如 `CefBrowser`）的 `FromNative()` — 如何从 C 指针创建 C# 对象

### 阶段五：进程间通信（2-3天）

```mermaid
graph TD
    S21["21. BrowserProcess/Program.cs<br/>子进程入口"] --> S22["22. RendererCefApp.cs<br/>Renderer 进程 App"]
    S22 --> S23["23. Common.Shared/<br/>IPC 消息定义"]
    S23 --> S24["24. MessageDispatcher<br/>消息路由"]
    S24 --> S25["25. JavascriptExecution/<br/>JS 执行引擎"]
    S25 --> S26["26. ObjectBinding/<br/>.NET ↔ JS 绑定"]

    style S21 fill:#fff9c4
    style S22 fill:#fff9c4
    style S23 fill:#fff9c4
    style S24 fill:#fff9c4
    style S25 fill:#fff9c4
    style S26 fill:#fff9c4
```

### 阶段六：实操改造（持续）

| 练习任务 | 涉及模块 | 难度 |
|----------|---------|------|
| 添加一个新的浏览器事件（如 favicon changed） | `ICefBrowserHost` + `CommonBrowserAdapter` + `BaseCefBrowser` | ⭐⭐ |
| 为 OSR 模式添加触摸事件支持 | `AvaloniaOffScreenControlHost` + `CommonOffscreenBrowserAdapter` | ⭐⭐⭐ |
| 实现自定义 CefSchemeHandler | `CustomScheme` + `CefResourceHandler` | ⭐⭐ |
| 新增 .NET 对象暴露给 JS 的方法 | `ObjectBinding/` | ⭐⭐⭐ |
| 支持一个新的 UI 框架（如 MAUI） | 参考 `CefGlue.Avalonia` 整层重写 | ⭐⭐⭐⭐⭐ |

---

## 七、关键设计模式

### 7.1 适配器模式 (Adapter)

`BaseCefBrowser` 使用 `CommonBrowserAdapter` 封装所有 CEF 交互，UI 框架层只需提供平台特定的控件实现。

```
BaseCefBrowser (抽象)          CommonBrowserAdapter (逻辑)
   ↓ 继承                          ↑ 组合
AvaloniaCefBrowser ───创建────→ AvaloniaControl (平台实现)
```

### 7.2 工厂方法模式 (Factory Method)

`BaseCefBrowser` 定义了三个抽象工厂方法，由具体 UI 框架子类实现：

```csharp
internal abstract IControl CreateControl();                     // Windowed 模式
internal abstract IOffScreenControlHost CreateOffScreenControlHost(); // OSR 模式
internal abstract IOffScreenPopupHost CreatePopupHost();        // 弹出窗口
```

### 7.3 策略模式 (Strategy)

根据 `WindowlessRenderingEnabled` 设置，选择 `CommonBrowserAdapter` 或 `CommonOffscreenBrowserAdapter`：

```csharp
// BaseCefBrowser 构造函数中
if (CefRuntimeLoader.IsOSREnabled)
    _adapter = new CommonOffscreenBrowserAdapter(...);  // OSR 策略
else
    _adapter = new CommonBrowserAdapter(...);           // Windowed 策略
```

### 7.4 代码生成模式

`CefGlue.Interop.Gen/` 项目自动生成 P/Invoke 绑定代码，保持与 CEF C API 的同步。

---

## 八、调试技巧

### 环境准备

```bash
# 1. 编译整个解决方案
dotnet build Xilium.CefGlue.sln

# 2. 运行 Avalonia Demo
dotnet run --project CefGlue.Demo.Avalonia
```

### 调试要点

1. **断点位置建议**：
   - `CefRuntimeLoader.InternalInitialize()` — CEF 初始化
   - `CommonBrowserAdapter.CreateBrowser()` — 浏览器创建
   - `CommonBrowserAdapter.OnBrowserCreated()` — 浏览器就绪
   - `CommonOffscreenBrowserAdapter.HandleViewPaint()` — 每帧渲染
   - `CommonCefClient.OnProcessMessageReceived()` — IPC 消息

2. **子进程调试**：
   - 在 `CefGlue.BrowserProcess/Program.cs` 的 `catch` 块中，Debug 模式下会调用 `Debugger.Launch()`
   - 也可以设置 `CefSettings.RemoteDebuggingPort = 9222` 启用 Chrome DevTools 远程调试

3. **日志查看**：
   - 设置 `CefSettings.LogSeverity` 和 `CefSettings.LogFile` 查看 CEF 内部日志

---

## 九、常见改造场景指引

### 场景1：拦截/修改网页请求

```
关键类: CefRequestHandler → CefResourceRequestHandler
文件: CefGlue/Classes.Handlers/CefRequestHandler.cs
路径: CommonBrowserAdapter.RequestHandler → CommonCefClient.GetRequestHandler()
```

### 场景2：注入 JavaScript / 与网页通信

```
关键类: JavascriptExecutionEngine, NativeObjectRegistry
文件: CefGlue.Common/JavascriptExecution/
     CefGlue.Common/ObjectBinding/
路径: BaseCefBrowser.ExecuteJavaScript() → adapter → frame.ExecuteJavaScript()
     BaseCefBrowser.RegisterJavascriptObject() → NativeObjectRegistry
```

### 场景3：自定义协议 (Scheme)

```
关键类: CustomScheme, CefSchemeHandlerFactory, CefResourceHandler
文件: CefGlue.Common.Shared/CustomScheme.cs
     CefGlue.Demo.Avalonia/CustomSchemeHandlerFactory.cs
路径: CefRuntimeLoader.Initialize(customSchemes: [...])
```

### 场景4：修改渲染方式

```
关键类: AvaloniaRenderSurface (OSR) / AvaloniaControl (Windowed)
文件: CefGlue.Avalonia/AvaloniaRenderSurface.cs
     CefGlue.Avalonia/Platform/AvaloniaControl.cs
关键方法: AvaloniaRenderSurface.UpdateBitmap() — 可替换为 GPU 纹理等
```

---

## 十、进一步阅读资源

- [CEF 官方文档 (C API)](https://bitbucket.org/chromiumembedded/cef/wiki/GeneralUsage)
- [CEF 架构概述](https://bitbucket.org/chromiumembedded/cef/wiki/Architecture)
- [Avalonia 文档 - NativeControlHost](https://docs.avaloniaui.net/docs/controls/nativecontrolhost)
- [P/Invoke 文档](https://learn.microsoft.com/en-us/dotnet/standard/native-interop/pinvoke)
- 本仓库 `CefGlue.Interop.Gen/` — 理解代码生成流程
