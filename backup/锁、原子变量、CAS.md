## 互斥锁
- 底层：操作系统内核实现。
- 锁被占用，当前线程进入睡眠状态，不会占用CPU资源。
- 系统维护等待队列，锁被释放时，唤醒等待队列里的线程。
- 适用于长时间的锁定操作（一般应用使用），涉及到线程调度的上下文切换。

## 自旋锁（基于CAS实现）
- 底层：实现依赖硬件提供的原子操作。
- 锁被占用，线程进入忙等待（死循环），会一直占用CPU资源。
- 因为可以避免上下文切换的开销，通常消息比较高。
- 适用于短时间的锁定操作（一般内核中使用），比如内核中断的处理程序中。
- **如果被保护的临界区处于中断里，那么只能使用自旋锁**。因为互斥锁可能会导致阻塞，而中断是不能被阻塞的。

## 原子变量（基于CAS实现）
- 底层：实现依赖硬件提供的原子操作。
- x86架构的原子操作：在x86架构上，许多原子操作由特殊的汇编指令支持，例如LOCK前缀指令。这些指令可以确保在多处理器环境下操作的原子性。
  ```
  static inline int atomic_compare_and_swap(int *ptr, int oldval, int newval) {
    int result;
    __asm__ __volatile__ (
        "lock; cmpxchgl %2, %1"
        : "=a" (result), "+m" (*ptr)
        : "r" (newval), "0" (oldval)
        : "memory");
    return result;
  }
  ```
- GCC内建函数：GCC提供了一些内建函数，可以直接使用这些原子操作，而无需手动编写汇编代码。
  ```
  int atomic_compare_and_swap(int *ptr, int oldval, int newval) {
    return __sync_val_compare_and_swap(ptr, oldval, newval);
  }
  ```
- 常见的硬件原子操作包括Test-and-Set、Compare-and-Swap和Fetch-and-Add等，这些操作由现代处理器直接支持，通过专门的汇编指令实现。
- 原子变量和操作通常被封装在库中，例如C++中的std::atomic，以提供更高层次的抽象和易用性。


## CAS
- 即Compare And Set（或Compare And Swap），实现无锁操作的一种技术手段。
- 是实现原子变量和自旋锁的基础；利用 CPU 指令保证了操作的原子性，以达到锁的效果。
- CAS操作包含三个操作数：内存位置（V）、预期原值（A）、新值(B)。如果V处的值，与A相等，则更新V处的值为B；否则，不做任何操作。