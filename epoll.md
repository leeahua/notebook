## epoll 机制

### 什么是epoll？

```
在linux还没有epoll机制前，select和poll作为IO多路复用的机制实现并发程序，但这两种方式有着如下缺点：

通过select方式单个进程能够监控的文件描述符不得超过 进程可打开的文件个数上限，默认为1024， 即便强行修改了这个上限，还会遇到性能问题；
select轮询效率随着监控个数的增加而性能变差
select从内核空间返回到用户空间的是整个文件描述符数组，应用程序还需要额外再遍历整个数组才知道哪些文件描述符触发了相应事件。
```



## epoll 函数

```c
int epoll_create(int size)；
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);

struct epoll_event {
    __uint32_t events;
    epoll_data_t data;
};



/* 
外部软中断系统调用处理函数
size仅仅用来检测是否大于0，并没有真正使用。sys_epoll_create1过程检查参数，然后再调用epoll_create1。
*/
SYSCALL_DEFINE1(epoll_create, int, size)
{
    if (size <= 0)
        return -EINVAL;
    return sys_epoll_create1(0);
}

SYSCALL_DEFINE1(epoll_create1, int, flags)
{
    int error, fd;
    struct eventpoll *ep = NULL;
    struct file *file;

    // 创建内部数据结构eventpoll 【小节2.3】
    error = ep_alloc(&ep);
    //查询未使用的fd
    fd = get_unused_fd_flags(O_RDWR | (flags & O_CLOEXEC));

    //创建file实例，以及匿名inode节点和dentry等数据结构
    file = anon_inode_getfile("[eventpoll]", &eventpoll_fops, ep,
                 O_RDWR | (flags & O_CLOEXEC));

    ep->file = file;
    fd_install(fd, file);  //建立fd和file的关联关系
    return fd;

out_free_fd:
    put_unused_fd(fd);
out_free_ep:
    ep_free(ep);
    return error;
}

//epoll_create的过程主要是创建并初始化数据结构eventpoll，以及创建file实例，并将ep放入file->private。


static int ep_alloc(struct eventpoll **pep)
{
    int error;
    struct user_struct *user;
    struct eventpoll *ep;  //【小节2.4.1】

    user = get_current_user();
    error = -ENOMEM;
    ep = kzalloc(sizeof(*ep), GFP_KERNEL);

    spin_lock_init(&ep->lock);
    mutex_init(&ep->mtx);
    init_waitqueue_head(&ep->wq); //初始化epoll文件的等待队列
    init_waitqueue_head(&ep->poll_wait); //初始化eventpoll文件的等待队列
    INIT_LIST_HEAD(&ep->rdllist);
    ep->rbr = RB_ROOT;
    ep->ovflist = EP_UNACTIVE_PTR;
    ep->user = user;
    *pep = ep;
    return 0;

free_uid:
    free_uid(user);
    return error;
}

//为了方便后续源码的阅读，这里列举前后文所涉及到的核心struct
struct eventpoll {
    spinlock_t lock;
    struct mutex mtx;

    wait_queue_head_t wq; //sys_epoll_wait（）使用的等待队列
    wait_queue_head_t poll_wait; //file->poll()使用的等待队列

    struct list_head rdllist; //所有准备就绪的文件描述符列表
    struct rb_root rbr; //用于储存已监控fd的红黑树根节点

    // 当正在向用户空间传递事件，则就绪事件会临时放到该队列，否则直接放到rdllist
    struct epitem *ovflist;
    struct wakeup_source *ws; // 当ep_scan_ready_list运行时使用wakeup_source
    struct user_struct *user; //创建eventpoll描述符的用户

    struct file *file;
    int visited;           //用于优化循环检测检查
    struct list_head visited_list_link;
};

struct epitem {
    union {
        struct rb_node rbn; //RB树节点将此结构链接到eventpoll RB树
        struct rcu_head rcu; //用于释放结构体epitem
    };

    struct list_head rdllink; //用于将此结构链接到eventpoll就绪列表的列表标头
    struct epitem *next; //配合ovflist一起使用来保持单向链的条目
    struct epoll_filefd ffd; //此条目引用的文件描述符信息
    int nwait; //附加到poll轮询中的活跃等待队列数

    struct list_head pwqlist;
    struct eventpoll *ep;  //epi所属的ep
    struct list_head fllink; //链接到file条目列表的列表头
    struct wakeup_source __rcu *ws; //设置EPOLLWAKEUP时使用的wakeup_source
    struct epoll_event event; //监控的事件和文件描述符
};

struct epoll_event {
    __u32 events;
    __u64 data;
} EPOLL_PACKED;
struct epoll_filefd {
    struct file *file;
    int fd;
} __packed;
struct ep_pqueue {
    poll_table pt;
    struct epitem *epi;
};
typedef struct poll_table_struct {
    poll_queue_proc _qproc;
    unsigned long _key;
} poll_table;

struct eppoll_entry {
    struct list_head llink; //指向epitem的列表头
    struct epitem *base; //指向epitem的指针
    wait_queue_t wait; //指向target file等待队列
    wait_queue_head_t *whead; //执行wait等待队列
};

//fd 操作函数
SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd,
        struct epoll_event __user *, event)
{
    int error;
    int full_check = 0;
    struct fd f, tf;
    struct eventpoll *ep;     //【小节2.4.1】
    struct epitem *epi;       //【小节2.4.2】
    struct epoll_event epds;  //【小节2.4.3】
    struct eventpoll *tep = NULL;

    error = -EFAULT;
    //将用户空间的epoll_event 拷贝到内核
    if (ep_op_has_event(op) &&
        copy_from_user(&epds, event, sizeof(struct epoll_event)))

    f = fdget(epfd); //epfd对应的文件
    tf = fdget(fd); //fd对应的文件

    if (!tf.file->f_op->poll) //目标文件描述符必须支持poll
        goto error_tgt_fput;

    if (ep_op_has_event(op))   //检查是否允许EPOLLWAKEUP
        ep_take_care_of_epollwakeup(&epds);

    ep = f.file->private_data; // 取出epoll_create过程创建的ep

    mutex_lock_nested(&ep->mtx, 0);
    ...
    epi = ep_find(ep, tf.file, fd); //ep红黑树中查看该fd
    switch (op) {
    case EPOLL_CTL_ADD:
        if (!epi) {
            epds.events |= POLLERR | POLLHUP;
            error = ep_insert(ep, &epds, tf.file, fd, full_check); //见【小节3.2】
        }
        if (full_check)
            clear_tfile_check_list();
        break;
    case EPOLL_CTL_DEL:
        if (epi)
            error = ep_remove(ep, epi); //见【小节3.3】
        break;
    case EPOLL_CTL_MOD:
        if (epi) {
            epds.events |= POLLERR | POLLHUP;
            error = ep_modify(ep, epi, &epds); //见【小节3.4】
        }
        break;
    }
    mutex_unlock(&ep->mtx);
    fdput(tf);
    fdput(f);
    ...
    return error;
}
static int ep_insert(struct eventpoll *ep, struct epoll_event *event,
             struct file *tfile, int fd, int full_check)
{
    int error, revents, pwake = 0;
    unsigned long flags;
    long user_watches;
    struct epitem *epi;
    struct ep_pqueue epq; //[小节2.4.5]

    user_watches = atomic_long_read(&ep->user->epoll_watches);
    if (unlikely(user_watches >= max_user_watches))
        return -ENOSPC;
    if (!(epi = kmem_cache_alloc(epi_cache, GFP_KERNEL)))
        return -ENOMEM;

    //构造并填充epi结构体
    INIT_LIST_HEAD(&epi->rdllink);
    INIT_LIST_HEAD(&epi->fllink);
    INIT_LIST_HEAD(&epi->pwqlist);
    epi->ep = ep;
    ep_set_ffd(&epi->ffd, tfile, fd); // 将tfile和fd都赋值给ffd，ffd结构体见[小节2.4.4]
    epi->event = *event;
    epi->nwait = 0;
    epi->next = EP_UNACTIVE_PTR;
    if (epi->event.events & EPOLLWAKEUP) {
        error = ep_create_wakeup_source(epi);
    } else {
        RCU_INIT_POINTER(epi->ws, NULL);
    }

    epq.epi = epi;
    //设置轮询回调函数【小节3.2.1】
    init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);

    //执行poll方法【小节3.2.2】
    revents = ep_item_poll(epi, &epq.pt);

    spin_lock(&tfile->f_lock);
    list_add_tail_rcu(&epi->fllink, &tfile->f_ep_links);
    spin_unlock(&tfile->f_lock);

    ep_rbtree_insert(ep, epi); //将将当前epi添加到RB树

    spin_lock_irqsave(&ep->lock, flags);
    //事件就绪 并且 epi的就绪队列有数据
    if ((revents & event->events) && !ep_is_linked(&epi->rdllink)) {
        list_add_tail(&epi->rdllink, &ep->rdllist);
        ep_pm_stay_awake(epi);

        //唤醒正在等待文件就绪，即调用epoll_wait的进程
        if (waitqueue_active(&ep->wq))
            wake_up_locked(&ep->wq);
        if (waitqueue_active(&ep->poll_wait))
            pwake++;
    }

    spin_unlock_irqrestore(&ep->lock, flags);
    atomic_long_inc(&ep->user->epoll_watches);

    if (pwake)
        ep_poll_safewake(&ep->poll_wait); //唤醒等待eventpoll文件就绪的进程
    return 0;
...
}
//将ep_pqueue->pt的成员变量_qproc设置为ep_ptable_queue_proc函数
static inline void init_poll_funcptr(poll_table *pt, poll_queue_proc qproc)
{
    pt->_qproc = qproc;
    pt->_key   = ~0UL; /* all events enabled */
}
//，poll会执行poll_wait()，poll_wait()会调用epq.pt.qproc所对应的回调函数ep_ptable_queue_proc，源码如下所示。
static inline unsigned int ep_item_poll(struct epitem *epi, poll_table *pt)
{
    pt->_key = epi->event.events;
    //调用文件系统的poll核心方法 回调
    return epi->ffd.file->f_op->poll(epi->ffd.file, pt) & epi->event.events;
}

static void __pollwait(struct file *filp, wait_queue_head_t *wait_address,
                poll_table *p)
{
    //根据poll_wqueues的成员pt指针p找到所在的poll_wqueues结构指针
    struct poll_wqueues *pwq = container_of(p, struct poll_wqueues, pt);
    struct poll_table_entry *entry = poll_get_entry(pwq);
    if (!entry)
        return;
    entry->filp = get_file(filp);
    entry->wait_address = wait_address;
    entry->key = p->_key;
    //设置entry->wait.func = pollwake 【小节2.4.5】
    init_waitqueue_func_entry(&entry->wait, pollwake); 
    entry->wait.private = pwq;       // 设置private内容为pwq
    add_wait_queue(wait_address, &entry->wait); // 将该pollwake加入到等待链表头
}

static inline void
init_waitqueue_func_entry(wait_queue_t *q, wait_queue_func_t func)
{
    q->flags      = 0;
    q->private    = NULL;
    q->func       = func;  //设置唤醒回调函数
}

//设置pwq->wait的成员变量func唤醒回调函数为ep_poll_callback；并将ep_poll_callback放入等待队列whead
static void ep_ptable_queue_proc(struct file *file, wait_queue_head_t *whead,
                 poll_table *pt)
{
    struct epitem *epi = ep_item_from_epqueue(pt);
    struct eppoll_entry *pwq;  //[小节2.4.7]

    if (epi->nwait >= 0 && (pwq = kmem_cache_alloc(pwq_cache, GFP_KERNEL))) {
        //初始化回调方法 将回调函数ep_poll_callback设置为唤醒函数
        init_waitqueue_func_entry(&pwq->wait, ep_poll_callback);
        pwq->whead = whead;
        pwq->base = epi;
        //将ep_poll_callback放入等待队列whead
        add_wait_queue(whead, &pwq->wait);
        //将llink 放入epi->pwqlist的尾部
        list_add_tail(&pwq->llink, &epi->pwqlist);
        epi->nwait++;
    } else {
        epi->nwait = -1; //标记错误发生
    }
}
//fd事件回调函数 ep_poll_callback函数核心功能是将被目标fd的就绪事件到来时，将fd对应的epitem实例添加到就绪队列。
//当应用调用epoll_wait()时，内核会将就绪队列中的事件报告给应用。
static int ep_poll_callback(wait_queue_t *wait, unsigned mode, int sync, void *key)
{
    int pwake = 0;
    unsigned long flags;
    struct epitem *epi = ep_item_from_wait(wait);
    struct eventpoll *ep = epi->ep;

    spin_lock_irqsave(&ep->lock, flags);
     // 如果正在将事件传递给用户空间，我们就不能保持锁定
     //（因为我们正在访问用户内存，并且因为linux f_op-> poll()语义）。
     // 在那段时间内发生的所有事件都链接在ep-> ovflist中并在稍后重新排队。
    if (unlikely(ep->ovflist != EP_UNACTIVE_PTR)) {
        if (epi->next == EP_UNACTIVE_PTR) {
            epi->next = ep->ovflist;
            ep->ovflist = epi;
            if (epi->ws) {
                __pm_stay_awake(ep->ws);
            }
        }
        goto out_unlock;
    }

    //如果此文件已在就绪列表中，很快就会退出
    if (!ep_is_linked(&epi->rdllink)) {
        //将epi就绪事件 插入到ep就绪队列
        list_add_tail(&epi->rdllink, &ep->rdllist);
        ep_pm_stay_awake_rcu(epi);
    }

    // 如果活跃，唤醒eventpoll等待队列和 ->poll()等待队列
    if (waitqueue_active(&ep->wq))
        wake_up_locked(&ep->wq);  //当队列不为空，则唤醒进程
    if (waitqueue_active(&ep->poll_wait))
        pwake++;

out_unlock:
    spin_unlock_irqrestore(&ep->lock, flags);
    if (pwake)
        ep_poll_safewake(&ep->poll_wait);

    if ((unsigned long)key & POLLFREE) {
        list_del_init(&wait->task_list); //删除相应的wait
        smp_store_release(&ep_pwq_from_wait(wait)->whead, NULL);
    }
    return 1;
}

//判断等待队列是否为空
static inline int waitqueue_active(wait_queue_head_t *q)
{
	return !list_empty(&q->task_list);
}

static int ep_modify(struct eventpoll *ep, struct epitem *epi, struct epoll_event *event)
{
    int pwake = 0;
    unsigned int revents;
    poll_table pt;

    init_poll_funcptr(&pt, NULL);

    epi->event.events = event->events; /* need barrier below */
    epi->event.data = event->data; /* protected by mtx */
    if (epi->event.events & EPOLLWAKEUP) {
        if (!ep_has_wakeup_source(epi))
            ep_create_wakeup_source(epi);
    } else if (ep_has_wakeup_source(epi)) {
        ep_destroy_wakeup_source(epi);
    }

    smp_mb();

    revents = ep_item_poll(epi, &pt);

    if (revents & event->events) {
        spin_lock_irq(&ep->lock);
        if (!ep_is_linked(&epi->rdllink)) {
            list_add_tail(&epi->rdllink, &ep->rdllist);
            ep_pm_stay_awake(epi);

            /* Notify waiting tasks that events are available */
            if (waitqueue_active(&ep->wq))
                wake_up_locked(&ep->wq);
            if (waitqueue_active(&ep->poll_wait))
                pwake++;
        }
        spin_unlock_irq(&ep->lock);
    }

    /* We have to call this outside the lock */
    if (pwake)
        ep_poll_safewake(&ep->poll_wait);

    return 0;
}


//用户发起系统调用引起软中断对应的中断系统处理函数
SYSCALL_DEFINE4(epoll_wait, int, epfd, struct epoll_event __user *, events,
        int, maxevents, int, timeout)
{
    int error;
    struct fd f;
    struct eventpoll *ep;

    //检测参数
    if (maxevents <= 0 || maxevents > EP_MAX_EVENTS)
        return -EINVAL;

    //检查用户空间传递的内存是否可写
    if (!access_ok(VERIFY_WRITE, events, maxevents * sizeof(struct epoll_event)))
        return -EFAULT;

    f = fdget(epfd);  //获取eventpoll文件
    ep = f.file->private_data;
    //真正执行的核心函数
    error = ep_poll(ep, events, maxevents, timeout);

error_fput:
    fdput(f);
    return error;
}

static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events,
           int maxevents, long timeout)
{
    int res = 0, eavail, timed_out = 0;
    unsigned long flags;
    long slack = 0;
    wait_queue_t wait;
    ktime_t expires, *to = NULL;

    if (timeout > 0) { //超时设置
        struct timespec end_time = ep_set_mstimeout(timeout);
        slack = select_estimate_accuracy(&end_time);
        to = &expires;
        *to = timespec_to_ktime(end_time);
    } else if (timeout == 0) {
        //timeout等于0为非阻塞操作，此处避免不必要的等待队列循环
        timed_out = 1;
        spin_lock_irqsave(&ep->lock, flags);
        goto check_events;
    }

fetch_events:
    spin_lock_irqsave(&ep->lock, flags);

    if (!ep_events_available(ep)) {  //【小节4.2.1】
        //没有事件就绪则进入睡眠状态，当事件就绪后可通过ep_poll_callback()来唤醒
        //将当前进程放入wait等待队列 【小节4.2.2】
        init_waitqueue_entry(&wait, current);
        //将当前进程加入eventpoll等待队列，等待文件就绪、超时或中断信号
        __add_wait_queue_exclusive(&ep->wq, &wait);

        for (;;) {
            set_current_state(TASK_INTERRUPTIBLE);
            if (ep_events_available(ep) || timed_out) //就绪队列不为空 或者超时，则跳出循环
                break;
            if (signal_pending(current)) { //有待处理信号，则跳出循环
                res = -EINTR;
                break;
            }

            spin_unlock_irqrestore(&ep->lock, flags);
            //主动出让CPU，从这里开始进入睡眠状态
            if (!freezable_schedule_hrtimeout_range(to, slack,
                                HRTIMER_MODE_ABS))
                timed_out = 1;

            spin_lock_irqsave(&ep->lock, flags);
        }
        __remove_wait_queue(&ep->wq, &wait); //从队列中移除wait
        set_current_state(TASK_RUNNING);
    }
check_events:
    eavail = ep_events_available(ep);
    spin_unlock_irqrestore(&ep->lock, flags);

    //尝试传输就绪事件到用户空间，如果没有获取就绪事件，但还剩下超时，则会再次retry
    if (!res && eavail &&
        !(res = ep_send_events(ep, events, maxevents)) && !timed_out)
        goto fetch_events;

    return res;
}

static inline void init_waitqueue_entry(wait_queue_t *q, struct task_struct *p)
{
	q->flags	= 0;
	q->private	= p;
	q->func		= default_wake_function;  //设置唤醒函数
}

static inline int ep_events_available(struct eventpoll *ep)
{
	return !list_empty(&ep->rdllist) || ep->ovflist != EP_UNACTIVE_PTR;
}
```

