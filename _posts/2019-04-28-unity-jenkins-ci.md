---
layout: post
title: Unity Jenkins CI
tags: [dev]

---

## 主要干什么？
1. 打脚本和配置表更新
2. 打资源更新
3. 打 bundle
4. 打 bundle patch
5. 打 APP


## 为什么要用 Jenkins ？
自动化打包流程，避免手动操作带来的人为失误；提升打包效率；可扩展；

## 目前的实现方式
Python + C#

## 打 APP和资源不就是一行代码调用unity 接口就行了吗，干嘛要整这么繁琐？
因为要区分 release和 debug，还有区分不同的证书（inhouse 和 正式），区分内网和外网不同的更新地址配置，区分不同的 OS，区分不同的开发分支（Dev 分支和 Release 分支），打 App 时是否更新 C#代码，是否拷贝资源更新到App内置，区分不同的版本（大陆、港台等）等等，这些都是需要预处理，所以采用 Python 脚本来做这些事情

## iOS 打包准备
证书，provisioning file，bundle name，来签名；
用到的工具：
* https://github.com/kronenthaler/mod-pbxproj
* https://docs.python.org/2/library/plistlib.html
* https://docs.unity3d.com/ScriptReference/iOS.Xcode.PBXProject.html，不推荐
命令行参数：
* `xcodebuild -project {}/Unity-iPhone.xcodeproj -scheme {} -configuration {} clean build archive -archivePath {}/target.xcarchive'.format(
                PROJ_PATH, SCHEME, CONFIG, ARCHIVE_PATH)`
* `xcodebuild -exportArchive -exportOptionsPlist {} -archivePath {}/target.xcarchive -exportPath {}'.format(
            EXPORT_OPTION_PLIST, ARCHIVE_PATH, EXPORT_PATH)`

其中 EXPORT_OPTION_PLIST 可以在第一次手动打包的时候生成；TODO: 这里的 plist 和 merged plist 分别起什么作用？
当然，这里的前提是已经使用 unity 接口 build 出 iOS 工程。
还有，IPA 签名的时候会需要登录密码，可以在脚本里面通过命令 `security unlock-keychain -p "password" ~/Library/Keychains/login.keychain' ` 来提前解锁。
一般第三方 SDK 都会提供自己的 plist 文件，可以在 unity 生成 ios 工程后将其 merge 到 Info.plist 中。


## 用到的 Jenkins 插件
* https://wiki.jenkins.io/display/JENKINS/Conditional+BuildStep+Plugin
* https://wiki.jenkins.io/display/JENKINS/Token+Macro+Plugin
* https://wiki.jenkins.io/display/JENKINS/Parameterized+Build
* https://wiki.jenkins.io/display/JENKINS/Active+Choices+Plugin
* https://wiki.jenkins.io/display/JENKINS/Parameterized+Trigger+Plugin
* https://wiki.jenkins.io/display/JENKINS/Throttle+Concurrent+Builds+Plugin
没有用到 Jenkins 的 pipline，直接 freestyle，简单粗暴。。。

## C# 插件
[XUPorter](https://github.com/onevcat/XUPorter), 但是这个插件很久没有更新了，添加第三方库的功能不好使，需要按照 issues40 (https://github.com/onevcat/XUPorter/issues/40) 里面说的修改下。
推荐直接用[pbxproj](https://pypi.org/project/pbxproj/) 

## cygwin 中 rsync 资源更新到服务器遇到的问题
1. 权限问题，rsync 到Linux机器之后权限不对， edit C:\cygwin64\etc\fstab add `noacl`, 最后的结果就是 `none /cygdrive cygdrive noacl,binary,posix=0,user 0 0`
2. ssh key 问题 https://superuser.com/questions/1296024/windows-ssh-permissions-for-private-key-are-too-open
3. https://superuser.com/questions/1026073/cygwin-ssh-not-reading-config-file/1026123#1026123
4. win10 自带ssh client 和 cygwin中的不兼容，所以不能使用自带的

## 重要的命令行参数
` -quit -batchmode -nographics -executeMethod AssetBuilder.StaticBuildMethod  -logFile logfile.log -projectPath ProjectRoot `

