参考资料，书籍《C++ Concurrency in Action - SECOND EDITION》章节：The C++ memory modeland operations on atomic types

## 关于原子变量
- 一般使用，定义一个原子变量，多个线程读、写这个变量不会有问题；即多线程中使用原子变量，数据本身是安全的。
- 多核系统上，一个线程上的对变量的读、写顺序与另一个线程观察的相同的变量的读、写顺序是不一致的，需要一套规则来保证多线程间看到的读、写顺序一致，这个就是Memory order的作用。
- 比如，一个线程先执行a.store(1)，再执行b.store(2)；当另外一个线程，获取的b.load()的值为2时，获取到的a.load()一定百分百为2，就可以通过设置Memory order来实现。


## 内存序（Memory order）种类
- 六种内存序
  - 1、memory_order_relaxed
    仅保证此操作的原子性，和其他读、写操作没有确定的顺序；
  - 2、memory_order_consume
    load 操作：①当前线程中，依赖这个变量load的读、写操作，不可能在load之前发生；②在release线程中，写入操作修改依赖的原子变量，当前线程可以获知；
  - 3、memory_order_acquire
    load 操作：①当前线程中，在这个变量load之后的读、写操作，不可能在load之前发生；②在release线程中，写入操作修改相同的原子变量，当前线程可以获知；
  - 4、memory_order_release
    store 操作：①当前线程中，在这个变量store之前的读、写操作，不可能在store之后发生；②当前线程会通知给，acquire相同变量的线程，以及consume依赖变量的线程；
  - 5、memory_order_acq_rel
    load/store 操作：①当前线程中，在这个变量load/store之前的读、写操作，不可能在load/store之后发生；同样的，在这个变量load/store之后的读、写操作，也不可能在load/store之前发生；
    ②在release线程的store操作会通知给当前线程，当前线程的load/store，会通知给另一个acquire/consume线程；
  - 6、memory_order_seq_cst（默认值）
    无论load操作，还是store操作，在或者load/store操作，都遵循一个总的内存顺序，即同一线程间它们之间顺序不会被打乱，严格按照code编写的顺序执行；

- 三种内存模型
  - 顺序一致性（默认），严格有序：memory_order_seq_cst
  - 获取-释放序，部分有序，介于有序和无序之间：memory_order_consume、memory_order_acquire、memory_order_release和memory_order_acq_rel
  - 自由序，完全无序：memory_order_relaxed


## 顺序一致性
- 强制要求有序，所有的多线程操作有一个全局的排序，类似在单线程中一样；
- 不存在一个线程看的是这样的顺序，另外一个线程看到的是那样的顺序的情况；非顺序一致性内存就会出现这样的情况，相同的操作，不同的线程可能看到的顺序不一致；
- 因为使用简单暴力，某些处理器架构上可能有性能影响；
- 例子代码分析：
    - （1）肯定不会触发语句5，即z值不可能为0；
	- （2）因为是全局有序的内存，所以语句1和语句2必定有先后顺序，要么1 → 2的顺序，要么2 → 1的顺序；
	- （3）根据逻辑条件，语句1一定在语句3的前面，语句2一定在语句4的前面；
	- （4）存在的执行结果有：①当为 1 → 3 → 2 → 4 和 2 → 4 → 1 → 3 时，z值为1；②当为 1 → 2 → 3 → 4 和 1 → 2 → 4 → 3 和 2 → 1 → 3 → 4 和 2 → 1 → 4 → 3 时，z值为2；
  ```
  #include <atomic>
  #include <thread>
  #include <assert.h>
  std::atomic<bool> x,y;
  std::atomic<int> z;
  void write_x()
  {
   x.store(true,std::memory_order_seq_cst); // 1
  }
  void write_y()
  {
   y.store(true,std::memory_order_seq_cst); // 2
  }
  void read_x_then_y()
  {
   while(!x.load(std::memory_order_seq_cst));
   if(y.load(std::memory_order_seq_cst)) // 3
   ++z;
  }
  void read_y_then_x()
  {
   while(!y.load(std::memory_order_seq_cst));
   if(x.load(std::memory_order_seq_cst)) // 4
   ++z;
  }
  int main()
  {
   x=false;
   y=false;
   z=0;
   std::thread a(write_x);
   std::thread b(write_y);
   std::thread c(read_x_then_y);
   std::thread d(read_y_then_x);
   a.join();
   b.join();
   c.join();
   d.join();
   assert(z.load()!=0); // 5
  }
  ```


