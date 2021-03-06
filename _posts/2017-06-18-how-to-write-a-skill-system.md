---
layout: post
title : 怎么写一个技能系统
tags : [gamedev]
---
对于一个 MOBA 游戏来说，一个完善的技能系统可以给游戏带来很多有趣的玩法，所以一直很想写篇博客总结下自己对如何实现一个技能系统的想法。

#### 首先，什么是技能系统，为什么需要它？

MOBA 游戏中会有很多英雄，每一个玩家都可以有好几个英雄，而每个英雄他们都各自有各自的脾气、性格，于是他们每个人掌握的技能也就不同了，有的善于用弓箭，有的善于用长矛，有的善于用斧头，有的既善于用长矛又善于用斧头，像在去年的手游项目中，一个游戏中总共有一百多个英雄，每个英雄有两三个技能，而且还可以继续学习新的技能，如果将每个英雄和他所有的技能实现放到一块，那么不同英雄相同技能就会产生重复，重复再程序中是不能忍受的，于是想到一个编程的原则：**将经常变化的部分抽取出来** ，那么我们就可以将技能抽取出来，抽象化成一个 `skill` 类，每个英雄只需要包含相关的 `skill` 类就行了。于是我们可以有 skillA， skillB， skillC … 这样想要多少个 skill 我们写多少个就可以了。 可是后来发现，写了很多 skill，他们的结构框架是不变的，英雄开始释放技能，做一个动作表示要开始了，记录技能CD 时间，选择技能作用对象，整个过程中英雄死亡了怎么办，释放结束后做什么，这个技能对被攻击者产生什么样的伤害，对自己又有什么作用，等等这些每个技能都需要去考虑的事情，如果这些都可以转换到配置表里面去让策划同事来配，那么岂不是灵活很多 ？是的。于是，我们的 `skill` 类就要发生变化，他只是一个架子，但是这架子里面填充什么东西是由配置文件决定的。于是，本来每个英雄所需要的 skillA、skillB、skillC 都被一个叫做 skillA、skillB、skillC 的配置文件名给替代。

那么，回到我们的问题，技能系统就是将游戏里面的技能相关的功能整合到一起，方便扩展和使用的一个系统。

#### 那么，怎么实现？

前面说了，每个 skill 要以配置文件为数据驱动，那么就先来看看这里需要哪些数据呢。

1. 技能自身相关属性。技能名称，主动技能、被动技能，CD 时间，目标类型，前摇时间，作用半径，体力消耗，技能动画等等
2. 技能释放过程中需要做的事情。例如释放前的动画播放，释放过程中的表现，释放之后的表现等等。
3. 技能的作用效果。被作用者攻击力、速度、血量、或者其他的什么，例如眩晕、护甲等
4. 各种数值。包括确定的数值和非确定的百分比数值

其中第二点可以使用事件驱动的方式来设计，一个技能的释放过程可以有多个步骤，每个步骤都产生一个相应的**事件**，例如 `OnSpellStart` 开始释放技能，`OnFocusEnergyBegin` 开始蓄力，`OnSpellEnd` 技能释放结束等等，每一个事件可以对应一组行为，行为可以是提前预定好的，比如 `DMG` ，`cure` ，`addBuff` ，`removeBuff` ，`createBullet` ， `playEffect` ，`checkCondition` 等等，每一个行为都有其相应的参数。在程序中可以定义一个 `SkillActionBase` 的抽象类，每一个不同的行为都是一个继承它的具体实现类。

有一些细节可能需要考虑，比如，程序中，英雄的状态一般都是用一个 FSM 来表示的，技能释放FSM 中的一个 state，创建子弹时，子弹有自己的属性，例如速度、方向、最大攻击半径、爆炸效果、碰撞筛选(不能打中队友)等等。

第四点，数值，这些可能会和英雄的等级相挂钩，在配置文件的其他部分要可以去使用这些数值，所以要提供一个获取数值的方法，不，是一个映射相应数值的方法，例如 `{"DMG"}` 可以获取配置名字为 DMG 的数值项，具体实现方式因人而异。

**最后说一下 buff，可以说是技能中最灵活的部分，也就是第三点，技能的作用效果。** 

一个 buff 是什么呢？buff 是英雄身上的属性，它在运行过程中修改英雄的其他属性。一个英雄身上一个挂多个 buff，buff 可以增加，也可以移除。相同的 buff 可以是叠加的，也可以是替换的的关系。

因为 buff 是可以移除的，所以我们不能直接去修改英雄的属性，例如英雄 H 攻击力20， buffA 增加 Atk 5， buffB Atk 加倍，于是 H 的 Atk 现在变成 (20 + 5) * 2 = 50；移除 buffA，移除 buffB，Atk 变成 (50 - 5)/2 = 22.5，纳尼，怎么和原来的不一样了？难道只能按照添加 buff 的顺序来移除 buff，这不合理。正确的做法应该是不去修改英雄 H 的攻击力，而是每帧去重新计算，如果移除了 buffA，那么重新计算的结果就和只作用 buffB 的效果一样。

**更新：**

**这里每帧去计算其实是不合适的，比较好的做法是定义一个基础属性，所有的 buff 加成都是以这个基础属性为参照，AddP 和 AddV，分别去增加一个基础值的百分比或者一个固定的值。例如加基础属性中伤害的30%，这样就可以在 buff 作用的时候去计算一次属性加成，避免了每帧都去计算。**

 buff 的结构可以和技能类似，有其自身的属性，例如 `actionTime` ，`isHoldOnDeath` ，`thinkInterval` 等，数值可以是修改百分比或者具体数值，相关的作用属性可以和英雄本身具有的熟悉相同，例如 HP，state 眩晕、隐身、无敌状态等等

~~有一点可能令人疑惑，上面说的技能配置的第二点，技能行为，技能行为可以是 `DMG` ，但是相同的效果也可以用 buff 来实现，这两种方式有什么区别呢？~~

~~区别就在于第一种是一次性的，第二种是持续的。~~

**更新：**

*上面说的有误，伤害都是通过子弹来实现的，如果是近身攻击，例如一把刀，那就是刀碰到 NPC 的时候产生一个**事件** `OnHitNpc` ，然后在这个事件中去执行 `DMG` Action。 buff 的作用只是用来修改属性，就是一个**modifier**，这个属性可以是英雄的属性，也可以是某一个技能的属性，例如一个技能导弹，攻击范围是50米，这个 buff 可以修改它的攻击范围为100米。*

总结：所以，技能的作用效果是通过事件来触发的，事件中通过不同的 Action 来具体实施的。常见的 Action 有 `DMG`，`Cure`, `PlayEffect`, `ApplyBuff`, `RemoveBuff` 等

如果要实现一个 buff，作用于队友，持续10秒钟，每2秒钟恢复血量20%，那么 actionTime 就是10S，thinkInterval 就是2S，10S 之后就会把这个 buff 从英雄身上移除掉。

**更新：**

对于技能的前摇时间以前一直没弄明白，为啥需要这个东西，有一天吃饭的时候突然想到，前摇的一个重要作用就是可以让对方**预判**，这样对方才有机会去躲避这个攻击。

**有关 Buff/Modifier**：

**Buff/Modifier** 的结构可以和技能类似，包括自身属性，事件，作用效果。作用效果也是通过 Action来具体实施的，Action 里面也可以有 Event，这样就可以实现无限层次的嵌套。

当然这里的 Event 和 Skill 中的 Event 是不同的，这里的 Event 主要包括 `OnCreate`, `OnDestroy`,  `OnAttacked`, `OnAttackLanded`, `OnDeath` 等等，具体意思可以按照字面意思来理解。