---
layout: post
title: "Go netpoller"
date: 2018-03-31 16:45:00 +0800
categories: go
---
## Go的并发模型
Go鼓励在goroutine中使用同步的编程方法，并创建多个goroutine，让它们之间通过channel通信。这种同步的编程方法大大方便了我们编写程序，不会出现callback满天飞的情况。

在之前的Go调度器的介绍中，我们已经知道，当在G(goroutine)中调用syscall时，不仅G会阻塞，相应的M(Thread)也会阻塞。这时go runtime会唤醒sleep状态的M，或创建新的M，来运行其他的G。这样就导致了一个问题：如果频繁调用会阻塞的syscall(例如对socket的fd执行阻塞的read)，go runtime就会不断创建新的M(Thread)，大量的Thread会耗光资源。

Go使用netpoller来避免这一问题。

## netpoller
netpoller待在它自己的Thread中，从希望做Network I/O的goroutine中接收事件。

当你在Go中accept一个连接时，相应的fd会被设置O_NONBLOCK flag。若goroutine在读写该fd时遇到了EAGAIN，就会调用netpoller的wait方法，令netpoller在fd可读/写后通知该goroutine。之后，goroutine就会进入等待状态，不再参与调度。netpoller会用系统上的非阻塞网络I/O来监视该fd，在linux上是epoll，在BSD上是kqueue。

当netpoller从操作系统收到通知说该fd可以读/写时，会检查是否有goroutine被阻塞在该fd上。如果有，就通知它们。这样goroutine就可以重新执行之前的读/写操作。

所以实际上go使用的是非阻塞网络I/O，但是用了一套机制，令goroutine保持睡眠，直到socket变为可以读/写后，重新进行读/写。将异步变成了同步，简化了编程。

## 相关代码
net/fd_unix.go:
{% highlight go %}
type netFD struct {
        pfd poll.FD

        // immutable until Close
        family      int
        sotype      int
        isConnected bool
        net         string
        laddr       Addr
        raddr       Addr
}

func (fd *netFD) Read(p []byte) (n int, err error) {
    	n, err = fd.pfd.Read(p)
    	runtime.KeepAlive(fd)
    	return n, wrapSyscallError("read", err)
}
{% endhighlight %}

internal/poll/fd_unix.go:

{% highlight go %}
func (fd *FD) Read(p []byte) (int, error) {
        ...
        for {
                n, err := syscall.Read(fd.Sysfd, p)
                if err != nil {
                        n = 0 
                        if err == syscall.EAGAIN && fd.pd.pollable() {
                                if err = fd.pd.waitRead(fd.isFile); err == nil {
                                        continue
                                }
                        }
                        ...
                }
                err = fd.eofError(n, err)
                return n, err 
        }
}
{% endhighlight %}

internal/poll/fd_poll_runtime.go:

{% highlight go %}
func (pd *pollDesc) waitRead(isFile bool) error {
        return pd.wait('r', isFile)
}

func (pd *pollDesc) wait(mode int, isFile bool) error {
        if pd.runtimeCtx == 0 { 
                return errors.New("waiting for unsupported file type")
        }
        res := runtime_pollWait(pd.runtimeCtx, mode)
        return convertErr(res, isFile)
}
{% endhighlight %}
