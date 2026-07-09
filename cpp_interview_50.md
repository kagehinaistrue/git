# 2025 大厂 C++ 后端 / 音视频岗 高频面试 50 考点

> **标记说明**：🔥 表示 Qt 程序员最容易答不上来的点（共 18 个），原因是 Qt 的信号槽/事件循环/MOC 等高层抽象屏蔽了底层细节。

---

## 一、C++ 语言核心（11 考点）

### 1. 右值引用与移动语义

**一句话结论**：`std::move` 不做移动，它只是无条件将左值转成右值引用，真正的移动发生在移动构造/赋值中。

```cpp
#include <vector>
#include <string>

class Buffer {
    std::vector<char> data_;
public:
    Buffer(size_t n) : data_(n) {}
    // 移动构造：直接"偷"走资源
    Buffer(Buffer&& other) noexcept : data_(std::move(other.data_)) {}
    // 移动赋值
    Buffer& operator=(Buffer&& other) noexcept {
        if (this != &other) data_ = std::move(other.data_);
        return *this;
    }
};

Buffer a(1024);
Buffer b = std::move(a); // a 被掏空，不可再使用
```

**常踩的坑**：`std::move` 之后继续使用源对象。被移动后的对象处于"合法但未指定"状态，调用 `a.size()` 返回 0 是符合标准的，但依赖任何具体值都会出 bug。

---

### 2. 完美转发与万能引用 🔥

**一句话结论**：`T&&` 在模板推导中不是右值引用，而是万能引用；配合 `std::forward<T>` 可保持实参的值类别原样转发。

```cpp
#include <utility>
#include <memory>

template <typename T, typename... Args>
std::unique_ptr<T> factory(Args&&... args) {
    // forward 保持每个 arg 的左/右值属性
    return std::make_unique<T>(std::forward<Args>(args)...);
}

struct Widget {
    Widget(int, double, std::string) {}
};

auto w = factory<Widget>(42, 3.14, std::string("hello"));
// 最后一个参数是右值，直接移动进构造
```

**常踩的坑**：拿 `std::move` 代替 `std::forward` 去转发——这会把左值参数强行转成右值，导致外部变量被意外修改或掏空。

---

### 3. RAII 与异常安全

**一句话结论**：把资源生命周期绑在栈对象的构造/析构上，就能自动保证异常安全，无需手动清理。

```cpp
#include <cstdio>
#include <stdexcept>

class FileHandle {
    FILE* f_;
public:
    explicit FileHandle(const char* path) : f_(std::fopen(path, "r")) {
        if (!f_) throw std::runtime_error("open failed");
    }
    ~FileHandle() { if (f_) std::fclose(f_); }
    FileHandle(const FileHandle&) = delete;
    FileHandle& operator=(const FileHandle&) = delete;
};
```

**常踩的坑**：析构函数抛异常——如果析构在栈展开时又抛异常，会直接 `std::terminate`。析构函数必须 `noexcept`。

---

### 4. 虚函数表 (vtable) 与动态派发 🔥

**一句话结论**：每个含虚函数的类有一张 vtable，对象首部存 vptr 指向它；虚调用 = 查表 + 函数指针调用，多一次间接寻址。

```cpp
#include <iostream>

struct Base {
    virtual void f() { std::cout << "Base\n"; }
    virtual ~Base() = default;
};
struct Derived : Base {
    void f() override { std::cout << "Derived\n"; }
};

Base* p = new Derived();
p->f();  // 通过 vptr -> vtable[0] 找到 Derived::f
delete p;
```

**常踩的坑**：构造/析构函数中调虚函数——构造时派生类还没初始化完毕，vptr 指向当前类的表，调不到派生类的 override。

---

### 5. 智能指针选型

**一句话结论**：`unique_ptr` 零开销，是默认选择；`shared_ptr` 有控制块开销（两次原子计数），仅在共享所有权时用；`weak_ptr` 打破循环引用。

```cpp
#include <memory>
#include <vector>

struct Node { std::vector<std::shared_ptr<Node>> children; };

// ❌ 循环引用
struct Bad { std::shared_ptr<Bad> other; };
auto a = std::make_shared<Bad>();
auto b = std::make_shared<Bad>();
a->other = b; b->other = a; // 永远不释放

// ✅ 用 weak_ptr 打破
struct Good { std::weak_ptr<Good> other; };
```

**常踩的坑**：从 `this` 拿 `shared_ptr` 时直接 `shared_ptr(this)`——这会产生第二套控制块，导致 double-free。应继承 `std::enable_shared_from_this<T>` 并用 `shared_from_this()`。

---

### 6. lambda 表达式与捕获列表

**一句话结论**：lambda 是匿名函数对象，默认值捕获 `[=]` 是 const 的，加 `mutable` 才可修改副本；引用捕获 `[&]` 要保证捕获对象生命周期长于 lambda。

```cpp
#include <functional>
#include <iostream>

auto make_counter() {
    int count = 0;
    return [=]() mutable { return ++count; }; // mutable 让 operator() 非 const
}
auto c = make_counter();
std::cout << c() << c(); // 输出 12
```

**常踩的坑**：`[=]` 捕获的指针——"值捕获指针"捕获的是指针本身的值，指向的对象仍可被外部修改，语义上等价于你持有同一把钥匙的副本。

---

### 7. SFINAE 与 Concepts 🔥

**一句话结论**：C++20 Concepts 用 `requires` 子句在编译期限制模板参数，彻底取代了用 `std::enable_if` 做 SFINAE 的"模板元编程地狱"。

