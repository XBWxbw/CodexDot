# VK_QCOM_tile_memory_heap：原理、使用流程与 UE 集成边界

- 原讨论日期：2026-10-26
- 归档日期：2026-10-28

> 历史讨论归档：保留原报告的技术结论、公开参考链接与证据限制。设备、驱动、引擎支持及资料访问状态均以原讨论日期为准，本次未重新核验；不代表新的实测、完整源码审计或对所有录像的完整观看。

先给结论：Tile Memory Heap 是“由应用显式管理的一小块 GPU 片上高速内存”，不是自动优化所有显存的开关。这里具体指高通 Vulkan 扩展 VK_QCOM_tile_memory_heap。

它的价值是让合适的 Image／Buffer 在多个 Render／Compute Pass 之间保留在片上，少往返系统内存；代价是应用必须管理容量、资源寿命、别名和提交边界。

截至 2026-10-26，官方明确列出的支持平台包括 Snapdragon 8 Elite Gen 5、Snapdragon X2 Elite／X2 Plus。UE5 已有实际定制集成案例，但不能据此认为标准 UE5 项目勾一个选项就能启用；UE4 也不能直接套用这些案例。

## 一、与普通 Tile-based Rendering 有什么不同？

普通分块渲染的典型做法，是在处理一个屏幕 tile 时，把颜色、深度等附件留在片上；处理完后，按需要保存结果。Subpass 等机制能利用这种局部性。

