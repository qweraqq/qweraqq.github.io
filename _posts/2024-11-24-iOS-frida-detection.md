---
layout: post
title: "Frida detection (iOS)"
excerpt_separator: <!--more-->
date: 2024-11-24 00:00:00 +0800
author: xiangxiang
categories: security
tags: [iOS frida security]
---

<!--more-->
* auto-gen TOC:
{:toc}

## 0x01 frida-server
- Use `SVC 0x08` to detect `frida-server`
- PoC trace with [https://github.com/juliangrtz/frida-iOS-syscall-tracer](https://github.com/juliangrtz/frida-iOS-syscall-tracer)

## 0x02 sysctl/ptrace 
- [https://github.com/rustymagnet3000/ios_debugger_challenge?tab=readme-ov-file#challenge-bypass-ptrace-asm-syscall](https://github.com/rustymagnet3000/ios_debugger_challenge?tab=readme-ov-file#challenge-bypass-ptrace-asm-syscall)

 ![](/img/iOS-frida-svc-tracing.jpg)

## 0x03 trampoline
- trampoline detection

```javascript
function Ins(nativePtr, instructionCount) {
    console.log(hexdump(nativePtr, {
        offset: 0,
        length: 32,
        header: true,
        ansi: false
    }));
    for (var i = 0; i < instructionCount; i++) {
        var dissAsm = Instruction.parse(nativePtr);
        console.log(`ins ${i}: ${dissAsm}`);
        nativePtr = dissAsm.next;
    }        
}

Ins(Module.findExportByName(null, "strstr"), 2);

```

```text
## BEFORE HOOK ##
            0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
1f2855920  7f 23 03 d5 f8 5f bc a9 f6 57 01 a9 f4 4f 02 a9  .#..._...W...O..
1f2855930  fd 7b 03 a9 fd c3 00 91 f4 03 01 aa f3 03 00 aa  .{..............
ins 0: pacibsp
ins 1: stp x24, x23, [sp, #-0x40]!


## AFTER HOOK ##
            0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
1f2855920  50 00 00 58 00 02 1f d6 00 40 65 0c 01 00 00 00  P..X.....@e.....
1f2855930  fd 7b 03 a9 fd c3 00 91 f4 03 01 aa f3 03 00 aa  .{..............
ins 0: ldr x16, #0x1f2855928
ins 1: br x16
```

 ![](/img/iOS-trampoline-detection.jpg)


## 0x04 port
- 27042

## 0x05 dylib
- One way is using `dyld_get_image_header` and `dladdr`. They iterate over `dyld_get_image_header` until it returns null and checks if it has seen everything with `dyld_image_count`. The `dladdr` method provides them with the path of the dylib. These are checked against a few unknown strings but removing everything substrate related from the dyld list fixes this check.

- The other method they use is calling `task_info` with flavor `TASK_DYLD_INFO`. This also gives them a list of all loaded dylibs and must be fixed separately from the first dyld check.


## 0x06 memory scan
- `memmem`

## refs
- [https://github.com/apkunpacker/Anti-Frida](https://github.com/apkunpacker/Anti-Frida)
- [https://frida.re/slides/osdc-2015-the-engineering-behind-the-reverse-engineering.pdf](https://frida.re/slides/osdc-2015-the-engineering-behind-the-reverse-engineering.pdf)
- [https://aeonlucid.com/Snapchat-detection-on-iOS/](https://aeonlucid.com/Snapchat-detection-on-iOS/)