```cpp
#include <concepts>
#include <vector>

// C++20: 简洁直观
template <std::integral T>
auto add(T a, T b) { return a + b; }

// C++17: 需要 enable_if
template <typename T>
typename std::enable_if<std::is_integral_v<T>, T>::type
add17(T a, T b) { return a + b; }
```

**常踩的坑**：concepts 是编译期约束，不会带来运行时开销，但嵌套 concept 出错时的编译报错仍然像以前一样长。用 `static_assert` 加友好信息是必备技巧。

---

### 8. constexpr 与编译期计算

**一句话结论**：C++17/20 大幅扩展了 constexpr 能力——编译期可分配内存、用 vector、甚至抛异常（但必须在编译期被捕获）。

```cpp
#include <array>

constexpr int fib(int n) {
    if (n <= 1) return n;
    return fib(n - 1) + fib(n - 2);
}

constexpr auto result = fib(20);          // 编译期算出 6765
std::array<int, fib(5)> arr;              // 编译期确定数组大小 = 5
```

**常踩的坑**：`constexpr` 不等于 `const`——`constexpr` 强调"编译期可求值"，`const` 强调"不可修改"。两者可以组合：`constexpr const int x = 42;`

---

### 9. 菱形继承与虚继承 🔥

**一句话结论**：虚继承用 `virtual public` 关键字解决菱形继承中共同基类的多份副本问题，代价是对象布局更复杂，访问基类成员多一次间接寻址。

```cpp
#include <iostream>

struct A { int data = 42; };
struct B : virtual public A {};
struct C : virtual public A {};
struct D : B, C {};

D d;
std::cout << d.data; // 只有一份 A::data，且 D 负责构造 A
```

**常踩的坑**：虚继承体系中最派生类负责构造虚基类，中间类对虚基类的构造调用会被忽略。Qt 的信号槽不与虚继承混用——MOC 对此支持不完善。

---

### 10. 三/五法则与 Rule of Zero

**一句话结论**：如果类管理了资源（指针、文件句柄等），就要显式定义析构/拷贝/移动；否则用 `=default`，让编译器自动生成，这就是 Rule of Zero。

```cpp
#include <memory>
#include <vector>

// Rule of Zero: 成员都是 RAII 类型，不写任何特殊函数
struct Widget {
    std::string name;
    std::vector<int> values;
    std::unique_ptr<int> cache;
    // 编译器自动生成正确的析构/拷贝/移动
};
```

**常踩的坑**：手写了析构函数但不写移动构造/赋值——编译器的隐式移动会被抑制，导致所有"移动"悄无声息地降级为拷贝。

---

### 11. 类型推导：auto、decltype、模板推导

**一句话结论**：`auto` 会剥离引用和顶层 const；`auto&&` 是万能引用；`decltype(auto)` 完整保留表达式的值和引用类别。

```cpp
#include <type_traits>

const int& foo();
auto a = foo();              // int（剥离引用和 const）
decltype(auto) b = foo();    // const int&（完整保留）
auto&& c = foo();            // const int&（万能引用折叠）
```

**常踩的坑**：`auto x = {1,2,3}` 推导为 `std::initializer_list<int>` 而非 `std::vector<int>`——这是 auto 唯一的特殊规则，花括号初始化列表是 auto 的死穴。

---

## 二、内存与对象模型（6 考点）

### 12. 内存对齐与 cache line 🔥

**一句话结论**：CPU 按 cache line（通常 64 字节）读取数据；未对齐或跨 line 的数据会增加延迟；`alignas` 可显式控制对齐，避免 false sharing。

```cpp
#include <new>

alignas(64) struct Aligned {
    int hot_data;   // 独占一个 cache line
};
alignas(64) char padding[64];

// false sharing 防护
struct alignas(64) PaddedCounter {
    volatile int64_t val = 0;
};
```

**常踩的坑**：多线程场景中两个线程分别写两个相邻的 `int`，它们落在同一 cache line 里，导致 cache 一致性协议疯狂抖动——性能下降几十倍。Qt 的 `QAtomicInt` 也不会帮你补 padding。

---

### 13. new/delete 与 placement new

**一句话结论**：`new` = 分配内存 + 调构造函数；`delete` = 调析构函数 + 释放内存。placement new 在已分配的内存上构造对象。

```cpp
#include <new>
#include <cstdlib>

// 手动分离两步操作
void* raw = std::malloc(sizeof(std::string));
auto* s = ::new(raw) std::string("hello"); // placement new
s->~basic_string();                         // 手动析构
std::free(raw);
```

**常踩的坑**：用 `new[]` 分配却用 `delete` 释放（不带 `[]`）——数组对象可能在前 8 字节存了元素个数，`delete` 按错误地址释放导致堆损坏。

---

### 14. 内存池与自定义分配器

**一句话结论**：`std::pmr::polymorphic_allocator` (C++17) 可运行时切换分配策略，配合 `std::pmr::monotonic_buffer_resource` 在音视频帧处理中大幅减少 malloc 次数。

```cpp
#include <memory_resource>
#include <vector>

// 预分配 1MB 栈缓冲，不分配时零堆开销
char buffer[1024 * 1024];
std::pmr::monotonic_buffer_resource pool(std::data(buffer), std::size(buffer));
std::pmr::vector<int> vec(&pool);
vec.reserve(1000); // 从 pool 分配，快过 malloc
```

**常踩的坑**：`monotonic_buffer_resource` 只能增长不能释放单个对象，只有销毁整个 resource 时才整体释放。适合一帧内用完即弃的场景。

---

### 15. 对象内存布局 🔥

**一句话结论**：C++ 标准保证成员按声明顺序排列；有虚函数的对象首部是 vptr；有继承时基类子对象在前；空基类优化 (EBO) 让空基类占 0 字节。

