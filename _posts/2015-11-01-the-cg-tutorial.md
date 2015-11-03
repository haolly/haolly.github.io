---
layout: post
title: "The Cg tutorial"
description: ""
category: 
tags: []
---

前段时间看了有关 Shader 方面的书籍，当然最先开始的就是 Nvidia 的 The Cg tutorial 了，因为直接看 Unity 中的 Shader 教程感觉有点不知所以然，于是就用了一周时间看完了 Cg tutorial， 里面的代码都没有自己去写，因为写出来也不知道如何去运行==

个人感觉里面比较重要的东西有下面几个

* 图形渲染管线
* 坐标系统转换
* 光照模型

其中不是很清楚的一点就是顶点程序和片段程序它们的分工，不过下面这句话倒是有些提示：

>The vertex program passes the specified 2D position of each vertex to the rasterizer. The rasterizer expects positions to be specified as coordinates in clip space. Clip space defines what is visible from the current viewpoint. If the vertex program supplies 2D coordinates, as is the case in Figure 2-5, the portion of the primitive that is rasterized is the portion of the primitive where x and y are between -1 and +1. The entire triangle is within the clipping region, so the complete triangle is rasterized.

但是另外一本书*Shaders.for.Game.Programming.and.Artists*，里面用的 renderMonkey 却分出了 Vertex shader 和 Pixel shader， 暂时感觉和上面的好像是一样的，等看完再说。

总之，这本书非常适合用于回顾忘掉的东西