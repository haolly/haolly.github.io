---
layout: post
title: UGUI 源码阅读
tags: [dev]

---

最近有空看了下UGUI 的源码，学习一下如何写一个UI 库，虽然UGUI 是开源了，但是核心的几个文件Canvas/CanvasRender/RectTransform等还是没有开源，所以UGUI 的开源意义其实不大，更多的是一个噱头吧。

这里记录一下对源码的理解。

可以被剪裁的对象：IClippable
IClippable 的cull 接口是对 clip 的一个优化？ 如果没有在rect里面，直接cull掉
可以被mask 的对象: IMaskable
这两个总是一起的？

RectMask2D 用来mask 那些 IClippable
mask 的时候会去遍历自己的所有父节点RectMask2D 来做交集， ref
RectMask2D:PerformClipping()
单个RectMask2D 的大小是根据Canvas 的大小决定的

RectMask2D VS Mask
RectMask2D 是用 rectClip + cull 来实现
Mask 是用 stencil, todo

MaskableGraphic 实现了这两种机制，前者在Notify2DMaskStageChanged() 函数中触发，后者在NotifyStencilStateChanged() 函数中触发
前者在 RectMask2D 的OnEnable/OnDisable 中触发，后者在 Mask and(or) MaskableGraphic 的 OnEanble/OnDisable 中触发

RectMask2D:OnEnable/OnDisable --> Notify2DMaskStageChanged()
导致所有孩子节点RecalculateClipping(),然后孩子节点将自己添加到父节点的clippable 列表里面去，AddClippable()

MaskableGraphic OnEnable/OnDisable/OnTransformParentChanged/.. -> 导致
UpdateClipParent() --> AddClippable(), 最后 ClipperRegistry:Cull() -->PerformClipping()

ICanvasElement 表示一个界面，CanvasUpdate 有5种枚举值，
PreLayout/Layout/PostLayout/PreRender/LateRender, 按照先后顺序执行

CanvasUpdateRegistry:PerfromUpdate()  所有UI Update 总入口，有两个列表， layourRebuildQueue
(mesh的重建） + PerformingGraphicUpdateQueue, layout rebuild 之后就会执行
 cull操作(ClipperRegistry:Cull())
Graphic 和 LayoutRebuilder 都实现了 ICanvasElement
接口，前者处理CanvasUpdate.PreRender，后者处理 CanvasUpdate.Layout
渲染的时候使用材质 materialForRendering, 这个是经过所有的 IMaterialModifier
处理过之后的材质；

IMaterialModifier 有哪些？TODO

所有显示对象的基类是 Graphic有layoutDirty/verticesDirty/materialDirty
三个层次的dirty
verticesDirty 会导致mesh 的重构

TODO
啥时候被加入layout 列表？
OnRecttransformDimensionsChange/OnTransformParentChanged/ Graphic:OnEnable/OnDisable,OnDidApplyAnimationProperties
1，Graphic:SetLayoutDirty()
LayoutRebuilder 负责具体执行

啥时候被加入graphic 列表？
verticesDirty/materialDirty的时候
todo

Canvas的排序是怎么做的？

EventSystem是如何将一个点击事件派送到一个按钮上的？
GraphicRaycaster 作为EventSystem 的一个module 来处理点击事件；
处理逻辑：找到当前Canvas 所对应的所有Graphic 对象并检测raycaster 位置是否正确，(每个Graphic对象在OnEnable的时候都会去注册自己跟父Canvas的关系) 
然后排除掉所有朝向与摄像机朝向角度大于90度的，以及所有位于摄像机后面的，最后按照 DESCENT order by depth/sortOrderPriority/sortingLayer/sortingOrder/depth/distance 排序
ref GraphicRaycaster:Raycast()


ICanvasRaycasterFilter 接口表示一个可以被Raycaster 的对象, Image 对象实现该接口

EventSystem:Update -> inputModule:Process() -> StandaloneInputModule:Process() -> ProcessTouchEvents() ->GetTouchPointerEventData() -> EventSystem:RaycastAll() -> raycastModule:Raycast()

StandaloneInputModule:ProcessTouchEvent() 处理touch事件

EventInterfaces.cs 中定义了相应的接口，例如 IPointerClickHandler / IBeginDragHandler/ IDragHandler/ IPointerDownHandler 等等
Selectable 实现了 MoveHandler+PointerDownHandler + PointerDownHandler + PointerEnterHandler + PointerExitHandler + SelectHandler + DeselectHandler
Button 继承了 Selectable, 并且自己实现了 IPointerClickHandler
ScrollRect 实现了 IBeginDragHandler, IEndDragHandler, IDragHandler, IScrollHandler 等

ExecuteEvents 类负责调用相应的事件 handler 进行处理, 主要有两个接口，ExecuteHierarchy / Execute, 前者从当前节点向上传递，直到遇到一个正常的handler来处理，后者直接在指定对象上面调用handler
还有一接口GetEventHandler 是从当前节点开始向上找handler
click 跟drop 事件是互斥的

以IPointerDown 为例：
在 StandalineInputModule:ProcessTouchPress() 中，在当前pointerEvent 的raycast对象上执行 (ExecuteHierarchy) PointerDownHandler