```cpp
#include <iostream>

struct Empty {};
struct A : Empty { int x; };

std::cout << sizeof(A); // 输出 4（不是 8，EBO 生效）

struct B { virtual ~B() = default; int x; };
// 布局：[vptr (8字节)] [x (4字节)] [padding (4字节)] = 16字节
```

**常踩的坑**：Qt 程序员的 `sizeof(QObject)` 远大于 0——因为 QObject 本身带 vptr 和 d_ptr，EBO 只在纯 C++ 中有意义。跨语言/跨平台结构体需要 `#pragma pack` 或 `alignas` 固定布局。

---

### 16. 内存序与无锁编程 🔥

**一句话结论**：`std::atomic` 的默认 `memory_order_seq_cst` 最安全也最慢；用 `acquire-release` 配对可实现高效的无锁队列。

```cpp
#include <atomic>
#include <thread>
#include <cassert>

std::atomic<bool> ready{false};
int data = 0;

// 写线程
void producer() {
    data = 42;
    ready.store(true, std::memory_order_release); // 保证上面的写入对消费者可见
}

// 读线程
void consumer() {
    while (!ready.load(std::memory_order_acquire));
    assert(data == 42); // 一定成立
}
```

**常踩的坑**：以为 `volatile` 能替代 `atomic`——`volatile` 只禁止编译器优化，不保证 CPU 层面的内存序，在多核环境下完全不是线程安全的。

---

### 17. copy-on-write (COW) 与 Qt 隐式共享 🔥

**一句话结论**：Qt 大量使用隐式共享（如 `QString`、`QByteArray`），读时共享一份数据，写时才真正复制；但多线程不加锁读也不是安全的一一引用计数是原子的但具体实现依赖版本。

```cpp
#include <QString>

QString s1 = "hello";
QString s2 = s1;          // 共享同一块堆数据，不复制
s2[0] = 'H';              // 触发 detach，真正复制
// s1 仍然是 "hello", s2 是 "Hello"
```

**常踩的坑**：在 STL 和 Qt 混用的代码里，直接对 `QString::data()` 做 const_cast 写入——这绕过了 COW 的 detach 机制，导致所有共享该字符串的副本都被修改。

---

## 三、并发与多线程（7 考点）

### 18. std::thread 与 std::jthread

**一句话结论**：C++20 的 `jthread` 相比 `thread` 多了两个关键改进：析构时自动 join（不 terminate），以及内置停止令牌 (stop_token)。

```cpp
#include <thread>
#include <chrono>

void worker(std::stop_token token) {
    while (!token.stop_requested()) {
        // 做事情，定期检查停止请求
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
    }
}

std::jthread t(worker);
// 作用域结束时 t 析构自动请求停止 + join，不会 terminate
```

**常踩的坑**：`std::thread` 析构时如果线程还是 joinable 状态，直接 `std::terminate`——这是 C++ 标准库最坑的设计之一。

---

### 19. mutex 与 lock 管理

**一句话结论**：永远不用裸 `lock()/unlock()`，用 `std::lock_guard`（简单场景）或 `std::unique_lock`（需要 defer/unlock 时），C++17 推荐 `std::scoped_lock` 同时锁多个 mutex 防死锁。

```cpp
#include <mutex>

std::mutex m1, m2;

void safe() {
    std::scoped_lock lock(m1, m2); // C++17，自动防死锁，同时锁定
    // 临界区
}

// 等价于手动版本：
void manual() {
    std::lock(m1, m2); // 原子地锁住两个
    std::lock_guard<std::mutex> lk1(m1, std::adopt_lock);
    std::lock_guard<std::mutex> lk2(m2, std::adopt_lock);
}
```

**常踩的坑**：同一个线程重复 lock 同一个 `std::mutex`（不可重入锁）→ 死锁。需要递归锁时用 `std::recursive_mutex`，但递归锁往往是设计缺陷的信号。

---

### 20. 条件变量与虚假唤醒

**一句话结论**：`wait` 必须带谓词（循环检查条件），因为条件变量存在虚假唤醒 (spurious wakeup)。

```cpp
#include <mutex>
#include <condition_variable>
#include <queue>

std::mutex m;
std::condition_variable cv;
std::queue<int> q;

// ❌ 错误写法
// cv.wait(lk); // 虚假唤醒时出错

// ✅ 正确写法：带谓词的 wait
void consumer() {
    std::unique_lock lk(m);
    cv.wait(lk, []{ return !q.empty(); }); // 等价 while(q.empty()) cv.wait(lk);
    int val = q.front(); q.pop();
}
```

**常踩的坑**：用 `notify_one` 唤醒时假定特定线程一定被唤醒——实际被唤醒的线程是任意的，你只能用条件来筛选它该做什么。

---

### 21. future/promise/async 模型

**一句话结论**：`std::async` 是获取 `future` 的最简单方式，但它的启动策略 `launch::async | launch::deferred` 会带来不确定性；固定用 `launch::async` 可避免。

```cpp
#include <future>
#include <iostream>

int compute() { return 42; }

auto fut = std::async(std::launch::async, compute); // 确定新线程
// auto fut = std::async(compute); // 不确定！可能是 deferred
std::cout << fut.get(); // 若 deferred，这里才真正执行
```

**常踩的坑**：`future` 的析构会阻塞等待——如果 `std::async` 返回一个 future 而你忘了存储它（临时对象立即析构），会变成同步阻塞调用。

---

### 22. thread_local 与线程局部存储

**一句话结论**：`thread_local` 变量每个线程独立一份，生命周期与线程绑定，常用于消除全局锁。