## 调用过程总结

```C
epoll_create()：创建并初始化eventpoll结构体ep，并将ep放入file->private，并返回fd
epoll_ctl()：以插入epi为例(进入ep_insert)
      init_poll_funcptr()：将ep_pqueue->pt的成员变量_qproc设置为ep_ptable_queue_proc函数，用于poll_wait()的回调函数；
      ep_item_poll()：执行上面设置的回调函数ep_ptable_queue_proc
      ep_ptable_queue_proc()：设置pwq->wait的成员变量func唤醒回调函数为ep_poll_callback，这是用于就绪事件触发时，唤醒该进程所用的回调函数；再将             	ep_poll_callback放入等待队列头whead
epoll_wait()：主要工作是执行ep_poll()方法
        wait->func的唤醒函数为default_wake_function()，并将等待队列项加入ep->wq
        freezable_schedule_hrtimeout_range()：出让CPU，进入睡眠状态之后，当其他进程就绪事件发生时便会唤醒相应等待队列上的进程。
    比如监控的是可写事件，则会在write()方法中调用wake_up方法唤醒相对应的等待队列上的进程，当唤醒后执行前面设置的唤醒回调函数ep_poll_callback函数。
ep_poll_callback()：目标fd的就绪事件到来时，将epi->rdllink加入ep->rdllist的队列，导致rdlist不空，从而进程被唤醒，epoll_wait得以继续执行。回到epoll_wait()，从队列中移除wait，再将传输就绪事件到用户空间。
epoll比select更高效的一点是：epoll监控的每一个文件fd就绪事件触发，导致相应fd上的回调函数ep_poll_callback()被调用
```





