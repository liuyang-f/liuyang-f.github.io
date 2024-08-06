## 什么是内存屏障
- 简单来说，内存屏障，就是在语句A和语句B之间设置一个屏障（类似一堵墙），阻止语句B超过语句A去执行，确保始终以A、B的顺序执行。
  - 现在多核计算机，在执行我们编写程序，并不一定是按语句顺序执行的；
  - 比如我们写的代码是，先执行语句A，在执行语句B，最终编译执行后，可能变成先执行语句B，在执行语句A；
  - 这种现象与编译器优化和多核并行执行程序有关（本意是提升运行速度，结果导致了另外一个问题）；
  - 需要一种技术手段，确保语句A一定在语句B之前执行完成，换句话说语句B不能跑到语句A的前面去执行；
  

## 编译器优化导致的乱序
- 编译器优化，往往会重排指令（compiler reordering），结果指令B（后一条指令）先于指令A（前一条指令）执行。
  （1）使用编译器提供的编译屏障（compiler barriers），禁止编译器重排指令；
	```
    #define barrier() __asm__ __volatile__("": : :"memory")
	int a, b;

	void foo(void) {
		a = b + 1;
		barrier();
		b = 0;
	}
	```
    barrier()的作用：①此语句后的数据访问，需要重新从内存获取，不能从寄存器获取值；②此语句前后的语句顺序不能被打乱，严格按照语句顺序执行。
  （2）使用 volatile 关键字，确保每一次的数据访问，都从内存中获取，不使用寄存器中的缓存值。


## CPU并行处理导致的乱序
- 由于并行处理的机制，多个语句同时执行，也可能出现指令B（后一条指令）先于指令A（前一条指令）执行的情况发生。linux提供的宏封装：
  （1）通用barrier，smp_mb()，此语句后面的读写操作不会乱序到宏前面的指令前执行。
  （2）读barrier，smp_rmb()，此语句后面的读操作不会乱序到宏前面的读操作，只针对读操作，不影响写操作。
  （3）写barrier，smp_wmb()，此语句后面的写操作不会乱序到宏前面的写操作。
    ```
	// global variables
	int flag, data;

	// thread1
	void write_data_cpu0(int value) {
	   data = value;
	   // smp_mb();
	   smp_wmb();
	   flag = 1;
	}

	// thread2
	void read_data_cpu1(void) {
	   while (flag == 0);
	   // smp_mb();
	   smp_rmb();
	   return data;
	}
	```

## Memory barrier 常用场景
- 1、实现同步原语
- 2、实现无锁结构
- 3、驱动程序


## Linux 内核中实现的无锁队列
环形缓冲区 kfifo（只有一个线程读，并且只有另一个线程写） 使用了 Memory barrier
  - kfifo_init()
  - kfifo_alloc()
  - kfifo_free()
  - __kfifo_put()
  - __kfifo_get()
