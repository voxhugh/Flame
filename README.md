## ③ Qt 

## 1. 信号与槽

**一句话本质：** Qt 的观察者模式实现，通过元对象系统在运行时动态绑定信号和槽。

**必须能写出来的关键词：**

- `Q_OBJECT` 宏是开启信号槽的前提，moc 预处理器会解析这个宏并生成元对象代码
- `connect(sender, SIGNAL(xxx()), receiver, SLOT(yyy()))` 建立连接
- 新式写法：`connect(sender, &Sender::signal, receiver, &Receiver::slot)`，编译期类型检查
- 一个信号可以连多个槽，一个槽可以被多个信号连接
- 连接类型：
  - `Qt::DirectConnection`：直接调用，同线程默认
  - `Qt::QueuedConnection`：投递到接收者的事件队列，跨线程默认
  - `Qt::AutoConnection`：自动判断，同线程直连，跨线程队列
- **底层原理：** 元对象系统为每个信号和槽分配索引，connect 时建立映射表，emit 信号时查表调用对应槽函数。跨线程队列连接本质是向目标线程的事件队列投递一个 `QMetaCallEvent`。

**简答题模板：**

> 信号槽是 Qt 的核心通信机制，本质是观察者模式。通过 moc 预处理生成元对象代码，为信号和槽建立索引映射。同线程发射信号直接调用槽函数，跨线程则通过事件队列投递，保证线程安全。相比回调函数，信号槽更灵活，支持一对多、多对一连接，且是类型安全的。

## 2. 事件循环

**一句话本质：** Qt 程序的主线程不断从事件队列取事件并分发给对应对象处理。

**关键点：**

- `QApplication::exec()` 启动主事件循环，阻塞直到 `quit()` 被调用
- 事件来源：用户输入（鼠标、键盘）、定时器、跨线程信号槽投递、网络事件、自定义事件
- `QEventLoop` 可以创建局部事件循环，实现嵌套
- 事件循环是 Qt 非阻塞异步模型的基础

**底层类比：** 类似 Windows 消息循环 `GetMessage/DispatchMessage`，也类似 Linux 的 epoll 循环。

## 3. 对象树与内存管理

**一句话本质：** QObject 通过父子关系自动管理子对象生命周期。

**关键点：**

- 创建 QObject 时指定 parent，该对象会被加入 parent 的 children 列表
- parent 析构时，自动 delete 所有 children
- 注意事项：手动 new 的对象如果没指定 parent，需要手动 delete 或用智能指针包裹
- 析构顺序：先子后父，确保子对象析构时父对象仍有效

**底层类比：** 类似 `shared_ptr` 的 ownership 模型，但方向相反——parent 拥有 child。

## 4. Model/View 架构

**一句话本质：** 数据（Model）与显示（View）分离，通过 Delegate 定制渲染和编辑。

**关键点：**

- `QAbstractItemModel`：数据接口，提供 `data()`、`rowCount()`、`columnCount()` 等
- `QAbstractItemView`：显示组件，如 `QTableView`、`QListView`、`QTreeView`
- `QAbstractItemDelegate`：负责单元格绘制和编辑控件
- 优势：一份数据可以被多个 View 以不同方式展示，数据变化自动通知所有 View 更新

## 5. QThread 与 Qt 并发

**两种用法：**

1. 继承 QThread，重写 `run()` 方法
2. 创建 QThread，将工作对象用 `moveToThread()` 移到该线程

**QtConcurrent：**

- `QtConcurrent::run()`：在线程池中异步执行函数
- 返回 `QFuture<T>`，通过 `waitForFinished()` 或 `QFutureWatcher` 获取结果

**线程安全规则：**

- UI 操作只能在主线程执行
- 跨线程信号槽默认使用队列连接，确保槽在目标线程执行
- 共享数据用 `QMutex`、`QReadWriteLock` 保护

**简答题模板：**

> Qt 多线程编程核心原则是 UI 只能在主线程操作。子线程执行耗时任务，通过信号槽把结果传回主线程更新界面。跨线程信号槽默认队列连接，事件循环负责投递，天然线程安全。QThread 可以继承重写 run 或使用 moveToThread，QtConcurrent 提供更高层的线程池接口。

## 6. 常用 Widget（选择/填空题可能考）

| Widget         | 用途                                                    |
| -------------- | ------------------------------------------------------- |
| `QWidget`      | 所有可视化控件的基类                                    |
| `QMainWindow`  | 主窗口，含菜单栏、工具栏、状态栏、中心区域              |
| `QDialog`      | 对话框基类，模态/非模态                                 |
| `QLabel`       | 显示文本或图片                                          |
| `QPushButton`  | 按钮                                                    |
| `QLineEdit`    | 单行文本输入框                                          |
| `QTextEdit`    | 多行富文本编辑器                                        |
| `QComboBox`    | 下拉选择框                                              |
| `QCheckBox`    | 复选框                                                  |
| `QRadioButton` | 单选按钮                                                |
| `QTableView`   | 表格视图（配合 Model 使用）                             |
| `QLayout`      | 布局管理器：`QVBoxLayout`、`QHBoxLayout`、`QGridLayout` |

## 7. 元对象系统（moc）

**必须知道：**

- 只有继承 `QObject` 且包含 `Q_OBJECT` 宏的类才支持信号槽
- moc（Meta-Object Compiler）是一个预处理工具，扫描头文件中的 `Q_OBJECT`，生成包含信号槽注册、类型信息、属性系统的 C++ 代码
- 生成的元对象代码包含：类名、父类、信号列表、槽列表、属性列表
- `qmake` 或 `CMake` 会自动调用 moc

