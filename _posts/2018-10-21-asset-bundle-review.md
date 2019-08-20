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

首先要考虑的是哪些资源文件打到同一个 Bundle 里面去，如何做拆分，这里因人而异。有的是按照目录结构，同一个目录下面的所有文件打包到同一个 Bundle 里面去，以前使用的 KEngine 就是这种策略，至少那时候是；有的是按照资源类型，例如每个 UI Prefab 打成一个 Bundle，所有的字体和 Shader 分别打包成一个 Bundle；有的按照使用场景，例如同一个NPC 打包成一个 Bundle，一个副本场景Scene 打包成一个 Bundle；我们把这些文件叫做 root 文件，因为使用中他们往往就是需要 `LoadAsset` 出来使用的；实际项目中往往是多种并存的，例如一个配置文件 A 目录下面的 *.prefab 文件都是 root 文件。

实际项目中是不可能手动去 editor 中设置 bundle name 的，一般会用AssetImporter 或者AssetBundleBuild 类来在代码里面设置 bundle name，后者用的比较多。


这里需要注意的就是资源的重复打包，unity manual 里面说：

>  A dependency does not occur if the `UnityEngine.Object` contains a reference to a `UnityEngine.Object` that is not contained in any AssetBundle. In this case, a copy of the object that the bundle would be dependent on is copied into the bundle when you build the AssetBundles. If multiple objects in multiple bundles contain a reference to the same object that isn’t assigned to a bundle, every bundle that would have a dependency on that object will make its own copy of the object and package it into the built AssetBundle.

所以我们需要在打包的时候解决这个问题。怎么解决呢？将被依赖的文件也打包就行了。在 unity 里面可以利用函数 `EditorUtility.CollectDependencies(objs.ToArray())` 来获取当前文件的依赖项，注意不是函数 <del>`AssetDatabase.GetDependencies`</del>，这个函数有问题，ref https://gist.github.com/QXSoftware/35a07738f481245d08b948ead3743a4b. 其中 `MonoScript` 类型的依赖可以排除掉。

meta 文件是 editor 下记录引用关系的，bundle 里面不需要用到，这一点我刚开始比较困惑，以为 bundle 里面也是这样记录的。

这里可以抽象出一个类 `BundleCandidate`，每一个Asset 都是`BundleCandidate`，这个类记录了她所依赖的其他`BundleCandidate` 和依赖于她的`BundleCandidate`，这个类有三种类型，root，rawAsset，standalone， 当依赖于她自己的`BundleCandidate` 数目大于1的时候，rawAsset 类型提升为 standalone 类型，表示需要单独打AB包，root 类型也需要单独打 AB 包。

最后需要注意的一点就是，开发环境一般用Resources 目录的方式加载资源，因为这样快速，不用每次修改后去打 bundle 才能看到最终效果，正式版使用 bundle 方式加载资源，所以，游戏中要做一个资源加载的抽象接口，有 Resources 和 Bundle 两种实现类型；打包 APP 的工程一般会和开发工程不是同一个，打包 APP 的工程 Resources 目录只保留游戏启动所需的最少的资源。



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

