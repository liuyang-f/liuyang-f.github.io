## 认识std::future
它是一种类型，让两个线程间以某种方式同步信息，比如一个线程设置了变量值或者执行了函数，另一个线程可以借助future获取对应变量值或者函数执行的结果。
- 往往使用async、promise::get_future、promise::get_future产生对象，默认构造的对象是invalid。
- 使用get能获取另一个线程传递的值或者结果，并且只能调用一次。
- wait函数会阻塞本线程，等待另一个线程传递变量值或者结果。
- wait_for和wait_until的返回值为future_status::deferred时，说明使用了std::async并且设置参数为std::launch::deferred。
- 由于future调用get后就变为无效，所以可以使用share方法把std::future转为std::shared_future，std::shared_future可以多次使用get。



## async
- std::future最简单的使用，和一个异步函数结合。
- 有时候，需要另外一个线程执行指定功能，接着在本线程的某个代码执行点等待异步执行的结果，可以采用这个方式。
```
// async example
#include <iostream>       // std::cout
#include <future>         // std::async, std::future

// a non-optimized way of checking for prime numbers:
bool is_prime (int x) {
  std::cout << "Calculating. Please, wait...\n";
  for (int i=2; i<x; ++i) if (x%i==0) return false;
  return true;
}

int main ()
{
  // call is_prime(313222313) asynchronously:
  std::future<bool> fut = std::async (is_prime,313222313);

  std::cout << "Checking whether 313222313 is prime.\n";
  // ...

  bool ret = fut.get();      // waits for is_prime to return

  if (ret) std::cout << "It is prime!\n";
  else std::cout << "It is not prime.\n";

  return 0;
}
```

## promise::get_future
- std::future和一个变量结合。
- 如果一个线程设置变量的值，而另外一个线程需要等待这个变量设置的结果，可以采用这个方式。
```
// promise example
#include <iostream>       // std::cout
#include <functional>     // std::ref
#include <thread>         // std::thread
#include <future>         // std::promise, std::future

void print_int (std::future<int>& fut) {
  int x = fut.get();
  std::cout << "value: " << x << '\n';
}

int main ()
{
  std::promise<int> prom;                      // create promise

  std::future<int> fut = prom.get_future();    // engagement with future

  std::thread th1 (print_int, std::ref(fut));  // send future to new thread

  prom.set_value (10);                         // fulfill promise
                                               // (synchronizes with getting the future)
  th1.join();
  return 0;
}
```

## packaged_task::get_future
- std::future和std::function的结合。
- 在生产者和消费者的场景中，生产者往往会传递function（task）给到消费者执行，如果生产者需要等待消费者执行的结果，可以采用这个方式。
```
// packaged_task::get_future
#include <iostream>     // std::cout
#include <utility>      // std::move
#include <future>       // std::packaged_task, std::future
#include <thread>       // std::thread

// a simple task:
int triple (int x) { return x*3; }

int main ()
{
  std::packaged_task<int(int)> tsk (triple); // package task

  std::future<int> fut = tsk.get_future();   // get future

  std::thread(std::move(tsk),33).detach();   // spawn thread and call task

  // ...

  int value = fut.get();                     // wait for the task to complete and get result

  std::cout << "The triple of 33 is " << value << ".\n";

  return 0;
}
```

## future和shared_future区别
方法都是一样的，除此以外：
- future是独占的，不能复制，只能移动。调用 .get() 后结果被获取，future 对象也会被标记为无效，无法再次访问。
- shared_future允许复制，因此可以共享所有权。你可以在多个 shared_future 实例上调用 .get()，使多个线程能够访问相同的结果而不需要转移所有权。
- 总结，future 适用于单一所有权，而 shared_future 适合在多个线程间共享结果。shared_future可以一个线程中多次get，也可以同时被多个线程调用get。
