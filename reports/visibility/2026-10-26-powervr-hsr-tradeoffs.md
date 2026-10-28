# PowerVR HSR 的优势与适用边界

- 原讨论日期：2026-10-26
- 归档日期：2026-10-28

> 本文整理自历史技术讨论，保留当时的参考资料与适用边界；归档不代表重新验证所有设备、驱动或引擎版本。机制比较不是跨芯片性能实测。

在正常不透明绘制、同一有效处理范围内，PowerVR 这种“先求可见性、再着色”的方案，在消除隐藏片元着色方面，比单独的 FPK 或 LRZ 更彻底。这是机制层面的比较，不是 GPU 性能排名。

## 1. 它同时避开了两类主要限制

- FPK：后来的遮挡物必须赶在旧片元离开可取消窗口之前到达。
- LRZ：提前建立了遮挡信息，但它是低分辨率的保守判断，不能证明隐藏的候选仍要继续处理。
- PowerVR HSR：ISP 先判可见性，Tag Buffer 记录每像素的前方图元，TSP 再为可见片元执行材质。[PowerVR 官方流程](https://docs.imgtec.com/starter-guides/powervr-architecture/html/topics/per-tile-rasterization.html)

因此它既不只是“及时取消”，也不只是“粗筛一下”。

## 2. 举个直观例子

假设同一屏幕区域有 10 层不透明表面，按远到近提交，最后只看到最近一层：

- FPK 可能取消一部分，但已经完成的着色追回不了。
- LRZ 可能提前拒绝很多隐藏工作；效果取决于粗粒度信息能否证明遮挡。
- PowerVR HSR 在理想兼容情况下，可以先选出可见层，只为它做主要着色。

这里的优势尤其适合“过绘制多、材质贵、提交顺序又不理想”的场景。PowerVR 官方也明确表示，普通 opaque 不需要仅为减少隐藏片元着色而按距离排序，可以优先按材质／状态组织。[官方排序建议](https://docs.imgtec.com/performance-guides/graphics-recommendations/html/topics/sorting-geometry-effectively-on-powervr.html)

## 3. 为什么仍不能说它全面最好？

第一，没着色不等于没处理。隐藏图元仍可能付出顶点、分块、位置读取、光栅与可见性判断成本。几何瓶颈场景未必因此大幅提速。

第二，透明、discard、Shader 深度反馈等不能都简化成“只留最近一层”。例如 alpha-test 必须先确认哪些样本真正存在，才能完成相应深度更新。[PowerVR 的反馈限制](https://docs.imgtec.com/starter-guides/powervr-architecture/html/topics/rules/do-not-use-discard.html)

第三，如果本来就低过绘制，或者 Early-Z 已经挡掉绝大多数隐藏片元，这种更完整可见性处理的增量收益会小很多。

第四，FPK 和 LRZ 只是各自 GPU 的一部分，不能拿它们单独对比 PowerVR 整套管线，得出芯片胜负。

因此结论应限定为：
“PowerVR 的不透明 HSR 更接近只为最终可见表面着色的目标”，而不是“PowerVR GPU 一定比 Mali／Adreno 好”。

另外，新 Arm Fragment Prepass 已经走向类似的“先确定兼容范围内可见性，再主要着色”思路，不能再用旧 Mali 的 FPK 单独代表它。

## 与 Masked Prepass 的关系

普通 opaque 的 HSR 优势不等于 Masked 材质无需 Prepass。后续讨论补充了 alpha-test 与颜色着色分离的价值，见 [Masked Prepass 的收益与验证](2026-10-26-masked-prepass.md)。
