---
layout: post
title: Unity AssetBundle 再回顾
tags: [dev]

---

## 使用AssetBundle 主要的问题点在哪里？

1. 如何避免打包时的资源重复，即打包策略
2. Bundle 的加载和卸载时机，如何避免同一份Bundle/资源 被多次加载，即使用策略
3. 如何打patch更新，即更新策略

## 打包策略

首先要考虑的是哪些资源文件打到同一个 Bundle 里面去，如何做拆分，这里因人而异。有的是按照目录结构，同一个目录下面的所有文件打包到同一个 Bundle 里面去，上面提到的 KEngine 就是这种策略，至少那时候是；有的是按照资源类型，例如每个 UI Prefab 打成一个 Bundle，所有的字体和 Shader 分别打包成一个 Bundle；有的按照使用场景，例如同一个NPC 打包成一个 Bundle，一个副本场景Scene 打包成一个 Bundle；我们把这些文件叫做 root 文件，因为使用中他们往往就是需要 `LoadAsset` 出来使用的；实际项目中往往是多种并存的，例如一个配置文件 A 目录下面的 *.prefab 文件都是 root 文件。

实际项目中是不可能手动去 editor 中设置 bundle name 的，一般会用AssetImporter 或者AssetBundleBuild 类来在代码里面设置 bundle name，后者用的比较多。


这里需要注意的就是资源的重复打包，unity manual 里面说：

>  A dependency does not occur if the `UnityEngine.Object` contains a reference to a `UnityEngine.Object` that is not contained in any AssetBundle. In this case, a copy of the object that the bundle would be dependent on is copied into the bundle when you build the AssetBundles. If multiple objects in multiple bundles contain a reference to the same object that isn’t assigned to a bundle, every bundle that would have a dependency on that object will make its own copy of the object and package it into the built AssetBundle.

所以我们需要在打包的时候解决这个问题。怎么解决呢？将被依赖的文件也打包就行了。在 unity 里面可以利用函数 `EditorUtility.CollectDependencies(objs.ToArray())` 来获取当前文件的依赖项，注意不是函数 <del>`AssetDatabase.GetDependencies`</del>，这个函数有问题，ref https://gist.github.com/QXSoftware/35a07738f481245d08b948ead3743a4b. 其中 `MonoScript` 类型的依赖可以排除掉。

meta 文件是 editor 下记录引用关系的，bundle 里面不需要用到，这一点我刚开始比较困惑，以为 bundle 里面也是这样记录的。

这里可以抽象出一个类 `BundleCandidate`，每一个Asset 都是`BundleCandidate`，这个类记录了她所依赖的其他`BundleCandidate` 和依赖于她的`BundleCandidate`，这个类有三种类型，root，rawAsset，standalone， 当依赖于她自己的`BundleCandidate` 数目大于1的时候，rawAsset 类型提升为 standalone 类型，表示需要单独打AB包，root 类型也需要单独打 AB 包。

最后需要注意的一点就是，开发环境一般用Resources 目录的方式加载资源，因为这样快速，不用每次修改后去打 bundle 才能看到最终效果，正式版使用 bundle 方式加载资源，所以，打包 APP 的工程一般会和开发工程不是同一个，打包 APP 的工程 Resources 目录只保留游戏启动所需的最少的资源。



**NOTE:** 
默认情况下，builtin shader不会被单独打成一个bundle。如果多个 prefab 依赖于 default-mat，这个 default-mat 就会重复。
解决办法就是下载一份当前 unity 版本的 builtin shader 到工程目录，这样内置 shader 就会引用下载的 builtin shader。shader 最后单独打成一个 AB。


## 使用策略

