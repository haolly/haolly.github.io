---
layout: post
title : AssetBundle 在项目中的使用
tags : [gamedev]
---
主要代码在KAssetBundleLoader文件中  
HotBytesLoader 用于下载字节文件

**加载Bundle A流程**
1. 用byteloader 加载对应平台的 **AssetBundleManifest** 
  ref: "load that additional AssetBundle (the one that’s named the same thing as the folder it’s in) and load an object of type AssetBundleManifest from it."
2. 用AssetBundleLoader加载依赖的bundle B
3. 第二步完成后用byteloader 加载 A 的字节文件
4. LoadFromMemory 同步或者异步加载Bundle 
5. 这个bundle 中只有一个和bundle名字相同的asset, 也就是一个prefab
6. 如果bundle C 也依赖于bundle B，那么加载C的时候，B的引用计数加一，而不是去重复加载B，直接返回bundle B 中加载的asset
7. 所以需要一个记录已经加载完的Bundle 和其url的对应关系

**卸载Bundle 流程<a id="destroy"></a>**
1. bundle.Unload(false) 卸载bundle 本身资源
2. 将从bundle load出来的 asset 全部销毁
3. 何时卸载？引用计数怎么用？
4. 定时回收标记为不用的bundle资源，这样从这个bundle里面加载的asset都会被销毁
5. 调用Release 后会将引用计数减一，当引用计数为零的时候，标记可以清除
6. 何时清除？ref [怎么用](#usage)

**构造Bundle**
1. 每一个canvas 打包成一个bundle
2. 所有的sprite pack 成atlas, 然后将atlas 打包成bundle
3. 如果bundle A 和 bundle B 共同依赖于 sprite C, 而sprite C 打包在一个common 的atlas图集中，
  这样bundle A 和 bundle B 共同依赖于bundle common 

**atlas 打包**
1. 用一个catalog bundle作为目录记录所有的sprite 包含在哪个bundle中
2. 将这个catalog 创建为一个prefab， 这个bundle中也只有一个prefab，
  用AssetDatabase.FindAssets() 找到指定目录的所有Texture2D类型的Asset，
  对于每一个assetPath，他的bundle 名字就是相对目录名，
  对于每一个assetPath，LoadAllAssetAtPath找到所有的Sprite类型的asset，放入这个bundle的spriteDepot 仓库中，
  对于每一个assetPath，创建一个prefab。createPrefab()

**怎么用<a id="usage"></a>**

1. 每一个UI界面都是一个Prefab，被打包成bundle。打开UI时直接去加载这个bundle，加载完卸载bundle本身
2. 关闭UI界面的时候直接Destroy掉所加载的gameObject。
3. UIModule 保存了一个界面的template，所以第二次打开的时候直接复制template gameObject，第一次打开的时候才去真正加载bundle
4. 这个template 什么时候卸载呢？定时执行回收 ref [卸载](#destroy)

