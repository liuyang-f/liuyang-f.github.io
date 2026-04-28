- 什么是风险指针
  这是解决在多线程中，如何安全释放内存的一种方法。思想：
  - 当需要使用某个指针的时候，把该指针标记为风险指针；当不需要使用时，清除该风险指针；
  - 释放指针之前判断该指针是否为风险指针，如果不是，释放指针；如果是，添加到回收列表；
  - 周期检查回收列表，清除表中的非风险指针（表中的风险指针随时会变为非风险指针），安全回收内存；


- 风险指针的使用
  - 关联函数
    - get_hazard_pointer_for_current_thread()
	  生成一个对象用于标记风险指针，当一个指针被放入该对象，这个指针就被标记为风险指针。
	- outstanding_hazard_pointers_for()
	  判断一个指针是否为风险指针。
	- reclaim_later()
	  把风险指针放入回收队列中。
	- delete_nodes_with_no_hazards()
	  清除掉回收队列中的非风险指针。
  - 代码分析：
    - （1）这段程序功能，是把head链表中头部的一个node数据从表中删掉，然后返回该node数据；
	- （2）语句②以前功能，head修改为原head后面的node；由于使用compare_exchange_strong更新head的时候，需要读取原head数据，故把原head标记为风险指针，并在结束后清除风险指针【语句②】；
	- （3）把old_head存储的数据给到res，然后返回res；使用res.swap语句，是因为old_head不需要使用data，直接转移所有权给到res；old_head本身是使用new创建的内存，需要解决delete的问题；
	- （4）判断old_head是否有标记为风险指针【语句③】，如果不是风险指针，直接delete即可；如果是风险指针，添加至回收列表，随后在某个时刻变为非风险指针后再执行删除；
  ```
  struct node
  {
    std::shared_ptr<T> data;
    node* next;

    node(T const& data_):
      data(std::make_shared<T>(data_))
    {}
  };
  std::atomic<node*> head;
  
  std::shared_ptr<T> pop()
  {
    std::atomic<void*>& hp=get_hazard_pointer_for_current_thread();
    node* old_head=head.load();
    do
    {
      node* temp;
      do  // 1 直到将风险指针设为head指针
      {
        temp=old_head;
        hp.store(old_head);
        old_head=head.load();
      } while(old_head!=temp);
    }
    while(old_head &&
      !head.compare_exchange_strong(old_head,old_head->next));
    hp.store(nullptr);  // 2 当声明完成，清除风险指针
    std::shared_ptr<T> res;
    if(old_head)
    {
      res.swap(old_head->data);
      if(outstanding_hazard_pointers_for(old_head))  // 3 在删除之前对风险指针引用的节点进行检查
      {
        reclaim_later(old_head);  // 4
      }
      else
      {
        delete old_head;  // 5
      }
      delete_nodes_with_no_hazards();  // 6
    }
    return res;
  }
  ```


