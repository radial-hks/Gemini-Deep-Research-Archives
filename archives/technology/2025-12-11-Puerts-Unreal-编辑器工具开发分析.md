---
title: Puerts Unreal 编辑器工具开发分析
date: 2025-12-11
category: Technology
tags: [Unreal, Puerts, Node]
author: radial-hks
model: Gemini 3 Pro Preview
source_issue: 
---

# 📝 深度分析报告：Puerts 在 Unreal Engine 编辑器工具开发中的适用性与技术架构

> **⚠️ 摘要/TL;DR**
> 对于追求高效迭代和现代化开发体验的中大型项目团队，引入 Puerts 作为编辑器扩展的高级层是明智的。建议采用混合策略：底层的、极少变动的核心逻辑用 C++ 实现；复杂的、变动频繁的 UI 和业务逻辑用 Puerts (TypeScript) 实现；而简单的批处理任务继续使用 Python。这种组合可以最大化地利用各技术的优势，构建出既强大又灵活的编辑器工具链。

## 💡 原始 Prompt
```text
讨论一个问题 Puerts 是否适合用于unreal 编辑器工具开发，详细分析说明 https://github.com/Tencent/puerts
```

## **1. 引言：Unreal Engine 编辑器扩展的技术演变与 Puerts 的定位**

在现代游戏工业的生产管线中，Unreal Engine (UE) 的编辑器工具开发一直是提升生产效率的核心环节。长期以来，这一领域主要由两种截然不同的技术范式所主导：一是基于 C++ 的原生模块开发，它提供了对引擎底层的完全控制能力和极致的性能，但伴随着极高的开发门槛、冗长的编译时间以及因宕机导致的开发中断风险；二是基于 Blueprint 的可视化脚本，它降低了入门门槛，实现了快速的原型迭代，但在处理复杂数据结构、大规模逻辑复用以及版本控制协作方面显得力不从心。

近年来，随着 Python 在数字内容创作（DCC）领域的标准化，Epic Games 引入了 Python Editor Script Plugin，这标志着编辑器自动化进入了脚本化时代。Python 凭借其庞大的生态系统和简洁的语法，迅速成为资产批量处理和管线自动化的首选 <sup>1</sup>。然而，Python 在 Unreal 中的集成主要基于即时解释模式，主要用于触发式脚本，缺乏构建高性能、交互式、且具有复杂状态管理的常驻型编辑器工具（如复杂的关卡设计面板或实时调试器）的能力。其 UI 开发主要依赖于封装层较浅的 Slate 绑定或较重的 PySide 集成，这在原生编辑器体验的融合度上存在局限 <sup>3</sup>。

在此背景下，腾讯开源的 Puerts (Puer Typescript) 方案引入了第四种范式：基于高性能 JavaScript 运行时（V8/Node.js）的强类型脚本开发环境 <sup>5</sup>。Puerts 不仅仅是一个脚本绑定库，它通过深度的 TypeScript 集成，试图在 C++ 的高性能与脚本语言的灵活性之间寻找新的平衡点。特别是对于编辑器工具开发而言，Puerts 承诺提供 TypeScript 强大的静态类型系统、Node.js 庞大的 npm 生态支持、以及类似于 Web 前端开发的 UI 构建体验（如 React-UMG） <sup>6</sup>。

本报告将从技术架构、开发工作流、UI 系统构建、性能特征以及风险评估等多个维度，详尽分析 Puerts 是否适合以及如何适用于 Unreal Engine 编辑器工具的开发。分析将基于现有的技术文档、社区实践案例以及底层原理进行深入剖析，旨在为技术总监、工具程序员及技术美术提供决策依据。


## **2. Puerts 核心架构解析及其在编辑器环境中的特殊性**

要评估 Puerts 的适用性，首先必须理解其底层的架构设计，特别是它是如何跨越 C++ 与 JavaScript 的边界，以及在编辑器这一特殊运行时环境中的表现。


