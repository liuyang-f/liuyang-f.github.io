### **5种顺序容器+3种容器适配器+8种关联容器**
> 顺序容器：array、vector、deque、list、forward_list。
关联容器：map、multimap、
set、multiset、
unordered_map、unordered_multimap、
unordered_set、unordered_multiset。
容器适配器：queue、priority_queue、stack。所谓容器适配器，是指它本身是一个封装层，必须依赖指定的底层容器（通过模板参数中的class Container指定），才能实现具体实现。
元组（tuple）：它可以把一组类型相同或不相同的元素组合到一起，且元素的数量不限。
①容器只能组合相同类型的元素，tuple可以组合任意不同类型的元素。
②pair可以看作是把tuple的size限制为2的一个特例，pair只能把一对元素组合到一	起。

### 5种顺序容器
**_1.array（顺序容器）_**
```
template < class T, size_t N > class array；
```
>（1）连续内存，固定大小，支持随机访问。
（2）与其他容器不同的点①对两个array执行swap操作，实际置换相应range内的元素，而不是交换内存地址。②不使用std::allocator 来动态的分配和回收内存空间。③array 可以被当作 std::tuple 使用。

**_2.vector（顺序容器）_**
```
template < class T, class Alloc = allocator< T > > class vector;
```
> （1）使用内存分配器【std::allocator】动态管理内存，连续内存，动态调整大小，支持随机访问。
（2）vector就是能够动态调整大小的array。
（3）和其他动态顺序容器【deque、list、forward_list】相比，vector在元素访问效率上最高，在尾部增删元素的效率也相对最高。
（4）vector的reallocate是一个很耗时的操作。
（5）vector的capacity()通常会大于size()。
（6）vecotr比array多了一些内存消耗，以换取更灵活的内存管理。

**_3.deque（顺序容器）_**
```
template < class T, class Alloc = allocate < T > > class deque;
```
> （1）使用内存分配器【std::allocator】动态管理内存，分段连续内存，动态调整大小，支持随机访问。
（2）vector使用的是单一的array，deque则会使用很多个离散的array来组织数据。如果说vector是连续的，deque则是分段连续的。
（3）deque（读作"deck"）是 double-ended queue 的缩写，是一个可以在首尾两端进行动态增删的顺序容器。
（4）deque与vector类似，但deque额外支持在头部动态增删元素。
（5）相比与vector和list，deque并不适合遍历；因为每次访问数据，deque底层都需要检查是否触达了内存片段的边界，造成额外的开销。
（6）deque的核心优势，是双端都支持高效的增删操作。

**_4.List（顺序容器）_**
```
template < class T, class Alloc = allocator < T > > class list;
```
> （1）使用内存分配器【std::allocator】动态管理内存，离散内存，双链表，可顺序访问，但不支持随机访问。
（2）支持在任意位置都可以快速插入和删除元素的容器，且支持双向遍历。
（3）和其他顺序容器【array、vector、deque】相比，list的最大优势在于支持在任意位置插入、删除和移动元素。
（4）list的最大缺点，是不支持随机访问；访问某个元素，需要一个个节点遍历，直到到达要访问的元素。
（5）list的空间使用效率不高，由于元素的内存地址是离散的而非连续的，导致需要保存额外的关联信息。

**_5.forward_list（顺序容器）_**
```
template < class T, class Alloc = allocator < T > > class forward_list;
```
> （1）使用内存分配器【std::allocator】动态管理内存，离散内存，单链表，可顺序访问，但不支持随机访问。
（2）forward_list相比于list的核心区别是它是一个单链表，因此每个元素只会与相邻的下一个元素关联，只能单向遍历。
（3）forward_list相比于list，占用的空间内存更小，且插入和删除的效率稍稍高于list。
（4）设计forward_list的目的，为了达到不输于任何一个C风格手写链表的极致效率。
（5）list兼顾了接口丰富，牺牲了效率；而forward_list舍弃了不必要的接口，只为追求极致效率。


### 3种容器适配器
**_6.queue（容器适配器：先进先出型容器FIFO）_**
```
template < class T, class Container = deque < T > > class queue;
```
> （1）queue【普通队列】是一个专为FIFO设计的容器适配器。
（2）只能从一端插入、从另一端删除。
（3）默认情况下，queue使用deque作为底层容器。
（4）queue可以接纳任何一个至少支持下列接口的容器作为底层容器；当然用户也可以自定义一个包含下列接口的容器。
empty();  size();	front();	back();	push_back();	pop_front();
（5）在标准模板库容器中，deque和list满足上述要求。

**_7.priority_queue（容器适配器：严格弱序【Strict Weak Ordering】，优先级队列）_**
```
template < class T, class Container = vector < T >,
      class Compare = less < typename Container::value_type> > class priority_queue;
```
> （1）priority_queue的核心特点在于其实严格弱序特性，即priority_queue保证容器中的第一个元素始终是所有元素中最大的。
（2）用户在实例化一个priority_queue时，必须为元素类型【class T】重载 < 运算符，以用于元素排序。
（3）priortiy_queue内部维护一个基于二叉树的大顶堆数据结构，在这个数据结构中，最大的元素始终处于堆顶部，且只有堆顶部的元素【max heap element】才能被访问和获取。
（4）默认情况下，priority_queue使用vector作为底层容器。
（5）相比于queue【普通队列】的先进先出FIFO，priority_queue实现了最高优先级先出。
（6）priority_queue的底层容器必须支持随机访问和至少以下接口：
empty();  size();	front();	push_back();	pop_back();
（7）标准模板库中的vector和deque能够满足上述要求。

