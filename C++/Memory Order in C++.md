### Memory Order in C++

自从C11开始，C++标准库开始支持原子操作，这意味着C++用户可以使用标准库中封装的原子性函数来实现lock-free编程了。之前一直没有用过这块的功能，这次刚好代码移植时遇到了几处代码使用了，刚好总结一下。无论是硬件还是编程语言，都有`Memory order`的概念，C++中引入的`std::memory_order`类型，指定了原子操作的内存序。内存序的定义如下^[1]^：

``` shell
Memory ordering describes the order of accesses to computer memory by a CPU. The term can refer either to the memory ordering generated by the compiler during compile time, or to the memory ordering generated by a CPU during runtime.
--
内存序是指CPU访问内存的顺序，既可以指编译器在编译时生成的内存访问顺序，也可以指在运行时CPU临时调度产生的内存访问顺序。
```

##### 0.1 内存序会导致的问题

一般来说内存序只对`lock-free`的编程场景有影响，在经典的有锁保护的场景下不存在内存序的问题（目前的锁实现会自动加上`full memory barrier`保证有序）。下面的一个[例子](http://senlinzhan.github.io/2017/12/04/cpp-memory-order/)说明了内存序可能导致的问题：

``` c++
          int x = 0;     // global variable
          int y = 0;     // global variable
		  
Thread-1:              Thread-2:
x = 100;               while (y != 200)
y = 200;                   ;
                       std::cout << x;
```

上面的例子中由于x,y的赋值操作可能被重排，所以Thread-2输出的值可能是0或者100。显然在lock-free的环境下，这种memory store reorder会带来严重的问题，因为x,y在其他线程可能有依赖关系，比如y是一个标志变量，记录着x的值是否准备好。

#### 1. 导致内存序改变的原因

当用户使用C++语言写出一组内存访问语句时，1）[编译器可能会在优化时进行内存序的重排](https://preshing.com/20120625/memory-ordering-at-compile-time/)，2）[在CPU运行时也会对内存序进行重排以提高执行效率](https://preshing.com/20120710/memory-barriers-are-like-source-control-operations/)。接下来首先看一下硬件上的内存序重排。CPU对内存序的重排和硬件上采用的内存模型有关。

##### 1.1 四种内存访问重排类型

根据内存操作类型，重排可能有四种类型：






