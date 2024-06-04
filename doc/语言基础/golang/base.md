<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [go的网络IO为什么那么快？](#go%E7%9A%84%E7%BD%91%E7%BB%9Cio%E4%B8%BA%E4%BB%80%E4%B9%88%E9%82%A3%E4%B9%88%E5%BF%AB)
- [你觉得go的网络IO还有什么优化空间么？](#%E4%BD%A0%E8%A7%89%E5%BE%97go%E7%9A%84%E7%BD%91%E7%BB%9Cio%E8%BF%98%E6%9C%89%E4%BB%80%E4%B9%88%E4%BC%98%E5%8C%96%E7%A9%BA%E9%97%B4%E4%B9%88)
- [epoll为什么那么快？](#epoll%E4%B8%BA%E4%BB%80%E4%B9%88%E9%82%A3%E4%B9%88%E5%BF%AB)
- [照你这么说，select和poll已经没有什么用了吗？](#%E7%85%A7%E4%BD%A0%E8%BF%99%E4%B9%88%E8%AF%B4select%E5%92%8Cpoll%E5%B7%B2%E7%BB%8F%E6%B2%A1%E6%9C%89%E4%BB%80%E4%B9%88%E7%94%A8%E4%BA%86%E5%90%97)
- [那你认为epoll还有什么优化空间么？](#%E9%82%A3%E4%BD%A0%E8%AE%A4%E4%B8%BAepoll%E8%BF%98%E6%9C%89%E4%BB%80%E4%B9%88%E4%BC%98%E5%8C%96%E7%A9%BA%E9%97%B4%E4%B9%88)
- [如果要你实现一个网络IO应该怎么实现？](#%E5%A6%82%E6%9E%9C%E8%A6%81%E4%BD%A0%E5%AE%9E%E7%8E%B0%E4%B8%80%E4%B8%AA%E7%BD%91%E7%BB%9Cio%E5%BA%94%E8%AF%A5%E6%80%8E%E4%B9%88%E5%AE%9E%E7%8E%B0)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

#### go的网络IO为什么那么快？

我认为go的网络IO那么快，需要从底层实现原理来分析。go语言的网络IO底层实现包括了：go的并发机制和网络轮询器（netpoller）机制。

其中网络轮询器就是IO多路复用，这种机制允许Go的网络库在**非阻塞方式**下高效地处理多个网络I/O操作。这种机制最常见的实现是**Linux上的epoll**，以及在**Windows上的IOCP**。

go的并发机制，实现的理论依据是一个协程对应一个网络连接。

在goroutine-per-connection模式中，每当一个新的网络连接建立，Go会启动一个新的goroutine来处理这个连接的所有I/O操作和相关逻辑。由于goroutines的调度开销非常小，可以在短时间内处理大量请求。

#### 你觉得go的网络IO还有什么优化空间么？

从之前说明的网络IO底层实现原理中，一个协程对应一个网络连接中，在大量网络IO中也会产生很多协程的，这里可以通过优化goroutine的数量进行优化，比如通过携程池来减少资源的消耗。

同时我之前还了解一个开源框架gnet，xxxx

#### epoll为什么那么快？

我认为这里说的快，主要相较于另外两个实现网络连接的IO多路复用技术select和poll来说的，可以通过他们的底层实现来说明epoll为什么那么快。

1）epoll的e代表的是事件，也就是epoll是基于事件驱动的，这个事件是指活跃的网格链接事件，比select和poll俩说，避免了轮询所有文件描述符。

2）无需每次都传输整个被监听的文件描述符列表：在select/poll中，每次调用都需要将全部的文件描述符从用户空间拷贝到内核空间，而epoll只需要将变化的文件描述符传递给内核即可。

#### 照你这么说，select和poll已经没有什么用了吗？

我觉得这个需要区分场景的，如果连接数量较少的场景下，select和poll可能是更简单的选择，但是需要处理大量的并发连接场景，epoll会表现得更好。

#### 那你认为epoll还有什么优化空间么？

针对epoll的优化空间，我个人理解是在一些场景下，它的使用策略调整：

1）减少epoll_wait的调用：epoll_wait 是阻塞等待事件发生的函数，频繁调用会增加上下文切换的开销。可以通过适合的超时设置，或者确认有数据待处理的时候调用来减少调用频率。

2）根据实际的应用场景设置水平触发(EPOLLT)和边缘触发(EPOLLET)：比如在处理请求不是特别密集，且处理速度相对较快的场景下，水平触发可能是一个更好的选择。而在请求非常密集，处理速度较慢的情况下，边缘触发可能会表现得更好。

> 1. **水平触发 (Level Triggered)**：进程被通知到有 I/O 事件发生，并且提供给它处理这个事件的机会。如果进程没有完全处理完这个事件，内核会再次通知它，因此事件会一直持续通知到已经完全被处理完。这就像是一个开关，只要打开，就会一直通知状态的改变。
> 
> 2. **边缘触发 (Edge Triggered)**：与水平触发不同，边缘触发只会告知进程 I/O 事件的那一刻发生了状态变化，即只有状态改变的“边缘”会被通知。如果事件没有被处理完，内核将不会再次通知。在这种模式下，系统通知的是“事件”，而不是“状态”。

#### 如果要你实现一个网络IO应该怎么实现？
