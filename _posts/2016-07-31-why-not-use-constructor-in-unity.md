---
layout: post
title: "Unity 中的 Serialization 和 Constructor"
description: ""
category: programming
tags: [gamedev, Unity]
---
最近发现公司 Unity项目中的代码有时候会出现一个错误，而且 Unity Editor 会卡死十几秒，控制台输入类似下面这样：

![unity-serialize](/assets/picture/201607/unity-serialize.png)

这个是我在家里用 Unity4.6.8出现的结果，公司里面是4.6.4，有一点点差异，但是意思都是可能出现了循环。

下面是我的测试代码：

```C#
[Serializable]
public class Book
{
	List<Book> books;
}
```

这个类里面的 books 本来应该是为 null 的，但是由于 Unity 的 Serialization 不支持 null，它会给 books new 出一个 List 来，所以，books 中就有了 Book，而 Book 中又有 books，于是就出现了循环。

下面是 Unity 的官方文档所说：
> The serializer does not support null. If it serializes an object, and a field is null, we just instantiate a new object of that type, and serialize that. Obviously this could lead to infinite cycles, so we have a relatively magical depth limit of 7 levels. At that point we just stop serializing fields that have types of custom classes/structs and lists and arrays.

当然，文档里面还说了好几点比较重要的内容，例如不支持引用和多态等等，这些可能也是 Unity 的坑，只是暂时没有遇到而已。而且这样解释了我的一个疑惑，当你在几个不同的MonoBehaviour 类中用拖拽的方式引用同一个 MonoBehaviour 类的时候，这几个 MonoBehaviour 是引用了同一个类呢还是说各自都有一个相应的实例？通过文档可以得出结论，引用的是同一份实例，因为从 UnityEngin.Object 继承而来的类在 Serialize 的过程中都是保持引用的完整性的。

基于上面对 Unity 中对象序列化的理解，我也终于知道了困惑已久的问题，为什么在 Unity 的 MonoBehaviour 脚本中不使用构造函数，而要用 Awake、Start 等函数，原因有两方面，第一是因为 Unity 在 创建 MonoBehaviour脚本的时候就会执行构造函数，但是这时候整个 Scene 文件还没有加载完成，所以这时候去调用 Unity API 就会出现问题。第二点是因为 Unity 在 Deserialize 的时候会调用对象的构造函数，也就是第一点所说，但是serialize 和 Deserialize 在 Unity 中不是只执行一遍的，当你点击 Play、Stop、Save 等等的时候都有可能，而且在内部也是，看下面的图就知道：**A picture speaks thousands words!**

![unity-serialize-internal](/assets/picture/201607/unity-internal-serialize.png)
