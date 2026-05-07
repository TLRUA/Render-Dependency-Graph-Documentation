# 使用 Unreal Engine 的渲染与图形编程

> 出处：本文档翻译自 staticJPL 的 **Render Dependency Graph Documentation** 项目，原仓库：https://github.com/staticJPL/Render-Dependency-Graph-Documentation 。翻译在尊重原意的基础上，对部分表述做了中文化整理。

前言：本文旨在概述 Unreal Engine 5.1 内部使用的渲染管线。文档会介绍图形管线的一些基础知识，以及 Unreal 如何通过多种图形 API 执行渲染。

### 目录
1. [图形编程基础](https://github.com/staticJPL/Render-Dependency-Graph-Documentation/blob/38befa753f23766c3c26196799a320c90a42b5e7/Fundamentals%20of%20Graphics%20Programming.md)
2. [Unreal Engine 渲染管线](https://github.com/staticJPL/Render-Dependency-Graph-Documentation/blob/38befa753f23766c3c26196799a320c90a42b5e7/Unreal%20Engine%20Rendering%20Pipeline.md)
3. [渲染依赖图](https://github.com/staticJPL/Render-Dependency-Graph-Documentation/blob/38befa753f23766c3c26196799a320c90a42b5e7/Render%20Dependency%20Graph%20(RDG).md)
4. [SceneView Extension 三角形 Shader](https://github.com/staticJPL/Render-Dependency-Graph-Documentation/blob/38befa753f23766c3c26196799a320c90a42b5e7/Triangle%20Shader.md)
5. [参考资料](https://github.com/staticJPL/Render-Dependency-Graph-Documentation/blob/38befa753f23766c3c26196799a320c90a42b5e7/References.md)
6. [三角形示例代码](https://github.com/staticJPL/TriangleViewExtensionExample)