首先找出哪些文件是新增的或是有修改过的，这个可以通过项目 SVN 记录来找到，当前的 SVN commit version 和项目正式上线时候的 SVN commit reversion 做对比 ，可以通过第三方库例如 [pysvn](<http://pysvn.tigris.org/>) 来简化工作。项目上线之时的 commit reversion 会是一个固定值。
如果一个UI prefab 依赖于一个 atlas shared.png, 这个 shared.png 再后面的版本中有修改，那么这个修改不会导致原来的 ui prefab 发生改变；
一个角色材质的改变也不会导致使用这个材质的角色prefab 发生改变；记录sprite 的.asset 文件(ie, catalog) 也不会因为原始图片的改变而改变；所以 patch 的时候需要找到所有依赖于这些文件的父文件，对其重新打包。
主要分为两步:

##### Step One， 找出被谁依赖
<pre>
foreach file in changeSet:
    如果是root文件
        打这个文件的 patch
    遍历所有的 bundle，找出依赖于这个文件的 所有的bundles A，遍历 A
        找出这个 bundle 中依赖于这个文件的文件B
        	打这个文件B的 patch
</pre>
root文件就是需要被第一次bundle 打进去的文件，可能会分布在不同的assets子目录中，并且会有不同的命名规则，例如texture atlas 和ui prefab ，ui  atlas 等；新增加的root文件会根据规则打进一个新包或者作为依赖被添加。changeSet 就是通过 svn 差异对比出来有变化(新增，修改)的文件集合。


##### Step Two, 找出依赖于谁，判断它所依赖的文件是否需要重新打 patch bundle
<pre>
打某个文件file的 patch:
找出或创建这个文件file所对应的 patch bundle P。[NOTE 1]
找出这个文件file所对应的 bundle oldP，找最接近的
找出这个文件file的所有依赖项file_dep，遍历
	1. 如果依赖项在changeSet 里面
		如果依赖项是顶层文件
			不用管，因为 修改的root 文件会自己打一个patch bundle。TODO: 这里没有手动记录依赖
			or，如果是贴图或者动画，单独打一个 bundle 作为 P 的依赖 bundle
		如果依赖项不是顶层文件
			加入到 P 里面
			or，如果是贴图或者动画，单独打一个 bundle 作为 P 的依赖 bundle
	2. 如果依赖项没有在changeSet 里面，即在旧包里面；按理来说，直接找出这个旧包作为P 的依赖就行了，但是，我们不能确定是否这个旧包的依赖发生变化导致这个旧包需要重新打 bundle；设 依赖项file_dep为 xOld (这里就是要检查依赖项 xOld 是否要被打包)
		2.1 找出 xOld 的依赖项file_dep_dep，遍历
			如果依赖项在 changeSet 里面，即依赖的依赖有修改
				如果xOld是顶层资源 (root 文件)
					给 xOld 单独打一个patch bundle，并作为 P 的依赖 bundle。[NOTE 2]
				如果xOld不是顶层资源: 
					加入到 P 里面
				dirty = true;
				break，goto 2.2； 即，只要依赖的依赖中有修改，依赖就会被重新打包
			如果依赖项没有在 changeSet 里面
				do nothing
		2.2 如果not dirty, 即xOld(ie file 的依赖文件 file_dep) 的所有依赖项都在旧包里面并且没有发生变化（更新 patch 对旧 bundle 的引用，**重点**）[NOTE 3]
				2.2.1 如果oldP 存在，即找到file 所在的 bundle，并且 oldP 包含xOld，不管是直接包含或者是间接依赖包含。(怎么知道 oldP 包含 xOld 呢？当然是利用已有的 bundle 信息啦)
					把 oldP 加入到 bundle build list里面。[NOTE 4]
					把oldP 作为 P 的依赖 bundle
				2.2.2 如果上一步不成立，即没有找到oldP 或者oldP 不包含 xOld。执行，如果 oldP 存在，并且 oldP 依赖的 bundle 中的某一个包含了 xOld，那么把这个依赖 bundle 作为P 的依赖 bundle。
				2.2.3 如果上面两个都不成立。找出 xOld 所对应的 bundle oldBDX，执行类似 2.2.1/2.2.2
				2.2.4 如果找不到 oldBDX，说明 xOld 不是 root 文件，是贴图、动画等原子文件
					2.2.4.1 如果是纹理或者 fbx，打 patch bundle 并加入 P 的依赖 ，如果不是，直接加入 P 中。


</pre>

[NOTE 1]：这一步是指找到 file 对应的 old bundle name，如果找到，新的 patch bundle name 就是 old bundle name 加后缀 .patch，如果没有找到，就创建对应的 .patch 文件按照起名规则；这里的新增一个 patch bundle 是指新增一个 bundle info，用来记录 bundle 信息的，包括它的名字和包含有哪些文件

[NOTE 2]：对于 root 文件，如果能找到其所对应的 old bundle name，就 用加.patch 后缀的形式命名新的 patch bundle，如果不能，就按照规则来命名。

[NOTE 3]：新的patch 要对旧bundle产生依赖，如果旧bundle 不和 patch一起重新打，path就不能记录到对旧bundle的依赖，因为没有办法去修改patch内部的依赖记录去让他指向旧bundle，而且，如果旧bundle 不重新打，这个依赖就会被重复包含在patch里面；
需要注意的是，旧bundle的重新打，需要名字和原来的一样。

[NOTE 4]：打 patch 的时候，跟 old bundle 名字一样的.bundle 文件是从哪一步出来的？ 就是这一步，找到原始的 bundle，并且把原始 bundle 所包含的文件加入到 同名的 bundle 里面去，还原 old bundle 的环境。最后，这些同名的bundle 只是为了更新引用，不会上传到服务器。

我们项目中使用 `ScriptableObject` 来存储 `Sprite` 信息，类似于一个 catalog，这个 `ScriptableObject` 是在编辑器中创建的，
例如 catalog.asset, 它记录了哪个 sprite 在哪个 png 文件中，以用于**运行时加载**。注，运行时加载 .asset 文件即可。

对于patch bundle，导出一个类似于 manifest 的文件，例如 patch_res.json，记录所有bundle 的文件名和md5 值，然后上传到服务器，每次游戏启动的时候就去检查有没有新的 patch bundle 可以下载，本地已经有的 patch bundle 全不全，md5 值是不是一样，如果不全或者md5 值不一样，则需要重新下载这个 patch bundle。

最后，自己手动记录的依赖信息可能不是很准确，所以可以通过 打 bundle 生成的 manifest 来修正。


## 怎么知道哪个文件在哪个 bundle 里面？
打完 bundle 需要导出一份文件，里面记录了**每个 bundle** 所包含的文件，所以加载文件的时候就可以反向查找 bundle，然后加载 bundle，再加载文件了。


