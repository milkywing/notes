## 模板进阶

### 类型萃取

通过定义一个模板类及其特例化，利用模板匹配的机制，使得该模板可以查询或修改给定类型的属性，称为==类型萃取==。头文件 [`<type_traits>`](https://www.apiref.com/cpp-zh/cpp/types.html) 提供了大量萃取相关的模板

其中的一大类是形如 `is_xxx` 的模板，这类 `is_xxx` 模板有着一个布尔成员 `value` 表示给定的类型是否满足指定的要求，在 C++17 中还会额外提供类型别名 `is_xxx_v` 直接指向 `value` 成员。比如 `is_same/is_same_v` 可以判断两个给定的类型是否严格相等

另一类名字没有固定格式的模板是用来做类型转换的，这类模板有着一个类型成员 `type` 表示转换后的类型，同样的 C++17 为他们额外提供了类型别名（通常叫 `xxx_t`）直接指向 `type` 成员。比如 `decay/decay_t` 可以将给定的类型退化（去除 `const` 及引用修饰）

> 还有一个特殊的模板函数 `declval`，他可以接受一个指定的类型并凭空构造出该类型的值出来，即使该类型的构造函数是删除的。通常配合 `decltype` 使用

### SFINAE

> SFINAE 是 「Substitution Failure Is Not An Error」 的缩写，大意为 「替换失败不是错误」
>
> 在 C++20 中，更推荐使用 `requires` 表达式来替代 SFINAE

```c++
/* SFINAE 机制示例1 */
struct X {
  typedef int type;
};

template <class T>
void foo(T);
template <class T>
void foo(typename T::type); // 假定类型 T 有 type 类型成员

void call_foo() {
  foo<X>(5);    // 因为 X != int，第一个模板匹配失败，采用第二个模板
  foo<int>(15); // 因为 int 没有 type 成员类型，第二个模板匹配失败，采用第一个模板
}

/* SFINAE 机制示例2 */
// 求取数组的长度（直接返回数组静态大小）
template <class T, std::size_t N>
constexpr std::size_t len(T (&)[N]) {
  return N;
}
// 通过调用容器的 size 成员函数，求取容器的长度
template <class T>
// 利用逗号表达式总是返回右边的值，在 decltype 中进行类型假定（类型 T 拥有成员函数 size）
auto len(T const &t) -> decltype(std::declval<T>().size(), T::size_type()) {
  return t.size();
}

void call_len() {
  int arr[5];
  std::vector<int> vec{1, 2, 3};
  len(arr); // 因为 int[] 类型没有 size 成员函数，第一个模板匹配失败，采用第二个模板
  len(vec); // 因为无法推断出 N，第二个模板匹配失败，采用第一个模板
}
```
对于特定的一个模板调用，编译器会使用显式指定的类型或者推断出来的类型去替换每个可用模板中的模板参数，如果替换后的声明部分是是非法或无意义的，编译器会把该模板从候选中剔除，而不是直接报错终止编译流程，这种机制称为 SFINAE

利用 SFINAE 机制，可以在模板及其特例化中编写带类型假定的声明，这样不满足假定的的类型就会跳过该候选，满足该假定的类型就会采用该候选，从而将不同的类型分配到不同的模板实现上，`type_traits` 中很多萃取模板是利用 SFINAE 机制实现的

------

为了更方便的利用 SFINAE，可以使用 `enable_if` 和 `void_t` 这两个萃取模板进行类型约束

`enable_if` 类模板有两个参数，第一参数是布尔类型，第二参数是默认为 void 的类型参数。仅当第一参数为 `true` 时采用带有类型成员 `type`（指向第二参数）的结构体，否则采用一个空结构体

其固定用法为 `enable_if_t<Con, T> = t`，第一参数放置约束条件，第二参数放置带默认值的占位类型，仅当满足约束条件时对 `type` 成员的访问是合法的（模板是可用的）

```c++
// enable_if 的实现
template <bool, class T = void>
struct enable_if {};

template <T>
struct enable_if<true, class T> {
  using type = T;
};

template<bool Con, class T = void>
using enable_if_t = typename enable_if<Con, T>::type;

// enable_if 的使用
#define REQUIRES(x) std::enable_if_t<(x), int> = 0
template <class F, REQUIRES(std::is_void_v<std::invoke_result_t<F>>)>
auto invoke(F f) {
  std::cout << "f has no return value";
  f();
}

template <class F, REQUIRES(!std::is_void_v<std::invoke_result_t<F>>)>
auto invoke(F f) {
  std::cout << "f has return value";
  return f();
}
```

`void_t` 其实是 `void` 的套壳别名，强行弄出模板参数出来是为了让开发者能在参数中编写类型假定，仅当满足假定时 `void` 的推断成功（模板是可用的）

```c++
// void_t 的实现
template<typename...> using void_t = void

// 使用 void_t 来实现一个类模板，用于查询给定类型是否有 do_print 成员函数
template <class T, class Dummy = void>
struct has_do_print {
  static constexpr bool value = false;
};

template <class T>
// 在 void_t 中假定类型 T 有 成员函数 do_print
struct has_do_print<T, std::void_t<decltype(sdt::declval<T>().do_print())>> {
  static constexpr bool value = true;
};
```
> 通常 `void_t` 用来约束类型**是否拥有**某个成员，常用于实现自定义的 `is_xxx` 萃取模板，然后供 `enable_if` 使用

### 条件编译（C++17）

==条件编译==指的是有 `constexpr` 修饰的 `if-else` 代码块，条件编译只在编译阶段执行，因此只能使用常量表达式作为条件，对于一个特定的调用，编译器只会保留条件为真的部分作为实例化结果

> 对于小规模的分支逻辑条件编译是很好的解决方案，如果规模较大还是建议利用 SFINAE 将分支逻辑拆分到不同的模板实现上

```c++
#include <type_traits>

template <class T>
auto add_one_for_int(T value) {
  // is_integral_v 用于判断一个类型是否为整型
  if constexpr (std::is_integral_v<T>) {
    return ++value;
  } else {
    return value;
  }
}

auto res1 = add_one_for_int(114); // 相当于实例化 add_one_for_int(int value) {return ++value;}
auto res2 = add_one_for_int("114"); // 相当于实例化 add_one_for_int(const char *p) {return value;}
```

### 模板嵌套模板

模板的模板参数甚至可以是模板，当模板参数是模板时，需要额外加上模板的声明用以约束模板的入参，之后便可当成模板在模板中使用

或者模板参数是一个带模板成员的结构体，通过作用域运算符访问其中的模板

```c++
// 方案一，直接传入模板
template <template <class... Ts> class Tpl, class... Ts>
struct tuple_apply {};

template <template <class... Ts> class Tpl, class... Ts>
struct tuple_apply<Tpl, std::tuple<Ts...>> {
  using type = Tpl<Ts...>;
};
// variant<int, double>
using what1 = tuple_apply<std::variant, std::tuple<int, double>>::type; 

// 方案二，将模板包装成一个结构体，之后通过 ::template 访问其中的模板
template <std::size_t N>
struct array_wrapper {
  template <class T>
  struct rebind {
    using type = std::array<T, N>;
  }
};

template <class TplStruct, class Tup>
struct tuple_map {};

template <class TplStruct, class... Ts>
struct tuple_map<TplStruct, std::tuple<Ts...>> {
  using type = std::tuple<typename TplStruct::template rebind<Ts>::type...>;
};
// tuple<array<3,int>, array<3,double>>
using what2 = tuple_map<array_wrapper<3>, std::tuple<int, double>>::type;
```

### requires 表达式（C++20）

TODO

### misc

含参数包的表达式可以展开，在 C++17 后，展开操作还可以和`+-*/&|,`等各种运算符结合

- 展开操作直接作用于参数包，等价于参数包中的参数以逗号分隔
- 展开操作邻接一个运算符（左右皆可）作用于一个带参数包的表达式，等价于这个表达式被参数替代后以该运算符分隔

```c++
template<class... Args>
void foo(Args... args) {
  // 假如参数包为 1,2,3
	auto res = (args + ...); // 展开为 res = 1 + 2 + 3;
  ((std::cout<< args), ...); // 展开为 (sdt::cout << 1), (sdt::cout << 2), (sdt::cout << 3);
}
```

## 宏进阶

### XMarco

是一种生成生成 list-like 数据或代码结构的宏应用，由存储序列的宏函数和遍历序列的宏函数共同组成，通过定义遍历宏函数并传递给序列宏函数实现序列的映射，这样需要修改序列的时候只需修改序列宏函数即可，其他相关代码由遍历宏函数自动映射出来，常用于枚举的定义

```c++
// 定义一个宏函数以存储序列，接受另一个宏函数来遍历该序列
#define FOR_LOG_LEVEL(f) \
	f(debug) \
	f(warn) \
	f(error)

enum class log_level {
// 该遍历宏函数用于生成逗号分隔的序列
#define _FUNC(name) name,
  FOR_LOG_LEVEL(_FUNC)
#undef _FUNC
};

inline std::string log_level_name(log_level lev) {
  switch (lev) {
// 该遍历宏函数用于生成 case 分支序列
#define _FUNC(name) case log_level::name: return #name;
    FOR_LOG_LEVEL(_FUNC)
#undef _FUNC
  }
}

inline log_level log_level_from_name(std::string lev_name) {
// 该遍历宏函数用于生成 if 分支序列
#define _FUNC(name) \
  if (lev_name == #name) return log_level::name;
  FOR_LOG_LEVEL(_FUNC)
#undef _FUNC
  return log_level::debug;
}
```