### **2.1 运行时桥接：V8 与 Node.js 后端的本质区别**

Puerts 的核心优势在于其灵活的后端选择。在运行时游戏逻辑（Runtime）中，为了满足移动端的包体大小和性能限制，通常使用纯净版的 V8 或 QuickJS <sup>5</sup>。然而，在 **编辑器工具开发** 这一特定场景下，Puerts 推荐并深度支持 **Node.js** 后端 <sup>8</sup>。这一架构决策具有深远的影响。

Node.js 后端的引入意味着 Unreal Editor 进程内嵌了一个完整的 Node.js 运行时实例。这不仅仅是执行 JavaScript 代码的能力，而是赋予了编辑器工具直接访问操作系统底层的权限。通过 Node.js 的内置模块（如 fs 文件系统、net 网络模块、child\_process 子进程模块），开发者可以在 Unreal Editor 内部无缝地执行诸如读写任意格式的外部文件、与版本控制系统（Git/Perforce）进行命令行交互、甚至搭建本地 HTTP 服务器来接收外部 DCC 工具（如 Maya 或 Blender）的指令 <sup>5</sup>。

相比之下，Unreal 自带的 Python 虽然也能进行文件操作，但其环境相对封闭，且缺乏 Node.js 那样成熟的异步 I/O 模型。Puerts 的这种设计使得“编辑器即后端”成为可能，极大地扩展了工具的边界。例如，一个编辑器工具可以直接监听本地端口，实时接收策划配置表的推送更新并自动重导入资源，而这一切都在一个高性能的异步事件循环中完成，不会阻塞编辑器的主线程 UI <sup>9</sup>。


### **2.2 静态反射与类型系统的工程化优势**

Unreal Engine 强大的反射系统（Reflection System）是其支持 Blueprints 和 Python 的基础。Puerts 同样利用了这一系统，但它走得更远。Puerts 会扫描引擎的 UClass、UFunction、UProperty 等元数据，并自动生成对应的 TypeScript 类型声明文件（.d.ts） <sup>10</sup>。

这意味着开发者在编写编辑器工具时，可以获得与 C++ 几乎同等的 API 发现能力和类型安全保障。在 VSCode 等 IDE 中，输入 UE. 后，智能提示会列出所有可用的引擎类和函数，并附带参数类型检查。这种 **静态类型检查** 极大地提高了代码的鲁棒性，特别是在维护大型工具集时，可以有效避免 Python 开发中常见的“拼写错误在运行时才崩溃”的问题 <sup>5</sup>。

|           |              |               |                         |                                          |
| --------- | ------------ | ------------- | ----------------------- | ---------------------------------------- |
| **特性维度**  | **C++ 原生开发** | **Python 脚本** | **Puerts (TypeScript)** | **架构洞察**                                 |
| **类型系统**  | 静态（编译期强校验）   | 动态（运行时解释）     | 静态（编译期强校验）              | TypeScript 提供了接近 C++ 的安全感，同时保留了脚本的灵活性。   |
| **运行时环境** | 编译为机器码       | Python 解释器    | V8 JIT / Node.js        | V8 的 JIT 编译使得 JS 在计算密集型任务上远超 Python。     |
| **反射访问**  | 原生访问         | 动态反射调用        | 生成静态绑定/反射               | Puerts 可生成静态 Wrapper 减少反射开销，适合高频调用。      |
| **系统权限**  | 完整 OS 权限     | 受限或需调用库       | 完整 Node.js 权限           | Node.js 后端使得 npm 包（如 Excel 解析、网络库）可直接使用。 |


### **2.3 虚拟机生命周期与编辑器状态管理**

在编辑器环境下，Puerts 创建的 FJsEnv（JavaScript 环境）通常作为插件模块的一部分启动 <sup>12</sup>。理解这个生命周期对于工具开发至关重要。不同于游戏运行时随关卡加载而重置，编辑器环境下的 JS 虚拟机通常是常驻的。这意味着全局变量的状态会一直保留，直到编辑器关闭或显式触发重载。

