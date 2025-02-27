+++
title = "C++ Template之SFINAE"
date = "2025-02-27"
description = "C++ SFINAE"
taxonomies.tags = [
    "C++",
]
+++

# Background
最近面试了几位拥有3年以上经历的工程师，简历上都写了熟悉STL标准库及模板编程，不过问到类型萃取和SFINAE之类的点都不怎么了解。虽说技术服务于业务，应用开发对模板要求并不高，不过多了解些模板能加深对标准库的理解。借此契机写点简单的demo介绍一下SFINAE这个Tricky的机制。

# Intro
SFINAE[Substitution failure is not an error]意思是当编译器遇到模板参数不匹配的情况下，会自动跳过匹配失败的模板，继续查找符合的模板。

```cpp
// foo.hpp
struct Foo {
  typedef int fooType;
};

template <typename T> bool test_foo(typename T::fooType) { // Defination #1
  // only if T has a nested type fooType
  return true;
}
template <typename T> bool test_foo(T) { return false; }  // Defination #2

inline void test1() {
  assert(test_foo<Foo>(1));
  assert(!test_foo<int>(1)); // compile error if #2 was deleted
}
```

上面的代码中共定义了两个版本的`test_foo`函数模板，#1要求模板参数T必须拥有内嵌类型`fooType`，因此`test_foo<Foo>`会优先匹配#1。实际的查找过程如下：
1. 根据调用进行[Name Loopup](https://en.cppreference.com/w/cpp/language/lookup)；
2. 查找到#1，进行参数类型替换，并加入到解析集中；
3. 查找到#2，参数类型不符合，从解析集中删除；
4. 如果解析集为空，编译失败；
5. 如果解析集中有匹配的函数，根据参数类型找到最匹配的函数；

借助这一特性可以在编译器做一些决断，例如判断模板类型T是否有内嵌类型，是否有指定成员函数之类。
```cpp
// run.hpp
struct Human {
  void eat() {}
  void run(double speed) {}
};

struct Robot {
  void charge() {}
  void run(double speed) {} // if delete this method, assert fails
};

template <typename T> struct has_run {
  template <typename U, void (U::*)(double)> struct SFINAE {};

  template <typename U> static char test(SFINAE<U, &U::run> *); // #1

  template <typename U> static int test(...); // #2

  static constexpr bool value = sizeof(test<T>(nullptr)) == sizeof(char);
};

// used a lot in type_traits
template <typename T> constexpr bool has_run_v = has_run<T>::value; // #3

inline void test() {
  assert(has_run<Human>::value);
  assert(has_run<Robot>::value);
  assert(has_run_v<Human>);
  assert(has_run_v<Robot>);
}
```
上例中定义了一个`has_run`模板检测T类型是否有`run`方法，SFINAE模板中模板第二个参数是一个参数类型为double的成员函数指针。如果T类型定义了`run`方法，`test<T>(nullptr)`会匹配返回类型为char的`test`方法，从而使得`value = true`。

> 在C++17前，如果要判断`template<typename T1, typename T2>`中的T1、T2是否为相同类型，通常需要使用`std::is_same<T1, T2>::value`的形式，新版本中则可以使用`std::is_same_v<T1, T2>`，底层实现类似#3处的定义，并没有什么magic。

# Into SFINAE

## 使用std::enable_if
上面的写法过于繁琐，C++11引入了`enable_if`，其定义如下：
```cpp
template< bool B, class T = void > // 第一个参数是bool类型
struct enable_if;
```
这里先给出简单demo，再解释其参数的含义。
```cpp
struct Human {
  static constexpr bool has_run = true;
  void eat() {}
  void run(double speed) {}
};

struct Robot {
  static constexpr bool has_run = false;
  void charge() {}
};

template <typename T> struct has_run {
  static constexpr bool value = T::has_run;
};

template <typename T> constexpr bool has_run_v = has_run<T>::value;

template <typename T, typename = std::enable_if_t<has_run_v<T>>> // #1
void run(T &t, double speed) {
  t.run(speed);
}

template <typename T, typename = std::enable_if_t<!has_run_v<T>>,
          typename = void> // #2
void run(T &t, double speed) {
  // do nothing
}

inline void test() {
  Human h;
  run(h, 1);

  Robot r;
  run(r, 1); // Should compile and do nothing
}
```

这里将SFINAE修改为内置`has_run`tag的方式进行类型萃取，简化部分代码。下面定义了两个`run`函数，两个模板参数个数不同，防止重定义，如果删除#2处的`typename = void`，由于两个模板函数的模版参数列表和函数参数列表完全相同，编译器将无法区分。

为了理解`enable_if`的原理，需要参考源码：
```cpp
  /// Alias template for enable_if
  template<bool _Cond, typename _Tp = void>
    using enable_if_t = typename enable_if<_Cond, _Tp>::type
      /// Define a member typedef `type` only if a boolean constant is true.
  template<bool, typename _Tp = void>
    struct enable_if
    { };
  // Partial specialization for true.
  template<typename _Tp>
    struct enable_if<true, _Tp>
    { typedef _Tp type; };
```
在我们的demo中，`Human`类具有`run`方法，且我们未在`enable_if_t`中指定第二个模板参数的类型，因此`has_run_v<Human>`匹配为`enable_if<true, void>::type`, 实际偏特化后为
```cpp
struct enable_if<true, void>
{
    typedef void type;
}
```
在使用`Robot`类调用`run`函数时，`has_run_v<Robot>`为false，#1, #2处的`enable_if_t`实例化后为
```cpp
struct enable_if { } // #1
struct enable_if<true, Robot> { // #2
    typedef Robot type;
}
template<Robot, Robot, void> // #2
void run(Robot & t, double speed) { }
```
此时#1的第二个模板参数typename = ?，无法找到对应的type，如果把#2删掉，这里就会报错。`enable_if`正是根据这种内嵌类型`type`的存在进行匹配。

## decltype & declval
`decltype`用于编译时类型推导，其参数是一个表达式。`decltype`返回该表达式的类型，但不会对表达式进行求值。
```cpp
class Test {
public:
  Test() { std::cout << "Test constructor\n"; }
  static Test build() { Test(); }
};
void func() {
  using tType = decltype(Test::build()); // no contruct
  tType t;
}
// output: Test constructor
```
假如Test类没有public的默认构造函数`decltype`会推导失败。
```cpp
class Test {
private:
  Test() = delete;
};
void func() { using tType = decltype(Test()); } // error
```
使用`std::declval`可解决这个问题，`std::declval`可以在类型没有默认构造函数时进行推导，它的作用是返回一个类型为`T&&`的Fake对象，并允许推导该类型的成员函数。
```cpp
class Test {
private:
  Test() = delete;
};
void func() { using tType = decltype(std::declval<Test>()); } // success! tType = Test
```
借助以上工具，可以写出以下代码检测类型是否具有某成员函数：
```cpp
template <typename T>
auto test_func() -> decltype(std::declval<T>().charge(), void()) {
  std::cout << "T has charge method\n";
}

test_func<Robot>(); // output: T has charge method
```
注意C++中逗号表达式的值是最后一个逗号后表达式的值，因此`test_func`的返回值类型是`decltype(void())`，由于`decltype`参数要求是表达式，所以需要写`void()`而非`void`。

## void_t
先看定义，简单来说`void_t`的作用就是将任意参数类型转换为`void`类型：
```cpp
#if __cplusplus >= 201703L || !defined(__STRICT_ANSI__) // c++17 or gnu++11
#define __cpp_lib_void_t 201411L
  /// A metafunction that always yields void, used for detecting valid types.
  template<typename...> using void_t = void;
#endif

  /// integral_constant
  template<typename _Tp, _Tp __v>
    struct integral_constant
    {
      static constexpr _Tp                  value = __v;
      typedef _Tp                           value_type;
      typedef integral_constant<_Tp, __v>   type;
      constexpr operator value_type() const noexcept { return value; }
#if __cplusplus > 201103L

#define __cpp_lib_integral_constant_callable 201304L

      constexpr value_type operator()() const noexcept { return value; }
#endif
    };

  /// The type used as a compile-time boolean with true value.
  using true_type =  integral_constant<bool, true>;

  /// The type used as a compile-time boolean with false value.
  using false_type = integral_constant<bool, false>;
```
借助以上工具，模板可以再次更新：
```cpp
// check.hpp
template <typename T, typename = void> struct check_run : std::false_type {}; // #1

template <typename T> // #2
struct check_run<T, std::void_t<decltype(std::declval<T>().run(1))>> // just a random number, does't important
    : std::true_type {};

inline void check() {
  assert(check_run<Human>::value);
  assert(check_run<Robot>::value); // failed
}
```
在使用`Human`类进行匹配时，#1可以成功推导`T = Human`，#2可以推导`T = Human`且使用以上工具推导第二个模版参数为`void`，此时两个模板都能成功推导，编译器会选择一个最合适的偏特化进行匹配。
在使用`Robot`类进行匹配时，只有#1可以成功推导，因此`check_run<Robot>::value`为false。

## constexpr
C++17中`constexpr`不仅可以定义编译器常量，更可以用在模板中用于编译器分支选择，对于上面的用例，更简单的写法如下：
```cpp
template <typename T> void check_human() {
  if constexpr (has_run_v<T>) {
    std::cout << "T has run method\n";
  } else {
    std::cout << "T does not have run method\n";
  }
}
```

## C++20，新时代
C++20引入了`concept`和`require`的机制，让C++开发者可以丢弃`enable_if`这种复杂且丑陋的语法，直接看代码：
```cpp
template <typename T>
concept has_run_method = requires(T t) { t.run(1); };

template <has_run_method T> void runrunrun(T &&t) {
  t.run(1);
  std::cout << "T has run method\n";
}
```
这里直接用`concept`约束T类型必须有`run`方法，
`concept`的约束初看类似Rust中的`Trait bound`，不过还略微有些不同。日后可以写篇文章单独分析一下。
```rust
trait Comparable {
    fn less_than(&self, other: &Self) -> bool;
    fn equals(&self, other: &Self) -> bool;
}

fn sort<T: Comparable>(vec: &mut Vec<T>) {
    vec.sort_by(|a, b| a.less_than(b) || !b.equals(a));
}
```