---

# ② 并发核心

## 1. 原子操作与内存序（你熟，扫一眼）

- `std::atomic<T>` 保证读写原子性，避免数据竞争
- 内存序级别：`seq_cst`（默认，全局顺序）、`acquire`（获取，后续操作不重排到前面）、`release`（释放，前面操作不重排到后面）、`relaxed`（只保证原子性）
- 典型场景：无锁计数器、自旋锁、无锁队列

## 2. 互斥锁与条件变量

- `std::mutex` + `std::unique_lock` + `std::condition_variable`
- 生产者消费者模式：生产者拿到锁后 push，`notify_one()`；消费者 `wait()` 等待，被唤醒后检查条件非空再消费
- **必须会用 `wait` 的谓词重载：** `cv.wait(lock, []{ return !queue.empty(); })`
- 死锁四条件：互斥、持有并等待、不可剥夺、循环等待

## 3. future / promise / async

- `std::async(std::launch::async, fn, args...)` 返回 `std::future<T>`
- `future.get()` 阻塞等待结果，只能调用一次
- `std::promise<T>` 在线程内设置值，通过 `get_future()` 获取对应 future
- 典型场景：异步任务执行后主线程获取返回值，避免手动管理线程生命周期

## 4. 线程安全编程原则（笔试题可能出简答）

- 不可变对象天然线程安全
- 尽量使用局部变量，减少共享
- 共享数据必须加锁或使用原子类型
- 锁的粒度越小越好
- 避免在持有锁时调用外部函数，防止死锁
- 优先使用 `std::atomic` 而非 `volatile` 做线程间同步

---

# ① C++ 核心

JD 明确要求：**C++17 标准、STL 底层、面向对象、设计模式、数据结构与算法**。你的笔记应该涵盖大部分，下面按笔试出现频率筛选：

## 1. C++17 高频特性（可能出现在选择题/简答）

| 特性               | 用途                   | 示例                             |
| ------------------ | ---------------------- | -------------------------------- |
| 结构化绑定         | 拆解 pair/tuple/struct | `auto [a, b] = make_pair(1, 2);` |
| `if constexpr`     | 编译期分支             | `if constexpr (sizeof(T) > 4)`   |
| `std::optional`    | 可能无值的返回         | `std::optional<int> find();`     |
| `std::variant`     | 类型安全联合体         | `std::variant<int, double> v;`   |
| `std::string_view` | 不拷贝的字符串视图     | 函数参数优化                     |
| `std::filesystem`  | 文件系统操作           | 路径、遍历目录                   |

## 2. 面向对象与多态（笔试必考）

- **虚函数表机制：** 含虚函数的类有一个 vptr 指向 vtable，vtable 中存放各虚函数地址。子类重写虚函数会替换 vtable 中对应项。通过基类指针调用虚函数时，运行时查表实现动态分派
- **虚析构函数：** 基类析构必须是 virtual，否则通过基类指针 delete 子类对象时只调用基类析构，导致子类资源泄漏
- **纯虚函数与抽象类：** `virtual void f() = 0;` 使类成为抽象类，不能实例化
- **访问控制：** public/protected/private 继承的区别与影响

## 3. STL 容器底层（你最强项，快速回忆）

| 容器            | 底层结构     | 关键特性                 |
| --------------- | ------------ | ------------------------ |
| `vector`        | 连续数组     | 扩容倍增，迭代器可能失效 |
| `deque`         | 分段连续数组 | 两端插入高效             |
| `list`          | 双向链表     | 任意位置插入 O(1)        |
| `map/set`       | 红黑树       | 有序，O(log n)           |
| `unordered_map` | 哈希表       | 平均 O(1)，无序          |
| `string`        | 连续字符数组 | SSO 小字符串优化         |

**迭代器失效问题：**

- `vector`：`push_back` 可能触发扩容，导致所有迭代器失效；`erase` 使被删元素及其后所有迭代器失效
- `map/set/unordered_map`：`erase` 只使被删元素迭代器失效
- `list`：插入不失效，删除只使被删元素迭代器失效

## 4. 智能指针（必考）

- `unique_ptr`：独占所有权，不可拷贝，可移动。替代裸指针，零开销
- `shared_ptr`：引用计数，拷贝时计数 +1，析构时 -1，归零释放。注意循环引用
- `weak_ptr`：不增加计数，配合 `shared_ptr` 打破循环引用，使用前需 `lock()` 提升
- `make_shared` 优于 `new shared_ptr`：一次内存分配，缓存友好

## 5. 右值引用与移动语义

- 右值引用 `T&&` 绑定临时对象
- 移动构造函数将资源"窃取"过来，原对象置空
- `std::move` 本质是 `static_cast<T&&>`，不实际移动数据
- 完美转发 `std::forward<T>` 保留实参的左值/右值属性

## 6. 设计模式（JD 要求，可能简答）

准备两个能流畅说出来的：

**单例模式：** 懒汉式（线程安全用局部静态变量），饿汉式。C++11 的局部 static 初始化天然线程安全

**观察者模式：** 发布-订阅，Qt 信号槽就是典型实现。抽象出 Subject 和 Observer 接口，Subject 维护 Observer 列表，状态变化时通知所有观察者

工厂模式，建造者模式，原型模式