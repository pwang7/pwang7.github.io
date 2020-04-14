# 垃圾回收机制与无锁化编程（Garbage Collection and Lock-Free Programming）

垃圾回收机制（GC）对大部分开发者来说应该不陌生，特别是Java开发者或多或少都跟GC打过交道。
GC的优点是实现对堆上分配的内存动态回收，避免内存泄漏。但是GC的缺点是对性能有一定影响，特别是stop the world问题，
而且GC什么时候回收内存是不确定的，开发者无法知晓。

在无锁化编程场景下，用Java这种有GC的语言，一定程度简化了对内存的管理，降低了无锁化编程的难度。
无锁化编程，顾名思义，就是不用锁。无锁化编程特指在多线程编程的时候，对线程间共享数据的并发修改不使用锁，
而采用基于硬件提供的原子操作能力来修改共享数据，进而提升性能，减少使用锁带来的互斥开销。

最常用的硬件提供的原子操作是Compare and Swap（简称CAS），编程语言基于硬件提供的原子操作能力封装出CAS库函数调用。
先看下C++里的CAS函数调用：
```
bool __sync_bool_compare_and_swap (type *ptr, type oldval, type newval, ...);
type __sync_val_compare_and_swap (type *ptr, type oldval, type newval, ...);
```
这两个函数的功能一样，只是返回值不同。为了简化描述，略去了一些细节
（这个细节是关于内存顺序，即程序指令在执行时的顺序问题，这个问题是无锁化编程能否正确实现的关键问题之一，后面我再专门详细介绍）。
第一个CAS函数比较指针ptr指向的变量，看是不是等于oldval，如果相等就把ptr指向的变量改为newval，并返回true，否则不做任何改变并返回false。
第二个CAS函数也是比较ptr指向的变量是否等于oldval，如果相等就把该变量改为newval，并返回oldval，如果不等就不做改变，并返回ptr指向变量的当前值。
C++的CAS函数调用可以保证对ptr指向的变量的修改是原子的，要么更改完成，要么不做更改。

再看下硬件提供的原子操作。比如X86架构提供了一条CAS指令cmpxchg，这条指令在执行时是由硬件保证原子性，不会被打断。
其他硬件架构，比如ARM、PowerPC也提供类似的CAS原子操作，但是和X86的实现机制不一样，这里先不展开。
编程语言的编译器在X86架构下编译时，会负责把CAS库函数调用编译成基于cmpxchg的机器代码。
比如C++的编译器GCC在编译时如果碰到上面两个CAS函数调用，会生成包含cmpxchg指令的目标码。

## 无锁化编程示例：无锁化堆栈（Lock-Free Stack）的Java实现

先来看个简单的无锁化编程的例子，一个无锁化堆栈的Java实现（从网上找了一段现成的代码，没经过编译验证，仅做示例）：
```
import java.util.concurrent.atomic.*;

import net.jcip.annotations.*;

/**
* ConcurrentStack, nonblocking stack using Treiber's algorithm
*
* @author Brian Goetz and Tim Peierls
*/
@ThreadSafe
public class ConcurrentStack <E> {
    AtomicReference<Node<E>> top = new AtomicReference<Node<E>>(); // 栈顶

    public void push( E item ) {
        Node<E> newNode = new Node<E>( item );
        Node<E> curTop;
        do {
          curTop = top.get();
          newNode.next = curTop;
        } while ( !top.compareAndSet( curTop, newNode ) ); // CAS调用修改栈顶
    }

    public E pop() {
        Node<E> curTop;
        Node<E> newTop;
        do {
            curTop = top.get();
            if ( curTop == null ) {
                return null;
            }
            newTop = curTop.next;
        } while ( !top.compareAndSet( curTop, newTop ) ); // CAS调用修改栈顶
        return curTop.item;
    }

    private static class Node <E> {
        public final E item;
        public Node<E> next;

        public Node( E item ) {
            this.item = item;
        }
    }
}
```
从代码可以看出，这个栈的实现完全没有用锁，栈顶top是当前堆栈顶端节点的原子引用AtomicReference，
每次出栈（pop）入栈（push）的时候，调用AtomicReference的compareAndSet方法来试图修改栈顶top。
因为有可能有多个线程竞争访问这个无锁化堆栈，即有可能有多个线程同时对栈顶进行修改，或同时pop、或同时push，或同时pop和push，
CAS的原子性保证了多个线程并发调用compareAndSet方法修改栈顶top时，仅有一个线程的调用能修改成功，其他线程的调用不成功，
所以pop和push操作里要用循环的方式重复调用compareAndSet方法试图修改栈顶top，直到成功返回。

从这个例子可以看出无锁化编程的一个显著特点，对共享数据的修改要多次重试CAS操作。
虽然CAS调用基于硬件提供的原子能力，但是CAS调用的代价也不小。
比如在X86多处理器架构下运行上面的无锁化堆栈，每次CAS方法调用会运行cmpxchg指令，这个指令使得多个处理器缓存里的栈顶top失效，
导致多个处理器要重新从内存加载top。如果频繁调用CAS试图修改共享数据，将导致处理器缓存里的共享数据频繁失效，这个对性能的影响也不小。
所以千万不要滥用CAS调用。

