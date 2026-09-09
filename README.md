## 面试



### 问题总结

我把这一年的问题归成四类：内存对象、编译器优化、类型系统、编码逻辑。

后来发现偶发崩溃大多数都命中了，所以总结了一套排查流程。

  Step 1: **控制变量**，先把问题压到最小复现案例，确认是哪个函数直接失败

  Step 2: **数据完整性**，比如 CheckSum 是否正确，这能帮我判断问题出在数据本身，还是后续处理（一致性问题）

  Step 3: **静态粗查**，先看有没有明显的逻辑错误、类型宽度问题、TODO、double free 这类常见坑

  Step 4: **动态调试**，如果崩溃，我用 GDB 抓第一现场，重点看空指针、越界、寄存器错位；如果不崩溃，就回到静态细查，配合日志输出，根据 CheckSum 错误阶段慢慢收敛



### 三个经典问题

1. *符号扩展*

两个层次，简单的是同宽度有无符号比较，有符号是负数时，隐式转换带来的逻辑求补；

当宽度不同时，比如 `int` 和 `int64_t` 比较，64位可能是拼接或无符号转储而来，`int` 是负数时隐式提升会带来高 32 位全是1。

我是看的汇编。应该是 `ja` 的但代码里生成的是 `jg`，说明编译器当有符号比较了。打印了高 32 位的数据，确认全 1 就锁定了。

2. *寄存器错位*

也是两种情况，第一种是 `this` 没有被用到。按照 ABI，成员函数的 `this` 放在 `rdi`。但编译器发现没用到，可能把原本 `rsi` 的参数塞进 `rdi`。优化本身合法，但如果调用方和被调用方的编译假设不一致，就会崩；

第二种，当非引用函数返回值超过 128 位时，ABI 规定 `rdi` 用来存返回对象的地址，`rsi` 放 `this`，如果碰到多继承的 thunk 调整，寄存器分配很复杂，做逆向时这种错位经常遇到 ABI 层的问题，先确认根因，再把影响面评估清楚，然后和主管一起决定修法。因为寄存器行为可能波及其他模块

3. *标签指针*

标签指针我做过一个类型复用的优化。简单说就是把 `T*` 和一个 `std::set<T>*` 复用同一个指针，用低 3 位做标记。

使用前先看低 3 位。如果全是 0，那就是普通的 `T*`，直接走正常逻辑。如果有标记，就先 `& 0xFFFFFFFFFFFFFFF8` 把低 3 位清掉，再按 `std::set<T>*` 去用。

这个方案在64位平台是成立的，性能较好。缺点是类型系统被破坏，需要手动解析，后来有一个 bug 就是因为 `std::hash<Ptr>` 的会 `>> 3` 导致标记判断失效出错，所以如果再让我选，我会把这种优化技巧限制在极小的局部，有扩散行为的不要去用。



### 修复的回归性证明

  以符号扩展为例，我追到根上是一个成员变量被错声明成了 64 位。查了所有用到它的地方，布局、布线、timing 的汇编都只取低 32 位。所以我把变量类型从源头拆开，回归全过才提交。后来凡看到这种高低位取用，我先怀疑根类型错了。



## 笔试

### 1. *RingBuffer

> 环形缓冲区，低延时、无锁、固定大小。

写时判断 `free_space >= len`，从 `tail_` 开始写，`tail_ = (tail_ + len) % capacity_`，更新 `size_`。读同理。

**Q**: 为什么用环形缓冲区？

**A**: 因为数据采集是连续的，缓冲区满了可以选择覆盖旧数据（实时系统常见策略），保证新数据不被阻塞。

```cpp
class Ring {
    vector<char> buf;
    size_t head = 0, tail = 0, cnt = 0, cap;
public:
    Ring(size_t n) : buf(n), cap(n) {}

    bool write(const char* p, size_t len) {
        if (len > cap - cnt) return false;
        size_t first = min(len, cap - tail);
        memcpy(buf.data() + tail, p, first);
        memcpy(buf.data(), p + first, len - first);
        tail = (tail + len) % cap;
        cnt += len;
        return true;
    }

    bool read(char* out, size_t len) {
        if (len > cnt) return false;
        size_t first = min(len, cap - head);
        memcpy(out, buf.data() + head, first);
        memcpy(out + first, buf.data(), len - first);
        head = (head + len) % cap;
        cnt -= len;
        return true;
    }
};
```

---

### 2. *Singleton

> 线程安全单例，用于全局配置、共享资源管理。

