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
   

## epoll系统调用
----------------------- 
### epoll_create
       #include <sys/epoll.h>
       
       int epoll_create(int size);

功能描述：创建一个epoll实例

关键参数说明：size无意义，但必须大于0

返回值：实例对应的文件描述符，即epoll_ctl()和epoll_wait()的epfd传参

参考链接：[epoll_create](http://man7.org/linux/man-pages/man2/epoll_create.2.html)

### epoll_ctl
       #include <sys/epoll.h>
       
       int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
       
功能描述：添加、删除、修改某一个监听事件

关键参数说明：

    op决定了这次epoll_ctl操作是添加还是删除还是修改
    fd为监听事件的文件描述符
    event结构体里存放用户指定的与该监听事件绑定的信息

返回值：实例对应的文件描述符，即epoll_ctl()和epoll_wait()的epfd传参

参考链接：[epoll_create](http://man7.org/linux/man-pages/man2/epoll_create.2.html)

### epoll_wait
       #include <sys/epoll.h>

       int epoll_wait(int epfd, struct epoll_event *events,
                      int maxevents, int timeout);

### 最小示例
## epoll实现原理
------------------------ 
### 数据结构
### 核心结构体
### 关键流程
## epoll动态追踪
------------------------ 
### 问题描述
### 解决方案