## 无锁化编程示例：无锁化堆栈（Lock-Free Stack）的C++实现

上面用Java实现无锁化堆栈，还是比较简单的，几十行代码就完成了。那用C++来实现无锁化堆栈会不会也很简单呢？
先看下面的无锁化堆栈C++实现片段（为了简化描述，还是省略了内存顺序的细节，代码没经过编译验证，仅做示例）：
```
#include <atomic>

struct Node {
    void* data;
    std::atomic< Node * > next;
};

std::atomic<Node *> top; // 栈顶
top.store( nullptr ); // 初始化栈顶为空指针

bool push( Node* new_node ) {
    Node * cur_top = top.load();
    while ( true ) {
        new_node->next.store( cur_top );
        if ( top.compare_exchange_weak( cur_top, new_node )) { // CAS调用修改栈顶
            return true;
        }
    }
}

Node * pop() {
    while ( true ) {
        Node * cur_top = top.load();
        if ( cur_top == nullptr ) {
            return nullptr; // 堆栈为空
        }
        Node * next = cur_top->next.load();
        if ( top.compare_exchange_weak( cur_top, next )) { // CAS调用修改栈顶
            return cur_top;
        }
    }
}
```
看上去C++的无锁化堆栈实现跟Java版本几乎一致，栈顶top也是原子类型，但是这个C++实现有问题。考虑如下使用无锁化堆栈的C++代码片段：
```
Node * new_node = new Node();
new_node.data = ...;
push( new_node );
...
Node * pop_node = pop();
if ( pop_node ) {
    ... // 消费出栈节点
    delete pop_node; // 不能保证没有其他线程在访问pop_node，所以此处不能delete
}
```
入栈的每个节点都是new出来的，所以大家可能觉得想当然出栈之后的每个节点在消费过过以后要被delete掉。
但是考虑多线程并发访问的场景，比如有两个线程同时调用出栈pop函数，假定这两个线程同时读取到当前栈顶```cur_top = top.load();```，
之后其中一个线程被抢占，另一个线程调用pop成功取出栈顶节点，并且消费完之后delete掉，
这时之前被抢占的线程恢复执行，判断`cur_top`不为空，然后读取`cur_top`的下一个节点```next = cur_top->next.load();```,
但是`cur_top`已经被另一个线程delete掉了，`cur_top`已经变成野指针了，一旦读取就会导致非法内存访问使得程序崩溃。

那为什么Java版本的实现就没有问题呢？是因为GC的缘故。Java里没有显式的delete操作，所有的内存回收是GC自动实现的。
回到上面两个线程并发调用出栈pop函数的场景，如果用Java实现，当两个线程都读取到当前栈顶的引用，其中一个线程被抢占，
另一个线程出栈调用成功，并完成消费出栈节点，此时Java的GC并不会回收出栈节点，因为GC发现出栈节点仍有被其他线程引用。
所以Java版本的无锁化编程是内存安全的。

# 对没有GC的语言做无锁化编程的思考

从上述无锁化堆栈的例子可以看出，Java的GC机制一定程度简化了无锁化编程，因为不用考虑内存回收的问题，Java的GC会安全的回收内存，
只是不能确定Java的GC什么时候回收内存。对比没有GC的语言，显然没有GC使无锁化编程变得复杂，因为实现的时候不得不自行考虑内存安全回收的问题。
这个问题相当于是在用C++这类没有GC的语言做无锁化编程的时候，要自行实现一个GC，专门处理无锁化编程场景下的内存回收问题，并保证内存安全同时防止内存泄漏。

针对这个问题已经有一些成型的算法，最简单的方法是Reference Counter（但是RC性能非常差，不实用，因此不做考虑），
复杂些的方法诸如Hazard pointer，Epoch-Based Reclamation或Quiescent State-Based Reclamation。
这些方法大都可以看做是一种确定型GC，更准确的说是相对Java的GC而言，这些方法执行内存回收的时刻是确定的（只是不同算法的具体内存回收触发逻辑不同）。
这种确定型GC的思路是，仅针对无锁化编程这种特定场景实现GC，降低无锁化编程的难度，而不是作为通用型GC，这比起Java的GC来说就简单多了。
因此，确定型GC相比Java的GC，一方面减少复杂度从而大大降低对程序性能的影响，另一方面因其内存回收的确定性所以防止stop the world问题出现。
当然对开发者而言确定型GC也是是透明的，开发者无需关心内存何时回收。限于篇幅，具体的细节这里先不展开，后续我再详细介绍这些确定型内存回收算法。

对没有GC的语言，比如C++，已经有一些针对无锁化编程的内存回收算法的实现，比如`libcds`，更进一步，`libcds`还实现了无锁化堆栈、无锁化队列、无锁化哈希表等等。
所以对于开发者而言，不需要重复造轮子，可以直接使用这些已有的实现来简化无锁化编程。
