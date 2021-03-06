---
layout: post
title: CPU是如何执行函数的?
date: 2020-06-04
tags: [jvm,cpu,函数]
author: loren1994
category: blog
---

##### 执行过程

首先，程序在Linux中由若干条指令构成，这些指令存在内存的某个位置，这个位置是由Linux指定分配，在分配到内存之前，存储在硬盘中，CPU要调用时，Linux才将他们分配至内存。

随后Linux告诉CPU程序入口点，即第一条指令的地址，CPU到相应地址获取指令，随后开始执行。

##### 那么CPU是如何读取指令的呢？

这里简单说一下，程序映射至内存后，PE loader将程序入口赋值给CPU的eip寄存器，然后通知CPU去执行，CPU读取并传送至指令缓冲区，此时eip的值增加，即下一条指令，最后执行。如此反复。

> eip寄存器：用来存储CPU要读取指令的地址，CPU通过eip寄存器读取即将要执行的指令。每次CPU执行完相应的汇编指令之后，eip寄存器的值就会增加。

指令一般包括以下几类：

1.把数据从内存加载到寄存器中

2.对寄存器数据进行运算

3.将寄存器的数据写入到内存

一旦遇到这样的指令：`"把寄存器ebp的值压到栈中"`，函数调用就开始了。

> ebp寄存器：一种特殊的寄存器，始终指向当前函数在一个栈的开始地址。对应栈帧开头。
>
> esp寄存器：一种特殊的寄存器，始终指向当前函数在一个栈的结束地址。对应栈帧结尾。

所以epb和esp之间的地址包含的指令就是对应的一个函数的信息。

##### 那么函数信息为什么在栈中？这里要说一下JVM内存模型。

> JVM内存区域分为：线程私有区（程序计数器、虚拟机栈、本地方法区）、线程共享区（堆、方法区）、直接内存。
>
> 直接内存：不受GC管控，用于提升特定场景下的性能（比如nio），避免java堆和native堆之间频繁来回复制数据。
>
> 本地方法区：和虚拟机栈类似，区别为它是为native方法服务。HotSpot直接将本地方法栈和虚拟机栈合二为一了。
>
> 方法区：存储被JVM加载的类信息、常量、静态变量、即时编译后的代码等。HotSpot为了将GC分代收集扩展至方法区，使用Java堆的永久代实现方法区。Java8中永久代被移除，使用元数据区代替，不在虚拟机中，使用本地内存。
>
> 虚拟机栈：java方法执行的内存模型，每个方法执行的同时都会创建一个栈帧，用于存储局部变量表、操作数栈、动态链接、方法出口等信息。

所以，多个栈帧（函数帧）在内存中排列起来，就像一个先进后出的栈一样，不过这个栈是从高地址向低地址排列。（栈底在上面）

##### 那么以下代码是如何运行的呢？

~~~~c
int x = 10;
int y = 20;
int sum = add(&x, &y);
printf("the sum is %d\n",sum);
~~~~

假如ebp地址为800，esp地址为776，每次操作4字节（在32位平台上，esp每次减少4字节）

"将值10放入ebp减4的地址（796）"

"将值20放入ebp减8的地址（792）"

"将地址796作为数据放到esp指向的地址776中"

"将地址792作为数据放到esp+4指向的地址780中"

"调用函数add"

到这里CPU会找到add函数返回以后的那条指令地址（假设地址是100），把它压入栈中

这时，将地址100的指令压入栈后，esp也发生了变化，指向了772的位置（776减4）

> 因为esp始终指向当前栈帧尾部

找到函数add 的指令，继续执行（ebp：800、esp：772）

##### 一个标准的函数起始代码

~~~~
push ebp ;保存当前ebp
mov ebp,esp ;EBP设为当前堆栈指针
sub esp, xxx ;预留xxx字节给函数临时变量.
~~~~

"把寄存器ebp的值压到栈里去"（ebp：800、esp：768）

"把esp的值赋给ebp"（ebp：768）

"把寄存器ebx的值压入栈"（ebp：768、esp：764）

这里额外把ebx这个寄存器压入栈， 是因为ebx可能被上个函数使用， 但是在add函数中也会用 ， 为了不破坏之前的值， 只能暂时放到内存里。

此时，768位置存储着 800，即上个函数帧的开始位置，764存储着ebx的值。

800到772为`调用者的函数帧`，768到764为`被调用函数的帧`

"把ebp加8的数据取出放到edx寄存器，即776，&x"

"把ebp加12的数据取出放到ecx寄存器，即780，&y"

"把edx中的值所指向的地址的数据取出来放到ebx中"

"把ecx中的值所指向的地址的数据取出来放到eax中"

此时就取到了值，ebx=10，eax=20

想必add函数的源码应该是

~~~~c
int add(int *xp , int *yp){
    int x = *xp;
    int y = *yp;
    ...
}
~~~~

"把ebx和eax的值加起来，放到eax寄存器中"

add函数已经完成，准备返回

"把esp指向的数据弹出到ebx寄存器"（恢复ebx之前的值）

"把ebp指向的数据弹出到ebp寄存器"（将ebp重新指向原函数帧所指向的800位置）

ebp：800、esp：772（因为数据弹出，所以栈底变为772）

此时add函数帧消失，换句话说，add函数帧的数据还在内存里，只不过我们不再关心。

"返回"

CPU会取出那个返回地址，也就是100，去这里找指令接着执行

printf("the sum is %d\n",sum);

而sum的值就存在eax寄存器里。

> eax 是累加器(accumulator)，它是很多加法乘法指令的缺省寄存器。
>
> ebx 是基地址(base)*寄存器*，在内存寻址时存放基地址。
>
> ecx 是计数器(counter)，是重复(REP)前缀指令和LOOP指令的内定计数器。
>
> edx 总是被用来放整数除法产生的余数。

#### 总结

函数调用，关键是：

1.把参数和返回地址准备好

2.然后各自都遵循约定，每次新函数都建立新的函数帧，即上文提到的`标准的函数起始代码`

3.函数调用完，重置ebp和esp，让他们重新指向调用者的函数帧。



参考文章：[函数调用的秘密](https://mp.weixin.qq.com/s?__biz=MzAxOTc0NzExNg==&mid=2665513039&idx=1&sn=381c1b8c7f86906c4838050b8c1db2bb&scene=21#wechat_redirect)