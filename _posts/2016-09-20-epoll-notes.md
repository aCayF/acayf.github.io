---

layout: post
title: "epoll学习笔记(更新中)"
category: "OS"
draft: false
tags: epoll

--- 

## 目录

1. epoll系统调用
   * epoll_create
   * epoll_ctl
   * epoll_wait
   * 最小示例
2. epoll实现原理
   * 数据结构
   * 核心结构体
   * 关键流程
3. epoll动态追踪
   * 问题描述
   * 解决方案

----------------    

## epoll系统调用

### epoll_create

```c
#include <sys/epoll.h>

int epoll_create(int size);
```

功能描述：创建一个epoll实例

关键参数说明：size无意义，但必须大于0

返回值：所创实例对应的文件描述符，即epoll_ctl()和epoll_wait()所需的epfd传参

参考链接：[epoll_create](http://man7.org/linux/man-pages/man2/epoll_create.2.html)

### epoll_ctl

```c
#include <sys/epoll.h>

int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```
       
功能描述：添加、删除、修改某一监听事件

关键参数说明：

* op决定了这次epoll_ctl操作是添加还是删除还是修改
* fd为所要操作的监听事件的文件描述符
* event结构体里存放用户定义的与该监听事件相关的信息，可称为用户自定义事件信息，存放何种信息完全是用户行为，结构体定义如下

```c
typedef union epoll_data {
    void        *ptr;
    int          fd;
    uint32_t     u32;
    uint64_t     u64;
} epoll_data_t;

struct epoll_event {
    uint32_t     events;      /* Epoll events */
    epoll_data_t data;        /* User data variable */
};
```

返回值：操作成功返回0，操作失败返回-1

参考链接：[epoll_ctl](http://man7.org/linux/man-pages/man2/epoll_ctl.2.html)

### epoll_wait

```c
#include <sys/epoll.h>

int epoll_wait(int epfd, struct epoll_event *events,
               int maxevents, int timeout);
```
                      
功能描述：等待直到有监听事件到达或等待超时或调用被中断，至多返回maxevents个事件，且到达的事件信息会存放在event结构体中

关键参数说明：

* event指向到达事件对应的用户自定义事件信息，由内核负责传入数据，结构体定义如epoll_ctl小节
* timeout设定超时值，为0则不管监听事件到达与否都立即返回，为-1则无限等待直到有监听事件到达

返回值：返回到达的监听事件数

参考链接：[epoll_wait](http://man7.org/linux/man-pages/man2/epoll_wait.2.html)      

### 最小示例

----------------

## epoll实现原理
 
### 数据结构

#### 双向循环列表

#### 红黑树

### 核心结构体

#### eventpoll

```c
    struct eventpoll {
        spinlock_t lock;//自旋锁，实现对wq，rdllist，ovflist的互斥访问？
        struct mutex mtx;//互斥锁，实现对epoll实例管理的事件结构体epitem的互斥访问？
        wait_queue_head_t wq;
        wait_queue_head_t poll_wait;
        struct list_head rdllist;//到达事件链
        struct rb_root rbr;
        struct epitem *ovflist;//传送事件到用户层时的临时到达事件链
        struct user_struct *user;
        struct file *file;
        int visited;
        struct list_head visited_list_link;
    };
```

epoll_event由epoll_create创建且存放在epoll_create返回的epfd所对应的struct file的private_data成员中

#### epitem

```c
    struct epitem {
        struct rb_node rbn;
        struct list_head rdllink;//用于将epitem链入rdllist链表的链表头
        struct epitem *next;//用于将eptiem链入ovflist链表的指针
        struct epoll_filefd ffd;//存放事件struct file与fd信息
        int nwait;
        struct list_head pwqlist;
        struct eventpoll *ep;
        struct list_head fllink;
        struct epoll_event event;//存放用户自定义事件信息
    };
```

### 关键流程

#### epoll实例管理监听事件

#### epoll实例等待以及被唤醒

执行epoll_wait系统调用会进入以下代码片段

```c
SYSCALL_DEFINE4(epoll_wait, int, epfd, struct epoll_event __user *, events,
        int, maxevents, int, timeout)
{
    /*验证maxevents传参*/
    /*验证events传参*/
    /*验证epfd传参*/
    file = fget(epfd);
    ep = file->private_data;
    error = ep_poll(ep, events, maxevents, timeout);
    return error;
}
```

从代码中可以看出epoll_wait核心是调用ep_poll函数

```c
static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events,
           int maxevents, long timeout)
{
    /*计算timeout,to,slack值*/
fetch_events:
    spin_lock_irqsave(&ep->lock, flags);
    if (!ep_events_available(ep)) {
        /*将当前进程加入到wait对列*/
        __add_wait_queue(&ep->wq, &wait);

        for (;;) {
            /*?????????*/
            set_current_state(TASK_INTERRUPTIBLE);
            if (ep_events_available(ep) || timed_out)
                break;
            if (signal_pending(current)) {
                res = -EINTR;
                break;
            }
            spin_unlock_irqrestore(&ep->lock, flags);
            if (!schedule_hrtimeout_range(to, slack, HRTIMER_MODE_ABS))
                timed_out = 1;
            spin_lock_irqsave(&ep->lock, flags);
        }
        __remove_wait_queue(&ep->wq, &wait);
        set_current_state(TASK_RUNNING);
    }
    eavail = ep_events_available(ep);
    spin_unlock_irqrestore(&ep->lock, flags);
    if (!res && eavail &&
        !(res = ep_send_events(ep, events, maxevents)) && !timed_out)
        goto fetch_events;//res=0,即没有signal_pending
                          //eavail=1,即有事件到达
                          //ep_send_events返回0,即没有一个到达事件被传送到用户层
                          //timed_out=0,即未超时

    return res;
}
```

从代码中可以总结出ep_poll返回即epoll_wait返回可能的情况有如下三种：

1. 至少有一件到达事件被传送到用户层，函数返回值res为传送事件数
2. 超时，函数返回值res为0
3. 有signal_pending，函数返回值res为-EINTR

同时可以得出如果epoll_wait出现多次调用ep_send_events情况，只有在最后一次调用中才有可能将到达事件信息传送到用户层

ep_poll_callback负责唤醒等待中的epoll_wait

```c
static int ep_poll_callback(wait_queue_t *wait, unsigned mode, int sync, void *key)
{
}
```

#### epoll实例返回到达事件信息

ep_send_events负责将到达事件对应的用户自定义信息传送到用户层

```c
static int ep_send_events(struct eventpoll *ep,
              struct epoll_event __user *events, int maxevents)
{
    struct ep_send_events_data esed;
    esed.maxevents = maxevents;
    esed.events = events;
    return ep_scan_ready_list(ep, ep_send_events_proc, &esed);
}
```

ep_send_events实质是调用ep_scan_ready_list函数，额外的传入了ep_send_events_proc回调，ep_scan_ready_list函数体内会执行ep_send_events_proc回调，我们先看看ep_scan_ready_list函数体的实现

```c
static int ep_scan_ready_list(struct eventpoll *ep,
                  int (*sproc)(struct eventpoll *,
                       struct list_head *, void *),
                  void *priv)
{
    LIST_HEAD(txlist);
    mutex_lock(&ep->mtx);
    
    spin_lock_irqsave(&ep->lock, flags);
    list_splice_init(&ep->rdllist, &txlist);//1.将rdllist指向的list转接到txlist上，清空rdllist
    ep->ovflist = NULL;//2.允许ep_poll_callback往ovflist链入到达事件epitem
    spin_unlock_irqrestore(&ep->lock, flags);
    
    //考虑到执行回调(往用户层传送信息)的时间会比较长，在这段时间内很有可能有事件到达，因此在这段时间内不能加锁，所以进行了前两步操作，保证了在回调执行时间内到达的事件只会链入ovflist，此时的rdllist只允许回调本身链入Level Trigger事件
    error = (*sproc)(ep, &txlist, priv);//执行ep_send_events_proc回调

    spin_lock_irqsave(&ep->lock, flags);
    /*将ovflist链接的到达事件链入rdllist*/
    ep->ovflist = EP_UNACTIVE_PTR;//禁止ep_poll_callback往ovflist链入到达事件epitem
    list_splice(&txlist, &ep->rdllist);//将txlist指向的残留list转接回rdllist

    if (!list_empty(&ep->rdllist)) {
        if (waitqueue_active(&ep->wq))
            wake_up_locked(&ep->wq);
        if (waitqueue_active(&ep->poll_wait))
            pwake++;
    }
    spin_unlock_irqrestore(&ep->lock, flags);

    mutex_unlock(&ep->mtx);
    if (pwake)
        ep_poll_safewake(&ep->poll_wait);

    return error;
}

```

真正将到达事件信息传送给用户层的函数为ep_send_events_proc

```c
static int ep_send_events_proc(struct eventpoll *ep, struct list_head *head,
                   void *priv)
{
    //head=txlist，head链接着转接过来的ep->rdllist链表，即存放到达事件链
    for (eventcnt = 0, uevent = esed->events;
         !list_empty(head) && eventcnt < esed->maxevents;) {
        epi = list_first_entry(head, struct epitem, rdllink);//得到到达事件的epitem结构体

        list_del_init(&epi->rdllink);//从到达事件链中摘除该事件

        revents = epi->ffd.file->f_op->poll(epi->ffd.file, NULL) &
            epi->event.events;//调用事件对应驱动实现的poll方法，返回值表征可立即执行的操作，如可读操作返回POLLIN，可写操作返回POLLOUT
            
        if (revents) {
            if (__put_user(revents, &uevent->events) ||//传送poll方法返回的flag
                __put_user(epi->event.data, &uevent->data)) {//传送用户自定义事件信息
                list_add(&epi->rdllink, head);//将失败事件添入到达事件链中
                return eventcnt ? eventcnt : -EFAULT;
            }
            eventcnt++;//记录成功传送事件数
            uevent++;//指针移向下一epoll_event结构体
            if (epi->event.events & EPOLLONESHOT)
                epi->event.events &= EP_PRIVATE_BITS;
            else if (!(epi->event.events & EPOLLET)) {
                list_add_tail(&epi->rdllink, &ep->rdllist);//向ep->rdlist添加Level Triger事件
            }
        }
    }

    return eventcnt;

```

回调ep_send_events_proc返回成功传送的事件数，回调在被执行之前head链表里存放着待传送的事件链，回调在被执行之后head链表存放未传送的事件链，所以探测回调执行前后head链表的变化就能获取成功发送的事件信息，这里有个特殊情况，就是当事件poll方法返回的操作并不是期望的操作时，如期望可读事件却返回可写事件，则revent为0，该到达事件被忽略，且不会被传送到用户层

----------------

## epoll动态追踪

### 追踪目标

#### 1. 追踪epoll系统调用工作流程

#### 2. 追踪epoll_wait系统调用，获取到达事件fd信息

### 解决方案

如果用户在event_poll结构体里存放了fd信息，则只需要追踪epoll_wait返回的结构体

```stap
global epoll

probe syscall.epoll_wait
{
    if(target() != pid()) next
    epoll[tid()] = 1
}

probe syscall.epoll_wait.return
{
    tid = tid()
    if(!(tid in epoll)) next
    delete epoll[tid]
    
    n = $return
    if(n<=0) next

    printf("[%d] t%d epoll send fds: [", gettimeofday_us(), tid)
    for(i=0; i<n; i++) {
        fd = @cast($events, "epoll_event", "<sys/epoll.h>")[i]->data->fd
        printf("%d,", fd)
    }
    printf("]\n")
}
```

然而会遇到用户不在event_poll结构体里存放fd的情况，所以在这种情况下只能通过追踪内核函数来获取到达事件信息

```stap
global epoll
global fds

probe syscall.epoll_wait
{
    if(target() != pid()) next
    epoll[tid()] = 1
}

probe kernel.function("ep_send_events_proc")
{
    tid = tid()
    if(!(tid in epoll)) next

    fds[tid] = extract_fd_from_rdllist($head)
}

/* TODO : remove fd which return 0 revent after calling poll method */
probe kernel.function("ep_send_events_proc").return
{
    tid = tid()
    if(!(tid in epoll)) next

    if($return > 0) {
        printf("[%d] t%d epoll send fds: [%s][0:%d]\n", gettimeofday_us(), tid, fds[tid], $return)
    }

    delete fds[tid]
}

probe syscall.epoll_wait.return
{
    tid = tid()
    if(!(tid in epoll)) next

    delete epoll[tid]
}
```
