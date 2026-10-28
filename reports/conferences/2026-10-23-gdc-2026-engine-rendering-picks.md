# GDC 2026：引擎、渲染与性能分享选读

- 原讨论日期：2026-10-23
- 归档日期：2026-10-28

> 历史讨论归档：保留原报告的技术结论、公开参考链接与证据限制。设备、驱动、引擎支持及资料访问状态均以原讨论日期为准，本次未重新核验；不代表新的实测、完整源码审计或对所有录像的完整观看。

本选读依据 GDC 2026 官方议程、厂商回顾和公开幻灯片，围绕移动端功耗、带宽、Forward／Deferred、GI 和 VRS，优先整理以下 10 场分享。

资料状态先说明：原讨论时，部分 GDC Vault 页面会跳转维护页；“免费内容”也可能只有幻灯片。下面分别标出公开 PDF、厂商录像入口和 Vault 状态。推荐依据是官方内容介绍及已核实的幻灯片，不是声称看完了所有录像。

## 一、移动端数据流与渲染管线

### 1. UE5 Mobile, Unleashed on Snapdragon: How We Turbocharged ‘Neverness to Everness’
Yizhe Chai｜Perfect World Games

讲 UE5 移动渲染如何适配 Adreno，涉及 Tile Memory Heap、Render Pass 优化和带宽压力。

