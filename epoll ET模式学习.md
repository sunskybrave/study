前面学习epoll的水平触发模式，实现了epoll+阻塞式I/O+线程池，现在进一步改进，使用epoll的边缘触发模式，实现epoll+非阻塞式I/O+线程池  
首先确认下需要注意的问题及解决方法  
1.边缘触发方式下的重复触发问题  
这里的重复触发是指对于同一个事件的重复触发，在水平触发模式的学习中已经讨论过此问题，那么对于边缘触发方式是否存在此问题呢？  
查找资料，一般来说大家都有以下的认识：  
*epoll_wait对如果采取水平触发模式的话，主线程检测到某个客户端socket数据可读时，通知工作线程去收取该socket上的数据，这个时候主线程继续循环，只要在工作线程没有将该socket上数据全部收完，或者在工作线程收取数据的过程中，客户端有新数据到来，主线程会继续发通知（通过pthread_cond_signal()）函数，再次通知工作线程收取数据。这样会可能导致多个工作线程同时调用recv函数收取该客户端socket上的数据，这样产生的结果将会导致数据错乱。采取边缘触发模式，只有等某个工作线程将那个客户端socket上数据全部收取完毕，主线程的epoll_wait才可能会再次触发来通知工作线程继续收取那个客户端socket新来的数据。*  
显然忽略了一种情况，即epoll工作在边缘触发方式下，当主线程检测到套接字A上有数据可读，则触发事件，分配一个工作线程thread1去处理套接字A，当工作线程在处理套接字A时此时客户端又发送了新的数据到套接字A，此时主线程检测到套接字A上有新的数据到来，再次触发事件，分配一个工作线程thread2去处理套接字A，此时显然会出现问题，不可以让多个线程去处理同一个套接字。（关于epoll在边缘触发方式下的触发条件可以参见《epoll学习》一文）。  
2.监听套接字的非阻塞设置
最好设置监听套接字为非阻塞套接字，因为服务器繁忙时不会立即调用accept，而此时客户可能会终止此连接并发送RST，此时服务器收到客户的RST,随后从已连接队列中移除这个连接，此后再调用accept，此时如果没有新的连接到来，线程会阻塞在accept的调用上