Tile Memory Heap 则把部分片上内存作为 VkMemoryHeap 暴露出来，由应用决定哪些资源占用它。它不只服务于一个 Render Pass 的附件，也可以让后续 Compute Pass、采样读取继续使用其中的数据。[高通技术说明](https://www.qualcomm.com/developer/blog/2026/05/high-performance-memory-extension-optimize-memory)

因此有三个区别：
- 普通缓存：主要由硬件／驱动决定哪些数据留下。
- 普通 subpass 优化：围绕特定渲染结构减少附件落地。
- Tile Memory Heap：应用显式选择资源、布局和保留范围。

不要把它理解成“所有 Render Pass 自动融合”，也不要与另一个扩展 VK_QCOM_tile_shading 混为一谈。两者可以配合，但不是同一个功能。[扩展设计说明](https://docs.vulkan.org/features/latest/features/proposals/VK_QCOM_tile_memory_heap.html)

## 二、最重要的限制：跨 Pass，不等于任意跨提交、跨帧

VkImage 对象可以长期存在，但里面的数据不是长期有效。

默认只保证同一个 command-buffer submission batch 内的内容有效。queueSubmitBoundary 为 true 时，才扩大到同一次 queue submit 调用内的多个 batch。

例如：一个 vkQueueSubmit2 调用提交两个 VkSubmitInfo2。如果 queueSubmitBoundary 为 false，不能因为“只调用了一次提交函数”就假定前后两个 batch 能共享保留下来的内容。

同样，Fence／Semaphore 只能解决执行和同步问题，不能让已经超出有效范围的数据复活。跨 Graphics／Async Compute 队列，也不能把这个 heap 当成普通共享显存：规范中不同队列有各自的内容语义。[Vulkan 内存规范](https://docs.vulkan.org/spec/latest/chapters/memory.html)

对引擎的实际影响：
- “这张纹理本帧才使用”还不够，必须确认它的整个生产与消费链没有越过有效边界。
- TAA／TSR 历史、需要保留到下一帧的结果，不能直接靠这块内存保存。
- 必须留下的数据，需要在失效前写到普通内存资源；具体复制／着色写出方式须符合该资源的 usage 和设备能力。

## 三、什么资源适合放进去？

建议按三个条件筛选：寿命短、反复使用、占用小。

优先候选：
- 多个后处理阶段都要读的中间颜色、深度、运动矢量。
- 部分 GBuffer，而不是默认把所有 GBuffer 都放进去。
- Compute 生成、随后由光照阶段读取的短命 Buffer。
- 小尺寸但反复读写的中间结果。

不应优先选择：
- 大型静态材质贴图。
- 需要跨帧保留的历史资源。
- 本来已被 subpass 很好地留在片上、几乎没有外部往返的资源。
- 只有一次便宜访问、却占用大量片上空间的资源。

这些是选型建议，不是所有资源都获得相同收益的保证。高通公开 sample 就采用了 clustered forward lighting，说明它并不是 Deferred 专属功能。[官方 Tile Memory Sample](https://github.com/SnapdragonGameStudios/adreno-gpu-vulkan-code-sample-framework/tree/main/samples/tile_memory)

## 四、容量不是“一个 tile 很小，所以整张纹理一定放得下”

实际资源绑定到 heap 后，需要满足驱动返回的 tile-memory 大小和对齐要求。不能只计算一个屏幕 tile 的附件大小，就认为整张全分辨率图像免费驻留。

做个未考虑压缩、布局和对齐的量级估算：
- 1920×1080、RGBA8：约 7.91 MiB。
- 同分辨率 RGBA16F：约 15.82 MiB。
- 三张 4 字节／像素的图：约 23.73 MiB。

所以 NTE 提到 SceneColor、Depth、Velocity，并不意味着任意分辨率下三张全尺寸纹理都能同时放入。还取决于格式、实际渲染分辨率、生命周期复用和驱动需求。

高通演讲举过 8 MB／14 MB 等 heap 容量，但这不是所有设备统一的可用值。宣传的 HPM 总容量也不能直接代替 Vulkan 查询结果。[高通 Vulkanised 2026 演讲，第 5 页](https://vulkan.org/user/pages/09.events/vulkanised-2026/1645-Srihari-Alla-Qualcomm.pptx.pdf)

## 五、Vulkan 如何使用？

这不是 Shader 里加个关键字，主要改动在资源管理和命令提交层。

### 1）检查并启用能力

- 用 vkEnumerateDeviceExtensionProperties 检查 VK_QCOM_tile_memory_heap。
- 通过 vkGetPhysicalDeviceFeatures2 查询 VkPhysicalDeviceTileMemoryHeapFeaturesQCOM.tileMemoryHeap。
- 创建 VkDevice 时，同时启用扩展及该 feature。
- 加载 vkCmdBindTileMemoryQCOM。

仅仅新版 SDK 里有这些定义，不代表手机驱动支持。[Feature 查询与启用规则](https://docs.vulkan.org/refpages/latest/refpages/source/VkPhysicalDeviceTileMemoryHeapFeaturesQCOM.html)

### 2）查询 heap 与行为

通过内存属性找到带 VK_MEMORY_HEAP_TILE_MEMORY_BIT_QCOM 标志的 heap，并确定对应 memory type、heap 容量。

通过 VkPhysicalDeviceTileMemoryHeapPropertiesQCOM 查询：
- queueSubmitBoundary：内容有效范围。
- tileBufferTransfers：放在其中的 Buffer 是否支持 transfer 操作。

不要把 Buffer 的 transfer 能力直接套到 Image 上。[属性说明](https://docs.vulkan.org/refpages/latest/refpages/source/VkPhysicalDeviceTileMemoryHeapPropertiesQCOM.html)

### 3）创建候选资源

Image 增加 VK_IMAGE_USAGE_TILE_MEMORY_BIT_QCOM；Buffer 增加 VK_BUFFER_USAGE_TILE_MEMORY_BIT_QCOM。同时仍需声明实际用途，例如 sampled、storage 或 attachment。

不要假定任意格式、MSAA、mip 链、数组层都支持。先查询具体格式／usage 组合；最初验证可从简单的单采样、单 mip、单层 2D Image 开始。Resolve attachment 不能直接绑定到 tile memory。[资源限制](https://docs.vulkan.org/features/latest/features/proposals/VK_QCOM_tile_memory_heap.html)

### 4）查询专用内存需求，再分配与绑定

在 vkGetImageMemoryRequirements2／vkGetBufferMemoryRequirements2 的输出链中加入 VkTileMemoryRequirementsQCOM。

必须使用它返回的 size、alignment，不能照搬普通显存需求。若返回 0，表示该资源不能绑定到 tile memory，应走普通内存回退路径。

然后从匹配的 tile memory type 分配 VkDeviceMemory，并用 vkBindImageMemory／vkBindBufferMemory 绑定资源及合适的偏移。[专用内存需求](https://docs.vulkan.org/refpages/latest/refpages/source/VkTileMemoryRequirementsQCOM.html)

这里有一个容易写错分配器的点：多次 vkAllocateMemory 不意味着拿到互不重叠的独立片上空间。Tile heap 的这些分配从同一 heap 起点开始，会发生别名。必须自己规划哪些资源同时存活、哪些偏移可以复用。[内存分配与别名规则](https://docs.vulkan.org/spec/latest/chapters/memory.html)

### 5）执行前，还要绑定到 Command Buffer

只把内存绑定给 Image／Buffer 不够。要在 Render Pass 外调用 vkCmdBindTileMemoryQCOM，把对应 VkDeviceMemory 绑定到命令缓冲。

之后才执行生产者 Pass、必要的 barrier／layout transition、消费者 Pass。换命令缓冲时要正确重新绑定；Render Pass 内的 secondary command buffer 还需处理继承信息。

尤其不要在中途解除／缩小绑定范围后执行其他工作，还期望之前范围里的数据一直存在。[命令绑定接口](https://docs.vulkan.org/refpages/latest/refpages/source/vkCmdBindTileMemoryQCOM.html)

### 6）最后一个消费者结束后才复用

例如资源 A 用于前半帧，资源 B 用于后半帧，两者寿命不重叠，才适合复用同一片空间。别名复用仍要遵守 Vulkan 同步规则。

工程上建议先实现一个小资源、单队列、单 batch 的正确版本，再扩展资源数量；不要一开始就改整套 Deferred 管线。

## 六、哪些设备和平台支持？

截至原讨论日期，明确官方资料确认的芯片平台是：
- Snapdragon 8 Elite Gen 5。
- Snapdragon X2 Elite。
- Snapdragon X2 Plus。

这是高通 Vulkanised 2026 演讲直接列出的范围，不是按 GPU 代际推测的。[官方支持列表，第 10 页](https://vulkan.org/user/pages/09.events/vulkanised-2026/1645-Srihari-Alla-Qualcomm.pptx.pdf)

但这不是完整的“手机型号＋系统版本＋驱动版本”认证表。实际部署要分三层看：

1. 硬件平台支持。
2. 当前 OEM 系统里的 Vulkan 驱动暴露扩展和 feature。
3. 具体资源的格式、尺寸、usage 和容量需求通过查询。

因此不能看到“骁龙旗舰”“Vulkan 1.3／1.4”“有 GMEM”就直接打开，也不能把上面的列表理解为所有更新型号均不支持。未核实的型号不列为确定支持或确定不支持。

平台/API 的区别：
- Android＋支持该扩展的 Adreno Vulkan 驱动：明确的实践路径，高通有 Android Vulkan 示例。
- Snapdragon X2 的 PC 平台：官方列出芯片支持，但仍须验证实际操作系统和 Vulkan 驱动，不能理解成所有 Windows PC 都支持。
- DirectX 12、OpenGL ES、Metal：不能直接调用这个 Vulkan 扩展。
- Mali／Immortalis、Apple GPU：不要把本扩展作为它们的通用能力；各自的片上存储机制要走对应 API／架构路径。

高通 QHWI 虽然是跨 API 抽象层，其 tile-memory 示例仍明确标为 Android Only／Vulkan Backend，不能从框架跨平台反推每个扩展都跨平台。[官方示例要求](https://github.com/qualcomm/qhwi/blob/main/examples/qcom_tile_memory_heap/README.md)

## 七、UE4、UE5 的支持状态（截至 2026-10-26）

需要把“能定制接入”和“标准版开箱即用”分开。

UE4／UE4.27：
原讨论时未找到 Epic 或 Qualcomm 公开的标准集成说明、最低版本表或启用开关。工程排期应按自行移植 Vulkan RHI 功能来评估，而不是按配置项来评估。

这不是说 UE4 原理上做不到。它可以通过源码改造接入，但不能因为手机驱动有扩展，就认为旧引擎会自动使用。

标准 UE5：
原讨论时同样未核实到 Epic 明确提供的通用开关、最低版本或默认启用承诺。当时没有检查具体 UE 分支，也没有访问授权引擎源码，因此不能得出“所有 UE5 版本都没有实现”的绝对结论。

定制 UE5：
有官方确认案例。Qualcomm 的 GDC 2026 回顾提到 Infinity Nikki 的 UE5 参考实现，以及 NTE 对 Tile Memory Heap 的集成。准确说法是“已有 UE5 项目级／参考实现”，而非“UE5 默认就支持”。[Qualcomm 官方 GDC 回顾](https://www.qualcomm.com/developer/blog/2026/05/snapdragon-game-studios-developer-forum-gdc-2026-recap)

公开插件：
原讨论日期所检查的 Qualcomm 公开 UE 插件目录没有单独列出 Tile Memory Heap 插件。不要把 SGSR 或 ANF 插件所支持的 UE 版本，套成这个功能的支持版本。[Qualcomm UE 插件目录](https://github.com/SnapdragonGameStudios/snapdragon-game-plugins-for-unreal-engine/blob/main/README.md)

还有个关键区别：
UE Mobile Deferred 已经利用 Vulkan subpass 来优化片上附件访问，但“支持 Mobile Deferred／Memoryless／subpass”不等于“实现了 VK_QCOM_tile_memory_heap”。[Epic Mobile Deferred 文档](https://dev.epicgames.com/documentation/en-us/unreal-engine/using-the-mobile-deferred-shading-mode-in-unreal-engine)

## 八、如果在 UE 项目里落地，实际要改哪些地方？

下面是依据 API 约束给出的工程拆解，不是某个 UE 版本已经提供的接口清单。

- Vulkan 初始化：查询、启用扩展和 feature，加载入口。
- RHI 资源创建：为选定 Image／Buffer 添加 tile-memory usage，并保留普通内存路径。
- Vulkan 分配器：识别专用 heap、专用 size／alignment，正确处理同堆别名。
- 命令缓冲管理：在正确位置绑定 tile memory，处理 secondary command buffer。
- RDG／资源寿命管理：知道最后一次读取发生在哪里，什么时候可以复用空间。
- 提交与队列调度：不能把有效性理解成“同一帧”；Async Compute、额外 submit、读回等都要检查。
- 设备与画质回退：容量不足、格式不支持、分辨率改变时，能切回原路径。

检查目标分支时，可以搜索 VK_QCOM_tile_memory_heap、VkTileMemoryRequirementsQCOM、vkCmdBindTileMemoryQCOM 等名称，但只在 Vulkan 头文件中找到它们不算集成成功。必须看到实际查询、分配、绑定和使用链。

## 九、值得做吗？如何验证收益？

建议先做一条窄路径：
选一个多次读取、当前确有外部流量、寿命落在同一 batch 的中间资源，完成 A/B 验证，再决定是否扩大改造。

测试至少保持相同画质、渲染分辨率和真实渲染帧率，比较：
- 外部读写字节／帧，以及 GB/s。
- 相关 Pass 时间和整帧 GPU 时间。
- 持续运行功率、温度和频率。
- 新增拷贝、barrier、队列串行化造成的成本。

不要预设“放进片上就一定更快”。应用占用更多片上空间，可能挤压驱动原本用于分块渲染的空间；为了保留数据而改动提交方式，也可能损失并行度。降低外部带宽并不自动等于同比降低整机功率。

从 ALU／带宽的成本分工看：这个功能主要改变的是数据放在哪里、怎样在 Pass 间流动，不是减少 Shader 数学指令。如果场景主要受 ALU 限制，可能功耗或带宽改善，但帧时间改善有限。

一句话：把它当成需要 RHI、分配器和帧图共同配合的“短生命周期片上资源管理方案”，而不是 UE 的画质选项。要判断项目接入成本，最关键的输入是 UE 精确版本／分支、目标 SoC 与驱动，以及准备迁移的资源链。