推荐理由：这是“Mobile Deferred 的中间数据能否留在片上”的实际工程案例。看时重点追问：哪些资源留在 tile 内？哪些 pass 的组织方式改变了外部读写？哪些收益是 Adreno 专属？
资料：Qualcomm 已发布[官方录像入口](https://www.qualcomm.com/developer/blog/2026/05/snapdragon-game-studios-developer-forum-gdc-2026-recap)，找到对应 NTE 标题即可。

### 2. Faster Frames, Lower Power: How ‘Infinity Nikki’ Leverages Adreno High Performance Memory
Qualcomm＋Infold Games

讲通过 Vulkan 扩展使用片上高性能内存，以及 UE5 集成和《无限暖暖》案例。

推荐理由：对应“带宽变化为什么影响功耗”的问题。它从资源放在哪里、怎样访问来优化，不只是减少 Shader 指令。注意这是特定硬件能力的应用，不应推广成所有手机都能照搬。
资料：[官方公开视频入口](https://www.qualcomm.com/developer/blog/2026/05/snapdragon-game-studios-developer-forum-gdc-2026-recap)，对应 Infinity Nikki 标题。

### 3. Modernizing the Rendering of ‘Minecraft’
A.J. Fairfield｜Mojang

讲 RenderDragon、Vibrant Visuals，以及跨平台 PBR／Deferred 管线的演进。

推荐理由：把引擎架构、光照和移动性能取舍串在一起，适合综合研究。
- PDF 第 34 页：Deferred 分阶段控制分辨率，透明物走 ForwardPBR，权衡 GBuffer 大小和逐像素光照成本。
- 第 58 页：移动端热降频、持续运行测试、分辨率与性能余量。

资料：[公开完整 PDF，65 页](https://media.gdcvault.com/gdc2026/Slides/Fairfield_AJ_ModernizingTheRenderingOfMinecraft.pdf)，已确认可读。
注意：这里的多分辨率着色不等于硬件 VRS。

## 二、光照与渲染架构，值得深入

### 4. Illuminating Roco’s Legacy: Scalable Global Illumination from Theory to Practice
Sheng Feng｜MoreFun Studios／Tencent Games

《洛克王国》的可扩展 GI：UE4 Forward+、预计算 PRT、Cascade Lighting Volume，以及 SH 压缩、ASTC 友好编码、探针预插值等优化。

推荐理由：很适合研究“怎样把 GI 做进移动端预算”，不是单纯展示效果。重点看预计算如何换取运行时成本，以及如何同时控制 GPU、内存和 CPU 卡顿。
资料：[公开 PDF，55 页](https://media.gdcvault.com/gdc2026/Slides/Sheng_Feng_IlluminatingRocosLegacyScalableGlobalIlluminationfromTheorytoPractice.pdf)。建议先看第 27–45 页的优化与实测部分。

### 5. The Evolution of Real-Time Global Illumination in ‘Love and Deepspace’
Tianlong Ruan｜Infold Games

《恋与深空》的移动端动态 GI，使用 Surfel＋3D Radiance Cascade，面向物体、灯光和场景动态变化。

推荐理由：与上一场对照看价值最大。一个偏预计算与预算控制，一个强调动态更新能力；比较它们的数据表示、更新成本和适用场景，比单看画面更有收获。
资料：[官方 Vault 录像入口](https://gdcvault.com/play/1035619/The-Evolution-of-Real-Time)，已核到免费标记和视频播放器；未核到独立 PDF。

### 6. Rip & Tear: Breaking Down the Rendering of ‘DOOM: The Dark Ages’
Philip Hammer、Dominik Lazarek｜id Software

idTech 8 的整帧拆解：Visibility Buffer、Compute 材质分派、Deferred Texturing、Tile Classification、Deferred Lighting，以及软件 Variable-Rate Compute Shaders。

推荐理由：直接承接 VRS 原理，进一步看“如何在计算着色路径里减少重复着色”，以及为什么引擎会重新组织几何、材质和光照阶段。它是桌面／主机参考，不能原样套到移动 TBDR。
资料：[官方会话与资料入口](https://gdcvault.com/play/1035744/Rip-Tear-Breaking-Down-the)。已核到内容及幻灯片入口，但资料请求有时跳维护页，未确认免费录像。

## 三、引擎与性能工程，不只是优化单个 Pass

### 7. Performance Tuning for Unreal Engine Mobile Development
Arvind Neelakantan｜Epic Games

覆盖 CPU 调度、Draw Call、内存占用、加载速度和运行时 Profiling。

推荐理由：补齐 GPU 之外的成本，避免把所有掉帧都归因于 Shader 或带宽。适合作为真机测试与定位的基础方法。
资料：[官方 Vault 录像入口](https://gdcvault.com/play/1035617/Performance-Tuning-for-Unreal-Engine)，已核到免费标记与视频播放器；另有[讲者配套 Epic 教程](https://dev.epicgames.com/community/learning/tutorials/X7Me/performance-tuning-for-unreal-engine-mobile-development)。

### 8. Metaperf: Industrializing Performance Optimization for ‘Delta Force’
Mingtong Chen、Qingyu Tan｜Team Jade／Tencent Games

讲低开销混合采样、逐帧卡顿分类、关联归因，以及如何把性能分析做成生产工具。

推荐理由：如果需要分析长期运行、偶发卡顿、多机型问题，这类“持续采集和自动归因”比只看一帧 GPU Capture 更重要。它不是单项渲染算法，而是性能问题的发现与处理体系。
资料：[官方 Vault](https://gdcvault.com/play/1036028/Metaperf-Industrializing-Performance-Optimization-for)。会话已核实，原讨论时页面可能处于维护状态，未找到已确认的独立公开视频／PDF。

### 9. Smooth Sailing: Using Asset Health Metrics to Optimize ‘Voyagers of Nera’
Eoin O’Grady｜Black Shamrock／Virtuos

用 UE5 插件、资产检查、看板和提交时校验，把性能规则放进制作流程。涉及纹理预算、Blueprint 卡顿、WPO 距离、Niagara 扩展设置等。

推荐理由：解决“今天优化完，明天资产又把性能吃回去”的问题。适合借鉴资产性能检查和持续集成，而不是等项目后期再集中救火。
资料：[官方视频入口＋文字复盘](https://www.virtuosgames.com/news/gdc-2026-talk-smooth-sailing-using-asset-health-metrics-to-optimize-voyagers-of-nera/)，公开可读。可先看 07:16 的架构和 09:21 的项目接入。

### 10. We’re Speaking Your Slang-uage Here: A Proof-of-Concept Console Backend for the Slang Shading Language
Adam Creighton、Kevin Nappoly｜Enduring Games

讲为 Slang 增加主机编译目标，涉及编译后端、能力定义、IR 转换、Bindless／指针支持和跨平台 Shader 管线。

推荐理由：偏引擎底层和 Shader 工具链。如果想理解“一套 Shader 如何覆盖多平台，同时保持性能”，这场比单个画面效果更值得研究。
资料：[官方介绍与录像入口](https://shader-slang.org/blog/2026/05/22/enduring-games-gdc-presentation/) · [公开幻灯片](https://shader-slang.org/docs/enduring-extending-slang/Enduring-Games-slides.pdf)。

还有三场可按方向补看：

- 移动端 GPU 定位方法：Arm 的 Performance Tuning and Upscaling for Mobile Development in Unity。虽用 Unity，但深度测试、透明裁剪、后处理和上采样的隐性开销很有通用价值。[官方技术复盘](https://developer.arm.com/community/arm-community-blogs/b/mobile-graphics-and-gaming-blog/posts/finding-and-fixing-hidden-performance-problems-in-mobile-games-with-arm-performance-studio)
- CPU／世界模拟架构：Supporting Thousands of NPCs in ‘Kingdom Come: Deliverance’ & ‘Kingdom Come: Deliverance II’，Petr Smrcek／Warhorse。看模拟 LOD 如何控制大量 NPC 的时间与内存预算。[官方会话](https://gdcvault.com/play/1035718/)
- PC 光追前沿：Implementing Real-Time Path Tracing in RE Engine for ‘Resident Evil Requiem’ and ‘Pragmata’，CAPCOM＋NVIDIA。看采样、光源裁剪、BVH 和既有引擎集成。[官方公开视频](https://www.youtube.com/watch?v=Gq1QdlQSzn0)

如果先只看三场，建议顺序是：NTE → Minecraft → 洛克王国。
先弄清移动 GPU 的片上数据组织，再看完整渲染管线的成本取舍，最后看 GI 怎样真正压进移动端预算。之后再接 DOOM，把 VRS、Compute 着色和新渲染架构连起来。
