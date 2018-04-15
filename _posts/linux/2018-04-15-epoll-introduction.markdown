---
layout: post
title:  "epoll详解"
date:   2018-04-15 20:32:41 +0800
categories: linux
---
## epoll使用方法
用epoll_create或epoll_create1创建epoll instance。与epoll_create不同的是，epoll_create1支持一个flag参数。

{% highlight c %}
    int epfd = epoll_create(1);                 // 参数会被忽略，但必须大于0
    int epfd = epoll_create1(EPOLL_CLOEXEC);    // 给epfd设置FD_CLOEXEC标志位
{% endhighlight %}

用epoll_ctl注册要监视的fd：

{% highlight c %}
    struct epoll_event ev;
    ev.events = EPOLLIN | EPOLLET;      // fd可读时通知，启用边缘触发(edge-trigger)
    ev.data.fd = server_socket;
    epoll_ctl(epfd, EPOLL_CTL_ADD, server_socket, &ev);
{% endhighlight %}

用epoll_wait等待事件。函数原型：int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout)。其中，若timeout为-1，表示永远等下去。

{% highlight c %}
    int nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);
    for (int i=0; i < nfds; ++i) {
        // read from events[i].data.fd
    }
{% endhighlight %}

## epoll的原理
### select和poll的原理
select和poll产生一个fd的数组，包含它们感兴趣的fd，然后发出系统调用，将它们交给内核。内核将fd数组从用户态复制到内核态，然后依次检查每一个fd。最后，select会生成一个bit数组(默认长度FD_SETSIZE被hardcode为1024)，每个bit对应一个fd，表示该fd是否有事件发生；poll则直接操作用户态的pollfd结构。两者的区别是，select只能监视有限数量的fd(FD_SETSIZE被hardcode为1024)。而poll直接操作用户提供的pollfd数组，没有这个限制。

可见这两个函数的时间复杂度都是O(n)。在服务器只有数千个TCP连接的年代，它们是够用的。但假如服务器有10万个连接，那很多CPU时间会浪费在挨个检查每个fd上。为了解决这个问题，内核引入了epoll。

### epoll概述
epoll与select/poll的最大区别是，epoll没有每次都把大量的fd传递给内核，而是在内核中获取一个epoll instance，并把fd注册到它上面。epoll没有对整个fd数组进行poll，而是epoll instance监视注册的fd，并将事件“报告”给用户程序。

当“有事件发生的fd”与“所有监视的fd”的比例较小时，这个机制非常有效。

### epoll_create代码分析
epoll instance是epoll子系统的核心。在linux下，我们用epoll_create()和epoll_create1()来获取一个epoll_instance，两个函数都返回一个fd。之所以把epoll也弄成fd，是因为这样做epoll instance自己也是pollable的，我们可以对epoll fd再进行epoll/select/poll。

epoll instance最重要的部分是struct eventpoll，在fs/eventpoll.c中定义。我们来看系统调用epoll_create1的部分代码：SYSCALL_DEFINE1(epoll_create1, int, flags)。首先，用ep_alloc分配一个struct eventpoll的空间：

{% highlight c %}
    struct eventpoll *ep = NULL;
    // Create the internal data structure ("struct eventpoll")
    error = ep_alloc(&ep);
{% endhighlight %}

之后，获取进程未使用的fd：

{% highlight c %}
    fd = get_unused_fd_flags(O_RDWR | (flags & O_CLOEXEC));
{% endhighlight %}

从系统获取anonymous inode，并保存到之前的struct eventpoll结构体中。

{% highlight c %}
    file = anon_inode_getfile("[eventpoll]", &eventpoll_fops, ep,
                              O_RDWR | (flags & O_CLOEXEC));
    ep->file = file;
{% endhighlight %}

将inode和fd bind起来，并将fd返回：

{% highlight c %}
    fd_install(fd, file);
    return fd;
{% endhighlight %}

### epoll instance如何保存要监视的fd？
epoll instance使用红黑树来保存要监视的fd，红黑树的root就是struct eventpoll中的rbr成员，是在ep_alloc()中初始化的。

对于每个要监视的fd，红黑树中都有一个struct epitem与之对应。我们来看系统调用epoll_ctl的代码：SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd, struct epoll_event __user *, event)

用ep_find来找到fd对应的epitem：

{% highlight c %}
    epi = ep_find(ep, tf.file, fd);
{% endhighlight %}

找到epitem后，如果epoll_ctl的op参数为EPOLL_CTL_ADD，则调用ep_insert将其添加；如果是EPOLL_CTL_DEL，则调用ep_remove将其移除；如果是EPOLL_CTL_MOD，则调用ep_modify将其修改。

在红黑树结点中，作为key的是struct epitem中的struct epoll_filefd，其定义如下：

{% highlight c %}
struct epoll_filefd {
    struct file *file;
    int fd;
} __packed;
{% endhighlight %}

执行key compare的是ep_cmp_ffd()函数，其定义如下：

{% highlight c %}
/* Compare RB tree keys */
static inline int ep_cmp_ffd(struct epoll_filefd *p1, 
                             struct epoll_filefd *p2) 
{
    return (p1->file > p2->file ? +1:
            (p1->file < p2->file ? -1 : p1->fd - p2->fd));
}
{% endhighlight %}

## 相关链接
* 《The Implementation of epoll (1)》 https://idndx.com/2014/09/01/the-implementation-of-epoll-1/
