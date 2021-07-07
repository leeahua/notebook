# select 、poll



## select 机制

```
do_select()是整个select的核心过程，主要工作流程如下：

poll_initwait()：设置poll_wqueues->poll_table的成员变量poll_queue_proc为__pollwait函数；同时记录当前进程task_struct记在pwq结构体的polling_task。
f_op->poll()：会调用poll_wait()，进而执行上一步设置的方法__pollwait();
__pollwait()：设置wait->func唤醒回调函数为pollwake函数，并将poll_table_entry->wait加入等待队列 4.poll_schedule_timeout()：该进程进入带有超时的睡眠状态。
之后，当其他进程就绪事件发生时便会唤醒相应等待队列上的进程。比如监控的是可写事件，则会在write()方法中调用wake_up方法唤醒相对应的等待队列上的进程，当唤醒后执行前面设置的唤醒回调函数pollwake函数。

pollwake()：先从wait->private的polling_task获取处于等待睡眠状态的目标进程，调用default_wake_function()来唤醒该进程；
poll_freewait()：当进程唤醒后，将就绪事件结果保存在fds的res_in、res_out、res_ex，然后把该等待队列从该队列头中移除。
回到core_sys_select()，将就绪事件结果拷贝到用户空间。
以上有个缺陷就是每次会轮询所有的fd的f_op->poll()。
```

## poll 机制

```
select 
    sys_select
        core_sys_select
            do_select
                poll_initwait
                while
                    poll
                    poll_schedule_timeout
                poll_freewait

poll
    sys_poll
        do_sys_poll
            poll_initwait
            do_poll
                do_pollfd
                poll_schedule_timeout
            poll_freewait
```

## select和poll的区别

```
我们会发现，select和poll机制的原理非常相近，主要是一些数据结构的不同，最终到驱动层都会执行f_op->poll()，执行__pollwait()把自己挂入等待队列。 一旦有事件发生时便会唤醒等待队列上的进程。比如监控的是可写事件，则会在write()方法中调用wakeup方法唤醒相对应的等待队列上的进程。这一切都是基于底层文件系统作为基石来完成IO多路复用的事件监控功能。
```

## select poll 缺陷

```
通过select方式单个进程能够监控的文件描述符不得超过 进程可打开的文件个数上限，默认为1024， 即便强行修改了这个上限，还会遇到性能问题；
select轮询效率随着监控个数的增加而性能变差
select从内核空间返回到用户空间的是整个文件描述符数组，应用程序还需要额外再遍历整个数组才知道哪些文件描述符触发了相应事件。
```

## 总结

## _1. select_

```
int select (int maxfd, fd_set *readfds, 
                   fd_set *writefds, 
                   fd_set *exceptfds, 
                   struct timeval *timeout);
maxfd：代表要监控的最大文件描述符fd+1
writefds：监控可写fd
readfds：监控可读fd
exceptfds：监控异常fd
timeout：超时时长
NULL，代表没有设置超时，则会一直阻塞直到文件描述符上的事件触发
0，代表不等待，立即返回，用于检测文件描述符状态
正整数，代表当指定时间没有事件触发，则超时返回
select函数监控3类文件描述符，调用select函数后会阻塞，直到描述符fd准备就绪（有数据可读、可写、异常）或者超时，函数便返回。 当select函数返回后，可通过遍历描述符集合，找到就绪的描述符。

select缺点

文件描述符个数受限：单进程能够监控的文件描述符的数量存在最大限制，在Linux上一般为1024，可以通过修改宏定义增大上限，但同样存在效率低的弱势;
性能衰减严重：IO随着监控的描述符数量增长，其性能会线性下降;
```

## _2.poll_

```
原型：

int poll (struct pollfd *fds, unsigned int nfds, int timeout);
其中pollfd表示监视的描述符集合，如下

struct pollfd {
    int fd; //文件描述符
    short events; //监视的请求事件
    short revents; //已发生的事件
};
pollfd结构包含了要监视的event和发生的event，并且pollfd并没有最大数量限制。 
和select函数一样，当poll函数返回后，可以通过遍历描述符集合，找到就绪的描述符。

poll缺点

从上面看select和poll都需要在返回后，通过遍历文件描述符来获取已经就绪的socket。
同时连接的大量客户端在同一时刻可能只有很少的处于就绪状态，因此随着监视的描述符数量的增长，其性能会线性下降。
```

