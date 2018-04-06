---
layout: post
title: "LVS的调度算法"
date: 2018-04-06 17:20:00 +0800
categories: system-design
---
LVS有以下调度算法：

* 轮流调度Round-Robin Scheduling
* 带权重的轮流调度Weighted Round-Robin Scheduling
* Least-Connection Scheduling
* Weighted Least-Connection Scheduling
* Locality-Based Least-Connection Scheduling
* Locality-Based Least-Connection with Replication Scheduling
* Destination Hashing Scheduling
* Source Hashing Scheduling
* Shortest Expected Delay Scheduling
* Never Queue Scheduling

## 轮流调度
每次都把请求发给服务器列表中的下一个。在这个算法中，所有服务器都将被视为平等的。

与轮流DNS(round-robin DNS，每次都将域名解析为列表中的下一个IP地址)相似，但由于DNS缓存的存在，轮流DNS可能会引起流量的不平衡。

## 带权重的轮流调度
本算法为更好的处理拥有不同性能的服务器而设计。根据处理能力，每个服务器都将被分到一个权重。较高权重的服务器会先收到新请求，最终也会收到更多的请求。相同权重的服务器收到相等数量的请求。例如有3个服务器A、B、C，分别拥有权重4、3、2，一个良好的调度顺序是：AABABCABC。

当服务器的处理能力不同时，带权重的轮流调度算法比普通的轮流调度要好。但是，如果请求的负载差别很大，仍然有可能导致不平衡。

以下是本算法的一个实现的伪代码：

{% highlight c %}
/*Supposing that there is a server set S = {S0, S1, …, Sn-1};
W(Si) indicates the weight of Si;
i indicates the server selected last time, and i is initialized with -1;
cw is the current weight in scheduling, and cw is initialized with zero; 
max(S) is the maximum weight of all the servers in S;
gcd(S) is the greatest common divisor of all server weights in S;*/

while (true) {
    i = (i + 1) mod n;
    if (i == 0) {
        cw = cw - gcd(S); 
        if (cw <= 0) {
            cw = max(S);
            if (cw == 0)
                return NULL;
        }
    } 
    if (W(Si) >= cw) 
        return Si;
}
{% endhighlight %}

解释：

1. 设置一个变量cw(current weight)表示本轮遍历可以接受的最低权重。
2. 遍历所有服务器，若其权重大于等于cw，就将其返回；若小于cw，则查看下一个服务器。
3. 扫描完整个S数组后，i会变成0。cw减去所有权重的最大公约数(gcd)，再开始下一轮遍历。若cw小于等于0，则将其重置为最大的权重。

对于权重分别为4、3、2的服务器A、B、C来说，第一轮遍历，i为0，cw为0，但会被置为最大权重4，表示本轮接受所有权重大于等于4的服务器。于是第一轮遍历，只有A符合要求。第二轮遍历，cw减去所有权重的最大公约数1，变为3，表示本轮接受权重大于等于3的服务器，A、B均符合要求。第三轮，cw变为2，A、B、C均符合要求。第四轮，cw变为1，A、B、C均符合要求。所有总的调度序列为：AABABCABC。

第五轮，cw变为-1，被重置为最大权重4，调度继续进行。

为什么每轮可接受的最小权重cw要减去最大公约数gcd，而不是每次都固定减去1呢？思考权重分别为4、2的服务器A、B。如果cw每次都固定减去1，调度序列会是：AAABAB。如果cw每次减去最大公约数2，调度序列会是：AABAAB。可见，前者对请求的分配更为密集，后者更好一些。还有一个方法也可以解决前者的问题，可以把所有权重都除以其最大公约数，那么A、B的权重变为2、1，就不会有之前的问题了。

## Least-Connection Scheduling
将请求发给目前连接数最少的服务器。属于动态调度算法，需要计算服务器实时的连接数。本算法能很好处理请求的负载差别很大的情况。

## Weighted Least-Connection Scheduling
与上述算法相似，只不过可以给服务器设置权重。当下列公式被满足时，请求会被发送给第j个服务器：(Cj/ALL_CONNECTIONS)/Wj = min { (Ci/ALL_CONNECTIONS)/Wi } (i=1,..,n)

由于ALL_CONNECTIONS在一次查询中是固定的，所以可以优化为：Cj/Wj = min { Ci/Wi } (i=1,..,n)

## Locality-Based Least-Connection Scheduling
将用户的IP map到固定的服务器(比如根据用户的位置，map到最近的服务器)。若该服务器已经过载(连接数大于其权重)，且有其他服务器的负载尚未过半，就按Least-Connection Scheduling来处理。

## Locality-Based Least-Connection with Replication Scheduling
将用户的IP map到一系列服务器，在这些服务器中实行Least-Connection Scheduling。如果这些服务器都过载了，再从整个集群里找连接数最少的服务器。

## Destination Hashing Scheduling
根据用户的IP，将请求发送到最近的服务器上。

## Source Hashing Scheduling
The source hashing scheduling algorithm assigns network connections to the servers through looking up a statically assigned hash table by their source IP addresses.

## Shortest Expected Delay Scheduling
将请求发给预期延迟最低的服务器。第i个服务器的预期延迟的计算公式为(Ci+1)/Ui，Ci是该服务器上的连接数量，Ui是权重。

## Never Queue Scheduling
当有空闲服务器存在时，直接发给空闲服务器；当没有空闲服务器时，按Shortest Expected Delay Scheduling处理。
