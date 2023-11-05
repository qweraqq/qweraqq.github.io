---
layout: post
title: "Android Dex Dumper"
date: 2023-11-05 00:00:00 +0800
author: xiangxiang
categories: android
tags: [android packer]
---
如何实现一个自定义rom的脱壳工具


## 0 参考材料

- [https://github.com/AlienwareHe/RDex](https://github.com/AlienwareHe/RDex)

- [https://github.com/dqzg12300/FartExt](https://github.com/dqzg12300/FartExt)

- [Fart说明](https://bbs.kanxue.com/thread-252630.htm)

- [看雪FartExt说明](https://bbs.kanxue.com/thread-268760.htm)

- [最早的脱壳dexhunter](https://github.com/zyq8709/dexhunter)

## 1 加壳的原理(非VMP)
- APP启动
- 自定义Application attachBaseContext
    + 解密源程序
    + 初始化自定义Classloader
    + 通过反射设置LoadedApk中加载器对象为自定义的Classloader
- 自定义Application onCreate
    + 获取源程序中的Application
    + 通过反射生成正确的Application对象
    + 反射设置ActivityThread
- Activity加载流程且源程序正常运行


- 当壳在函数attachBaseContext和onCreate中执行完加密的dex文件的解密后，通过自定义的Classloader在内存中加载解密后的dex文件。为了解决后续应用在加载执行解密后的dex文件中的Class和Method的问题，接下来就是通过利用java的反射修复一系列的变量。其中最为重要的一个变量就是应用运行中的Classloader，只有Classloader被修正后，应用才能够正常的加载并调用dex中的类和方法，否则的话由于Classloader的双亲委派机制，最终会报ClassNotFound异常，应用崩溃退出，这是加固厂商不愿意看到的。由此可见Classloader是一个至关重要的变量，所有的应用中加载的dex文件最终都在应用的**Classloader**中


- 随着加壳技术的发展，为了对抗dex整体加固更易于内存dump来得到原始dex的问题，各加固厂商又结合hook技术，通过hook dex文件中类和方法加载执行过程中的关键流程，来实现在函数执行前才进行解密操作的指令抽取的解决方案。此时，就算是对内存中的dex整体进行了dump，但是由于其方法的最为重要的函数体中的指令被加密，导致无法对相关的函数进行脱壳。脱壳工具可以通过欺骗壳而主动调用dex中的各个函数，完成调用流程，让壳主动解密对应method的指令区域，从而完成对指令抽取型壳的脱壳


## 2 整体思路
- 核心是FART的思路, FART有一个XPosed的实现版本RDex
- 主动调用应对指令抽取不需要, 我们可以 延迟 + 主动点击应用(半自动化)实现

## 3 具体实现:
1. 参考FartExt在ActivityThread中的handleBindApplication启动**dump线程** -> 参考FartExt
2. **dump线程**通过反射获取mCookie, **根据classLoader->pathList->dexElements->dexFile->mCookie**  -> 参考RDex
3. 在framework层的DexFile类中添加Native函数`dump`供调用, `dump`作用就是将dex保存下来, 具体实现需要修改`art/runtime/native/dalvik_system_DexFile.cc` -> 参考Fart