这种常驻特性是一把双刃剑。一方面，它允许工具维护跨上下文的状态（例如，一个跨多个关卡操作的批量任务进度）；另一方面，它要求开发者必须极其谨慎地管理内存和事件监听器。如果在 JS 中注册了 Unreal 的全局委托（Delegate）而未在重载前清理，可能会导致“野指针”式的崩溃或逻辑重复执行 <sup>13</sup>。Puerts 提供了一套机制来处理模块的加载与卸载，但在编辑器工具开发中，正确管理 JsEnv 的重置是避免内存泄漏的关键挑战 <sup>15</sup>。


## **3. 编辑器 UI 开发范式的革命：从 UMG 到 React**

如果说 Node.js 生态是 Puerts 的左膀，那么其对 UI 开发模式的革新就是右臂。传统的 UE 编辑器 UI 开发主要依赖 Slate（C++），其声明式语法极其繁琐且难以调试；或者使用 Editor Utility Widget (UMG)，虽然可视化但逻辑编写受限于蓝图，难以处理复杂交互。Puerts 通过引入 TypeScript 和 React，彻底改变了这一局面。


### **3.1 React-UMG：现代 Web 开发体验的迁移**

Puerts 社区提供了 ReactUMG 绑定，允许开发者使用 React 框架来构建 Unreal 的 UMG 界面 <sup>7</sup>。这不仅仅是语法的改变，而是**UI 编程思想的迁移**。

在传统的 UMG 开发中，开发者需要手动管理 Widget 的层级、数据绑定和状态同步，随着 UI 复杂度的增加，Widget Blueprint 会迅速变成难以维护的“面条代码”。而在 React-UMG 模式下，UI 被视为状态（State）的函数。开发者只需定义数据状态，React 的 Reconciliation 算法会自动计算出最小的 UMG Widget 更新操作 <sup>7</sup>。

例如，开发一个包含数千个资产的复杂筛选面板：

- **Blueprint 方式**：需要手动创建 ScrollBox，循环添加子 Widget，处理每个子 Widget 的点击事件，并在数据变化时手动清空并重建列表，效率低下且逻辑分散。

- **React-UMG 方式**：定义一个组件 \<AssetList items={assets} />。当 assets 数组变化时，React 自动处理列表项的增删改。这使得构建复杂的、数据驱动的编辑器面板变得异常高效且逻辑清晰 <sup>16</sup>。


### **3.2 BindWidget 模式：混合开发的最佳实践**

对于不希望引入 React 复杂度的团队，Puerts 支持一种更为轻量级的 **Code-Behind** 模式，即利用 TypeScript 的装饰器（Decorators）与 UMG 的 BindWidget 机制相结合 <sup>10</sup>。

这种工作流完美地解耦了视觉设计与逻辑实现：

1. **视觉设计**：美术人员在 UMG 编辑器中拖拽控件，布局界面，设置颜色和动画，完全不需要写代码。只需保证关键控件的命名符合规范（如 btnSubmit）。

2. **逻辑实现**：程序员编写 TypeScript 类继承自 UE.UserWidget。使用 @UE.bindWidget 装饰器声明同名属性 <sup>10</sup>。

3. **自动绑定**：当 Widget 被实例化时，Puerts 会自动将 UMG 中的控件实例挂载到 TypeScript 类的属性上。开发者可以在 TypeScript 中编写 this.btnSubmit.OnClicked.Add(() => {... }) 来处理逻辑 <sup>18</sup>。

这种模式既保留了 UMG 可视化布局的优势，又规避了蓝图编写逻辑的低效，是编辑器工具开发中最务实的选择。它解决了 C++ 开发 UI 时的“盲写”痛点，同时也解决了纯蓝图开发时的“逻辑维护”痛点。


### **3.3 ImGui 集成：即时模式 GUI 的快速原型能力**

