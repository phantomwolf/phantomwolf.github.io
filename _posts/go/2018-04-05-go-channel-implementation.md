---
layout: post
title: "Go channel实现"
date: 2018-04-05 14:15:03 +0800
categories: go
---
## 概述
以下是channel的结构体，在runtime/chan.go中定义：

{% highlight go %}
type hchan struct {
        qcount   uint           // buffer中的元素数量
        dataqsiz uint           // size of the circular queue(buffer的总大小)
        buf      unsafe.Pointer // points to an array of dataqsiz elements(指向buffer的指针)
        closed   uint32         // 为1时表示channel关闭
        recvq    waitq  // list of recv waiters，等待从channel接收数据的goroutine们
        sendq    waitq  // list of send waiters，等待向channel发送数据的goroutine们
        lock mutex      // 锁
        ...
}
{% endhighlight %}

[!channel structure](go-channel-structure.png)

一般来说，go channel在做操作之前，会先获取锁，结束以后再解锁。不过有时做一些non-blocking操作时，会不获取锁就检查，以提高效率。

channel可以是同步的(无缓存)，也可以是异步的(有缓存)。

## 同步channel
用以下代码创建一个同步的channel并从channel读取数据：
{% highlight go %}
func main() {
    ch := make(chan bool)
    go func() {
        ch <- true
    }()
    <-ch
}
{% endhighlight %}

刚创建的时候，channel的情况如下图。Go不会为同步的channel分配buffer，因此buffer的指针为nil，qcount和dataqsiz都是0。

[!new channel](go-channel-new.png)

两个goroutine的运行次序是未知的，我们假定main函数先对goroutine进行读取。goroutine对channel进行读操作时，会先进行一些检查：channel是否已关闭，是否有缓存，sendq中是否有goroutine等待发送数据。在本例中，当main函数中对channel进行读取时，channel没有buffer，sendq中也没有goroutine等待发送数据。这种情况下，goroutine会把自己加到channel的recvq中，并阻塞：

[!channel read blocks](channel-read-block.png)

细节：该goroutine会使用acquireSudog()获取一个sudog结构，并将自己的waiting设置为它

{% highlight go %}
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	...
        // no sender available: block on this channel.
        gp := getg()				// 获取当前goroutine
        mysg := acquireSudog()			// 获取sudog
        mysg.releasetime = 0 
        if t0 != 0 { 
                mysg.releasetime = -1
        }
        // No stack splits between assigning elem and enqueuing mysg
        // on gp.waiting where copystack can find it.
        mysg.elem = ep
        mysg.waitlink = nil 
        gp.waiting = mysg
        mysg.g = gp
        mysg.isSelect = false
        mysg.c = c 
        gp.param = nil 
        c.recvq.enqueue(mysg)
        goparkunlock(&c.lock, "chan receive", traceEvGoBlockRecv, 3)
	...
}
{% endhighlight %}

关于sudog的解释：sudog represents a g in a wait list, such as for sending/receiving on a channel. sudog is necessary because the g ↔ synchronization object relation is many-to-many. A g can be on many wait lists, so there may be many sudogs for one g; and many gs may be waiting on the same synchronization object, so there may be many sudogs for one object. sudogs are allocated from a special pool. Use acquireSudog and releaseSudog to allocate and free them.

一段时间后，另一个goroutine向channel发送数据，它会做类似的检查，发现recvq中有个goroutine正在等待，于是将数据写入其stack中，并唤醒它。这是唯一一个go runtime会向另一个goroutine的stack内写入数据的地方。此后，两个goroutine都将返回。

被唤醒的goroutine会继续执行chanrecv函数的剩余部分：

{% highlight go %}
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	...
        // someone woke us up
        if mysg != gp.waiting {
                throw("G waiting list is corrupted")
        }
        gp.waiting = nil
        if mysg.releasetime > 0 {
                blockevent(mysg.releasetime-t0, 2)
        }
        closed := gp.param == nil
        gp.param = nil
        mysg.c = nil
        releaseSudog(mysg)
        return true, !closed
}
{% endhighlight %}

## 异步channel(带缓存的channel)
请看以下例子：

{% highlight go %}
func main() {
    ch := make(chan bool, 1)
    ch <- true
    go func() {
        <-ch 
    }()
    ch <- true
 }
{% endhighlight %}

两个goroutine的运行次序依然是未知的，我们假定两个元素先被发送到channel中，之后另一个goroutine从channel中读取。channel被创建后如下图。与同步channel的区别是，分配了一个buffer，且dataqsiz为1。

[!buffered channel](channel-buffered.png)

接下来，main goroutine向channel写入第一个元素。首先做如下检查：检查recvq是否为空；若buffer为空，检查buffer中是否有空闲空间。在本例中，recvq中没有goroutine，buffer仍然有1个空位，因此向buffer写入数据，并返回。此时，channel的情况如下图：

[!buffered channel with data](channel-buffered-with-data.png)

下一步，main goroutine向channel发送第二个元素，此时buffer已满。当buffer已满时，有缓存的channel与无缓存的行为一致，main goroutine将自己放入sendq中并阻塞，如下图：

[!sendq](channel-buffered-sendq.png)

最后，另一个goroutine从channel中接收数据。这时问题来了，Go保证channel以FIFO的方式工作，所以它不能从buffer获取数据后就继续执行，否则main goroutine将永远阻塞。为了处理这个问题，它会在从buffer读取数据后，将sendq中的第一个goroutine要写的数据加入到buffer中，并将其从sendq中移除并唤醒。

## Select
我们知道，Go支持用select语句操作channel：

{% highlight go %}
select {
case <-ch:
    foo()
default:
    bar()
}
{% endhighlight %}

按照我们之前的分析，如果channel无缓存，或者有缓存但是里面没有元素，那么试图从channel读取数据的goroutine将会阻塞。那还怎么执行select语句的default情况呢？我们来看一下从channel接收数据的函数：

{% highlight go %}
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool)
{% endhighlight %}

其中，c指向channel的struct；ep指向一块内存，从channel中读出的数据应该写到这里；若block == false，将使用非阻塞模式。非阻塞模式下，goroutine检查buffer，以及sendq，如果读到了元素，就写入ep中，如果没有，就立刻返回(false, false)。buffer和queue的检查使用了原子操作，而没有使用锁。

## 关闭channel
关闭channel时，go遍历channel的队列中所有发送者和接受者，并解锁它们。所有接收者会得到该类型的默认值，所有发送者会panic。

## 相关链接
* Golang: channels implementation：http://dmitryvorobev.blogspot.hk/2016/08/golang-channels-implementation.html