## 自由序
- 同一个线程中，同一个变量的操作，有明确的先后顺序；但是，在不同线程中没有确定的顺序；
  比如：`①x.store(true,std::memory_order_relaxed); ②y.load(std::memory_order_relaxed)`，如果①②在同一线程，y值一定是ture；如果①②在不同线程的两个线程中，y值不一定是ture；
- 自由序的理解
  - 1、假如有一个人叫张三，他有一份工作，守在电话旁，接到电话后，为其他人记录数字；没有人打电话给他的时候，他记录的值为0；
  - 2、某一天，有个叫做A的人，打电话给张三让他记录10；接着有个叫做B的人，打电话给张三让他记录20；在接着有个叫做C的人，打电话给张三让他记录40；
  - 3、张三先后接到3个电话，需要记录3的人的不同数字；他是这样做的，他整理的一张表，按照3人要求的顺序记录保存了一组数据为：0,10,20,40；
  - 4、接着，有个叫D的人，打电话问张三他现在记录的值是多少？
  - 5、由于张三记录的是一组数据，他需要从这组数据（0, 10, 20, 40）选择一个值告诉D；具体选择哪个值完全看张三自己的心情，可以是0，也可以是20，当然也可以是40；
  - 6、如果张三告诉D的数字为0后；D觉得张三告诉自己的值不是最新的，所以再次询问张三此时值是多少？
  - 7、此时，张三已经告诉过D一次数字为0，他可以选择再次告诉D数字为0，也可以选择其他任意值；张三决定第二次告诉D的数字为20；
  - 8、D获取20数字后，还是觉得张三告诉他的数字不是最新的，于是就第三次问张三他现在保存的数值是多少？
  - 9、张三有自己的规则，每次选择数字只会在上次告诉对方的数字和最新数字之间选择；他上次告诉D的数字为20，所以他不会选择0和10告诉张三，他可以选择20或者40告诉张三；
  - 10、这次张三选择20告诉D，D觉得张三告诉自己的数字是最新的了（实际上不是最新的），也就不再找张三询问了；
  - 11、随后，又有一个人叫E，他找张三询问最新的数值；张三知道E是第一次询问自己数值，所以会在再（0, 10, 20, 40）之间任意选择一个数值通知给E；同样的，以后E再多次找张三询问数字，张三也只会在上次通知给E的数字和最新数字之前选择一个数字通知给E；
  - 12、E查询以后，又告诉张三写入2个数字123和888，张三保存的数字表变为（0, 10, 20, 40，123, 888）；
  - 13、接着，A又多次询问张三保存的数值是多少？张三可以选择的数值顺序为（0, 10, 20, 40，123, 888）；
  - 14、A一共询问了十次，张三给A的十次回复数字为：0,0,0,0,0,0,0,0,0,0；即每一次都是0，张三可以这样做；
  - 15、B也询问了张三十次，张三给B的十次回复数字为：0,0,0,20,123,123,123,888,888,888；每一次的回复数字，都必须是在上一次数字和最新数字之间选择；
**_类比到程序中，张三其实是一个原子变量，A、B、C、D、E是5个不同的线程，即多线程对一个原子变量（设置了std::memory_order_relaxed）store、load的规则。_**
- 例子一代码分析：
  - （1）语句5有可能触发，即z值有可能为0；
  - （2）虽然语句1和语句2在同一个线程，但是语句1和语句2是操作两个不同的变量，没有依赖关系，所以没有确定的先后顺序；即执行顺序可能是 ① → ②，也可能是 ② → ①；
  - （3）语句1和语句4也没有确定的执行顺序；
  - （4）当执行顺序为 ② → ③ → ④ → ① 时，z的值为0；即语句5触发；
  ```
  #include <atomic>
  #include <thread>
  #include <assert.h>
  std::atomic<bool> x,y;
  std::atomic<int> z;
  void write_x_then_y()
  {
   x.store(true,std::memory_order_relaxed); // 1
   y.store(true,std::memory_order_relaxed); // 2
  }
  void read_y_then_x()
  {
   while(!y.load(std::memory_order_relaxed)); // 3
   if(x.load(std::memory_order_relaxed)) // 4
   ++z;
  }
  int main()
  {
   x=false;
   y=false;
   z=0;
   std::thread a(write_x_then_y);
   std::thread b(read_y_then_x);
   a.join();
   b.join();
   assert(z.load()!=0); // 5
  }
  ```