除了 UMG，Puerts 还可以通过 EasyEditorPlugin 等扩展集成 **ImGui** <sup>6</sup>。ImGui 是一种即时模式（Immediate Mode）GUI 库，其特点是代码即 UI，不需要预先创建资源文件。

在编辑器工具的早期原型阶段，或开发纯技术向的调试面板时，ImGui 具有无可比拟的优势。开发者可以在 TypeScript 的 Tick 回调中直接写：

```TypeScript
if (ImGui.Button("Build Navigation")) {\
    UE.NavigationSystem.Build(GWorld);\
}
```

几行代码即可生成一个功能按钮。这种极速的迭代能力弥补了 UMG 需要创建资产、编译蓝图的繁琐流程，非常适合程序化生成工具或即时参数调试工具的开发 <sup>6</sup>。


## **4. 资产管线与深度自动化**

编辑器工具的另一个核心职能是资产管线管理。不同于 Python 脚本通常用于离线批处理，Puerts 工具通常作为编辑器的一部分实时运行，这要求更深的系统集成。


### **4.1 扩展 Unreal 核心功能：AssetTools 与 Subsystem**

Puerts 允许 TypeScript 脚本直接调用 UnrealEd 模块中的核心类，如 AssetToolsHelpers，这是操作资产的核心入口 <sup>5</sup>。通过 UE.AssetToolsHelpers.GetAssetTools()，开发者可以获取 IAssetTools 接口，从而实现资产的创建、复制、重命名、缩略图生成等操作 <sup>20</sup>。

更重要的是，Puerts 支持访问和继承 **Editor Subsystem** <sup>5</sup>。UE5 引入的 Subsystem 架构提供了一种模块化的方式来扩展编辑器功能。

- **访问 Subsystem**：通过 GEditor.GetEditorSubsystem(UE.SomeEditorSubsystem)，脚本可以获取现有的编辑器子系统实例，调用其公开方法 <sup>22</sup>。

- **继承 Subsystem**：开发者甚至可以用 TypeScript 编写一个新的 Subsystem 类。由于 Puerts 支持继承 C++ 类并重写虚函数（在一定限制下），这意味着可以用 TS 实现一个随编辑器启动自动初始化的管理模块，监听编辑器事件（如 OnMapOpened, OnAssetImported）并作出响应 <sup>23</sup>。

这种能力使得 Puerts 工具不再是游离于编辑器之外的脚本，而是可以成为编辑器原生功能的一部分。例如，可以编写一个 TS Subsystem，自动监听所有新导入的纹理，根据命名规范自动设置压缩格式和纹理组，而无需人工干预。


### **4.2 接入 Node.js 生态：无限的扩展能力**

由于 Puerts 在编辑器下运行在 Node.js 环境中，这为资产管线带来了革命性的变化。

- **Excel/CSV 处理**：在游戏开发中，配置数据常存储于 Excel。使用 Python 解析 Excel 需要安装 pandas 等库，配置环境较麻烦。而在 Puerts 中，只需 npm install xlsx，即可在工具中直接读取 Excel 文件并生成 UE 的 DataTable 或 DataAsset <sup>5</sup>。

- **网络协作**：通过 Node.js 的 axios 或 request 库，编辑器工具可以轻松对接飞书/钉钉 API、Jira 任务系统或 Jenkins 构建服务器。例如，一个“提交检查工具”可以在美术点击提交时，自动通过 HTTP 请求检查 Jira 任务状态，并在 Slack 上发送构建通知 <sup>6</sup>。

- **文件系统操作**：Node.js 的 fs 模块比 Unreal 的 IFileManager 更符合 Web 开发者的习惯，且功能极其丰富（如流式读写、文件监听），非常适合编写复杂的资源扫描和同步工具 <sup>9</sup>。


## **5. 性能特征、热重载与开发效率**

