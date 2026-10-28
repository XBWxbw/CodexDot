# UE Mobile Forward 与 Mobile Deferred：画质、性能及功耗取舍

- 原讨论日期：2026-10-23
- 归档日期：2026-10-28

> 本文整理自历史技术讨论，保留原有技术边界及公开参考链接；归档时未重新核验最新引擎、API 或设备支持状态，也不是新增的实机性能测试。具体支持与收益需按实际版本、平台和驱动验证。

以 Unreal 的 Mobile Renderer 为前提，先给结论：

- 烘焙光照为主、动态灯少、需要 MSAA、覆盖较广设备：Mobile Forward 通常更合适。
- 动态光照复杂、局部灯较多、目标 GPU 能有效利用片上 tile memory：Mobile Deferred 更值得考虑。
- 不能简单认为 Forward 省带宽、Deferred 费带宽，更不能直接据此判断功耗。

下面从 ALU、带宽和功耗分别分析。

## 1. 核心区别：光照在哪一步算

Mobile Forward：
绘制物体时，在材质 Shader 中计算材质与光照，输出颜色。

Mobile Deferred：
先计算材质属性，写入 GBuffer，例如法线、基础颜色、粗糙度等；后续光照阶段读取这些属性，计算最终照明。

Deferred 的优势是把“材质计算”和“光照计算”分开，避免每个物体的材质 Shader 都携带完整的光照处理逻辑，也能减少部分逐 draw 的光照资源绑定。