- 例子二代码分析：
  - （1）一共5个线程，线程1、2、3分别在自增x、y、z的值；
  - （2）线程1在线性自增x的值：0，1，2，3……9；但是线程2获取的x值不是线性的：0，0，0，1，8，8，8，8，10，10；且只会获离新值最近的这个方向的值；
  - （3）虽然线程1在自增x，现成2在自增y，但是线程3获取的x、y值可能一直都是0；但是线程4和线程5又可能获取到x、y的值，但是结果也不一样；
  - （4）在一个线程修改一个变量的值，不同线程获取到的这个变量值可能千差万别，可能是初始值，也可能是过程修改值，也可能是最终修改值；
  - 程序一种可能的输出为：
  ```
  (0,0,0),(1,0,0),(2,0,0),(3,0,0),(4,0,0),(5,7,0),(6,7,8),(7,9,8),(8,9,8),(9,9,10)
  (0,0,0),(0,1,0),(0,2,0),(1,3,5),(8,4,5),(8,5,5),(8,6,6),(8,7,9),(10,8,9),(10,9,10)
  (0,0,0),(0,0,1),(0,0,2),(0,0,3),(0,0,4),(0,0,5),(0,0,6),(0,0,7),(0,0,8),(0,0,9)
  (1,3,0),(2,3,0),(2,4,1),(3,6,4),(3,9,5),(5,10,6),(5,10,8),(5,10,10),(9,10,10),(10,10,10)
  (0,0,0),(0,0,0),(0,0,0),(6,3,7),(6,5,7),(7,7,7),(7,8,7),(8,8,7),(8,8,9),(8,8,9)
  ```
  - 代码：
  ```
  #include <thread>
  #include <atomic>
  #include <iostream>
  std::atomic<int> x(0),y(0),z(0); // 1
  std::atomic<bool> go(false); // 2
  unsigned const loop_count=10;
  struct read_values
  {
   int x,y,z;
  };
  read_values values1[loop_count];
  read_values values2[loop_count];
  read_values values3[loop_count];
  read_values values4[loop_count];
  read_values values5[loop_count];
  void increment(std::atomic<int>* var_to_inc,read_values* values)
  {
   while(!go)
   std::this_thread::yield(); // 3 自旋，等待信号
   for(unsigned i=0;i<loop_count;++i)
   {
   values[i].x=x.load(std::memory_order_relaxed);
   values[i].y=y.load(std::memory_order_relaxed);
   values[i].z=z.load(std::memory_order_relaxed);
   var_to_inc->store(i+1,std::memory_order_relaxed); // 4
   std::this_thread::yield();
   }
  }
  void read_vals(read_values* values)
  {
   while(!go)
   std::this_thread::yield(); // 5 自旋，等待信号
   for(unsigned i=0;i<loop_count;++i)
   {
   values[i].x=x.load(std::memory_order_relaxed);
   values[i].y=y.load(std::memory_order_relaxed);
   values[i].z=z.load(std::memory_order_relaxed);
   std::this_thread::yield();
   }
  }
  void print(read_values* v)
  {
   for(unsigned i=0;i<loop_count;++i)
   {
   if(i)
   std::cout<<",";
   std::cout<<"("<<v[i].x<<","<<v[i].y<<","<<v[i].z<<")";
   }
   std::cout<<std::endl;
  }
  int main()
  {
   std::thread t1(increment,&x,values1);
   std::thread t2(increment,&y,values2);
   std::thread t3(increment,&z,values3);
   std::thread t4(read_vals,values4);
   std::thread t5(read_vals,values5);
   go=true; // 6 开始执行主循环的信号
   t5.join();
   t4.join();
   t3.join();
   t2.join();
   t1.join();
   print(values1); // 7 打印最终结果
   print(values2);
   print(values3);
   print(values4);
   print(values5);
  }
  ```