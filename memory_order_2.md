# 内存顺序（Memory Order）问题（二）

上一篇[Blog](memory_order_1.html)介绍了内存模型，并介绍了两种内存顺序，
`memory_order_acquire`（Acquire）和`memory_order_release`（Release）。
个人认为，这两种内存顺序是C++定义的六种内存顺序中最重要的两种，
只有理解了Acquire和Release的语义，才能更好理解其他四种内存顺序的语义。
更进一步，在实际使用场景中，Acquire和Release是最常见的两种内存顺序。

如何判断该使用哪种内存顺序？这是开发者在使用原子类型和无锁化编程时最常碰到的问题。
本篇Blog用实际的例子来说明，如何判断该使用哪种内存顺序，并同时介绍Happen-Before关系和Synchronize-With关系。

## Happen-Before关系

先来看一个简单的例子，这个例子解释了Happen-Before关系：
```
int data = 0;
int flag = 0;

// thread 1
void thread_func1() {
    data = 42;
    flag = 1; // 事件1
}

// thread 2
void thread_func2() {
    if (flag == 1) // 事件2
        printf("%d", data);
}
```
上面的例子里定义了两个全局变量，线程1设置`flag = 1`表示完成对`data`的赋值，
线程2读取`flag`的值用以判断线程1是否完成对`data`的赋值，如果`flag == 1`则输出`data`的值。

我们对上面的例子期望两种合理的结果：要么线程2输出`data`的值`42`，要么不输出。
也就是说要么事件1（`thread_func1`里对`flag`赋值）Happen-Before事件2（`thread_func2`里判断`flag == 1`），
要么事件2 Happen-Before事件1。
但是，由于上面的例子没有用信号量，也没有用原子操作，来对两个线程进行同步，线程2有可能输出`data`的值为0，为什么呢？
因为编译器或CPU会对程序进行优化，使得指令的执行顺序跟代码的逻辑顺序不一致。比如编译器可能对`thread_func2`进行如下优化：
```
// thread 2
void thread_func2() {
    int tmp = data;
    if (flag == 1)
        printf("%d", tmp);
}
```
这里`tmp`可能是某个寄存器，编译器优化`thread_func2`导致在判断`flag == 1`前把`data`的值先载入寄存器，此时`data`的值可能为0，
判断完`flag == 1`之后再输出寄存器的值，此时即便`data`已经被`thread_func1`赋值为1，但是寄存器tmp里的值仍然是0。
因此，为了保证上面的例子运行产生合理的结果，我们需要确保上面提到的Happen-Before关系。简单的做法是信号量，这里我们用内存顺序和原子操作来实现。

先来考虑如何用内存顺序来确保上述例子中两个线程间的Happen-Before关系。
从根本上上讲，我们要用内存顺序来告诉编译器和CPU确保指令执行顺序和代码的逻辑顺序一致。
上述例子里，`thread_func1`里的两行赋值语句（两个写操作）顺序不能颠倒，`thread_func2`里判断语句和打印语句（两个读操作）顺序不能颠倒：
```
int data = 0;
int flag = 0;

// thread 1
void thread_func1() {
    data = 42;
    // 写操作之前的写操作，之间的顺序不能改变
    flag = 1; // 事件1
}

// thread 2
void thread_func2() {
    if (flag == 1) // 事件2
        // 读操作之后的读操作，之间的顺序不能改变
        printf("%d", data);
}
```
不熟悉读写操作顺序的读者建议先读一下上一篇[Blog](memory_order_1.html)里介绍的四种读操作与写操作的先后顺序关系。
回想上一篇[Blog](memory_order_1.html)定义过Acquire和Release的语义：

|内存顺序|先后次序|语义|
|---|---|---|
|Acquire|读操作在前|读读、读写|
|Release|写操作在后|读写、写写|

可以看出：要规约“写操作之前的写操作之间的顺序不能改变”（写写），要采用Release语义；
要规约“读操作之后的读操作，之间的顺序不能改变”（读读），要采用Acquire语义。