```cpp
#include <iostream>
#include <thread>

thread_local int tls_counter = 0; // 每个线程有自己的 counter

void work(int id) {
    ++tls_counter;
    std::cout << "thread " << id << ": " << tls_counter << "\n";
}
// 两个线程的输出都是 1，互不影响
```

**常踩的坑**：`thread_local` 变量有延迟初始化——只有被某线程首次 odr-used 时才构造。对性能敏感代码，首次访问可能触发锁（编译器内部的 TLS 初始化锁）。

---

### 23. 无锁 SPSC 队列实现 🔥

**一句话结论**：单生产者单消费者 (SPSC) 队列是无锁编程的入门经典——只用 ring buffer + 两个 atomic 游标就能实现，零 mutex 开销。

```cpp
#include <atomic>
#include <vector>

template <typename T>
class SPSCQueue {
    std::vector<T> buffer;
    std::atomic<size_t> head_{0}, tail_{0};
    const size_t capacity_;
public:
    explicit SPSCQueue(size_t cap) : buffer(cap + 1), capacity_(cap + 1) {}
    bool push(T item) {
        size_t h = head_.load(std::memory_order_acquire);
        size_t t = tail_.load(std::memory_order_relaxed);
        if ((t + 1) % capacity_ == h) return false; // 满
        buffer[t] = std::move(item);
        tail_.store((t + 1) % capacity_, std::memory_order_release);
        return true;
    }
    bool pop(T& out) {
        size_t h = head_.load(std::memory_order_relaxed);
        size_t t = tail_.load(std::memory_order_acquire);
        if (h == t) return false; // 空
        out = std::move(buffer[h]);
        head_.store((h + 1) % capacity_, std::memory_order_release);
        return true;
    }
};
```

**常踩的坑**：对多生产者/多消费者用同一套 SPSC——会丢数据或产生 data race。MPMC 需要 CAS 循环（`compare_exchange_weak`），复杂度完全不同。

---

### 24. 线程池设计要点 🔥

**一句话结论**：线程池核心 = 任务队列 + 工作线程 + 条件变量；生产级线程池必须考虑 backpressure（队列满时阻塞提交者）、优雅关闭、CPU affinity。

```cpp
#include <thread>
#include <vector>
#include <queue>
#include <mutex>
#include <condition_variable>
#include <functional>

class ThreadPool {
    std::vector<std::jthread> workers_;
    std::queue<std::function<void()>> tasks_;
    std::mutex m_;
    std::condition_variable cv_;
    bool stop_ = false;
public:
    explicit ThreadPool(size_t n) {
        for (size_t i = 0; i < n; ++i)
            workers_.emplace_back([this](std::stop_token st) {
                while (!st.stop_requested()) {
                    std::function<void()> task;
                    {
                        std::unique_lock lk(m_);
                        cv_.wait(lk, [&]{ return stop_ || !tasks_.empty(); });
                        if (stop_ && tasks_.empty()) return;
                        task = std::move(tasks_.front());
                        tasks_.pop();
                    }
                    task();
                }
            });
    }
    void enqueue(std::function<void()> f) {
        {
            std::lock_guard lk(m_);
            tasks_.push(std::move(f));
        }
        cv_.notify_one();
    }
    ~ThreadPool() {
        { std::lock_guard lk(m_); stop_ = true; }
        cv_.notify_all();
    } // jthread 自动 join
};
```

**常踩的坑**：析构顺序——在线程池析构前确保所有提交的 task 中持有的引用/指针仍然有效。某个 task 捕获了已析构对象的引用是最常见的 crash。

---

## 四、网络编程（5 考点）

### 25. TCP 粘包/拆包处理

**一句话结论**：TCP 是流式协议，没有消息边界。解决粘包的四种方式：固定长度、分隔符、长度前缀（最常用）、自描述协议。

```cpp
#include <cstdint>
#include <vector>
#include <cstring>

// 长度前缀方案：4字节头 + payload
std::vector<char> pack(const std::vector<char>& payload) {
    std::vector<char> result(4 + payload.size());
    uint32_t len = htonl(payload.size()); // 网络字节序
    std::memcpy(result.data(), &len, 4);
    std::memcpy(result.data() + 4, payload.data(), payload.size());
    return result;
}
```

**常踩的坑**：`recv` 返回的字节数可能小于你请求的字节数，不等于对方发了多少个 `send`。必须循环接收并拼包。Qt 的 `QTcpSocket` 也是基于事件的，不要假设一次 `readyRead` 就是一个完整包。

---

### 26. epoll / IOCP 模型对比 🔥

**一句话结论**：Linux 用 epoll (reactor)，Windows 用 IOCP (proactor)；epoll 通知你"可读了"，你自己调 read；IOCP 直接帮你完成读并把数据交给你。

```cpp
// epoll 模式（简化）
int epfd = epoll_create1(0);
epoll_event ev{};
ev.events = EPOLLIN | EPOLLET; // 边缘触发
ev.data.fd = socket_fd;
epoll_ctl(epfd, EPOLL_CTL_ADD, socket_fd, &ev);

epoll_event events[64];
int nfds = epoll_wait(epfd, events, 64, -1);
for (int i = 0; i < nfds; ++i) {
    // 必须循环读到 EAGAIN（边缘触发模式）
    while (recv(events[i].data.fd, buf, len, 0) > 0) { /*...*/ }
}
```

**常踩的坑**：epoll 边缘触发 (ET) 下，收到事件后必须读到 `EAGAIN`，否则剩余数据永远不会再触发通知。Qt 程序员习惯的 `QSocketNotifier` 默认是水平触发 (LT)，切到 ET 后常在此翻车。

---

### 27. HTTP/2 多路复用与 gRPC 基础

