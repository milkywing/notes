## C++17

### optional

头文件 `<optional>` 提供 `std::optional<T>` 类模板用于包装一个可空的值。一个 `optioanl` 对象要么包含一个类型为 `T` 值，要么包含一个 `std::nullopt` 表示为空。一个空的 `optioanl` 对象在条件表达式中会隐式的转换为 `false`，用于判断是否有值

在确定有值的情况下，可以像使用指针一样使用 `*/->` 访问 `optional` 对象中的当前值。成员函数 `op.value()` 同样可以获取当前值，但是会在值不存在时抛异常，成员函数 `op.value_or(def)` 则可以在值不存在时返回 `def` 作为替代

> 使用 `*/->` 访问 `optional` 对象不会进行非法访问的检查，访问前需确保值存在

### variant

头文件 `variant` 提供 `std::variant<T...>` 类模板包装一个可能有多个类型的值。一个 `variant` 对象可以直接被赋值为其支持的类型的值，由对象自动管理旧类型值的销毁和新类型值的构建

访问 `variant` 对象中保存的值必须知道当前值的类型，然后通过 `std::get<T>(v)` 获取当前值的引用。或者知道当前值类型在模板实参列表的下标，然后通过 `std::get<N>(v)` 获取当前值的引用。为了确定当前值的类型或类型下标，需要使用如下函数

- `std::holds_alternative<T>(v)`，判断当前值的类型是否为 `T`
- `v.index()`，返回当前值的类型在模板实参列表的下标

```c++
std::variant<int, float> var = 1;
// 如果当前值为整型，以该类型获取
if (std::holds_alternative<int>(var)) {
  std::cout << std::get<int>(var); // 1
}
// 如果当前值的类型为第二个类型，以该类型获取
var = 2.0;
if (var.index() == 1) {
  std::cout << std::get<1>(var); // 2.0
}
```

为了处理 `variant` 对象，可以使用 `std::visit(callable, vars...)` 将不同类型的处理方式“打包”成一个可调用对象，根据实际的类型选择对应的处理方式。“打包”的方式可以是 `()` 运算符的重载，也可以是自带模板特性的 `auto lambda`，后者会在编译阶段生成所有可能类型的实例，确保运行时能处理所有可能的情况

```c++
template<class... Fs>
struct overload: Fs... {
  using Fs::operator()...;
};
// 将多个函数包装成一个可调用对象
template<class... Fs>
overload(Fs const &...) -> overload<Fs...>;

std::variant<int, std::string> var = "123";
// 1. 分发到可调用对象的不同重载上
std::visit(overload(
  [] (int v) {/* 处理 int 的情况 */},
  [] (std::string v) {/* 处理 string 的情况 */},
), var);

// 2. 分发到 auto lambda 的不同实例上
std::visit([] (auto v) {
  if constexpr (std::is_intergral<decltype(v), int>){
    // 处理 int 的情况
  }else {
    // 处理 string 的情况
  }
}, var);
```

### string_view

`string_view` 底层由一个指向起始位置的指针和一个长度组成，用于表示字符串视图的**只读引用**，`string_view` 可以构造自 `string/const char*`

- `string_view(cs)`，构造对字符串 `cs` 的完整引用
- `string_view(cp, [n])`，构造对常量 C 风格字符串 `cp` 范围 `[0, n)` 的引用

`string_view` 只是一个只读引用，本身并不管理内存的分配和回收，因此拷贝一个 `string_viwe` 对象只是拷贝了内部的指针和长度，底层的字符串并不会被拷贝

> 和指针/引用类似，如果源字符串因某种原因被销毁，`string_view` 对象将失效，对其访问的行为是未定义的
>
> `string_view` 的成员函数基本上对应着 `string` 的成员函数（除了部分会修改字符串本身的成员函数）

各种字符串类型的转换规则

| 源类型      | explicit? | 目标类型    |
| ----------- | --------- | ----------- |
| const char* | No      | string_view |
| string      | No      | string_view |
| const char* | No      | string      |
| string_view | Yes      | string      |

`string/const char*` 可以隐式的转换为 `string_view`，`string_view` 类型的函数形参可以接受各种形式的常量字符串，避免函数重载和实参拷贝

### 结构化绑定

==结构化绑定==允许将一个静态数组中的元素或者结构体中的成员按照其顺序绑定到指定的变量序列中，由编译器推断各个变量的类型。其基本形式为 `auto [v1, v2] = arr/struct`，和平时的 `auto` 类型推断一样，`auto` 关键字前可选配 `const` 限定符，关键字后可选配 `&/&&` 说明符表示引用绑定

> 结构化绑定要求全量绑定，不能只绑定部分成员，实在用有用不上的变量可以加上 `[[maybe_unused]]` 属性抑制警告

```c++
struct Person {
  string name;
  unsigned int age;
}

int arr[3] = {1, 2, 3};
Person p{"student", 24};
// 拷贝绑定
auto [e1, e2, e3] = arr;
// 引用绑定
auto& [name, age] = p;
```

### 初始化语句

允许在 `if/switch` 判断之前额外执行一条表达式，放在条件之前，用分号分隔

```c++
if (int flag = false; condiction) {
  flag = true;
}
// 等价于
int flag = false;
if (condiction) {
  flag = true;
}
```

### misc

- 类模板的模板参数可以自动推导了，不用显式指定，如 `vector v = {1, 2};` 将自动推导出 `vector<int>`

## C++20

### misc

- 普通函数/labmda 可以将参数声明为 `auto` 让编译器根据实参推断参数类型，其效果等价于函数模板
- lambda 也可以模板化了，只需在捕获列表之后加上模板参数列表即可，如 `[] <typename T> (T a) {return a * 2;}`