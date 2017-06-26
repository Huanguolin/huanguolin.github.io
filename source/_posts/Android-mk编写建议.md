---
title: Android.mk编写建议
categories: Android
tags: Android makefile Android.mk
date: 2017-06-26 08:11:58
updated: 
---

最近要将一个 C/C++ 程序移植到 Android 平台，所以学习了 [NDK](https://developer.android.com/ndk/guides/index.html) 的基本使用。其中就有 [Android.mk](https://developer.android.com/ndk/guides/android_mk.html) 的编写。这里记录下我个人推荐的一个编写形式，这种形式推崇使用 Android.mk 的导出功能，使得模块依赖的处理更加简洁便利。本文主要分三个部分，第一部分描述 Android.mk 的基本结构，通用格式；第二部分通过举例描述推荐写法，并与普通写法作对比；最后一部分做简单的总结。


## 1. 通用基本格式

这里是一个通用格式，大部分的 Android.mk 都可以按照这个格式来编写的。我所说的推荐格式当然也是符合这个格式的，当然要遵守人家的标准嘛 O(∩_∩)O~~

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

# 可选项，依据需要填写。此部分是导出项，我们要利用的就是这( ^_^ )
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
(1) `ndk-build` 会自动推导头文件，源文件所在的路径都将作为头文件搜索路径，可以使用 `LOCAL_C_INCLUDES` 添加其他路径；  
(2) `*_LDLIBS` 填写示例：`-lz –ldl`。**注意**：`-lpthread`,`–lm`,`–lrt`不需要填写，Android 下会自动链接；     
(3) `LOCAL_SRC_FILES` 默认从`$(LOCAL_PATH)`开始，通常只需列出源文件，如 `foo.c main.c`; 在使用预编译时要写对应的库名，如 `libfoo.a` 或者 `libfoo.so`；  
(4) 一个 Android.mk 可以含多个模块。一个模块的内容从 `include $(CLEAR_VARS)` 开始，到定义编译输出结束。所以一个模块的内容务必夹在二者之间。此时，以上通用格式的第一行内容务必作为整个文件的开始,且后续不需要再写它；    
(5) Android.mk 是一种形式化的 makefile。因而也可以自定义变量，使用 makefile 的内置函数等，这些不受以上说明约束，不过最好参考 [NDK 的建议](https://developer.android.com/ndk/guides/android_mk.html#var)。


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
LOCAL_EXPORT_C_INCLUDES	:= $(LOCAL_PATH)   
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

两种写法的差异是：把 foo 依赖于 bar 的头文件从 foo 自身导入变成了 bar 自行导出。貌似这点改进也没带来什么便利~~。 从这个简单的例子看确实如此。但是想想，引用 bar 的模块比较多的时候，按照普通写法，每个里面都要写引用 bar 的头文件，而用推荐的写法中，只需要在 bar 中写一次，以后不管有多少模块引用它，都不用再多写了。在模块多，引用复杂的项目中，这一点能带来很大便捷！

推荐写法是 “高内聚，低耦合” 的。因为被依赖项把依赖于它的模块需要的东西都在 “隐藏” 在自己内部，把耦合仅仅体现于依赖引用上。如此引用的一方只需要写 “我” 需要 “你” 就好了，而不必再显示的写需要 “你” 的头文件，需要 “你” 要链接的系统库等等。   

另外说一点，写依赖项时，务必保持 “干净”。 即不要添加不需要的项。   


## 3. 总结  

编写 Android.mk 的原则：   
(1)	保持内聚，简单引用。尽量使用导出功能。  
(2)	不添加不必要的依赖，保持干净！