在评估工具方案时，性能往往被误解。对于编辑器工具，性能不仅仅是代码执行速度，更包括开发者的迭代速度（开发效率）。


### **5.1 执行性能：V8 JIT vs. Python 解释器**

在纯计算性能上，V8 引擎的 JIT（即时编译）技术使得 JavaScript 的执行速度显著优于 Python 的标准解释器 <sup>5</sup>。虽然 Python 在调用 C++ 底层库（如 NumPy）时很快，但在执行纯脚本逻辑（如复杂的循环、图算法、字符串处理）时，V8 具有压倒性优势。

对于涉及大量 UObject 交互的场景（例如遍历场景中 10 万个 Actor 并修改属性），跨语言调用（Interop）的开销成为瓶颈。Puerts 对此进行了深度优化，支持生成静态 C++ Wrapper，这意味着 JS 调用 C++ 函数时，不再需要通过反射查找函数地址，而是直接调用函数指针。基准测试显示，在生成静态绑定的情况下，Puerts 的互操作性能甚至可以接近原生 C#（在 Unity 中的对比数据，可类推至 UE） <sup>25</sup>。这意味着用 Puerts 编写的场景处理工具，在大规模场景下的响应速度会明显优于 Python 脚本 <sup>26</sup>。


### **5.2 热重载（Hot-Reload）：开发效率的倍增器**

C++ 开发编辑器工具最大的痛点是迭代慢。修改一行 UI 布局代码可能需要关闭编辑器、编译、重新打开，耗时数分钟。Unreal 的 Live Coding 虽然有所改善，但不支持头文件修改和构造函数变更 <sup>27</sup>。

Puerts 提供了真正的、基于虚拟机重启的热重载机制 <sup>10</sup>。

- **工作流**：开发者在 VSCode 中修改 .ts 文件 -> 保存 -> TypeScript 编译器（tsc）自动增量编译为 .js -> Puerts 监听到文件变化 -> 自动重置 JS 虚拟机并重新加载脚本。

- **体验**：整个过程通常在 1-2 秒内完成。编辑器界面会立即刷新，逻辑立即生效。这种即时反馈机制极大地鼓励了开发者打磨工具的交互细节，使得开发高易用性的工具成为可能 <sup>6</sup>。


### **5.3 调试体验**

Puerts 支持基于 Chrome DevTools Protocol 的调试 <sup>12</sup>。这意味着开发者可以在 VSCode 中直接对运行在 Unreal Editor 中的 TypeScript 代码打断点、查看变量、单步执行。这种调试体验远优于 Blueprint 的连线调试，也比调试 Python 脚本（通常需要附加远程调试器）更为流畅和稳定 <sup>12</sup>。


## **6. 风险评估与潜在陷阱**

尽管 Puerts 提供了强大的功能，但在生产环境中引入它也面临着不可忽视的风险。


### **6.1 内存泄漏与对象生命周期管理**

这是 Puerts 编辑器开发中最棘手的问题。V8 拥有自己的垃圾回收（GC）机制，而 Unreal 也有自己的 GC。当两者交互时，容易产生“循环引用”或“幽灵引用”。

- **泄漏场景**：如果 JS 对象持有了 Unreal 对象（UObject）的强引用，并且这个 JS 对象挂载在全局变量或长期存活的闭包中，那么即便 Unreal 世界切换或资源卸载，该 UObject 也无法被回收 <sup>14</sup>。在编辑器长时间运行的情况下，这会导致内存持续上涨，最终崩溃 <sup>15</sup>。

- **解决方案**：开发者必须严格遵循内存管理规范，例如在不再需要时手动释放引用（release()），或利用 Puerts 提供的弱引用机制。此外，在热重载时，必须确保清理所有旧的事件监听器，否则重载次数多了内存也会飙升 <sup>15</sup>。


### **6.2 长期维护与“胶水代码”陷阱**

Puerts 是一个社区驱动的开源项目，尽管由腾讯初始开源，但其维护依赖于核心贡献者和社区热情 <sup>7</sup>。

