**他山之石可以攻玉**

analogous_love的文章写的很好，此处引用下。  
https://blog.csdn.net/analogous_love/article/details/53033793  
服务器端为了能流畅处理多个客户端链接，一般在某个线程A里面accept新的客户端连接并生成新连接的socket fd，然后将这些新连接的socketfd给另外开的数个工作线程B1、B2、B3、B4，这些工作线程处理这些新连接上的网络IO事件（即收发数据），同时，还处理系统中的另外一些事务。这里我们将线程A称为主线程，B1、B2、B3、B4等称为工作线程。 
```c++
while (!m_bQuit)
{  
    epoll_or_select_func();
 
    handle_io_events();
 
    handle_other_things();  //服务器没有检测到新的连接，就安排线程去做其他事情
}
```
 在上述while循环里面，epoll_or_selec_func()中的epoll_wait/poll/select等函数一般设置了一个超时时间。如果设置超时时间为0，那么在没有任何网络IO时间和其他任务处理的情况下，这些工作线程实际上会空转，白白地浪费cpu时间片。如果设置的超时时间大于0或者设置为无限等待，在没有网络IO时间的情况，epoll_wait/poll/select仍然要挂起指定时间才能返回，导致handle_other_thing()不能及时执行，影响其他任务不能及时处理，也就是说其他任务一旦产生，其处理起来具有一定的延时性。这样也不好。那如何解决该问题呢？

 其实我们想达到的效果是，如果没有网络IO时间和其他任务要处理，那么这些工作线程最好直接挂起而不是空转；如果有其他任务要处理，这些工作线程要立刻能处理这些任务而不是在epoll_wait/poll/selec挂起指定时间后才开始处理这些任务。

 我们采取如下方法来解决该问题，以linux为例，不管epoll_fd上有没有文件描述符fd，我们都给它绑定一个默认的fd，这个fd被称为唤醒fd。当我们需要处理其他任务的时候，向这个唤醒fd上随便写入1个字节的，这样这个fd立即就变成可读的了，epoll_wait()/poll()/select()函数立即被唤醒，并返回，接下来马上就能执行handle_other_thing()，其他任务得到处理。反之，没有其他任务也没有网络IO事件时，epoll_or_select_func()就挂在那里什么也不做。

这个唤醒fd，在linux平台上可以通过以下几种方法实现：

1. 管道pipe，创建一个管道，将管道绑定到epoll_fd上。需要时，向管道一端写入一个字节，工作线程立即被唤醒。

2. linux 2.6新增的eventfd：

int eventfd(unsigned int initval, int flags);  
步骤也是一样，将生成的eventfd绑定到epoll_fd上。需要时，向这个eventfd上写入一个字节，工作线程立即被唤醒。

3. 第三种方法最方便。即linux特有的socketpair，socketpair是一对相互连接的socket，相当于服务器端和客户端的两个端点，每一端都可以读写数据。

int socketpair(int domain, int type, int protocol, int sv[2]);

调用这个函数返回的两个socket句柄就是sv[0]，和sv[1]，在一个其中任何一个写入字节，在另外一个收取字节。

将收取的字节的socket绑定到epoll_fd上。需要时，向另外一个写入的socket上写入一个字节，工作线程立即被唤醒。

如果是使用socketpair，那么domain参数一定要设置成AFX_UNIX。

由于在windows，select函数只支持检测socket这一种fd，所以windows上一般只能用方法3的原理。而且需要手动创建两个socket，然后一个连接另外一个，将读取的那一段绑定到select的fd上去。这在写跨两个平台代码时，需要注意的地方。


作为服务器端程序最好对侦听socket调用setsocketopt()设置SO_REUSEADDR和SO_REUSEPORT两个标志，因为服务程序有时候会需要重启（比如调试的时候就会不断重启），如果不设置这两个标志的话，绑定端口时就会调用失败。因为一个端口使用后，即使不再使用，因为四次挥手该端口处于TIME_WAIT状态，有大约2min的MSL（Maximum Segment Lifetime，最大存活期）。这2min内，该端口是不能被重复使用的。你的服务器程序上次使用了这个端口号，接着重启，因为这个缘故，你再次绑定这个端口就会失败（bind函数调用失败）。要不你就每次重启时需要等待2min后再试（这在频繁重启程序调试是难以接收的），或者设置这种SO_REUSEADDR和SO_REUSEPORT立即回收端口使用。

其实，SO_REUSEADDR在windows上和Unix平台上还有些细微的区别，我在libevent源码中看到这样的描述：

在Unix平台上设置这个选项意味着，任意进程可以复用该地址；而在windows，不要阻止其他进程复用该地址。也就是在在Unix平台上，如果不设置这个选项，任意进程在一定时间内，不能bind该地址；在windows平台上，在一定时间内，其他进程不能bind该地址，而本进程却可以再次bind该地址。

问题讨论  
epoll_wait对新连接socket使用的是边缘触发模式EPOLLET（edge trigger），而不是默认的水平触发模式（level trigger)。因为如果采取水平触发模式的话，主线程检测到某个客户端socket数据可读时，通知工作线程去收取该socket上的数据，这个时候主线程继续循环，只要在工作线程没有将该socket上数据全部收完，或者在工作线程收取数据的过程中，客户端有新数据到来，主线程会继续发通知（通过pthread_cond_signal()）函数，再次通知工作线程收取数据。这样会可能导致多个工作线程同时调用recv函数收取该客户端socket上的数据，这样产生的结果将会导致数据错乱。
相反，采取边缘触发模式，只有等某个工作线程将那个客户端socket上数据全部收取完毕，主线程的epoll_wait才可能会再次触发来通知工作线程继续收取那个客户端socket新来的数据。  
以上说法存在问题，epoll无论是工作在水平触发方式下还是工作在边缘触发方式下都存在重复触发的问题，都可以通过设置EPOLLONESHOT来解决重复触发的问题