函数内局部静态变量初始化线程安全，编译器会插入类似 `__cxa_guard_acquire` 的同步代码，保证只有一个线程执行初始化。

**Q**: 不用局部静态变量，怎么写？

**A**: 双检锁 + `std::atomic` + 内存序 `acquire/release`

```cpp
class Single {
public:
    static Single& get() {
        static Single s;
        return s;
    }
private:
    Single() = default;
    Single(const Single&) = delete;
    Single& operator=(const Single&) = delete;
};
```

---

### 3. *BitSet

> 手写 bitset，与Tagged-Pointer异曲同工

```cpp
class Bits {
    vector<uint64_t> w;
    size_t n;
public:
    Bits(size_t num) : w((num + 63) / 64), n(num) {}
    
    void set(size_t i)   { w[i >> 6] |=  (1ULL << (i & 63)); }
    void reset(size_t i) { w[i >> 6] &= ~(1ULL << (i & 63)); }
    bool test(size_t i)  { return w[i >> 6] & (1ULL << (i & 63)); }
    size_t size()        { return n; }
};
```

---

### 4. SharedPtr

**Q1**:  控制块里有什么？

**A1**: 强引用计数、弱引用计数、删除器、分配器。

**Q2**: `make_shared` 为什么高效？

**A2**: 一次内存分配同时容纳对象和控制块，缓存友好。

```cpp
template <typename T>
class SP {
    T* p;
    size_t* rc;
public:
    SP(T* ptr = nullptr) : p(ptr), rc(new size_t(1)) {}
    
    SP(const SP& other) : p(other.p), rc(other.rc) { ++(*rc); }
    
    ~SP() {
        if (--(*rc) == 0) { delete p; delete rc; }
    }
    
    T* get() const { return p; }
    T& operator*() const { return *p; }
};
```

---

### 5. MemoryPool

> 内存池，预分配一大块连续内存，用空闲链表管理未使用的块，分配/释放对链表存取，优点O(1) 无碎片

**Q**: 为什么仿真系统需要内存池？

**A**: 因为飞行器控制周期可能是 1ms，如果每次 `new/delete` 都去问 OS 要内存，可能触发系统调用和锁竞争，导致周期抖动。内存池把分配变成常数时间。

```cpp
class Pool {
    vector<char> mem;
    size_t block, cap;
    vector<void*> free_list;
public:
    Pool(size_t block_size, size_t count)
        : mem(block_size * count), block(block_size), cap(count) {
        for (size_t i = 0; i < count; i++)
            free_list.push_back(mem.data() + i * block);
    }
    
    void* alloc() {
        if (free_list.empty()) return nullptr;
        void* p = free_list.back();
        free_list.pop_back();
        return p;
    }

    void free(void* p) {
        if (!is_valid(p)) return;      // 不在池内或不对齐
        if (is_in_free(p)) return;     // 重复释放
        free_list.push_back(p);
    }
};
```



## 附录

| 分类               | 问题                                                         |
| ------------------ | ------------------------------------------------------------ |
| 内存和对象生命周期 | double free、delete 虚析构、聚合类值初始化清值、多继承上行转换 |
| 编译器优化与 UB    | 4路循环展开、符号扩展、编译器优化导致 clone、SSE 位宽、this 没用/大对象返回寄存器错位 |
| 类型系统陷阱       | CRTP this 多态、标签指针类型错误、lambda 类型擦除、is_same 未闭合分支、指针/引用重载 |
| 编码逻辑边界       | auto 拷贝乱用、内联比较逻辑错误                              |

CheckSum 问题总结

1. double free
2. 4路循环展开
3. delete 触发的虚析构
4. 多继承中的上行转换
5. auto 拷贝乱用
6. 内联比较逻辑错误
7. 符号扩展
8. CRTP的this多态
9. 标签指针类型错误
10. SSE指令集及位宽
11. lambda的类型擦除
12. 聚合类值初始化的清值
13. is_same未闭合分支
14. this没用或大对象返回导致的寄存器错位
15. 指针/引用的重载调用
16. 编译器优化导致的clone函数


排查路径：
  1. 控制变量，case failed by f() directly
  2. CheckSum is OK?
  3. OK: 检查log打印完整性，gdb调试
  4. nOK: 静态粗查 => 逻辑错误，宽度，TODO，double free, etc.
  5. nOK: 动态调试 => Crash?
  6. crash: 空野指针访问，gdb break and watch，arr out of bound, regester mismatch
  7. ncrash: 静态细查 & 动态调试 & log输出
