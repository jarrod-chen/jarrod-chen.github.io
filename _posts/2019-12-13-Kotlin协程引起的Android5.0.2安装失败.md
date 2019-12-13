
---
layout:     post
title:      Kotlin协程引起的Android5.0.2安装失败
subtitle:   
date:       2019-12-13
author:     Jarrod
header-img: img/post-bg-desk.jpg
catalog: true
tags:
    - Android 
    - kotlin
# Kotlin协程引起的Android5.0.2安装失败
## 问题
红米Note3 Android5.0.2版本测试机上，App安装不上。
## 原因分析
1. 找了几台其他版本的机器，Android 5.1；Android 7、8、9，都可以。所以可能只是这个机型或者系统版本上有问题，但是其他应用在红米Note3 上是可以正常安装的，但是我们的App不可以，所以一定是代码出问题了。
2. 找到历史发布版本，定位的是哪两个版本之间出的问题，发现之前某个A版本能正常安装，但是之后一个B版本就不行了，这中间花了好多时间去找。
3. 比对两个版本的代码，差距在一个C moudle上，v2.1.6中没有引用这个moudle，而后一个版本引用了。业务代码是不会引起这类问题的，所以关注重点放在gradle构建脚本上。发现好几个不同的地方，比如
 * buildToolVersion配置一个有一个没有
 * B版本用version.gradle进行了版本库统一配置，导致整个构建配置被大面积更改
 * C moudle是一个只有kotlin代码的库
4. 在还原buildToolVersion后依然不能正常安装，version.gradle应该是不会影响到安装的
5. 在仔细比对C moudle的build.gradle和app moudle的build有什么不同时发现唯一大的不同之处就是kotlin协程的使用，抱着试一试的态度，去掉协程的使用确实在红米上能安装了。
6. Google了kotlin协程会引起安装问题，果不其然官网上有反馈，也是Android 5.0.2

## 解决
kotlin协程在C moudle中并没有大面积使用，而是用在了一个倒计时功能，所以就找到对应得开发改了实现方式，抛弃了协程

## 总结
新技术的使用是每个开发者都想去尝鲜的，这个无可厚非，如果不去尝试就不能进步，这种隐蔽性的问题，一时也很难发现，如果不是一点点尝试去分析解决问题，谁也不会想到协程会引起安装问题，感觉像八竿子打不着。
这个问题的发现和解决花了一天的时间，所以才想记录下，有时候问题的解决并没有什么巧妙的方法，只是一点点花时间去找到问题出现的点，一点点尝试分析，试错也是一种学习。