- 风险指针的实现
  - 关键结构：get_hazard_pointer_for_current_thread()
  - 代码分析：
    - （1）结构hazard_pointer存放线程id和风险指针，因为多线程操作所以都使用了原子变量；
	- （2）数组hazard_pointers，预分配一段空间，风险指针就存在这块内存区域；
	- （3）构造函数hp_owner，从hazard_pointers中取出一个未使用的空间，标记为本线程使用，然后使用get_pointer()返回引用把pointer暴露出去，用于在外部存储风险指针；
	- （4）析构函数，把从hazard_pointers中取出的一个空间用完后，设置为未使用状态，id标记为默认值，pointer标记为nullptr，类似用完还给hazard_pointers；
	- （5）语句⑤处，使用compare_exchange_strong把判断和赋值合为一个原子操作，即只有id为默认值说明该空间未使用，然后标记id本线程id，暴露pointer后续存放风险指针；
  ```
  unsigned const max_hazard_pointers=100;
  struct hazard_pointer
  {
    std::atomic<std::thread::id> id;
    std::atomic<void*> pointer;
  };
  hazard_pointer hazard_pointers[max_hazard_pointers];
  
  class hp_owner
  {
    hazard_pointer* hp;
  
  public:
    hp_owner(hp_owner const&)=delete;
    hp_owner operator=(hp_owner const&)=delete;
    hp_owner():
      hp(nullptr)
    {
      for(unsigned i=0;i<max_hazard_pointers;++i)
      {
        std::thread::id old_id;
        if(hazard_pointers[i].id.compare_exchange_strong(  // 6 尝试声明风险指针的所有权
           old_id,std::this_thread::get_id()))
        {
          hp=&hazard_pointers[i];
          break;  // 7
        }
      }
      if(!hp)  // 1
      {
        throw std::runtime_error("No hazard pointers available");
      }
    }
  
    std::atomic<void*>& get_pointer()
    {
      return hp->pointer;
    }
  
    ~hp_owner()  // 2
    {
      hp->pointer.store(nullptr);  // 8
      hp->id.store(std::thread::id());  // 9
    }
  };
  
  std::atomic<void*>& get_hazard_pointer_for_current_thread()  // 3
  {
    thread_local static hp_owner hazard;  // 4 每个线程都有自己的风险指针
    return hazard.get_pointer();  // 5
  }
  ```
  
  - 判断是否为风险指针：outstanding_hazard_pointers_for()
  - 只需要该指针在hazard_pointers风险队列中存放，即可判定该指针为风险指针；
  ```
  bool outstanding_hazard_pointers_for(void* p)
  {
    for(unsigned i=0;i<max_hazard_pointers;++i)
    {
      if(hazard_pointers[i].pointer.load()==p)
      {
        return true;
      }
    }
    return false;
  }
  ```
  
  - 把风险指针放入回收队列：data_to_reclaim()，add_to_reclaim_list()
    - 引入一个新的类型data_to_reclaim管理风险指针，并用new创建对象；
	- add_to_reclaim_list()中代码很简洁，使用compare_exchange_weak是因为nodes_to_reclaim多线程中随时可能变化，需要保证node->next指向旧的nodes_to_reclaim，才能把nodes_to_reclaim更新为node；
  - 清空回收队列中的非风险指针：delete_nodes_with_no_hazards()
    - 语句⑥，使用exchange获取回收队列，同时把nodes_to_reclaim置为nullptr，表明即将进行回收工作；
	- 遍历取出的回收队列，再次使用outstanding_hazard_pointers_for判断每个元素是否为风险指针，如果是风险指针，再次放入回收队列中；如果不是风险指针直接delete队列元素；
	- delete队列元素，实际上是执行data_to_reclaim的析构函数，最终使用删除器删除data_to_reclaim中保存的指针data；
  - 结构data_to_reclaim，用于管理风险指针：
    - data，存放风险指针；
	- deleter，函数对象，执行删除风险指针的任务；
	- next，回收队列连接指针；
	- 引入模板参数T，是因为删除风险指针需要明确指针类型，即函数do_delete()中的内容；
	- 构造函数，给deleter一个默认的删除函数；
	- 析构函数，使用删除函数deleter；
  - 原子变量nodes_to_reclaim，回收队列的头部指针；回收队列元素都是data_to_reclaim结构，使用nodes_to_reclaim和data_to_reclaim::next可以遍历完整的回收队列；
  ```
  template<typename T>
  void do_delete(void* p)
  {
    delete static_cast<T*>(p);
  }
  
  struct data_to_reclaim
  {
    void* data;
    std::function<void(void*)> deleter;
    data_to_reclaim* next;
  
    template<typename T>
    data_to_reclaim(T* p):  // 1
      data(p),
      deleter(&do_delete<T>),
      next(0)
    {}
  
    ~data_to_reclaim()
    {
      deleter(data);  // 2
    }
  };
  
  std::atomic<data_to_reclaim*> nodes_to_reclaim;
  
  void add_to_reclaim_list(data_to_reclaim* node)  // 3
  {
    node->next=nodes_to_reclaim.load();
    while(!nodes_to_reclaim.compare_exchange_weak(node->next,node));
  }
  
  template<typename T>
  void reclaim_later(T* data)  // 4
  {
    add_to_reclaim_list(new data_to_reclaim(data));  // 5
  }
  
  void delete_nodes_with_no_hazards()
  {
    data_to_reclaim* current=nodes_to_reclaim.exchange(nullptr);  // 6
    while(current)
    {
      data_to_reclaim* const next=current->next;
      if(!outstanding_hazard_pointers_for(current->data))  // 7
      {
        delete current;  // 8
      }
      else
      {
        add_to_reclaim_list(current);  // 9
      }
      current=next;
    }
  }
  ```


- 风险指针的问题
  - 问题
    - 每一次检查风险指针【outstanding_hazard_pointers_for】，都需要扫描max_hazard_pointers个原子变量，原子变量比较耗时。
    - pop需要检查风险指针，不仅需要遍历完整的回收队列，还需要遍历风险指针队列，每次pop都需要遍历两个队列并访问原子变量，这成为一个性能瓶颈。
  - 方案
    - 考虑不需要每次pop都去清空回收队列，可以用一个原子变量记录回收队列的总数，当总数超过n×max_hazard_pointers后，pop执行清空回收队列的动作。
	  存在问题
	  - 多个线程访问同一个回收队列，存在竞争关系。
	  - 回收队列的总数，是个原子变量，产生新的多线程访问原子变量的场景。
	- 另一个思路，每个线程维护一个回收队列，所有线程的回收队列总数不会超过max_hazard_pointers*×max_hazard_pointers个。
	  - 回收队列作为线程内数据，不存在多线程竞争。
	  - 每次pop，各个线程检查自己的回收队列，进行清空。
	  - 如果线程退出时，自己还存在回收队列，通过全局变量，把自己的回收队列，添加到其他的线程回收队列中，由其他线程清空回收队列。