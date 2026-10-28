# Masked 材质的 Prepass：discard 分离与 PowerVR HSR

- 原讨论日期：2026-10-26
- 归档日期：2026-10-28

> 本文整理自历史技术讨论，保留当时的参考资料与适用边界；归档不代表重新验证所有设备、驱动或引擎版本。机制比较不是跨芯片性能实测。

PowerVR 官方明确支持 Masked 材质这一例外：它把“用 depth prepass＋深度测试替代颜色阶段的 alpha-test 工作”列为优化策略，说明这会增加 draw call／几何成本，但可能显著降低 ALU 和寄存器压力。[PowerVR 官方负载平衡指南](https://docs.imgtec.com/performance-guides/graphics-recommendations/html/topics/balancing-workloads-on-powervr-to-eliminate-bottlenecks.html)

所以，针对 Masked 材质，不能因为 PowerVR 有 HSR 就建议关闭 Prepass。这修正了仅以普通 opaque HSR 为依据讨论 Prepass 时容易遗漏的边界。

## 1. 关键不是“消灭 discard”，而是把它搬到更便宜的阶段

没有专门拆分时：
- Masked 材质需要计算 UV、采样 alpha、判断 discard。
- 覆盖尚未确定，硬件不能完全按普通 opaque 的方式提前确定遮挡。
- Shader 中还有法线、光照等颜色计算；实际执行多少，取决于编译器调度和硬件，不能一概假设所有被 discard 的片元都执行了完整材质。

拆分后：
- Prepass：只执行确定覆盖／深度所需的计算，alpha-test 通过才写深度。
- BasePass：复用这份深度，典型做法是 EQUAL 测试、关闭深度写入，并使用不再执行 Mask 裁剪的颜色 Shader。
- 不匹配深度的位置，在昂贵颜色着色前被拒绝。

它把“为了知道有没有这片叶子而必须做的计算”和“叶子的最终材质计算”隔离开来。HSR 并不能凭几何轮廓猜出 alpha 纹理里的洞，因此这个拆分不是普通 opaque HSR 的简单重复。[PowerVR 对 alpha-test 深度反馈的解释](https://docs.imgtec.com/starter-guides/powervr-architecture/html/topics/rules/do-not-use-discard.html)

## 2. 但“打开 Prepass”不等于完成了这项优化

必须确认 BasePass 真的去掉了 Mask／discard 路径。

如果深度阶段算一遍 alpha，BasePass 又重新算一次，可能只是增加了一轮工作，只获得部分深度拒绝收益。

UE 把“Early Z Pass”和“Mask Material Only in Early Z Pass”列为两个独立设置，正是这个区别。后者涉及 Shader 编译配置，官方注明更改需要重启编辑器。[Epic 渲染设置说明](https://dev.epicgames.com/documentation/unreal-engine/rendering-settings-in-the-unreal-engine-project-settings?lang=en-US)

另外，仅把深度测试改成 EQUAL 并不自动保证可以安全删掉 discard：
- 两遍的 WPO／蒙皮位置、alpha 阈值、LOD／抖动和采样覆盖必须匹配。
- MSAA、Alpha-to-Coverage、Pixel Depth Offset 需要单独处理。
- 深度不携带图元身份，共面或等深度情况也需要保证正确性。

所以应使用引擎完整支持的路径，不能只改一个 DepthFunc。

## 3. 那么 PowerVR 关闭 Prepass，什么时候才可能更快？

更可能适合关闭：
- 主要是普通 opaque，HSR 已经充分处理隐藏片元。
- Masked 覆盖少、重叠少，颜色 Shader 也便宜。
- 顶点／蒙皮／WPO 很贵，重复几何处理成为主要成本。
- 新增 Pass 导致实际的深度保存、加载或同步开销，而节省的着色不足以抵消。

更可能适合保留：
- 大面积重叠植被、毛发卡片，颜色着色昂贵。
- Alpha／覆盖判定比完整颜色 Shader 便宜很多。
- BasePass 确实移除了裁剪路径。
- 后续功能本来就需要这份预先生成的深度。

这些是机制上的判断方向，不是跨所有 PowerVR 型号的性能保证。额外 Pass 是否产生外存往返，也必须看实际 Render Pass／subpass 组织，不能只凭 UE 的 Pass 名称判断。

## 4. UE 移动端也有官方正面案例

Epic 的 MetaHuman 配置文档明确说 Early Z Pass 可以加速移动端头发渲染，并列出 Android／iOS 的 r.Mobile.EarlyZPassOnlyMaterialMasking 配置。这不能作为某款 PowerVR 的实测证据，但足以说明“移动 TBDR 一律关 Prepass”不成立。[Epic MetaHuman 配置建议](https://dev.epicgames.com/documentation/metahuman/requirements-and-configuration-settings-for-metahumans-in-unreal-engine?lang=de)

## 5. 真正该比的是三组，而不是简单开／关

A：没有这条 Masked Prepass 优化路径。
B：有 Prepass，但 BasePass 仍计算 Mask。
C：有 Prepass，而且 Mask 只在 Prepass 计算。

在引擎支持且画面一致的条件下，比较整帧 GPU 时间，同时查看 Prepass＋BasePass 总成本、着色调用、ALU／纹理负载、Tiler／ISP、外部带宽。

只看“关闭后少了一个 Prepass”不成立；只看“开启后 BasePass 快了”也不成立。

准确结论是：PowerVR 的 HSR 会削弱普通 opaque Prepass 的必要性，却不会自动消除 Masked Prepass 的价值。discard 与颜色着色的分离，恰恰是保留它的重要理由之一。
