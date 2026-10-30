# Vulkan 入门：从显式控制到移动端渲染

Vulkan 是 Khronos 制定的跨平台图形与计算 API。它把资源分配、命令记录、同步和资源状态管理更多地交给应用程序：这样能够减少驱动的隐式工作，并支持多线程命令记录，但也要求引擎承担更多正确性与性能责任。

## 1. 先建立一张概念地图

- **Instance / Physical Device / Device**：初始化 API 环境、选择 GPU、创建应用使用的逻辑设备。
- **Queue**：提交图形、计算或传输工作；不同队列不一定对应独立硬件执行单元。
- **Buffer / Image / Device Memory**：数据与存储分离。通常先创建资源，再为资源绑定适合的内存。
- **Descriptor / Pipeline Layout**：描述着色器如何访问纹理、缓冲区等资源。
- **Pipeline**：把着色器及大量固定功能状态组织为可执行的图形或计算配置。
- **Command Buffer**：记录绘制、计算、复制、屏障等命令，然后提交到队列；记录命令不等于立即执行命令。
- **Swapchain**：窗口呈现所使用的一组图像，与离屏渲染目标是不同概念。

## 2. 一帧如何执行

典型窗口渲染流程是：获取交换链图像 → 确保本帧要复用的 CPU/GPU 资源已可用 → 记录命令 → 提交队列 → 呈现图像。实际顺序会因引擎的多帧并行策略而变化。

同步是 Vulkan 最容易出现错误的部分。**提交先后不能替代所有必需的执行与内存依赖**；读取前一次写入的数据时，要保证写入完成，并对读取方可见。

- **Pipeline barrier**：表达管线阶段、访问类型、图像布局等依赖关系；现代代码可优先理解 Synchronization 2 的模型。
- **Semaphore**：协调队列之间的工作，以及交换链获取、渲染和呈现之间的依赖。Timeline semaphore 适合按递增计数值组织 GPU 工作；呈现路径仍需注意接口支持的 semaphore 类型。
- **Fence**：让 CPU 观察一次提交是否完成，例如等待后安全复用命令缓冲区或帧内资源。
- **Image layout**：描述图像在特定用途下的布局要求；不是仅为了性能的提示，正确转换是使用图像的前提之一。

优化同步不是“尽量不加屏障”，而是使用**足够且精确**的阶段、访问范围和资源范围，避免无关工作被迫等待。参见 [Khronos 同步指南](https://docs.vulkan.org/guide/latest/synchronization.html)。

## 3. Render pass、subpass 与 attachment

传统 Vulkan render pass 描述一组 attachment 在渲染过程中的使用方式；subpass 划分其中的阶段，subpass dependency 描述阶段之间的依赖。

例如延迟渲染可以先写 G-buffer，再在后续 subpass 读取同一像素位置的 G-buffer 数据进行光照。Input attachment 适用于这类局部像素读取，不等同于能够任意采样邻近位置的普通纹理。

`loadOp` 决定渲染开始时是否需要已有内容；`storeOp` 决定结束时是否必须保留结果。若某附件只在当前渲染内部使用，不需要在结束后保留，应避免无必要的加载与存储。

现代 Vulkan 也提供 dynamic rendering，让应用不必为常见渲染路径预先创建传统 render pass / framebuffer 对象。但它不会消除同步、布局转换或附件生命周期管理。单独的 dynamic rendering 不能直接替代所有 subpass 技巧；设备支持并启用 `dynamicRenderingLocalRead` 功能时，可通过局部读取与相应屏障表达这类同像素数据依赖。参见 [Khronos Dynamic Rendering Local Read 示例](https://docs.vulkan.org/samples/latest/samples/extensions/dynamic_rendering_local_read/README.html)。

## 4. Tile memory 为什么重要

许多移动 GPU 采用 tile-based rendering：把屏幕划分为小块，在片上存储中处理中间颜色、深度等数据，尽量减少外部显存访问。合适的附件生命周期、局部依赖以及 render pass 组织，可能帮助驱动在片上保留中间结果。

这不是“使用 subpass 就一定保存在 tile memory”的保证。是否合并、溢出或写回受 GPU、驱动、附件大小与数量、MSAA 和渲染结构影响。`transient attachment` 与设备支持的 lazily allocated memory 可以表达某些临时附件的用途，但不能假定所有设备都支持同样的实现。

实用原则：减少不必要的 attachment；不要保存之后不会读取的中间结果；谨慎增加 MRT 数量与 MSAA；尽量避免破坏局部性的跨像素访问与频繁渲染目标切换；最终使用真机 GPU 工具测量带宽和执行时间。参见 [Khronos Tile-Based Rendering 最佳实践](https://docs.vulkan.org/guide/latest/tile_based_rendering_best_practices.html)。

## 5. 在 Unreal Engine 中怎么理解

UE 通常通过 RHI 把引擎渲染命令映射到 Vulkan；开发游戏时不必逐条手写 Vulkan 命令。理解 Vulkan 的价值，在于更好地分析资源生命周期、同步开销、移动端带宽以及渲染路径的差异。

排查时先确认目标平台、UE 版本、启用的 RHI、实际使用的移动或桌面渲染路径，再结合 GPU 捕获观察。不要因为设备支持 Vulkan，就假定桌面渲染器、移动渲染器和所有功能组合具有同样的性能或支持范围。Epic 的移动渲染文档明确说明 Android Vulkan 的移动延迟渲染利用 subpass 访问 tile memory；这是理解 G-buffer 带宽优化的直接切入点。参见 [UE 移动渲染模式](https://dev.epicgames.com/documentation/en-us/unreal-engine/mobile-rendering-and-shading-modes-for-unreal-engine)。

## 6. 建议学习顺序

先跑通验证层与最小三角形，再学习 buffer/image 与 descriptor；随后掌握多帧资源复用和同步；最后深入 render pass、局部读取与移动 GPU 优化。每次只引入一个新概念，并保持 validation layer 无错误。

## 官方资料

- [Vulkan 规范](https://docs.vulkan.org/spec/latest/index.html)
- [Vulkan Guide](https://docs.vulkan.org/guide/latest/)
- [Vulkan Tutorial](https://docs.vulkan.org/tutorial/latest/)
- [Vulkan Samples](https://docs.vulkan.org/samples/latest/)
- [UE Vulkan Mobile Renderer：Android 配置与预览](https://dev.epicgames.com/documentation/en-us/unreal-engine/using-the-android-vulkan-mobile-renderer-in-unreal-engine)
- [UE Mobile Rendering：前向、延迟与 tile memory](https://dev.epicgames.com/documentation/en-us/unreal-engine/mobile-rendering-and-shading-modes-for-unreal-engine)

> 本文为入门概念梳理；各扩展、平台特性和 UE 配置应以对应版本的官方文档及目标设备能力查询为准。