**一句话结论**：HTTP/2 用单一 TCP 连接上的多个 stream 实现并发请求，gRPC 基于此做 RPC，核心是 protobuf 序列化 + stream 帧模型。

```protobuf
// example.proto
service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}
message HelloRequest { string name = 1; }
message HelloReply { string message = 1; }
```

**常踩的坑**：HTTP/2 的 header 压缩 (HPACK) 和 stream 优先级在实际实现中差异巨大，不要假定所有 gRPC 实现（C-core / Rust tonic / Go）行为一致。

---

### 28. TCP 三次握手/四次挥手与 TIME_WAIT

**一句话结论**：TIME_WAIT 是主动关闭方必经状态，持续 2MSL；大量 TIME_WAIT 会耗尽端口，解决方案是 SO_REUSEADDR 或让客户端主动关闭。

```cpp
int opt = 1;
setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
```

**常踩的坑**：服务端在 TIME_WAIT 期间不能用同一个 (ip, port) 对重新启动——`SO_REUSEADDR` 允许绑定但可能收旧连接数据。正确做法是用 `SO_REUSEPORT` 或者确保旧连接完全消失。

---

### 29. WebSocket 协议帧格式

**一句话结论**：WebSocket 是消息边界协议，每个帧有操作码、mask、payload长度；客户端到服务端必须 mask，服务端到客户端不能 mask。

```cpp
// WebSocket 帧格式（简化）
struct WsFrame {
    uint8_t opcode : 4;
    uint8_t rsv3  : 1;
    uint8_t rsv2  : 1;
    uint8_t rsv1  : 1;
    uint8_t fin   : 1;
    uint8_t len   : 7;
    uint8_t mask  : 1;
    // 可变长度扩展 + mask key + payload
};
```

**常踩的坑**：payload 长度编码分三档（7位/7+16位/7+64位），解析时漏掉任何一档长度计算逻辑都会导致帧边界错位。Qt 的 `QWebSocket` 封装了这一层，手写协议时常低估其复杂度。

---

## 五、操作系统与系统编程（5 考点）

### 30. 虚拟内存与页表 🔥

**一句话结论**：每个进程有独立虚拟地址空间，OS 通过多级页表映射到物理内存；TLB 是页表的硬件缓存，miss 代价可达数百 cycle。

**常踩的坑**：mmap/munmap 后访问已释放的虚拟地址——直接 SIGSEGV。在音视频场景中把 GPU DMA 缓冲区 unmap 后驱动还在写，随机 crash 极难排查。

---

### 31. 进程间通信 (IPC) 选型

**一句话结论**：管道/消息队列（简单流式）、共享内存（高吞吐，需同步）、Unix domain socket（最灵活，支持 fd 传递）、信号量（同步原语本身不是 IPC）。

```cpp
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>

// 共享内存创建
int fd = shm_open("/my_shm", O_CREAT | O_RDWR, 0666);
ftruncate(fd, 4096);
void* addr = mmap(nullptr, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
// 配合信号量或 futex 同步
```

**常踩的坑**：共享内存忘记同步——多进程同时写同一块地址，不 crash 但数据错乱。Qt 的 `QSharedMemory` 自带锁机制，裸写 shm_open 时很多人忘了这一点。

---

### 32. CPU 缓存层次与亲和性 🔥

**一句话结论**：L1 Cache（32-64KB/核，~4 cycle）→ L2（256KB-1MB/核，~12 cycle）→ L3（8-32MB/共享，~40 cycle）→ 内存（~100ns = ~400 cycle）；跨 NUMA 节点代价更大。

```cpp
#include <pthread.h>

// 把线程绑到 CPU 核 3
cpu_set_t cpuset;
CPU_ZERO(&cpuset);
CPU_SET(3, &cpuset);
pthread_setaffinity_np(pthread_self(), sizeof(cpuset), &cpuset);
```

**常踩的坑**：以为把所有线程绑核一定更好——如果两个协作线程被绑在不同 NUMA 节点，共享数据的访问延迟翻倍。音视频管线中把 decoder 和 render 绑到同 NUMA 节点是优化关键。

---

### 33. 零拷贝技术：sendfile / splice / DMA-BUF 🔥

**一句话结论**：零拷贝 = 数据从磁盘到网卡不经用户态内存；Linux 用 sendfile/splice；音视频用 DMA-BUF 实现 GPU→显示零拷贝。

```cpp
#include <sys/sendfile.h>

// 文件直接发送到 socket，数据不经过用户态
off_t offset = 0;
sendfile(socket_fd, file_fd, &offset, file_size);
```

**常踩的坑**：sendfile 在输入不是常规文件 fd 时直接返回 EINVAL——不能从 socket 到 socket。音视频中 ffmpeg 的 `avio_alloc_context` 配合自定义 IO 时通常不走 sendfile，务必验证实际路径。

---

### 34. 信号处理与安全函数

**一句话结论**：信号处理函数中只能调 async-signal-safe 函数（man 7 signal-safety）；`printf`、`malloc`、`new` 都不安全；常用方案是 signalfd + 事件循环。

```cpp
#include <csignal>
#include <atomic>

std::atomic<bool> g_shutdown{false};

extern "C" void handle_sigint(int) {
    g_shutdown.store(true); // atomic 且无锁，信号安全
}
// signal(SIGINT, handle_sigint);
// 主循环检查 g_shutdown
```

**常踩的坑**：在信号处理函数里做复杂清理——deadlock（主线程持有 malloc 锁时被信号中断，handler 又调 malloc）。Qt 的 `QSocketNotifier` 配合 signalfd 是更安全的方案。

---

## 六、STL 与数据结构（6 考点）