## 2. epoll 的优点

```
（1） 支持一个进程打开大数目的socket描述符(FD)

    select 最不能忍受的是一个进程所打开的FD是有一定限制的，由FD_SETSIZE设置，默认值是2048。对于那些需要支持的上万连接数目的IM服务器来说显 然太少了。这时候你一是可以选择修改这个宏然后重新编译内核，不过资料也同时指出这样会带来网络效率的下降，二是可以选择多进程的解决方案(传统的 Apache方案)，不过虽然linux上面创建进程的代价比较小，但仍旧是不可忽视的，加上进程间数据同步远比不上线程间同步的高效，所以也不是一种完 美的方案。不过 epoll则没有这个限制，它所支持的FD上限是最大可以打开文件的数目，这个数字一般远大于2048,举个例子,在1GB内存的机器上大约是10万左 右，具体数目可以cat /proc/sys/fs/file-max察看,一般来说这个数目和系统内存关系很大。
（2） IO 效率不随FD数目增加而线性下降
      传统的select/poll另一个致命弱点就是当你拥有一个很大的socket集合，不过由于网络延时，任一时间只有部分的socket是"活跃"的， 但是select/poll每次调用都会线性扫描全部的集合，导致效率呈现线性下降。但是epoll不存在这个问题，它只会对"活跃"的socket进行 操作---这是因为在内核实现中epoll是根据每个fd上面的callback函数实现的。那么，只有"活跃"的socket才会主动的去调用 callback函数，其他idle状态socket则不会，在这点上，epoll实现了一个"伪"AIO，因为这时候推动力在os内核。在一些 benchmark中，如果所有的socket基本上都是活跃的---比如一个高速LAN环境，epoll并不比select/poll有什么效率，相反，如果过多使用epoll_ctl,效率相比还有稍微的下降。但是一旦使用idle connections模拟WAN环境,epoll的效率就远在select/poll之上了。
（3）使用mmap加速内核 与用户空间的消息传递。
    这点实际上涉及到epoll的具体实现了。无论是select,poll还是epoll都需要内核把FD消息通知给用户空间，如何避免不必要的内存拷贝就 很重要，在这点上，epoll是通过内核与用户空间mmap同一块内存实现的。而如果你想我一样从2.5内核就关注epoll的话，一定不会忘记手工 mmap这一步的。
（4）内核微调
    这一点其实不算epoll的优点了，而是整个linux平台的优点。也许你可以怀疑 linux平台，但是你无法回避linux平台赋予你微调内核的能力。比如，内核TCP/IP协议栈使用内存池管理sk_buff结构，那么可以在运行时期动态调整这个内存pool(skb_head_pool)的大小--- 通过echo XXXX>/proc/sys/net/core/hot_list_length完成。再比如listen函数的第2个参数(TCP完成3次握手 的数据包队列长度)，也可以根据你平台内存大小动态调整。更甚至在一个数据包面数目巨大但同时每个数据包本身大小却很小的特殊系统上尝试最新的NAPI网卡驱动架构。
```