**_8.stack（容器适配器：后进先出型容器LIFO）_**
```
template < class T, class Container = deque < T > > class stack;
```
> （1）stack【栈】是一个专为LIFO设计的容器适配器，也即只能从一端插入和删除。
（2）默认情况下，stack使用deque作为底层元素。
（3）stack的特点是后进先出（一端进出），不允许遍历；任何时候外界只能访问stack顶部的元素，只有在移除stack顶部的元素后，才能访问下方的元素。
（4）stack容器应用广泛，例如编辑器中的undo【撤销操作】机制，就是用栈来记录连续的操作；stack的设计场景和自助餐中堆叠的盘子、摞起来的一堆书类似。
（5）stack可以接纳任何一个至少支持以下接口的容器作为底层容器：
empty();  size();	back();	push_back();	pop_back();
（6）在标准模板库容器中，vector和deque、list满足上述要求。


### 8种关联容器
_**9.map（有序关联容器，key唯一）**_
```
template < class Key, // map::key_type
class T, // map::mapped_type
class Compare = less < Key >, // map::key_compare
class Alloc = allocator < pair < const Key, T > > // map::allocator_type
class map;
```
> （1）map是一个关联容器，使用内存分配器动态管理内存。
（2）其元素类型是由 key 和 value 组成的 std::pair ，实际上 map 中元素的类型是：typedef pair < const Key, T > value_type；
（3）关联容器是指所有元素的检索都是通过元素 key 进行的（而非元素的内存地址）；
（4）map通过底层的红黑树数据结构来将所有的元素按照 key 的相对大小进行排序，所实现的排序效果也是严格弱序特性；开发者需要重载 key 的 < 运算符或者模板参数中的 class Comprae；
（5）Map的访问元素的速度要稍慢于 unordered_map；map支持在一个子集合上进行直接迭代器访问。

**_10.multimap（有序关联容器，允许不同元素 key 相同）_**
```
template < class Key, // map::key_type
class T, // map::mapped_type
class Compare = less < Key >, // map::key_compare
class Alloc = allocator < pair < const Key, T > > // map::allocator_type
class map;
```
> （1）使用内存分配器管理内存。
（2）multimap与map底层原理完全一样，使用红黑树对元素按 key 的比较关系，进行快速的插入、删除和检索操作。
（3）在multimap容器中，元素的key与元素value的映射关系是一对多的；因此，multimap是多重映射容器。
（4）访问一个key对应的所有value方法有三种：①通过迭代器和lower_bound()和upper_bound()访问；②使用equal_range()访问；③通过find() 配合count() 访问。

**_11.set（有序关联容器，元素自身即key，元素有唯一性）_**
```
template < class T, // set::key_type/value_type
class Compare = less < T >, // set::key_compare/value_compare
class Alloc = allocator < T > // set::allocator_type
class set;
```
> （1）使用内存分配器管理内存。
（2）set和map一样，差别是元素的value本身是否也作为key来标识自己。

**_12.multiset（有序关联容器，可以保存多个相同的key）_**
```
template < class T, // multiset::key_type/key_value
class Compare = less < T >, // multiset::key_compare/value_compare
class Alloc = allocator< T >, // multiset::allocator_type
class multiset;
```
> （1）使用内存分配器动态管理内存。
（2）multiset和set底层都是红黑树，multiset相比于set支持保存多个相同的元素。

**_13.unordered_map（无序关联容器，key是唯一的）_**
```
template < class Key, // unordered_map::key_type
class T, // unordered_map::mapped_type
class Hash = hash<Key>, // unordered_map::hasher
class Pred = equal_to<Key>, // unordered_map::key_equal
class Alloc = allocator< pair < const Key, T> > // unordered_map::allocator_type
class unordered_map;
```
> （1）使用内存分配器动态管理内存。
（2）unordered_map以哈希表（hash table）作为底层数据结构来组织数据。
（3）不支持排序，在使用迭代器做范围访问时（迭代器自加自减）效率更低；但是直接访问元素速度更快（尤其规模很大时），因为它是通过直接计算key的哈希值来访问元素。
（4）map与unordered_map比较：map的增删效率更高，unordered_map访问元素的效率更高；unordered_map的内存占用更高，因为底层的哈希表需要预分配足够的内存。
（5）unoreded_map更适用于增删操作不多，但需要频繁访问，且内存资源充足的场合。

**_14.unordered_multimap（无序关联容器，允许不同元素 key 相同）_**
```
template < class Key, // unordered_multimap::key_type
class T, // unordered_multimap::mapped_type
class Hash = hash<Key>, // unordered_multimap::hasher
class Pred = equal_to<Key>, // unordered_multimap::key_equal
class Alloc = allocator< pair < const Key, T> > // unordered_multimap::allocator_type
class unordered_multimap;
```
>（1）使用内存分配器动态管理内存。
（2）unordered_multimap 是对 unordered_map 的拓展，唯一区别在于 unordered_multimap 允许不同元素的 key 相同。

**_15.unordered_set（无序关联容器，元素是唯一的）_**
```
template < class Key, // unordered_set::key_type/value_type
class Hash = hash<Key>, // unordered_set::hasher
class Pred = equal_to<Key>, // unordered_set::key_equal
class Alloc = allocator< pair < const Key, T> > // unordered_set::allocator_type
class unordered_set;
```
> （1）使用内存分配器动态管理内存。
（2）所有unordered_XXX类容器的特点都是以哈希表作为底层结构；所有 XXX_set 类容器的特点都是「元素自身也作为key」来标识自己。我们在把两类特性叠加到一起，就得到了 unordered_set。
（3）和所有的unordered_XXX类容器一样：1. unordered_set 直接用迭代器做范围访问时（迭代器自加自减）效率更低，低于 set；2. 但 unordered_set 直接访问元素的速度更快（尤其在规模很大时），因为它通过直接计算 key 的哈希值来访问元素。