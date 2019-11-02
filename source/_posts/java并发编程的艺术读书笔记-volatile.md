---
title: java并发编程的艺术读书笔记（volatile）
categories:
  - 多线程
tags:
  - JMM
  - 读书笔记
  - volatile
abbrlink: 26538
---



## java 并发编程的艺术读书笔记（volatile）

### volatile的内存语义

#### 一.volatile的特性

1. **可见性**：对一个volatile变量的读，总是能看到（任意线程）对这个volatile变量最后的写入。
2. **原子性**：对任意单个volatile变量的读/写具有原子性，但类似于volatile++这种复合操作不具有原子性。

####  二.volatile写-读建立的happens-before关系

```
class VolatileExample {
    int a = 0;
    volatile boolean flag = false;
    public void writer() {
        a = 1; // 1
        flag = true; // 2
    } 
    public void reader() {
        if (flag) { // 3
            int i = a; // 4
        }
    }
}
```

假设线程A执行writer()方法之后，线程B执行reader()方法。根据happens-before规则，这个过程建立的happens-before关系可以分为3类： 

1. 根据程序次序规则，1 happens-before 2;3 happens-before 4 
2. 根据程序次序规则，1 happens-before 2;3 happens-before 4 
3. 根据happens-before的传递性规则，1 happens-before 4

#### 三.volatile写-读的内存语义

**当写一个volatile变量时，JMM会把该线程对应的本地内存中的共享变量值刷新到主内存 **



#### 四.volatile读的内存语义如下：

**当读一个volatile变量时，JMM会把该线程对应的本地内存置为无效。线程接下来将从主内存中读取共享变量 **



#### 五.volatile内存语义的实现 

为了实现volatile的内存语义，编译器在生成字节码时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。对于编译器来说，发现一个最优布置来最小化插入屏障的总数几乎不可能。为此，JMM采取保守策略。下面是基于保守策略的JMM内存屏障插入策略 。

* 在每个volatile写操作的前面插入一个StoreStore屏障 。
* 在每个volatile写操作的后面插入一个StoreLoad屏障 。
* 在每个volatile读操作的后面插入一个LoadLoad屏障 。
* 在每个volatile读操作的后面插入一个LoadStore屏障 。

volatile写插入内存屏障后生成的指令序列示意图 

![](<https://raw.githubusercontent.com/mzl1989325/ImageBed/master/threads/volatile1.jpg>)







volatile读插入内存屏障后生成的指令序列示意图 

![](<https://raw.githubusercontent.com/mzl1989325/ImageBed/master/threads/volatile2.jpg>)

图中的StoreStore屏障可以保证在volatile写之前，其前面的所有普通写操作已经对任意处理器可见了。这是因为StoreStore屏障将保障上面所有的普通写在volatile写之前刷新到主内存。

这里比较有意思的是，volatile写后面的StoreLoad屏障。此屏障的作用是避免volatile写与后面可能有的volatile读/写操作重排序。因为编译器常常无法准确判断在一个volatile写的后面是否需要插入一个StoreLoad屏障（比如，一个volatile写之后方法立即return）。为了保证能正确实现volatile的内存语义，JMM在采取了保守策略：在每个volatile写的后面，或者在每个volatile读的前面插入一个StoreLoad屏障。从整体执行效率的角度考虑，JMM最终选择了在每个volatile写的后面插入一个StoreLoad屏障。因为volatile写-读内存语义的常见使用模式是：一个
写线程写volatile变量，多个读线程读同一个volatile变量。当读线程的数量大大超过写线程时，选择在volatile写之后插入StoreLoad屏障将带来可观的执行效率的提升。从这里可以看到JMM在实现上的一个特点：首先确保正确性，然后再去追求执行效率。

![volatile3](<https://raw.githubusercontent.com/mzl1989325/ImageBed/master/threads/volatile3.jpg>)



图中的LoadLoad屏障用来禁止处理器把上面的volatile读与下面的普通读重排序。LoadStore屏障用来禁止处理器把上面的volatile读与下面的普通写重排序。



上述volatile写和volatile读的内存屏障插入策略非常保守。在实际执行时，只要不改变volatile写-读的内存语义，编译器可以根据具体情况省略不必要的屏障。下面通过具体的示例代码进行说明 

```
class VolatileBarrierExample {
	int a;
	volatile int v1 = 1;
	volatile int v2 = 2;
	void readAndWrite() {
		int i = v1; // 第一个volatile读
		int j = v2; // 第二个volatile读
		a = i + j; // 普通写
		v1 = i + 1; // 第一个volatile写
		v2 = j * 2; // 第二个 volatile写
	} …
	// 其他方法
}
```



注意，最后的StoreLoad屏障不能省略。因为第二个volatile写之后，方法立即return。此时编译器可能无法准确断定后面是否会有volatile读或写，为了安全起见，编译器通常会在这里插入一个StoreLoad屏障。

![](<https://raw.githubusercontent.com/mzl1989325/ImageBed/master/threads/volatile4.jpg>)

上面的优化针对任意处理器平台，由于不同的处理器有不同“松紧度”的处理器内存模型，内存屏障的插入还可以根据具体的处理器内存模型继续优化。以X86处理器为例，上图中除最后的StoreLoad屏障外，其他的屏障都会被省略 。

前面保守策略下的volatile读和写，在X86处理器平台可以优化成如下图所示。 

X86处理器仅会对写-读操作做重排序。X86不会对读-读、读-写和写-写操作做重排序，因此在X86处理器中会省略掉这3种操作类型对应的内存屏障。在X86中，JMM仅需在volatile写后面插入一个StoreLoad屏障即可正确实现volatile写-读的内存语义。这意味着在X86处理器中，volatile写的开销比volatile读的开销会大很多（因为执行StoreLoad屏障开销会比较大） 

![volatile5](<https://raw.githubusercontent.com/mzl1989325/ImageBed/master/threads/volatile5.jpg>)

