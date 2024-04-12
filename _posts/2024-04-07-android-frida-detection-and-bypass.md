---
layout: post
title: "Frida detection and bypass (Android)"
excerpt_separator: <!--more-->
date: 2024-04-07 00:00:00 +0800
author: xiangxiang
categories: security
tags: [android frida security]
---

<!--more-->
* auto-gen TOC:
{:toc}

## 0x01 Detection
### 1.1 Port
- Frida RPC
- Frida default port: 27049
- Send "D-BUS" and check response
- Send "D-BUS" to 1-65535 ports OR only send to only default port

### 1.2 Dual Process (ptrace)
- `fork` -> child process
- child process `ptrace` parent process
- child prcocess's `PPid` should be parent process
- parent process's `TracerPid` should be child process
- other processes cannot `prace` parent process

### 1.3 /proc/<pid>/maps
- Scanning a process memory for artifacts
- Full memory scan, such as strings (e.g. "LIBFRIDA")
- Scan so file name or file path in maps (e.g. "/data/local/tmp", "frida-server")

### 1.4 /proc/<pid>/status
- `TracerPid`

### 1.5 /proc/<pid>/task/<tid>/stat
- `Tracing` status

### 1.6 /proc/<pid>/task/<tid>/status
- task name  (e.g. "gum-js-loop", "gmain", "gdbus", "pool-frida")

### 1.7 /proc/<pid>/fd
- `readlink` (e.g. "/data/local/tmp", "linjector")

### 1.8 injection (memory modification)
- system libraries memory-disk-integrity (e.g. "libc.so", "libart.so")
- integrity of your own code (e.g. "int f(int x)") code from [Detecting and Bypassing Frida Dynamic Function Call Tracing: Exploitation and Mitigation](https://burjcdigital.urjc.es/bitstream/handle/10115/25911/2023-frida-bypass-repositorio.pdf) 

```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
void dumpprelude(unsigned char *f)
{
    int i;
    int c;
    for(i=0; i<32; i++){
        fprintf(stderr, "%02x", *(f+i));
    }
    fprintf(stderr, "\npress enter...\n");
    read(0, &c, 1);
}

int f(int x)
{
    fprintf(stderr, "f says hi!\n");
    return ++x;
}

int main(int argc, char *argv[])
{
    int x;
    dumpprelude((unsigned char*)f);
    dumpprelude((unsigned char*)f);
    memcpy((void*)f, "\xf3\x0f\x1e\xfa\x55", 5);     // Mock injection
    dumpprelude((unsigned char*)f);
    x = f(13);
    fprintf(stderr, "x is %d\n", x);
    exit(EXIT_SUCCESS);
}
```
- some simple detections, see [Android 用户态注入隐藏已死](https://nullptr.icu/index.php/archives/182/)


## 0x02 Bypass
Two main ideas
- Special frida (can pass any detection)
- patch detection code (e.g. `nop` the detection logic)

### 2.1 Special frida
- port: `./frida-server -l 127.0.0.1:8088` change default listening port
- Dual Process (ptrace): spawn mode
- /proc/<pid>/status: spawn mode
- /proc/<pid>/task/<tid>/stat: spawn mode
- /proc/<pid>/task/<tid>/status: compile frida to remove special strings
- /proc/<pid>/fd: compile frida to remove special strings
- injection (memory modification): cannot bypass all
- /proc/<pid>/maps
  + change frida-serve filename
  + compile frida to remove special strings
  + shamiko

```c
void * m = mmap(nullptr, len, map.perms | PROT_WRITE, MAP_ANONYMOUS | MAP_SHARED, -1, 0);
memcpy(m, reinterpret_cast<void *>(map.start), len);
mprotect(m, len, map.perms);
mremap(m, len, len, MREMAP_FIXED | MREMAP_MAYMOVE, reinterpret_cast<void *>(map.start));
```

### 2.2 Patch detection code
- most dection thread/process is created by `pthread_create`
- just trace `pthread_create` and `nop` detection call
  

### 2.3 Custom kernel
My custom kernel hack for kernel 4.19

- `fs/proc/internal.h`

```diff
 static inline struct task_struct *get_proc_task(const struct inode *inode)
 {
-       return get_pid_task(proc_pid(inode), PIDTYPE_PID);
+       struct task_struct * p = get_pid_task(proc_pid(inode), PIDTYPE_PID);
+       char tcomm[64];
+       if (p->flags & PF_WQ_WORKER)
+               wq_worker_comm(tcomm, sizeof(tcomm), p);
+       else
+               __get_task_comm(tcomm, sizeof(tcomm), p);
+       if (strstr(tcomm, "frida") || strstr(tcomm, "gmain") || strstr(tcomm, "gum-js") || strstr(tcomm, "linjector") ||  strstr(tcomm, "gdbus"))
+               return NULL;
+       return p;
 }
```

- `fs/proc/task_mmu.c`

```diff
+static int bypass_show_map_vma(struct vm_area_struct *vma) {
+        struct file *file = vma->vm_file;
+        vm_flags_t flags = vma->vm_flags;
+        if (file && file->f_path.dentry && (strstr(file->f_path.dentry->d_iname, "frida-") || strstr(file->f_path.dentry->d_iname, "/data/local/tmp/")))
+                return 1;
+        if (file && file->f_path.dentry && strstr(file->f_path.dentry->d_iname, "libart.so") && (flags & VM_EXEC))
+                return 1;
+        if (file && file->f_path.dentry && (strstr(file->f_path.dentry->d_iname, "memfd:jit-cache") || strstr(file->f_path.dentry->d_iname, "memfd:jit-zygote-cache")))
+                return 1;
+        return 0;
+}
+
 static void
 show_map_vma(struct seq_file *m, struct vm_area_struct *vma)
 {
@@ -360,6 +372,9 @@ show_map_vma(struct seq_file *m, struct vm_area_struct *vma)
        dev_t dev = 0;
        const char *name = NULL;

+        if (bypass_show_map_vma(vma) == 1)
+                return;
+
        if (file) {
                struct inode *inode = file_inode(vma->vm_file);
                dev = inode->i_sb->s_dev;
@@ -369,6 +384,7 @@ show_map_vma(struct seq_file *m, struct vm_area_struct *vma)

        start = vma->vm_start;
        end = vma->vm_end;
+
        show_vma_header_prefix(m, start, end, flags, pgoff, dev, ino);

        /*
@@ -393,6 +409,11 @@ show_map_vma(struct seq_file *m, struct vm_area_struct *vma)
                        name = "[vdso]";
                        goto done;
                }
+
+               if ((flags & VM_EXEC)) {
+                        name = "[vdso]";
+                       goto done;
+               }

                if (vma->vm_start <= mm->brk &&
                    vma->vm_end >= mm->start_brk) {
@@ -846,6 +867,9 @@ static int show_smap(struct seq_file *m, void *v)
        struct vm_area_struct *vma = v;
        struct mem_size_stats mss;

+        if (bypass_show_map_vma(vma) == 1)
+                return 0;
+
        memset(&mss, 0, sizeof(mss));

        smap_gather_stats(vma, &mss);
```


## refs
- [Android 用户态注入隐藏已死](https://nullptr.icu/index.php/archives/182/)
- [Detecting and Bypassing Frida Dynamic Function Call Tracing: Exploitation and Mitigation](https://burjcdigital.urjc.es/bitstream/handle/10115/25911/2023-frida-bypass-repositorio.pdf)