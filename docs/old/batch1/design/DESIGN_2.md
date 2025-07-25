# RimAI 框架 - 技术设计文档 (DESIGN.md)

本文档是 `README.md` 的技术性配套文件，旨在深入阐述实现RimAI框架所需的核心技术、设计模式及其在项目中的具体应用场景。

## 核心设计哲学：框架 (Engine) vs. 内容 (Content)

在深入技术细节之前，必须明确本项目的核心设计哲学：**我们正在构建的是一个“AI叙事引擎”，而非一个单一的、功能固化的故事Mod。**

这个理念借鉴了RimWorld本身的设计：RimWorld有一个强大的游戏引擎(Engine)，以及一个基于该引擎构建的官方核心内容包(Core)。我们的框架也遵循此模式：

*   **框架 (Engine) 的职责**:
    *   **提供API**: 定义清晰、稳定、易于使用的API，如 `JudiciaryAPI`, `ColonyChronicleAPI`, `AiToolRegistryAPI`。
    *   **建立标准**: 定义数据结构和“蓝图”，如 `ToolDef`, `CaseDef`, `IContextProvider` 接口。
    *   **处理通用逻辑**: 负责与LLM的通信、API密钥管理、请求的发送与接收、错误的优雅处理、数据的持久化存储等。
    *   **提供UI容器**: 提供如“终端”这样的可扩展的UI框架，让其他内容可以向其中添加标签页。
    *   **保持中立**: 框架本身不应包含任何硬编码的“故事”或“规则”。它不知道什么是“盗窃”，只知道如何记录一个“案件”。

*   **内容 (Content) 的职责**:
    *   **实现具体叙事**: 提供具体的 `ToolDef` XML文件，定义AI可以使用的工具。
    *   **连接游戏事件**: 通过Harmony Patch，将游戏中的具体事件（如偷窃、斗殴）连接到框架提供的API上，从而创建案件、记录历史。
    *   **定义规则**: 提供具体的 `CaseDef` XML文件，定义什么是“谋杀案”，它的基础严重性是多少。
    *   **填充UI**: 向框架的UI容器中添加具体的交互界面和功能标签页。

我们自己开发的功能，如“司法系统”和“殖民地编年史”，应被视为搭载在该框架上的**第一个官方内容包(Official Core Content)**。在开发过程中，我们必须时刻自省：**“这个功能是属于所有人都需要的基础引擎，还是属于我们这个特定故事包的内容？”**

遵循此原则，将确保RimAI框架具有最大的灵活性和扩展性，方便其他Mod开发者在此基础上构建他们自己的AI驱动的叙事。

## 核心技术选型与应用指南

### 1. Harmony：事件驱动的基石与逻辑注入的“手术刀”

*   **核心思想**: 在不修改游戏源文件的前提下，于运行时动态地在现有C#方法前后或内部注入我们自己的代码。
*   **何时使用**:
    *   **创建事件系统**: 当需要响应游戏中发生的、但游戏本身并未提供API的事件时。这是我们“事件驱动”架构的命脉。
    *   **修改游戏行为**: 当需要改变原版游戏机制以适应框架逻辑时。
    *   **跨Mod兼容**: 当需要与另一个没有提供API的Mod进行交互时。
*   **具体应用场景**:
    *   **殖民地编年史 (阶段二)**:
        *   **Patch `InteractionWorker_SocialFight.Interacted()`**: 在此方法执行后 (Postfix)，捕获斗殴事件的参与者和结果，创建`SocialFightEvent`并调用`ColonyChronicleAPI.LogEvent()`。
        *   **Patch `TradeDeal.TryExecuteDeal()`**: 在交易成功后 (Postfix)，记录交易内容，创建`TradeEvent`。
        *   **Patch `Pawn_HealthTracker.MakeDowned()`**: 在小人倒下时 (Postfix)，记录倒下的原因和状态，创建`PawnDownedEvent`。
    *   **司法系统 (阶段三)**:
        *   **Patch `JobDriver_Steal.MakeNewToils()`**: 在小人执行偷窃工作时 (Postfix)，捕获偷窃行为，自动调用`JudiciaryAPI.FileNewCase()`创建盗窃案件。
        *   **Patch `Pawn.SetExecution()`**: 在处决命令下达时 (Prefix)，检查是否由本框架发起，若是，则广播`OnJudgmentIssued`事件。

