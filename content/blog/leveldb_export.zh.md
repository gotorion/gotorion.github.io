+++
title = "LevelDB 源码剖析之符号导出"
date = "2025-02-15"
description = "Leveldb"
taxonomies.tags = [
    "leveldb",
]
+++

# Background
在开发公司项目使用FastDDS的时候看到如下代码：

```cpp
#if defined(_WIN32)
#if defined(EPROSIMA_ALL_DYN_LINK) || defined(FASTCDR_DYN_LINK)
#if defined(fastcdr_EXPORTS)
#define Cdr_DllAPI __declspec( dllexport )
#else
#define Cdr_DllAPI __declspec( dllimport )
#endif // FASTCDR_SOURCE
#else
#define Cdr_DllAPI
#endif // if defined(EPROSIMA_ALL_DYN_LINK) || defined(FASTCDR_DYN_LINK)
#else
#define Cdr_DllAPI
#endif // _WIN32
```

不过我入职以来学习的开发方式一直都是提供接口头文件和库文件给使用者，不是很清楚为什么需要单独导出符号。今天阅读leveldb源码的时候看到[export.h](https://github.com/google/leveldb/blob/main/include/leveldb/export.h)中类似snippets如下，特此研究。

# 区别声明和符号
首先需要了解程序编译的过程：
- 编译阶段：头文件提供了函数和类的声明，让编译器知道这些符号存在；
- 链接阶段：链接器需要找到符号的实现；

对于动态库，只有被显式导出的符号才会被放入符号表，否则链接阶段会出现未定义错误。基于以上可知，声明的符号是在两个阶段作用的，虽然头文件中包含了声明，
依然可以通过设置符号可见性控制调用者的使用。


# 控制符号可见性的优点

- 在程序链接的库较多时可以减少冲突并隐藏部分内部方法，类型于开发库的时候加一个`details`的namespace防止污染。
- 导出符号过多会导致动态库的导出表增大，从而影响库文件体积和加载速度。

# Linux/Win32控制符号可见性
- Linux平台通过attribute的`visibility`控制，可选为`hidden`和`default`；
- Win32平台使用`dllexport`和`dllimport`共同控制符号导出，`__declspec(dllexport)`标记需要导出的符号，`__declspec(dllimport)`标记该符号需要从其他dll或exe中导入；

PS：在Windows平台上默认是不导出符号的，需要通过`dllexport`显示导出；Linux上默认导出所有非静态符号，需要通过`__attribute__((visibility("hidden")))`显示隐藏。

# 查看符号表
平时开发我一般会使用`readelf`和`nm`命令配合`grep`等命令查找，查找资料才了解到有`dlopen`等系列系统调用可以解析动态库，不过目前工作中没有使用场景，暂时留个坑。