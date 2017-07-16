---
layout: post
category : gamedev
title : 关于游戏服务器端框架的思考
tags : [gamedev]
---
这段时间飞机的项目进度不怎么赶了，于是可以把以前所想的记录下。这里说说我所遇到过的游戏服务器的框架吧，还有我对这些框架的看法。

第一个是在做页游时的，用Java NIO 实现的 [Reactor 模型](https://en.wikipedia.org/wiki/Reactor_pattern)。简单来说就是运用多线程将客户端的请求分发给不同的Worker 去处理，多线程不但可以保证服务器的资源得到有效利用。当初我自己也随便写了一个简单的版本，[my-nio-server](https://github.com/haolly/my-nio-server)作为练习。 由于使用了多线程，数据的锁定和同步问题有时会有点麻烦。服务器的更新主要是通过反射将写好的Extension 加载进去，每个Extension就相当于一个模块，但是每个Extension 不保证在同一个线程中执行，而且每个玩家的同一个Extension也不保证在同一线程中执行。我发现上面GitHub上面的连接已经坏掉了，但是有篇博客[bbsmax](https://www.bbsmax.com/A/8Bz8p7axzx/) 是一个很好的补充。

第二个是在做第一款卡牌手游的时候接触的。整个服务器按照功能被分拆成GS(GameServer)，GC（GameCenter），ServiceCenter，ZoneServer，Gateway，BS（BattleServer）等，每个 Server 都是独立运行的进程，每个进程中都是单线程的，各个 Server 启动后都会去 ServiceCenter 注册自己，这样需要通信的 Server 之间就可以查询到彼此的信息。
个人觉得这种架构比较好，首先是编写代码方便，不需要去操心多线程的问题，可以更多的将精力放在业务上，其次这种架构在扩展上面也很方便，只需要多开几个 GS/GC/ZoneServer 就行，这样也充分利用你物理机器的资源，而且最重要的是，这种架构天然支持分布式。

那么上面两种孰优孰劣呢？我个人比较倾向于第二种。陈硕有篇文章有描述[多线程服务器的适用场合](http://blog.csdn.net/Solstice/article/details/5334243),可以看看。

在做第二款 MOBA手游的时候，同事longwei(https://github.com/lailongwei) 对服务器中的玩家的数据持久化方式进行了改进，引入了 `BaseGameMgr` 和 `BaseGlobalMgr` 这两个概念，前者用于单个玩家的功能模块，比如背包、好友等，后者用于全局的功能模块，例如排行榜等。以前的玩家数据都是自己创建表，然后提交给 DB 模块去定时写入数据库的，现在引入一种标脏的机制和可定制的序列化方式，默认会去保存所有 `ProtoMember` 标示的字段，玩家登陆的时候去反序列化所有的 GameMgr，调用其 `OnLogin` 等方法，如果发现新的 mgr 那就在 role 中添加新的字段来存储新的数据。 **但是这里有一个问题，如果游戏中有大量的 BaseGameMgr，但其实玩家只是用到了其中的一部分，那么全部加载会浪费内存**