但注意：Base Pass 指令变少，不等于整帧指令同幅减少。光照工作有一部分只是移到了后续阶段，比较时必须把两边都算上。[Epic：Mobile Deferred](https://dev.epicgames.com/documentation/en-us/unreal-engine/using-the-mobile-deferred-shading-mode-in-unreal-engine)

## 2. 带宽：为什么移动 Deferred 不一定更贵？

桌面 Deferred 常见的成本模型是：GBuffer 写入显存，再被光照阶段读回来。

移动端不能直接照搬这个模型。UE Mobile Deferred 在适配硬件上可以利用 Vulkan subpass、Metal 对应机制，让中间 GBuffer 在片上 tile memory 内生成并消费，不必完整写到 LPDDR 再读回来。

所以：
- Forward 不需要这套中间 GBuffer，结构更简单。
- Deferred 有 GBuffer，但它不一定产生大量外部读写。
- Deferred 的片上读写、属性编码／解码、同步和光照阶段仍有成本，绝不是“免费”。

如果中间附件不能留在片上，或者实际渲染流程要求它们落到系统内存，Deferred 的带宽优势就可能减弱。要看目标设备和真实 pass 组织，不能只看管线名字。

与深度／模板格式分析一样：附件的逻辑位宽、实际分配大小和实际 LPDDR 流量，是三个不同的量。

## 3. ALU 和动态灯：Deferred 的主要机会在哪里？

动态光照少时：
Forward 在材质 Shader 里直接完成少量光照，可能很划算。Deferred 则还要付出 GBuffer 处理和后续光照阶段的开销，未必赚得到。

动态光照复杂时：
Deferred 将光照与材质解耦，通常更便于处理多个局部灯光，材质 Shader 和资源绑定也可能更简单。这时它更有机会降低总成本。

但要避免三个误解：
- Forward 也能支持动态局部灯，不是只有 Deferred 才能用。
- Deferred 的灯也有成本，受影响屏幕面积、灯重叠程度、光照算法都会影响耗时。
- Deferred 不会自动消除阴影贴图生成和阴影采样成本。

如果主要耗时是 Shadow Pass，而不是材质内的光照计算，仅切换 Deferred 不一定能显著改善。

## 4. 烘焙光照：Forward 往往更有优势

如果场景大部分照明已经烘进 Lightmap，只配少量动态灯，运行时没有那么多复杂光照需要 Deferred 来处理。

这时 Forward 的直接着色路径通常更合适，Epic 也明确建议以预计算光照为主的项目优先考虑 Forward。Deferred 不是不能用烘焙，而是它的优势可能不足以抵消额外成本。

另外，原讨论核对的 Epic 文档说明：Mobile Deferred 在使用预计算光照时，存在 DefaultLit／Unlit 的着色模型限制。这个条件不能省略，不能泛化成“Mobile Deferred 永远只有这两种材质”。[Epic：移动渲染模式选择](https://dev.epicgames.com/documentation/unreal-engine/mobile-rendering-and-shading-modes-for-unreal-engine?lang=en-US)

## 5. 画质功能：MSAA 是非常明确的分界线

Mobile Forward：
- 支持 MSAA，适合重视几何边缘清晰度、部分移动 XR 等场景。
- 但 MSAA 也有采样、存储和 resolve 成本，不是白送的画质。

Mobile Deferred：
- 不支持 MSAA，需要选择 TAA、FXAA 等适用方案。
- 动态照明和受光 decal 等需求是它的重要使用场景，但 decal、着色模型等仍有具体限制。

特别注意：两条路径都属于 Mobile Renderer。切到 Mobile Deferred 不等于切到桌面 Deferred，也不会自动获得 Lumen 或 Nanite。[Epic：移动端功能表](https://dev.epicgames.com/documentation/unreal-engine/rendering-features-reference)

## 6. 透明和 Overdraw：Deferred 不是通用解药

Mobile Deferred 的半透明物体仍然走 Forward 路径。

如果场景主要问题是大量透明粒子、玻璃、水面叠加，切换 Deferred 不会自动解决这些透明层的重复着色成本。

对不透明物体，也不能说“Forward 一定把所有被挡住的层都算完光照”。Early-Z，以及 Apple GPU 的隐藏面移除等机制，会影响实际执行的着色量。

还要区分两个不同概念：
- GPU 硬件的 Tile-based Deferred Rendering。
- UE 选择的 Mobile Deferred Shading。

UE Mobile Forward 同样可以运行在这种分块 GPU 上，并受益于它的片上存储和遮挡处理。[Apple：TBDR 架构](https://developer.apple.com/documentation/metal/tailor-your-apps-for-apple-gpus-and-tile-based-deferred-rendering)

## 7. 兼容性和工程成本

Forward：
- 通常更适合作为广泛机型覆盖的起点。
- 简单光照场景下，管线和性能分析更直接。
- 复杂光照可能让材质 Shader、资源绑定和相关变体更复杂。

Deferred：
- 对目标 GPU、图形 API、驱动和片上附件处理方式更敏感。
- GBuffer 容量有限，材质信息如何编码、特殊着色模型怎么支持，都需要考虑。
- 更适合针对明确的设备范围验证，而不是假定所有手机都能得到同样收益。

这些细项要以实际 UE 版本和 RHI 为准，尤其不能沿用旧版本的灯数量、GLES 或材质支持结论。

## 8. 回到功耗：哪条更省电？

真正要比较的是：
少掉的计算、资源管理和外存访问成本，是否超过新增的 GBuffer 处理、光照阶段和同步成本。

Deferred 可能减少外部流量，却因执行依赖或调度使 GPU 更忙；也可能同时减少总执行时间和能耗。Arm 对 Mali 的说明就提过：片上 PLS／subpass 可以降低带宽，但某些旧架构上的依赖串行化会影响性能。[Arm：Mali-G710 架构说明](https://developer.arm.com/community/arm-community-blogs/b/mobile-graphics-and-gaming-blog/posts/mali-g710-developer-overview)

因此，不应该只盯着 GB/s。建议 A/B 时保持同一设备、场景、分辨率和目标 FPS，先用双方都支持的同一抗锯齿方案，再比较：
- 全帧 GPU 时间，以及 Base Pass＋Lighting 的合计成本。
- Shadow、Translucency、Post Process 各自耗时。
- 外部读写带宽、GPU 频率与活跃时间。
- 稳态功率、每帧能量和持续运行后的帧率。

选型建议：Lightmap＋少量动态光，先选 Forward；大量动态局部光，优先测试 Deferred；透明、阴影或后处理已经占大头，先优化这些，不要期待切管线就解决。

最终选择需要结合具体 UE 版本、目标 GPU，以及烘焙光照与动态光照的占比。
