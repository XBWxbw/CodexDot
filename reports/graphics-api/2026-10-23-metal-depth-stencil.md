# Metal 深度／模板格式：有效位数、分配大小与带宽

原讨论日期：2026-10-23  
归档日期：2026-10-28

> 本文是历史技术讨论的整理归档，保留当时的结论、适用边界与公开参考链接；不是新增设备基准测试，也不是对当前平台支持状态的重新审计。涉及实验数据时，仅作为匿名报告的局部案例，不代表所有设备。

部分设备上组合深度／模板格式会分配 64 bit／像素，但不是 Metal 统一规定“32-bit depth 必须按 64 bit 存”。关键看像素格式：

- Depth32Float：只有 32-bit 深度，没有隐含的 stencil。
- Depth32Float_Stencil8：32-bit 深度＋8-bit 模板，逻辑上共 40 bit；Apple 明确说明，部分 Metal 设备会为它分配 64 bit／像素。[Apple 官方格式说明](https://developer.apple.com/documentation/metal/mtlpixelformat/depth32float_stencil8)

所以，“按 64 bit”确实可能发生，但应理解为组合格式在特定设备上的存储分配方式，不能推广成所有设备都如此，也不能据此认定 depth 和 stencil 在内存中一定是某种交错布局。

## 如何区分内存占用与实际流量

### 1. 实际内存占用
看创建后的 `texture.allocatedSize`；使用 heap 时，可以预先查询 `heapTextureSizeAndAlign`。注意，后者的 `align` 是整个纹理资源的对齐要求，不是单像素的 64-bit 对齐。[分配大小](https://developer.apple.com/documentation/metal/mtlresource/allocatedsize)

### 2. 实际带宽消耗
即使按 8 字节／像素分配，也不等于每次深度测试都从 LPDDR 读写 8 字节。缓存、片上 tile 存储、压缩以及附件的 load／store 行为都会影响外部流量。尤其是 Apple GPU 的 memoryless 深度／模板附件，没有全屏的系统内存后备存储，不能按“分辨率 × 8 字节 × 测试次数”来算带宽。[Memoryless 说明](https://developer.apple.com/documentation/metal/mtlstoragemode/memoryless)

准确结论：D32S8 的有效数据是 40 bit，部分设备分配 64 bit；分配位宽不等于实际外存读写量。
