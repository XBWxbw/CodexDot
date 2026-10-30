# CodexDot

整理图形 API、GPU 渲染与 Unreal Engine 光照学习资料，按“原理 → 实现 → 画面问题 → 性能代价”逐步学习。

## 技术报告归档

[技术问题报告总索引](reports/README.md)：共 15 篇 Markdown 报告。2026-10-30 新增 [Vulkan 基础概念讲解](reports/graphics-api/2026-10-30-vulkan-basics.md)，从设备、资源和命令讲到同步与一帧渲染流程。

原有 14 篇报告涵盖移动 GPU 功耗与带宽、Metal 深度格式、Forward／Deferred、VRS、Tile Memory Heap、HSR／Prepass、Shader 分支，以及 GDC 2026／NTE 案例；原讨论日期为 2026-10-22 至 2026-10-27，归档日期为 2026-10-28。

各篇保留公开参考资料、适用条件与验证边界。历史归档的兼容性和资料访问状态按原讨论日期记录，不等同于最新状态或新的性能实测。已有光照系列正文保持不变。

## 阅读入口

1. [光照学习路线](unreal-lighting/00-roadmap.md)：七个主题的学习顺序、基础分类与技术边界。
2. [第 1 章：光照基础与 PBR](unreal-lighting/01-pbr-basics.md)：从一个像素的颜色理解光照、材质与成像处理。
3. [第 2 章：直接光与阴影](unreal-lighting/02-direct-light-and-shadows.md)：光源、衰减、Shadow Map、CSM、VSM、移动性与移动端取舍。
4. [练习 01：金属与非金属材质球](unreal-lighting/exercises/01-material-spheres.md)：控制变量的建议实验与待填写记录。
5. [图片目录说明](unreal-lighting/images/README.md)：后续截图的保存和标注方式。

## 当前内容范围

已整理学习路线和第 1、2 章完整内容。第 3 至第 7 章是后续学习主题，尚未在此提供正文。第 2 章内含四个待执行的控制实验。材质球练习是建议方案，尚未执行；图片目录目前没有实验截图。

正文保留参考资料链接。UE 功能和具体设置应结合实际使用的引擎版本、平台与渲染路径核对。第 1 章主要讨论常见不透明表面的传统 Default Lit 金属度工作流，并说明与 Substrate 的接口边界。

## 使用方式

先阅读路线和第 1 章，再搭建材质球实验。固定曝光，每次只改变一种因素，记录实际观察和测试条件；后续可以沿用彩色墙房间、金属／非金属球和移动物体逐章验证光照贡献。

