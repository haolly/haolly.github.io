---
layout: post
title: "C語言面試題"
description: ""
category: 生活
tags: [linux, c]
---

這幾天去了幾家公司面試，做了一些題，但是有些知識點好久沒有用就給忘掉了，所以寫篇博客來記錄下最近面試題中容易忘掉的知識。

第一題:   
`int i; (!! i) `的值是什麼？   
**表達式最後的結果是一個bool值，如果i 是全局變量，默認初始化爲0，那麼結果就是0，如果 i 是局部變量，初始化時是不確定的，所以一般爲 1** 

第二題:
  
```c
int a = (int*)((void*)0) + 5;   
a = ?
```   
**這個涉及到指針的運算，指針和整數相加時的步長是和指針所指向的數據類型相關的，具體到這裏就是 5 個 int的大小 20字節，所以在地址0的基礎上加20，最後的結果就是20**   
**另外再補充一點，對於二位數組 int x[2][3], &x[0] + 1, x + 1, 它們都是將 地址移動一行，既3 * sizeof(int), 而 &x + 1 是移動整個數組的大小，雖然 x, &x[0], &x 它們的數值都是一樣的，但是它們的語義不一樣**

第三題:

```c
int a = 0x12345678;
char b = (char)a;
b = ?
```
**char的大小爲1字節，所以取值爲0x12或者0x78，具體取決於所使用的機器的endian，是這樣子的嗎？剛開始我也覺得是，後面google了下發現不是。不管[大端還是小端](https://en.wikipedia.org/wiki/Endianness) 最後都是0x78**
**順便發現一個有用信息，bit-shift 操作也是和endianness無關的，參考SO鏈接http://stackoverflow.com/questions/7184789/does-bit-shift-depend-on-endianness**   
但是對於指針的轉換就有所不同，將int\*轉換爲char\* 就會出現關於endianness的問題。
*額外參考以下回答http://stackoverflow.com/questions/1041554/bitwise-operators-and-endianness (1800 INFORMATION)*   
*http://stackoverflow.com/questions/7504277/int-to-char-casting (Dmitri)*

第四題:

```c 
struct {
	int a;
	char b;
	short c;
	short x;
	long d;
	char e;
}s;
sizeof(s) = ?
```
**1.規則1，結構體中每個基本類型都有其自身的對齊要求，char可以對齊到任意地址，short必須是2的倍數，int 必須是4的倍數，指針必須是4（或者8在64爲機器上）的倍數。所以s中爲了對齊到short c，b必須填充一個字節,爲了對齊long(8 byte in 64 system)，x需要填充6個字節，這樣d的地址才是8的倍數。  
2.規則2，對於結構體中最後一個成員，它的對齊要求是要使整個機構體的大小是最大對齊成員的整數倍。所以s中e爲了對齊最大的long（8字節對齊在64爲系統中）的整數倍，需要填充7字節，4\*8=32。**最後，sizeof(s)= 4 + (1 + 1) + 2  + (2 + 6) + 8 + (1 + 7) = 32;   
上面的規則是我自己總結的，其實可以參考[wikipedia](https://en.wikipedia.org/wiki/Data_structure_alignment) 和 [The Lost Art of C Structure Packing](http://www.catb.org/esr/structure-packing/)。不過記住以上兩點基本就可以了。    
再補充：可以使用**-Wpadded**編譯參數來查看編譯器的對齊填充效果，clang有這個選項，不知道gcc有沒有
待補充：如果結構體中使用位域大小限定。==TODO==

第五題:

```c
void getIntFromFunc(int* p)
{
	p = malloc(sizeof(int));
}
int main()
{
	int* a = NULL;
	getIntFromFunc(a);
	printf("%d\n", *a);
}
```
**這裏考察的是c語言中指針的傳遞方式，指針本身也是一個數值，在32位系統中它的值大小爲4位，其本身就是一個整數，(所以你可以用整數來表示指針),例如0x12ff0011，在函數參數傳遞過程中，指針本身也是作爲值來傳遞的，就像一個int一樣，也是一個副本。但是在函數中對指針解引用操作時，它所修改的就是這個地址所指向的內存，例如`*a=3;`, 但是`a=malloc(4);` 的意思卻是將指針指向領一塊內存，它只是修改來這個臨時的整數副本而已**   
更新：其實將getIntFromFunc的參數改爲**p就可以了。參考函數`posix_memalign`的申明。

第六題：   
進程和線程的區別是什麼，各自有什麼優勢，使用的合適時機    
**這個問題很常見，被問到好幾次   
==TODO==**

第七題:    
tcp的建立和斷開   
**網上有一個比較好的文章可以[參考](http://blog.csdn.net/lostyears/article/details/7104349)**   
斷開鏈接時比建立鏈接時多一次握手主要是因爲建立鏈接時服務器可以將SYN和ACK合在一起發送，而斷開的時候ACK和FIN是不能合在一起的，因爲有可能還有數據沒有寫完，要等到寫完數據後再發送FIN。

第八題：   
一個宏`IS_POW_OF_2(X)`  
**使用`do ... while(0)` 技巧，不然將宏替換后可以出現一些語法錯誤或者執行了非預期的代碼，記住while(0)後面沒有分號。詳細可以參考鏈接http://www.bruceblinn.com/linuxinfo/DoWhile.html**

第九題：

```c
union A
{
	char a[10];
	int b;
}
sizeof(A) = ?
```
這道題和第四道題類似，長度爲**union中最大成員長度，並且是最大*變量*長度的整數倍，**,所以上面的結果是10 => 12(4*3) = 12

第十題：

```c
char* a = "abc";
a[2] = 'x';
printf("%s", a);
這段程序有問題嗎？
```
運行這段程序出現bus error，**因爲字符串字面常量是只讀的，不能修改，[參考SO](http://stackoverflow.com/questions/7480597/why-cant-i-edit-a-char-in-a-char)**

第十一題：

```c
vector刪除元素
```

第十二題：

```c
map, vector, set, list, deque中，哪個不能用sort 函數？
```