### 35. vector 扩容机制与内存策略

**一句话结论**：vector 按倍数扩容（MSVC 1.5x，GCC/Clang 2x），扩容时重新分配 + 移动/拷贝旧元素；`reserve` 预分配可消除扩容抖动。

```cpp
#include <vector>
#include <iostream>

std::vector<int> v;
std::cout << v.capacity(); // 0
v.reserve(1000);
std::cout << v.capacity(); // 1000，后续 1000 次 push 不会重新分配
```

**常踩的坑**：`resize` vs `reserve`——`resize(n)` 分配 + 构造 n 个元素，`reserve(n)` 只分配不构造。对大数据量的 `resize` 后紧接着 `push_back`，会从 n+1 个开始而不是覆盖。

---

### 36. unordered_map 与哈希冲突

**一句话结论**：`unordered_map` 用拉链法处理冲突；C++11 的 bucket 接口可遍历冲突链；自定义 key 时必须同时提供哈希和相等比较。

```cpp
#include <unordered_map>
#include <string>

struct Key { int a, b; };
struct KeyHash {
    size_t operator()(const Key& k) const {
        return std::hash<int>()(k.a) ^ (std::hash<int>()(k.b) << 1);
    }
};
struct KeyEqual {
    bool operator()(const Key& x, const Key& y) const {
        return x.a == y.a && x.b == y.b;
    }
};

std::unordered_map<Key, std::string, KeyHash, KeyEqual> map;
```

**常踩的坑**：rehash 使所有迭代器失效——在遍历时插入可能导致死循环或漏遍历。Qt 的 `QHash` 迭代器更宽容，但规则不同，跨容器使用时容易混淆。

---

### 37. map vs unordered_map 选择

**一句话结论**：需要有序遍历或范围查询用 `map`(红黑树 O(logn))；只做点查用 `unordered_map`(哈希表均摊 O(1))。

```cpp
#include <map>
#include <unordered_map>

// 范围查询：按 key 顺序迭代
std::map<int, int> m;
auto lower = m.lower_bound(10); // O(log n)

// 点查优先
std::unordered_map<int, int> um;
um.find(42); // O(1) 均摊
```

**常踩的坑**：`unordered_map` 的最坏情况——如果哈希函数质量差（全部碰撞到一个 bucket），退化成 O(n)。标准库的 `std::hash<int>` 实现是恒等函数，连续整数 key 在特定 bucket_count 下可能全部碰撞。注：VS 中 `power_of_two` 策略缓解了此问题但 GCC 仍是质数 bucket。

---

### 38. std::string 的 SSO 优化

**一句话结论**：小字符串优化 (SSO) 让短字符串（通常 ≤15 字符 GCC/Clang，≤15 MSVC）存储在 string 对象内部而不是堆上。

```cpp
#include <string>
#include <iostream>

std::string s1 = "short";       // SSO，不分配堆
std::string s2 = "very long string that exceeds SSO buffer"; // 堆分配

// std::string 对象本身大小不变（通常 32 字节）
std::cout << sizeof(std::string); // 32
```

**常踩的坑**：移动一个 SSO 状态的 string——数据在栈上没法"偷"，std::move 后会真的复制短字符串，性能不如预期。

---

### 39. std::deque 的内部结构

**一句话结论**：deque 是分块数组（chunk array），每块固定大小，用指针数组索引；头尾插入 O(1)，中间插入/删除 O(n)；元素不保证连续。

```cpp
#include <deque>
std::deque<int> dq;
dq.push_front(1);  // O(1)，不需要移动已有元素
dq.push_back(2);   // O(1)
// 但 &dq[0] 和 &dq[1] 可能不在同一块内存！
```

**常踩的坑**：把 deque 当连续内存用——不能像 `vector` 那样传 `&dq[0]` 给需要裸指针的 C API。如果需要连续，老老实实用 vector 再按需插入。

---

### 40. std::priority_queue 与堆操作

**一句话结论**：`priority_queue` 默认是大顶堆（最大元素在顶）；底层是 `std::vector` + `make/push/pop_heap` 系列算法；自定义比较要传第二个模板参数。

```cpp
#include <queue>
#include <vector>
#include <functional>

// 小顶堆：最大音视频帧的 pts 在顶出
using Frame = std::pair<int64_t, int>; // pts, data
auto cmp = [](const Frame& a, const Frame& b) { return a.first > b.first; };
std::priority_queue<Frame, std::vector<Frame>, decltype(cmp)> pq(cmp);
```

**常踩的坑**：priority_queue 用 `std::less` 产生大顶堆，这和 `std::sort` 的语义相反（sort + less = 升序）。面试高频陷阱题。

---

## 七、音视频专项（8 考点）

### 41. H.264/H.265 编码基础

**一句话结论**：H.264 的核心是预测编码：帧内预测（同帧相邻块）+ 帧间预测（运动估计）+ 变换量化 + 熵编码（CABAC）。H.265 在同样画质下码率减半。

```bash
# ffmpeg 软编码 H.264
ffmpeg -i input.yuv -c:v libx264 -preset medium -crf 23 output.mp4
# ffmpeg 硬编码 H.265 (NVIDIA)
ffmpeg -i input.yuv -c:v hevc_nvenc -preset p4 output.mp4
```

**常踩的坑**：把 B 帧的参考关系搞混——B 帧可以参考前后帧，编码顺序 ≠ 显示顺序。解码后必须按 PTS 重排序才能正确显示。

---

### 42. YUV/RGB 色彩空间转换 🔥

**一句话结论**：Y 是亮度，UV 是色度；NV12（Y 平面 + 交织 UV 平面）是硬编解码最常用格式；YUV→RGB 的矩阵转换系数因 BT.601/BT.709/BT.2020 不同。

