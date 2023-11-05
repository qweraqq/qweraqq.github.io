---
layout: post
title: "Android packer behaviours"
date: 2023-10-06 00:00:00 +0800
author: xiangxiang
categories: android
tags: [android packer]
---
安卓APP的壳有哪些功能

* auto-gen TOC:
{:toc}
## 1 Anti-Debugging
1. `Debug.isDebuggerConnected`
2. `fork` and `ptrace`
3. process status `/proc/{pid}/status`
4. hook system functions releted to debugging


## 2 Anti-Emulator
1. inspecting specific system files (`/proc/tty/drivers`)
2. checking the existence or values of particular system properties (`init.svc.qemud`)
3. checking the existenc of hardware sensors

## 3 Anti-DBI
1. `/proc/{pid}/maps`

## 4 Time Checking
1. Calculating the time consumed for executing a specical task


## 5 System Library Hooking
1. GOT/PLT hooking
2. inline hooking
   
- libart.so
- libc.so
- loblog.so


## 6 Dynamic Dex File Loading
1. class loader
2. DexFile class
3. call methods in the DexPathList

related system functions

### 7 Dynamic Dex Data Modification
1. JNI_OnLoad
2. ClassLinker::LoadClass


## 8 Dynamic Runtime Object Modification
1. hook `ClassLinker::LoadMethod` to modify `ArtMethod` objects
2. static initialization method of each app class to modify the relevant `mirror:Class` object


## 9 Dex Data Fragmentation
1. release the protected Dex file's `class_data_items` or `code_items` to separated memory regions


## 10 JNI Transformation
1. VMP
2. DEX2C


动态环境检测, properties举例
[https://github.com/maddiestone/IDAPythonEmbeddedToolkit/blob/master/Android/WeddingCake_sysprops.txt]([https://github.com/maddiestone/IDAPythonEmbeddedToolkit/blob/master/Android/WeddingCake_sysprops.txt)

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

- Xposed 

```
Checks if LIBXPOSED_ART.SO or XPOSEDBRIDGE.JAR exist in
/proc/self/maps

Tries to find either of the following two classes using the JNI
FindClass() method
○ XC_MethodHook: de/robv/android/xposed/XC_MethodHook
○ XposedBridge: de/robv/android/xposed/XposedBridge
```

## refs
- [Happer: Unpacking Android Apps via a Hardware-Assisted Approach](https://yajin.org/papers/sp21_happer.pdf)