---
title: Android.mk编写建议
date: 2017-06-26 08:11:58
tags:
---

最近要将一个 C/c++ 程序移植到 Android 平台，所以学习了 [NDK](https://developer.android.com/ndk/guides/index.html) 的基本使用。其中就有 [Android.mk](https://developer.android.com/ndk/guides/android_mk.html) 的编写。这里记录下我个人推荐的一个编写形式，这种形式借助了 Android.mk 的导出功能，使得模块依赖的处理非常简洁，我非常喜欢。


## 1. 通用基本格式

这里是一个通用格式，大部分的 Android.mk 都可以按照这个格式来编写的。我所说的推荐格式当然也是符合这个格式的（当然要遵守人家的标准嘛 O(∩_∩)O~~）。

```makefile
# 必选项，都需要填写。此部分定义了模块必备信息
LOCAL_PATH                  := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE                := (name)
LOCAL_SRC_FILES             := (source file list)

# 可选项，依据需要填写。此部分是模块的依赖项
LOCAL_C_INCLUDES            := [header file path]
LOCAL_CFLAGS                := [c flags]
LOCAL_STATIC_LIBRARIES      := [depended module name] 
LOCAL_SHARED_LIBRARIES      := [depended module name]
LOCAL_LDLIBS                := [ld libs]
LOCAL_LDFLAGS               := [ld flags]

# 可选填，依据需要填写。此部分是导出项，我们要利用的就是这( ^_^ )
LOCAL_EXPORT_C_INCLUDES     := [header file path] 
LOCAL_EXPORT_CFLAGS         := [c flags]
LOCAL_EXPORT_LDLIBS         := [ld libs] 
LOCAL_EXPORT_LDFLAGS        := [ld flags]

# 必选项，但只能选下列之一。此部分定义了模块的编译输出
include $(BUILD_EXECUTABLE) 
include $(BUILD_STATIC_LIBRARY) 
include $(BUILD_SHARED_LIBRARY) 
include $(PREBUILT_SHARED_LIBRARY)
include $(PREBUILT_STATIC_LIBRARY)
```
说明：  
a. `ndk-build` 会自动推导头文件，默认源文件所在的路径都将作为头文件搜索路径，所以不存在其他路径的头文件可以不使用 `LOCAL_C_INCLUDES`；  
b. `*_LDLIBS` 填写示例：`-lz –ldl`。**注意**：`-lpthread`, `–lm`, `–lrt`都是不必要，Android 下会自动链接；  
c. `LOCAL_SRC_FILES` 从当前相对路径开始，通常不需要使用 `$(LOCAL_PATH)`; 在使用预编译时要写对应的库名，如 `libfoo.a` 或者 `libfoo.so`；
d. 一个 Android.mk 可以含多个模块。一个模块自身的内容从
`include $(CLEAR_VARS)` 开始，到定义编译输出结束。所以一个模块的内容务必夹在二者之间。此时，以上通用格式的第一行内容务必作为整个文件的开始；  
f. Android.mk 是一种形式化的 makefile。因而也可以自定义变量，使用 makefile 的内置函数等，这些不受以上说明约束，不过最好参考 [NDK 的建议](https://developer.android.com/ndk/guides/android_mk.html#var)。


## 2. 推荐写法

举例说明：foo 模块依赖 bar 模块,那么至少 foo 要引用 bar 的头文件, 现在假设两个模块目录结构如下：   

```
-src
    -lib
        -foo
            foo.c
            foo.h
            Android.mk
        -bar
            bar.c
            bar.h
            Android.mk
```

### 1) 普通写法

```makefile
# foo 的 Android.mk
LOCAL_PATH              := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE            := foo
LOCAL_SRC_FILES         := foo.c
# ** 差异 ***
LOCAL_C_INCLUDES        := $(LOCAL_PATH)/../bar
LOCAL_STATIC_LIBRARIES  := bar 
include $(BUILD_SHARED_LIBRARY) 

# bar 的 Android.mk
LOCAL_PATH              := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE            := bar
LOCAL_SRC_FILES         := bar.c
include $(BUILD_STATIC_LIBRARY) 
```

### 2) 推荐写法

```makefile
# foo 的 Android.mk
LOCAL_PATH              := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE            := foo
LOCAL_SRC_FILES         := foo.c
LOCAL_STATIC_LIBRARIES  := bar 
include $(BUILD_SHARED_LIBRARY) 

# bar 的 Android.mk
LOCAL_PATH              := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE            := bar
LOCAL_SRC_FILES         := bar.c
# ** 差异 ***
LOCAL_EXPORT_C_INCLUDES	:= $(LOCAL_PATH)  
include $(BUILD_STATIC_LIBRARY) 
```

大家应该看到了，差异是把 foo 依赖于 bar 的头文件从 foo 自身导入变成了 bar 自行导出。貌似这点改进也没有带来什么便利~~。 从这个简单的例子看确实如此。但是想想，引用 bar 的模块比较多的时候，比如说 5 个时，按照普通写法，每个里面都要写引用 bar 的头文件，而用推荐的写法，只需要在 bar 中写一次就好了。

我称这种写法为 “内聚的” 写法。因为被依赖项把依赖于它的模块需要的东西都在自己内部处理掉，即把复杂繁琐的东西内聚了。那么引用的一方只需要写 “我” 依赖于 “你” 就好了。而不需要再考虑 “你” 的头文件，“你” 可能需要链接的系统库等等。   

另外说一点，写依赖项时，务必保持 “干净”。 即不需要，不必要的项都不要添加，保持干净整洁。   


## 总结  

编写 Android.mk 的原则：   
(1)	保持内聚，简单引用。尽量使用被依赖的导出。
(2)	不添加不必要的依赖，保持干净！
