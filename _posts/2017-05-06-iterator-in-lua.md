---
layout: post
title: Iterator in Lua
tags : [programming]
---

最近在看 [Slua](https://github.com/pangweiwei/slua) 的代碼，因爲現在做的項目中使用到了它，在看的過程中逐漸的對 Lua 的理解越來越清晰。

這裏說一些Lua中的迭代器。很多語言中都有迭代器這個概念，早期在學校學C++ 的時候就接觸過，後面在Java 裏面也看到了，再後來在C#裏面也看到了，如今在Lua中又見到了。

JAVA和C++ 太久遠了，已經記不清楚了。在C#中，和枚舉有關的概念有兩個，一個是可枚舉接口 `IEnumerable`  和 枚舉器（迭代器） `IEnumerator` ，一個集合要想可以被枚舉，就必須要實現 `IEnumerable` 這個接口，而這個接口中定義了方法 `GetEnumerator` ，返回類型爲迭代器 `IEnumerator` ，迭代器就像是一個函數，這個函數保存了當前迭代的位置信息，每次當這個函數調用的時候，它就返回當前位置的元素，然後將位置信息加一。**在C#中實現迭代器需要同時實現 `IEnumerable` 和 `IEnumerator` 這兩個接口 [^1]。**在 Lua中也有類似的概念，實現`IEnumerable`  接口就是給這種類型寫一個迭代器，然後這個迭代器就可以在 `for`  語句中使用了， 就像C#中的 `foreach` 一樣。

那麼Lua中的迭代器如何寫呢？類似於C#中的 `GetEnumerator` ,這個函數就像是一個工廠函數，它產生一個迭代器，Lua中也需要一個這樣的工廠函數，例如下面：

```lua
function values(t)
  local i = 0
  return function() i = i+1; return t[i] end
end
t = {1, 2, 3}
for element in values(t) do
  print(element)
end
```

這裏的 `values()` 函數就是這個工廠。

再來看看Slua中的代碼：

```csharp
[MonoPInvokeCallbackAttribute(typeof(LuaCSFunction))]
static public int _iter(IntPtr l)
{
  object obj = checkObj(l, LuaDLL.lua_upvalueindex(1));
  IEnumerator it = (IEnumerator)obj;
  if (it.MoveNext())
  {
    pushVar(l, it.Current);
    return 1;
  }
  else
  {
    if (obj is IDisposable)
      (obj as IDisposable).Dispose();
  }
  return 0;
}

[MonoPInvokeCallbackAttribute(typeof(LuaCSFunction))]
static public int iter(IntPtr l)
{
  object o = checkObj(l, 1);
  if (o is IEnumerable)
  {
    IEnumerable e = o as IEnumerable;
    IEnumerator iter = e.GetEnumerator();
    pushValue(l, true);
    //the only upvalue
    pushLightObject(l, iter);
    LuaDLL.lua_pushcclosure(l, _iter, 1);
    return 2;
  }
  return error(l,"passed in object isn't enumerable");
}
```

`iter` 函數就是那個產生迭代器的工廠函數，這裏的迭代器就是函數 `_iter` ，每次調用迭代器都會產生下一個元素。

於是就可以在Lua中這樣使用了：

```lua
for element in Slua.iter(collection) do
  print(element)
end
```



同樣需要注意的是，迭代器只用予迭代，不能在迭代過程中去修改被迭代的集合。這幾乎在所有語言裏面都是一樣的。


[^1]: [也可以使用yield 關鍵字](https://docs.microsoft.com/en-us/dotnet/articles/csharp/language-reference/keywords/yield)