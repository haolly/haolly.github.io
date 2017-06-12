---
layout: post
title: "Lua調用C模塊"
description: ""
category: programming
tags: [gamedev, lua]
---
這幾天剛好放假，利用這時間看了下《Programming In Lua》這本書，雖然公司新項目中使用了slua來做Unity的更新，但是自己對lua和其他語言的交互還不是很清楚，今天下午在做書裏的練習題時遇到了些問題，在此記錄下，順便對自己的理解做個梳理。

先回答幾個問題：

1. lua是如何知道所要調用的C函數是在哪裏的？
2. 所要調用的C函數的參數和返回值是怎麼知道的？

先來回答第一個問題，lua是通過`require`  來尋找所要調用的C函數的，`require`指定了所要尋找的C模塊的名字，lua通過預定義的路徑`package.cpath` 來尋找匹配的`.so`  或者`.dll` 文件，如果找到了，就在這個動態鏈接庫裏面去尋找函數`luaopen_xxx` 函數，註冊它爲一個lua中的C函數，並且去執行它，這裏的xxx就表示C模塊的名字。通常來說，在這個open函數中會調用`luaL_newlib` 去註冊所有的C函數。至於lua怎麼去加載so文件的，找到響應的函數的，搜索`dlopen，dlsym` 即可知道了。順便推薦《程序員的自我修養》這本書。

在實際寫代碼的過程中遇到一個問題，我記得linux中是通過命令`gcc -fPIC -shared -o xx.so`  來創建動態鏈接庫文件的，可是我在OSX 10.11 下就是會報錯誤：

```c
➜  lua_c_module gcc -fPIC -shared lua_sum.c -o summation.o
 sumation.lua+                                                                                                         
Undefined symbols for architecture x86_64:
  "_luaL_error", referenced from:
 summation.lua                                                                                                         
      _summation in lua_sum-e15a34.o
  "_luaL_setfuncs", referenced from:
      _luaopen_summation in lua_sum-e15a34.o
  "_lua_createtable", referenced from:
      _luaopen_summation in lua_sum-e15a34.o
  "_lua_gettop", referenced from:
      _summation in lua_sum-e15a34.o
  "_lua_isnumber", referenced from:
      _summation in lua_sum-e15a34.o
  "_lua_pushnumber", referenced from:
      _summation in lua_sum-e15a34.o
  "_lua_tonumberx", referenced from:
      _summation in lua_sum-e15a34.o
ld: symbol(s) not found for architecture x86_64
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

能看出來是鏈接錯誤，而且這些函數都是lua的API函數，難道需要鏈接lua庫文件才行？

google了一下，找到答案[how to compile shared lib with clang on osx](http://stackoverflow.com/questions/21907504/how-to-compile-shared-lib-with-clang-on-osx) , 不曉得爲啥要加 `-undefined dynamic_lookup` 

好了，再來回答第二個問題，所要調用的C函數的參數和返回值是怎麼知道的？簡單來說，寫模塊的人告訴你的😂

詳細來說，是通過一個**虛擬棧** 來實現的。lua和C的交互，需要傳遞的數據都是一方先壓入這個棧中，然後另一方在這個棧中去取。

那麼是不是如果從Lua中調用一個C 模塊的函數之前要提前將所需要傳遞的參數壓入這個虛擬棧中呢？是的，但是你不必手動做，因爲當你調用C模塊的函數時,你所傳遞給函數的參數都會被按照順序壓入這個棧中。雖然C 函數的原型都是只接受一個 `lua_State * L` 的參數，但是在Lua中，你可以傳遞任意多個參數給這個函數，所傳遞的參數的個數可以通過函數 `lua_gettop()` 來獲得。

那麼，C函數的返回值類型和個數是怎麼讓lua中的調用者知道的呢，答案也是通過虛擬棧來實現的。C函數將所要返回的所有值都壓入棧中，這裏可不是只能返回一個值。然後C函數返回一個整數，表示自己所要返回給調用者的結果的個數。這些信息都可以通過lua的API 文檔得知，搜索 [`lua_CFunction`](https://www.lua.org/manual/5.2/manual.html#lua_CFunction) 即可知道。值得注意的地方是，返回值的個數是根據C函數的return值來確定的，但是你可以往棧上push多個值，多餘的值會被lua丟棄掉。