- **引擎升级风险**：Unreal Engine 的 API 在每个大版本（如 5.3 到 5.4）都可能发生破坏性变更。Puerts 需要适配这些变更（例如 UHT 的改变、构建系统的调整）。如果社区更新不及时，项目组可能被迫自己维护 Puerts 插件，这是一项高难度的底层工作 <sup>30</sup>。

- **非官方标准**：相比于 Python（Epic 官方支持），Puerts 属于“非标准”方案。这意味着遇到引擎底层 Bug 时，很难获得 Epic 的官方支持。团队需要权衡引入该技术栈带来的效率提升是否足以抵消维护技术债务的成本 <sup>32</sup>。


### **6.3 打包与分发限制**

编辑器工具通常不需要打包进最终游戏，但需要在团队内部通过版本控制分发。

- **环境依赖**：其他团队成员（如关卡策划）如果要使用这些工具，通常需要他们的机器上也具备 Node.js 环境，或者项目组需要将编译好的 Node.js 二进制文件和插件二进制文件一同提交到仓库。这增加了项目的体积和配置复杂度 <sup>13</sup>。

- **路径长度限制**：Windows 系统下，node\_modules 的嵌套目录极易导致文件路径超过 260 字符限制，引发 Perforce 或 Git 同步失败，甚至导致 UE 打包报错。这需要通过扁平化依赖或开启长路径支持来规避 <sup>34</sup>。


## **7. 结论与建议**

综合以上分析，Puerts 对于 Unreal Engine 编辑器工具开发具有极高的适用性，但其适用范围有明确的边界。

**适用场景（强烈推荐）：**

1. **复杂交互式工具开发**：如关卡设计面板、技能编辑器、剧情分支编辑器等。React-UMG 带来的 UI 开发效率提升是革命性的，远超 Slate 和 Blueprint <sup>7</sup>。

2. **Web 技术栈团队**：如果团队中有熟悉 TypeScript/Web 前端的开发者（如技术美术或全栈工程师），Puerts 可以极大地释放他们的生产力 <sup>5</sup>。

3. **深度管线集成**：需要频繁与外部系统（Jira, Shotgrid, 构建服务器）交互，或需要利用 npm 生态处理复杂数据格式（Excel, JSON, XML）的场景 <sup>6</sup>。

**不适用场景（建议保留 Python/BP）：**

1. **简单的一次性脚本**：如“重命名选中资产”这类简单任务，Python 更加轻量，无需配置环境 <sup>3</sup>。

2. **对稳定性要求极高的核心管线**：如果团队缺乏底层 C++ 维护能力，在核心构建管线中引入非官方插件存在风险。

3. **纯美术团队**：如果工具维护者主要是非技术背景的美术人员，Blueprint Editor Utility 依然是更直观的选择 <sup>35</sup>。

最终建议：

对于追求高效迭代和现代化开发体验的中大型项目团队，引入 Puerts 作为编辑器扩展的高级层是明智的。建议采用混合策略：底层的、极少变动的核心逻辑用 C++ 实现；复杂的、变动频繁的 UI 和业务逻辑用 Puerts (TypeScript) 实现；而简单的批处理任务继续使用 Python。这种组合可以最大化地利用各技术的优势，构建出既强大又灵活的编辑器工具链。


### **数据引用汇总表**

|                  |               |                                         |
| ---------------- | ------------- | --------------------------------------- |
| **核心论点**         | **数据来源**      | **关键信息摘要**                              |
| **Node.js 后端优势** | <sup>5</sup>  | 编辑器模式下使用 Node.js 后端，支持 npm 生态与完整 OS 访问。 |
| **类型系统价值**       | <sup>10</sup> | 自动生成 .d.ts 定义，提供静态类型检查与 IDE 智能提示。       |
| **UI 开发模式**      | <sup>7</sup>  | 支持 React-UMG 声明式开发与 BindWidget 混合开发模式。  |
| **热重载机制**        | <sup>10</sup> | 基于虚拟机重置的秒级热重载，大幅提升迭代效率。                 |
| **性能优势**         | <sup>25</sup> | V8 JIT 执行效率远超 Python，静态绑定减少互操作开销。       |
| **内存风险**         | <sup>14</sup> | JS 与 UE GC 交互可能导致内存泄漏，需谨慎管理对象引用。        |
| **维护风险**         | <sup>30</sup> | 社区维护模式可能导致引擎升级时的兼容性滞后与“胶水代码”负担。         |

