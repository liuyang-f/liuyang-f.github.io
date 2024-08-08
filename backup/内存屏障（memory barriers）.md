# linux源码在线网站：https://elixir.bootlin.com/linux/latest/source

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

## 内存重新排序（Memory Reordering）
- 假如我们希望的语句顺序是，先执行语句A，在执行语句B；
- 编译器的问题
  - 由于编译器优化，导致生成的汇编代码，变成了先执行B，在执行A；
  - 使用编译器屏障，可以避免生成的汇编代码乱序的问题；
- 多处理器的问题
  - 虽然解决了编译器带来的乱序问题，但实际运行时，还是会出现先执行B，在执行A的情况（可能和我们预测的不一样）；
  - 在多个处理器的场景，每个处理器会在保证结果一致性的前提下，在运行期间对语句的处理顺序进行优化；即只要不影响最终的运行结果，处理器可能先执行B，在执行A；
  - 特别是语句A和语句B没有关联的情况下，先执行A或者先执行B都不会影响最终的结果；
- 解决方案
  - 同时编译器屏障和内存屏障，asm volatile 用于编译器屏障，mfence 用于内存屏障；
  ```
  asm volatile("" ::: "memory");        // Prevent compiler reordering
  asm volatile("mfence" ::: "memory");  // Prevent memory reordering
  ```

## Memory barrier 常用场景
- 1、实现同步原语
- 2、实现无锁结构
- 3、驱动程序


## Linux 内核中实现的无锁队列
环形缓冲区 kfifo（只有一个线程读，并且只有另一个线程写） 使用了 Memory barrier
```
__kfifo_alloc()
__kfifo_free()
__kfifo_init
__kfifo_in //写入数据
__kfifo_out // 读取数据
```
无锁化环形队里的简单理解：
1、有一个结构体，data用于存储数据，out和in作为读写的位置标识；
2、一个线程仅从结构中读取数据（out增加），另一个线程仅向结构中写入数据（in增加）；
3、当写入数据时，先把数据copy到data中，中间使用 smp_wmb，然后再增加in；这样另一个线程，检测到in改变时，更新data中的数据一定是完成的；
4、当读取数据时，先从data中copy数据，中间使用 smp_wmb，然后再增加out；这样另一个线程，检测到out改变时，获取data中的数据一定是完成的；
5、无论读写，都是先执行动作，然后改变标志变量，借用smp_wmb确保改变标志变量一定在执行动作的后面，另一个线程可以通过标志判断动作是否完成；

头文件：https://elixir.bootlin.com/linux/v6.10.3/source/include/linux/kfifo.h
源文件：https://elixir.bootlin.com/linux/v6.10.3/source/lib/kfifo.c

![image](https://github.com/user-attachments/assets/5528df09-56ec-4f87-a3cd-091ce0dcadac)
![image](https://github.com/user-attachments/assets/59124275-c8eb-4402-91f3-b51c77043d89)
![image](https://github.com/user-attachments/assets/f441d8fb-c970-416c-a358-1821a2e5c946)
![image](https://github.com/user-attachments/assets/0f825f10-2eff-46d3-88c0-89431e9a6027)
