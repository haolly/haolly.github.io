---
layout: post
title: "Entity System"
description: ""
category: 
tags: [game]
---
{% include JB/setup %}

#Entity System
最近在网上看到一个游戏框架，[Ash](http://www.ashframework.org/),作者Richard Lord在自己的博客上也对这个框架做了介绍，看过之后算是对中基于组件的架构有了些认识。

其主要思想是**组合优于继承**，将游戏系统中大量复杂的功能通过组合的形式而不是继承来结合在一起。

这个系统里面主要分为三块：

1. Entity
2. Component
3. System

组件就是物体所具有的某一功能或者属性的抽象，例如一个声音组件，一个灯光组件，一个显示组件等，可以参考 Unity3d 中的组件概念。组件只是数据的 Container，它本身是不包含具体操作的。Entity 是由一个个的组件组合而成，每个组件代表着一个实体的具体功能或者属性，例如一个游戏玩家具有显示组件和移动组件等，那么这个玩家的状态是如何更新的呢，这就是System 的作用了。System 是根据每个 Entity 中的组件数据来跟新Entity 的状态，一个游戏中会有不同的 System，例如 RenderSystem，MoveSystem，SoundSystem 等，每个 System 负责更新游戏中所有具有某个组件的实体。

这样的好处就是容易扩展实体的功能和熟悉，而不会对已有的代码产生影响。


*参考*

1. [Richard Lord 博客](http://www.richardlord.net/blog)
2. [Introduction to Component Based Architecture in Games](http://www.raywenderlich.com/24878/introduction-to-component-based-architecture-in-games)