***

**注**：本报告基于截至 2025 年的技术文档与社区反馈撰写，随着 Unreal Engine 和 Puerts 的版本迭代，部分技术细节（如具体的 API 绑定方式）可能会发生演变。


#### **Works cited**

1. Scripting and Automating the Unreal Editor | Unreal Engine 5.7 Documentation, accessed December 11, 2025, <https://dev.epicgames.com/documentation/en-us/unreal-engine/scripting-and-automating-the-unreal-editor>

2. Understanding and Teaching Python for Unreal Engine | Unreal Educator Livestream, accessed December 11, 2025, <https://www.youtube.com/watch?v=D-mDLwNawVU>

3. Scripting the Unreal Editor Using Python | Unreal Engine 5.7 Documentation, accessed December 11, 2025, <https://dev.epicgames.com/documentation/en-us/unreal-engine/scripting-the-unreal-editor-using-python>

4. What exactly can I do with Python in Unreal Editor? : r/unrealengine - Reddit, accessed December 11, 2025, <https://www.reddit.com/r/unrealengine/comments/169fm2e/what_exactly_can_i_do_with_python_in_unreal_editor/>

5. Tencent/puerts: PUER(普洱) Typescript. Let's write your game in UE or Unity with TypeScript., accessed December 11, 2025, <https://github.com/Tencent/puerts>

6. puerts/EasyEditorPlugin: Write your editor plugin with TypeScript in unreal engine. - GitHub, accessed December 11, 2025, <https://github.com/puerts/EasyEditorPlugin>

7. puerts - GitHub, accessed December 11, 2025, <https://github.com/puerts>

8. install - PUER Typescript, accessed December 11, 2025, <https://puerts.github.io/en/docs/puerts/unreal/install/>

9. dawnarc/PuertsGame: PuerTS example to demonstrate shooting and RPC in Unreal Engine 4. - GitHub, accessed December 11, 2025, <https://github.com/dawnarc/PuertsGame>

10. Automatic Binding Mode - PUER Typescript, accessed December 11, 2025, <https://puerts.github.io/en/docs/puerts/unreal/uclass_extends/>

11. Generating d.ts - PUER Typescript, accessed December 11, 2025, <https://puerts.github.io/en/docs/puerts/unity/knowjs/typescript/>

12. Debugging - PUER Typescript, accessed December 11, 2025, <https://puerts.github.io/en/docs/puerts/unreal/vscode_debug/>

13. FAQ | PUER Typescript, accessed December 11, 2025, <https://puerts.github.io/en/docs/puerts/unreal/faq/>

14. The easiest way to memory leak in Unreal Engine | by Igor Karatayev | Medium, accessed December 11, 2025, <https://medium.com/@igor.karatayev/the-easiest-way-to-memory-leak-in-unreal-engine-9e2aeaa7b104>

15. Memory Leaks! What's the best way to diagnose and find memory leaks in the editor? : r/unrealengine - Reddit, accessed December 11, 2025, <https://www.reddit.com/r/unrealengine/comments/14pru6t/memory_leaks_whats_the_best_way_to_diagnose_and/>

16. puerts/ReactUMG - GitHub, accessed December 11, 2025, <https://github.com/puerts/ReactUMG>

17. Binding actor components like widgets? - C++ - Unreal Engine Forums, accessed December 11, 2025, <https://forums.unrealengine.com/t/binding-actor-components-like-widgets/723058>

