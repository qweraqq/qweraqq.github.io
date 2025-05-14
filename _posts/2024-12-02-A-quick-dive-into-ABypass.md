---
layout: post
title: "A quick dive into A-Bypass"
excerpt_separator: <!--more-->
date: 2024-12-02 00:00:00 +0800
author: xiangxiang
categories: security
tags: [iOS security]
---

<!--more-->
* auto-gen TOC:
{:toc}

## A-Bypass
- Disable ABASM (patch SVC code)
- Enable hookSVC80 (hook SVC code)
- Enable noHookingPlz (no MSHookFunction)
- MSHookFunction (open/fcntl/opendir/strstr/ptrace/syscall)
- Enforce DYLD Hook (MSHookFunction dyld functions)

```c
// Enable hookSVC80 (hook SVC code)
if([ABSI.hookSVC80 containsObject:identifier]) hookingSVC80();
// ...

// Enable noHookingPlz (no MSHookFunction)
isSubstitute = ([manager fileExistsAtPath:@"/usr/lib/libsubstitute.dylib"] && ![manager fileExistsAtPath:@"/usr/lib/substrate"] && ![manager fileExistsAtPath:@"/usr/lib/libhooker.dylib"]);
BOOL isLibHooker = [[NSFileManager defaultManager] fileExistsAtPath:@"/usr/lib/libhooker.dylib"];
HBLogError(@"[ABPattern] It is libhooker? %d", isLibHooker);

[[ABPattern sharedInstance] setup:dic[@"data"]];
if([ABSI.noHookingPlz containsObject:identifier]) return hideProgress(); // toast and return
// ...

// Disable ABASM (patch SVC code)
if(![ABSI.ABASMBlackList containsObject:identifier] && ![ABSI.hookSVC80 containsObject:identifier]) {
    removeSYS_open();
    removeSYS_access();
    removeSYS_access2();
    removeSYS_symlink();
}
// ...

// MSHookFunction hook
if(!(iosVersion > 14 && !isLibHooker && !isSubstitute)) {
    MSHookFunction((void *)open, (void *)hook_open, (void **)&orig_open);
    MSHookFunction((void *)fcntl, (void *)hook_fcntl, (void **)&orig_fcntl);
}
else abreset((struct rebinding[1]){{"open", (void *)hook_open, (void **)&orig_open}}, 1);

abreset((struct rebinding[1]){{"opendir", (void *)hook_opendir, (void **)&orig_opendir}}, 1);
MSHookFunction((void *)dlsym(RTLD_DEFAULT, "strstr"), (void *)my_strstr, (void **)&orig_strstr);
MSHookFunction((void *)dlsym(RTLD_DEFAULT, "ptrace"),(void*)my_ptrace,(void**)&orig_ptrace);
if(0)MSHookFunction((void *)dlsym(RTLD_DEFAULT, "syscall"), (void *)hooked_syscall, (void **)&orig_syscall);
// ...

// Enforce DYLD Hook (hook dyld functions)
if(objc_getClass("Eversafe") || (isLibHooker && iosVersion < 14) || [ABSI.enforceDYLD containsObject:identifier]) {
    void DYLDSaver();
    DYLDSaver();
} else {
    %init(dyldUnc0ver);
}
```
### Disable ABASM
- [Entry point: CAN NOT work with "Enable hookSVC80"](https://github.com/Baw-Appie/A-Bypass/blob/97831a597c09083a43f46aeacb134837ca4fce54/abypassloader/Tweak.xm#L2026-L2031)
- [Iterate Dyld Images](https://github.com/Baw-Appie/A-Bypass/blob/97831a597c09083a43f46aeacb134837ca4fce54/abypassloader/ImagePatcher.xm#L82-L93)
- [Search Memory for SVC calls](https://github.com/Baw-Appie/A-Bypass/blob/97831a597c09083a43f46aeacb134837ca4fce54/abypassloader/ImagePatcher.xm#L53-L76)
- [Patch code](https://github.com/Baw-Appie/A-Bypass/blob/97831a597c09083a43f46aeacb134837ca4fce54/abypassloader/ImagePatcher.xm#L186-L206)

```c
// ...
    if(![ABSI.ABASMBlackList containsObject:identifier] && ![ABSI.hookSVC80 containsObject:identifier]) {
      removeSYS_open();
      removeSYS_access();
      removeSYS_access2();
      removeSYS_symlink();
    }
// ...

bool patchCode(void *target, const void *data, size_t size) {
  @try {
    kern_return_t err;
    mach_port_t port = mach_task_self();
    vm_address_t address = (vm_address_t) target;

    err = vm_protect(port, address, size, false, VM_PROT_READ | VM_PROT_WRITE | VM_PROT_COPY);
    if (err != KERN_SUCCESS) return false;

    err = vm_write(port, address, (vm_address_t) data, size);
    if (err != KERN_SUCCESS) return false;

    err = vm_protect(port, address, size, false, VM_PROT_READ | VM_PROT_EXECUTE);
    if (err != KERN_SUCCESS) return false;
  } @catch(NSException *e) {
    debugMsg(@"[ABASM] ABASM Patcher has crashed. Aborting patch.. (%p)", target);
    return false;
  }

  return true;
}
```

### Enable hookSVC80
- [Entry point](https://github.com/Baw-Appie/A-Bypass/blob/97831a597c09083a43f46aeacb134837ca4fce54/abypassloader/Tweak.xm#L1943)
- [Iterate Dyld Images](https://github.com/Baw-Appie/A-Bypass/blob/97831a597c09083a43f46aeacb134837ca4fce54/abypassloader/ImagePatcher.xm#L82-L93)
- [Search Memory for SVC calls](https://github.com/Baw-Appie/A-Bypass/blob/97831a597c09083a43f46aeacb134837ca4fce54/abypassloader/ImagePatcher.xm#L53-L76)
- [Dobby Hook](https://github.com/Baw-Appie/A-Bypass/blob/97831a597c09083a43f46aeacb134837ca4fce54/abypassloader/ImagePatcher.xm#L704-L725)

```c
/* 
 * 0x0000000000000000:  30 04 80 D2    movz x16, #0x21
 * 0x0000000000000004:  01 10 00 D4    svc  #0x80
 */

void hookingSVC80Handler(RegisterContext *reg_ctx, const HookEntryInfo *info) {
    int num_syscall = (int)(uint64_t)(reg_ctx->general.regs.x16);
    char *arg1 = (char *)reg_ctx->general.regs.x0;
    debugMsg(@"[ABZZ] System PRECALL %d %p %p", num_syscall, info->target_address, (uint8_t *)_dyld_get_image_vmaddr_slide(0));
    
    if(num_syscall == SYS_symlink) {
      char *arg2 = (char *)reg_ctx->general.regs.x1;
      [[ABPattern sharedInstance] usk:@(arg1) n:@(arg2)];
    }
    if(num_syscall == SYS_open || num_syscall == SYS_access || num_syscall == SYS_statfs64 || num_syscall == SYS_statfs || num_syscall == SYS_lstat64 || num_syscall == SYS_stat64 || num_syscall == SYS_rename || num_syscall == SYS_setxattr || num_syscall == SYS_pathconf) {
        debugMsg(@"[ABZZ] SYS_open with SVC 80, %s", arg1);
        if([@(arg1) isEqualToString:@"/dev/urandom"]) {
          *(unsigned long *)(&reg_ctx->general.regs.x0) = (unsigned long long)"/Protected.By.ABypass";
          return;
        }
        if(![[ABPattern sharedInstance] u:@(arg1) i:30001] && ![@(arg1) isEqualToString:@"/sbin/mount"]) {
          *(unsigned long *)(&reg_ctx->general.regs.x0) = (unsigned long long)"/Protected.By.ABypass";
        } else {
          debugMsg(@"[ABZZ] not blocked!");
        }
    }
}
```

### Enable noHookingPlz
- [Entry point](https://github.com/Baw-Appie/A-Bypass/blob/97831a597c09083a43f46aeacb134837ca4fce54/abypassloader/Tweak.xm#L1982)
- [Toast](https://github.com/Baw-Appie/A-Bypass/blob/97831a597c09083a43f46aeacb134837ca4fce54/Tweak.xm#L126-L154)
- Return ( Stop `MSHookFunction` etc )

```c
%new
-(NSDictionary *)handleUpdateLicense:(NSDictionary *)userInfo {
  NSError *error = nil;
  if([userInfo[@"type"] isEqual:@3]) {
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^() {
      dispatch_async(dispatch_get_main_queue(), ^() {
        [self.loadingToastForABypass showToast:@"Loading A-Bypass" withMessage:[NSString stringWithFormat:@"0/%@ Completed", userInfo[@"max"]]];
      });
    });
  } else if([userInfo[@"type"] isEqual:@4]) {
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^() {
      dispatch_async(dispatch_get_main_queue(), ^() {
        [self.loadingToastForABypass setPercent:[userInfo[@"per"] doubleValue]/[userInfo[@"max"] doubleValue]];
        self.loadingToastForABypass.subtitleLabel.text = [NSString stringWithFormat:@"%@/%@ Completed", userInfo[@"per"], userInfo[@"max"]];
      });
    });
  } else if([userInfo[@"type"] isEqual:@5]) {
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^() {
      dispatch_async(dispatch_get_main_queue(), ^() {
        [self.loadingToastForABypass hideToast];
      });
    });
  } else if([userInfo[@"type"] isEqual:@6]) {
    [fileManager removeItemAtPath:@"/var/mobile/Library/Preferences/ABLivePatch" error:&error];
  }
  if(error) return @{@"success": @0, @"message": [error localizedDescription]};
  return @{@"success": @1};
}
%end
```


### Enforce DYLD Hook
- [Entry point](https://github.com/Baw-Appie/A-Bypass/blob/97831a597c09083a43f46aeacb134837ca4fce54/abypassloader/Tweak.xm#L2058-L2063)
- [Hook dyld functions](https://github.com/Baw-Appie/A-Bypass/blob/97831a597c09083a43f46aeacb134837ca4fce54/abypassloader/DYLDSaver.xm#L96C1-L104C2)

```c
void DYLDSaver() {
	MSHookFunction((void *)_dyld_image_count, (void *)hook_dyld_image_count, (void **)&orig_dyld_image_count);
	MSHookFunction((void *)_dyld_get_image_name, (void *)hook_dyld_get_image_name, (void **)&orig_dyld_get_image_name);
	MSHookFunction((void *)_dyld_get_image_header, (void *)hook_dyld_get_image_header, (void **)&orig_dyld_get_image_header);
	MSHookFunction((void *)_dyld_get_image_vmaddr_slide, (void *)hook_dyld_get_image_vmaddr_slide, (void **)&orig_dyld_get_image_vmaddr_slide);
	// %init(DYLDSaver);
	syncDyldArray();
	_dyld_register_func_for_add_image(&add_binary_image);
}
```