## _3. epoll_

```
epoll是在内核2.6中提出的，是select和poll的增强版。相对于select和poll来说，epoll更加灵活，没有描述符数量限制。epoll使用一个文件描述符管理多个描述符，将用户空间的文件描述符的事件存放到内核的一个事件表中，这样在用户空间和内核空间的copy只需一次。epoll机制是Linux最高效的I/O复用机制，在一处等待多个文件句柄的I/O事件。

select/poll都只有一个方法，epoll操作过程有3个方法，分别是epoll_create()， epoll_ctl()，epoll_wait()。
3.1 epoll_create
int epoll_create(int size)；
功能：用于创建一个epoll的句柄，size是指监听的描述符个数， 现在内核支持动态扩展，该值的意义仅仅是初次分配的fd个数，后面空间不够时会动态扩容。 当创建完epoll句柄后，占用一个fd值.

ls /proc/<pid>/fd/  //可通过终端执行，看到该fd
使用完epoll后，必须调用close()关闭，否则可能导致fd被耗尽。

3.2 epoll_ctl
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
功能：用于对需要监听的文件描述符(fd)执行op操作，比如将fd加入到epoll句柄。

epfd：是epoll_create()的返回值；
op：表示op操作，用三个宏来表示，分别代表添加、删除和修改对fd的监听事件；
EPOLL_CTL_ADD(添加)
EPOLL_CTL_DEL(删除)
EPOLL_CTL_MOD（修改）
fd：需要监听的文件描述符；
epoll_event：需要监听的事件，struct epoll_event结构如下：

  struct epoll_event {
    __uint32_t events;  /* Epoll事件 */
    epoll_data_t data;  /*用户可用数据*/
  };
events可取值：(表示对应的文件描述符的操作)

EPOLLIN ：可读（包括对端SOCKET正常关闭）；
EPOLLOUT：可写；
EPOLLERR：错误；
EPOLLHUP：中断；
EPOLLPRI：高优先级的可读（这里应该表示有带外数据到来）；
EPOLLET： 将EPOLL设为边缘触发模式，这是相对于水平触发来说的。
EPOLLONESHOT：只监听一次事件，当监听完这次事件之后就不再监听该事件

3.3 epoll_wait
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
功能：等待事件的上报

epfd：等待epfd上的io事件，最多返回maxevents个事件；
events：用来从内核得到事件的集合；
maxevents：events数量，该maxevents值不能大于创建epoll_create()时的size；
timeout：超时时间（毫秒，0会立即返回）。
该函数返回需要处理的事件数目，如返回0表示已超时。
```



## select 、 poll 、 epoll 对比

```
1）在 select/poll中，进程只有在调用一定的方法后，内核才对所有监视的文件描述符进行扫描，而epoll事先通过epoll_ctl()来注册一个文件描述符，一旦基于某个文件描述符就绪时，内核会采用类似callback的回调机制，迅速激活这个文件描述符，当进程调用epoll_wait() 时便得到通知。
此处去掉了遍历文件描述符，而是通过监听回调的的机制。这正是epoll的魅力所在。

epoll优势

监视的描述符数量不受限制，所支持的FD上限是最大可以打开文件的数目，具体数目可以cat /proc/sys/fs/file-max查看，一般来说这个数目和系统内存关系很大，以3G的手机来说这个值为20-30万。

IO性能不会随着监视fd的数量增长而下降。epoll不同于select和poll轮询的方式，而是通过每个fd定义的回调函数来实现的，只有就绪的fd才会执行回调函数。

如果没有大量的空闲或者死亡连接，epoll的效率并不会比select/poll高很多。但当遇到大量的空闲连接的场景下，epoll的效率大大高于select/poll。


```