### 2. ThingComp：赋予物品“灵魂”的组件系统

*   **核心思想**: 将自定义的数据和逻辑封装成一个独立的“组件”类，然后像配件一样“附加”到游戏内的任何物品（Thing）上。
*   **何时使用**: 当你需要为一个或一类物品添加全新的、可独立存储的状态和持续运行的功能时。
*   **具体应用场景**:
    *   **帝国精神思维传输终端 (阶段一)**:
        *   创建 `Comp_Terminal` 类，并将其附加到终端的 `ThingDef` 上。
        *   **`parent`**: 用来获取终端在地图上的位置、电力状态等。
        *   **`Props`**: 用来读取在XML中配置的终端基础参数（如基础功耗）。
        *   **`PostExposeData()`**: **极其重要**。用于保存和加载此终端**独有**的数据，比如当前正在处理的案件ID，或者与AI的对话上下文。
        *   **`CompTickRare()`**: 用于执行不频繁的后台逻辑，比如检查网络连接状态、更新终端待机动画等。
        *   **`GetGizmos()`**: 用于向玩家提供交互按钮，如“打开AI控制台”。点击此按钮将创建并显示我们的主UI窗口。
        *   **API载体**: `Comp_Terminal` 将包含 `public List<ITab> tabs` 字段和 `RegisterTab(ITab)` 方法，作为其他Mod向终端UI添加标签页的直接API入口。

### 3. GameComponent：框架的“中央服务器”与全局数据中枢

*   **核心思想**: 一个在单个存档的整个生命周期中全局唯一的实例，用于管理独立于任何地图和物品的系统级数据和逻辑。
*   **何时使用**: 当你需要一个地方来存储和管理整个游戏世界范围内的、需要持久化的数据时。
*   **具体应用场景**:
    *   **数据中枢**: 创建 `RimAI_GameComponent`。
        *   **`ExposeData()`**: **框架的记忆核心**。在这里保存和加载所有全局列表，包括：
            *   `List<ChronicleEvent> allChronicleEvents` (完整的编年史)
            *   `List<CaseFile> allCaseFiles` (所有的案件)
            *   `List<ToolDef> registeredAiTools` (所有已注册的AI工具)
            *   `List<IContextProvider> contextProviders` (所有已注册的上下文提供者)
    *   **后台服务**:
        *   **`GameComponentTick()`**: 执行全局后台任务，例如：
            *   定期检查是否有超时的案件需要自动关闭。
            *   定期触发AI对当前殖民地状态进行分析，以驱动叙事。
    *   **API的后端**:
        *   我们设计的 `ColonyChronicleAPI`, `JudiciaryAPI`, `AiToolRegistryAPI` 等静态API类，其本身不存储任何数据。它们所有的方法（如`LogEvent`, `FileNewCase`, `RegisterTool`）都应在内部获取`Current.Game.GetComponent<RimAI_GameComponent>()`的实例，并对其持有的数据进行操作。这是一种兼具易用性（静态API）和健壮性（数据集中于GameComponent）的设计。

### 4. 自定义Def类：创造全新的“概念蓝图”

