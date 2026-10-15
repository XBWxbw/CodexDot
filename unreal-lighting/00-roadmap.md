# Unreal Engine 光照学习路线

建议按下面这条路线系统学，每一章都串起“原理 → UE 实现 → 画面问题 → 性能代价”，其他引擎用来对照。

## 基础分类

先建立一个重要的分类：
- 直接光／间接光：光经过怎样的传播路径
- 漫反射／镜面反射：表面如何散射光，两者都能接收直接光和间接光
- 烘焙／实时：什么时候计算
- Lightmass／Lumen：具体的求解方案
- AO：遮蔽的近似，不能代替完整的间接光

## 推荐学习顺序

### 1. 光照基础与 PBR
从渲染方程、BRDF、能量守恒入手，弄清 Base Color、Metallic、Roughness、线性空间和曝光如何影响结果。先能解释“一束光照到表面后，为什么会形成这样的像素颜色”。

### 2. 直接光与阴影
学习方向光、点光、聚光的计算与衰减，再看阴影贴图、CSM、VSM。结合 Static／Stationary／Movable，拆清光照和阴影分别在哪计算、存在哪里。

### 3. Lightmass 与烘焙光照
理解多次反弹、Lightmap UV／分辨率、漏光和接缝，以及动态角色如何通过 Volumetric Lightmap 接收预计算照明。注意：烘焙不等于只烘间接光，Static 和 Stationary 保存的贡献不同。[Lightmass 官方资料](https://dev.epicgames.com/documentation/unreal-engine/cpu-lightmass-global-illumination-in-unreal-engine)

### 4. 反射
把直接光的高光与环境的镜面反射联系起来，再比较 Reflection Capture、SSR、平面反射、光追和 Lumen Reflections。重点理解粗糙度、离屏信息、遮挡和不同方案之间的回退。

### 5. AO
区分材质 AO、烘焙 AO、SSAO／GTAO、DFAO，以及 GI 本身产生的间接阴影。重点回答：它影响哪部分光照、漏掉哪些信息、为什么叠加后容易过黑。

### 6. Lumen
按“追踪什么 → 用什么表示场景 → 如何求交与获取光照 → 如何积累和降噪”理解 Screen Traces、Surface Cache、软件／硬件追踪、间接光和反射。再分析漏光、离屏变化、拖影与小发光体噪点。[Lumen 技术细节](https://dev.epicgames.com/documentation/unreal-engine/lumen-technical-details-in-unreal-engine)

### 7. 移动端光照与跨引擎对照
把前面的模块重新组合：烘焙间接光、动态直射光、阴影、反射捕获和 AO，分析带宽、采样、ALU、内存及功耗。对照 Unity URP；高端动态方案再对照 HDRP。具体支持范围按 UE 版本和渲染路径核对。[移动端功能表](https://dev.epicgames.com/documentation/unreal-engine/mobile-rendering-features-in-unreal-engine)

## Lumen 与烘焙光照的边界

一个容易混淆的边界：Lumen GI 开启后，预计算静态光照贡献会被禁用；但独立的 Lumen Reflections 可以在满足硬件光追要求时搭配烘焙光照。[官方说明](https://dev.epicgames.com/documentation/en-us/unreal-engine/lumen-global-illumination-and-reflections-in-unreal-engine)

## 贯穿全程的实践场景

实践上，用同一个“彩色墙房间＋金属／非金属球＋移动物体”贯穿全程，固定曝光，每次只切换一种贡献，并记录 GPU 毫秒。

我建议先从“一个像素的最终颜色由哪些光照贡献组成”讲起，再逐步落到 UE 的数据和渲染流程。

## 开始学习

- [第 1 章：光照基础与 PBR](01-pbr-basics.md)
- [第 2 章：直接光与阴影](02-direct-light-and-shadows.md)
- [练习 01：金属与非金属材质球](exercises/01-material-spheres.md)
- [返回仓库首页](../README.md)