```cpp
// BT.601 limited range YUV to RGB (简化)
void yuv2rgb(uint8_t y, uint8_t u, uint8_t v,
             uint8_t& r, uint8_t& g, uint8_t& b) {
    int C = y - 16, D = u - 128, E = v - 128;
    r = clamp((298 * C + 409 * E + 128) >> 8);
    g = clamp((298 * C - 100 * D - 208 * E + 128) >> 8);
    b = clamp((298 * C + 516 * D + 128) >> 8);
}
```

**常踩的坑**：混淆 full range (0-255) 和 limited range (16-235) YUV——视频编码通常用 limited range，但屏幕抓取往往用 full range。不做 range 转换画面发灰或过曝。

---

### 43. FFmpeg 解码管线 🔥

**一句话结论**：FFmpeg 解码标准管线 = `avformat_open_input` → `avformat_find_stream_info` → `avcodec_find_decoder` → `avcodec_open2` → `av_read_frame` → `avcodec_send_packet / avcodec_receive_frame`。

```cpp
extern "C" {
#include <libavformat/avformat.h>
#include <libavcodec/avcodec.h>
}

AVFormatContext* fmt_ctx = nullptr;
avformat_open_input(&fmt_ctx, "input.mp4", nullptr, nullptr);
avformat_find_stream_info(fmt_ctx, nullptr);
int video_idx = av_find_best_stream(fmt_ctx, AVMEDIA_TYPE_VIDEO, -1, -1, nullptr, 0);
AVCodecContext* codec_ctx = avcodec_alloc_context3(nullptr);
avcodec_parameters_to_context(codec_ctx, fmt_ctx->streams[video_idx]->codecpar);
const AVCodec* codec = avcodec_find_decoder(codec_ctx->codec_id);
avcodec_open2(codec_ctx, codec, nullptr);

AVPacket* pkt = av_packet_alloc();
AVFrame* frame = av_frame_alloc();
while (av_read_frame(fmt_ctx, pkt) >= 0) {
    avcodec_send_packet(codec_ctx, pkt);
    while (avcodec_receive_frame(codec_ctx, frame) == 0) {
        // frame 就是解码后的 YUV 数据
    }
    av_packet_unref(pkt);
}
```

**常踩的坑**：`avcodec_send_packet` 返回 `AVERROR(EAGAIN)` 时不处理——这意味着解码器内部缓冲满了，需要先 `avcodec_receive_frame` 腾出空间再重试发送。

---

### 44. 音视频同步机制

**一句话结论**：三种同步策略——音频为主时钟（最常见）、视频同步到音频、外部时钟。核心是不断比较各流的 PTS 与主时钟的差值，决定丢帧还是等待。

```cpp
// 同步逻辑伪代码
double audio_clock = get_audio_clock();
double video_pts = frame->pts * av_q2d(time_base);
double diff = video_pts - audio_clock;

if (diff > 0.1) {
    // 视频超前，等待
    av_usleep((unsigned)(diff * 1000000));
} else if (diff < -0.01) {
    // 视频落后，丢帧
    av_frame_free(&frame);
    return; // 丢弃当前帧
}
// 在阈值内，正常渲染
```

**常踩的坑**：时间基 (time_base) 不统一——音频用采样率做时间基，视频用帧率相关的时间基。直接比较 PTS 前务必用 `av_rescale_q` 转换到同一时间基。

---

### 45. 推流协议：RTMP vs RTSP vs WebRTC

**一句话结论**：RTMP 延迟 ~1-3s 适合直播推流（已被 SRT 替代趋势）；RTSP 延迟 ~0.5-2s 适合安防监控；WebRTC 延迟 <500ms 但架构重，适合视频会议。

```cpp
// RTMP 推流（ffmpeg API，简化）
AVFormatContext* out_ctx = nullptr;
avformat_alloc_output_context2(&out_ctx, nullptr, "flv", "rtmp://server/live/stream");
// 添加视频/音频流...
avio_open(&out_ctx->pb, "rtmp://server/live/stream", AVIO_FLAG_WRITE);
avformat_write_header(out_ctx, nullptr);
// 循环 av_interleaved_write_frame(out_ctx, pkt);
```

**常踩的坑**：RTMP 底层 TCP 断开后不会自动重连——应用层必须实现重连 + 关键帧重传。WebRTC 的 ICE 连通率在内网环境很高，越复杂的 NAT 环境失败率越高，需要 TURN 兜底。

---

### 46. OpenGL/D3D 渲染管线与着色器

**一句话结论**：GPU 渲染管线：顶点着色器 → 光栅化 → 片元着色器 → 输出合并。音视频最常见的自定义 shader 是把 YUV→RGB 转换放到 GPU。

```glsl
// 片元着色器：NV12 → RGB (简化)
#version 330 core
uniform sampler2D texY, texUV;
in vec2 texCoord;
out vec4 fragColor;
void main() {
    float y = texture(texY, texCoord).r;
    vec2 uv = texture(texUV, texCoord).rg;
    // YUV → RGB 矩阵运算
    float r = y + 1.402 * (uv.y - 0.5);
    float g = y - 0.344 * (uv.x - 0.5) - 0.714 * (uv.y - 0.5);
    float b = y + 1.772 * (uv.x - 0.5);
    fragColor = vec4(r, g, b, 1.0);
}
```

**常踩的坑**：OpenGL context 只能在创建它的线程使用——Qt 中 `QOpenGLWidget` 封装了这一点，直接用原生 GL 在线程间传 context 会黑屏或 crash。

---

### 47. 音频 PCM 与重采样