18. How to bind text in a UserWidget in Cpp : r/unrealengine - Reddit, accessed December 11, 2025, <https://www.reddit.com/r/unrealengine/comments/gmxugw/how_to_bind_text_in_a_userwidget_in_cpp/>

19. BindWidget Null when using Create Widget - C++ - Unreal Engine Forum, accessed December 11, 2025, <https://forums.unrealengine.com/t/bindwidget-null-when-using-create-widget/1246603>

20. Official Puerts Demos? - PUER Typescript, accessed December 11, 2025, <https://puerts.github.io/en/docs/puerts/unreal/demos/>

21. Puerts - Unreal Engine User Manual - PUER Typescript, accessed December 11, 2025, <https://puerts.github.io/en/docs/puerts/unreal/manual/>

22. Editor Subsystems - hzFishy's Game Dev Notes, accessed December 11, 2025, <https://notes.hzfishy.fr/Unreal-Engine/Editor-Only/Types/Editor-Subsystems>

23. Programming Subsystems in Unreal Engine - Epic Games Developers, accessed December 11, 2025, <https://dev.epicgames.com/documentation/en-us/unreal-engine/programming-subsystems-in-unreal-engine>

24. What is a Subsystem? What should it be used for? - #2 by Jay2645 - Unreal Engine Forums, accessed December 11, 2025, <https://forums.unrealengine.com/t/what-is-a-subsystem-what-should-it-be-used-for/446506/2>

25. Performance of PuerTS | PUER Typescript, accessed December 11, 2025, <https://puerts.github.io/en/docs/puerts/unity/performance/>

26. High speed bridge between unreal 5.5 and python : r/unrealengine - Reddit, accessed December 11, 2025, <https://www.reddit.com/r/unrealengine/comments/1o07739/high_speed_bridge_between_unreal_55_and_python/>

27. C++ Hot Reload not working in UE5 (but works in UE4) : r/unrealengine - Reddit, accessed December 11, 2025, <https://www.reddit.com/r/unrealengine/comments/tz121h/c_hot_reload_not_working_in_ue5_but_works_in_ue4/>

28. Hot-reload doesn't work and there are some errors in empty project - Unreal Engine Forum, accessed December 11, 2025, <https://forums.unrealengine.com/t/hot-reload-doesnt-work-and-there-are-some-errors-in-empty-project/554815>

29. V8 Memory leak when using optional chaining in script - Stack Overflow, accessed December 11, 2025, <https://stackoverflow.com/questions/70157768/v8-memory-leak-when-using-optional-chaining-in-script>

30. \[UE] Bug: Cannot Build UE5.7 · Issue #2237 · Tencent/puerts - GitHub, accessed December 11, 2025, <https://github.com/Tencent/puerts/issues/2237>

31. Upgrade Guide - PUER Typescript, accessed December 11, 2025, <https://puerts.github.io/en/docs/puerts/unity/other/upgrade/>

32. Avoiding the 'Being Glue' Trap: Recognizing and Rebalancing Non-Promotable Work, accessed December 11, 2025, <https://visualstudiomagazine.com/articles/2025/12/10/avoiding-the-glue-trap-recognizing-and-rebalancing-non-promotable-work.aspx>

33. \[UE] suggestion: PuerTS as Engine Plugin · Issue #1608 - GitHub, accessed December 11, 2025, <https://github.com/Tencent/puerts/issues/1608>

34. \[FEATURE REQUEST] Extension/Removal of Path Length Limitation When Cooking, accessed December 11, 2025, <https://forums.unrealengine.com/t/feature-request-extension-removal-of-path-length-limitation-when-cooking/355273>

35. Editor Utility Widgets Feedback - Page 2 - Editor Scripting - Unreal Engine Forum, accessed December 11, 2025, <https://forums.unrealengine.com/t/editor-utility-widgets-feedback/124680?page=2>