*   **核心思想**: 通过继承`Def`来创建一种全新的、拥有自定义数据结构和内置逻辑的XML“蓝图”。
*   **何时使用**: 当你需要定义一类全新的、自成体系的概念，而不仅仅是为现有概念附加数据时。
*   **具体应用场景**:
    *   **AI工具定义 (阶段四)**:
        *   创建 `public class ToolDef : Def`。
        *   内部包含字段：`string functionToCall` (工具要调用的C#方法), `List<string> parameters`, `int energyCost`等。
        *   其他Modder可以通过创建`<RimAI.ToolDef>`的XML文件来定义他们自己的AI工具，并被我们的框架自动识别和注册。
    *   **案件类型定义 (阶段三)**:
        *   创建 `public class CaseDef : Def`。
        *   内部可包含字段：`bool isViolent`, `float baseSeverity`等，让案件类型本身就携带了基础元数据。

### 5. DefModExtension：兼容性的“万能贴纸”

*   **核心思想**: 在不改变任何原有`Def`类的情况下，为其附加额外的数据字段。这是保证Mod兼容性的关键。
*   **何时使用**: 当你想让你的框架能够理解和响应其他Mod的内容，或者让其他Mod能方便地为你的框架提供额外信息时。
*   **具体应用场景**:
    *   **让其他Mod适应你的框架**:
        *   一个魔法Mod的作者，想让他的一把“灵魂匕首”在被用于谋杀时，能被你的司法系统识别为“性质极其恶劣”。
        *   他只需在他自己的Mod里，为“灵魂匕首”的`ThingDef`添加一个`<li Class="RimAI.CrimeSeverityExtension">`，并在里面设置`<severityModifier>2.0</severityModifier>`。
        *   你的框架在处理案件时，只需检查物品上是否有这个“贴纸”，并读取其中的值，而完全无需知道“灵魂匕首”是什么。
    *   **为原版内容添加元数据**:
        *   你可以通过XML Patch，为所有原版的毒品`ThingDef`贴上一个`<li Class="RimAI.LegalityExtension">`并设置`<isIllegal>true</isIllegal>`。

### 6. ModSettings：面向玩家的“控制面板”

*   **核心思想**: 在游戏选项中为你的Mod提供一个专属的、可永久保存设置的UI界面。
*   **何时使用**: 当你需要向玩家提供可配置的选项时。
*   **具体应用场景**:
    *   **API密钥管理**: 提供文本框让玩家输入并保存LLM的API Key。
    *   **功能开关**: 提供复选框，允许玩家启用/禁用“编年史”、“司法系统”等主要模块。
    *   **参数微调**: 提供滑块，让玩家调整“AI的创造性（temperature）”、“API调用频率限制”等。
    *   **调试选项**: 提供“开发者模式”开关，开启后输出更详细的日志。

### 7. 调试工具：`DebugActions` & `TweakValue`

*   **核心思想**: 仅在开发者模式下可用的、用于触发逻辑和实时调整数值的“作弊”工具，旨在极大提升开发和测试效率。
*   **何时使用**: 在整个开发周期中持续使用。
*   **具体应用场景**:
    *   **`DebugActions` (一键触发)**:
        *   `[DebugAction("RimAI", "Test: Create Murder Case")]`: 无需等待，立刻创建一个谋杀案以测试UI和处理逻辑。
        *   `[DebugAction("RimAI", "Test: Open Terminal UI")]`: 无需建造建筑，立刻打开终端UI进行布局调试。
        *   `[DebugAction("RimAI", "Test: Trigger AI Analysis")]`: 立刻强制AI对当前局势进行一次分析。
    *   **`TweakValue` (实时微调)**:
        *   `TweakValue.Look(ref someRect.width, "Terminal Left Pane Width")`: 在游戏里实时拖动UI元素的尺寸，完美布局。
        *   `TweakValue.Look(ref aiTemperature, "AI Temperature")`: 实时调整AI参数，立刻观察其回复风格的变化。

### 8. 外部化数据存储

*   **核心思想**: 对于体积庞大、仅供查阅的历史数据，将其存储在与主存档文件关联的外部独立文件中，以避免主存档膨胀。
*   **何时使用**: 当你需要记录大量不直接参与核心游戏逻辑的文本或数据时。
*   **具体应用场景**:
    *   **对话与案件历史 (阶段六)**:
        *   玩家与AI的完整对话记录、已归档的案件详情等，都应写入到一个如`MySaveName.rimai_log.xml`的外部文件中。
        *   终端UI在需要显示历史记录时，才从该文件按需读取，而不是在游戏启动时全部加载到内存。

### 9. 音效系统：`SoundDef` & `SoundStarter`

*   **核心思想**: 通过XML定义声音，通过C#代码在特定时机播放，以增强沉浸感。
*   **何时使用**: 在UI交互和关键事件发生时，提供听觉反馈。
*   **具体应用场景 (阶段六)**:
    *   **`SoundDef`**: 创建`RimAI_KeyPress`, `RimAI_DiskRead_Sustainer`, `RimAI_MindUpload`等声音定义。
    *   **`SoundStarter.PlayOneShotOnCamera()`**: 在UI代码中，检测到键盘输入时播放`RimAI_KeyPress`。
    *   **`Sustainer`**: 在开始调用AI服务时生成一个`RimAI_DiskRead_Sustainer`的循环音效实例，在收到回复后将其`.End()`。