**一句话结论**：PCM 是原始音频采样数据；重采样改变采样率/声道数/采样格式；libswresample 是 FFmpeg 生态中做重采样的标准工具。

```cpp
#include <libswresample/swresample.h>

SwrContext* swr = swr_alloc_set_opts(nullptr,
    AV_CH_LAYOUT_STEREO, AV_SAMPLE_FMT_S16, 48000,       // 输出
    AV_CH_LAYOUT_MONO,   AV_SAMPLE_FMT_FLTP, 44100, 0);  // 输入
swr_init(swr);

// 在循环中：输入的音频数据
uint8_t* in_data[] = { (uint8_t*)input_samples };
uint8_t* out_data[] = { output_buffer };
swr_convert(swr, out_data, out_samples, (const uint8_t**)in_data, in_samples);
```

**常踩的坑**：浮点 (FLTP) 和整型 (S16) 的 PCM 范围不同——float 归一化到 [-1.0, 1.0]，int16 范围 [-32768, 32767]。重采样时格式不匹配会导致音量爆表或静音。

---

### 48. GStreamer 管道与自定义插件

**一句话结论**：GStreamer 用 pipeline 连接 element 的 pad 构建媒体处理管线；C++ 中使用 `gst_init` + `gst_parse_launch` 或用 `Gst::Element` (gstreamermm)。

```cpp
#include <gst/gst.h>

gst_init(nullptr, nullptr);
GstElement* pipeline = gst_parse_launch(
    "filesrc location=test.mp4 ! qtdemux ! h264parse ! avdec_h264 ! "
    "videoconvert ! autovideosink", nullptr);
gst_element_set_state(pipeline, GST_STATE_PLAYING);
```

**常踩的坑**：pad 的动态连接——demuxer 的 src pad 在收到数据后才创建，必须用 `pad-added` 信号回调来动态连接。静态 pipeline 字符串对 demuxer 无效。

---

## 八、安全与调试（4 考点）

### 49. 缓冲区溢出与 AddressSanitizer

**一句话结论**：缓冲区溢出是最经典的 C/C++ 安全漏洞，ASan (AddressSanitizer) 在编译时插入红区守卫，运行时检测越界读写。

```bash
g++ -fsanitize=address -g -O1 bug.cpp -o bug
./bug  # ASan 精确报告越界的行号和偏移
```

**常踩的坑**：ASan 会把内存开销放大 ~2x，不能用于生产二进制。release 模式下要关掉，但 debug 阶段的栈溢出 ASan 不一定能检测（需要 ASAN_OPTIONS=detect_stack_use_after_return=1）。

---

### 50. UAF (Use After Free) 与智能指针防护 🔥

**一句话结论**：UAF 是 C++ 安全漏洞之王——释放后继续使用指针；改用 `unique_ptr`/`shared_ptr` 可以从设计上消除 UAF，但 `shared_ptr` 循环引用仍是死角。

```cpp
#include <memory>
#include <iostream>

// ❌ UAF 典型场景
int* p = new int(42);
delete p;
std::cout << *p; // UAF! 行为未定义

// ✅ 智能指针防护
auto sp = std::make_unique<int>(42);
// sp 离开作用域自动释放，不存在 UAF
```

**常踩的坑**：`shared_ptr` 循环引用导致的内存泄漏比 UAF 更难排查——内存分析工具看到的是还在引用图中的对象，不报告泄漏。Qt 的 QObject 父子树用 `deleteLater()` 避免了大部分 UAF，但跨越信号槽的生命周期问题仍然存在。

---

## 九、Qt 程序员最易失分的 18 个点汇总

| 序号 | 考点 | 失分原因 |
|------|------|----------|
| 2 | 完美转发 | Qt 信号槽自动处理参数传递，很少手写模板转发 |
| 4 | vtable 机制 | MOC 生成元对象系统，隐藏了虚函数表细节 |
| 7 | C++20 Concepts | Qt 依赖 MOC，C++20 新特性在 Qt 项目中普及较慢 |
| 9 | 虚继承 | Qt 极少使用多重继承（QObject 除外），MOC 也对它不友好 |
| 12 | 内存对齐 | Qt 容器封装好了内存管理 |
| 15 | 对象内存布局 | QObject 有 vptr + d_ptr + 元对象，裸 C++ 布局反而陌生 |
| 16 | 内存序 | QReadWriteLock/QMutex 屏蔽了原子操作细节 |
| 17 | COW/隐式共享 | 这是 Qt 强项，但底层实现细节可能不熟 |
| 23 | 无锁队列 | Qt 事件循环驱动，多线程通信用信号槽 |
| 24 | 线程池设计 | QtConcurrent 封装了线程池 |
| 26 | epoll vs IOCP | Qt 网络模块抽象了平台差异 |
| 30 | 虚拟内存 | Qt 应用层开发很少触及 |
| 32 | CPU 缓存/亲和性 | Qt 框架屏蔽了底层 |
| 33 | 零拷贝 | Qt 内存管理是拷贝为主 |
| 42 | YUV 色彩空间 | 纯 UI 开发不接触视频像素 |
| 43 | FFmpeg 管线 | Qt Multimedia 封装了底层解码 |
| 44 | 音视频同步 | 同上，QMediaPlayer 对外屏蔽了同步逻辑 |
| 50 | UAF | Qt 的 QObject 树 + deleteLater 减少了手动内存管理错误 |

---

> **最后的话**：Qt 是杰出的应用框架，但大厂音视频/后端面试考察的是你对 C++ 语言本身、操作系统、网络协议、编解码管线的底层理解。那些被 Qt 优雅封装掉的细节，恰恰是面试官最想听的。
