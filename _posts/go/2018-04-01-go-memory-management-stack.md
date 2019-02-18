---
layout: post
title: "Go内存管理 - Stack"
date: 2018-04-01 16:28:00 +0800
categories: go
---
## Stack
### 传统的Stack
#### 普通进程内存布局：

[!内存布局](images/linux-memory-layout.jpg)

user space stack的大小是固定的，不能动态扩展，linux中默认为8192KB。若程序运行时，栈的内存占用超过上限，程序会出现segment fault错误。

#### 每个thread都有自己的stack
每个thread都有自己的stack。在linux上，thread是由clone系统调用创建的(根据参数不同，也可以创建子进程)，其函数原型为：

{% highlight c %}
int clone(int (*fn)(void *), void *child_stack,
          int flags, void *arg, ...
          /* pid_t *ptid, void *newtls, pid_t *ctid */ );
{% endhighlight %}

可以看到，创建thread时，需要提供一个child_stack。

#### 引发的问题
如果程序大量使用stack空间(比如大量递归调用)，stack的空间会被耗尽。这就导致了一个难题：如果栈设置得太小，有可能会被耗尽，导致segment fault；如果栈设置得太大，即使程序并未使用栈，也会占用很多内存。虽然可以在clone()时设定每个thread栈的大小，但这需要精确计算每个thread所需栈的大小，比较困难。


### Go中的“Stack”
#### 每个goroutine都有自己的stack
Go的每个goroutine都有自己的“stack”。与thread不同，goroutine的stack被设计为初始大小很小(只有2KB)，但可以在需要的时候扩展。见runtime/runtime2.go对goroutine的定义:

{% highlight go %}
type g struct {
        // Stack parameters.
        // stack describes the actual stack memory: [stack.lo, stack.hi).
        // stackguard0 is the stack pointer compared in the Go stack growth prologue.
        // It is stack.lo+StackGuard normally, but can be StackPreempt to trigger a preemption.
        // stackguard1 is the stack pointer compared in the C stack growth prologue.
        // It is stack.lo+StackGuard on g0 and gsignal stacks.
        // It is ~0 on other goroutine stacks, to trigger a call to morestackc (and crash).
        stack       stack   // offset known to runtime/cgo
        stackguard0 uintptr // offset known to liblink
        stackguard1 uintptr // offset known to liblink
	...
}
{% endhighlight %}

#### goroutine stack的布局
由于linux提供的thread stack并不能动态扩展，因此go runtime自己实现了一套stack，goroutine的stack实际上是位于.heap段的。以下是goroutine stack的布局：

[!goroutine stack layout](images/goroutine-stack-layout.jpg)

* stack.lo: 栈空间的低地址
* stack.hi: 栈空间的高地址
* stackguard0: stack.lo + StackGuard, 用于stack overlow的检测
* StackGuard: 保护区大小，常量Linux上为880字节
* StackSmall: 常量大小为128字节，用于小函数调用的优化

## 相关链接
* 聊一聊goroutine stack：https://zhuanlan.zhihu.com/p/28409657
