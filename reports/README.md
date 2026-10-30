# 技术问题报告归档

首次归档日期：2026-10-28；最近更新：2026-10-30。将技术讨论与学习讲解整理为可检索的 Markdown 报告，保留结论、推理、适用条件、实验建议和公开参考链接。

## 归档范围与阅读说明

- 当前共 **15 篇报告**：2026-10-28 归档 14 篇历史讨论（原讨论日期为 2026-10-22 至 2026-10-27）；2026-10-30 新增 Vulkan 基础概念讲解。
- 更早的光照学习路线、PBR、直接光与阴影及材质球练习已经归档在 [`unreal-lighting/`](../unreal-lighting/00-roadmap.md)，本次不重复生成或改写其正文。
- 2026-10-28 批次覆盖截至当日可读取历史中的已形成技术总结；尚未展开的学习路线章节不冒充已完成报告。
- 历史讨论归档不是新一轮设备实测或最新兼容性审计。设备、驱动、UE 版本、资料可访问性等判断，以各篇注明的讨论或编写日期和证据范围为准。
- 来源区分为官方文档、公开会议资料与社区逆向资料；机制推断不等于跨芯片实测。未能可靠核实的性能百分比没有补写为确定结论。
- 内容采用公开发布版本：不包含聊天原始记录、账号标识、私有链接或个人信息。匿名测量案例仅用于解释方法，不能外推为设备通用常数。
- HSR 总览用可随 Markdown 保存的 Mermaid 示意图重述流程；它们不是厂商电路图，也不是原聊天位图附件的复制。

## 性能、带宽与功耗

| 原讨论日期 | 报告 | 重点 |
| --- | --- | --- |
| 2026-10-22 | [移动端功耗：带宽与 ALU 谁占主导](performance/2026-10-22-mobile-power-alu-bandwidth.md) | 单次能耗与总功耗、增量斜率、DVFS 与控制实验 |
| 2026-10-22 | [带宽和 ALU 如何消耗电能](performance/2026-10-22-power-mechanisms.md) | 电容充放电、数据搬运、运算切换及调频影响 |
| 2026-10-22 | [带宽、Load／Store 与外部流量](performance/2026-10-22-bandwidth-load-store.md) | 带宽能力与占用、缓存、纹理访问和 LS 指标 |

## 图形 API 与片上内存

建议初学 Vulkan 时先读基础概念，再读 QCOM Tile Memory Heap 扩展专题。

| 讨论／编写日期 | 报告 | 重点 |
| --- | --- | --- |
| 2026-10-30 | [Vulkan 基础概念讲解](graphics-api/2026-10-30-vulkan-basics.md) | 设备、队列、资源、Pipeline、Descriptor、同步与一帧渲染 |
| 2026-10-23 | [Metal 深度／模板格式与存储](graphics-api/2026-10-23-metal-depth-stencil.md) | Depth32Float、Depth32Float_Stencil8、分配量与实际带宽 |
| 2026-10-26 | [Vulkan QCOM Tile Memory Heap](graphics-api/2026-10-26-qcom-tile-memory-heap.md) | 资源创建、作用域、同步、设备查询及 UE 支持边界 |

## 渲染路径与着色

| 原讨论日期 | 报告 | 重点 |
| --- | --- | --- |
| 2026-10-23 | [UE Mobile Forward 与 Mobile Deferred](unreal/2026-10-23-mobile-forward-vs-deferred.md) | 光照、片上 GBuffer、MSAA、功耗与设备取舍 |
| 2026-10-23 | [VRS 可变速率着色原理](rendering/2026-10-23-vrs-principles.md) | 覆盖与着色频率、粗着色、收益及画质风险 |
| 2026-10-27 | [Uniform 动态分支与 Shader 性能](shaders/2026-10-27-uniform-dynamic-branches.md) | 发散、编译结果、寄存器、occupancy 与静态变体 |

## 可见性、HSR 与 Prepass

建议先读总览，再读三篇机制比较与例外分析。

| 原讨论日期 | 报告 | 重点 |
| --- | --- | --- |
| 2026-10-26 | [各 GPU 架构的 HSR 原理](visibility/2026-10-26-hsr-architectures.md) | Early-Z、LRZ、FPK、Fragment Prepass、Apple 与 PowerVR |
| 2026-10-26 | [FPK 与 LRZ 的取舍](visibility/2026-10-26-fpk-vs-lrz.md) | 有限取消窗口与提前建立粗深度的差别 |
| 2026-10-26 | [PowerVR HSR 的优势和边界](visibility/2026-10-26-powervr-hsr-tradeoffs.md) | 更完整的不透明可见性，不等于 GPU 整体排名 |
| 2026-10-26 | [Masked Prepass 与 discard 分离](visibility/2026-10-26-masked-prepass.md) | 纠正“有 HSR 就应关 Prepass”的过度概括、验证 A/B/C 路径 |

## 会议资料与案例

| 原讨论日期 | 报告 | 重点 |
| --- | --- | --- |
| 2026-10-23 | [GDC 2026 引擎、渲染与性能分享](conferences/2026-10-23-gdc-2026-engine-rendering-picks.md) | 十场精选、公开资料入口、获取限制与阅读顺序 |
| 2026-10-24 | [NTE／异环的 Adreno 渲染优化](conferences/2026-10-24-nte-adreno-rendering.md) | Tile Memory Heap、Render Pass、SDOC、帧生成与证据边界 |

## 已有光照系列

- [光照学习路线](../unreal-lighting/00-roadmap.md)
- [第 1 章：光照基础与 PBR](../unreal-lighting/01-pbr-basics.md)
- [第 2 章：直接光与阴影](../unreal-lighting/02-direct-light-and-shadows.md)
- [练习：金属与非金属材质球](../unreal-lighting/exercises/01-material-spheres.md)

[返回仓库总目录](../README.md)