```c++
extern "C" {
  #include <stdio.h>
  #include <stdlib.h>
  #include "unpthread.h"
}

#include <sys/epoll.h>
#include <iostream>
#include <queue>
using namespace std;

typedef struct {
  pthread_t		thread_tid;		/* 工作线程 ID */
  long			thread_count;	/* # 工作线程完成任务的计数器 */
} Thread;
Thread	*tptr;		/* 工作线程数组首地址 */

int		  *iptr,nwork; //nwork为当前的任务个数
queue<int> work_queue; //分配给工作线程的任务，分为读任务和写任务，每次分配分配任务先压入描述符号，再压入任务类别，0为读，1为写
queue<int> socket_close_queue; //需要关闭的套接字

static int			nthreads;
pthread_mutex_t		clifd_mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t		clifd_cond = PTHREAD_COND_INITIALIZER;

int epfd,nfds; //epoll描述符，主线程和工作线程都只读，不用加锁

pthread_key_t r1_key;
pthread_once_t r1_once = PTHREAD_ONCE_INIT;

#define	MAXN	16384		/* max # bytes client can request */
void web_child(int sockfd,int work_mode)
{

	char		line[MAXLINE], result[MAXN];

	if(!work_mode)
	{
		//对于read，反复读取直到读到EOF或者EAGAIN
		ssize_t n=0,nread;
		while(1)
		{
			nread = read(sockfd,line+n,MAXLINE-1-n); //显然MAXLINE足够大，当调用read进行读取时可能一次无法读取完，反复读取直至返还EAGAIN，考虑到一个特殊的问题，就是接受缓冲区一直有新的数据在进来，此时可能会出现数据过多的问题，如果此时设定的读的值比line数组的剩余空间大，则会有数据丢失
			if(nread>0)
			{
				n+=nread;  //注意记录line数组中的位置
				if( MAXLINE-1-n > 0) //即当前line数组还没有写满，可以继续读
				{
					continue;
				}
				else if( MAXLINE-1-n == 0) //即line数组已经写满了，此时可以选择将line中的数据保存下后继续读数据或者直接跳出，在epoll中注册此套接字的可读，下次再读取
				{
					//必须要特殊处理此情况，如果设置要读取0个数据，也会返回0，会被误认为接受到EOF
				   //A value of zero indicates end-of-file (except if the value of the size argument is also zero).
					break;
				}
				else //显然此情况不会出现，不用考虑
				{

				}
			}
			else if(nread == 0 ) //读取到EOF
			{
				break;//此处只是简单处理，实际上可能要额外处理EOF
			}
			else if(errno == EINTR) //被中断打断，即使是非阻塞I/O仍然可能会被EINTR打断
			{
				continue;
			}
			else if(errno == EAGAIN)  //缓冲区数据读取完，直接跳出
			{
				break;
			}
			else //read出错
			{
				err_quit("read error");
			}
		}

		struct epoll_event ev_temp;
		ev_temp.data.fd=sockfd; //epoll是线程安全的，在主线程和工作线程对其进行操作
		ev_temp.events=EPOLLOUT | EPOLLONESHOT; //使用默认水平触发方式，注意水平触发下可能多个线程处理同一个套接字，需要设置EPOLLONESHOT
		epoll_ctl(epfd,EPOLL_CTL_MOD,sockfd,&ev_temp); //注册可写
	}
	else
	{
		ssize_t n,nwrite,data_size;
		for(int i=0;i<4000;++i)
		{
			result[i]='a';
		}
		result[4000]='\0';
		n=data_size=strlen(result);

		//对于write，反复写直到写完
		while(n>0)
		{
			nwrite=write(sockfd,result+data_size-n,n);

			if( nwrite > 0 ) //一次write没有全发完，即当前套接字的发送缓冲区较小，只能发送一部分，需要再次write
			{
				n-=nwrite;
				continue;
			}
			else if( errno == EINTR)  //被中断打断，即使是非阻塞I/O仍然可能会被EINTR打断
			{
				continue;
			}
			else if( errno == EAGAIN)  //write上读取到EAGAIN,一种简单而常见的做法是直接跳出
			{
				//如果直接跳出就存在问题，即当前的数据并没有完全发完，想到的改进方法是在此时等待一定间隔后再次尝试
				usleep(100); //睡眠0.1秒后再次尝试读
				continue;
			}
			else  //write出错
			{
				err_quit("write error");
			}
		}

		epoll_ctl(epfd,EPOLL_CTL_DEL,sockfd,NULL); //直接删除

		Pthread_mutex_lock(&clifd_mutex);
		socket_close_queue.push(sockfd);
		Pthread_mutex_unlock(&clifd_mutex);
	}
}

void threadid_destructor(void *ptr)
{
	free(ptr);
}

void threadid_once(void)
{
	pthread_key_create(&r1_key,threadid_destructor);
}

void *thread_main(void *arg) //子线程建立函数
{
	//先建立线程自带数据
	Pthread_once(&r1_once,threadid_once);
	int *ptr;
	if( (ptr = (int *)pthread_getspecific(r1_key)) == NULL)
	{
		ptr=(int *)Malloc(sizeof(int));
		Pthread_setspecific(r1_key,ptr);
	}
   *ptr=*(int *)arg; //在线程自带数据中记录线程id

   free(arg); //回收动态分配的内存

	int		connfd,work_mode;

	printf("thread %d starting\n", *(int *) ptr);
	for ( ; ; ) {

    	Pthread_mutex_lock(&clifd_mutex); //使用互斥锁，确保只有一个工作线程接受任务分配
		while (0==nwork)
			Pthread_cond_wait(&clifd_cond, &clifd_mutex); //当主线程有任务下达时，通知某个睡眠在条件变量上的工作线程，注意已经接受任务的线程不会再次接受，除非已经处理完之前的任务

		connfd = work_queue.front();	 /* 得到已连接套接字的编号 */
		work_queue.pop();
		work_mode=work_queue.front(); /* 得到操作类型 */
		work_queue.pop();
		printf("工作线程 %d 接受套接字 %d 任务类型 %d \n", *(int *) ptr,connfd,work_mode);
		--nwork; //完成一个任务
		Pthread_mutex_unlock(&clifd_mutex);

		tptr[*(int *) ptr].thread_count++;
		web_child(connfd,work_mode);		/* 处理工作线程 */

		printf("工作线程 %d 目前已经处理了 %d 个任务\n", *(int *) ptr,(int)tptr[*(int *) ptr].thread_count);
	}
	return NULL;
}

void thread_make(int i)
{
   iptr=(int *) Malloc(sizeof(int)); //对原来的程序做了修改，原来pthread_create中将值强转成i传递，现在改成动态分配,但是要记得回收
   *iptr=i;
	Pthread_create(&tptr[i].thread_tid, NULL, &thread_main,iptr);
	return;		/* 主线程返回 */
}

void set_nonblocking(int fd) //设置非阻塞I/O
{
    int flags;
    flags = Fcntl(fd, F_GETFL, 0);
    Fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

int main(int argc, char **argv)
{
	int	 i, listenfd, connfd;
	socklen_t	addrlen, clilen;
	struct sockaddr	*cliaddr;

	if (argc == 3)
	{
		listenfd = Tcp_listen(NULL, argv[1], &addrlen);
		printf("端口号为%s\n",argv[1]);
	}
	else if (argc == 4)
		listenfd = Tcp_listen(argv[1], argv[2], &addrlen);
	else
		err_quit("usage: serv08 [ <host> ] <port#> <#threads>");
	cliaddr = (sockaddr *)Malloc(addrlen);

	nthreads = atoi(argv[argc-1]);
	tptr = (Thread *)Calloc(nthreads, sizeof(Thread)); //初始化Thread结构体数组
		/* 预先生成线程池 */
	for (i = 0; i < nthreads; i++)
		thread_make(i);		/* 只有主线程返回 */
	printf("线程池生成完毕\n");

	struct epoll_event ev,events[100]; //ev用于注册事件，events数组用于返回要处理的事件
	//首先在epoll中注册监听套接字
	epfd=epoll_create(1);
	set_nonblocking(listenfd); //设置I/O非阻塞，最好设置监听套接字为非阻塞套接字，因为服务器繁忙时不会立即调用accept，此时可能服务器提前收到客户的RST,随后从已连接队列中移除这个连接，此后再调用accept，此时如果没有新的连接到来，线程会阻塞在accept的调用上
	ev.data.fd=listenfd;
	ev.events=EPOLLIN | EPOLLET; //采用边缘触发模式，监听是否有新的连接，注意每次触发可能有多个连接，需要使用while循环accept
	epoll_ctl(epfd,EPOLL_CTL_ADD,listenfd,&ev);
	nwork=0;

	for( int count=0; ;count++)
	{
		//printf("count = %d\n",count);
		//检查是否有套接字需要关闭
		Pthread_mutex_lock(&clifd_mutex);
       while(!socket_close_queue.empty())
        {
    	   Close(socket_close_queue.front());
    	   printf("成功关闭套接字%d\n",socket_close_queue.front());
    	   socket_close_queue.pop();
        }
      Pthread_mutex_unlock(&clifd_mutex);

		nfds=epoll_wait(epfd,events,100,-1);
      for(int i=0;i<nfds;++i)
       {
    	  if(events[i].data.fd==listenfd) //如果是监听套接字则说明有新的连接，此时循环accept，并注册新连接的可读
    	  {
    		  //printf("监听到连接\n");
    		  clilen = addrlen;
    		  while( (connfd = accept(listenfd, cliaddr, &clilen))> 0 )//边缘触发方式下可能有多个连接需要接受，使用while
    		  {
    			  //printf("%d\n",connfd);
    			  //注册已连接套接字
    			  set_nonblocking(connfd); //设置I/O非阻塞
    			  ev.data.fd=connfd;
    			  ev.events=EPOLLIN | EPOLLET | EPOLLONESHOT; //使用边缘触发方式，注意边缘触发下也存在可能多个线程处理同一个套接字，需要设置EPOLLONESHOT
    			  epoll_ctl(epfd,EPOLL_CTL_ADD,connfd,&ev);
    		  }
    	  }
    	  else if(events[i].events & EPOLLIN )  //如果是已连接套接字，则判断是可读，将任务分给工作线程
    	  {
    		  //printf("监听到可读事件\n");
    		  Pthread_mutex_lock(&clifd_mutex);
    		  nwork++;
    		  work_queue.push(events[i].data.fd);
    		  work_queue.push(0);
    		  Pthread_cond_signal(&clifd_cond);  //发出信号，唤醒工作线程起床干活
    		  Pthread_mutex_unlock(&clifd_mutex);
    	  }
    	  else if(events[i].events & EPOLLOUT )  //如果是已连接套接字，则判断是可写，将任务分给工作线程
    	  {
    		  //printf("监听到可写事件\n");
    		  Pthread_mutex_lock(&clifd_mutex);
    		  nwork++;
    		  work_queue.push(events[i].data.fd);
    		  work_queue.push(1);
    		  Pthread_cond_signal(&clifd_cond);  //发出信号，唤醒工作线程起床干活
    		  Pthread_mutex_unlock(&clifd_mutex);
    	  }
       }
	}

}
```
