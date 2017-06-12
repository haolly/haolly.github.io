---
layout: post
title: slua 代码阅读
---
[slua](https://github.com/pangweiwei/slua) 的代码看了有一段时间了，基本上是了解了它的工作原理，我将自己对代码的注释放在了 [github 上面](https://github.com/haolly/slua_source_note)，其中有很多 **TODO:** 的标记，都是我刚开始看的时候有疑惑的地方，后面慢慢的看懂后就逐个将这些标记去掉，不过还有一些没有消灭掉，等有空在继续看。先把笔记写下来

## 一，Lua 调用 C# 方法

### 1.1 如何调用一个导出的类的实例方法和静态方法？

通过设置 userData 的 metatable 为导出类的 instanc table，而 userData 的 `__index` 和 `__newindex` 每次都会被调用到 [^1] ，这两个方法首先获取ud的metatable， 然后从metatable中rawget，参考 LuaObject 类中的 `newindexfunc` 和 `indexfunc`
static 方法的调用和instance 方法的类似，因为创建类型table的时候设置了static table 为它的metatable，所以访问类型table(C#导出类)上面的方法、属性时回去调用static table 的`__index` 和 `__newindex` 方法

### 1.2 如何知道传递给Lua 函数的参数和C#中导出的函数参数一致？
不能知道，但是在调用函数之前checkType函数可以保证类型一致

### 1.3 如何在Lua中定义一个导出的C#类？{#2}
```lua
    local go = UnityEngine.GameObject("name")
    print(go.name)
```
因为在导出C#类的时候，会根据C#类的FullName 来定义一个三个Table，一个用于标示类型，一个用于存储static 内容，一个用于存储instance 内容，并且将第一个类型Table和Global 表串联在一起。static 表作为类型表的metatable，后者定义了`__gc` 元方法来调用构造函数，构造函数中将C#对象push到栈中，参考 2.3

### 1.4 如何在Lua中定义一个没有导出的C#类？
使用Slua.CreateClass()，这里会使用反射的方法来创建

### 1.5 能不能访问一个没有导出的C#类中的方法？
如果一个类没有导出，仍然可以在Lua 中使用，但是访问它的方法、属性、字段都会使用反射的方式。
当一个类没有导出的时候，这个类的userData 的metatable 会被设置为LuaVarObject类。参考 LuaVarObject

## 二，C# 调用 Lua 方法
### 2.1 如何从Lua 栈上获取值？
LuaTable 和 LuaThread、LuaFunction 都有自己的 C# 标识类，它们都是wrap 了一个lua reference，实现了一些 Lua 中的方法， 例如函数调用(`lua_call`)、Table 的下表访问(`__index`)。 从Lua 栈上获取值的时候，根据`lua_type`来获取相应的C#类型数据。如果是userdata，那么从缓存中去取，参考问题2.3。参考函数`checkVar`

### 2.2 Lua 函数是怎么调用的？ 传给函数的参数是怎么回收的？
函数调用和 Lua C API 一样，先push 函数，然后是参数。第二点参考2.3

### 2.3 如何将一个类实例 push 到 Lua 栈上去？如何防止托管代码中的类实例被 GC 回收？

每次 push 上去的其实只是一个 userData，这个 userData 设置的 metatable。push 到栈上去的类实例都在托管代码中有一个缓存着，等到 userData 被 Lua GC 回收的时候再去从缓存中删除。参考 `pushVar` 函数

[^1]: For tables, this metamethod is called whenever Lua cannot find a value for a given key. For userdata, it is called in every access, because userdata have no keys at all. https://www.lua.org/pil/28.3.html