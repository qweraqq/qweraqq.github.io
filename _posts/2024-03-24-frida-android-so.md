---
layout: post
title: "Frida动态分析Android so"
excerpt_separator: <!--more-->
date: 2024-03-24 00:00:00 +0800
author: xiangxiang
categories: security
tags: [android frida security]
---
也许Frida比断点调试更友好

<!--more-->
* auto-gen TOC:
{:toc}



## 0x01 Android Linker源码分析
- 通过分析源码分析，我们可以利用执行so文件`.init`、`.init_array`前后系统调用打印日志的时机，进行hook

{% highlight text %}
// 特征
init
"[ Calling c-tor %s @ %p for '%s' ]"
"[ Done calling c-tor %s @ %p for '%s' ]"

"[ Calling d-tor %s @ %p for '%s' ]"
"[ Done calling d-tor %s @ %p for '%s' ]"


init_array
"[ Calling %s (size %zd) @ %p for '%s' ]"
"[ Done calling %s for '%s' ]"
{% endhighlight %}

### 1.1 dlopen_ext
- [linker/dlfcn.cpp](https://android.googlesource.com/platform/bionic/+/refs/heads/android13-gsi/linker/dlfcn.cpp)

```cpp
static void* dlopen_ext(const char* filename,
                        int flags,
                        const android_dlextinfo* extinfo,
                        const void* caller_addr) {

  // 信号互斥量（锁）
  ScopedPthreadMutexLocker locker(&g_dl_mutex);
  g_linker_logger.ResetState();

  // 调用do_dlopen()函数实现so库文件的加载
  void* result = do_dlopen(filename, flags, extinfo, caller_addr);

  // 判断so库文件是否加载成功
  if (result == nullptr) {
    __bionic_format_dlerror("dlopen failed", linker_get_error_buffer());
    return nullptr;
  }

  // 返回加载后so库文件的文件句柄
  return result;
}
```

### 1.2 do_dlopen
- [linker/linker.cpp](https://android.googlesource.com/platform/bionic/+/refs/heads/android13-gsi/linker/linker.cpp)

```cpp
void* do_dlopen(const char* name, int flags,
                const android_dlextinfo* extinfo,
                const void* caller_addr) {
  std::string trace_prefix = std::string("dlopen: ") + (name == nullptr ? "(nullptr)" : name);
  ScopedTrace trace(trace_prefix.c_str());
  ScopedTrace loading_trace((trace_prefix + " - loading and linking").c_str());
  soinfo* const caller = find_containing_library(caller_addr);
  android_namespace_t* ns = get_caller_namespace(caller);
  LD_LOG(kLogDlopen,
         "dlopen(name=\"%s\", flags=0x%x, extinfo=%s, caller=\"%s\", caller_ns=%s@%p, targetSdkVersion=%i) ...",
         name,
         flags,
         android_dlextinfo_to_string(extinfo).c_str(),
         caller == nullptr ? "(null)" : caller->get_realpath(),
         ns == nullptr ? "(null)" : ns->get_name(),
         ns,
         get_application_target_sdk_version());
  auto purge_guard = android::base::make_scope_guard([&]() { purge_unused_memory(); });
  auto failure_guard = android::base::make_scope_guard(
      [&]() { LD_LOG(kLogDlopen, "... dlopen failed: %s", linker_get_error_buffer()); });

  // 判断加载so文件的flags是否符合要求
  if ((flags & ~(RTLD_NOW|RTLD_LAZY|RTLD_LOCAL|RTLD_GLOBAL|RTLD_NODELETE|RTLD_NOLOAD)) != 0) {
    DL_OPEN_ERR("invalid flags to dlopen: %x", flags);
    return nullptr;
  }

  if (extinfo != nullptr) {
    if ((extinfo->flags & ~(ANDROID_DLEXT_VALID_FLAG_BITS)) != 0) {
      DL_OPEN_ERR("invalid extended flags to android_dlopen_ext: 0x%" PRIx64, extinfo->flags);
      return nullptr;
    }
    if ((extinfo->flags & ANDROID_DLEXT_USE_LIBRARY_FD) == 0 &&
        (extinfo->flags & ANDROID_DLEXT_USE_LIBRARY_FD_OFFSET) != 0) {
      DL_OPEN_ERR("invalid extended flag combination (ANDROID_DLEXT_USE_LIBRARY_FD_OFFSET without "
          "ANDROID_DLEXT_USE_LIBRARY_FD): 0x%" PRIx64, extinfo->flags);
      return nullptr;
    }
    if ((extinfo->flags & ANDROID_DLEXT_USE_NAMESPACE) != 0) {
      if (extinfo->library_namespace == nullptr) {
        DL_OPEN_ERR("ANDROID_DLEXT_USE_NAMESPACE is set but extinfo->library_namespace is null");
        return nullptr;
      }
      ns = extinfo->library_namespace;
    }
  }
  // Workaround for dlopen(/system/lib/<soname>) when .so is in /apex. http://b/121248172
  // The workaround works only when targetSdkVersion < Q.
  std::string name_to_apex;
  if (translateSystemPathToApexPath(name, &name_to_apex)) {
    const char* new_name = name_to_apex.c_str();
    LD_LOG(kLogDlopen, "dlopen considering translation from %s to APEX path %s",
           name,
           new_name);
    // Some APEXs could be optionally disabled. Only translate the path
    // when the old file is absent and the new file exists.
    // TODO(b/124218500): Re-enable it once app compat issue is resolved
    /*
    if (file_exists(name)) {
      LD_LOG(kLogDlopen, "dlopen %s exists, not translating", name);
    } else
    */
    if (!file_exists(new_name)) {
      LD_LOG(kLogDlopen, "dlopen %s does not exist, not translating",
             new_name);
    } else {
      LD_LOG(kLogDlopen, "dlopen translation accepted: using %s", new_name);
      name = new_name;
    }
  }
  // End Workaround for dlopen(/system/lib/<soname>) when .so is in /apex.
  std::string asan_name_holder;
  const char* translated_name = name;
  if (g_is_asan && translated_name != nullptr && translated_name[0] == '/') {
    char original_path[PATH_MAX];
    if (realpath(name, original_path) != nullptr) {
      asan_name_holder = std::string(kAsanLibDirPrefix) + original_path;
      if (file_exists(asan_name_holder.c_str())) {
        soinfo* si = nullptr;
        if (find_loaded_library_by_realpath(ns, original_path, true, &si)) {
          PRINT("linker_asan dlopen NOT translating \"%s\" -> \"%s\": library already loaded", name,
                asan_name_holder.c_str());
        } else {
          PRINT("linker_asan dlopen translating \"%s\" -> \"%s\"", name, translated_name);
          translated_name = asan_name_holder.c_str();
        }
      }
    }
  }
  ProtectedDataGuard guard;

  // find_library会判断so是否已经加载，
  // 如果没有加载，对so进行加载，完成一些初始化工作
  soinfo* si = find_library(ns, translated_name, flags, extinfo, caller);

  loading_trace.End();
  if (si != nullptr) {
    void* handle = si->to_handle();
    LD_LOG(kLogDlopen,
           "... dlopen calling constructors: realpath=\"%s\", soname=\"%s\", handle=%p",
           si->get_realpath(), si->get_soname(), handle);

    // ++++++ so加载成功，调用构造函数 ++++++++
    si->call_constructors();
   // ++++++++++++++++++++++++++++++++++++++++

    failure_guard.Disable();
    LD_LOG(kLogDlopen,
           "... dlopen successful: realpath=\"%s\", soname=\"%s\", handle=%p",
           si->get_realpath(), si->get_soname(), handle);

    // 返回so内存模块
    return handle;
  }
  return nullptr;
}
```
### 1.3 soinfo::call_constructors
- [linker/linker_soinfo.cpp](https://android.googlesource.com/platform/bionic/+/refs/heads/android13-gsi/linker/linker_soinfo.cpp)

```cpp
void soinfo::call_constructors() {
  if (constructors_called || g_is_ldd) {
    return;
  }
  // We set constructors_called before actually calling the constructors, otherwise it doesn't
  // protect against recursive constructor calls. One simple example of constructor recursion
  // is the libc debug malloc, which is implemented in libc_malloc_debug_leak.so:
  // 1. The program depends on libc, so libc's constructor is called here.
  // 2. The libc constructor calls dlopen() to load libc_malloc_debug_leak.so.
  // 3. dlopen() calls the constructors on the newly created
  //    soinfo for libc_malloc_debug_leak.so.
  // 4. The debug .so depends on libc, so CallConstructors is
  //    called again with the libc soinfo. If it doesn't trigger the early-
  //    out above, the libc constructor will be called again (recursively!).
  constructors_called = true;
  if (!is_main_executable() && preinit_array_ != nullptr) {
    // The GNU dynamic linker silently ignores these, but we warn the developer.
    PRINT("\"%s\": ignoring DT_PREINIT_ARRAY in shared library!", get_realpath());
  }
  get_children().for_each([] (soinfo* si) {
    si->call_constructors();
  });
  if (!is_linker()) {
    bionic_trace_begin((std::string("calling constructors: ") + get_realpath()).c_str());
  }
  // DT_INIT should be called before DT_INIT_ARRAY if both are present.

  // 先调用.init段的构造函数
  call_function("DT_INIT", init_func_, get_realpath());

  // 再调用.init_array段的构造函数
  call_array("DT_INIT_ARRAY", init_array_, init_array_count_, false, get_realpath());

  if (!is_linker()) {
    bionic_trace_end();
  }
}
```

### 1.4 call_function & call_array
- [linker/linker_soinfo.cpp](https://android.googlesource.com/platform/bionic/+/refs/heads/android13-gsi/linker/linker_soinfo.cpp)

```cpp
template <typename F>
static inline void call_array(const char* array_name __unused, F* functions, size_t count,
                              bool reverse, const char* realpath) {
  if (functions == nullptr) {
    return;
  }
  TRACE("[ Calling %s (size %zd) @ %p for '%s' ]", array_name, count, functions, realpath);
  int begin = reverse ? (count - 1) : 0;
  int end = reverse ? -1 : count;
  int step = reverse ? -1 : 1;
  for (int i = begin; i != end; i += step) {
    TRACE("[ %s[%d] == %p ]", array_name, i, functions[i]);
    call_function("function", functions[i], realpath);
  }
  TRACE("[ Done calling %s for '%s' ]", array_name, realpath);
}
```


```cpp
static void call_function(const char* function_name __unused,
                          linker_ctor_function_t function,
                          const char* realpath __unused) {
  if (function == nullptr || reinterpret_cast<uintptr_t>(function) == static_cast<uintptr_t>(-1)) {
    return;
  }
  TRACE("[ Calling c-tor %s @ %p for '%s' ]", function_name, function, realpath);
  function(g_argc, g_argv, g_envp);
  TRACE("[ Done calling c-tor %s @ %p for '%s' ]", function_name, function, realpath);
}

static void call_function(const char* function_name __unused,
                          linker_dtor_function_t function,
                          const char* realpath __unused) {
  if (function == nullptr || reinterpret_cast<uintptr_t>(function) == static_cast<uintptr_t>(-1)) {
    return;
  }
  TRACE("[ Calling d-tor %s @ %p for '%s' ]", function_name, function, realpath);
  function();
  TRACE("[ Done calling d-tor %s @ %p for '%s' ]", function_name, function, realpath);
}
```


### 1.5 TRACE macro
- [linker/linker_debug.h](https://android.googlesource.com/platform/bionic/+/refs/heads/android13-gsi/linker/linker_debug.h)

```h
#define LINKER_VERBOSITY_TRACE  1

#define TRACE(x...)          _PRINTVF(LINKER_VERBOSITY_TRACE, x)

#define _PRINTVF(v, x...) \
    do { \
      if (g_ld_debug_verbosity > (v)) linker_log((v), x); \
    } while (0)
```

- [linker/linker_debug.cpp](https://android.googlesource.com/platform/bionic/+/refs/heads/android13-gsi/linker/linker_debug.cpp)

```cpp
void linker_log_va_list(int prio __unused, const char* fmt, va_list ap) {
#if LINKER_DEBUG_TO_LOG
  async_safe_format_log_va_list(5 - prio, "linker", fmt, ap);
#else
  async_safe_format_fd_va_list(STDOUT_FILENO, fmt, ap);
  write(STDOUT_FILENO, "\n", 1);
#endif
}
void linker_log(int prio, const char* fmt, ...) {
  va_list ap;
  va_start(ap, fmt);
  linker_log_va_list(prio, fmt, ap);
  va_end(ap);
}
```

## 0x02 dump so
- APK文件中的so可能无法直接解析, 原因是so是在dlopen阶段动态释放的
- 现在的so可能会在`.init`、`.init_array`或者`JNI_OnLoad`中进行环境检测或者闪退, 故我们需要尽可能在各个可能的阶段前后断点, 然后进行dump
- 对于从内存中导出的so文件, 可以借助工具
- 工具代码请见 [https://github.com/qweraqq/frida-dump-android-so](https://github.com/qweraqq/frida-dump-android-so)

 ![](/img/frida-dump-android-so-1.jpg)

- dump得到的so可以正常解析

 ![](/img/frida-dump-android-so-2.jpg)


## 0x03 stalker
- TODO
- 接下来就可以通过frida的stalker进行动态分析
- 动静态可以对应分析

 ![](/img/frida-android-trace-so-1.jpg)

```bash
cat /data/tombstones/...
```

## 0x04 misc
### 4.1 魔改去特征版Frida
- [https://github.com/hzzheyang/strongR-frida-android](https://github.com/hzzheyang/strongR-frida-android)

### 4.2 修改文件名启动Frida
- 将二进制通过adb push至安卓手机
- 将二进制移动到`/data/local/tmp`或`/sbin/`目录下、更名并增加执行权限
- 重命名后只能通过网络方式调用, 原因可能是本地frida-tools与魔改版不匹配? 参考[https://github.com/frida/frida/issues/2326](https://github.com/frida/frida/issues/2326)
- 可能需要shamiko, 猜测是需要隐藏内存, 原理参考 [https://nullptr.icu/index.php/archives/182/](https://nullptr.icu/index.php/archives/182/)
- 网络方式可以通过`adb`进行转发

手机上
```bash
cd /sbin/
cp /sdcard/Download/hluda-server-16.2.1-android-arm64 ./h-s
chmod +x h-s
./h-s -l 0.0.0.0:8081
```

PC端
```bash
adb forward tcp:8081 tcp:8081
frida -H 127.0.0.1:8081 -l xxx.js -f com.android.xxx
```

### 4.3 Frida Scripts Collection
- [https://github.com/apkunpacker/FridaScripts](https://github.com/apkunpacker/FridaScripts)
- [https://github.com/hluwa/frida-dexdump](https://github.com/hluwa/frida-dexdump)
- [https://github.com/sensepost/objection.git](https://github.com/sensepost/objection.git)
### 4.4 Frida Snippets

```js
Java.perform(function() {
    Java.enumerateLoadedClasses({
        onMatch: function(className) {
            if(className.indexOf("something") >=0 ){
              console.log(className);
            }
        },
        onComplete: function() {}
    });
});
```

## refs
- [在Android so文件的.init、.init_array上和JNI_OnLoad处下断点](https://blog.csdn.net/QQ1084283172/article/details/54233552)