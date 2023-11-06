---
layout: post
title: "Android Packer & Dex Dumper"
date: 2023-10-06 00:00:00 +0800
author: xiangxiang
categories: android
tags: [android packer dex]
---

How Android packer works & Dump dex with custom Android rom

* auto-gen TOC:
{:toc}

## 0 参考材料
- [Happer: Unpacking Android Apps via a Hardware-Assisted Approach](https://yajin.org/papers/sp21_happer.pdf)
- [https://github.com/AlienwareHe/RDex](https://github.com/AlienwareHe/RDex)
- [https://github.com/dqzg12300/FartExt](https://github.com/dqzg12300/FartExt)
- [Fart说明](https://bbs.kanxue.com/thread-252630.htm)
- [看雪FartExt说明](https://bbs.kanxue.com/thread-268760.htm)
- [最早的脱壳dexhunter](https://github.com/zyq8709/dexhunter)

## 1 Android Packer
### Anti-Debugging
1. `Debug.isDebuggerConnected`
2. `fork` and `ptrace`
3. process status `/proc/{pid}/status`
4. hook system functions releted to debugging


### Anti-Emulator
1. inspecting specific system files (`/proc/tty/drivers`)
2. checking the existence or values of particular system properties (`init.svc.qemud`)
3. checking the existenc of hardware sensors

### Anti-DBI
1. `/proc/{pid}/maps`

### Time Checking
1. Calculating the time consumed for executing a specical task

### System Library Hooking
1. GOT/PLT hooking
2. inline hooking
   - libart.so
   - libc.so
   - loblog.so

### Dynamic Dex File Loading
1. class loader
2. DexFile class
3. call methods in the DexPathList

### Dynamic Dex Data Modification
1. JNI_OnLoad
2. ClassLinker::LoadClass

### Dynamic Runtime Object Modification
1. hook `ClassLinker::LoadMethod` to modify `ArtMethod` objects
2. static initialization method of each app class to modify the relevant `mirror:Class` object

### Dex Data Fragmentation
1. release the protected Dex file's `class_data_items` or `code_items` to separated memory regions

### JNI Transformation
1. VMP
2. DEX2C

### 动态环境检测
- properties举例
- [https://github.com/maddiestone/IDAPythonEmbeddedToolkit/blob/master/Android/WeddingCake_sysprops.txt]([https://github.com/maddiestone/IDAPythonEmbeddedToolkit/blob/master/Android/WeddingCake_sysprops.txt)

```
System Property Checked,Value that Causes Exit
init.svc.gce_fs_monitor,running
init.svc.dempeventlog,running
init.svc.dumpipclog,running
init.svc.dumplogcat,running
init.svc.dumplogcat-efs,running
init.svc.filemon,running
ro.hardware.virtual_device,gce_x86
ro.kernel.androidboot.hardware,gce_x86
ro.hardware.virtual_device,gce_x86
ro.boot.hardware,gce_x86
ro.boot.selinux,disable
ro.factorytest,true OR 1 OR y
ro.kernel.android.checkjni,true OR 1 OR y
ro.hardware.virtual_device,vbox86
ro.kernel.androidboot.hardware,vbox86
ro.hardware,vbox86
ro.boot.hardware,vbox86
ro.build.product,google_sdk
ro.build.product,Droid4x
ro.build.product,sdk_x86
ro.build.product,sdk_google
ro.build.product,vbox86p
ro.product.manufacturer,Genymotion
ro.product.brand,generic
ro.product.brand,generic_x86
ro.product.device,generic
ro.product.device,generic_x86
ro.product.device,generic_x86_x64
ro.product.device,Droid4x
ro.product.device,vbox86p
ro.kernel.androidboot.hardware,goldfish
ro.hardware,goldfish
ro.boot.hardware,goldfish
ro.hardware.audio.primary,goldfish
ro.kernel.androidboot.hardware,ranchu
ro.hardware,ranchu
ro.boot.hardware,ranchu


# If any of these System Properties exist, the application exits
init.svc.vbox86-setup
qemu.sf.fake_camera
init.svc.goldfish-logcat
init.svc.goldfish-logcat
init.svc.qemud
```

### Xposed检测

```
Checks if LIBXPOSED_ART.SO or XPOSEDBRIDGE.JAR exist in
/proc/self/maps

Tries to find either of the following two classes using the JNI
FindClass() method
○ XC_MethodHook: de/robv/android/xposed/XC_MethodHook
○ XposedBridge: de/robv/android/xposed/XposedBridge
```

## 2 源资源加密壳的原理(非VMP)
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

## 3 APP脱壳整体思路
- 核心是FART的思路, FART有一个XPosed的实现版本RDex
- 主动调用应对指令抽取不需要, 我们可以 延迟 + 主动点击应用(半自动化)实现

## 4 APP脱壳具体实现
1. 参考FartExt在ActivityThread中的handleBindApplication启动**dump线程** -> 参考FartExt
2. **dump线程**通过反射获取mCookie, **根据classLoader->pathList->dexElements->dexFile->mCookie**  -> 参考RDex
3. 在framework层的DexFile类中添加Native函数`dump`供调用, `dump`作用就是将dex保存下来, 具体实现需要修改`art/runtime/native/dalvik_system_DexFile.cc` -> 参考Fart

