### 1、沙箱机制
- 通过虚拟化技术创建一个‌**隔离的运行环境**‌，使程序或进程在其中受限执行，无法直接访问真实系统资源（如文件、注册表、网络）。
- 类似“安全堡垒”，风险操作被限制在沙箱内部，即使程序被恶意攻击或崩溃，也不会危害主机系统
### 2、Unsafe类

“Unsafe” 类的命名与其在 ‌**线程安全**‌ 中的作用看似矛盾，实则深刻反映了其定位与风险。核心矛盾在于：
##### 2.1 “安全”的所指不同
- 线程安全（Concurrency Safety）
    指多线程环境下数据操作的原子性、可见性和有序性。`Unsafe` 提供的 ‌**CAS 操作**‌ 正是实现锁无关（Lock-Free）线程安全的关键底层基石（如 `AtomicInteger` 的原子递增）。
 -  ‌内存与系统安全（Memory/System Safety）
    Java 设计初衷是建立受控的“沙箱环境”（Sandbox），禁止直接操作内存或硬件（如指针）。而 `Unsafe` 通过 ‌**绕过 JVM 安全检查机制**‌，允许以下危险操作：
    - 直接读写任意内存地址（可能越界导致 JVM 崩溃）
    - 手动分配/释放堆外内存（易引发内存泄漏）
    - 修改 `final` 字段（破坏不可变性）

 ‌**结论**‌：
 - `Unsafe` 的“不安全”是指 ‌**破坏 Java 内存安全模型**‌，而非指其线程同步能力弱。
 - `JUC` 原子类：以 `Unsafe` 的 CAS 操作为基石（如 `AtomicLong` 的 `compareAndSet`）
##### 2.2 unsafe 类概述
	 1. 定义与作用
		- sun.misc.Unsafe是JDK提供的底层工具类，通过JNI直接调用操作系统功能，提供硬件级原子操作和内存管理能力（如内存分配、CAS、线程挂起/恢复等）。
		- 主要用于Java核心类库（如JUC并发包）的实现，普通代码不推荐使用。
	 2. 安全性风险
		- 绕过Java语言安全检查，直接操作内存和线程，可能导致内存泄漏、指针越界等问题 （有点像C、C++中的指针操作）

##### 2.3 unsafe的核心功能
	1. 内存操作
		- 分配/释放内存：`allocateMemory`、`reallocateMemory`、`freeMemory`（类似C的`malloc`/`free`）
		- 内存读写：`putXXX`/`getXXX`系列方法直接修改对象字段值（需配合字段偏移量`objectFieldOffset`）
	2. cas操作
		-提供`compareAndSwapObject`、`compareAndSwapInt`等原子方法，是JUC并发工具（如`AtomicInteger`）的底层实现
	3. 线程调度
		-`park`/`unpark`：直接挂起或唤醒线程（`LockSupport`的底层依赖）
	4. 类与对象操作
		-绕过构造器创建实例：`allocateInstance`
		-动态定义类：`defineAnonymousClass`（已废弃）
	5. 内存屏障
		-强制读写主存：`loadFence`、`storeFence`（避免指令重排序）

##### 2.4 获取unsafe实例
	1.限制条件：默认仅允许BootstrapClassLoader加载的类调用`Unsafe.getUnsafe()`，否则抛出`SecurityException`
	
	