确定了内存顺序，我们再考虑该如何使用原子操作。
一种做法，我们可以让`data`成为原子变量，那就不需要`flag`这个通知变量了，两个线程直接原子访问`data`。
但是实际中，往往`data`代表的数据会比较大，不适合作为原子变量，因此才需要`flag`这个通知变量。
因此，我们让`flag`成为原子变量，两个线程原子访问`flag`来实现同步：
```
#include <atomic>

std::atomic_int flag(0); // 初始值为零
int data = 0;

// thread 1
void thread_func1() {
    data = 42;
    flag.store(1, // 事件1
            std::memory_order_release); 
}

// thread 2
void thread_func2() {
    int ready = flag.load( // 事件2
            std::memory_order_acquire);
    if (ready == 1)
        printf("%d", data);
}
```

## Synchronize-With关系

Synchronize-With关系是指，当程序运行时，事件1 Happen-Before事件2，确保事件2得到事件1发生的通知，即把事件1同步给事件2。

回到上面用内存顺序和原子操作的实现，确保了事件1和事件2之间存在Happen-Before关系，两个线程运行结果无外乎两种：
1. `thread_func1`给`flag`赋值之后，`thread_func2`读取`flag`的值，并输出`data`的值为42，即事件1 Happen-Before事件2；
2. `thread_func1`给`flag`赋值之前，`thread_func2`读取`flag`的值，然后不输出`data`的值，即事件2 Happen-Before事件1。

如果采用如下所示信号量机制来实现事件1和事件2之间的同步：
```
sem_t flag; 
// 初始化信号量，初始值为0，最大值为1
sem_init(&flag, 0, 1); 

int  = 0;

void thread_func1() {
    data = 42;
    sem_post(&flag); // 事件1
}

void thread_func2() {
    sem_wait(&flag); // 事件2
    printf("%d", data);
}
```
那么这两个线程运行结果只有一种，即只有一种Happen-Before关系，事件1 Happen-Before事件2：
* 不论`thread_func1`和`thread_func2`谁先开始运行，`thread_func2`都会等`thread_func1`执行完`sem_post(&flag)`之后，才输出`data`的值42。

显然，大家看到了基于原子操作和内存顺序，与基于信号量的实现，得到不同的结果。
这也就是我在上一篇[Blog](memory_order_1.html)里提到的，原子操作和内存顺序，跟基于锁和信号量实现的线程间同步关系有本质的差异。
基于锁和信号量的线程间同步关系，比基于原子操作和内存顺序的线程间同步关系要更强。怎么理解呢？

回到之前的例子，基于信号量实现两个线程间的同步，只有一种运行结果，
`thread_func1`里`sem_post(&flag)`一定在`thread_func2`输出`data`的值之前。
也就是说，信号量确保了事件1 Happen-Before事件2，同时也在运行时确保了事件1 Synchronize-With事件2
（通过`thread_func1`里`sem_post(&flag)`和`thread_func2`里`sem_wait(&flag)`来确保Synchronize-With关系），
因而基于信号量的实现保证最终结果一定是`thread_func2`输出`data`的值42。

但是，基于原子操作和内存顺序实现两个线程间的同步，会有两种运行结果，要么`thread_func2`输出`data`的值42，要么`thread_func2`不输出`data`的值。
也就是说，基于原子操作和内存顺序，只能保证事件1和事件2之间存在Happen-Before关系，
即要么事件1 Happen-Before事件2，要么事件2 Happen-Before事件1。
但是，基于原子操作和内存顺序，无法保证一定只有事件1 Happen-Before事件2这一种关系。
另外，在运行时，如果事件1 Happen-Before事件2，
基于原子操作和内存顺序的实现可以确保事件1 Synchronize-With事件2
（通过`thread_func1`里`flag.store(1, std::memory_order_release)`和`thread_func2`里`flag.load(std::memory_order_acquire)`来确保）；
如果在运行时，事件2 Happen-Before事件1，
那基于原子操作和内存顺序的实现无法确保事件1和事件2之间有Synchronize-With关系。

由此，基于锁和信号量的线程同步机制，确保运行时的Happen-Before关系（事件先后关系）和Synchronize-With关系（事件通知到达）；
基于锁和信号量，只能确保两个事件之间在运行时存在Happen-Before关系（两个事件一定有先有后，但不保证谁先谁后），
而且只有事先确定的某种Happen-Before关系在运行时发生时，才保证两个事件的Synchronize-With关系。