* 原始包里面的 bundle 在目录 StreamAssetsPath/bundles 中，新下载的patch bundle 在 persistentDataPath/bundles 中
* 加载的时候根据文件名去确定是哪个 bundle，然后加载 bundle，loadAsset 资源；**NOTE** ： 如果是 prefab资源需要 Instantiate
* 会先去从 patch bundle 列表中找，找不到才去原始 bundle 中找
* 对已经加载的 bundle 做 cache，确保不会去加载第二遍
* **对加载的 bundle 和其所依赖的 bundle 做引用管理**
* **对从 bundle 中加载的文件做引用管理**
* 对每一个加载的 bundle，记录哪些文件对自身是依赖关系，包括直接和间接的
* 每新增一个依赖关系文件，refCount++。例如prefab UI A、B、C 都对 common.png 所打的 bundle 有依赖，所以common.png这个 bundle 的 refCount==3。这是 bundle 引用管理。asset 引用管理类似。
* 当卸载一个文件的时候，找到其依赖的 bundle，refCount--，当 refCount == 0 的时候，卸载整个 bundle
* 使用 `AssetBundle.Unload(true)`，参数不是 false
* load 同一个资源多次怎么办？bundle 有 cache，所以 bundle 不会重新加载，但是会去 loadAsset 多次，应该对每一个 asset 做引用关系管理

## 更新策略

首先找出哪些文件是新增的或是有修改过的，这个可以通过项目 SVN 记录来找到，当前的 SVN commit version 和上一个 SVN commit reversion ，可以通过第三方库例如pysvn 来简化工作。   
如果一个UI prefab 依赖于一个 atlas shared.png, 这个 shared.png 再后面的版本中有修改，那么这个修改不会导致原来的 ui prefab 发生改变；
一个角色材质的改变也不会导致使用这个材质的角色prefab 发生改变；记录sprite 的.asset 文件(ie, catalog) 也不会因为原始图片的改变而改变；所以 patch 的时候需要找到所有依赖于这些文件的父文件，对其重新打包。   
主要分为两步:
##### Step One， 找出被谁依赖
如果是root文件
​	打这个文件的 patch
​	遍历所有的 bundle，找出依赖于这个文件的 bundle
​		找出这个 bundle 中依赖于这个文件的文件
​			打这个文件的 patch
​	
如果不是顶层文件，（例如 Asset/Raw 等目录）
​	遍历所有的 bundle，找出依赖于这个文件的 bundle
​		找出这个 bundle 中依赖于这个文件的文件
​			打这个文件的 patch

##### Step Two, 找出依赖于谁
打某个文件的 patch:
找出这个文件所对应的 patch bundle P
找出这个文件所对应的 bundle oldP
找出这个文件的所有依赖项，遍历
​	如果依赖项在changeSet 里面
​		如果依赖项是顶层文件
​			不用管，因为已经在 bundle 里面了
​			or，如果是贴图或者动画，单独打一个 bundle 作为 P 的依赖 bundle
​		如果依赖项不是顶层文件
​			加入到 P 里面。（TODO， 这里会有重复）
​			or，如果是贴图或者动画，单独打一个 bundle 作为 P 的依赖 bundle
​	
​	如果依赖项没有在changeSet 里面，即在旧包里面，设 依赖项为 xOld  (TODO, 后面的应该不用管了)
​		找出 xOld 的依赖项，遍历
​			如果依赖项在 changeSet 里面，即依赖有修改
​				如果依赖项是顶层资源
​					给 xOld 单独打一个patch bundle，并作为 P 的依赖 bundle，被依赖的顶层资源就不用管了，反正已经打到 bundle 里面去了
​				如果依赖项不是顶层资源
​					给 xOld 单独打一个patch bundle，并作为 P 的依赖 bundle，被依赖的项加入到 patch bundle 中。（TODO, 这里会有重复）
​				dirty = true;
​				break;
​			如果依赖项没有在 changeSet 里面
​				do nothing
​		如果not dirty, 即xOld 的所有依赖项都在旧包里面
​			do nothing

对于patch bundle，导出一个类似于 manifest 的文件，例如 patch_res.json，记录所有bundle 的文件名和md5 值，每次游戏启动的时候就去检查有没有新的 patch bundle 可以下载，本地已经有的 patch bundle 全不全，md5 值是不是一样，如果不全或者md5 值不一样，则需要重新下载这个 patch bundle。