3. epoll 的工作模式

 ```
令人高兴的是，2.6内核的epoll比其2.5开发版本的/dev/epoll简洁了许多，所以，大部分情况下，强大的东西往往是简单的。唯一有点麻烦 是epoll有2种工作方式:LT和ET。
LT(level triggered 水平触发)是缺省的工作方式：
并且同时支持block和no-block socket.在这种做法中，内核告诉你一个文件描述符是否就绪了，然后你可以对这个就绪的fd进行IO操作。如果你不作任何操作，内核还是会继续通知你的，所以，这种模式编程出错误可能性要小一点。传统的select/poll都是这种模型的代表．
ET (edge-triggered 边缘触发)是高速工作方式：
只支持no-block socket。
在这种模式下，当描述符从未就绪变为就绪时，内核通过epoll告诉你。然后它会假设你知道文件描述符已经就绪，并且不会再为那个文件描述符发送更多的就绪通知，直到你做了某些操作导致那个文件描述符不再为就绪状态了(比如，你在发送，接收或者接收请求，或者发送接收的数据少于一定量时导致 了一个EWOULDBLOCK错误）。但是请注意，如果一直不对这个fd作IO操作(从而导致它再次变成未就绪)，内核不会发送更多的通知(only once)。

epoll只有epoll_create,epoll_ctl,epoll_wait 3个系统调用，具体用法请参考，在也有一个完整的例子，大家一看就知道如何使用了Leader/follower模式线程 pool实现，以及和epoll的配合。
 ```



