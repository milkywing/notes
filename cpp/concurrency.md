### 计时支持

> 由头文件 `<chrono>` 提供

包含三大类型，他们都定义在 `std::chrono` 命名空间下

- ==时钟==可通过成员函数 `c.now()` 获取当前时刻，根据计时起点的不同存在不同类型的时钟
  
  - `system_clock`，系统时钟，通常对应系统的 UTC 时间
  - `steady_clock`，单调时钟
- ==时间段 duration<T, Period>==表示两个时刻的时间间隔，其中 `Period` 表示单位，`T` 表示该单位下的数值类型。`std::chrono` 提供了常用单位下的类型别名，如 `milliseconds/seconds/minutes`，可使用这些预设类型来构造时间段对象
- ==时间点 time_point<Clock, duration>==表示当前时钟的某个时刻，本质是计时起点 + 时间段，因此时间点也隐含着单位

时间点和时间段之间可以相互运算，如时间点作差可得到时间段，时间点加减时间段得到新的时间点；当不同单位的对象参与运算时，其结果单位会隐式的转换到更高精度的单位上

不同精度单位进行隐式转换时，仅当没有潜在的精度丢失时转换才会成立，使用 `duration_cast<to_duration>()` 则可以忽略精度丢失进行强制转换，具体规则可看下面示例

```c++
std::chrono::milliseconds i_ms{3}; // 3ms
auto d_us = std::chrono::duration_cast<std::chrono::duration<double, std::micro>>(i_ms); // 3000.0us
// 允许构造自低精度单位
std::chrono::microseconds i_us = i_ms; // 3ms -> 3000us
// 当源数值单位为整型时，禁止构造自高精度单位，因为有可能被截断导致精度丢失
std::chrono::milliseconds from_i_us = i_us; // error
// 当源数值单位为浮点时，可以构造自任意精度的单位 (C++17)
std::chrono::milliseconds ms_from_d_us = d_us;
```

### 多线程/异步支持

> 多线程支持由头文件 `<thread>` 提供
>
> 需要注意的是该库的实现在不同系统上是不一样的，需要在 cmake 中加上依赖，使其根据系统链接不同的线程实现
>
> ```cmake
> find_package(Threads REQUIRED)
> target_link_libraries(target PUBLIC Threads::Threads)
> ```

`thread` 类型负责单个线程的管理，其构造函数接受一个无返回值的可调用对象作为线程任务。`thread` 对象构造时将创建一个系统级别的线程，销毁时终止线程任务并回收线程资源（可调用成员函数 `t.detach()` 解除 `thread` 对象对线程的所有权来阻止销毁时的回收行为）。调用成员函数 `t.join()` 以等待线程的结束

```c++
int main() {
  // 创建线程
  std::thread t([] () {
    std::this_thread::sleep_for(std::milliseconds(1000));
    std::cout << "sleep 1s finish\n";
  });
  // 主线程工作...
  std::cout << "working in main_thread\n"
  // 等待线程结束
  t.join();
  return 0;
}
```

> 异步支持由头文件 `<future>` 提供

`future<T>` 类型表示一个异步任务的未来返回值（其中 `T` 为异步任务返回值类型），可使用 `std::async(callable)` 来创建一个线程来执行异步任务，返回一个 `future` 对象。调用成员函数 `f.wait()` 可以等待异步任务的完成，或者调用 `f.wait_for(duration)` 可以在一段时间内等待异步任务并返回一个枚举值表示这段时间内任务是否完成。为了获取异步任务的返回值必须调用 `f.get()`，若任务未完成则会等待至任务完成

```c++
// 创建一个线程用于执行异步任务（相当于 new Promise()）
std::future<int> f = std::async([] () { return 114514; });
// 获取异步任务的返回值，若任务未完成，等待任务的完成（相当于 await promise）
int res = f.get();
```

> 因为管理着线程资源，`thread` 对象和 `future` 对象都是不可拷贝的。如果有共享的需要，可以使用可浅拷贝的 `shared_future`

### 互斥量（锁）

> 由头文件 `<mutex>` 提供

互斥量有多种类型，如基础互斥量 `mutex`、带时限互斥量 `timed_mutex` 、递归互斥量 `recursive_mutex` 和共享互斥量 `shared_mutex`。所有类型的互斥量都有 `m.lock()`、`m.try_lock()`、`m.unlock()` 三个成员函数，用于上锁和释放锁，使得被锁区间的代码只有一个线程执行。`m.lock()` 和 `m.try_lock()` 的区别是前者会等待锁的释放而后者会立即返回一个布尔表示是否取得锁的所有权

不同的互斥量的说明如下

- `timed_mutex`，额外提供成员函数 `tm.try_lock_for(duration)/tm.try_lock_until(time_point)` 在一段时间内等待锁的释放
- `recursive_mutex`，允许在同一线程内重复上锁而不引发阻塞（计数），并在计数归零后才释放锁
- `shared_mutex`，读写锁，额外提供成员函数 `sm.lock_shared()`、`sm.unlock_shared()` 用于上读锁和释放读锁

除了手动管理互斥量，还可以使用以下工具类来管理互斥量，他们的构造函数都接受互斥量对象

- `lock_gurad`，会在构造的时候自动上锁，在销毁时自动释放锁（基于作用域）
- `unquie_lock`，在 `lock_guard` 的基础上暴露 `lock/unlock()` 成员函数允许开发者中途上锁或释放锁
- `scoped_lock`，多互斥量版本的 `lock_guard`，统一对多个互斥量进行上锁或释放锁
- `shared_lock`，专供 `shared_mutex` 的 `lock_gurad`，会在构造的时候自动上读锁，在销毁时自动释放读锁

```c++
std::mutex mtx;
void task1() {
  mtx.lock();
  // 需要上锁的区域
  mtx.unlock();
}
void task2() {
  std::lock_guard(mtx);
  // 下面都是上锁的区域，直至退出函数时释放锁
}
```

### 条件变量

> 由头文件 `<condition_variable>` 提供

条件变量用于线程间的通信，他的 `wait` 和 `notify` 成员分别供通信的双方使用

- `cv.wait(lockable, predicate)`，阻塞当前线程，直至收到通知且条件满足（如果有设置条件的话）时解除
  - `wait` 陷入阻塞期间会暂时释放锁，`wait` 阻塞接触后会执行上锁操作，因此调用 `wait` 之前要求已经是上锁状态
- `cv.notify_one()`，通知任意一个在等待的线程
- `cv.notfy_all()`，通知所有在等待的线程

```c++
std::mutex mtx;
std::condition_variable cv;

boolean ready = false;

std::thread t1([&] () {
	std::unique_lock lck(mtx);
	cv.wait(lck, [&] () { return ready; });
  // cv 被唤醒后，会进行上锁，如果之后的代码允许多个线程同时执行，可在 wait 之后手动释放锁
  std::cout << "执行上锁区域代码\n";
});

ready = true;
// 通知任意一个已经在等待中的线程
cv.notify_one();
t1.join();
```
